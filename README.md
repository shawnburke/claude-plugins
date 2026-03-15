# claude-plugins

Claude Code plugins by shawnburke.

## Installation

Add the marketplace:

```bash
claude marketplace add https://raw.githubusercontent.com/shawnburke/claude-plugins/main/marketplace.json
```

Then install any plugin:

```bash
claude plugin install manage-pr@shawnburke-plugins
```

## Plugins

| Plugin | Description |
|--------|-------------|
| [manage-pr](./manage-pr/) | Create a PR and shepherd it to completion — monitors CI, fixes failures, addresses review comments |
