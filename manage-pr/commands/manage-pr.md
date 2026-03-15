---
description: Create a PR and monitor it until CI passes and all comments are addressed
allowed-tools: Bash(gh:*), Bash(git:*), Read, Write, Edit, Grep, Glob, Agent, CronCreate, CronDelete, CronList
argument-hint: "[base-branch]"
---

## Context

- Current git status: !`git status`
- Current git diff (staged and unstaged): !`git diff HEAD`
- Current branch: !`git branch --show-current`
- Recent commits on this branch: !`git log --oneline -10`
- Remote tracking: !`git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null || echo "No upstream set"`

## Task: Create and Shepherd a Pull Request

This is a multi-phase workflow. Execute each phase in order.

### Phase 1: Create the Pull Request

1. If there are uncommitted changes, stage and commit them with an appropriate message.
2. If on `main` or `master`, create a new descriptive branch first.
3. Push the branch to origin (use `-u` to set upstream).
4. Determine the base branch: use `$ARGUMENTS` if provided, otherwise default to `main` (or `master` if `main` doesn't exist).
5. Create the PR using `gh pr create`. Write a clear title and body summarizing the changes. Include a `## Test plan` section.
6. Capture and display the PR URL and number.

### Phase 2: Monitor CI Status

After creating the PR, monitor CI checks by polling with `gh pr checks` and `gh pr view --json statusCheckRollup`.

**Polling strategy:**
- Poll every 15 seconds while checks are pending or in progress.
- Use a bash loop for the tight polling since cron only supports minute-level granularity:
  ```
  Poll CI by running: gh pr checks <pr-number> --watch --fail-fast
  ```
  If `--watch` is available, prefer it. Otherwise, manually poll in a loop using `gh pr view <number> --json statusCheckRollup` every 15 seconds with a timeout of 30 minutes.

**On CI failure:**
1. Identify which check(s) failed using `gh pr checks <number>`.
2. Fetch the failed check's logs: `gh run view <run-id> --log-failed` (extract the run ID from the checks output).
3. Analyze the failure — understand the root cause.
4. Fix the issue in the code.
5. Stage, commit, and push the fix.
6. Return to the top of Phase 2 to monitor again.

**On CI success:** proceed to Phase 3.

### Phase 3: Monitor for PR Comments

Once CI is green, set up recurring comment monitoring using CronCreate with a 1-minute interval (`*/1 * * * *`). The cron prompt should instruct the following:

**Cron prompt content:**
> Check PR #<number> for new review comments. Run `gh pr view <number> --json reviews,comments` and `gh api repos/<owner>/<repo>/pulls/<number>/comments`. Compare against the last known comment count stored in `/tmp/manage-pr-<number>-comment-count`. If there are new comments:
> 1. Read and understand each new comment
> 2. Address the feedback by modifying code as requested
> 3. Stage, commit, and push changes with a message referencing the feedback
> 4. Reply to the comment on the PR using `gh pr comment` acknowledging the change
> 5. Update the stored comment count
> 6. After pushing, re-check CI status by running `gh pr checks <number> --watch --fail-fast` or polling `gh pr view <number> --json statusCheckRollup` every 15 seconds until checks complete. If CI fails, diagnose and fix as described in Phase 2.
>
> If there are no new comments, do nothing.
>
> If CI is green AND there have been no new comments for the last 3 consecutive checks, output: "PR #<number> is stable — CI green, no pending comments." and delete this cron job.

Also initialize the comment count file:
```bash
gh pr view <number> --json comments,reviews --jq '.comments | length + (.reviews | length)' > /tmp/manage-pr-<number>-comment-count
```

### Phase 4: Completion

The workflow completes when:
- CI is green
- No new comments have appeared for 3 consecutive polling cycles

At completion, display a summary:
- PR URL
- Number of CI fix iterations
- Number of comment rounds addressed
- Final CI status

## Important Guidelines

- Never force-push. Always create new commits for fixes.
- When addressing review comments, make targeted changes — do not refactor unrelated code.
- If a comment is a question rather than a change request, reply to it on the PR without modifying code.
- If CI fails more than 5 times consecutively on the same check, stop and ask the user for help rather than looping indefinitely.
- If a review comment is unclear or contradictory, ask the user before making changes.
- Keep commit messages descriptive: reference what feedback or CI failure is being addressed.
- Use `gh api` for operations not directly supported by `gh pr` subcommands.
