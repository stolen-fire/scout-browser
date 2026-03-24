---
name: validate-plugin
description: Validate plugin structure — checks version sync across plugin.json, marketplace.json, and README badge; validates skill frontmatter; verifies hooks.json schema
disable-model-invocation: false
---

# Validate Plugin

Run all validation checks and report results. Fix any issues found.

## Checks

### 1. Version Sync

Read the version from these 3 sources and verify they match:
- `.claude-plugin/plugin.json` → `"version"` field
- `.claude-plugin/marketplace.json` → `plugins[0].version` field
- `README.md` → badge URL pattern `version-X.Y.Z-blue`

If any versions differ, report which files are out of sync and offer to fix them.

### 2. Plugin Manifest Validation

Verify `.claude-plugin/plugin.json` contains all required fields:
- `name`, `version`, `description`, `license`, `repository`, `author`, `category`, `keywords`

Verify `version` is valid semver (X.Y.Z format).

### 3. Marketplace Metadata Validation

Verify `.claude-plugin/marketplace.json`:
- Has `name`, `owner.name`, and `plugins` array
- `plugins[0].version` matches plugin.json version
- `plugins[0].source` is a valid path

### 4. Skill Frontmatter Validation

For every `skills/*/SKILL.md` file:
- Verify YAML frontmatter exists (between `---` delimiters)
- Verify `name` field is present and matches directory name
- Verify `description` field is present and non-empty

### 5. Hooks Validation

Verify `hooks/hooks.json`:
- Is valid JSON
- Has top-level `hooks` object
- Each key is a valid event name (SessionStart, SessionEnd, PreToolUse, PostToolUse, Stop, SubagentStop, UserPromptSubmit, PreCompact, Notification)
- Each entry has `matcher` and `hooks` array
- Each hook has `type` and `command`

## Output Format

Report results as a checklist:
```
Plugin Validation Results:
[PASS] Version sync: all files at vX.Y.Z
[PASS] plugin.json: all required fields present
[PASS] marketplace.json: valid, version matches
[PASS] Skills: N skills validated
[PASS] hooks.json: valid schema

(or [FAIL] with description of what's wrong)
```

If any checks fail, offer to fix the issues automatically.
