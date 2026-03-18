# Scout — Browser Automation for Claude Code

[![Version](https://img.shields.io/badge/version-1.0.0-blue)](https://github.com/stemado/scout-browser/releases)
[![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![MCP Server](https://img.shields.io/npm/v/@stemado/scout-mcp?label=scout-mcp)](https://www.npmjs.com/package/@stemado/scout-mcp)

Give Claude Code a browser. Scout page structure, interact with websites, and export replayable automations.

Scout reports cost **~200 tokens** vs ~124,000 for screenshot-based approaches — making browsing cheap enough to be casual.

## Why Scout over Playwright?

Playwright is built for developers writing test scripts. Scout is built for **Claude to browse the web directly** — and hand you a finished automation when it's done.

| | Scout | Playwright |
|---|---|---|
| **Token cost per page** | ~200 (structured report) | ~1,600+ (screenshot) |
| **Sites that block automation** | Works — realistic browser fingerprinting | Often blocked |
| **Credential handling** | `fill_secret` — credentials stay in `.env`, never sent to AI | No built-in credential isolation |
| **Output** | Standalone Python scripts you can schedule | Requires Playwright runtime |
| **Claude integration** | 3 commands + auto-trigger skill + system prompt | MCP wrapper only |

## Prerequisites

- **Node.js** (for MCP server transport)
- **Google Chrome** (for browser automation)
- **Python 3.11+** (for the MCP server runtime)

Scout checks your environment automatically on startup and reports any missing dependencies.

## Install

From the Claude Code plugin marketplace:

```
/plugin install scout@claude-plugins-official
```

Or install directly from GitHub:

```
/plugin marketplace add stemado/scout-browser
/plugin install scout@stemado-scout-browser
```

Or load locally for development:

```bash
claude --plugin-dir ./scout-browser
```

## How It Works

Claude **automatically knows how to use Scout**. The plugin includes a browsing skill that triggers whenever you mention websites, automation, page structure, or browsing — no slash command required. Just say what you need:

> "Log into our vendor portal and download this month's invoice."

> "What does the signup flow look like on example.com?"

> "Automate checking our benefits platform for open enrollment status."

Claude launches a browser, scouts the page, navigates step by step, and reports what it finds.

## Commands

### `/scout:scout <url>`

Open a browser and scout a website's page structure. Returns a compact overview of metadata, iframes, shadow DOM boundaries, and interactive elements. The session stays alive for follow-up interactions — click buttons, fill forms, navigate pages.

### `/scout:export-workflow [name]`

Export the current browser session as a replayable automation package:

- **Python script** — standalone botasaurus-driver script with human-like timing
- **requirements.txt** — pip dependencies
- **.env.example** — credential template (only when credentials are detected)

The exported script runs independently — no Claude Code or MCP server required.

### `/scout:schedule`

Schedule exported workflows to run automatically using your OS task manager (Windows Task Scheduler, macOS launchd, or Linux cron).

```
/scout:schedule                              # list all schedules
/scout:schedule enrollment every weekday at 9am   # create a schedule
/scout:schedule delete enrollment            # remove a schedule
```

## Use Cases

### For Developers

- **API discovery** — Scout's network monitor intercepts XHR/fetch calls in Chrome DevTools, exposing the underlying APIs behind any web dashboard. No export button? No problem — find the endpoint, capture the payload shape, and hit it directly.
- **Enterprise portal automation** — Navigate SSO login flows, multi-step forms, and nested iframe structures that break standard automation tools. Export a standalone script that runs without Claude.
- **Credential-safe CI workflows** — Export automations that read credentials from `.env` files at runtime. Secrets never touch the AI model or session logs.

### For Marketing and Research

- **Competitive intelligence** — Scout a competitor's pricing page, product catalog, or feature list. The network monitor reveals the API calls that populate dynamic content — structured JSON data you can capture and analyze, even when the site has no public API.
- **Ad platform data extraction** — Pull performance metrics from dashboards (Google Ads, Meta Business Suite, LinkedIn Campaign Manager) that don't offer CSV export for the exact view you need. Scout sees the same API responses your browser sees.
- **UX research and site audits** — Scout any website to get an instant structural map: how many forms, buttons, links, iframes, and interactive elements exist on each page. Compare competitor signup flows side by side without manually clicking through each one.
- **SEO and content analysis** — Crawl page metadata, heading structures, and link patterns across a site. Scout reports give you the DOM structure at a fraction of the token cost of screenshots, so you can analyze dozens of pages in a single conversation.
- **Social listening and review aggregation** — Navigate review sites, forums, or social platforms to extract structured data from pages that require login or infinite scroll. Export the workflow to run on a schedule and capture changes over time.

## The Lifecycle

```
Scout  →  Interact  →  Export  →  Schedule
```

1. **Scout** a page to understand its structure
2. **Interact** — click, type, navigate through the workflow
3. **Export** the session as a standalone Python script
4. **Schedule** it to run on a recurring basis

## Security

- **Credential isolation** — `fill_secret` injects credentials from `.env` files directly into form fields. Values are never sent to the AI model, never logged, never included in session history.
- **Realistic browser sessions** — Botasaurus provides human-like browser fingerprinting and behavior, reducing CAPTCHAs and bot detection without relying on detection-evasion techniques.
- **Untrusted content handling** — All data from websites is treated as untrusted. Invisible characters are stripped and content boundaries are enforced before data reaches the model.

## MCP Server

This plugin uses [scout-mcp-server](https://pypi.org/project/scout-mcp-server/) ([`@stemado/scout-mcp`](https://www.npmjs.com/package/@stemado/scout-mcp) on npm) for browser automation. The MCP server is installed automatically when the plugin loads.

## License

MIT
