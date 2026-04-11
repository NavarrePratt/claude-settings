# No Fragile Counts

Never include specific counts of items in commit messages, PR descriptions,
or bead descriptions.

Bad: "Add 7 tests", "Update 3 config files", "Fix 12 linting errors"
Good: "Add tests for auth module", "Update config for new logging", "Fix linting errors in handlers"

Counts go stale before the commit is pushed or the PR is created. A rebase,
amend, fixup, or late-stage edit changes the number and the message becomes
inaccurate. The diff shows exactly what changed - the message should explain why.

If a count is truly load-bearing context (rare), put it in the PR body where
it can be edited, not in a commit message.
