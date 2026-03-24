# Changelog

## 1.2.0 (2026-03-23)

- Add `browse` tool documentation and routing guidance to skills
- Migrate GitHub references from `stemado` to `stolen-fire` organization

## 1.1.2 (2026-03-21)

- Migrate commands to skills (`commands/` → `skills/` directory restructure)

## 1.1.1 (2026-03-18)

- Auto-detect localhost URLs in `/scout` command
- Force latest scout-mcp version via `@latest` tag in `.mcp.json`
- Add port-scoped localhost access design spec and implementation plan

## 1.1.0 (2026-03-17)

- Add `marketplace.json` for plugin version detection
- Simplify export-workflow command structure
- Expand README with use cases, comparison table, and lifecycle diagram
- Fix repository URLs (mtsteinle → stemado)

## 1.0.0 (2026-03-17)

Initial release.

- `/scout:scout` — browse and explore websites
- `/scout:export-workflow` — export sessions as replayable Python scripts
- `/scout:schedule` — schedule workflows via OS task manager
- Scout browsing skill with automatic trigger phrases
- SessionStart hook for dependency checking (Python, Node.js, Chrome)
- MCP server integration via `npx -y @stemado/scout-mcp`
