---
name: watch-pr
description: >
  Watch a pull request and react locally to upstream events - CI failures,
  reviews, approvals, merge conflicts, and the merge itself - without ever
  writing back to GitHub. Use when the user says "watch this PR", "keep an
  eye on PR #N", "monitor my PR", "let me know when CI passes", "tell me
  when someone reviews", "alert me when it merges", or any variation of
  wanting a background observer on a PR. Use this skill proactively
  whenever a user has just pushed a PR and wants hands-off follow-through
  while they work on something else. This skill uses the Monitor tool to
  poll gh on an interval and fires LOCAL-ONLY reactions (investigate
  failures, import reviews, start rebases, surface notifications). It
  NEVER posts comments, creates reviews, merges PRs, or pushes commits.
argument-hint: "[pr-number-or-url] [--poll-interval=60]"
---

# Watch PR

Run a background watcher on a pull request that reacts locally to
upstream events as they happen. The skill's value is eliminating manual
polling ("did CI pass yet?", "any reviews come in?") while preserving
strict local-only boundaries - every reaction is something you'd do on
your own machine, never something visible to the team.

## When to use

Trigger on any variation of:

- "watch this PR"
- "monitor PR #14"
- "keep an eye on the PR"
- "let me know when CI passes / when someone reviews / when it merges"
- "tell me if there's a merge conflict"
- "watch-pr"

The user wants a continuous observer, not a one-shot check. If they just
want the current state of a PR, use `gh pr view` directly instead.

## Arguments

- **Optional positional**: a PR number (`14`) or URL
  (`https://github.com/org/repo/pull/14`). If omitted, resolve from the
  current branch via `gh pr view --json number,url,headRefName`. If the
  current branch has no associated PR, stop and ask the user which PR to
  watch.
- **`--poll-interval=N`**: seconds between gh polls. Default `60`,
  minimum `30` (below that you will hit gh rate limits and look like a
  bot). Values below 30 are clamped with a warning.

There is intentionally no `--auto` flag. This skill is already a
hands-off observer; every user-visible reaction is either a notification
or invokes another skill that has its own approval flow.

## Boundaries: What this skill will NEVER do

The entire point of this skill is to be the opposite of a bot. These
actions are categorically forbidden, even if the user asks mid-session:

- **Never post a PR comment or review** (`gh pr comment`, `gh pr review`,
  `gh api .../comments POST`).
- **Never merge, close, or reopen the PR** (`gh pr merge`, `gh pr close`,
  `gh pr reopen`).
- **Never push to the remote** (`git push` in any form, including force
  variants).
- **Never modify the PR title, description, labels, reviewers, or base
  branch** via any `gh` or `gh api` write call.
- **Never trigger workflow reruns** (`gh run rerun`, `gh workflow run`).
- **Never acknowledge a review to the reviewer in any form visible to
  them** - notifications go to the user's local session only.

If the user asks the watcher to "reply to that comment" or "push the
fix", stop the Monitor, tell them this skill is read-only, and hand off
to them (or to an explicitly write-capable skill if one exists). Do not
silently extend scope.

Local reactions are fine: reading logs, starting rebases that stay
uncommitted, importing review comments into local state, opening files,
running tests. Anything that stays on this machine.

---

## Phase 0: Resolve the target PR and sanity checks

Before starting the Monitor, determine exactly what we're watching and
verify we can read it.

### Step 1: Resolve the PR identifier

Parse arguments:
- If a PR number or URL was provided, use it.
- Otherwise, run `gh pr view --json number,url,headRefName,state` from the
  current directory. If the current branch has a PR, use that.
- If neither works, stop and ask the user which PR to watch.

Record:
- **PR_NUMBER**: numeric PR number
- **PR_URL**: full URL
- **HEAD_REF**: the PR's head branch (may differ from current local branch)
- **REPO**: `owner/repo` form (derive from `gh repo view --json
  nameWithOwner -q .nameWithOwner`)

### Step 2: Verify PR state

```bash
gh pr view "$PR_NUMBER" --json state,mergeable,isDraft
```

Refuse to watch if:
- State is `MERGED` or `CLOSED` - the PR is already over. Tell the user
  and suggest `/postmerge` if it's merged.
- `isDraft` is true - ask the user if they really want to watch a draft;
  draft PRs won't get reviews and CI may be limited. Proceed if they
  confirm.

### Step 3: Verify gh is authenticated

```bash
gh auth status
```

If this fails, stop and ask the user to run `gh auth login`. The Monitor
loop depends on gh working for the entire watch duration.

### Step 4: Poll interval

Parse `--poll-interval=N`. Default 60. If N < 30, warn and clamp to 30.

---

## Phase 1: Initialize state file

The Monitor script needs a durable state file so it can diff "what's new
since last poll" and avoid re-firing on events we've already reacted to.

### Create the state directory

```bash
STATE_DIR="/tmp/watch-pr-${PR_NUMBER}"
mkdir -m 700 -p "$STATE_DIR"
# Verify we own it - /tmp is predictable, guard against symlink tricks
[ "$(stat -f %u "$STATE_DIR" 2>/dev/null || stat -c %u "$STATE_DIR")" = "$(id -u)" ] \
  || { echo "Refusing to use $STATE_DIR: not owned by current user" >&2; exit 1; }
```

### Prime the state file

```bash
STATE_FILE="$STATE_DIR/state.json"
```

On first watch of this PR (no existing state file), prime two files with
the current snapshot so we don't flood the user with "event" lines for
comments and reviews that already existed when the watch started:

```bash
gh pr view "$PR_NUMBER" --json \
  state,mergeable,mergeStateStatus,reviewDecision,isDraft,statusCheckRollup,reviews,comments,headRefOid \
  > "$STATE_FILE"

# Inline review comments live on a separate endpoint - gh pr view only
# returns issue-level PR comments (the main conversation thread), not
# the ones tied to specific code lines.
INLINE_FILE="$STATE_DIR/inline.json"
gh api "repos/$REPO/pulls/$PR_NUMBER/comments" --paginate \
  --jq '[.[] | {id, author: .user.login, path, line}]' \
  > "$INLINE_FILE"
```

Record the initial values for later diffing:
- `state` (OPEN / MERGED / CLOSED)
- `mergeable` (MERGEABLE / CONFLICTING / UNKNOWN)
- `mergeStateStatus` (CLEAN / DIRTY / BLOCKED / etc.)
- `reviewDecision` (APPROVED / CHANGES_REQUESTED / REVIEW_REQUIRED / null)
- `statusCheckRollup` - list of checks with `state` per check
- `reviews[].id` and `comments[].id` as sets for diff detection
- `INLINE_FILE` inline review comment IDs as a separate set
- `headRefOid` - detect force-pushes

If `$STATE_FILE` already exists from a previous `/watch-pr` session for
the same PR, keep it. The script below will diff against it and only
emit events for changes since last observation.

---

## Phase 2: Start the Monitor

This is the heart of the skill. The Monitor runs a polling script that
emits one tagged line per detected event. Every stdout line becomes a
notification in the chat; the script writes raw API output to a side
file for reaction handlers to read.

### The polling script

Write this script to `$STATE_DIR/poll.sh` so the Monitor command stays
readable:

```bash
cat > "$STATE_DIR/poll.sh" <<'POLL_EOF'
#!/usr/bin/env bash
set -uo pipefail

STATE_DIR="${1:?state dir required}"
PR_NUMBER="${2:?pr number required}"
POLL_INTERVAL="${3:-60}"
REPO="${4:?repo owner/name required}"
STATE_FILE="$STATE_DIR/state.json"
LATEST_FILE="$STATE_DIR/latest.json"
INLINE_FILE="$STATE_DIR/inline.json"
INLINE_LATEST="$STATE_DIR/inline_latest.json"

while true; do
  if ! gh pr view "$PR_NUMBER" --json \
      state,mergeable,mergeStateStatus,reviewDecision,isDraft,statusCheckRollup,reviews,comments,headRefOid \
      > "$LATEST_FILE" 2>"$STATE_DIR/gh.err"; then
    echo "GH_ERROR $(tr '\n' ' ' < "$STATE_DIR/gh.err" | head -c 200)"
    sleep "$POLL_INTERVAL"
    continue
  fi

  # Inline review comments live on a separate endpoint. Failures here
  # aren't fatal - keep the rest of the poll running with stale inline data.
  if ! gh api "repos/$REPO/pulls/$PR_NUMBER/comments" --paginate \
       --jq '[.[] | {id, author: .user.login, path, line}]' \
       > "$INLINE_LATEST" 2>>"$STATE_DIR/gh.err"; then
    cp "$INLINE_FILE" "$INLINE_LATEST" 2>/dev/null || echo '[]' > "$INLINE_LATEST"
  fi

  # Merge / close - terminal states, emit and exit
  new_state=$(jq -r .state "$LATEST_FILE")
  old_state=$(jq -r .state "$STATE_FILE" 2>/dev/null || echo OPEN)
  if [ "$new_state" != "$old_state" ]; then
    case "$new_state" in
      MERGED) echo "MERGED pr=$PR_NUMBER"; cp "$LATEST_FILE" "$STATE_FILE"; exit 0 ;;
      CLOSED) echo "CLOSED pr=$PR_NUMBER"; cp "$LATEST_FILE" "$STATE_FILE"; exit 0 ;;
    esac
  fi

  # CI check transitions
  new_failed=$(jq -r '[.statusCheckRollup[]? | select(.conclusion=="FAILURE" or .conclusion=="TIMED_OUT" or .conclusion=="CANCELLED") | .name] | sort | .[]' "$LATEST_FILE")
  old_failed=$(jq -r '[.statusCheckRollup[]? | select(.conclusion=="FAILURE" or .conclusion=="TIMED_OUT" or .conclusion=="CANCELLED") | .name] | sort | .[]' "$STATE_FILE" 2>/dev/null)
  newly_failed=$(comm -23 <(printf '%s\n' "$new_failed" | sort -u) <(printf '%s\n' "$old_failed" | sort -u) | grep -v '^$' || true)
  if [ -n "$newly_failed" ]; then
    while IFS= read -r check; do
      [ -z "$check" ] || echo "CHECK_FAILED check=$check"
    done <<< "$newly_failed"
  fi

  # CI now fully passing (transition from not-all-green to all-green)
  new_total=$(jq -r '[.statusCheckRollup[]?] | length' "$LATEST_FILE")
  new_success=$(jq -r '[.statusCheckRollup[]? | select(.conclusion=="SUCCESS" or .conclusion=="NEUTRAL" or .conclusion=="SKIPPED")] | length' "$LATEST_FILE")
  old_total=$(jq -r '[.statusCheckRollup[]?] | length' "$STATE_FILE" 2>/dev/null || echo 0)
  old_success=$(jq -r '[.statusCheckRollup[]? | select(.conclusion=="SUCCESS" or .conclusion=="NEUTRAL" or .conclusion=="SKIPPED")] | length' "$STATE_FILE" 2>/dev/null || echo 0)
  if [ "$new_total" -gt 0 ] && [ "$new_success" = "$new_total" ] && \
     { [ "$old_total" = 0 ] || [ "$old_success" != "$old_total" ]; }; then
    echo "CI_PASSING checks=$new_total"
  fi

  # Reviews (submitted = a review with a state, not just inline comments).
  # Filter out PENDING: gh pr view exposes the authenticated user's own
  # draft review with state=PENDING, and without this filter we'd fire
  # REVIEW_SUBMITTED every time someone starts typing a review, and then
  # fire it a second time when PENDING transitions to the real state.
  new_reviews=$(jq -r '[.reviews[]? | select(.state != "PENDING") | {id,state,author:.author.login}] | tostring' "$LATEST_FILE")
  old_reviews=$(jq -r '[.reviews[]? | select(.state != "PENDING") | {id,state,author:.author.login}] | tostring' "$STATE_FILE" 2>/dev/null || echo '[]')
  if [ "$new_reviews" != "$old_reviews" ]; then
    added=$(jq -r --argjson old "$old_reviews" \
      '[.reviews[]? | select(.state != "PENDING") | {id,state,author:.author.login}] as $new
       | ($new - $old)[] | "\(.author) \(.state)"' "$LATEST_FILE")
    if [ -n "$added" ]; then
      while IFS= read -r line; do
        [ -z "$line" ] || echo "REVIEW_SUBMITTED $line"
      done <<< "$added"
    fi
  fi

  # PR-thread comments (issue-level) + inline review comments, together
  new_issue_ids=$(jq -r '[.comments[]?.id] | sort | .[]' "$LATEST_FILE" 2>/dev/null)
  old_issue_ids=$(jq -r '[.comments[]?.id] | sort | .[]' "$STATE_FILE" 2>/dev/null)
  new_inline_ids=$(jq -r '[.[]?.id] | sort | .[]' "$INLINE_LATEST" 2>/dev/null)
  old_inline_ids=$(jq -r '[.[]?.id] | sort | .[]' "$INLINE_FILE" 2>/dev/null)
  issue_new=$(comm -23 <(printf '%s\n' "$new_issue_ids" | sort -u) <(printf '%s\n' "$old_issue_ids" | sort -u) | grep -v '^$' | wc -l | tr -d ' ')
  inline_new=$(comm -23 <(printf '%s\n' "$new_inline_ids" | sort -u) <(printf '%s\n' "$old_inline_ids" | sort -u) | grep -v '^$' | wc -l | tr -d ' ')
  total_new=$((issue_new + inline_new))
  if [ "$total_new" -gt 0 ]; then
    echo "REVIEW_COMMENTS_ADDED count=$total_new inline=$inline_new thread=$issue_new"
  fi

  # Merge conflict (transition into CONFLICTING, not steady state)
  new_mergeable=$(jq -r .mergeable "$LATEST_FILE")
  old_mergeable=$(jq -r .mergeable "$STATE_FILE" 2>/dev/null || echo UNKNOWN)
  if [ "$new_mergeable" = "CONFLICTING" ] && [ "$old_mergeable" != "CONFLICTING" ]; then
    echo "MERGE_CONFLICT pr=$PR_NUMBER"
  fi

  cp "$LATEST_FILE" "$STATE_FILE"
  cp "$INLINE_LATEST" "$INLINE_FILE"
  sleep "$POLL_INTERVAL"
done
POLL_EOF
chmod +x "$STATE_DIR/poll.sh"
```

### Launch the Monitor

```
Monitor(
  description: "watch-pr #<PR_NUMBER>: CI, reviews, merge status",
  persistent: true,
  timeout_ms: 3600000,
  command: "$STATE_DIR/poll.sh $STATE_DIR $PR_NUMBER $POLL_INTERVAL $REPO"
)
```

Use `persistent: true` because PR watching runs for the life of the
session - could be minutes or hours. The Monitor will stop cleanly via
`TaskStop` (Phase 4) or when the PR hits a terminal state (MERGED /
CLOSED - the script self-exits).

After launching, tell the user: "Watching PR #<N> every <interval>s.
I'll react locally to CI failures, reviews, and merge conflicts. Say
`stop watching` when you want to end."

---

## Phase 3: React to events

Each notification from the Monitor is a line of the form `<TAG> <key=value> ...`.
Parse the tag and run the matching reaction. Reactions must be
**local-only** - see Boundaries.

### MERGED

PR has landed. Two steps:
1. Surface a notification: "PR #<N> merged. Running /postmerge to
   clean up locally."
2. Invoke `/postmerge <PR_NUMBER>` (the postmerge skill will verify the
   merge and tear down the worktree, branch, and /tmp artifacts).

The Monitor script will self-exit after emitting MERGED. Proceed to
Phase 4 cleanup after /postmerge returns.

### CLOSED

PR was closed without merging. Surface a notification and stop:
"PR #<N> was closed without merging. Local branch and worktree are
intact if you want to reopen - use `gh pr reopen <N>` yourself."

Do NOT delete the branch or worktree automatically - a closed PR may be
reopened or resurrected as a new PR.

### CHECK_FAILED check=<name>

A CI check just flipped to failure. Investigate locally:

1. Surface the failure: "CI check `<name>` failed on PR #<N>.
   Investigating."
2. Fetch logs: `gh pr checks <PR_NUMBER>` then for the failing check
   run `gh run view --log-failed <run_id>` (get run_id from `gh pr
   checks`).
3. Read the failure in enough detail to identify the root cause - which
   test, which file, which assertion. Don't summarize and stop;
   summarize with a diagnosis.
4. Surface a concise diagnosis to the user and offer a next step (e.g.
   "I can run that test locally to confirm" or "Looks like a flake -
   want me to keep watching?"). Do NOT rerun the workflow (`gh run
   rerun`) - that's an upstream write.

Multiple `CHECK_FAILED` lines may arrive in quick succession if several
checks fail together. Group them in your response if they arrive close
in time.

### REVIEW_SUBMITTED <author> <state>

A reviewer submitted a review. State is one of `APPROVED`,
`CHANGES_REQUESTED`, `COMMENTED`, `DISMISSED`.

1. Surface: "<author> submitted a <state> review on PR #<N>."
2. If state is `CHANGES_REQUESTED` or `COMMENTED`, invoke
   `/pr-review-import <PR_NUMBER>` so the inline comments are pulled
   into local state ready to address.
3. If state is `APPROVED`, surface prominently (the user asked to be
   alerted on approvals) and note that merging is still their call.
4. If state is `DISMISSED`, just surface the fact.

### REVIEW_COMMENTS_ADDED count=<N> inline=<N> thread=<N>

New comments were added since the last poll. `inline` counts inline
review comments tied to specific code lines; `thread` counts issue-level
comments on the PR conversation. Either type warrants pulling the review
in:

1. Invoke `/pr-review-import <PR_NUMBER>` to ingest the new comments.
2. Mention the breakdown in your surface ("3 inline, 1 thread").

If a `REVIEW_SUBMITTED` line arrived in the same batch, skip the
import here - the review-submission path already handles it.

### MERGE_CONFLICT

The PR just became un-mergeable. Start a local rebase attempt but do NOT
push:

1. Surface: "PR #<N> now has merge conflicts with the base branch.
   Starting a local rebase."
2. `cd` to the worktree for this PR's branch (check `git worktree
   list` - the /implement pattern puts it at
   `.claude/worktrees/implement-<slug>`).
3. Run:
   ```bash
   git fetch origin <base_ref>
   git rebase origin/<base_ref>
   ```
4. If the rebase has conflicts, STOP at the conflict and surface the
   conflicting files to the user with:
   "Rebase paused on conflicts in: <files>. Resolve them yourself - I
   won't touch the conflict markers. Run `git rebase --continue` when
   done. **I will NOT push the rebased branch** - that's your call."
5. If the rebase succeeds cleanly, surface: "Rebase applied cleanly.
   Branch is ahead of remote by <N> commits. **Push when ready** - I
   won't."

Do not run `git push --force-with-lease` or any push variant, even if
the rebase was clean. That's an upstream write.

### CI_PASSING checks=<N>

All required checks just transitioned to success. Surface once:
"All <N> CI checks now passing on PR #<N>."

This event fires on the transition only, not every poll. No action
beyond the notification.

### GH_ERROR <message>

`gh pr view` failed. Usually transient (network blip, auth token
refresh, rate limit). The polling script logs the error and keeps
going.

Surface only if the same error repeats for 3+ polls in a row - before
that it's noise. If it repeats:
1. Surface: "gh has been failing for <N> polls: <message>"
2. Ask the user if they want to keep watching (auth may need
   refreshing).

Track repeat counts in memory for this session, not in the state file.

---

## Phase 4: Stop the watch

The watch ends in one of four ways:

1. **Natural terminal**: the PR merged or closed. The polling script
   self-exits; the Monitor ends; do the MERGED / CLOSED reaction from
   Phase 3.
2. **User says stop**: "stop watching", "kill the watcher", "that's
   enough". Call `TaskStop` on the Monitor task. Surface: "Watch
   stopped."
3. **Session end**: the Monitor dies with the session. No action
   needed - the state file in `/tmp/watch-pr-<N>/` survives for a
   future watch.
4. **Repeated gh errors**: the user tells the skill to give up after
   `GH_ERROR` spam. Same as case 2.

Do not auto-stop the watcher on CI failures or review comments - the
user may want to address those and keep watching for follow-up events.

### Cleanup

On explicit stop (case 2 or 4), leave the state file in place - if
the user restarts the watch shortly, priming skips re-processing old
events. Only remove `/tmp/watch-pr-<N>/` when invoked by `/postmerge`
(which sweeps all `/tmp` artifacts for a merged PR).

---

## Composition with other skills

- `/postmerge` runs automatically on MERGED. The user doesn't need to
  invoke it separately.
- `/pr-review-import` runs automatically on REVIEW_SUBMITTED
  (non-approval) and REVIEW_COMMENTS_ADDED.
- `/implement` is the typical upstream caller - someone finishes
  /implement, pushes the PR, then runs /watch-pr to babysit the review
  cycle.

These compositions are the reason this skill exists as a separate
watcher rather than being folded into /implement Phase 6. The PR
lifecycle after push is unbounded in time and orthogonal to
implementation - keeping it separate lets the user dismiss the watcher
without losing /implement state, and vice versa.

---

## Error handling

| Scenario | Recovery |
|----------|----------|
| PR already merged/closed at Phase 0 | Stop, suggest /postmerge if merged |
| Current branch has no PR and no arg given | Ask user for PR number |
| `gh auth status` fails | Stop, ask user to `gh auth login` |
| Poll interval < 30s | Clamp to 30, warn |
| gh returns error during polling | Log, skip this poll, continue loop |
| gh errors 3+ polls in a row | Surface and ask user whether to keep watching |
| Worktree not found during MERGE_CONFLICT rebase | Surface; skip the rebase; tell user to handle manually |
| Rebase has conflicts | Stop at the conflict, surface files, do NOT resolve or push |
| Monitor tool unavailable | Skill cannot function; tell user and exit |
| `/postmerge` or `/pr-review-import` not installed | Surface the event anyway, skip the invocation |

---

## Guidelines

- **Read-only posture**: every reaction is a local action or a
  notification. This is non-negotiable. If you're tempted to make an
  exception ("just this one comment"), stop and tell the user to do it
  themselves.
- **One event, one notification**: emit events on transitions, not
  steady state. A check that has been failing for an hour should only
  fire CHECK_FAILED once, not every 60s.
- **State file is a diff base**: prime it on first watch to avoid
  flooding the user with historical events.
- **Don't narrate polls**: the user shouldn't see "polling...",
  "polling...", "polling...". Only surface events and their reactions.
- **Keep reactions snappy**: a CHECK_FAILED investigation shouldn't
  block for minutes. If the logs are huge, summarize and move on.
- **Minimum 30s poll interval**: gh rate limits are per-hour, and a 60
  poll/hour watcher flies well under them. 30s/60s is plenty to catch
  events with human-latency tolerance.
- **/tmp ownership check**: /tmp paths are predictable and attacker-
  writable. Always verify the state directory is owned by the current
  user before writing to it.
- **Persistent Monitor**: use `persistent: true` - PR watches run as
  long as the session. Timeout is a safety net, not the normal exit.
