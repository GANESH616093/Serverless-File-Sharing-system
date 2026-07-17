# Depot — Serverless File Sharing System

A complete, deployable serverless file-sharing system on AWS: upload a file from
a browser, get a time-limited download link, share it. No server ever holds
the file bytes.

## Architecture

```
 Browser (index.html)
   │
   │ 1. POST /files/upload-url  ──────────────►  Lambda: generate_upload_url
   │                                                  │
   │                                                  ├─ writes metadata → DynamoDB
   │                                                  └─ returns presigned PUT URL
   │
   │ 2. PUT file bytes directly to S3  ───────────────────────────► S3 (FilesBucket)
   │
   │ 3. GET /files  ───────────────────────────►  Lambda: list_files → DynamoDB
   │
   │ 4. GET /files/{id}/download-url  ─────────►  Lambda: generate_download_url
   │                                                  └─ returns presigned GET URL
   │
   │ 5. GET the presigned URL directly  ──────────────────────────► S3 (FilesBucket)
   │
   │ 6. DELETE /files/{id}  ───────────────────►  Lambda: delete_file
   │                                                  └─ removes from S3 + DynamoDB
   ▼
 API Gateway (HTTP API) fronts all four Lambdas
```

Key design point: **the file itself never passes through Lambda or API Gateway.**
Both upload and download happen as direct browser↔S3 transfers using
presigned URLs, so there's no Lambda payload-size limit, no API Gateway
timeout risk on large files, and near-zero compute cost regardless of
transfer size. Lambda's only job is issuing short-lived, single-purpose URLs
and tracking metadata.

Components:
- **S3 `FilesBucket`** — stores uploaded files. Private by default; a
  lifecycle rule auto-deletes objects after `FileRetentionDays` (default 7).
- **DynamoDB `FilesTable`** — one row per file: name, size, upload time,
  download count, and a TTL attribute so expired rows disappear automatically.
- **4 Lambda functions** — `generate_upload_url`, `generate_download_url`,
  `list_files`, `delete_file`.
- **API Gateway (HTTP API)** — routes `/files*` to the functions above, CORS-enabled.
- **S3 `WebsiteBucket`** — hosts the static frontend (`index.html`) as a public website.

## What's in this project

```
template.yaml              SAM template — defines every AWS resource above
src/lambda/
  generate_upload_url.py
  generate_download_url.py
  list_files.py
  delete_file.py
frontend/
  index.html                The website — drop-zone upload + live manifest of active links
README.md                   This file
```

## Deploying it (get your real hosted link)

You'll need an AWS account and the AWS SAM CLI installed
(`brew install aws-sam-cli` or see AWS's install docs).

```bash
# 1. From the project root, build
sam build

# 2. Deploy — this walks you through settings the first time
sam deploy --guided
```

Accept the defaults or set your own `ProjectName` / `LinkExpiryMinutes` /
`FileRetentionDays`. When it finishes, note the two outputs:

```
ApiUrl       → https://xxxxxxxxxx.execute-api.<region>.amazonaws.com
WebsiteUrl   → http://<bucket-name>.s3-website-<region>.amazonaws.com
```

Then wire the frontend to your API:

```bash
# Open frontend/index.html and set:
const API_BASE_URL = "https://xxxxxxxxxx.execute-api.<region>.amazonaws.com";

# Upload it to the website bucket SAM created:
aws s3 cp frontend/index.html s3://<website-bucket-name>/index.html
```

Open the `WebsiteUrl` from the outputs — that's your presentable, hosted link.

### Locking it down further (recommended before sharing widely)
- Redeploy with `AllowedOrigin` set to your actual `WebsiteUrl` instead of `*`.
- Put the website bucket behind **CloudFront + Origin Access Control** instead
  of public S3 website hosting, and get free HTTPS + a CDN in the process.
- Add **Cognito** or an API key on the HTTP API if uploads/downloads shouldn't
  be open to anyone with the link.
- Add an S3 event notification → Lambda to flip each DynamoDB row's `status`
  from `pending` to `uploaded` once the object actually lands, so `list_files`
  can filter out abandoned uploads.

## Cost

Everything here is pay-per-use: S3 storage + requests, DynamoDB on-demand,
Lambda invocations, API Gateway requests. For light/demo usage this
comfortably sits in AWS's free tier; there's no idle server cost since
nothing runs when no one's using it.


