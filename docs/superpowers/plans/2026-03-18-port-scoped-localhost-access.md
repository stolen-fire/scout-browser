# Port-Scoped Localhost Access Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Allow developers to scout localhost apps by adding a port-scoped `allow_localhost_port` parameter to `launch_session`, with the `/scout` command auto-detecting localhost URLs.

**Architecture:** The validation layer (`validate_url`) gains port-aware localhost checking. The parameter flows from `launch_session` → `BrowserSession` → all three `validate_url` call sites (two in session.py, one in actions.py). The env var `SCOUT_ALLOW_LOCALHOST` remains as an all-ports override.

**Tech Stack:** Python 3.11+, pytest, botasaurus-driver

**Spec:** `D:\Projects\scout-browser\docs\superpowers\specs\2026-03-18-port-scoped-localhost-access-design.md`

**Two repos involved:**
- `D:\Projects\scout-mcp` — MCP server (Tasks 1-4)
- `D:\Projects\scout-browser` — Plugin (Task 5)

---

### Task 1: Port-scoped validation in `validate_url`

**Files:**
- Modify: `D:\Projects\scout-mcp\src\scout\validation.py:62-84`
- Test: `D:\Projects\scout-mcp\tests\test_validation.py`

- [ ] **Step 1: Write failing tests for port-scoped localhost**

Add a new test class to `D:\Projects\scout-mcp\tests\test_validation.py`:

```python
class TestValidateUrlLocalhostPort:
    """Port-scoped localhost access via allow_localhost_port parameter."""

    def test_allows_localhost_on_matching_port(self):
        validate_url("http://localhost:3000/app", allow_localhost_port=3000)

    def test_blocks_localhost_on_different_port(self):
        with pytest.raises(ValueError, match="Blocked URL host"):
            validate_url("http://localhost:6379", allow_localhost_port=3000)

    def test_blocks_localhost_with_no_parameter(self):
        with pytest.raises(ValueError, match="Blocked URL host"):
            validate_url("http://localhost:3000")

    def test_allows_127001_on_matching_port(self):
        validate_url("http://127.0.0.1:3000/path", allow_localhost_port=3000)

    def test_blocks_127001_on_different_port(self):
        with pytest.raises(ValueError, match="Blocked URL host"):
            validate_url("http://127.0.0.1:9200", allow_localhost_port=3000)

    def test_allows_ipv6_loopback_on_matching_port(self):
        validate_url("http://[::1]:3000/", allow_localhost_port=3000)

    def test_blocks_ipv6_loopback_on_different_port(self):
        with pytest.raises(ValueError, match="Blocked URL host"):
            validate_url("http://[::1]:6379", allow_localhost_port=3000)

    def test_allows_implicit_port_80_for_http(self):
        validate_url("http://localhost", allow_localhost_port=80)

    def test_allows_implicit_port_443_for_https(self):
        validate_url("https://localhost", allow_localhost_port=443)

    def test_allows_explicit_port_80_for_http(self):
        validate_url("http://localhost:80/path", allow_localhost_port=80)

    def test_blocks_implicit_port_when_different(self):
        with pytest.raises(ValueError, match="Blocked URL host"):
            validate_url("http://localhost", allow_localhost_port=3000)

    def test_env_var_overrides_port_restriction(self, monkeypatch):
        monkeypatch.setenv("SCOUT_ALLOW_LOCALHOST", "1")
        # allow_localhost=True (from env var) should permit any port
        validate_url("http://localhost:6379", allow_localhost=True, allow_localhost_port=3000)

    def test_cloud_metadata_blocked_with_any_flags(self):
        with pytest.raises(ValueError, match="Blocked URL host"):
            validate_url("http://169.254.169.254/latest/meta-data/", allow_localhost_port=80)

    def test_rejects_port_zero(self):
        with pytest.raises(ValueError, match="allow_localhost_port must be 1-65535"):
            validate_url("http://localhost:3000", allow_localhost_port=0)

    def test_rejects_negative_port(self):
        with pytest.raises(ValueError, match="allow_localhost_port must be 1-65535"):
            validate_url("http://localhost:3000", allow_localhost_port=-1)

    def test_rejects_port_above_65535(self):
        with pytest.raises(ValueError, match="allow_localhost_port must be 1-65535"):
            validate_url("http://localhost:3000", allow_localhost_port=70000)

    def test_non_localhost_url_unaffected(self):
        validate_url("http://example.com:3000", allow_localhost_port=3000)

    def test_non_localhost_url_still_works_without_port(self):
        validate_url("http://example.com")

    def test_allows_ipv6_mapped_ipv4_loopback_on_matching_port(self):
        validate_url("http://[::ffff:127.0.0.1]:3000/", allow_localhost_port=3000)

    def test_blocks_ipv6_mapped_ipv4_loopback_on_different_port(self):
        with pytest.raises(ValueError, match="Blocked URL host"):
            validate_url("http://[::ffff:127.0.0.1]:6379", allow_localhost_port=3000)

    def test_allows_all_ports_with_allow_localhost_true(self):
        validate_url("http://localhost:6379", allow_localhost=True)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd D:\Projects\scout-mcp && python -m pytest tests/test_validation.py::TestValidateUrlLocalhostPort -v`
Expected: FAIL — `validate_url() got an unexpected keyword argument 'allow_localhost_port'`

- [ ] **Step 3: Implement port-scoped validation**

Replace `validate_url` in `D:\Projects\scout-mcp\src\scout\validation.py:62-84` with:

```python
def validate_url(url: str, *, allow_localhost: bool = False, allow_localhost_port: int | None = None) -> None:
    """Validate a URL is safe to navigate to.

    Raises ValueError for non-http(s) schemes or blocked hosts.
    Allows empty strings (callers handle those as no-ops).

    Args:
        url: The URL to validate.
        allow_localhost: If True, permit all loopback addresses (env var override).
        allow_localhost_port: If set, permit loopback addresses only on this port (1-65535).
    """
    if not url:
        return

    if allow_localhost_port is not None:
        if not isinstance(allow_localhost_port, int) or allow_localhost_port < 1 or allow_localhost_port > 65535:
            raise ValueError(f"allow_localhost_port must be 1-65535, got: {allow_localhost_port}")

    parsed = urlparse(url)

    # Scheme allowlist: only http and https are permitted.
    # Empty scheme (e.g. bare "example.com") is allowed — the browser normalizes it.
    if parsed.scheme and parsed.scheme.lower() not in _ALLOWED_SCHEMES:
        raise ValueError(f"Only http and https URLs are allowed, got scheme: {parsed.scheme}")

    hostname = parsed.hostname
    if not hostname:
        return

    # Determine effective localhost permission.
    # allow_localhost=True (env var) grants all-ports access and takes precedence.
    # Port-scoped access only permits the specific port.
    effective_allow = allow_localhost
    if not effective_allow and allow_localhost_port is not None:
        parsed_port = parsed.port
        if parsed_port is None:
            parsed_port = 443 if parsed.scheme == "https" else 80
        effective_allow = (parsed_port == allow_localhost_port)

    if _is_blocked_host(hostname, allow_localhost=effective_allow):
        raise ValueError(f"Blocked URL host: {hostname}")
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd D:\Projects\scout-mcp && python -m pytest tests/test_validation.py -v`
Expected: ALL PASS (both existing `TestValidateUrl` and new `TestValidateUrlLocalhostPort`)

- [ ] **Step 5: Commit**

```bash
cd D:\Projects\scout-mcp
git add src/scout/validation.py tests/test_validation.py
git commit -m "feat: add port-scoped localhost validation to validate_url"
```

---

### Task 2: Thread `allow_localhost_port` through `BrowserSession`

**Files:**
- Modify: `D:\Projects\scout-mcp\src\scout\session.py:35-45` (constructor), `:154` (_launch_browser), `:217` (_launch_extension)

- [ ] **Step 1: Add parameter to `BrowserSession.__init__`**

In `D:\Projects\scout-mcp\src\scout\session.py`, add `allow_localhost_port: int | None = None` to the constructor signature (after `profile`), and store it:

```python
    def __init__(
        self,
        headless: bool = False,
        proxy: str | None = None,
        download_dir: str = os.path.join(os.path.expanduser("~"), ".scout", "downloads"),
        user_agent: str | None = None,
        window_size: tuple[int, int] | None = None,
        connection_mode: ConnectionMode = ConnectionMode.LAUNCH,
        allowed_domains: list[str] | None = None,
        profile: str | None = None,
        allow_localhost_port: int | None = None,
    ) -> None:
```

Add after line 74 (`self._profile = profile`):

```python
        self._allow_localhost_port = allow_localhost_port
```

- [ ] **Step 2: Update `_launch_browser` validate_url call**

Replace line 154 in `D:\Projects\scout-mcp\src\scout\session.py`:

```python
                validate_url(url, allow_localhost=os.environ.get("SCOUT_ALLOW_LOCALHOST", "").lower() in ("1", "true", "yes"))
```

With:

```python
                _env_allow = os.environ.get("SCOUT_ALLOW_LOCALHOST", "").lower() in ("1", "true", "yes")
                validate_url(url, allow_localhost=_env_allow, allow_localhost_port=self._allow_localhost_port)
```

- [ ] **Step 3: Update `_launch_extension` validate_url call**

Replace line 217 in `D:\Projects\scout-mcp\src\scout\session.py`:

```python
            validate_url(url, allow_localhost=os.environ.get("SCOUT_ALLOW_LOCALHOST", "").lower() in ("1", "true", "yes"))
```

With:

```python
            _env_allow = os.environ.get("SCOUT_ALLOW_LOCALHOST", "").lower() in ("1", "true", "yes")
            validate_url(url, allow_localhost=_env_allow, allow_localhost_port=self._allow_localhost_port)
```

- [ ] **Step 4: Run existing tests to verify no regressions**

Run: `cd D:\Projects\scout-mcp && python -m pytest tests/ -v --timeout=30`
Expected: ALL PASS — the new parameter defaults to `None`, preserving existing behavior.

- [ ] **Step 5: Commit**

```bash
cd D:\Projects\scout-mcp
git add src/scout/session.py
git commit -m "feat: thread allow_localhost_port through BrowserSession"
```

---

### Task 3: Thread `allow_localhost_port` through `execute_action`

**Files:**
- Modify: `D:\Projects\scout-mcp\src\scout\actions.py:41-48` (signature), `:118-122` (navigate case)
- Modify: `D:\Projects\scout-mcp\src\scout\server.py:490-498` (call site)

- [ ] **Step 1: Add parameter to `execute_action` signature**

In `D:\Projects\scout-mcp\src\scout\actions.py:41-48`, change:

```python
def execute_action(
    driver: Driver,
    action: str,
    selector: str | None = None,
    value: str | None = None,
    frame_context: str | None = None,
    wait_after: int = 500,
) -> tuple[ActionResult, ActionRecord]:
```

To:

```python
def execute_action(
    driver: Driver,
    action: str,
    selector: str | None = None,
    value: str | None = None,
    frame_context: str | None = None,
    wait_after: int = 500,
    allow_localhost_port: int | None = None,
) -> tuple[ActionResult, ActionRecord]:
```

- [ ] **Step 2: Update the navigate action case**

In `D:\Projects\scout-mcp\src\scout\actions.py:118-122`, change:

```python
            case "navigate":
                _require(value, "value (URL) required for navigate")
                validate_url(value, allow_localhost=os.environ.get("SCOUT_ALLOW_LOCALHOST", "").lower() in ("1", "true", "yes"))
                driver.get(value)
                action_desc = f"Navigated to '{value}'"
```

To:

```python
            case "navigate":
                _require(value, "value (URL) required for navigate")
                _env_allow = os.environ.get("SCOUT_ALLOW_LOCALHOST", "").lower() in ("1", "true", "yes")
                validate_url(value, allow_localhost=_env_allow, allow_localhost_port=allow_localhost_port)
                driver.get(value)
                action_desc = f"Navigated to '{value}'"
```

- [ ] **Step 3: Update the call site in `server.py`**

In `D:\Projects\scout-mcp\src\scout\server.py:490-498`, change:

```python
    result, record = await asyncio.to_thread(
        execute_action,
        session.driver,
        action,
        selector,
        value,
        frame_context,
        wait_after,
    )
```

To:

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

- [ ] **Step 4: Write integration test for navigate action port threading**

Add to `D:\Projects\scout-mcp\tests\test_validation.py`:

```python
class TestExecuteActionNavigateLocalhostPort:
    """Verify execute_action threads allow_localhost_port to validate_url for navigate."""

    def test_navigate_blocks_localhost_without_port(self):
        """Navigate to localhost is blocked when allow_localhost_port is None."""
        from unittest.mock import MagicMock
        from scout.actions import execute_action

        driver = MagicMock()
        result, _record = execute_action(driver, "navigate", value="http://localhost:3000")
        assert not result.success
        assert "Blocked URL host" in result.error

    def test_navigate_allows_localhost_with_matching_port(self):
        """Navigate to localhost succeeds when allow_localhost_port matches."""
        from unittest.mock import MagicMock
        from scout.actions import execute_action

        driver = MagicMock()
        result, _record = execute_action(
            driver, "navigate", value="http://localhost:3000", allow_localhost_port=3000
        )
        assert result.success
        driver.get.assert_called_once_with("http://localhost:3000")

    def test_navigate_blocks_localhost_with_wrong_port(self):
        """Navigate to localhost:6379 is blocked when allow_localhost_port=3000."""
        from unittest.mock import MagicMock
        from scout.actions import execute_action

        driver = MagicMock()
        result, _record = execute_action(
            driver, "navigate", value="http://localhost:6379", allow_localhost_port=3000
        )
        assert not result.success
        assert "Blocked URL host" in result.error
```

- [ ] **Step 5: Run all tests**

Run: `cd D:\Projects\scout-mcp && python -m pytest tests/test_validation.py -v`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
cd D:\Projects\scout-mcp
git add src/scout/actions.py src/scout/server.py tests/test_validation.py
git commit -m "feat: thread allow_localhost_port through execute_action"
```

---

### Task 4: Add `allow_localhost_port` parameter to `launch_session` MCP tool

**Files:**
- Modify: `D:\Projects\scout-mcp\src\scout\server.py:147-263` (launch_session tool)

- [ ] **Step 1: Add parameter to `launch_session` signature**

In `D:\Projects\scout-mcp\src\scout\server.py:147-155`, change:

```python
async def launch_session(
    url: str | None = None,
    headless: bool = False,
    profile: str | None = None,
    proxy: str | None = None,
    download_dir: str | None = None,
    connection_mode: str = "launch",
    allowed_domains: list[str] | None = None,
    ctx: Context[ServerSession, AppContext] = None,
) -> dict:
```

To:

```python
async def launch_session(
    url: str | None = None,
    headless: bool = False,
    profile: str | None = None,
    proxy: str | None = None,
    download_dir: str | None = None,
    connection_mode: str = "launch",
    allowed_domains: list[str] | None = None,
    allow_localhost_port: int | None = None,
    ctx: Context[ServerSession, AppContext] = None,
) -> dict:
```

- [ ] **Step 2: Add docstring line**

Add to the `launch_session` docstring (after the `allowed_domains` description):

```
        allow_localhost_port: Optional port number (1-65535) to allow localhost/loopback
                             navigation on. Example: 3000 permits http://localhost:3000 only.
                             If omitted, localhost is blocked. The SCOUT_ALLOW_LOCALHOST env var
                             overrides this to allow all ports.
```

- [ ] **Step 3: Add localhost URL hint for better error messages**

In `D:\Projects\scout-mcp\src\scout\server.py`, add a check after the `connection_mode` validation block (after the profile+extension check, before the `async with app_ctx._launch_lock` line). This detects when a caller passes a localhost URL without `allow_localhost_port` and returns a helpful hint instead of a generic "Blocked URL host" error:

```python
    # Hint: detect localhost URL without allow_localhost_port
    if url and allow_localhost_port is None:
        from urllib.parse import urlparse as _urlparse
        _parsed = _urlparse(url)
        _host = (_parsed.hostname or "").lower()
        _env_allow = os.environ.get("SCOUT_ALLOW_LOCALHOST", "").lower() in ("1", "true", "yes")
        if not _env_allow and (_host in ("localhost",) or _host.startswith("127.") or _host == "::1"):
            _port = _parsed.port or (443 if _parsed.scheme == "https" else 80)
            return {
                "error": f"Localhost navigation is blocked by default. "
                f"To access {url}, pass allow_localhost_port={_port} to launch_session."
            }
```

- [ ] **Step 4: Pass to `BrowserSession` constructor**

In `D:\Projects\scout-mcp\src\scout\server.py:219-226`, change:

```python
        session = BrowserSession(
            headless=headless,
            proxy=proxy,
            download_dir=download_dir,
            connection_mode=mode,
            allowed_domains=allowed_domains,
            profile=profile,
        )
```

To:

```python
        session = BrowserSession(
            headless=headless,
            proxy=proxy,
            download_dir=download_dir,
            connection_mode=mode,
            allowed_domains=allowed_domains,
            profile=profile,
            allow_localhost_port=allow_localhost_port,
        )
```

- [ ] **Step 5: Run tests to verify no regressions**

Run: `cd D:\Projects\scout-mcp && python -m pytest tests/ -v --timeout=30`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
cd D:\Projects\scout-mcp
git add src/scout/server.py
git commit -m "feat: add allow_localhost_port parameter to launch_session tool"
```

---

### Task 5: Update `/scout` command to auto-detect localhost URLs

**Files:**
- Modify: `D:\Projects\scout-browser\commands\scout.md`

- [ ] **Step 1: Update the command instructions**

Replace the contents of `D:\Projects\scout-browser\commands\scout.md` with:

```markdown
---
description: Open a browser and scout a website's page structure
argument-hint: "<url>"
allowed-tools:
  - "mcp__plugin_scout_scout__*"
---

Open a browser and scout the page structure of the provided URL. Follow these steps:

1. Call `launch_session` with the URL from the user's argument. Use headed mode (headless=false) so the user can observe.

   **Localhost detection:** If the URL targets localhost, 127.0.0.1, [::1], or any other loopback address, extract the port number from the URL and pass it as `allow_localhost_port=<port>` to `launch_session`. If no explicit port is in the URL, use 80 for http or 443 for https.

2. Call `scout_page_tool` with the session_id to get a compact page overview (metadata, iframes, shadow DOM, element counts).

3. Present the scout report to the user in a clear summary:
   - Page title and URL
   - Number of iframes (noting any cross-origin)
   - Shadow DOM boundaries found
   - Element counts by type (buttons, inputs, links, etc.)

4. Ask the user what they'd like to do next:
   - Find specific elements (use `find_elements` with a query)
   - Explore further (click, type, navigate)
   - Export the session as an automation script
   - Close the session

Keep the session alive for follow-up interactions. Do NOT close it unless the user asks.

If no URL is provided, ask the user what site they want to scout.
```

- [ ] **Step 2: Verify the command renders correctly**

Read the file back and confirm the YAML frontmatter parses correctly (description, argument-hint, allowed-tools are present).

- [ ] **Step 3: Commit**

```bash
cd D:\Projects\scout-browser
git add commands/scout.md
git commit -m "feat: auto-detect localhost URLs in /scout command"
```

---

### Task 6: End-to-end smoke test

This task verifies the full integration manually since `launch_session` requires a real browser.

- [ ] **Step 1: Start a simple local HTTP server**

```bash
python -m http.server 8080
```

- [ ] **Step 2: Test via `/scout http://localhost:8080`**

In a Claude Code session with the updated plugin and MCP server:
1. Run `/scout http://localhost:8080`
2. Verify `launch_session` is called with `allow_localhost_port=8080`
3. Verify the page loads successfully
4. Try navigating to `localhost:9999` via `execute_action_tool` — verify it is blocked with "Blocked URL host"

- [ ] **Step 3: Test without the parameter (regression)**

1. Run `/scout https://example.com`
2. Verify `launch_session` is called without `allow_localhost_port`
3. Verify normal browsing works

- [ ] **Step 4: Final commit (plan complete marker)**

```bash
cd D:\Projects\scout-mcp
git commit -m "chore: port-scoped localhost access complete" --allow-empty
```
