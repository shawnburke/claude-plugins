---
description: Create a PR and monitor it until CI passes and all comments are addressed
allowed-tools: Bash(gh:*), Bash(git:*), Read, Write, Edit, Grep, Glob, Agent
argument-hint: "[base-branch] [--ci-interval SECONDS] [--comment-interval SECONDS] [--rebase]"
---

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

### Phase 2: Monitor CI Status

Poll CI checks until they all pass or a failure is detected.

**How to poll:**
Use `gh pr checks <number> --watch --fail-fast` if available. If `--watch` is not supported, poll manually by running `gh pr view <number> --json statusCheckRollup` in a bash loop, sleeping for the CI interval between checks. Time out after 30 minutes of polling.

**On CI failure:**
1. Identify which check(s) failed: `gh pr checks <number>`.
2. Fetch logs for the failed run: `gh run view <run-id> --log-failed` (get the run ID from checks output).
3. Analyze the failure and understand the root cause.
4. Fix the issue in the code.
5. Stage, commit (with a message describing what CI failure is being fixed), and push.
6. Go back to the top of Phase 2 to monitor the new run.

If CI fails more than 5 times consecutively on the same check, stop and ask the user for help.

**On CI success:** proceed to Phase 3.

### Phase 3: Check for Unaddressed Review Comments

Once CI is green, check for review comments that need attention.

1. Fetch PR reviews and inline comments:
   - `gh pr view <number> --json reviews,comments`
   - `gh api repos/<owner>/<repo>/pulls/<number>/comments` for inline review comments
2. Identify any unaddressed comments — comments requesting changes that haven't been resolved or replied to.
3. If there are unaddressed comments:
   a. Read and understand each comment.
   b. If a comment requests a code change, make the change, stage, commit (referencing the feedback), and push.
   c. If a comment is a question or discussion, reply to it on the PR using `gh pr comment` or `gh api` to reply to the specific review comment.
   d. After pushing any changes, go back to Phase 2 to wait for CI to pass again.
4. If there are no unaddressed comments, proceed to Phase 4.

### Phase 4: Completion

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
