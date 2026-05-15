# Design Document: Wan 2.7 Image-to-Video Integration

## Overview

This document describes the technical design for integrating the Alibaba Cloud Model Studio (DashScope) Wan 2.7 image-to-video API as a swappable video generation provider in the Showcaster backend. The integration is additive — no existing code paths are broken. The `WanClient` already exists in `internal/clients/wan.go` and partially implements the `ReplicateClient` interface. This design formalises the gaps and the wiring changes needed in `config.go` and `main.go`.

## Architecture

```
HTTP Handler
     │
     ▼
Job_Service ──► Job_Queue (chan models.Job, buf=100)
                     │
                     ▼
               Job_Worker.Run()
                     │
                     ├─ ReplicateClient.Generate()  ◄── WanClient  (VIDEO_PROVIDER=wan)
                     │                              ◄── VeoClient  (VIDEO_PROVIDER=veo)
                     │
                     └─ CloudinaryClient.UploadVideoFromURL()
```

The `Job_Worker` depends on the `ReplicateClient` interface. `WanClient` and `VeoClient` both implement it. `main.go` selects the concrete implementation at startup based on `Config.VideoProvider`.

## Component Design

### 1. `internal/config/config.go` — Changes Required

The `Config` struct already has `VideoProvider` and `DashScopeAPIKey` fields. Two gaps remain:

**Gap 1 — Validation of `VIDEO_PROVIDER`:** The current `Load()` accepts any string for `VideoProvider`. It must reject values other than `veo` and `wan`.

**Gap 2 — Conditional requirement of `DASHSCOPE_API_KEY`:** When `VIDEO_PROVIDER=wan`, `DASHSCOPE_API_KEY` must be non-empty. Currently it is always optional.

```go
// After loading VideoProvider:
switch cfg.VideoProvider {
case "veo":
    // no extra required vars
case "wan":
    if cfg.DashScopeAPIKey == "" {
        return nil, fmt.Errorf("required environment variable %q is missing or empty (required when VIDEO_PROVIDER=wan)", "DASHSCOPE_API_KEY")
    }
default:
    return nil, fmt.Errorf("invalid VIDEO_PROVIDER value %q: must be \"veo\" or \"wan\"", cfg.VideoProvider)
}
```

### 2. `cmd/server/main.go` — Provider Selection

Replace the hardcoded `veoClient` construction with a provider switch:

```go
var videoClient clients.ReplicateClient

switch cfg.VideoProvider {
case "wan":
    wanClient := clients.NewWanClient(cfg.DashScopeAPIKey, openaiClient)
    videoClient = wanClient
    logger.Info("video provider: wan", "model", "wan2.7-i2v-2026-04-25")
default: // "veo"
    vc, err := clients.NewVeoClient(cfg.GoogleAIAPIKey, openaiClient)
    if err != nil {
        logger.Error("failed to initialise Veo client", "error", err)
        os.Exit(1)
    }
    videoClient = vc
    logger.Info("video provider: veo", "model", "veo-3.1-generate-preview")
}

// Pass videoClient to the worker instead of veoClient
go worker.NewJobWorker(jobQueue, gormDB, videoClient, cloudinaryClient, logger).Run(context.Background())
```

### 3. `internal/clients/wan.go` — Gaps to Close

The existing `wan.go` is largely correct. The following gaps must be addressed:

#### 3a. `ModelImageURL` guard (Requirement 4.7)
Before building the request body, check that `job.ModelImageURL` is non-empty:

```go
if job.ModelImageURL == "" {
    return "", fmt.Errorf("wan: job.ModelImageURL is empty — cannot submit task")
}
```

#### 3b. Resolution warning for `4k` and unknown values (Requirements 6.3, 6.4)
The current `wanResolution` function silently maps `4k` → `1080P` and unknown → `720P`. Add structured log warnings:

```go
func (c *WanClient) resolveResolution(internal string) string {
    switch internal {
    case "720p":
        return "720P"
    case "1080p":
        return "1080P"
    case "4k":
        c.logger.Warn("wan: 4k resolution not supported, substituting 1080P", "original", internal, "substituted", "1080P")
        return "1080P"
    default:
        c.logger.Warn("wan: unrecognized resolution, defaulting to 720P", "original", internal, "defaulted", "720P")
        return "720P"
    }
}
```

#### 3c. Duration clamping (Requirements 7.3, 7.4)
Add a helper that clamps and logs:

```go
func clampDuration(d int, logger *slog.Logger) int {
    if d < 2 {
        logger.Warn("wan: duration below minimum, clamping to 2", "configured", d)
        return 2
    }
    if d > 15 {
        logger.Warn("wan: duration above maximum, clamping to 15", "configured", d)
        return 15
    }
    return d
}
```

#### 3d. `InvalidApiKey` detection (Requirement 14.1)
After a non-2xx response on task submission, check for the `InvalidApiKey` code:

```go
type wanErrorResponse struct {
    Code      string `json:"code"`
    Message   string `json:"message"`
    RequestID string `json:"request_id"`
}

// In submitTask, after reading a non-2xx response:
var apiErr wanErrorResponse
if json.Unmarshal(rawBody, &apiErr) == nil && apiErr.Code == "InvalidApiKey" {
    return "", fmt.Errorf("wan: invalid DashScope API key (request_id=%s): %s", apiErr.RequestID, apiErr.Message)
}
return "", fmt.Errorf("wan: submit task: status %d: %s", status, string(rawBody))
```

#### 3e. `UNKNOWN` task status includes Task_ID (Requirement 13.1)
```go
case "UNKNOWN":
    return "", fmt.Errorf("wan: task %s status UNKNOWN — task may have expired or does not exist", taskID)
```

#### 3f. Driving audio support (Requirement 4.6)
The `submitTask` signature should accept an optional `drivingAudioURL string`. When non-empty, append the `driving_audio` media entry. For the current pipeline this will always be empty string (no audio), but the field is wired through for future use.

Since `Generate(ctx, job, stepName)` is the interface boundary and `models.Job` does not have a `DrivingAudioURL` field, the driving audio URL is sourced from a new optional field `Job.DrivingAudioURL` (type `string`, GORM column `driving_audio_url`). If the field is absent or empty, no `driving_audio` entry is added.

### 4. `internal/models/job.go` — Optional Field Addition

Add an optional `DrivingAudioURL` field:

```go
DrivingAudioURL string `gorm:"column:driving_audio_url;default:''" json:"drivingAudioUrl,omitempty"`
```

GORM's `AutoMigrate` will add the column on next startup without data loss.

## Data Flow: Task Submission and Polling

```
Generate(ctx, job, stepName)
  │
  ├─ guard: job.ModelImageURL != ""
  ├─ buildPrompt (GPT or template)
  ├─ resolveResolution(job.Resolution)
  ├─ clampDuration(wanVideoDuration)
  │
  ├─ POST /api/v1/services/aigc/video-generation/video-synthesis
  │    Headers: Authorization, Content-Type, X-DashScope-Async: enable
  │    Body: { model, input: { prompt, negative_prompt, media: [first_frame, ?driving_audio] },
  │            parameters: { resolution, duration, prompt_extend, watermark } }
  │
  ├─ Extract task_id from response
  │
  └─ Poll loop (every 15s, deadline = now + 15min)
       │
       ├─ GET /api/v1/tasks/{task_id}
       │    Headers: Authorization
       │
       ├─ PENDING / RUNNING  → continue
       ├─ SUCCEEDED          → return video_url
       ├─ FAILED / CANCELED  → return ErrWanGenerationFailed (with code + message)
       ├─ UNKNOWN            → return error (task expired)
       ├─ network error      → log warn, continue
       └─ deadline exceeded  → return ErrWanTimeout
```

## Request / Response Structures

### Create Task Request
```json
{
  "model": "wan2.7-i2v-2026-04-25",
  "input": {
    "prompt": "<generated or template prompt>",
    "negative_prompt": "low resolution, blurry, distorted, deformed, extra fingers, bad proportions, watermark, text, logo, typography",
    "media": [
      { "type": "first_frame", "url": "<job.ModelImageURL>" },
      { "type": "driving_audio", "url": "<job.DrivingAudioURL>" }  // omitted if empty
    ]
  },
  "parameters": {
    "resolution": "720P",
    "duration": 8,
    "prompt_extend": true,
    "watermark": false
  }
}
```

### Create Task Response (success)
```json
{
  "output": { "task_status": "PENDING", "task_id": "<uuid>" },
  "request_id": "<uuid>"
}
```

### Poll Response (SUCCEEDED)
```json
{
  "output": {
    "task_id": "<uuid>",
    "task_status": "SUCCEEDED",
    "video_url": "https://dashscope-result-sh.oss-cn-shanghai.aliyuncs.com/xxx.mp4?Expires=xxx",
    "orig_prompt": "..."
  }
}
```

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `VIDEO_PROVIDER` | No | `veo` | Active video backend: `veo` or `wan` |
| `DASHSCOPE_API_KEY` | When `VIDEO_PROVIDER=wan` | — | Alibaba Cloud DashScope API key |

All other variables (`GOOGLE_AI_API_KEY`, `OPENAI_API_KEY`, etc.) are unchanged.

## Error Sentinel Mapping

| DashScope condition | Returned error |
|---|---|
| `task_status: FAILED` | `ErrWanGenerationFailed` (wrapped) |
| `task_status: CANCELED` | `ErrWanGenerationFailed` (wrapped) |
| `task_status: UNKNOWN` | plain `error` (task expired) |
| Polling deadline exceeded | `ErrWanTimeout` (wrapped) |
| `code: InvalidApiKey` | plain `error` (config issue) |
| `video_url` empty on SUCCEEDED | plain `error` |

## Files Changed

| File | Change type |
|---|---|
| `internal/config/config.go` | Modify — add `VIDEO_PROVIDER` validation and conditional `DASHSCOPE_API_KEY` requirement |
| `cmd/server/main.go` | Modify — replace hardcoded Veo wiring with provider switch |
| `internal/clients/wan.go` | Modify — close gaps: ModelImageURL guard, resolution warnings, duration clamping, InvalidApiKey detection, UNKNOWN includes task_id, driving audio support |
| `internal/models/job.go` | Modify — add optional `DrivingAudioURL` field |
