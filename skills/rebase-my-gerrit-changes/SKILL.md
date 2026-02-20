---
name: rebase-my-gerrit-changes
description: Automatically rebase Gerrit changes that are behind master and have an unaddressed rebase warning comment. Runs up to four times a day (6am, 12pm, 6pm, midnight). Requires the ger CLI.
---

# Rebase My Gerrit Changes

Automatically rebase your Gerrit changes that have received a rebase warning comment and haven't been rebased since.

## Prerequisites

Requires the `ger` CLI. If not installed:

```bash
# Install Bun runtime
curl -fsSL https://bun.sh/install | bash

# Install ger
bun install -g @aaronshaf/ger
```

See: https://github.com/aaronshaf/ger

## Schedule

Run this skill at: **6am, 3pm** (twice daily).

Cron expression: `0 6,15 * * *`

## Criteria for Rebasing

A change qualifies for automatic rebase if ALL of the following are true:

1. **Owned by you** — you are the change owner
2. **Verified: +1** — the change has a Verified +1 label
3. **No Code-Review -2** — the change does not have a CR -2
4. **Has an unaddressed rebase warning comment** posted within the last 72 hours matching:
   > "This commit may not be safe to merge ([X] commits behind master). Please rebase to make sure all the tests still pass."
5. **Not yet rebased since the warning** — the change has not been rebased after the warning comment was posted
6. **No existing merge conflict** — the change does not currently have a merge conflict
7. **No previously failed rebase attempt** — a rebase has not already been attempted and failed for this change (if it failed before, leave it alone)

## Steps

### 1. Find qualifying changes

```bash
ger search --json --limit 200 'owner:self status:open label:Verified=+1 -label:Code-Review=-2'
```

This returns changes with Verified+1 and no CR-2 in one query. Note: the Gerrit `message:` query operator searches commit messages, not review comments — so the rebase warning must be found by inspecting each change's messages individually.

### 2. Check each change for the rebase warning

For each change number from step 1, run:

```bash
ger show --json <change-number>
```

From the response:
- **`messages[]`** — scan for any message containing `"not be safe to merge"` posted within the last 72 hours
- **`change.revisions`** — check if any patchset was created *after* the warning date (if so, already rebased — skip)
- **`change.mergeable`** — if `false`, skip (merge conflict)

Skip silently if a rebase was previously attempted and failed (track this in memory during the session).

You can parallelize these requests (e.g. 10 at a time) for speed.

### 3. Rebase each qualifying change

```bash
ger rebase <change-number>
```

Run for each change that passes all checks. No confirmation needed.

### 4. Report

Only report if at least one change was rebased. Use terse bullet points:

- ✅ Rebased #401518, #401525
- ⏭️ Skipped #401530 (merge conflict)
- ⏭️ Skipped #401531 (prior failed rebase)

If nothing was rebased, stay silent.
