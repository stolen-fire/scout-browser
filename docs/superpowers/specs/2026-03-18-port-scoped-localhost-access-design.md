# Port-Scoped Localhost Access for Scout

**Date:** 2026-03-18
**Status:** Draft
**Repos:** scout-browser (plugin), scout-mcp (MCP server)

## Problem

Scout blocks all localhost/loopback navigation by default as SSRF protection. Developers working on local web applications must set the `SCOUT_ALLOW_LOCALHOST` environment variable before starting the MCP server — this is friction-heavy and grants access to all loopback ports, offering no protection if a malicious page attempts SSRF to other local services.

## Solution

Add an `allow_localhost_port` parameter to the `launch_session` MCP tool that accepts an optional port number (1-65535). When set, only that specific port on loopback addresses is permitted. The `/scout` command in scout-browser detects localhost URLs and passes the port through automatically.

## Security Properties

- **Default unchanged.** No parameter = all localhost blocked.
- **Port-scoped.** `allow_localhost_port=3000` permits only `localhost:3000`, not `localhost:6379` or `localhost:9200`.
- **Per-session.** The permission dies with the session — no persistence across sessions.
- **Env var preserved.** `SCOUT_ALLOW_LOCALHOST=1` continues to work as an all-ports override for CI/testing environments. When the env var is set, it takes precedence (broadest access wins).
- **Cloud metadata always blocked.** `169.254.169.254` and friends are never permitted regardless of any flag.
- **Known limitation: browser-initiated redirects.** A page at `localhost:3000` could redirect the browser (via HTTP 3xx, `window.location`, or meta refresh) to `localhost:6379`. Port-scoping validates at MCP tool call time, but browser-internal redirects bypass the MCP layer. This is an inherent limitation of the architecture — the existing `NavigationGuard` (extension mode) has the same characteristic at the domain level. Future work could add CDP-level navigation interception, but that is out of scope.

## Design

### Data Flow

```
User: /scout http://localhost:3000
  → /scout command instructs Claude to pass allow_localhost_port=3000 to launch_session
    → launch_session stores allowed port on BrowserSession
      → validate_url() checks: loopback + port 3000? ✅ allow
      → validate_url() checks: loopback + port 6379? ❌ block (wrong port)
```

### Changes by File

#### scout-mcp (MCP server)

**1. `src/scout/validation.py`**

Add `allow_localhost_port` parameter to `validate_url` only. `_is_blocked_host` stays unchanged — port logic lives entirely in `validate_url` since it has access to the parsed URL.

```python
def validate_url(url: str, *, allow_localhost: bool = False, allow_localhost_port: int | None = None) -> None:
    if not url:
        return

    parsed = urlparse(url)

    if parsed.scheme and parsed.scheme.lower() not in _ALLOWED_SCHEMES:
        raise ValueError(f"Only http and https URLs are allowed, got scheme: {parsed.scheme}")

    hostname = parsed.hostname
    if not hostname:
        return

    # Determine effective localhost permission.
    # Env var (allow_localhost=True) grants all-ports access and takes precedence.
    # Port-scoped access only permits the specific port.
    effective_allow = allow_localhost
    if not effective_allow and allow_localhost_port is not None:
        # Resolve default port: urlparse returns None for implicit ports
        parsed_port = parsed.port
        if parsed_port is None:
            parsed_port = 443 if parsed.scheme == "https" else 80
        effective_allow = (parsed_port == allow_localhost_port)

    if _is_blocked_host(hostname, allow_localhost=effective_allow):
        raise ValueError(f"Blocked URL host: {hostname}")
```

Port validation (added at top of function or in `launch_session`):
```python
if allow_localhost_port is not None:
    if not isinstance(allow_localhost_port, int) or allow_localhost_port < 1 or allow_localhost_port > 65535:
        raise ValueError(f"allow_localhost_port must be 1-65535, got: {allow_localhost_port}")
```

**2. `src/scout/session.py`**

- Add `allow_localhost_port: int | None = None` parameter to `BrowserSession.__init__`.
- Store as `self._allow_localhost_port`.
- In `_launch_browser()` (line 154) and `_launch_extension()` (line 217), replace the env var check:
  ```python
  # Before:
  validate_url(url, allow_localhost=os.environ.get("SCOUT_ALLOW_LOCALHOST", "").lower() in ("1", "true", "yes"))

  # After:
  _env_allow = os.environ.get("SCOUT_ALLOW_LOCALHOST", "").lower() in ("1", "true", "yes")
  validate_url(url, allow_localhost=_env_allow, allow_localhost_port=self._allow_localhost_port)
  ```

**3. `src/scout/actions.py`**

Add `allow_localhost_port: int | None = None` parameter to `execute_action()`:

```python
def execute_action(
    driver: Driver,
    action: str,
    selector: str | None = None,
    value: str | None = None,
    frame_context: str | None = None,
    wait_after: int = 500,
    allow_localhost_port: int | None = None,  # new
) -> tuple[ActionResult, ActionRecord]:
```

In the `navigate` action case (line 118-122):
```python
case "navigate":
    _require(value, "value (URL) required for navigate")
    _env_allow = os.environ.get("SCOUT_ALLOW_LOCALHOST", "").lower() in ("1", "true", "yes")
    validate_url(value, allow_localhost=_env_allow, allow_localhost_port=allow_localhost_port)
    driver.get(value)
    action_desc = f"Navigated to '{value}'"
```

**4. `src/scout/server.py`**

- Add `allow_localhost_port: int | None = None` parameter to `launch_session` tool (line 147).
- Docstring: `"Optional port number to allow localhost/loopback navigation on (1-65535). Example: 3000 permits http://localhost:3000 only. If omitted, localhost is blocked. The SCOUT_ALLOW_LOCALHOST env var overrides this to allow all ports."`
- Pass to `BrowserSession` constructor (line 219): `allow_localhost_port=allow_localhost_port`.
- In `execute_action_tool` where it calls `execute_action()`, pass the session's port through:
  ```python
  result, record = await asyncio.to_thread(
      execute_action,
      session.driver,
      action,
      selector,
      value,
      frame_context,
      wait_after,
      allow_localhost_port=session._allow_localhost_port,
  )
  ```

#### scout-browser (plugin)

**5. `commands/scout.md`**

Add instruction after step 1:

> If the URL targets localhost, 127.0.0.1, `[::1]`, or any other loopback address, extract the port number from the URL and pass it as `allow_localhost_port=<port>` to `launch_session`. If no port is specified in the URL, use port 80 for http or 443 for https.

### Interaction with Existing Features

- **`SCOUT_ALLOW_LOCALHOST` env var**: Acts as an all-ports override. When set, `allow_localhost=True` takes precedence over port scoping (broadest access wins). Both mechanisms coexist — the env var is for CI/testing, the parameter is for interactive use.
- **`NavigationGuard`** (extension mode): Operates at the domain level, independent of this change. Both guards apply — a URL must pass both the loopback check and the navigation guard.
- **`allow_navigation` MCP tool**: Grants one-time cross-origin permits. Does not affect loopback validation.

### Edge Cases

- **No port in localhost URL**: `http://localhost` → resolved to port 80. `https://localhost` → resolved to port 443. Both sides (command and validation) use the same defaults.
- **Explicit default port**: `http://localhost:80` — `urlparse` returns `port=80`, matches `allow_localhost_port=80`. Works correctly.
- **Port out of range**: Values outside 1-65535 are rejected with a ValueError.
- **Multiple sessions with different ports**: Each session stores its own `_allow_localhost_port` independently.
- **Env var + parameter both set**: Env var wins (broadest access). The `allow_localhost` bool is checked first; port-scoping only applies when `allow_localhost` is False.
- **IPv6 loopback**: `http://[::1]:3000/` — `urlparse` extracts hostname `::1` and port `3000`. `_is_blocked_host` recognizes `::1` as loopback via `ipaddress.ip_address()`. Port matching works normally.

## What We Are NOT Building

- No `/localhost` toggle command
- No persistent settings across sessions
- No mid-session port changes
- No multi-port allowlists (one port per session; use env var for all-ports access)
- No CDP-level redirect interception (see known limitation above)

## Testing

Unit tests for `validate_url` with `allow_localhost_port` parameter:
- `localhost:3000` with `allow_localhost_port=3000` → allowed
- `localhost:6379` with `allow_localhost_port=3000` → blocked
- `localhost:3000` with no parameter → blocked (default behavior)
- `http://localhost` (no port) with `allow_localhost_port=80` → allowed
- `https://localhost` (no port) with `allow_localhost_port=443` → allowed
- `http://localhost:80` (explicit) with `allow_localhost_port=80` → allowed
- `127.0.0.1:3000` with `allow_localhost_port=3000` → allowed
- `[::1]:3000` with `allow_localhost_port=3000` → allowed
- Env var `SCOUT_ALLOW_LOCALHOST=1` overrides port restriction → all ports allowed
- Cloud metadata still blocked with any flag combination
- Port 0, -1, 70000 → rejected with ValueError

Integration test:
- Launch session with `allow_localhost_port=3000`, verify navigation to `localhost:3000` succeeds and `localhost:3001` fails
- Document redirect behavior: page at `localhost:3000` redirecting to `localhost:6379` is NOT blocked (known limitation)
