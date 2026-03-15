# claude-plugins

Claude Code plugins by shawnburke.

## Installation

Inside a Claude Code session, add the marketplace:

```
/plugin marketplace add shawnburke/claude-plugins
```

Then install a plugin:

```
/plugin install manage-pr@shawnburke-plugins
```

Or run `/plugin` and use the interactive Marketplaces tab to browse and install.

## Plugins

### manage-pr

Create a PR and shepherd it to completion — monitors CI, fixes failures, and addresses review comments.

#### What it does

`/manage-pr` automates the full lifecycle of getting a PR to a mergeable state:

1. **Creates the PR** — commits any pending changes, creates a branch if needed, pushes, and opens a PR via `gh pr create`. If a PR already exists for the branch, it uses that one.
2. **Monitors CI** — polls check status (default: every 15s). When a check fails, it fetches the logs, diagnoses the root cause, fixes the code, pushes a new commit, and waits for CI again. Gives up after 5 consecutive failures on the same check and asks for help.
3. **Addresses review comments** — once CI is green, checks for unaddressed review comments. For each comment requesting a change, it modifies the code, pushes a fix, and replies to the comment on the PR. For questions or discussion, it replies without code changes. After pushing fixes, it loops back to wait for CI again.
4. **Completes** — when CI is green and all comments are addressed, it prints a summary and stops.

If new comments arrive after completion, just run `/manage-pr` again.

#### Usage

```
/manage-pr                                          # PR against main, default intervals
/manage-pr develop                                  # PR against develop
/manage-pr --ci-interval 30                         # poll CI every 30s instead of 15s
/manage-pr --comment-interval 120                   # poll comments every 2 minutes
/manage-pr --rebase                                        # rebase onto latest main before pushing
/manage-pr develop --rebase --ci-interval 10                # combine options
```

#### Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `base-branch` | `main` (or `master`) | Target branch for the PR |
| `--ci-interval SECONDS` | `15` | How often to poll CI status |
| `--comment-interval SECONDS` | `60` | How often to poll for review comments |
| `--rebase` | off | Rebase onto latest base branch HEAD before pushing |

#### Requirements

- [GitHub CLI](https://cli.github.com/) (`gh`) installed and authenticated
- A git repository with a remote

#### Safety

- Never force-pushes unless `--rebase` is used, in which case uses `--force-with-lease`
- Stops after 5 consecutive CI failures on the same check and asks for help
- Asks the user before acting on unclear or contradictory review comments
- Replies to review comments when addressing them so reviewers can see what changed
