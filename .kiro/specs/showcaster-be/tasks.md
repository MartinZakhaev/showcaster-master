# Implementation Plan: showcaster-be

## Overview

Implement the Showcaster backend as a Go + Fiber v2 REST API service. The implementation follows the directory structure defined in the design, building from the ground up: configuration and data models first, then service and handler layers, then the async worker and cleanup routines, and finally wiring everything together in the router and entry point.

## Tasks

- [x] 1. Initialize Go module and project structure
  - Create `apps/showcaster-be/go.mod` with module path `showcaster-be` and Go 1.22+
  - Add all required dependencies to `go.mod`: `github.com/gofiber/fiber/v2`, `github.com/gofiber/contrib/jwt`, `github.com/golang-jwt/jwt/v5`, `gorm.io/gorm`, `gorm.io/driver/sqlite`, `github.com/google/uuid`, `golang.org/x/crypto`, `github.com/go-playground/validator/v10`, `github.com/cloudinary/cloudinary-go/v2`, `github.com/joho/godotenv`, `pgregory.net/rapid`
  - Create the full directory skeleton: `cmd/server/`, `internal/config/`, `internal/db/`, `internal/models/`, `internal/dto/`, `internal/middleware/`, `internal/handlers/`, `internal/services/`, `internal/worker/`, `internal/cleanup/`, `internal/clients/`, `internal/router/`
  - _Requirements: 14.1_

- [x] 2. Implement configuration loading
  - [x] 2.1 Create `internal/config/config.go` with the `Config` struct and `Load()` function
    - Read `PORT`, `JWT_SECRET`, `REPLICATE_API_TOKEN`, `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`, `DB_PATH` from environment variables
    - Default `PORT` to `8080` and `DB_PATH` to `./showcaster.db` when absent or empty
    - Return a non-zero exit and log the missing variable name when any required secret is absent or empty
    - Validate that `PORT` is an integer in range 1–65535; exit non-zero with a descriptive log on invalid value
    - _Requirements: 14.1, 14.2, 14.3, 14.4, 14.5_

  - [ ]* 2.2 Write property test for config validation (Property 17)
    - **Property 17: Config validation rejects missing required environment variables**
    - **Validates: Requirements 14.2, 14.5**
    - Generate random subsets of missing required vars and assert non-zero exit behavior
    - Generate invalid PORT values (non-integer, out-of-range) and assert non-zero exit behavior

- [x] 3. Implement data models and database setup
  - [x] 3.1 Create GORM model files
    - `internal/models/user.go`: `User` struct with all fields from the design (ID, Email, FullName, PasswordHash, IsVerified, OTPCode, OTPExpiresAt, OTPFailCount, CreatedAt, UpdatedAt)
    - `internal/models/job.go`: `Job` struct with all fields including `Steps []Step` association and `User User` association
    - `internal/models/step.go`: `Step` struct with ID, JobID, Name, Status, VideoURL, CreatedAt, UpdatedAt
    - Use `gorm:"primaryKey;type:text"` tags and all index/constraint annotations from the design
    - _Requirements: 6.1, 7.4, 9.1_

  - [x] 3.2 Create `internal/db/db.go` with GORM SQLite connection and auto-migration
    - Open SQLite connection using `gorm.io/driver/sqlite` with the configured `DB_PATH`
    - Run `db.AutoMigrate(&models.User{}, &models.Job{}, &models.Step{})` on startup
    - Create composite index `idx_jobs_status_created_at` on `(status, created_at)` after migration
    - Return the `*gorm.DB` instance for injection into services
    - _Requirements: 6.1, 12.2_

- [x] 4. Implement DTOs and request validation
  - [x] 4.1 Create `internal/dto/auth.go` with auth request/response types
    - `RegisterRequest` with `validate` tags: `email` (required, email, max=254), `fullName` (required, max=100), `password` (required, min=8, max=72)
    - `LoginRequest`, `VerifyOTPRequest` with appropriate `validate` tags
    - `TokenResponse` with `Token string`
    - _Requirements: 1.1, 1.3, 1.4, 1.5, 2.6, 3.1_

  - [x] 4.2 Create `internal/dto/job.go` with job request/response types
    - `CreateJobRequest` with `validate` tags for all enum fields (`oneof=...`) and URL fields (`url,startswith=https://`)
    - `JobResponse`, `StepResponse`, `JobListResponse`, `JobSummary` structs matching the design exactly
    - `PaginationParams` struct with `Page` and `Limit` fields
    - _Requirements: 6.2, 6.3, 6.4, 6.5, 6.6, 6.7, 9.1, 10.1, 10.4_

- [x] 5. Implement JWT middleware
  - [x] 5.1 Create `internal/middleware/jwt.go` with the `JWTMiddleware` factory function
    - Use `gofiber/contrib/jwt` with `jwtware.Config{SigningKey: ...}` and a custom `ErrorHandler` returning HTTP 401 `{"error": "unauthorized"}`
    - Export a helper `ExtractUserID(c *fiber.Ctx) string` that reads `c.Locals("user")`, casts to `*jwt.Token`, and returns `claims["sub"].(string)`
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

  - [ ]* 5.2 Write property test for JWT middleware (Property 5)
    - **Property 5: JWT middleware rejects all invalid tokens on protected routes**
    - **Validates: Requirements 4.1, 4.2, 4.3, 4.4**
    - Generate random tokens: no header, malformed header, expired token, wrong-key token — all must return HTTP 401
    - Generate valid tokens and assert `userID` is correctly extracted

- [x] 6. Implement Auth_Service
  - [x] 6.1 Create `internal/services/auth_service.go` implementing the `AuthService` interface
    - `Register`: validate uniqueness, hash password with bcrypt cost 12, create `User` record, generate 6-digit OTP with 10-minute expiry, send OTP email (stub/log if email client not wired), return HTTP 201
    - `Login`: look up user by email, compare bcrypt hash, return identical HTTP 401 for wrong password and unknown email, issue HS256 JWT with `sub=userID`, `iat=now`, `exp=iat+86400`
    - `VerifyOTP`: check expiry (10 min), check fail count (lock at ≥5), mark `IsVerified=true`, return JWT on success
    - `ResendOTP`: generate new OTP, reset fail count and expiry, persist
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 2.1, 2.2, 2.3, 2.4, 2.5, 3.1, 3.2, 3.3, 3.4, 3.5_

  - [ ]* 6.2 Write property test for registration uniqueness (Property 1)
    - **Property 1: Registration creates a unique user record**
    - **Validates: Requirements 1.1, 1.2**
    - Generate random valid registration payloads; assert exactly one DB record created and HTTP 201 returned
    - Re-register same email; assert HTTP 409 and no duplicate record

  - [ ]* 6.3 Write property test for password complexity validation (Property 2)
    - **Property 2: Password complexity validation rejects all non-conforming inputs**
    - **Validates: Requirements 1.4**
    - Generate passwords violating each rule (too short, too long, no uppercase, no lowercase, no digit); assert HTTP 400 identifying `password` field

  - [ ]* 6.4 Write property test for login JWT round-trip (Property 3)
    - **Property 3: Login round-trip issues a correctly-formed JWT**
    - **Validates: Requirements 2.1, 2.2, 2.3, 2.4**
    - Generate random registered+verified users; assert HTTP 200 and JWT with `exp = iat + 86400`
    - Assert wrong password and unknown email both return HTTP 401 with identical bodies

  - [ ]* 6.5 Write property test for OTP lockout (Property 4)
    - **Property 4: OTP verification is idempotent on failure and locks after 5 attempts**
    - **Validates: Requirements 3.3, 3.5**
    - Submit 5 incorrect OTPs; assert subsequent attempts (including correct OTP) return HTTP 400 "no longer valid"
    - Verify correct OTP on already-verified account returns HTTP 409

- [x] 7. Implement Cloudinary_Client
  - [x] 7.1 Create `internal/clients/cloudinary.go` implementing the `CloudinaryClient` interface
    - Initialize `cloudinary-go/v2` SDK with `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`
    - `UploadImage`: upload `io.Reader` to `showcaster/images` folder, return public URL
    - `UploadVideoFromURL`: upload from source URL to `showcaster/videos` folder, return public URL
    - Return a typed error on upload failure for HTTP 502 mapping
    - _Requirements: 5.1, 5.2, 5.6, 7.4, 7.9_

- [x] 8. Implement upload handler
  - [x] 8.1 Create `internal/handlers/upload.go` with the image upload handler
    - Parse `file` field from multipart form; return HTTP 400 `{"error": "file field is required"}` if absent
    - Check file size ≤ 10 MB; return HTTP 413 on violation
    - Check MIME type is `image/jpeg` or `image/png`; return HTTP 415 on violation
    - Call `CloudinaryClient.UploadImage`; return HTTP 200 `{"url": "..."}` on success, HTTP 502 on failure
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6_

  - [ ]* 8.2 Write property test for image upload validation (Property 6)
    - **Property 6: Image upload validates file type and size before uploading**
    - **Validates: Requirements 5.1, 5.2, 5.3, 5.4**
    - Generate random file sizes and MIME types; assert HTTP 415 for non-JPEG/PNG, HTTP 413 for >10 MB
    - Assert valid JPEG/PNG ≤ 10 MB calls Cloudinary and returns HTTP 200

- [x] 9. Implement Replicate_Client
  - [x] 9.1 Create `internal/clients/replicate.go` implementing the `ReplicateClient` interface
    - Define sentinel errors `ErrGenerationFailed` and `ErrTimeout`
    - `Generate`: POST to `https://api.replicate.com/v1/predictions` with model `wan-ai/wan2.1-i2v-480p` and all required inputs; include `Authorization: Bearer <REPLICATE_API_TOKEN>` on every request
    - Build step prompt from the step name using the prompt table in the design
    - Poll `GET .../predictions/{id}` every 5 seconds; return output URL on `succeeded`, `ErrGenerationFailed` on `failed`
    - After 10 minutes without terminal state: POST `.../predictions/{id}/cancel`, return `ErrTimeout`
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5_

  - [ ]* 9.2 Write property test for Replicate_Client request correctness (Property 11)
    - **Property 11: Replicate_Client constructs correct requests and handles all terminal states**
    - **Validates: Requirements 8.1, 8.2, 8.3, 8.4, 8.5**
    - Generate random job/step combinations; assert all required inputs present and `Authorization` header set
    - Mock `succeeded` response → assert output URL returned; mock `failed` → assert `ErrGenerationFailed`; mock timeout → assert `ErrTimeout` (distinguishable from `ErrGenerationFailed`)

- [x] 10. Implement Job_Service
  - [x] 10.1 Create `internal/services/job_service.go` implementing the `JobService` interface
    - `CreateJob`: run GORM transaction inserting one `Job` (status=pending) and four `Step` rows (Hook, Problem, Solution, Closure, each status=pending) atomically; perform non-blocking channel send; return HTTP 503 if channel is full
    - `GetJob`: fetch job with steps by ID; return HTTP 404 if not found, HTTP 403 if `job.UserID != userID`; map `videoUrl` to `null` for pending/processing steps
    - `ListJobs`: query jobs by `userID` ordered by `createdAt DESC`; apply `page`/`limit` pagination; return `total`, `page`, `limit` in response; default limit 100
    - `DeleteJob`: return HTTP 404 if not found, HTTP 403 on ownership mismatch, HTTP 409 if status is `pending` or `processing`; atomically delete job and steps in a transaction for `completed`/`failed` jobs
    - _Requirements: 6.1, 6.8, 9.1, 9.2, 9.3, 9.4, 9.5, 10.1, 10.2, 10.3, 10.4, 10.5, 11.1, 11.2, 11.3, 11.4, 11.5_

  - [ ]* 10.2 Write property test for job creation atomicity (Property 7)
    - **Property 7: Job submission atomically creates one Job and four Steps**
    - **Validates: Requirements 6.1**
    - Generate random valid job payloads; assert exactly one `Job` (status=pending) and exactly four `Step` records (Hook, Problem, Solution, Closure, all status=pending) with matching `job_id`; assert HTTP 202 with UUID `jobId`

  - [ ]* 10.3 Write property test for job payload validation (Property 8)
    - **Property 8: Job payload validation rejects all out-of-range and missing field inputs**
    - **Validates: Requirements 6.2, 6.3, 6.4, 6.5, 6.6, 6.7**
    - Generate payloads with each field invalid or missing; assert HTTP 400 identifying the invalid field by name

  - [ ]* 10.4 Write property test for cross-user ownership enforcement (Property 12)
    - **Property 12: Job access enforces ownership — cross-user requests return 403**
    - **Validates: Requirements 9.3, 11.3**
    - Generate random pairs of users and jobs; assert GET/DELETE by non-owner returns HTTP 403; assert owner access succeeds

  - [ ]* 10.5 Write property test for job list isolation (Property 13)
    - **Property 13: Job list returns only the authenticated user's jobs in descending order**
    - **Validates: Requirements 10.1, 10.2**
    - Generate random multi-user job sets; assert each user's list contains only their own jobs, ordered by `createdAt` descending, capped at 100

  - [ ]* 10.6 Write property test for pagination correctness (Property 14)
    - **Property 14: Pagination returns the correct subset and metadata**
    - **Validates: Requirements 10.4**
    - Generate random `page`, `limit`, and total job counts; assert response contains exactly `min(limit, max(0, T-(page-1)*limit))` jobs and correct `total`, `page`, `limit` metadata

  - [ ]* 10.7 Write property test for deletion atomicity and status constraints (Property 15)
    - **Property 15: Job deletion is atomic and respects status constraints**
    - **Validates: Requirements 11.1, 11.4**
    - Generate random jobs with `completed`/`failed` status; assert DELETE removes job and all steps atomically (HTTP 204)
    - Generate jobs with `pending`/`processing` status; assert DELETE returns HTTP 409

- [x] 11. Checkpoint — core services complete
  - Ensure all tests pass, ask the user if questions arise.

- [x] 12. Implement Job_Worker
  - [x] 12.1 Create `internal/worker/job_worker.go` with the `JobWorker` struct and `Run` method
    - `Run(ctx context.Context)`: loop over `w.queue` channel, call `w.processJob(ctx, job)` for each job
    - `processJob`: wrap in `recover()` to catch panics, log and mark job `failed` on unrecoverable error
    - Update `job.status = processing` at the start of each job
    - Iterate steps in order Hook → Problem → Solution → Closure; update `step.status = processing` before each step
    - For each step: call `ReplicateClient.Generate`, then `CloudinaryClient.UploadVideoFromURL`; on success atomically update `step.status=completed`, `step.videoUrl`, and `job.progress=(completedCount/4)*100`
    - On error: retry up to 3 times with 30-second sleep between attempts; after 3 failures set `step.status=failed` and `job.status=failed` and return
    - After all 4 steps complete: set `job.status=completed`
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 7.7, 7.8, 7.9_

  - [ ]* 12.2 Write property test for Job_Worker status transitions (Property 9)
    - **Property 9: Job_Worker drives correct status transitions for every step**
    - **Validates: Requirements 7.2, 7.3, 7.4, 7.5, 7.7**
    - Mock Replicate with random success/fail patterns; assert DB transitions follow `pending → processing → completed/failed`
    - Assert `job.progress` after `k` completed steps equals `(k/4)*100`
    - Assert steps are processed in Hook → Problem → Solution → Closure order

  - [ ]* 12.3 Write property test for retry exhaustion (Property 10)
    - **Property 10: Step retry logic exhausts exactly 3 attempts before marking failure**
    - **Validates: Requirements 7.6, 7.9**
    - Mock Replicate/Cloudinary to always fail; assert exactly 3 call attempts per step before `step.status=failed` and `job.status=failed`
    - Assert no further attempts are made after the third failure

- [x] 13. Implement cleanup routine
  - [x] 13.1 Create `internal/cleanup/cleanup.go` with `StartCleanupRoutine` and `runCleanup`
    - `StartCleanupRoutine(db *gorm.DB, logger *slog.Logger)`: create `time.NewTicker(24 * time.Hour)`, start goroutine that calls `runCleanup` on each tick
    - `runCleanup`: execute a single GORM transaction deleting steps then jobs where `status IN ('completed','failed') AND created_at < now - 120h`; log count of deleted jobs (including zero); on any DB error roll back, log error, and return without crashing
    - _Requirements: 12.1, 12.2, 12.3, 12.4, 12.5_

  - [ ]* 13.2 Write property test for TTL cleanup precision (Property 16)
    - **Property 16: TTL cleanup deletes exactly the right jobs**
    - **Validates: Requirements 12.2**
    - Generate random job sets with varying statuses and ages; after `runCleanup`, assert all `completed`/`failed` jobs older than 120h are deleted along with their steps; assert all other jobs remain untouched

- [x] 14. Implement job and auth HTTP handlers
  - [x] 14.1 Create `internal/handlers/auth.go` with handlers for all auth routes
    - `Register`: parse and validate `RegisterRequest`; call `AuthService.Register`; return HTTP 201 on success, HTTP 400/409 on errors
    - `Login`: parse and validate `LoginRequest`; call `AuthService.Login`; return HTTP 200 with `TokenResponse` on success, HTTP 400/401/503 on errors
    - `VerifyOTP`: parse and validate `VerifyOTPRequest`; call `AuthService.VerifyOTP`; return HTTP 200 with `TokenResponse` on success, HTTP 400/404/409 on errors
    - `ResendOTP`: parse email; call `AuthService.ResendOTP`; return HTTP 200 on success, HTTP 404 on unknown email
    - _Requirements: 1.1–1.7, 2.1–2.6, 3.1–3.5_

  - [x] 14.2 Create `internal/handlers/job.go` with handlers for all job routes
    - `CreateJob`: parse and validate `CreateJobRequest`; call `JobService.CreateJob`; return HTTP 202 `{"jobId": "..."}` on success, HTTP 400/503 on errors
    - `GetJob`: extract `userID` from JWT context; call `JobService.GetJob`; return HTTP 200 with `JobResponse`, HTTP 403/404/503 on errors
    - `ListJobs`: parse `page`/`limit` query params (return HTTP 400 on invalid values); call `JobService.ListJobs`; return HTTP 200 with `JobListResponse`
    - `DeleteJob`: extract `userID`; call `JobService.DeleteJob`; return HTTP 204 on success, HTTP 403/404/409/503 on errors
    - _Requirements: 6.1–6.8, 9.1–9.5, 10.1–10.5, 11.1–11.5_

- [x] 15. Implement health check handler and router
  - [x] 15.1 Create health check handler in `internal/handlers/` (or inline in router)
    - Attempt DB ping with a 2-second context deadline
    - Return HTTP 200 `{"status": "ok"}` on success; HTTP 503 `{"status": "degraded", "error": "..."}` on failure or timeout
    - Ensure the handler always responds within 3 seconds
    - _Requirements: 13.1, 13.2, 13.3_

  - [x] 15.2 Create `internal/router/router.go` with all route registrations
    - Register Fiber `Recover` middleware globally
    - Register public routes: `POST /api/auth/register`, `POST /api/auth/login`, `POST /api/auth/verify-otp`, `POST /api/auth/resend-otp`, `GET /health`
    - Apply `JWTMiddleware` to a route group and register protected routes: `POST /api/upload/image`, `POST /api/jobs/generate`, `GET /api/jobs`, `GET /api/jobs/:id`, `DELETE /api/jobs/:id`
    - _Requirements: 4.4, 13.1_

- [x] 16. Wire everything together in main.go
  - [x] 16.1 Create `cmd/server/main.go` as the application entry point
    - Load `.env` file via `godotenv` (dev only, ignore error if absent)
    - Call `config.Load()` and exit non-zero on error
    - Initialize DB via `db.New(cfg.DBPath)` and run auto-migration
    - Instantiate all clients (`CloudinaryClient`, `ReplicateClient`), services (`AuthService`, `JobService`), and handlers
    - Create the `Job_Queue` channel (buffer size 100)
    - Start `JobWorker.Run` in a goroutine
    - Start `cleanup.StartCleanupRoutine` in a goroutine
    - Register all routes via `router.Setup`
    - Start Fiber server on `cfg.Port`
    - _Requirements: 7.8, 12.1, 14.1–14.5_

- [x] 17. Write integration tests
  - [ ]* 17.1 Write integration test for the full auth flow
    - Test: register → verify OTP → login using in-memory SQLite
    - _Requirements: 1.1, 1.6, 3.1, 2.1_

  - [ ]* 17.2 Write integration test for the full job submission and polling flow
    - Test: submit job → worker processes with mocked Replicate/Cloudinary → poll status until `completed`
    - _Requirements: 6.1, 7.1–7.5, 9.1_

  - [ ]* 17.3 Write integration test for the health check endpoint
    - Test: health check with real DB connection returns HTTP 200; health check with unavailable DB returns HTTP 503
    - _Requirements: 13.1, 13.2_

- [x] 18. Final checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- The design uses Go throughout — no language selection was needed
- Property tests use `pgregory.net/rapid` with a minimum of 100 iterations per property; set `RAPID_CHECKS=500` for more thorough runs
- Run tests with `go test ./...` from `apps/showcaster-be/`; use `go test -race ./...` to catch data races
- Checkpoints ensure incremental validation before proceeding to the next phase
- The `Job_Queue` channel buffer size of 100 absorbs bursts; HTTP 503 is returned only when the buffer is full
- All DB operations in services should wrap GORM errors and return HTTP 503 when the DB is unreachable

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1", "3.1", "4.1", "4.2"] },
    { "id": 1, "tasks": ["2.2", "3.2", "5.1"] },
    { "id": 2, "tasks": ["5.2", "6.1", "7.1"] },
    { "id": 3, "tasks": ["6.2", "6.3", "6.4", "6.5", "8.1", "9.1"] },
    { "id": 4, "tasks": ["8.2", "9.2", "10.1"] },
    { "id": 5, "tasks": ["10.2", "10.3", "10.4", "10.5", "10.6", "10.7", "12.1"] },
    { "id": 6, "tasks": ["12.2", "12.3", "13.1"] },
    { "id": 7, "tasks": ["13.2", "14.1", "14.2"] },
    { "id": 8, "tasks": ["15.1", "15.2"] },
    { "id": 9, "tasks": ["16.1"] },
    { "id": 10, "tasks": ["17.1", "17.2", "17.3"] }
  ]
}
```
