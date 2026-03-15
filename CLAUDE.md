# Claude Code Guidelines for claude-plugins

## Versioning

When bumping a plugin version, it **must** be updated in both places:

1. `<plugin-name>/.claude-plugin/plugin.json` — the `"version"` field
2. `.claude-plugin/marketplace.json` — the `"version"` field for that plugin's entry

Both must match. If only one is updated, users won't see the new version when they run `/plugin update`.
