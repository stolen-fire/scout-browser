---
description: Rules for developing and maintaining the Scout plugin
globs: ["**/*.json", "**/*.md", "**/*.sh"]
---

# Plugin Development Rules

## Skill Files (skills/*/SKILL.md)

Every SKILL.md must have YAML frontmatter with at minimum:
- `name` — skill identifier (matches directory name)
- `description` — what the skill does and when to trigger it (used for auto-matching)

Optional frontmatter: `allowed-tools`, `disable-model-invocation`, `model`.

## Plugin Manifest (.claude-plugin/plugin.json)

Required fields: `name`, `version`, `description`, `license`, `repository`, `author`, `category`, `keywords`.

Version must be valid semver (X.Y.Z).

## Marketplace Metadata (.claude-plugin/marketplace.json)

The `plugins[0].version` must match `plugin.json` version exactly. The `source` field points to the plugin root (currently `"./"`).

## Hooks (hooks/hooks.json)

Uses the Claude Code plugin hooks schema:
```json
{
  "hooks": {
    "<EventName>": [{ "matcher": "<pattern>", "hooks": [{ "type": "command", "command": "...", "timeout": <ms> }] }]
  }
}
```

Valid events: SessionStart, SessionEnd, PreToolUse, PostToolUse, Stop, SubagentStop, UserPromptSubmit, PreCompact, Notification.

## README Badge

The version badge in README.md follows this pattern:
```
https://img.shields.io/badge/version-X.Y.Z-blue
```

Must match the version in plugin.json.

## Shell Scripts

All shell scripts in `hooks/` must be POSIX-compatible (no bash arrays, no bash-specific syntax) for macOS bash 3.2 compatibility.
