# -------------------------------------------------------
# SmartParcel — NET_214 Project, Spring 2026
# Author  : Mardan Ali
# ID      : 20210001377
# Email   : 20210001377@students.cud.ac.ae
# AWS Acc : 718417034014
# -------------------------------------------------------

from flask import Flask, request, jsonify
from functools import wraps
import uuid
import socket
from datetime import datetime
from decimal import Decimal
import json
import logging

import boto3
from boto3.dynamodb.conditions import Key
from botocore.exceptions import ClientError
from werkzeug.utils import secure_filename
from werkzeug.exceptions import RequestEntityTooLarge

app = Flask(__name__)
app.config["MAX_CONTENT_LENGTH"] = 5 * 1024 * 1024  # 5 MB max upload

logging.basicConfig(level=logging.INFO)

REGION = "ap-southeast-2"
TABLE_NAME = "smartparcel-parcels-20210001377"   # rubric requires this exact table name
BUCKET_NAME = "smartparcel-photos-20210001377"
QUEUE_URL = "https://sqs.ap-southeast-2.amazonaws.com/718417034014/smartparcel-notifications-20210001377"

dynamodb = boto3.resource("dynamodb", region_name=REGION)
table = dynamodb.Table(TABLE_NAME)

s3 = boto3.client("s3", region_name=REGION)
sqs = boto3.client("sqs", region_name=REGION)

API_KEYS = {
    "admin-key-123": "admin",
    "driver-key-123": "driver",
    "customer-key-123": "customer"
}

VALID_STATUSES = ["created", "picked_up", "in_transit", "delivered", "cancelled"]
STATUS_UPDATE_ALLOWED = ["picked_up", "in_transit", "delivered"]


def require_auth(allowed_roles=None):
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            api_key = request.headers.get("x-api-key")

            if not api_key or api_key not in API_KEYS:
                return jsonify({"error": "Unauthorized"}), 401

            user_role = API_KEYS[api_key]

            if allowed_roles and user_role not in allowed_roles:
                return jsonify({"error": "Forbidden"}), 403

            request.user_role = user_role
            return f(*args, **kwargs)
        return wrapper
    return decorator


def convert_decimals(obj):
    if isinstance(obj, list):
        return [convert_decimals(i) for i in obj]
    if isinstance(obj, dict):
        return {k: convert_decimals(v) for k, v in obj.items()}
    if isinstance(obj, Decimal):
        if obj % 1 == 0:
            return int(obj)
        return float(obj)
    return obj


def contains_injection(text):
    if text is None:
        return False

    value = str(text).lower()
    bad_patterns = [
        "drop table",
        "select *",
        "insert into",
        "delete from",
        "update ",
        "--",
        ";",
        "'"
    ]
    return any(pattern in value for pattern in bad_patterns)


def validate_text_field(field_name, value, max_len=200):
    if value is None or not str(value).strip():
        return f"Missing or empty field: {field_name}"

    if not isinstance(value, str):
        return f"Invalid data type for field: {field_name}"

    if len(value.strip()) > max_len:
        return f"Field too long: {field_name}"

    if contains_injection(value):
        return f"Invalid characters or injection pattern detected in field: {field_name}"

    return None


def validate_email(email):
    if not isinstance(email, str) or not email.strip():
        return "Missing or empty field: customer_email"

    if len(email.strip()) > 254:
        return "Field too long: customer_email"

    if "@" not in email or "." not in email.split("@")[-1]:
        return "Invalid email format"

    if contains_injection(email):
        return "Invalid characters or injection pattern detected in field: customer_email"

    return None


def send_status_change_to_sqs(parcel, new_status):
    driver_name = request.headers.get("X-Driver-Name", "Driver")

    message = {
        "parcel_id": parcel["parcel_id"],
        "new_status": new_status,
        "customer_email": parcel.get("customer_email", ""),
        "driver_name": driver_name,
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "photo_url": parcel.get("photo_url", "")
    }

    sqs.send_message(
        QueueUrl=QUEUE_URL,
        MessageBody=json.dumps(message)
    )


@app.before_request
def log_request():
    app.logger.info(
        f"{datetime.utcnow().isoformat()} {request.method} {request.path}"
    )


@app.after_request
def log_response(response):
    app.logger.info(
        f"{datetime.utcnow().isoformat()} {request.method} {request.path} {response.status_code}"
    )
    return response


@app.errorhandler(RequestEntityTooLarge)
def handle_large_file(e):
    return jsonify({"error": "File too large. Maximum allowed size is 5 MB"}), 400


@app.errorhandler(404)
def not_found(e):
    return jsonify({"error": "Endpoint not found"}), 404


@app.errorhandler(405)
def method_not_allowed(e):
    return jsonify({"error": "Method not allowed"}), 405


@app.errorhandler(Exception)
def handle_exception(e):
    app.logger.exception("Unhandled exception")
    return jsonify({"error": "Internal server error"}), 500


@app.route("/health", methods=["GET"])
def health():
    return jsonify({
        "status": "healthy",
        "hostname": socket.gethostname(),
        "timestamp": datetime.utcnow().isoformat()
    }), 200


@app.route("/api/parcels", methods=["POST"])
@require_auth(allowed_roles=["admin", "driver"])
def create_parcel():
    data = request.get_json(silent=True)

    if not data:
        return jsonify({"error": "Request body must be valid JSON"}), 400

    sender_error = validate_text_field("sender_name", data.get("sender_name"))
    if sender_error:
        return jsonify({"error": sender_error}), 400

    receiver_error = validate_text_field("receiver_name", data.get("receiver_name"))
    if receiver_error:
        return jsonify({"error": receiver_error}), 400

    address_error = validate_text_field("delivery_address", data.get("delivery_address"), max_len=500)
    if address_error:
        return jsonify({"error": address_error}), 400

    email_error = validate_email(data.get("customer_email"))
    if email_error:
        return jsonify({"error": email_error}), 400

    now = datetime.utcnow().isoformat()
    parcel_id = str(uuid.uuid4())

    parcel = {
        "parcel_id": parcel_id,
        "sender_name": data["sender_name"].strip(),
        "receiver_name": data["receiver_name"].strip(),
        "delivery_address": data["delivery_address"].strip(),
        "customer_email": data["customer_email"].strip(),
        "status": "created",
        "created_at": now,
        "status_history": [
            {
                "status": "created",
                "timestamp": now
            }
        ]
    }

    table.put_item(Item=parcel)

    return jsonify({
        "message": "Parcel created successfully",
        "parcel_id": parcel_id,
        "parcel": parcel
    }), 201


@app.route("/api/parcels/<parcel_id>", methods=["GET"])
@require_auth(allowed_roles=["admin", "driver", "customer"])
def get_parcel(parcel_id):
    if not parcel_id.strip():
        return jsonify({"error": "Invalid parcel ID"}), 400

    response = table.get_item(Key={"parcel_id": parcel_id})
    parcel = response.get("Item")

    if not parcel:
        return jsonify({"error": "Parcel not found"}), 404

    return jsonify(convert_decimals(parcel)), 200


@app.route("/api/parcels", methods=["GET"])
@require_auth(allowed_roles=["admin"])
def list_parcels():
    status_filter = request.args.get("status")

    if status_filter:
        if status_filter not in VALID_STATUSES:
            return jsonify({"error": "Invalid status filter"}), 400

        response = table.query(
            IndexName="status-index",
            KeyConditionExpression=Key("status").eq(status_filter)
        )
        items = response.get("Items", [])
        return jsonify(convert_decimals(items)), 200

    response = table.scan()
    items = response.get("Items", [])
    return jsonify(convert_decimals(items)), 200


@app.route("/api/parcels/<parcel_id>/status", methods=["PUT"])
@require_auth(allowed_roles=["driver"])
def update_status(parcel_id):
    response = table.get_item(Key={"parcel_id": parcel_id})
    parcel = response.get("Item")

    if not parcel:
        return jsonify({"error": "Parcel not found"}), 404

    if parcel["status"] == "cancelled":
        return jsonify({"error": "Cannot update a cancelled parcel"}), 409

    data = request.get_json(silent=True)

    if not data or "status" not in data:
        return jsonify({"error": "Missing status field"}), 400

    new_status = data["status"]

    if not isinstance(new_status, str):
        return jsonify({"error": "Invalid data type for status"}), 400

    if new_status not in STATUS_UPDATE_ALLOWED:
        return jsonify({"error": "Invalid status value"}), 400

    if parcel["status"] == "delivered":
        return jsonify({"error": "Delivered parcels cannot be updated again"}), 409

    now = datetime.utcnow().isoformat()

    parcel["status"] = new_status
    parcel["status_history"].append({
        "status": new_status,
        "timestamp": now
    })

    table.put_item(Item=parcel)

    send_status_change_to_sqs(parcel, new_status)

    return jsonify({
        "message": "Parcel status updated successfully",
        "parcel": convert_decimals(parcel)
    }), 200


@app.route("/api/parcels/<parcel_id>", methods=["DELETE"])
@require_auth(allowed_roles=["admin"])
def cancel_parcel(parcel_id):
    response = table.get_item(Key={"parcel_id": parcel_id})
    parcel = response.get("Item")

    if not parcel:
        return jsonify({"error": "Parcel not found"}), 404

    if parcel["status"] != "created":
        return jsonify({"error": "Only parcels not yet picked up can be cancelled"}), 409

    now = datetime.utcnow().isoformat()

    parcel["status"] = "cancelled"
    parcel["status_history"].append({
        "status": "cancelled",
        "timestamp": now
    })

    table.put_item(Item=parcel)

    return jsonify({
        "message": "Parcel cancelled successfully",
        "parcel": convert_decimals(parcel)
    }), 200


@app.route("/api/parcels/<parcel_id>/photo", methods=["POST"])
@require_auth(allowed_roles=["driver"])
def upload_photo(parcel_id):
    response = table.get_item(Key={"parcel_id": parcel_id})
    parcel = response.get("Item")

    if not parcel:
        return jsonify({"error": "Parcel not found"}), 404

    if "photo" not in request.files:
        return jsonify({"error": "No photo file uploaded"}), 400

    file = request.files["photo"]

    if not file or file.filename == "":
        return jsonify({"error": "Empty filename"}), 400

    allowed_extensions = {"jpg", "jpeg", "png"}
    filename = secure_filename(file.filename)

    if "." not in filename:
        return jsonify({"error": "File must have an extension"}), 400

    ext = filename.rsplit(".", 1)[1].lower()
    if ext not in allowed_extensions:
        return jsonify({"error": "Only jpg, jpeg, and png files are allowed"}), 400

    s3_key = f"parcel-photos/{parcel_id}/{filename}"

    try:
        s3.upload_fileobj(
            file,
            BUCKET_NAME,
            s3_key,
            ExtraArgs={
                "ContentType": file.content_type
            }
        )
    except ClientError:
        app.logger.exception("S3 upload failed")
        return jsonify({"error": "Failed to upload photo"}), 500

    photo_url = f"s3://{BUCKET_NAME}/{s3_key}"

    parcel["photo_url"] = photo_url
    parcel["status_history"].append({
        "status": "photo_uploaded",
        "timestamp": datetime.utcnow().isoformat()
    })

    table.put_item(Item=parcel)

    return jsonify({
        "message": "Photo uploaded successfully",
        "photo_url": photo_url,
        "parcel": convert_decimals(parcel)
    }), 200


if __name__ == "__main__":
