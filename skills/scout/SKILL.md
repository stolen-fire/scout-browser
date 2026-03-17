---
name: scout
description: "Use when the user asks to automate a website, scout a page, explore page structure, build an automation script, navigate a portal, log into a site, inspect iframes or shadow DOM, or download files from a web application. Trigger phrases include scout, automate this site, explore this page, figure out how this works, what does this page look like, navigate to, open a browser, browse this site, find elements, find the button."
---

You have access to a live browser via the `scout` MCP server. It launches a browser session (Botasaurus) and lets you interactively explore websites through Chrome DevTools Protocol.

## Core Workflow: Scout -> Find -> Act

1. **Launch** — `launch_session` with the target URL (headless=false so the user can observe)
2. **Scout** — `scout_page_tool` for a compact page overview (~200 tokens): metadata, iframes, shadow DOM, element counts
3. **Find** — `find_elements` to search by text, type, or CSS selector
4. **Act** — `execute_action_tool` for one interaction (click, type, select, navigate)
5. **Scout again** — See what changed after the action
6. **Repeat** until the workflow is complete

## Key Tools

| Tool | Purpose |
|------|---------|
| `launch_session` | Open browser at a URL |
| `scout_page_tool` | Compact page structure report |
| `find_elements` | Search for interactive elements |
| `execute_action_tool` | Click, type, select, navigate, scroll |
| `fill_secret` | Type credentials from .env (never exposed to AI) |
| `execute_javascript` | Run arbitrary JS in page context |
| `take_screenshot_tool` | Visual capture for debugging |
| `inspect_element_tool` | Deep element inspection (visibility, shadow DOM, overlays) |
| `monitor_network` | Intercept XHR/fetch calls to discover APIs |
| `close_session` | Release the browser |

## When Clicks Fail

If `execute_action_tool` returns a `warning` on a click:
1. Use `inspect_element_tool` to check visibility, overlays, and shadow DOM context
2. Use `take_screenshot_tool` for visual confirmation
3. Try `execute_javascript` to dispatch a JS click: `document.querySelector('selector').click()`

## Session Rules

- **Login once.** Repeated logins trigger CAPTCHAs and account locks.
- **Scout after every action.** The DOM may have changed entirely.
- **Never guess structure.** Always scout before generating automation code.

## Script Generation

When composing botasaurus-driver scripts:
- Use **exact selectors** from `find_elements`
- Include **iframe context switching** where elements live in iframes
- **Parameterize credentials** — use env vars, never hardcode
- Realistic browser behavior is automatic — `Driver()` handles this out of the box
