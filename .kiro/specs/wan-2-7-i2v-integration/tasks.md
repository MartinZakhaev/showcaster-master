# Implementation Plan: Wan 2.7 Image-to-Video Integration

## Overview

Close the gaps between the existing partial `WanClient` implementation and the full requirements. Changes touch four files: `config.go`, `main.go`, `wan.go`, and `models/job.go`. No new files are needed.

## Tasks

- [x] 1. Harden `config.go` — validate `VIDEO_PROVIDER` and conditionally require `DASHSCOPE_API_KEY`
  - In `config.Load()`, after reading `VIDEO_PROVIDER`, add a `switch` that:
    - Accepts `"veo"` (no extra required vars)
    - Accepts `"wan"` and returns an error if `DASHSCOPE_API_KEY` is empty, naming the missing variable
    - Returns an error for any other value, identifying the invalid `VIDEO_PROVIDER` string
  - Keep the existing `cfg.DashScopeAPIKey = os.Getenv("DASHSCOPE_API_KEY")` assignment; move it before the switch so the switch can inspect it
  - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

- [x] 2. Update `main.go` — wire provider selection at startup
  - Replace the hardcoded `veoClient` construction with a `switch cfg.VideoProvider` block:
    - `"wan"`: call `clients.NewWanClient(cfg.DashScopeAPIKey, openaiClient)`, assign to a `clients.ReplicateClient` variable, emit `logger.Info("video provider: wan", "model", "wan2.7-i2v-2026-04-25")`
    - `"veo"` (default): call `clients.NewVeoClient(cfg.GoogleAIAPIKey, openaiClient)`, assign to the same variable, emit `logger.Info("video provider: veo", ...)`
  - Pass the `clients.ReplicateClient` variable (not the concrete `veoClient`) to `worker.NewJobWorker`
  - When `VIDEO_PROVIDER=wan`, `GOOGLE_AI_API_KEY` is no longer required — guard the `NewVeoClient` call so it only runs in the `veo` branch
  - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

- [x] 3. Add `DrivingAudioURL` field to `models/job.go`
  - Add `DrivingAudioURL string` with GORM tag `gorm:"column:driving_audio_url;default:''"` and JSON tag `json:"drivingAudioUrl,omitempty"`
  - No migration script needed — `AutoMigrate` handles the new column
  - _Requirements: 4.6_

- [x] 4. Harden `wan.go` — close all implementation gaps
  - **4a. ModelImageURL guard** (Req 4.7): at the top of `submitTask`, return an error immediately if `job.ModelImageURL == ""`
  - **4b. Resolution warnings** (Req 6.3, 6.4): replace the package-level `wanResolution` function with a method `(c *WanClient) resolveResolution(internal string) string` that logs a structured `Warn` when substituting `4k → 1080P` or an unrecognized value `→ 720P`
  - **4c. Duration clamping** (Req 7.3, 7.4): add a `clampDuration(d int, logger *slog.Logger) int` helper that clamps to [2, 15] and logs a `Warn` when clamping occurs; use it when setting `Parameters.Duration`
  - **4d. InvalidApiKey detection** (Req 14.1): after a non-2xx response in `submitTask`, attempt to unmarshal a `wanErrorResponse{Code, Message, RequestID}` struct; if `Code == "InvalidApiKey"`, return a descriptive error including `RequestID`
  - **4e. UNKNOWN includes Task_ID** (Req 13.1): in the poll switch, change the `"UNKNOWN"` case to include `taskID` in the error message
  - **4f. Driving audio support** (Req 4.6): in `submitTask`, after appending the `first_frame` media entry, check `job.DrivingAudioURL != ""`; if true, append `{Type: "driving_audio", URL: job.DrivingAudioURL}` to the media slice
  - **4g. Use `resolveResolution` method**: update `submitTask` to call `c.resolveResolution(job.Resolution)` instead of the old package-level function
  - _Requirements: 4.6, 4.7, 6.3, 6.4, 7.3, 7.4, 13.1, 14.1_

- [x] 5. Build verification
  - Run `go build ./...` from `apps/showcaster-be/` and confirm zero errors
  - Run `go vet ./...` and confirm zero warnings
  - _Requirements: all_

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1", "3"] },
    { "id": 1, "tasks": ["4"] },
    { "id": 2, "tasks": ["2"] },
    { "id": 3, "tasks": ["5"] }
  ]
}
```

## Notes

- Tasks 1 and 3 are independent and can be done in parallel
- Task 4 depends on task 3 (needs `DrivingAudioURL` on the model)
- Task 2 depends on task 1 (needs validated `VideoProvider` in config) and task 4 (needs the complete `WanClient`)
- Task 5 is the final build gate
- No new dependencies are introduced — all changes use the standard library and existing packages
- The `.env` file needs `VIDEO_PROVIDER=wan` and `DASHSCOPE_API_KEY=<key>` to activate the Wan provider; `GOOGLE_AI_API_KEY` remains required only when `VIDEO_PROVIDER=veo`
