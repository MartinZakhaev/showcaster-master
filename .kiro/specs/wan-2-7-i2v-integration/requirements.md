# Requirements Document

## Introduction

This feature integrates the Alibaba Cloud Model Studio (DashScope) Wan 2.7 image-to-video API as a swappable video generation provider in the Showcaster backend. The Wan_Client SHALL implement the same `ReplicateClient.Generate` interface used by the Job_Worker, enabling the operator to switch the active video provider between Veo and Wan via a configuration value with no changes to the worker pipeline.

The integration covers configuration loading, asynchronous task submission, polling for task completion, mapping of internal job parameters to Wan's API parameters, terminal-state error handling, and handing the resulting (24-hour expiring) video URL to the existing Cloudinary uploader for permanent storage.

## Glossary

- **Wan_Client**: The Go client (`internal/clients/wan.go`) that calls the DashScope Wan 2.7 image-to-video API and implements the `ReplicateClient` interface.
- **Job_Worker**: The existing worker (`internal/worker/job_worker.go`) that consumes jobs from the queue and invokes `ReplicateClient.Generate` for each pipeline step (Hook, Problem, Solution, Closure).
- **DashScope_API**: The Alibaba Cloud Model Studio HTTP API rooted at `https://dashscope-intl.aliyuncs.com/api/v1`.
- **Wan_Model_ID**: The identifier `wan2.7-i2v-2026-04-25` passed in the `model` field of the create-task request.
- **Task_ID**: The `task_id` returned by DashScope when a video generation task is created. Valid for 24 hours.
- **Video_URL**: The signed URL of a generated MP4 returned by DashScope on a SUCCEEDED task. Valid for 24 hours.
- **Cloudinary_Client**: The existing client (`internal/clients/cloudinary.go`) that uploads video assets via `UploadVideoFromURL` for permanent storage.
- **OpenAI_Client**: The existing client (`internal/clients/openai.go`) that produces structured scene prompts using GPT-4.1-nano.
- **Job**: A `models.Job` row containing the user's input (`ModelImageURL`, `ProductName`, `Resolution`, `Orientation`, etc.) for a video generation request.
- **Internal_Resolution**: The resolution value stored on `Job.Resolution`, one of `720p`, `1080p`, or `4k`.
- **Wan_Resolution**: The resolution value accepted by DashScope, one of `720P` or `1080P`.
- **Driving_Audio_URL**: An optional HTTPS URL to an audio asset that, when supplied, drives lip-sync and motion in the generated video; passed to DashScope as a `driving_audio` entry in `input.media`.
- **Video_Provider**: The configuration value (`VIDEO_PROVIDER`) selecting which video backend the Job_Worker uses; one of `veo` or `wan`.
- **Pipeline_Step**: One of `Hook`, `Problem`, `Solution`, or `Closure` — the four scenes processed for each Job.
- **Terminal_Status**: A `task_status` value from which a task does not transition further: `SUCCEEDED`, `FAILED`, `CANCELED`, or `UNKNOWN`.
- **Non_Terminal_Status**: A `task_status` value indicating the task is still in progress: `PENDING` or `RUNNING`.

## Requirements

### Requirement 1: Configuration Loading

**User Story:** As an operator, I want to configure the Wan provider via environment variables, so that I can deploy the backend without code changes.

#### Acceptance Criteria

1. WHEN `Config.Load` is called and `VIDEO_PROVIDER` is set to `wan`, THE Configuration_Loader SHALL require `DASHSCOPE_API_KEY` to be a non-empty string.
2. IF `VIDEO_PROVIDER` is `wan` AND `DASHSCOPE_API_KEY` is missing or empty, THEN THE Configuration_Loader SHALL return an error identifying `DASHSCOPE_API_KEY` as the missing variable and SHALL NOT return a populated `Config`.
3. WHEN `VIDEO_PROVIDER` is unset, THE Configuration_Loader SHALL default it to `veo`.
4. WHERE `VIDEO_PROVIDER` is set to a value other than `veo` or `wan`, THE Configuration_Loader SHALL return an error identifying the invalid value.
5. WHEN `Config.Load` succeeds, THE Configuration_Loader SHALL expose `DashScopeAPIKey` and `VideoProvider` fields on the returned `Config` struct.

### Requirement 2: Provider Selection in Server Bootstrap

**User Story:** As an operator, I want the server to wire the Wan_Client into the Job_Worker when configured, so that jobs are routed to DashScope automatically.

#### Acceptance Criteria

1. WHEN the server starts AND `Config.VideoProvider` equals `wan`, THE Server_Bootstrap SHALL construct a Wan_Client using `Config.DashScopeAPIKey` and pass it to the Job_Worker as the `ReplicateClient` dependency.
2. WHEN the server starts AND `Config.VideoProvider` equals `veo`, THE Server_Bootstrap SHALL construct a Veo_Client and pass it to the Job_Worker as the `ReplicateClient` dependency.
3. WHEN the server starts AND `Config.OpenAIAPIKey` is non-empty AND `Config.VideoProvider` equals `wan`, THE Server_Bootstrap SHALL pass an OpenAI_Client to the Wan_Client constructor.
4. WHEN the server starts AND `Config.OpenAIAPIKey` is empty AND `Config.VideoProvider` equals `wan`, THE Server_Bootstrap SHALL pass `nil` for the OpenAI_Client argument to the Wan_Client constructor.
5. WHEN the server logs its startup banner AND `Config.VideoProvider` equals `wan`, THE Server_Bootstrap SHALL emit a structured log line containing the field `provider=wan` and the resolved `Wan_Model_ID`.

### Requirement 3: ReplicateClient Interface Conformance

**User Story:** As a developer, I want the Wan_Client to implement the existing `ReplicateClient` interface, so that the Job_Worker can use it without modification.

#### Acceptance Criteria

1. THE Wan_Client SHALL satisfy the `clients.ReplicateClient` interface defined in `internal/clients/replicate.go`.
2. THE Wan_Client SHALL expose a `Generate(ctx context.Context, job models.Job, stepName string) (string, error)` method.
3. WHEN `Generate` succeeds, THE Wan_Client SHALL return a non-empty MP4 URL string and a nil error.
4. WHEN `Generate` fails, THE Wan_Client SHALL return an empty string and a non-nil error.
5. THE Wan_Client SHALL return errors that wrap one of the sentinel errors `ErrWanGenerationFailed` or `ErrWanTimeout` for terminal failures and timeout conditions respectively, so callers can match using `errors.Is`.

### Requirement 4: Task Submission Request Construction

**User Story:** As a developer, I want the Wan_Client to build a correct DashScope create-task request from a Job, so that the API receives all required fields.

#### Acceptance Criteria

1. WHEN the Wan_Client submits a task, THE Wan_Client SHALL issue an HTTP `POST` to `https://dashscope-intl.aliyuncs.com/api/v1/services/aigc/video-generation/video-synthesis`.
2. WHEN the Wan_Client submits a task, THE Wan_Client SHALL set the request header `Authorization` to `Bearer ` concatenated with `Config.DashScopeAPIKey`.
3. WHEN the Wan_Client submits a task, THE Wan_Client SHALL set the request header `Content-Type` to `application/json` and the request header `X-DashScope-Async` to `enable`.
4. WHEN the Wan_Client submits a task, THE Wan_Client SHALL set the JSON body field `model` to the Wan_Model_ID `wan2.7-i2v-2026-04-25`.
5. WHEN the Wan_Client submits a task, THE Wan_Client SHALL set `input.media` to a JSON array containing one element with `type` equal to `first_frame` and `url` equal to `Job.ModelImageURL`.
6. WHERE a non-empty Driving_Audio_URL is supplied for the current Pipeline_Step, THE Wan_Client SHALL append a second element to `input.media` with `type` equal to `driving_audio` and `url` equal to the Driving_Audio_URL.
7. WHEN the Wan_Client submits a task AND `Job.ModelImageURL` is empty, THE Wan_Client SHALL return an error without sending an HTTP request.

### Requirement 5: Prompt Generation

**User Story:** As a content producer, I want each Pipeline_Step to receive a step-appropriate visual prompt, so that generated clips reflect the intended scene narrative.

#### Acceptance Criteria

1. WHEN the Wan_Client constructs a task body AND an OpenAI_Client is configured, THE Wan_Client SHALL call `OpenAIClient.GenerateScenePrompt(ctx, job, stepName)` and use the returned `VisualPrompt` as the `input.prompt` field.
2. IF the call to `OpenAIClient.GenerateScenePrompt` returns an error, THEN THE Wan_Client SHALL log a warning containing the step name and the error, and SHALL fall back to the template prompt produced by `buildVeoPrompt(stepName, job)`.
3. WHEN the Wan_Client constructs a task body AND no OpenAI_Client is configured, THE Wan_Client SHALL use the template prompt produced by `buildVeoPrompt(stepName, job)` as the `input.prompt` field.
4. THE Wan_Client SHALL set `input.negative_prompt` to a fixed string that lists undesired artifacts including at minimum: low resolution, blurry, distorted, deformed, watermark, text, logo, typography.

### Requirement 6: Resolution Mapping

**User Story:** As a developer, I want internal resolutions mapped to Wan's accepted values, so that the API does not reject the request.

#### Acceptance Criteria

1. WHEN the Wan_Client constructs the `parameters.resolution` field AND `Job.Resolution` equals `720p`, THE Wan_Client SHALL set `parameters.resolution` to `720P`.
2. WHEN the Wan_Client constructs the `parameters.resolution` field AND `Job.Resolution` equals `1080p`, THE Wan_Client SHALL set `parameters.resolution` to `1080P`.
3. WHEN the Wan_Client constructs the `parameters.resolution` field AND `Job.Resolution` equals `4k`, THE Wan_Client SHALL set `parameters.resolution` to `1080P` and SHALL log a warning containing the original `Job.Resolution` value and the substituted Wan_Resolution value.
4. WHERE `Job.Resolution` does not match any documented Internal_Resolution value, THE Wan_Client SHALL set `parameters.resolution` to `720P` and SHALL log a warning identifying the unrecognized input.

### Requirement 7: Duration Handling

**User Story:** As a developer, I want the task `duration` parameter constrained to Wan's accepted range, so that the API does not reject the request.

#### Acceptance Criteria

1. THE Wan_Client SHALL set `parameters.duration` to an integer in the inclusive range 2 through 15.
2. WHEN the Wan_Client constructs `parameters.duration` AND no per-step duration override is configured, THE Wan_Client SHALL set `parameters.duration` to the package-level constant `wanVideoDuration` whose value is between 2 and 15 inclusive.
3. IF a configured duration value is less than 2, THEN THE Wan_Client SHALL set `parameters.duration` to 2 and SHALL log a warning identifying the clamped value.
4. IF a configured duration value is greater than 15, THEN THE Wan_Client SHALL set `parameters.duration` to 15 and SHALL log a warning identifying the clamped value.

### Requirement 8: Watermark and Prompt-Extend Parameters

**User Story:** As a content producer, I want watermarking off and prompt extension on by default, so that output videos look clean and prompts include richer context.

#### Acceptance Criteria

1. THE Wan_Client SHALL set `parameters.watermark` to `false` by default.
2. THE Wan_Client SHALL set `parameters.prompt_extend` to `true` by default.

### Requirement 9: Task Submission Response Handling

**User Story:** As a developer, I want the Wan_Client to validate and extract the Task_ID from the create-task response, so that polling can proceed reliably.

#### Acceptance Criteria

1. WHEN the create-task HTTP response has a status code in the range 200 through 299 AND the response body contains a non-empty `output.task_id`, THE Wan_Client SHALL store that `task_id` as the Task_ID for subsequent polling.
2. IF the create-task HTTP response has a status code outside the range 200 through 299, THEN THE Wan_Client SHALL return an error containing the HTTP status code and the response body.
3. IF the create-task HTTP response body cannot be parsed as JSON, THEN THE Wan_Client SHALL return an error containing the response body.
4. IF the create-task HTTP response body has an empty `output.task_id` field, THEN THE Wan_Client SHALL return an error containing the `task_status`, `code`, and `message` fields from the response body.

### Requirement 10: Task Polling

**User Story:** As a developer, I want the Wan_Client to poll DashScope at a steady cadence until the task reaches a Terminal_Status, so that the worker is informed when the video is ready.

#### Acceptance Criteria

1. WHEN polling a task, THE Wan_Client SHALL issue an HTTP `GET` to `https://dashscope-intl.aliyuncs.com/api/v1/tasks/{task_id}` with the same `Authorization` header used for task submission.
2. THE Wan_Client SHALL wait 15 seconds between successive poll requests for the same Task_ID.
3. WHEN the polled `task_status` is `PENDING` or `RUNNING`, THE Wan_Client SHALL continue polling without modifying job state.
4. IF a poll request returns a network error or a non-2xx HTTP status, THEN THE Wan_Client SHALL log a warning containing the error and SHALL continue polling until the timeout deadline is reached.
5. WHEN the elapsed time since task submission exceeds 15 minutes AND the task has not reached a Terminal_Status, THE Wan_Client SHALL stop polling and return an error wrapping `ErrWanTimeout`.
6. WHEN the calling `context.Context` is cancelled, THE Wan_Client SHALL stop polling and return the context's error from `Generate`.

### Requirement 11: Successful Task Outcome

**User Story:** As a developer, I want the Wan_Client to return the generated video URL when a task succeeds, so that the worker can hand it to Cloudinary.

#### Acceptance Criteria

1. WHEN the polled `task_status` is `SUCCEEDED` AND `output.video_url` is non-empty, THE Wan_Client SHALL return `output.video_url` and a nil error from `Generate`.
2. IF the polled `task_status` is `SUCCEEDED` AND `output.video_url` is empty, THEN THE Wan_Client SHALL return an error indicating that the task succeeded with an empty video URL.

### Requirement 12: Failed and Canceled Task Handling

**User Story:** As a developer, I want clear errors when DashScope reports a terminal failure, so that the worker can mark the step failed and surface the reason in logs.

#### Acceptance Criteria

1. WHEN the polled `task_status` is `FAILED`, THE Wan_Client SHALL return an error wrapping `ErrWanGenerationFailed` and SHALL include the `code` and `message` fields from the response in the error message.
2. WHEN the polled `task_status` is `CANCELED`, THE Wan_Client SHALL return an error wrapping `ErrWanGenerationFailed` and SHALL include the `code` and `message` fields from the response in the error message.

### Requirement 13: Unknown and Expired Task Handling

**User Story:** As a developer, I want explicit handling of expired Task_IDs, so that callers receive a distinct, descriptive error.

#### Acceptance Criteria

1. WHEN the polled `task_status` is `UNKNOWN`, THE Wan_Client SHALL return an error whose message indicates the Task_ID may have expired or does not exist and SHALL include the Task_ID in the error message.
2. WHEN the polled `task_status` is a value not equal to `PENDING`, `RUNNING`, `SUCCEEDED`, `FAILED`, `CANCELED`, or `UNKNOWN`, THE Wan_Client SHALL log a warning containing the unrecognized status value and SHALL continue polling until the timeout deadline is reached.

### Requirement 14: API Key Rejection Handling

**User Story:** As an operator, I want a clear error when DashScope rejects the API key, so that I can fix configuration without searching through generic 4xx noise.

#### Acceptance Criteria

1. IF the create-task HTTP response has a status code of 401 or contains a top-level `code` field equal to `InvalidApiKey`, THEN THE Wan_Client SHALL return an error whose message identifies the failure as an invalid DashScope API key and SHALL include the DashScope `request_id` from the response when present.

### Requirement 15: Permanent Storage of Generated Video

**User Story:** As an operator, I want generated videos copied to Cloudinary before the DashScope URL expires, so that the video remains accessible after 24 hours.

#### Acceptance Criteria

1. WHEN the Wan_Client returns a Video_URL from `Generate`, THE Job_Worker SHALL pass that Video_URL to `CloudinaryClient.UploadVideoFromURL` and SHALL persist the resulting Cloudinary URL on the corresponding Step row.
2. IF `CloudinaryClient.UploadVideoFromURL` returns an error after a successful Wan_Client generation, THEN THE Job_Worker SHALL treat the step as failed and apply its existing retry-and-fail logic.
3. THE Wan_Client SHALL NOT itself call the Cloudinary_Client; it SHALL only return the Video_URL string.

### Requirement 16: Logging and Observability

**User Story:** As an operator, I want structured logs for each stage of a Wan task, so that I can troubleshoot failures and monitor latency.

#### Acceptance Criteria

1. WHEN the Wan_Client submits a task, THE Wan_Client SHALL emit a structured log line with level `info` containing the fields `step`, `endpoint`, `model`, and `resolution`.
2. WHEN the create-task request returns a Task_ID, THE Wan_Client SHALL emit a structured log line with level `info` containing the fields `step` and `taskId`.
3. WHEN the Wan_Client receives a poll response, THE Wan_Client SHALL emit a structured log line with level `info` containing the fields `jobId`, `step`, `status`, and `elapsed`.
4. WHEN the Wan_Client returns a successful Video_URL, THE Wan_Client SHALL emit a structured log line with level `info` containing the fields `jobId`, `step`, `elapsed`, and `videoUrl`.
5. WHEN the Wan_Client returns an error wrapping `ErrWanGenerationFailed`, THE Wan_Client SHALL emit a structured log line with level `error` containing the fields `jobId`, `step`, `code`, and `message`.
6. WHEN the Wan_Client returns an error wrapping `ErrWanTimeout`, THE Wan_Client SHALL emit a structured log line with level `error` containing the fields `jobId`, `step`, and `elapsed`.

### Requirement 17: HTTP Client Behavior

**User Story:** As a developer, I want the Wan_Client to use a bounded HTTP timeout per request, so that a hung connection cannot stall the worker indefinitely.

#### Acceptance Criteria

1. THE Wan_Client SHALL use an `http.Client` whose per-request `Timeout` is set to a finite duration of 60 seconds or less.
2. WHEN an HTTP request times out, THE Wan_Client SHALL treat the timeout as a transient poll error per Requirement 10.4 if it occurred during polling, and as a submit error per Requirement 9.2 if it occurred during task submission.
3. WHEN constructing any HTTP request, THE Wan_Client SHALL use `http.NewRequestWithContext` with the `context.Context` passed into `Generate`, so that context cancellation interrupts in-flight requests.
