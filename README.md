# SmartParcel — NET_214 Project

**Student Name:** Mardan Ali  
**Student ID:** 20210001377  
**Email:** 20210001377@students.cud.ac.ae  

## Project Overview

SmartParcel is a cloud-native parcel tracking system built using **Flask** and deployed on **AWS**.  
It supports parcel creation, tracking, status updates, photo uploads, and asynchronous customer notifications.

The project demonstrates:
- REST API design
- cloud deployment on AWS
- secure storage and processing
- asynchronous event-driven architecture
- concurrency and scalability

---

## Features

- Create parcel records
- Retrieve parcel by ID
- List all parcels
- Filter parcels by status
- Update parcel status
- Upload parcel photo
- Role-based API key authentication
- DynamoDB parcel storage
- S3 parcel photo storage
- SQS queue for event handling
- Lambda consumer
- SNS email notifications
- Health check endpoint
- Concurrent request handling using Gunicorn

---

## API Endpoints

- `GET /health`
- `POST /api/parcels`
- `GET /api/parcels/<parcel_id>`
- `GET /api/parcels`
- `PUT /api/parcels/<parcel_id>/status`
- `DELETE /api/parcels/<parcel_id>`
- `POST /api/parcels/<parcel_id>/photo`

---

## Authentication

The API uses simple API key authentication with role-based access:

- `admin-key-123`
- `driver-key-123`
- `customer-key-123`

---

## AWS Services Used

- **EC2** — hosts Flask API
- **S3** — stores uploaded parcel photos
- **DynamoDB** — stores parcel data
- **SQS** — queues parcel status events
- **Lambda** — processes queue messages
- **SNS** — sends email notifications
- **CloudWatch** — logs and monitoring

---

## Architecture Overview

Client → Flask API (EC2 + Gunicorn) → DynamoDB / S3  
                             ↓  
                           SQS  
                             ↓  
                          Lambda  
                             ↓  
                           SNS (Email)

---

## Concurrency

The application uses Gunicorn with multiple workers and threads:

```bash
gunicorn --bind 0.0.0.0:8080 --workers 4 --threads 2 app:app

A load test with 20 concurrent requests confirmed stable responses.

Security
API key authentication
Role-based access control
Input validation
Basic injection protection
File size limits
S3 encryption enabled
Security Group rules configured for controlled access