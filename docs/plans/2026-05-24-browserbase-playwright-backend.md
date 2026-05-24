# Browserbase Playwright Backend Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Replace the fragile Browserbase `agent-browser --cdp` subprocess path with a persistent Browserbase + Playwright backend so Hermes can drive one stable Browserbase session, expose its live-view URL, and close it only on explicit or inactivity cleanup.

**Architecture:** Browserbase remains the cloud session provider. Hermes creates one keepAlive Browserbase session per browser task, connects to it through a persistent Playwright CDP client, and routes browser actions through that client instead of spawning a CLI subprocess for every command. The existing `agent-browser` path stays as fallback until the Playwright backend reaches parity.

**Tech Stack:** Python Playwright async API, Browserbase sessions API, existing Hermes `tools/browser_tool.py`, `tools/browser_providers/browserbase.py`, `tools/browser_supervisor.py`, pytest with mocks for unit tests and optional live Browserbase smoke tests.

---

## Background / Problem Statement

Browserbase live-view URLs are generated correctly, but Hermes-created Browserbase sessions currently die seconds after `browser_navigate` under the Telegram gateway path. Debugging established:

- `keepAlive: true` is present on Browserbase sessions.
- Raw CDP clean disconnects survive for minutes and then expire normally.
- Plain `agent-browser --cdp <connectUrl>` outside the gateway also survives.
- Gateway/browser-tool-created sessions die early, before `expiresAt`, with no local evidence of explicit `REQUEST_RELEASE`.

The most likely failure mode is lifecycle coupling between per-turn subprocess execution and the Browserbase CDP connection. The long-term fix should remove that class of bug by making Browserbase sessions first-class, persistent browser sessions in Hermes.

## Non-Goals

- Do not remove `agent-browser` support in the first implementation.
- Do not rewrite Firecrawl support.
- Do not require live Browserbase credentials for ordinary unit tests.
- Do not implement a full browser automation framework beyond current Hermes tool parity.
- Do not keep Browserbase sessions running forever; cleanup remains mandatory.

## Design Invariants

- One Hermes browser task maps to one Browserbase session while active.
- `bb_session_id`, `connectUrl`, and `live_view_url` are stored together in `_active_sessions`.
- Browserbase sessions are released only on:
  - explicit browser cleanup,
  - configured inactivity timeout,
  - gateway shutdown,
  - failed session creation/initialization rollback.
- Normal browser tool calls must not close the Browserbase browser or its last page.
- The same session Hermes drives is the session the user can watch via Browserbase live view.
- Browserbase live-view URLs are remote-control handles; only surface them in private channels unless the user explicitly asks otherwise.

## Recommended Approach

Use Python Playwright as the primary Browserbase control layer:

```text
browser_navigate / browser_click / browser_type / browser_snapshot / browser_console
  -> browser_tool.py dispatch layer
  -> BrowserbasePlaywrightSession for task_id
  -> Playwright chromium.connect_over_cdp(connectUrl)
  -> Browserbase keepAlive session
```

Keep `agent-browser` as fallback while parity is built:

```text
if browser.playwright_backend.enabled and provider == Browserbase:
    try Playwright backend
    on unsupported command or backend failure: fallback to existing _run_browser_command
else:
    existing _run_browser_command path
```

---

## Task 1: Add Playwright dependency detection and config flags

**Objective:** Add a guarded config surface so Playwright can be enabled without breaking installations that do not have Playwright installed.

**Files:**
- Modify: `hermes_cli/config.py`
- Modify: `tools/browser_tool.py`
- Test: `tests/tools/test_browser_playwright_config.py`

**Step 1: Write failing tests**

Create `tests/tools/test_browser_playwright_config.py`:

```python
import importlib


def test_playwright_backend_disabled_by_default(monkeypatch):
    import tools.browser_tool as bt

    monkeypatch.delenv("HERMES_BROWSER_PLAYWRIGHT_BACKEND", raising=False)
    monkeypatch.setattr(bt, "_get_config", lambda: {"browser": {}})

    assert bt._is_playwright_backend_enabled() is False


def test_playwright_backend_can_be_enabled_by_config(monkeypatch):
    import tools.browser_tool as bt

    monkeypatch.delenv("HERMES_BROWSER_PLAYWRIGHT_BACKEND", raising=False)
    monkeypatch.setattr(
        bt,
        "_get_config",
        lambda: {"browser": {"playwright_backend": {"enabled": True}}},
    )

    assert bt._is_playwright_backend_enabled() is True


def test_playwright_backend_env_override(monkeypatch):
    import tools.browser_tool as bt

    monkeypatch.setenv("HERMES_BROWSER_PLAYWRIGHT_BACKEND", "1")
    monkeypatch.setattr(bt, "_get_config", lambda: {"browser": {}})

    assert bt._is_playwright_backend_enabled() is True
```

**Step 2: Run test to verify failure**

Run:

```bash
python -m pytest tests/tools/test_browser_playwright_config.py -q -o 'addopts='
```

Expected: FAIL because `_is_playwright_backend_enabled` does not exist.

**Step 3: Implement minimal config helper**

In `tools/browser_tool.py`, add:

```python
def _is_playwright_backend_enabled() -> bool:
    raw = os.environ.get("HERMES_BROWSER_PLAYWRIGHT_BACKEND")
    if raw is not None:
        return raw.strip().lower() in {"1", "true", "yes", "on"}
    cfg = _get_config() or {}
    browser_cfg = cfg.get("browser") or {}
    pw_cfg = browser_cfg.get("playwright_backend") or {}
    return bool(pw_cfg.get("enabled", False))
```

In `hermes_cli/config.py`, add defaults:

```yaml
browser:
  playwright_backend:
    enabled: false
    heartbeat_interval_seconds: 240
```

Use the actual Python default-config structure in that file, not literal YAML if the file stores a dict.

**Step 4: Run tests**

```bash
python -m pytest tests/tools/test_browser_playwright_config.py -q -o 'addopts='
```

Expected: PASS.

**Step 5: Commit**

```bash
git add hermes_cli/config.py tools/browser_tool.py tests/tools/test_browser_playwright_config.py
git commit -m "feat(browser): add Playwright backend config flag"
```

---

## Task 2: Add optional Playwright import and requirement diagnostics

**Objective:** Detect whether Python Playwright is installed and return actionable errors without making the browser tool unavailable globally.

**Files:**
- Modify: `tools/browser_tool.py`
- Test: `tests/tools/test_browser_playwright_config.py`

**Step 1: Add failing tests**

Append:

```python
def test_playwright_requirement_missing(monkeypatch):
    import tools.browser_tool as bt

    def fake_import(name):
        raise ImportError("no playwright")

    monkeypatch.setattr(bt.importlib, "import_module", fake_import)

    ok, message = bt._check_playwright_available()
    assert ok is False
    assert "pip install playwright" in message


def test_playwright_requirement_present(monkeypatch):
    import types
    import tools.browser_tool as bt

    monkeypatch.setattr(bt.importlib, "import_module", lambda name: types.SimpleNamespace())

    ok, message = bt._check_playwright_available()
    assert ok is True
    assert message is None
```

**Step 2: Implement helper**

In `tools/browser_tool.py`:

```python
import importlib


def _check_playwright_available() -> tuple[bool, str | None]:
    try:
        importlib.import_module("playwright.async_api")
    except ImportError:
        return (
            False,
            "Playwright backend is enabled but Python Playwright is not installed. "
            "Install with: pip install playwright && playwright install chromium",
        )
    return True, None
```

**Step 3: Verify**

```bash
python -m pytest tests/tools/test_browser_playwright_config.py -q -o 'addopts='
```

Expected: PASS.

**Step 4: Commit**

```bash
git add tools/browser_tool.py tests/tools/test_browser_playwright_config.py
git commit -m "feat(browser): detect optional Playwright backend dependency"
```

---

## Task 3: Introduce `BrowserbasePlaywrightSession` lifecycle class

**Objective:** Create a focused class that owns one Browserbase Playwright CDP connection and never closes the Browserbase browser during normal per-command operations.

**Files:**
- Create: `tools/browser_playwright_backend.py`
- Test: `tests/tools/test_browser_playwright_backend.py`

**Step 1: Write failing tests with fakes**

Create `tests/tools/test_browser_playwright_backend.py`:

```python
import pytest


@pytest.mark.asyncio
async def test_connect_stores_browser_context_and_page(monkeypatch):
    from tools.browser_playwright_backend import BrowserbasePlaywrightSession

    calls = []

    class FakePage:
        url = "about:blank"

    class FakeContext:
        pages = [FakePage()]

    class FakeBrowser:
        contexts = [FakeContext()]

        async def close(self):
            calls.append("browser.close")

    class FakeChromium:
        async def connect_over_cdp(self, url):
            calls.append(("connect_over_cdp", url))
            return FakeBrowser()

    class FakePlaywright:
        chromium = FakeChromium()

    class FakeStart:
        async def start(self):
            return FakePlaywright()

    monkeypatch.setattr(
        "tools.browser_playwright_backend.async_playwright",
        lambda: FakeStart(),
    )

    session = BrowserbasePlaywrightSession(
        task_id="task",
        bb_session_id="sid",
        cdp_url="wss://example",
        live_view_url="https://live",
    )

    await session.connect()

    assert calls == [("connect_over_cdp", "wss://example")]
    assert session.page is not None
    assert session.live_view_url == "https://live"
```

**Step 2: Implement class skeleton**

`tools/browser_playwright_backend.py`:

```python
from __future__ import annotations

import asyncio
import logging
import time
from dataclasses import dataclass, field
from typing import Any

from playwright.async_api import async_playwright

logger = logging.getLogger(__name__)


@dataclass
class BrowserbasePlaywrightSession:
    task_id: str
    bb_session_id: str
    cdp_url: str
    live_view_url: str | None = None
    heartbeat_interval_seconds: float = 240.0
    playwright: Any = None
    browser: Any = None
    context: Any = None
    page: Any = None
    last_activity: float = field(default_factory=time.time)
    _heartbeat_task: asyncio.Task | None = None

    async def connect(self) -> None:
        if self.browser is not None:
            return
        self.playwright = await async_playwright().start()
        self.browser = await self.playwright.chromium.connect_over_cdp(self.cdp_url)
        self.context = self.browser.contexts[0] if self.browser.contexts else await self.browser.new_context()
        self.page = self.context.pages[0] if self.context.pages else await self.context.new_page()
        self.touch()

    def touch(self) -> None:
        self.last_activity = time.time()
```

**Step 3: Verify**

```bash
python -m pytest tests/tools/test_browser_playwright_backend.py -q -o 'addopts='
```

Expected: PASS.

**Step 4: Commit**

```bash
git add tools/browser_playwright_backend.py tests/tools/test_browser_playwright_backend.py
git commit -m "feat(browser): add Browserbase Playwright session object"
```

---

## Task 4: Add graceful disconnect vs release semantics

**Objective:** Ensure Playwright disconnects cleanly between lifecycle events and only releases Browserbase via provider cleanup when Hermes intends to close the session.

**Files:**
- Modify: `tools/browser_playwright_backend.py`
- Test: `tests/tools/test_browser_playwright_backend.py`

**Step 1: Add failing tests**

```python
@pytest.mark.asyncio
async def test_disconnect_does_not_close_browser(monkeypatch):
    from tools.browser_playwright_backend import BrowserbasePlaywrightSession

    calls = []

    class FakeBrowser:
        async def close(self):
            calls.append("close")
        async def disconnect(self):
            calls.append("disconnect")

    session = BrowserbasePlaywrightSession("task", "sid", "wss://example")
    session.browser = FakeBrowser()
    session.playwright = type("PW", (), {"stop": lambda self: calls.append("stop")})()

    await session.disconnect()

    assert "disconnect" in calls
    assert "close" not in calls


@pytest.mark.asyncio
async def test_force_close_is_explicit(monkeypatch):
    from tools.browser_playwright_backend import BrowserbasePlaywrightSession

    calls = []

    class FakeBrowser:
        async def close(self):
            calls.append("close")
        async def disconnect(self):
            calls.append("disconnect")

    session = BrowserbasePlaywrightSession("task", "sid", "wss://example")
    session.browser = FakeBrowser()

    await session.close_browser_for_shutdown()

    assert calls == ["close"]
```

**Step 2: Implement methods**

```python
async def disconnect(self) -> None:
    browser = self.browser
    self.browser = None
    self.context = None
    self.page = None
    if browser is not None and hasattr(browser, "disconnect"):
        await browser.disconnect()
    if self.playwright is not None:
        stop = getattr(self.playwright, "stop", None)
        if stop:
            maybe = stop()
            if hasattr(maybe, "__await__"):
                await maybe
        self.playwright = None

async def close_browser_for_shutdown(self) -> None:
    browser = self.browser
    self.browser = None
    self.context = None
    self.page = None
    if browser is not None:
        await browser.close()
```

**Step 3: Verify**

```bash
python -m pytest tests/tools/test_browser_playwright_backend.py -q -o 'addopts='
```

Expected: PASS.

**Step 4: Commit**

```bash
git add tools/browser_playwright_backend.py tests/tools/test_browser_playwright_backend.py
git commit -m "feat(browser): separate Playwright disconnect from browser close"
```

---

## Task 5: Add heartbeat support

**Objective:** Keep active Browserbase CDP connections healthy during idle-but-active tasks without consuming excessive traffic.

**Files:**
- Modify: `tools/browser_playwright_backend.py`
- Test: `tests/tools/test_browser_playwright_backend.py`

**Step 1: Add failing test**

```python
@pytest.mark.asyncio
async def test_heartbeat_calls_page_evaluate(monkeypatch):
    from tools.browser_playwright_backend import BrowserbasePlaywrightSession

    calls = []

    class FakePage:
        async def evaluate(self, expr):
            calls.append(expr)

    session = BrowserbasePlaywrightSession(
        "task", "sid", "wss://example", heartbeat_interval_seconds=0.01
    )
    session.page = FakePage()

    await session._heartbeat_once()

    assert calls == ["() => undefined"]
```

**Step 2: Implement heartbeat**

```python
async def _heartbeat_once(self) -> None:
    if self.page is not None:
        await self.page.evaluate("() => undefined")

async def start_heartbeat(self) -> None:
    if self._heartbeat_task is not None:
        return
    self._heartbeat_task = asyncio.create_task(self._heartbeat_loop())

async def _heartbeat_loop(self) -> None:
    while True:
        await asyncio.sleep(self.heartbeat_interval_seconds)
        try:
            await self._heartbeat_once()
        except asyncio.CancelledError:
            raise
        except Exception:
            logger.debug("Browserbase Playwright heartbeat failed", exc_info=True)

async def stop_heartbeat(self) -> None:
    task = self._heartbeat_task
    self._heartbeat_task = None
    if task is not None:
        task.cancel()
        try:
            await task
        except asyncio.CancelledError:
            pass
```

Call `await self.start_heartbeat()` at the end of `connect()`.

**Step 3: Verify**

```bash
python -m pytest tests/tools/test_browser_playwright_backend.py -q -o 'addopts='
```

Expected: PASS.

**Step 4: Commit**

```bash
git add tools/browser_playwright_backend.py tests/tools/test_browser_playwright_backend.py
git commit -m "feat(browser): add Browserbase Playwright heartbeat"
```

---

## Task 6: Add synchronous registry wrapper for async Playwright sessions

**Objective:** Let synchronous browser tool handlers reuse Playwright sessions safely.

**Files:**
- Modify: `tools/browser_playwright_backend.py`
- Test: `tests/tools/test_browser_playwright_backend_registry.py`

**Step 1: Write failing tests**

Create `tests/tools/test_browser_playwright_backend_registry.py`:

```python
def test_registry_reuses_session_for_task():
    from tools.browser_playwright_backend import PlaywrightSessionRegistry

    registry = PlaywrightSessionRegistry()
    first = registry.get_or_create(
        task_id="task",
        bb_session_id="sid",
        cdp_url="wss://example",
        live_view_url="https://live",
    )
    second = registry.get_or_create(
        task_id="task",
        bb_session_id="sid",
        cdp_url="wss://example",
        live_view_url="https://live",
    )

    assert first is second


def test_registry_removes_session():
    from tools.browser_playwright_backend import PlaywrightSessionRegistry

    registry = PlaywrightSessionRegistry()
    session = registry.get_or_create("task", "sid", "wss://example")
    assert registry.get("task") is session

    registry.remove("task")

    assert registry.get("task") is None
```

**Step 2: Implement registry**

```python
class PlaywrightSessionRegistry:
    def __init__(self) -> None:
        self._sessions: dict[str, BrowserbasePlaywrightSession] = {}
        self._lock = threading.RLock()

    def get(self, task_id: str) -> BrowserbasePlaywrightSession | None:
        with self._lock:
            return self._sessions.get(task_id)

    def get_or_create(self, task_id: str, bb_session_id: str, cdp_url: str, live_view_url: str | None = None, **kwargs) -> BrowserbasePlaywrightSession:
        with self._lock:
            existing = self._sessions.get(task_id)
            if existing is not None:
                return existing
            session = BrowserbasePlaywrightSession(task_id, bb_session_id, cdp_url, live_view_url, **kwargs)
            self._sessions[task_id] = session
            return session

    def remove(self, task_id: str) -> BrowserbasePlaywrightSession | None:
        with self._lock:
            return self._sessions.pop(task_id, None)


PLAYWRIGHT_SESSION_REGISTRY = PlaywrightSessionRegistry()
```

Import `threading`.

**Step 3: Verify**

```bash
python -m pytest tests/tools/test_browser_playwright_backend_registry.py -q -o 'addopts='
```

Expected: PASS.

**Step 4: Commit**

```bash
git add tools/browser_playwright_backend.py tests/tools/test_browser_playwright_backend_registry.py
git commit -m "feat(browser): add Playwright session registry"
```

---

## Task 7: Route `browser_navigate` through Playwright when enabled

**Objective:** Implement the first end-to-end browser tool path using Playwright.

**Files:**
- Modify: `tools/browser_tool.py`
- Modify: `tools/browser_playwright_backend.py`
- Test: `tests/tools/test_browser_playwright_navigation.py`

**Step 1: Add helper on session**

Add to `BrowserbasePlaywrightSession`:

```python
async def navigate(self, url: str, wait_until: str = "domcontentloaded") -> dict[str, Any]:
    await self.connect()
    response = await self.page.goto(url, wait_until=wait_until)
    self.touch()
    return {
        "title": await self.page.title(),
        "url": self.page.url,
        "status": response.status if response is not None else None,
    }
```

**Step 2: Write failing dispatch test**

Create `tests/tools/test_browser_playwright_navigation.py` with mocks for:

- `_is_playwright_backend_enabled` -> `True`
- `_get_cloud_provider()` -> fake `BrowserbaseProvider`
- `_get_session_info()` -> Browserbase-like session info
- `PLAYWRIGHT_SESSION_REGISTRY.get_or_create()` -> fake session whose `navigate()` returns title/url
- `_run_browser_command()` -> pytest.fail

Assert `browser_navigate` returns:

```python
{
  "success": True,
  "title": "Alchemy Visions",
  "url": "https://www.alchemyvisions.ai/",
  "bb_session_id": "sid",
  "live_view_url": "https://live",
  "method": "playwright_browserbase"
}
```

**Step 3: Implement dispatch helper**

In `tools/browser_tool.py`, add a focused helper:

```python
def _should_use_playwright_browserbase(session_info: dict[str, Any]) -> bool:
    if not _is_playwright_backend_enabled():
        return False
    if not session_info.get("bb_session_id") or not session_info.get("cdp_url"):
        return False
    provider = _get_cloud_provider()
    return provider is not None and type(provider).__name__ == "BrowserbaseProvider"
```

Add `_navigate_with_playwright_browserbase(task_id, url, session_info)` that runs the async session method using a safe loop helper.

**Step 4: Integrate into `browser_navigate`**

Before `_run_browser_command(..., "open", ...)`, branch to Playwright when enabled. Preserve existing response fields, especially `stealth_features`, `bb_session_id`, and `live_view_url`.

**Step 5: Verify**

```bash
python -m pytest tests/tools/test_browser_playwright_navigation.py tests/tools/test_browser_eval_supervisor_path.py -q -o 'addopts='
```

Expected: PASS.

**Step 6: Commit**

```bash
git add tools/browser_tool.py tools/browser_playwright_backend.py tests/tools/test_browser_playwright_navigation.py
git commit -m "feat(browser): route Browserbase navigation through Playwright"
```

---

## Task 8: Implement Playwright snapshot extraction

**Objective:** Replace `agent-browser snapshot` for Browserbase Playwright sessions with an equivalent accessibility/text snapshot.

**Files:**
- Modify: `tools/browser_tool.py`
- Modify: `tools/browser_playwright_backend.py`
- Test: `tests/tools/test_browser_playwright_snapshot.py`

**Step 1: Add backend method**

```python
async def snapshot(self, full: bool = False) -> dict[str, Any]:
    await self.connect()
    title = await self.page.title()
    url = self.page.url
    text = await self.page.locator("body").inner_text(timeout=5000)
    # Optional: use accessibility tree if available in current Playwright version.
    return {
        "title": title,
        "url": url,
        "text": text,
        "snapshot": text if full else text[:8000],
    }
```

**Step 2: Preserve interactive refs**

If the current `browser_snapshot` depends on `agent-browser` ref IDs like `@e28`, add a minimal ref extraction layer:

```javascript
Array.from(document.querySelectorAll('a,button,input,textarea,select,[role="button"],[tabindex]'))
  .filter(el => el.offsetParent !== null)
  .slice(0, 200)
  .map((el, i) => ({ ref: `@e${i+1}`, text: el.innerText || el.value || el.getAttribute('aria-label') || el.getAttribute('placeholder') || '', tag: el.tagName }))
```

Store the ref mapping on the session so `browser_click` and `browser_type` can use it.

**Step 3: Test**

Mock `page.locator("body").inner_text()` and the ref-evaluation call. Assert snapshot payload includes text and refs.

**Step 4: Verify**

```bash
python -m pytest tests/tools/test_browser_playwright_snapshot.py -q -o 'addopts='
```

Expected: PASS.

**Step 5: Commit**

```bash
git add tools/browser_tool.py tools/browser_playwright_backend.py tests/tools/test_browser_playwright_snapshot.py
git commit -m "feat(browser): add Playwright snapshot extraction"
```

---

## Task 9: Implement Playwright click/type/press/scroll

**Objective:** Reach functional parity for the high-use browser interaction tools.

**Files:**
- Modify: `tools/browser_tool.py`
- Modify: `tools/browser_playwright_backend.py`
- Test: `tests/tools/test_browser_playwright_actions.py`

**Step 1: Backend methods**

Add methods:

```python
async def click_ref(self, ref: str) -> dict[str, Any]: ...
async def type_ref(self, ref: str, text: str) -> dict[str, Any]: ...
async def press(self, key: str) -> dict[str, Any]: ...
async def scroll(self, direction: str) -> dict[str, Any]: ...
```

Implementation rules:

- `click_ref` resolves from the stored snapshot ref map and clicks the matching locator/element handle.
- `type_ref` fills inputs/textareas/contenteditable; if fill fails, fall back to click + keyboard insert.
- `press` delegates to `page.keyboard.press(key)`.
- `scroll` uses mouse wheel or `window.scrollBy`.

**Step 2: Dispatch tests**

For each public browser action, mock a Playwright session and assert `_run_browser_command` is not called when Playwright is enabled and session supports the action.

**Step 3: Fallback behavior**

If ref map is missing/stale or Playwright raises a known actionability error, return a structured failure that asks the agent to refresh `browser_snapshot` rather than silently falling back to a different backend with different page state.

**Step 4: Verify**

```bash
python -m pytest tests/tools/test_browser_playwright_actions.py -q -o 'addopts='
```

Expected: PASS.

**Step 5: Commit**

```bash
git add tools/browser_tool.py tools/browser_playwright_backend.py tests/tools/test_browser_playwright_actions.py
git commit -m "feat(browser): add Playwright browser actions"
```

---

## Task 10: Integrate `browser_console` / evaluation with Playwright sessions

**Objective:** Keep console/eval behavior on the same Browserbase session as navigation and clicks.

**Files:**
- Modify: `tools/browser_tool.py`
- Modify: `tools/browser_playwright_backend.py`
- Test: `tests/tools/test_browser_playwright_console.py`

**Step 1: Backend method**

```python
async def evaluate(self, expression: str) -> dict[str, Any]:
    await self.connect()
    result = await self.page.evaluate(expression)
    self.touch()
    return {"ok": True, "result": result, "result_type": type(result).__name__}
```

**Step 2: Console logs**

Subscribe to Playwright page console events during `connect()` and keep a bounded ring buffer:

```python
self.console_messages: deque[dict[str, Any]] = deque(maxlen=200)
self.page.on("console", lambda msg: self.console_messages.append(...))
self.page.on("pageerror", lambda exc: self.console_messages.append(...))
```

**Step 3: Dispatch integration**

When Playwright backend is enabled and a session exists, `browser_console(expression=...)` should evaluate through Playwright. When `expression` is absent, return the ring buffer.

**Step 4: Verify**

```bash
python -m pytest tests/tools/test_browser_playwright_console.py tests/tools/test_browser_console.py -q -o 'addopts='
```

Expected: PASS.

**Step 5: Commit**

```bash
git add tools/browser_tool.py tools/browser_playwright_backend.py tests/tools/test_browser_playwright_console.py
git commit -m "feat(browser): route console evaluation through Playwright"
```

---

## Task 11: Wire cleanup and inactivity release

**Objective:** Release Browserbase sessions exactly when Hermes intends to, and not at normal browser tool boundaries.

**Files:**
- Modify: `tools/browser_tool.py`
- Modify: `tools/browser_playwright_backend.py`
- Test: `tests/tools/test_browser_playwright_cleanup.py`

**Step 1: Add tests**

Cover:

- Inactivity cleanup stops heartbeat, disconnects Playwright, and calls provider `close_session(bb_session_id)` once.
- Normal navigation/click/snapshot does not call provider `close_session`.
- `cleanup_all_browsers()` releases all Playwright-backed Browserbase sessions.

**Step 2: Implement cleanup hook**

When `cleanup_browser(task_id)` finds a Browserbase-backed Playwright session:

1. Remove it from `PLAYWRIGHT_SESSION_REGISTRY`.
2. Stop heartbeat.
3. Disconnect Playwright cleanly.
4. Call provider `close_session(bb_session_id)`.
5. Remove `_active_sessions[task_id]`.

**Step 3: Verify**

```bash
python -m pytest tests/tools/test_browser_playwright_cleanup.py tests/tools/test_browser_cleanup.py -q -o 'addopts='
```

Expected: PASS.

**Step 4: Commit**

```bash
git add tools/browser_tool.py tools/browser_playwright_backend.py tests/tools/test_browser_playwright_cleanup.py
git commit -m "feat(browser): release Playwright Browserbase sessions on cleanup"
```

---

## Task 12: Add live Browserbase smoke test script

**Objective:** Provide an opt-in verification script for real Browserbase sessions without making CI depend on paid credentials.

**Files:**
- Create: `scripts/smoke_browserbase_playwright_live.py`
- Modify: `docs/plans/2026-05-24-browserbase-playwright-backend.md` if findings change

**Step 1: Create smoke script**

The script should:

1. Load env from `~/.hermes/.env` if present.
2. Assert `BROWSERBASE_API_KEY` and `BROWSERBASE_PROJECT_ID` are set.
3. Create session via `BrowserbaseProvider` with keepAlive and timeout.
4. Connect via Playwright backend.
5. Navigate to `https://www.alchemyvisions.ai/`.
6. Print `bb_session_id` and `live_view_url`.
7. Sleep 60 seconds while sending heartbeat.
8. Poll Browserbase API and assert session is still `RUNNING`.
9. Release session and assert it becomes `COMPLETED`.

**Step 2: Run manually**

```bash
source venv/bin/activate
python scripts/smoke_browserbase_playwright_live.py
```

Expected:

- Prints a Browserbase live-view URL.
- Dashboard shows the same session `RUNNING` while the script sleeps.
- Session remains alive for the full 60 seconds.
- Session is released at the end.

**Step 3: Commit**

```bash
git add scripts/smoke_browserbase_playwright_live.py
git commit -m "test(browser): add Browserbase Playwright live smoke test"
```

---

## Task 13: Documentation and operator runbook

**Objective:** Document how the backend works, how to enable it, and how to recover if Browserbase sessions leak or live view breaks.

**Files:**
- Create: `website/docs/developer-guide/browserbase-playwright-backend.md`
- Modify: `tools/browser_tool.py` docstrings/comments as needed
- Modify: `~/.hermes/skills/autonomous-ai-agents/hermes-agent/references/browserbase-live-view-lifecycle.md` after implementation is proven

**Step 1: Write architecture doc**

Include:

```text
Browser tool call
  -> browser_tool dispatch
  -> BrowserbaseProvider session metadata
  -> PlaywrightSessionRegistry
  -> BrowserbasePlaywrightSession
  -> Browserbase CDP + live view
```

**Step 2: Add config docs**

Document:

```bash
hermes config set browser.cloud_provider browserbase
hermes config set browser.playwright_backend.enabled true
hermes config set browser.inactivity_timeout 900
```

Also document env override:

```bash
export HERMES_BROWSER_PLAYWRIGHT_BACKEND=1
```

**Step 3: Add troubleshooting**

Include:

- Live-view link disconnects immediately: check session status/endedAt/expiresAt.
- Session absent from Browserbase dashboard: verify provider is Browserbase and gateway restarted after config changes.
- Browser minutes leaking: run cleanup, inspect `_active_sessions`, manually release via Browserbase dashboard/API.
- Playwright missing: install dependency or disable backend.

**Step 4: Commit**

```bash
git add website/docs/developer-guide/browserbase-playwright-backend.md tools/browser_tool.py
# Skill updates are outside repo unless mirrored intentionally; commit only repo files.
git commit -m "docs(browser): document Browserbase Playwright backend"
```

---

## Task 14: Full regression pass

**Objective:** Verify the implementation does not regress existing browser providers or gateway cleanup.

**Files:**
- No new files unless fixes are needed.

**Step 1: Run targeted browser tests**

```bash
python -m pytest tests/tools/test_browser_*.py -q -o 'addopts='
```

Expected: PASS.

**Step 2: Run Browserbase-specific tests**

```bash
python -m pytest \
  tests/tools/test_managed_browserbase_and_modal.py \
  tests/tools/test_browser_cloud_provider_cache.py \
  tests/tools/test_browser_cloud_fallback.py \
  tests/tools/test_browser_playwright_*.py \
  -q -o 'addopts='
```

Expected: PASS.

**Step 3: Run gateway shutdown tests**

```bash
python -m pytest tests/gateway/test_gateway_shutdown.py tests/tools/test_zombie_process_cleanup.py -q -o 'addopts='
```

Expected: PASS.

**Step 4: Optional live smoke**

```bash
python scripts/smoke_browserbase_playwright_live.py
```

Expected: Browserbase session stays `RUNNING` while live view is open and releases on script completion.

**Step 5: Commit fixes if needed**

```bash
git add <fixed files>
git commit -m "fix(browser): address Playwright backend regression"
```

---

## Task 15: Enable for Daren’s gateway after merge

**Objective:** Turn on the backend in the real gateway only after tests and smoke pass.

**Files:**
- Modify: `~/.hermes/config.yaml` on Daren’s machine only, not repo defaults unless maintainers choose to make it default later.

**Step 1: Set config**

```bash
hermes config set browser.cloud_provider browserbase
hermes config set browser.playwright_backend.enabled true
hermes config set browser.inactivity_timeout 900
```

**Step 2: Restart gateway**

```bash
systemctl --user restart hermes-gateway.service
```

**Step 3: Verify live view**

Ask Hermes to run:

```text
Go to https://www.alchemyvisions.ai/ and share the Browserbase live-view URL.
```

Expected:

- `browser_navigate` response includes `bb_session_id` and `live_view_url`.
- Browserbase dashboard shows that same session as `RUNNING`.
- User can watch the exact session Hermes drives.
- Session remains `RUNNING` across a normal chat delay and releases after inactivity cleanup.

**Step 4: Commit repo docs update if operational findings differ**

```bash
git add website/docs/developer-guide/browserbase-playwright-backend.md

git commit -m "docs(browser): update Playwright backend runbook after live rollout"
```

---

## QA Checklist Before Build

- [ ] Confirm Python Playwright is acceptable as a dependency or optional extra.
- [ ] Confirm whether `playwright install chromium` is needed for CDP-only Browserbase usage; if not, avoid requiring local browser binaries unnecessarily.
- [ ] Verify Playwright `browser.disconnect()` is available and does not close the Browserbase browser.
- [ ] Confirm live-view and Playwright CDP connection can coexist on the same Browserbase session.
- [ ] Confirm Browserbase context persistence still works with Playwright CDP.
- [ ] Confirm WBR Amplitude hover tooltips can be triggered through Playwright.
- [ ] Confirm no sessions leak after test failures, exceptions, or gateway restarts.
- [ ] Confirm Firecrawl and local browser paths are untouched when Playwright backend is disabled.

## Rollback Plan

1. Disable the backend:

```bash
hermes config set browser.playwright_backend.enabled false
systemctl --user restart hermes-gateway.service
```

2. Release any running Browserbase sessions from the Browserbase dashboard or API:

```bash
curl -X POST \
  -H "X-BB-API-Key: $BROWSERBASE_API_KEY" \
  -H "Content-Type: application/json" \
  "https://api.browserbase.com/v1/sessions/$SESSION_ID" \
  -d '{"status":"REQUEST_RELEASE"}'
```

3. Fall back to existing `agent-browser`/Firecrawl behavior.

4. If needed, revert the implementation commits:

```bash
git revert <commit-sha-range>
```

## Success Criteria

- Browserbase sessions created by `browser_navigate` remain running across normal Telegram response/click delays.
- User can open `live_view_url` and watch the exact session Hermes controls.
- `browser_click`, `browser_type`, `browser_snapshot`, `browser_console`, hover-driven extraction, and screenshots work through Playwright for Browserbase sessions.
- Sessions release after configured inactivity timeout or explicit cleanup.
- Existing browser tests pass with Playwright backend disabled.
- Optional live smoke test passes with Browserbase credentials.
