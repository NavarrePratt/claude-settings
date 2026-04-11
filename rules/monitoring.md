# Monitoring & Repeated Checks

## Banned: Bash polling loops

NEVER use bash `for`/`while` loops with `sleep` to poll or monitor anything.
These are flaky, inconsistent, and unreliable - timing drifts, loops die
silently, and output is hard to act on.

```bash
# NEVER do this
while true; do kubectl get pods; sleep 30; done
for i in $(seq 1 60); do curl ...; sleep 10; done
watch -n 5 "gh run view ..."
```

## Use cron tools for monitoring

When you need to repeatedly check on something - deploys, CI runs, pod status,
build progress, resource convergence - use `CronCreate` to schedule it.

**Creating a monitor:**
```
CronCreate(cron: "*/5 * * * *", prompt: "Check if the deploy to staging has completed by running ...")
```

**Key parameters:**
- `cron` - standard 5-field cron expression in local time
- `prompt` - the full instruction to execute each time it fires
- `recurring` - `true` (default) for ongoing monitors, `false` for one-shot reminders
- `durable` - `true` to survive session restarts (persists to disk)

**Managing monitors:**
- `CronList()` - see all active jobs
- `CronDelete(id: "...")` - cancel a monitor when done

**Constraints to tell the user:**
- Jobs only fire while the REPL is idle (not mid-query)
- Recurring jobs auto-expire after 7 days
- Session-only by default - gone when Claude exits (use `durable: true` to persist)

## Use Monitor for continuous streaming

When you need to watch something live and react as events arrive - log tailing,
CI status changes, directory watches - use the `Monitor` tool. It runs a script
in the background and feeds each output line back so you can interject mid-conversation.

Good use cases for Monitor:
- Tail a log file and flag errors as they appear
- Poll a PR or CI job and report when its status changes
- Watch a directory for file changes
- Track output from any long-running script

Monitor uses the same permission rules as Bash. Not available on Bedrock,
Vertex AI, or Foundry. Requires v2.1.98+.

**Monitor vs CronCreate:** CronCreate fires a prompt on a schedule (minimum
granularity: 1 minute, only fires while REPL is idle). Monitor streams output
continuously and lets you react line-by-line in real time. Use CronCreate for
periodic checks with coarse timing. Use Monitor when you need to watch a live
stream and respond to individual events.

## Background tasks are still fine

Long-lived one-shot work - builds, test suites, large git operations - should
still use `run_in_background: true` on Bash or the Agent tool. The distinction:

| Pattern | Tool |
|---|---|
| Run once, wait for result | `run_in_background: true` |
| Check repeatedly until condition met | `CronCreate` |
| Watch a live stream and react to events | `Monitor` |
| Remind me to do X at time Y | `CronCreate(recurring: false)` |
