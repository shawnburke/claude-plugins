---
description: Create a PR and monitor it until CI passes and all comments are addressed
allowed-tools: Bash(gh:*), Bash(git:*), Read, Write, Edit, Grep, Glob, Agent
argument-hint: "[base-branch] [--ci-interval SECONDS] [--comment-interval SECONDS] [--rebase]"
---

## Preflight Check

Verify that `gh` is installed and authenticated:

```!
gh auth status 2>&1 || echo "PREFLIGHT_FAILED: gh CLI is not installed or not authenticated. Install it from https://cli.github.com/ and run 'gh auth login'."
```

If the preflight check failed, inform the user and stop. Do not proceed with any other phases.

## Context

- Current git status: !`git status`
- Current git diff (staged and unstaged): !`git diff HEAD`
- Current branch: !`git branch --show-current`
- Recent commits on this branch: !`git log --oneline -10`
- Remote tracking: !`git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null || echo "No upstream set"`

## Parse Arguments

Parse `$ARGUMENTS` for the following options. All are optional:

- **Base branch**: The first positional argument (not starting with `--`). Defaults to `main`, or `master` if `main` doesn't exist on the remote.
- **`--ci-interval SECONDS`**: How often to poll CI status, in seconds. Default: `15`.
- **`--comment-interval SECONDS`**: How often to poll for new review comments, in seconds. Default: `60`.
- **`--rebase`**: Before pushing, rebase the branch onto the latest HEAD of the base branch. Resolves conflicts if possible; if conflicts cannot be resolved automatically, stop and ask the user for help.

## Warm Up Tool Permissions

Before starting the workflow, run the following commands so the user can approve all necessary permissions up front rather than being prompted mid-workflow. Run all of these in a single message:

1. `git status` — git read access
2. `git add --dry-run .` — git staging
3. `gh pr list --limit 1` — gh PR read access
4. `gh pr checks --help 2>/dev/null || true` — gh CI status access
5. `gh api rate_limit` — gh API access
6. `gh run list --limit 1` — gh run log access

These are no-ops that establish the permission patterns (`git:*`, `gh pr:*`, `gh api:*`, `gh run:*`) so the user won't be interrupted during the actual workflow.

## Task: Create and Shepherd a Pull Request

Execute each phase in order.

### Phase 1: Create the Pull Request

1. If there are uncommitted changes, stage and commit them with an appropriate message.
2. If on `main` or `master`, create a new descriptive branch first.
3. If `--rebase` was specified, rebase onto the latest base branch:
   - `git fetch origin <base-branch>`
   - `git rebase origin/<base-branch>`
   - If there are conflicts, attempt to resolve them. If conflicts cannot be resolved, stop and ask the user for help.
4. Push the branch to origin (use `-u` to set upstream). If the rebase rewrote history, use `--force-with-lease` (never `--force`).
5. Create the PR using `gh pr create` against the base branch. Write a clear title and body summarizing the changes. Include a `## Test plan` section.
6. Display the PR URL and number.

If a PR already exists for this branch, skip creation and use the existing PR.

### Phase 2: Monitor CI and Review Comments

After creating (or finding) the PR, monitor both CI status and review comments simultaneously. Do not wait for CI to pass before checking comments — address both as they come in.

**Monitoring loop:**

On each iteration of the loop:

1. **Check CI status**: Run `gh pr view <number> --json statusCheckRollup` to get current check status.
2. **Check for review comments**: Run `gh pr view <number> --json reviews,comments` and `gh api repos/<owner>/<repo>/pulls/<number>/comments` to find unaddressed review comments.
3. **Take action on whatever needs attention:**

   **If there are unaddressed review comments:**
   - Read and understand each comment.
   - If a comment requests a code change, make the change, stage, commit (referencing the feedback), and push.
   - If a comment is a question or discussion, reply to it on the PR using `gh pr comment` or `gh api` to reply to the specific review comment.

   **If CI has failed:**
   - Identify which check(s) failed: `gh pr checks <number>`.
   - Fetch logs for the failed run: `gh run view <run-id> --log-failed` (get the run ID from checks output).
   - Analyze the failure and understand the root cause.
   - Fix the issue in the code.
   - Stage, commit (with a message describing what CI failure is being fixed), and push.

   **If both comments and CI failures exist**, address the review comments first (since the push to fix comments will trigger a new CI run anyway, and the CI failure may be related to the requested changes).

4. **After pushing any changes**, sleep for the CI interval and loop back to step 1 to monitor the new run.
5. **If nothing needs attention** (CI is still pending, no new comments), sleep for the CI interval and check again. Time out after 30 minutes of no progress.

If CI fails more than 5 times consecutively on the same check, stop and ask the user for help.

**Exit condition:** CI is green AND there are no unaddressed review comments. Proceed to Phase 3.

### Phase 3: Completion

CI is green and all comments are addressed. Display a summary:

- PR URL
- Number of CI fix iterations performed (0 if CI passed on first try)
- Number of review comments addressed (0 if none)
- Final status: ready for review / ready to merge

The workflow is now complete. If new comments arrive later, the user can run `/manage-pr` again.

## Important Guidelines

- Never force-push unless `--rebase` was specified, in which case use `--force-with-lease` (never `--force`) only after a rebase.
- When addressing review comments, make targeted changes — do not refactor unrelated code.
- If a review comment is unclear or contradictory, ask the user before making changes.
- Keep commit messages descriptive: reference what CI failure or feedback is being addressed.
- Use `gh api` for operations not directly supported by `gh pr` subcommands (e.g., replying to specific inline comments).
