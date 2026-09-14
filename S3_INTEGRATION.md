# AWS S3 File Upload and Download Integration

This project is a Spring Boot REST API that connects to an Amazon S3 bucket and stores files in a configured folder (S3 key prefix). It provides two endpoints: one to upload a file and one to download it.

## What was implemented

1. Added the Spring Web dependency so the application can expose REST APIs and accept multipart file uploads.
2. Added the AWS SDK for Java v2 S3 dependency (`software.amazon.awssdk:s3`).
3. Configured the S3 bucket name, AWS region, credentials, and target folder in `application.properties`.
4. Created an `S3Client` Spring bean in `S3Config` using the configured AWS region and credentials.
5. Created `S3Service` to contain the S3 upload and download operations.
6. Created `S3Controller` with REST endpoints under `/api/s3`.

## S3 configuration

The application reads these properties from `src/main/resources/application.properties`:

```properties
cloud.aws.s3.bucket=<your-bucket-name>
cloud.aws.region.static=us-east-1
cloud.aws.credentials.access-key=<your-access-key>
cloud.aws.credentials.secret-key=<your-secret-key>
cloud.aws.s3.folder=BasicFiles
```

`S3Config` creates an AWS SDK `S3Client` with `AwsBasicCredentials`, a static credentials provider, and the configured AWS region. Spring injects this client into `S3Service`.

The folder is not a physical directory in S3. It is included at the beginning of each object key. For example, uploading `report.pdf` with `cloud.aws.s3.folder=BasicFiles` stores the object with this key:

```text
BasicFiles/report.pdf
```

The service trims extra spaces and leading/trailing slashes from the configured folder before creating this key.

## REST endpoints

### 1. Upload a file

```http
POST /api/s3/upload
Content-Type: multipart/form-data
```

The multipart form field must be named `file`.

Example using cURL:

```bash
curl -X POST http://localhost:8080/api/s3/upload \
  -F "file=@/path/to/report.pdf"
```

Request flow:

1. `S3Controller` receives the `MultipartFile`.
2. `S3Service` builds the object key as `<folder>/<original-file-name>`.
3. The service calls `S3Client.putObject(...)` and sends the uploaded file bytes to the configured bucket.
4. The API returns a success message containing the S3 object key.

### 2. Download a file

```http
GET /api/s3/download/{fileName}
```

Example:

```bash
curl -OJ http://localhost:8080/api/s3/download/report.pdf
```

Request flow:

1. `S3Controller` receives the file name from the URL.
2. `S3Service` builds the same S3 object key, for example `BasicFiles/report.pdf`.
3. The service calls `S3Client.getObjectAsBytes(...)` to read the object from S3.
4. The endpoint returns the file as `application/octet-stream` and sets `Content-Disposition: attachment`, so the browser downloads it using the requested file name.

## Project structure

```text
src/main/java/com/example/S3Application/
├── config/S3Config.java          # Creates the configured S3Client bean
├── service/S3Service.java        # Builds keys and calls S3 upload/download APIs
└── controller/S3Controller.java  # Defines /api/s3 REST endpoints
```

## AWS permissions required

The IAM user or role used by the application needs permission to write and read objects in the chosen bucket and folder prefix. At minimum, grant `s3:PutObject` and `s3:GetObject` for:

```text
arn:aws:s3:::<your-bucket-name>/BasicFiles/*
```

## Security note

Do not commit AWS access keys or secret keys to source control. The current configuration approach uses properties for local development, but production applications should obtain credentials from environment variables, an AWS profile, or an IAM role. If real credentials have already been committed or shared, rotate them immediately in AWS IAM and replace them with secure configuration.
