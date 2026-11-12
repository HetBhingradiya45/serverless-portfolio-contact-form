# Serverless Portfolio Contact Form

A serverless contact form for a portfolio website, built on AWS using **S3, API Gateway, Lambda and DynamoDB**. Visitors submit their name, email and message; the request is validated server-side and stored in DynamoDB, with no servers to manage.

![AWS](https://img.shields.io/badge/AWS-Serverless-orange)
![Python](https://img.shields.io/badge/Python-3.x-blue)
![Region](https://img.shields.io/badge/Region-ap--south--1-green)

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Tech Stack](#tech-stack)
4. [Features](#features)
5. [Project Structure](#project-structure)
6. [AWS Resources](#aws-resources)
7. [Setup Guide](#setup-guide)
8. [API Reference](#api-reference)
9. [Lambda Function](#lambda-function)
10. [IAM Policy](#iam-policy)
11. [Data Model](#data-model)
12. [Testing](#testing)
13. [Screenshots](#screenshots)
14. [Security Notes](#security-notes)
15. [Possible Improvements](#possible-improvements)
16. [Author](#author)

---

## Overview

Static websites cannot process form submissions on their own. This project adds a backend to a static portfolio page without running any server:

- The **frontend** (`index.html`) is hosted as a static file in **Amazon S3**.
- The form sends a `POST` request to an **API Gateway** endpoint.
- API Gateway triggers an **AWS Lambda** function (Python) that validates the input.
- Valid submissions are saved in an **Amazon DynamoDB** table.

---

## Architecture

```
┌───────────┐     ┌───────────────┐     ┌──────────────┐     ┌────────────────┐
│  Browser  │ --> │ API Gateway   │ --> │   Lambda     │ --> │   DynamoDB     │
│ (S3 site) │     │ POST /contact │     │ (Python)     │     │ PortfolioSubmi │
└───────────┘     └───────────────┘     └──────────────┘     │ ssions         │
      ^                                        |             └────────────────┘
      └──────────── JSON response (success / error) ──────────┘
```

**Request flow**

1. User fills in the form on the page hosted in S3.
2. JavaScript sends `POST /contact` with a JSON body.
3. API Gateway (HTTP API) invokes the Lambda function.
4. Lambda validates name, email and message.
5. If valid, Lambda writes an item to DynamoDB with a unique `SubmissionID`.
6. Lambda returns a JSON response, and the page shows a success or error message.

---

## Tech Stack

| Layer | Service / Tool |
|-------|----------------|
| Frontend | HTML, CSS, JavaScript |
| Hosting | Amazon S3 |
| API | Amazon API Gateway (HTTP API) |
| Compute | AWS Lambda (Python) |
| Database | Amazon DynamoDB (on-demand capacity) |
| Security | AWS IAM (least-privilege policy) |
| Region | Asia Pacific (Mumbai) `ap-south-1` |

---

## Features

- Fully serverless, pay-per-use, no servers to maintain
- Server-side validation (required fields, email format, length limits)
- Unique ID (UUID) and UTC timestamp for every submission
- Least-privilege IAM: Lambda can only perform `dynamodb:PutItem` on one table
- CORS configured so the static site can call the API
- Clear success and error messages on the form (for example "Message is required.")
- DynamoDB on-demand mode, so no capacity planning

---

## Project Structure

```
serverless-portfolio-contact-form/
├── frontend/
│   └── index.html            # Portfolio contact form (uploaded to S3)
├── lambda/
│   └── lambda_function.py    # Lambda handler (validation + DynamoDB write)
├── iam/
│   └── dynamodb-putitem-policy.json
├── screenshots/
│   ├── 01-dynamodb-table.png
│   ├── 02-lambda-function.png
│   ├── 03-lambda-code.png
│   ├── 04-iam-policy.png
│   ├── 05-api-gateway-route.png
│   ├── 06-form-success.png
│   ├── 07-dynamodb-items.png
│   ├── 08-form-validation.png
│   ├── 09-s3-bucket.png
│   └── 10-s3-upload.png
└── README.md
```

---

## AWS Resources

| Resource | Name / Value |
|----------|--------------|
| S3 bucket | `my-portfolio-lc72` |
| S3 file | `portfolio/index45.html` |
| DynamoDB table | `PortfolioSubmissions` |
| Partition key | `SubmissionID` (String) |
| Capacity mode | On-demand |
| Lambda function | `ProcessPortfolioContact` |
| Lambda environment variable | `TABLE_NAME = PortfolioSubmissions` |
| API Gateway | `PortfolioAPI` (HTTP API) |
| Route | `POST /contact` |
| Region | `ap-south-1` (Mumbai) |

---

## Setup Guide

### 1. Create the DynamoDB table

1. Open **DynamoDB → Tables → Create table**.
2. Table name: `PortfolioSubmissions`
3. Partition key: `SubmissionID` (String)
4. Capacity mode: **On-demand**
5. Create the table and wait for the status to become **Active**.

### 2. Create the Lambda function

1. Open **Lambda → Create function → Author from scratch**.
2. Function name: `ProcessPortfolioContact`
3. Runtime: **Python 3.x**
4. Paste the code from [`lambda/lambda_function.py`](#lambda-function) and click **Deploy**.
5. Go to **Configuration → Environment variables** and add:
   - `TABLE_NAME` = `PortfolioSubmissions`

### 3. Grant least-privilege IAM permission

1. Open the Lambda's execution role (**Configuration → Permissions**).
2. Choose **Add permissions → Create inline policy**.
3. Use the JSON from the [IAM Policy](#iam-policy) section, replacing the account ID placeholder.
4. Save the policy.

### 4. Create the API in API Gateway

1. Open **API Gateway → Create API → HTTP API**, name it `PortfolioAPI`.
2. Add an integration pointing to the `ProcessPortfolioContact` Lambda function.
3. Create the route: **`POST /contact`**.
4. Configure **CORS**:
   - Allowed origins: your S3 website URL (or `*` while testing)
   - Allowed methods: `POST, OPTIONS`
   - Allowed headers: `content-type`
5. Deploy to a stage (for example `$default`) and copy the **Invoke URL**.

### 5. Connect the frontend

In `index.html`, set the API endpoint used by the form's `fetch` call:

```js
const API_URL = "https://<api-id>.execute-api.ap-south-1.amazonaws.com/contact";
```

### 6. Host the frontend on S3

1. Open **S3 → Create bucket** (example: `my-portfolio-lc72`, region `ap-south-1`).
2. Upload `index.html` (inside a `portfolio/` folder).
3. To serve it publicly, enable static website hosting or put CloudFront in front of the bucket (recommended, see [Security Notes](#security-notes)).

---

## API Reference

### `POST /contact`

**Request headers**

```
Content-Type: application/json
```

**Request body**

```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "message": "Hello, I would like to connect."
}
```

**Validation rules**

| Field | Rule |
|-------|------|
| `name` | Required, max 100 characters |
| `email` | Required, valid email format, max 254 characters |
| `message` | Required, max 2000 characters |

**Success response: `200 OK`**

```json
{
  "message": "Thank you! Your message was received."
}
```

**Error response: `400 Bad Request`**

```json
{
  "error": "Message is required."
}
```

**Server error: `500 Internal Server Error`**

```json
{
  "error": "Something went wrong. Please try again later."
}
```

---

## Lambda Function

Reference implementation (`lambda/lambda_function.py`). Replace with your deployed code if it differs.

```python
import json
import os
import re
import uuid
from datetime import datetime, timezone

import boto3
from botocore.exceptions import ClientError

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table(os.environ["TABLE_NAME"])

EMAIL_RE = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")
MAX_NAME, MAX_EMAIL, MAX_MESSAGE = 100, 254, 2000

HEADERS = {
    "Content-Type": "application/json",
    "Access-Control-Allow-Origin": "*",  # restrict to your site in production
}


def response(status, body):
    return {"statusCode": status, "headers": HEADERS, "body": json.dumps(body)}


def lambda_handler(event, context):
    try:
        data = json.loads(event.get("body") or "{}")
    except json.JSONDecodeError:
        return response(400, {"error": "Invalid JSON body."})

    name = str(data.get("name", "")).strip()
    email = str(data.get("email", "")).strip()
    message = str(data.get("message", "")).strip()

    if not name:
        return response(400, {"error": "Name is required."})
    if len(name) > MAX_NAME:
        return response(400, {"error": "Name is too long."})
    if not email or not EMAIL_RE.match(email) or len(email) > MAX_EMAIL:
        return response(400, {"error": "A valid email is required."})
    if not message:
        return response(400, {"error": "Message is required."})
    if len(message) > MAX_MESSAGE:
        return response(400, {"error": "Message is too long."})

    item = {
        "SubmissionID": str(uuid.uuid4()),
        "createdAt": datetime.now(timezone.utc).isoformat(),
        "name": name,
        "email": email,
        "message": message,
    }

    try:
        table.put_item(Item=item)
    except ClientError as e:
        print(f"DynamoDB error: {e}")
        return response(500, {"error": "Something went wrong. Please try again later."})

    return response(200, {"message": "Thank you! Your message was received."})
```

---

## IAM Policy

The Lambda execution role only needs permission to write one item type to one table (`iam/dynamodb-putitem-policy.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "dynamodb:PutItem",
      "Resource": "arn:aws:dynamodb:ap-south-1:<ACCOUNT_ID>:table/PortfolioSubmissions"
    }
  ]
}
```

Replace `<ACCOUNT_ID>` with your own AWS account ID. Never commit real account IDs, keys or secrets to a public repository.

---

## Data Model

Each submission is stored as one item in `PortfolioSubmissions`:

| Attribute | Type | Description |
|-----------|------|-------------|
| `SubmissionID` | String (partition key) | Unique UUID |
| `createdAt` | String | UTC timestamp in ISO 8601 format |
| `name` | String | Sender's name |
| `email` | String | Sender's email |
| `message` | String | Message text |

---

## Testing

**1. From the website**

Open the form, fill in all fields and click **Send message**. You should see:
`Thank you! Your message was received.`

**2. Validation check**

Leave the message empty and submit. You should see:
`Message is required.`

**3. From the command line**

```bash
curl -X POST "https://<api-id>.execute-api.ap-south-1.amazonaws.com/contact" \
  -H "Content-Type: application/json" \
  -d '{"name":"Test","email":"test@example.com","message":"Hello"}'
```

**4. Verify in DynamoDB**

Open **DynamoDB → Explore items → PortfolioSubmissions** and confirm the new item appears.

---

## Screenshots

### 1. DynamoDB table created
![DynamoDB table](screenshots/01-dynamodb-table.png)

### 2. Lambda function created
![Lambda function](screenshots/02-lambda-function.png)

### 3. Lambda code (validation + DynamoDB write)
![Lambda code](screenshots/03-lambda-code.png)

### 4. IAM policy (least privilege)
![IAM policy](screenshots/04-iam-policy.png)

### 5. API Gateway route
![API Gateway route](screenshots/05-api-gateway-route.png)

### 6. Form success state
![Form success](screenshots/06-form-success.png)

### 7. Data stored in DynamoDB
![DynamoDB items](screenshots/07-dynamodb-items.png)

### 8. Form validation
![Form validation](screenshots/08-form-validation.png)

### 9. S3 bucket created
![S3 bucket](screenshots/09-s3-bucket.png)

### 10. Frontend uploaded to S3
![S3 upload](screenshots/10-s3-upload.png)

---

## Security Notes

- **Least privilege:** the Lambda role is limited to `dynamodb:PutItem` on a single table.
- **Server-side validation:** never trust client-side checks alone; Lambda validates every field.
- **CORS:** restrict `Access-Control-Allow-Origin` to your real site domain in production instead of `*`.
- **Public S3 access:** prefer CloudFront with Origin Access Control instead of making the bucket public.
- **Rate limiting:** enable API Gateway throttling to reduce spam and abuse.
- **Privacy:** screenshots may show account IDs and names; blur them before publishing.

---

## Possible Improvements

- Send an email notification via Amazon SES or SNS for each new submission
- Add CAPTCHA or honeypot field to block bots
- Serve the site through CloudFront with HTTPS and a custom domain
- Add API Gateway throttling and AWS WAF
- Manage the infrastructure as code with AWS SAM, CloudFormation or Terraform
- Add CloudWatch alarms for Lambda errors
- Add a CI/CD pipeline using GitHub Actions

---

## Author

**Het Bhingradiya

---

## License

This project is open source under the [MIT License](LICENSE).
