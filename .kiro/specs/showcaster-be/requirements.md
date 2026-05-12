# Requirements Document

## Introduction

Showcaster Backend (showcaster-be) is a Go-based REST API service that powers the Showcaster AI affiliate video generation SaaS. It accepts video generation jobs from the frontend, processes them asynchronously using Replicate's Wan 2.1 model to produce a 4-part video sequence (Hook → Problem → Solution → Closure), stores results in Cloudinary, and persists job state in SQLite. The service also handles user authentication via JWT and provides job history management with a 5-day TTL.

## Glossary

- **API_Server**: The Fiber v2 HTTP server that handles all incoming REST requests.
- **Auth_Service**: The component responsible for user registration, login, OTP verification, and JWT issuance/validation.
- **Job_Service**: The component that creates, reads, updates, and deletes video generation jobs.
- **Job_Worker**: The background goroutine that reads from the job channel and processes video generation asynchronously.
- **Job_Queue**: The Go channel used to pass accepted jobs from the HTTP handler to the Job_Worker.
- **Replicate_Client**: The component that calls the Replicate API to generate video clips using the Wan 2.1 model.
- **Cloudinary_Client**: The component that uploads images and generated video clips to Cloudinary and returns public URLs.
- **DB**: The SQLite database accessed via GORM that persists users, jobs, and job steps.
- **Job**: A record representing a single video generation request, containing metadata, status, progress, and references to its 4 pipeline steps.
- **Step**: One of the four pipeline segments of a Job: Hook, Problem, Solution, or Closure. Each Step has its own status and output video URL.
- **JWT**: A JSON Web Token issued by the Auth_Service upon successful login or OTP verification, used to authenticate subsequent requests.
- **OTP**: A one-time password sent to the user's email for account verification during registration. OTPs are 6-digit numeric codes.
- **TTL**: Time-to-live; the 5-day period after which a completed Job and its associated Steps are automatically deleted from the DB.
- **Progress**: An integer from 0 to 100 representing job completion, calculated as `(number of completed Steps / 4) * 100`.
- **Orientation**: The aspect ratio of the generated video: `portrait` (9:16), `landscape` (16:9), or `square` (1:1).
- **Resolution**: The output video quality: `720p`, `1080p`, or `4k`.
- **Pipeline**: The ordered sequence of 4 Steps (Hook → Problem → Solution → Closure) that constitute a complete video generation Job.

---

## Requirements

### Requirement 1: User Registration

**User Story:** As a new user, I want to register an account with my email and password, so that I can access the Showcaster platform.

#### Acceptance Criteria

1. WHEN a POST request is received at `/api/auth/register` with a valid email (RFC 5322 format, max 254 characters), a `fullName` (non-empty string, max 100 characters), and a password meeting complexity requirements (minimum 8 characters, maximum 72 characters, containing at least one uppercase letter, one lowercase letter, and one digit), THE Auth_Service SHALL create a new user record in the DB with the password stored as a bcrypt hash and return HTTP 201 with a success message.
2. WHEN a POST request is received at `/api/auth/register` with an email that already exists in the DB, THE Auth_Service SHALL skip user record creation entirely and return HTTP 409 with an error message indicating the email is already registered, without sending any success response.
3. WHEN a POST request is received at `/api/auth/register` with a missing or malformed email (failing RFC 5322 format or exceeding 254 characters), THE Auth_Service SHALL return HTTP 400 with an error message identifying the `email` field as invalid.
4. WHEN a POST request is received at `/api/auth/register` with a password shorter than 8 characters, longer than 72 characters, or missing at least one uppercase letter, one lowercase letter, or one digit, THE Auth_Service SHALL return HTTP 400 with an error message identifying the `password` field as invalid.
5. WHEN a POST request is received at `/api/auth/register` with a missing or empty `fullName` field, THE Auth_Service SHALL return HTTP 400 with an error message identifying the `fullName` field as invalid.
6. WHEN a new user record is successfully created, THE Auth_Service SHALL send a 6-digit numeric OTP to the registered email address and return HTTP 201 with a success message.
7. IF the OTP email send fails after the user record is created, THEN THE Auth_Service SHALL still return HTTP 201 with a success message and SHALL expose a resend-OTP mechanism so the user can request a new OTP.

---

### Requirement 2: User Login

**User Story:** As a registered user, I want to log in with my email and password, so that I can receive a JWT to authenticate my API requests.

#### Acceptance Criteria

1. WHEN a POST request is received at `/api/auth/login` with a request body containing an email address in RFC 5322 format and a matching password for that account, THE Auth_Service SHALL return HTTP 200 with a signed JWT in the response body, where the JWT contains the authenticated user's identity and an expiry of exactly 86400 seconds from the time of issuance.
2. WHEN a POST request is received at `/api/auth/login` with a valid email and incorrect password, THE Auth_Service SHALL return HTTP 401 with an error message indicating invalid credentials and SHALL NOT return HTTP 200.
3. WHEN a POST request is received at `/api/auth/login` with an email address that does not exist in the DB, THE Auth_Service SHALL return HTTP 401 with the same error message as criterion 2, indistinguishable from an incorrect-password response.
4. THE Auth_Service SHALL sign all issued JWTs using a secret key and SHALL set the expiry claim (`exp`) to exactly 86400 seconds after the time of issuance (`iat`).
5. IF the DB is unavailable during a login attempt, THEN THE Auth_Service SHALL return HTTP 503 with an error message indicating that the service is temporarily unavailable.
6. IF a POST request is received at `/api/auth/login` with a missing or malformed request body (absent `email` field, absent `password` field, or non-string values for either field), THEN THE Auth_Service SHALL return HTTP 400 with an error message indicating which field is invalid or missing.

---

### Requirement 3: OTP Verification

**User Story:** As a newly registered user, I want to verify my email address via OTP, so that my account is activated and I can use the platform.

#### Acceptance Criteria

1. WHEN a POST request is received at `/api/auth/verify-otp` with a valid email and a correct, non-expired 6-digit numeric OTP, THE Auth_Service SHALL mark the user's account as verified in the DB and return HTTP 200 with a signed JWT valid for 86400 seconds.
2. WHEN a POST request is received at `/api/auth/verify-otp` with a correct OTP that was issued more than 10 minutes before the request time, THE Auth_Service SHALL return HTTP 400 with an error message indicating the OTP has expired.
3. WHEN a POST request is received at `/api/auth/verify-otp` with an incorrect 6-digit numeric OTP, THE Auth_Service SHALL return HTTP 400 with an error message. IF the same OTP has been submitted incorrectly 5 or more times, THEN THE Auth_Service SHALL invalidate the OTP and return HTTP 400 with an error message indicating the OTP is no longer valid.
4. WHEN a POST request is received at `/api/auth/verify-otp` with an email that does not exist in the DB, THE Auth_Service SHALL return HTTP 404 with an error message.
5. WHEN a POST request is received at `/api/auth/verify-otp` for a user whose account is already verified, THE Auth_Service SHALL return HTTP 409 with an error message indicating the account is already verified.

---

### Requirement 4: JWT Authentication Middleware

**User Story:** As the system, I want all protected endpoints to require a valid JWT, so that only authenticated users can access job and upload features.

#### Acceptance Criteria

1. WHEN a request is received on a protected endpoint without an `Authorization` header, or with an `Authorization` header that is not in the format `Bearer <token>`, THE API_Server SHALL return HTTP 401 with an error message.
2. WHEN a request is received on a protected endpoint with an `Authorization: Bearer <token>` header where the token has an invalid signature, is expired, or was not signed with the configured secret key, THE API_Server SHALL return HTTP 401 with an error message.
3. WHEN a request is received on a protected endpoint with a valid JWT, THE API_Server SHALL extract the user ID from the token claims and make it available to downstream handlers, which SHALL use it for ownership and access control decisions.
4. THE API_Server SHALL apply JWT authentication middleware to all `/api/jobs/*` and `/api/upload/*` routes.

---

### Requirement 5: Image Upload

**User Story:** As a user, I want to upload model and product images before generating a video, so that the AI model has the visual inputs it needs.

#### Acceptance Criteria

1. WHEN a POST request is received at `/api/upload/image` with a valid image file (JPEG or PNG, maximum 10 MB) in the `file` form field, THE API_Server SHALL validate the file and pass it to the Cloudinary_Client for upload.
2. WHEN the Cloudinary_Client successfully uploads the image, THE API_Server SHALL return HTTP 200 with the Cloudinary URL in the response body as `{ "url": "<cloudinary_url>" }`.
3. WHEN a POST request is received at `/api/upload/image` with a file exceeding 10 MB, THE API_Server SHALL return HTTP 413 with `{ "error": "file size exceeds the 10 MB limit" }`.
4. WHEN a POST request is received at `/api/upload/image` with a file whose MIME type is not `image/jpeg` or `image/png`, THE API_Server SHALL return HTTP 415 with `{ "error": "unsupported file type; only JPEG and PNG are accepted" }`.
5. WHEN a POST request is received at `/api/upload/image` with no `file` field in the form data, THE API_Server SHALL return HTTP 400 with `{ "error": "file field is required" }`.
6. IF the Cloudinary upload fails, THEN THE API_Server SHALL return HTTP 502 with `{ "error": "upstream image upload service failed" }`.

---

### Requirement 6: Submit Video Generation Job

**User Story:** As a user, I want to submit a video generation job and receive a job ID immediately, so that I can continue using the app while the video is being generated in the background.

#### Acceptance Criteria

1. WHEN a POST request is received at `/api/jobs/generate` with all required fields (`modelImageUrl`, `productImageUrl`, `productName`, `productCategory`, `targetAudience`, `orientation`, `resolution`) containing valid values, THE Job_Service SHALL atomically create a Job record in the DB with status `pending` and 4 Step records (Hook, Problem, Solution, Closure) each with status `pending`, and return HTTP 202 with `{ "jobId": "<uuid>" }`.
2. WHEN a POST request is received at `/api/jobs/generate` with any required field missing, THE API_Server SHALL return HTTP 400 with an error message listing each missing field by name.
3. WHEN a POST request is received at `/api/jobs/generate` with a `productCategory` value not in `[beauty, fashion, electronics, health]`, THE API_Server SHALL return HTTP 400 with an error message identifying `productCategory` as invalid and listing the accepted values.
4. WHEN a POST request is received at `/api/jobs/generate` with an `orientation` value not in `[portrait, landscape, square]`, THE API_Server SHALL return HTTP 400 with an error message identifying `orientation` as invalid and listing the accepted values.
5. WHEN a POST request is received at `/api/jobs/generate` with a `resolution` value not in `[720p, 1080p, 4k]`, THE API_Server SHALL return HTTP 400 with an error message identifying `resolution` as invalid and listing the accepted values.
6. WHEN a POST request is received at `/api/jobs/generate` with `modelImageUrl` or `productImageUrl` that is not a valid HTTPS URL, THE API_Server SHALL return HTTP 400 with an error message identifying the invalid URL field.
7. WHEN a POST request is received at `/api/jobs/generate` with a `productName` exceeding 200 characters or a `targetAudience` value not in `[man, woman, children, unisex]`, THE API_Server SHALL return HTTP 400 with an error message identifying the invalid field.
8. WHEN a Job record and its Steps are successfully created in the DB, THE Job_Service SHALL send the Job to the Job_Queue without blocking the HTTP response. IF the Job_Queue is full and the send would block, THEN THE Job_Service SHALL return HTTP 503 with an error message indicating the service is temporarily unavailable.

---

### Requirement 7: Asynchronous Job Processing

**User Story:** As the system, I want to process video generation jobs in the background, so that users are not blocked waiting for long-running AI generation tasks.

#### Acceptance Criteria

1. WHILE the API_Server is running, THE Job_Worker SHALL read Jobs from the Job_Queue within 2 seconds of them being enqueued and process them one at a time.
2. WHEN the Job_Worker begins processing a Job, THE Job_Worker SHALL update the Job status to `processing` in the DB.
3. WHEN the Job_Worker begins processing a Step, THE Job_Worker SHALL update that Step's status to `processing` in the DB.
4. WHEN the Replicate_Client successfully generates a video clip for a Step, THE Job_Worker SHALL upload the clip to Cloudinary via the Cloudinary_Client, then atomically update the Step's status to `completed`, set its `videoUrl` to the Cloudinary URL, and update the Job's `progress` field to `(completed_steps / 4) * 100` in the DB before beginning the next Step.
5. WHEN all 4 Steps of a Job are completed, THE Job_Worker SHALL update the Job status to `completed` in the DB.
6. IF the Replicate_Client returns an error for a Step, THEN THE Job_Worker SHALL retry that Step up to 3 times with a 30-second interval between attempts. IF all 3 retry attempts fail, THEN THE Job_Worker SHALL update that Step's status to `failed` and update the Job status to `failed` in the DB.
7. WHILE a Job is being processed, THE Job_Worker SHALL process the 4 Steps sequentially in the order Hook → Problem → Solution → Closure, not beginning a subsequent Step until the preceding Step has status `completed`.
8. WHEN the API_Server starts, THE Job_Worker SHALL begin consuming from the Job_Queue and SHALL continue doing so until the process exits.
9. IF the Cloudinary upload for a completed Step fails, THEN THE Job_Worker SHALL treat the failure as a Step error and apply the same retry logic defined in criterion 6.

---

### Requirement 8: Replicate Video Generation

**User Story:** As the system, I want to call the Replicate Wan 2.1 model for each pipeline step, so that AI-generated video clips are produced for each part of the affiliate video sequence.

#### Acceptance Criteria

1. WHEN the Job_Worker processes a Step, THE Replicate_Client SHALL submit a generation request to the Replicate API using the `wan-ai/wan2.1-i2v-480p` model with the Job's `modelImageUrl`, `productImageUrl`, `productName`, `productCategory`, `targetAudience`, `orientation`, `resolution`, and a text prompt derived from the Step name (Hook, Problem, Solution, or Closure) as inputs.
2. WHEN a Replicate generation request is submitted, THE Replicate_Client SHALL poll the prediction status endpoint every 5 seconds until the prediction status is `succeeded` or `failed`. WHEN the prediction status is `succeeded`, THE Replicate_Client SHALL extract the output video URL from the prediction response and return it to the Job_Worker.
3. IF a Replicate prediction status is `failed`, THEN THE Replicate_Client SHALL return a generation-failed error to the Job_Worker, distinguishable from a timeout error.
4. IF a Replicate prediction does not reach a terminal state (`succeeded` or `failed`) within 10 minutes of submission, THEN THE Replicate_Client SHALL send a cancellation request to the Replicate API for that prediction and return a timeout error to the Job_Worker, distinguishable from a generation-failed error.
5. THE Replicate_Client SHALL include the Replicate API token as a `Bearer` token in the `Authorization` header of all requests to the Replicate API.

---

### Requirement 9: Poll Job Status

**User Story:** As a user, I want to poll the status of my video generation job, so that I can display real-time progress in the frontend.

#### Acceptance Criteria

1. WHEN a GET request is received at `/api/jobs/:id` for a Job that belongs to the authenticated user, THE Job_Service SHALL return HTTP 200 with a JSON body containing: `id` (string), `status` (one of `pending`, `processing`, `completed`, `failed`), `progress` (integer 0–100), `steps` (array of objects each with `name` (string), `status` (one of `pending`, `processing`, `completed`, `failed`), and `videoUrl` (string or null)), and `createdAt` (ISO 8601 timestamp).
2. WHEN a GET request is received at `/api/jobs/:id` for a Job ID that does not exist in the DB, THE Job_Service SHALL return HTTP 404 with an error message.
3. WHEN a GET request is received at `/api/jobs/:id` for a Job that exists in the DB but belongs to a different user, THE Job_Service SHALL return HTTP 403 with an error message.
4. THE Job_Service SHALL return each Step's `videoUrl` as `null` when the Step status is `pending` or `processing`, and as the Cloudinary URL string when the Step status is `completed`.
5. IF the DB is unavailable when the request is received, THEN THE Job_Service SHALL return HTTP 503 with an error message indicating the service is temporarily unavailable.

---

### Requirement 10: List Job History

**User Story:** As a user, I want to retrieve a list of my past video generation jobs, so that I can review and manage my generation history.

#### Acceptance Criteria

1. WHEN a GET request is received at `/api/jobs` without pagination parameters, THE Job_Service SHALL return HTTP 200 with an array of up to 100 of the authenticated user's Jobs ordered by `createdAt` descending, each containing `id`, `status`, `createdAt`, and `thumbnailUrl` (null if no thumbnail exists).
2. THE Job_Service SHALL return only Jobs belonging to the authenticated user.
3. WHEN the authenticated user has no Jobs in the DB, THE Job_Service SHALL return HTTP 200 with an empty array.
4. WHERE pagination is requested via `page` (positive integer, minimum 1) and `limit` (integer 0–100 inclusive) query parameters, THE Job_Service SHALL return the corresponding subset of Jobs and include `total`, `page`, and `limit` in the response. WHEN `limit` is 0, THE Job_Service SHALL return an empty array alongside the pagination metadata.
5. IF a `page` or `limit` query parameter is present but is not a valid integer or is outside its accepted range, THEN THE Job_Service SHALL return HTTP 400 with an error message identifying the invalid parameter.

---

### Requirement 11: Delete a Job

**User Story:** As a user, I want to delete a video generation job, so that I can remove unwanted entries from my history.

#### Acceptance Criteria

1. WHEN a DELETE request is received at `/api/jobs/:id` for a Job that belongs to the authenticated user and has status `completed` or `failed`, THE Job_Service SHALL atomically delete the Job and all associated Steps from the DB in a single transaction and return HTTP 204.
2. WHEN a DELETE request is received at `/api/jobs/:id` for a Job ID that does not exist in the DB, THE Job_Service SHALL return HTTP 404 with an error message.
3. WHEN a DELETE request is received at `/api/jobs/:id` for a Job that belongs to a different user, THE Job_Service SHALL return HTTP 403 with an error message.
4. WHEN a DELETE request is received at `/api/jobs/:id` for a Job with status `processing` or `pending`, THE Job_Service SHALL return HTTP 409 with an error message indicating the Job cannot be deleted while it is in progress or queued.
5. IF the DB is unavailable during the delete operation, THEN THE Job_Service SHALL return HTTP 503 with an error message indicating the service is temporarily unavailable.

---

### Requirement 12: Job TTL Auto-Deletion

**User Story:** As the system, I want completed jobs to be automatically deleted after 5 days, so that the database does not grow unbounded and storage costs are controlled.

#### Acceptance Criteria

1. WHILE the API_Server is running, THE Job_Service SHALL run a background cleanup routine that executes once every 24 hours.
2. WHEN the cleanup routine executes, THE Job_Service SHALL delete all Jobs with status `completed` or `failed` whose `createdAt` timestamp is more than 120 hours before the cleanup execution time, along with their associated Steps.
3. WHEN the cleanup routine executes, THE Job_Service SHALL log the number of Jobs deleted, including when the count is zero.
4. IF the DB becomes unavailable mid-execution during a cleanup run, THEN THE Job_Service SHALL roll back any partial deletions for that run, log the error, and resume on the next scheduled execution without crashing.
5. IF the DB is unavailable at the start of a cleanup execution, THEN THE Job_Service SHALL log the error and retry on the next scheduled execution without crashing.

---

### Requirement 13: Health Check Endpoint

**User Story:** As an operator, I want a health check endpoint, so that I can verify the service is running and the database is reachable.

#### Acceptance Criteria

1. WHEN a GET request is received at `/health`, THE API_Server SHALL attempt a DB ping. IF the ping completes successfully within 2 seconds, THEN THE API_Server SHALL return HTTP 200 with a response body indicating the service is healthy.
2. IF the DB ping fails or does not complete within 2 seconds, THEN THE API_Server SHALL return HTTP 503 with a response body indicating the service is degraded and including the reason for the failure.
3. THE API_Server SHALL respond to GET `/health` within 3 seconds under all conditions, including when the DB is unresponsive.

---

### Requirement 14: Configuration and Environment

**User Story:** As an operator, I want all secrets and environment-specific values to be loaded from environment variables, so that the service can be deployed across different environments without code changes.

#### Acceptance Criteria

1. THE API_Server SHALL load the following configuration values from environment variables at startup: `PORT`, `JWT_SECRET`, `REPLICATE_API_TOKEN`, `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`, `DB_PATH`.
2. IF any required environment variable (`JWT_SECRET`, `REPLICATE_API_TOKEN`, `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`) is absent or set to an empty string at startup, THEN THE API_Server SHALL log an error message that names the missing variable and exit with a non-zero status code.
3. THE API_Server SHALL default `PORT` to `8080` if the `PORT` environment variable is absent or set to an empty string.
4. THE API_Server SHALL default `DB_PATH` to `./showcaster.db` if the `DB_PATH` environment variable is absent or set to an empty string.
5. IF the `PORT` environment variable is set to a value that is not a valid TCP port number (integer in range 1–65535), THEN THE API_Server SHALL log an error message identifying the invalid value and exit with a non-zero status code.
