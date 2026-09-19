# S3 Application Project Overview

## Purpose

This project is a Spring Boot REST API for storing and retrieving files from an Amazon S3 bucket. Files are saved beneath a configurable S3 key prefix (called a folder in the application configuration).

## What has been implemented

- A Spring Boot application entry point in `S3Application`.
- REST support and multipart upload handling through Spring Web.
- Amazon S3 integration using AWS SDK for Java v2.
- An `S3Client` Spring bean configured by the active environment profile.
- An `S3Service` that builds S3 object keys and performs upload/download requests.
- An `S3Controller` exposing upload and download APIs under `/api/s3`.
- A basic Spring Boot context-loading test.

## Application flow

```text
Client request
    -> S3Controller
    -> S3Service
    -> AWS SDK S3Client
    -> Amazon S3 bucket
```

For every file, the service creates an object key in this format:

```text
<AWS_S3_FOLDER>/<file-name>
```

The folder value is trimmed and any leading or trailing `/` characters are removed. With the default folder, `report.pdf` is stored as `BasicFiles/report.pdf`.

## Available API endpoints

| Method | Endpoint | Description |
| --- | --- | --- |
| `POST` | `/api/s3/upload` | Uploads a multipart file. The form field must be named `file`. |
| `GET` | `/api/s3/download/{fileName}` | Downloads the requested file as an attachment. |

Example upload:

```bash
curl -X POST http://localhost:8080/api/s3/upload \
  -F "file=@/path/to/report.pdf"
```

Example download:

```bash
curl -OJ http://localhost:8080/api/s3/download/report.pdf
```

## Configuration

The application reads its settings from environment variables through `application.properties`:

| Environment variable | Purpose |
| --- | --- |
| `ENVIRONMENT` | Activates the Spring profile (`local` or `dev`). |
| `AWS_S3_BUCKET` | Target S3 bucket name. |
| `AWS_REGION` | AWS region containing the bucket. |
| `AWS_S3_FOLDER` | Optional object-key prefix; defaults to `BasicFiles`. |
| `AWS_ACCESS_KEY_ID` | Access key used by the `local` profile. |
| `AWS_SECRET_ACCESS_KEY` | Secret key used by the `local` profile. |

### Credential profiles

- `local`: creates the S3 client with `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.
- `dev`: uses the AWS SDK default credential provider chain, such as an IAM role, local AWS profile, or environment credentials.

## Project structure

```text
src/
├── main/
│   ├── java/com/example/S3Application/
│   │   ├── S3Application.java       # Application bootstrap
│   │   ├── config/S3Config.java     # S3 client configuration
│   │   ├── controller/S3Controller.java # HTTP endpoints
│   │   └── service/S3Service.java   # S3 operations
│   └── resources/application.properties # Environment-backed settings
└── test/java/com/example/S3Application/S3ApplicationTests.java
```

## Required AWS access

The credentials or IAM role need permission to upload and download objects in the configured bucket and prefix, including `s3:PutObject` and `s3:GetObject`.

## Running the JAR on EC2 with `nohup`

On the EC2 instance, build the executable JAR (or copy a previously built JAR to the instance):

```bash
./mvnw clean package -DskipTests
```

For an EC2 IAM role or another default AWS credential source, pass configuration as JVM system properties when starting the JAR. `BUCKETNAME` alone will not work with the current code; use `AWS_S3_BUCKET`, which is the name expected by `application.properties`.

```bash
nohup java \
  -DENVIRONMENT=dev \
  -DAWS_S3_BUCKET=<your-bucket-name> \
  -DAWS_REGION=<your-aws-region> \
  -DAWS_S3_FOLDER=BasicFiles \
  -jar target/S3Application-0.0.1-SNAPSHOT.jar \
  > s3-application.log 2>&1 &
```

Example:

```bash
nohup java \
  -DENVIRONMENT=dev \
  -DAWS_S3_BUCKET=my-s3-bucket \
  -DAWS_REGION=ap-south-1 \
  -jar target/S3Application-0.0.1-SNAPSHOT.jar \
  > s3-application.log 2>&1 &
```

Useful process and log commands:

```bash
tail -f s3-application.log              # Follow application logs
ps -ef | grep S3Application             # Find the running process
kill <process-id>                        # Stop the application
```

If the `local` profile is used instead, replace `-DENVIRONMENT=dev` with `-DENVIRONMENT=local` and add `-DAWS_ACCESS_KEY_ID=<access-key>` and `-DAWS_SECRET_ACCESS_KEY=<secret-key>`. On EC2, an attached IAM role and the `dev` profile are preferred so long-lived access keys are not stored on the server.

## Security note

Keep AWS credentials out of source control. Prefer environment variables for local work and IAM roles or the default AWS credential chain for deployed environments.
