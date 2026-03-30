# SmartParcel — NET_214 Project

Cloud-native parcel tracking system built with Flask and AWS.

## Features
- Parcel CRUD API
- Role-based API key authentication
- DynamoDB parcel storage
- S3 photo upload
- SQS queue for parcel status events
- Lambda consumer
- SNS email notifications
- Gunicorn multi-threaded deployment on EC2

## Endpoints
- GET /health
- POST /api/parcels
- GET /api/parcels/<parcel_id>
- GET /api/parcels
- PUT /api/parcels/<parcel_id>/status
- DELETE /api/parcels/<parcel_id>
- POST /api/parcels/<parcel_id>/photo

## Run locally
```bash
pip install -r requirements.txt
python app.py
gunicorn --bind 0.0.0.0:8080 --workers 4 --threads 2 app:app


AWS Services Used
EC2
S3
DynamoDB
SQS
Lambda
SNS
CloudWatch

## Architecture Overview

Client → Flask API (EC2 + Gunicorn) → DynamoDB / S3  
                             ↓  
                           SQS  
                             ↓  
                          Lambda  
                             ↓  
                           SNS (Email)

## Concurrency

The application uses Gunicorn with multiple workers and threads:

```bash
gunicorn --bind 0.0.0.0:8080 --workers 4 --threads 2 app:app

