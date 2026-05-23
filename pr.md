# fix(cron): add missing `action` arg for command job execution

## Problem

Cron jobs with `payload.kind = "command"` never actually execute their shell command. The cron scheduler fires on schedule, logs completion in ~6ms, and marks `lastStatus: "ok"` — but the command is silently dropped.

This affects any command job scheduled via `jobs.json`, including jobs created by the `cron` agent tool or inserted directly.

## Root Cause

In `pkg/tools/cron.go`, `CronTool.ExecuteJob()` builds an args map for `ExecTool.Execute()` but omits the required `"action": "run"` field:

```go
// Before (broken)
args := map[string]any{
    "command":   job.Payload.Command,
    "__channel": channel,
    "__chat_id": chatID,
}
```

`ExecTool.Execute()` (`pkg/tools/shell.go:264`) requires `action` and returns `ErrorResult("action is required")` immediately when it's missing.

## Why It Was Silent

Three compounding issues made this invisible:

1. **`ExecuteJob` always returns `"ok"`** — the exec tool error is captured into an output string and published to the message bus, but the return value is unconditionally `"ok"`. The cron service never sees an error and always sets `lastStatus: "ok"`.

2. **Error output goes to a dead channel** — command jobs without an explicit `channel`/`to` default to `cli`/`direct`, which has no listener when picoclaw runs as a gateway service. The error message is published but nobody receives it.

3. **No logging at the exec level** — the gateway log level is typically `warn` or higher, and the `ErrorResult` from the exec tool doesn't generate a log entry through the cron code path.

## Fix

One-line change — add `"action": "run"` to the args map:

```go
// After (fixed)
args := map[string]any{
    "action":    "run",
    "command":   job.Payload.Command,
    "__channel": channel,
    "__chat_id": chatID,
}
```

## Testing

Verified on a live picoclaw gateway (NixOS, systemd service):

- **Before fix**: command jobs complete in ~6ms, no shell execution, no output, no errors logged
- **After fix**: command jobs execute the shell command, produce output, and publish results to the message bus

Test command used: `echo cron-works >> /tmp/pico-cron-test.txt` — file was created successfully after the fix.

## Suggested Follow-Up

The silent failure pattern could be improved separately:

- `ExecuteJob` should propagate exec errors to the cron service so `lastStatus` reflects actual command success/failure
- Command job errors should be logged at `warn` level regardless of whether a message bus listener exists
