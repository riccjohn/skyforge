# Contributing

## Changing the plugin

1. Skills live under `plugins/skyforge/skills/<name>/`, with `SKILL.md` as the entry point. A skill's folder name becomes its command: `skills/start/` is `/skyforge:start`.
2. Pre-approve only read-only commands in a skill's `allowed-tools`. Anything that writes, starts sessions, or changes remote state stays un-pre-approved.
3. Update the README when you add a prerequisite or change how a command is used.
4. Bump `version` in `plugins/skyforge/.claude-plugin/plugin.json` using semver: patch for fixes, minor for new behaviour, major for breaking changes.

> The `version` in `.claude-plugin/marketplace.json` tracks the marketplace catalog itself (adding or removing plugins, renaming the marketplace). It does not change when the plugin's version does.

## Testing locally

```shell
# If you installed the remote marketplace, remove it first:
claude plugin marketplace remove skyforge

claude plugin marketplace add ~/path/to/skyforge
claude plugin install skyforge@skyforge
```

Start a new Claude Code session inside Herdr to pick up the changes.
