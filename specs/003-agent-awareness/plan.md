# Agent Awareness Implementation Plan

**Branch**: `003-agent-awareness` | **Date**: 2026-03-24 | **Spec**: `spec.md`
**Constitution**: `.specify/memory/constitution.md`

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

## Summary

Make the dashboard reflect the combined state of the main Claude process and all active agents, so idle-while-agents-work is no longer invisible.

## Technical Context

**Language/Version**: Python 3.11+
**Primary Dependencies**: Tkinter, psutil, pystray, Pillow
**Storage**: JSON settings file (no new storage for this feature)
**Testing**: pytest with 80%+ coverage
**Target Platform**: Windows 11, Linux (cross-platform per constitution)
**Project Type**: Desktop app (Tkinter)
**Performance Goals**: State updates within one hook event; debounce 200-500ms for auto-wake noise
**Constraints**: Local-only, no elevated privileges, read-only observation of Claude sessions

## Constitution Check

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Cross-Platform First | PASS | No platform-specific code needed — hooks and Tkinter are cross-platform |
| II. Low Profile | PASS | Debounce (FR-043) prevents visual noise from auto-wake transitions |
| III. Proven Stack | PASS | Python + Tkinter, existing patterns |
| IV. Configuration Over Opinion | PASS | No new settings fields; `(+N)` format is informational, not opinionated |
| V. Incremental Delivery | PASS | Single user story (US9), independently testable |
| VI. Simplicity | PASS | Flat agent dict, no hierarchy (FR-044). Module-level priority constant shared between classes |

## Architecture

Hook server extracts `agent_id` from payloads and passes it through a new callback. Controller tracks agents per session via `_AgentEntry` on `_SessionEntry`. An `effective_state` property computes the highest-priority state across main + agents. The UI receives the effective state and an agent count — the count is appended to the CWD display as `(+N)`. State display updates are debounced (200-500ms) to absorb auto-wake noise after agent completion, with high-priority states (PermissionRequired, AwaitingInput) bypassing the debounce.

**Spec:** `spec.md`. State machine: `docs/state-transitions.md`.

---

## File Structure

| File | Action | Responsibility |
|------|--------|----------------|
| `claude_dashboard/hook_server.py` | Modify | Extract `agent_id`/`agent_type`, new `on_agent_stop` callback, pass `agent_id` through `on_hook_event` |
| `claude_dashboard/controller.py` | Modify | `_AgentEntry` class, agent tracking on `_SessionEntry`, `effective_state`, clear agents on `UserPromptSubmit`, agent count in UI data |
| `claude_dashboard/ui/main_window.py` | Modify | Display agent count `(+N)` in CWD label, accept agent count in `update_sessions` |
| `tests/test_hook_server.py` | Modify | Tests for `agent_id` extraction and `SubagentStop` callback |
| `tests/test_integration.py` | Modify | Agent lifecycle integration tests |

---

## Task 1: Hook Server — Extract agent_id and SubagentStop

**Files:**
- Modify: `claude_dashboard/hook_server.py:45-95` (`_HookHandler.do_POST`)
- Modify: `claude_dashboard/hook_server.py:102-114` (`HookServer.__init__`)
- Modify: `tests/test_hook_server.py`

The hook server needs three changes:
1. Extract `agent_id` and `agent_type` from payloads
2. Pass `agent_id` and `agent_type` through the `on_hook_event` callback (new signature)
3. Add `on_agent_stop` callback for `SubagentStop` events (parallel to `on_session_end`)

- [ ] **Step 1: Write tests for agent_id extraction**

Add to `tests/test_hook_server.py`:

```python
def test_agent_event_passes_agent_id(self):
    """Hook event with agent_id passes it through the callback."""
    received: list[tuple] = []
    event = threading.Event()

    def on_hook(session_id, hook_event, state, cwd="", agent_id="", agent_type=""):
        received.append((session_id, hook_event, state, agent_id, agent_type))
        event.set()

    server = HookServer(
        on_hook_event=on_hook,
        on_session_end=lambda sid: None,
        on_agent_stop=lambda sid, aid: None,
        port=0,
    )
    server.start()

    try:
        payload = json.dumps(
            {
                "session_id": "test-session",
                "hook_event_name": "PreToolUse",
                "tool_name": "Bash",
                "agent_id": "a41d1801a7ca",
                "agent_type": "general-purpose",
            }
        ).encode()

        req = urllib.request.Request(
            f"http://127.0.0.1:{server.port}/hook",
            data=payload,
            headers={"Content-Type": "application/json"},
            method="POST",
        )
        with urllib.request.urlopen(req, timeout=5):
            pass

        event.wait(timeout=2)
        assert len(received) == 1
        assert received[0][3] == "a41d1801a7ca"
        assert received[0][4] == "general-purpose"
    finally:
        server.stop()

def test_agent_stop_fires_callback(self):
    """SubagentStop fires the on_agent_stop callback."""
    stopped: list[tuple] = []
    event = threading.Event()

    def on_stop(session_id, agent_id):
        stopped.append((session_id, agent_id))
        event.set()

    server = HookServer(
        on_hook_event=lambda *a, **kw: None,
        on_session_end=lambda sid: None,
        on_agent_stop=on_stop,
        port=0,
    )
    server.start()

    try:
        payload = json.dumps(
            {
                "session_id": "test-session",
                "hook_event_name": "SubagentStop",
                "agent_id": "a41d1801a7ca",
                "agent_type": "general-purpose",
            }
        ).encode()

        req = urllib.request.Request(
            f"http://127.0.0.1:{server.port}/hook",
            data=payload,
            headers={"Content-Type": "application/json"},
            method="POST",
        )
        with urllib.request.urlopen(req, timeout=5):
            pass

        event.wait(timeout=2)
        assert stopped == [("test-session", "a41d1801a7ca")]
    finally:
        server.stop()

def test_main_event_has_empty_agent_id(self):
    """Hook event without agent_id passes empty strings."""
    received: list[tuple] = []
    event = threading.Event()

    def on_hook(session_id, hook_event, state, cwd="", agent_id="", agent_type=""):
        received.append((agent_id, agent_type))
        event.set()

    server = HookServer(
        on_hook_event=on_hook,
        on_session_end=lambda sid: None,
        on_agent_stop=lambda sid, aid: None,
        port=0,
    )
    server.start()

    try:
        payload = json.dumps(
            {
                "session_id": "test-session",
                "hook_event_name": "Stop",
            }
        ).encode()

        req = urllib.request.Request(
            f"http://127.0.0.1:{server.port}/hook",
            data=payload,
            headers={"Content-Type": "application/json"},
            method="POST",
        )
        with urllib.request.urlopen(req, timeout=5):
            pass

        event.wait(timeout=2)
        assert received[0] == ("", "")
    finally:
        server.stop()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python -m pytest tests/test_hook_server.py -x -q`
Expected: FAIL — `on_agent_stop` not accepted, callback signature mismatch.

- [ ] **Step 3: Update existing test callbacks**

All existing tests use `on_hook_event` callbacks with signature `(session_id, hook_event, state, cwd="")`. Update them to also accept `agent_id=""` and `agent_type=""` kwargs. Add `on_agent_stop=lambda sid, aid: None` to all `HookServer(...)` calls.

- [ ] **Step 4: Implement hook server changes**

In `hook_server.py`:

1. Update `_HookHandler.do_POST` to extract `agent_id` and `agent_type`:

```python
agent_id = body.get("agent_id", "")
agent_type = body.get("agent_type", "")
```

2. Pass them through to `on_hook_event`:

```python
if state is not None and session_id:
    self.server.on_hook_event(  # type: ignore[attr-defined]
        session_id, event, state, cwd,
        agent_id=agent_id, agent_type=agent_type,
    )
```

3. Add `SubagentStop` handling (parallel to `SessionEnd`):

```python
if event == "SubagentStop" and session_id and agent_id:
    self.server.on_agent_stop(session_id, agent_id)  # type: ignore[attr-defined]
```

4. Update `HookServer.__init__` to accept and store `on_agent_stop` (optional with default no-op for backward compatibility — allows incremental implementation without breaking the app mid-task):

```python
def __init__(
    self,
    *,
    on_hook_event: Callable[..., None],
    on_session_end: Callable[[str], None],
    on_agent_stop: Callable[[str, str], None] = lambda sid, aid: None,
    port: int = 17384,
):
    ...
    self._server.on_agent_stop = on_agent_stop  # type: ignore[attr-defined]
```

5. Do NOT add `SubagentStop` to `_EVENT_STATE_MAP` — it has no state transition. It only fires `on_agent_stop`. The `map_event_to_state` function correctly returns `None` for it, so `on_hook_event` is never called for `SubagentStop` events. This is intentional — FR-036's "do not register on first-and-only SubagentStop" is enforced architecturally by this design.

- [ ] **Step 5: Run tests**

Run: `python -m pytest tests/test_hook_server.py -x -v`
Expected: All pass (existing + 3 new).

- [ ] **Step 6: Run full suite**

Run: `python -m pytest tests/ -x -q`
Expected: Some failures in `test_integration.py` — the `HookServer` constructor now requires `on_agent_stop`. Fix those in Task 2.

---

## Task 2: Controller — Agent Tracking and Effective State

**Files:**
- Modify: `claude_dashboard/controller.py:33-42` (`_SessionEntry`)
- Modify: `claude_dashboard/controller.py:74-79` (HookServer init)
- Modify: `claude_dashboard/controller.py:189-239` (hook event handling)
- Modify: `claude_dashboard/controller.py:259-271` (`_refresh_ui`)
- Modify: `claude_dashboard/controller.py:412-434` (priority calculation)
- Modify: `tests/test_integration.py`

- [ ] **Step 1: Add `_AgentEntry` class and update `_SessionEntry`**

```python
class _AgentEntry:
    """Tracks an agent within a session."""

    __slots__ = ("agent_id", "state", "agent_type")

    def __init__(self, *, agent_id: str, state: StatusState, agent_type: str = ""):
        self.agent_id = agent_id
        self.state = state
        self.agent_type = agent_type


class _SessionEntry:
    """Tracks a live session: its info, container, and current state."""

    __slots__ = ("session", "container", "state", "hidden", "agents")

    def __init__(self, session: SessionInfo):
        self.session = session
        self.container: ContainerInfo | None = None
        self.state: StatusState = StatusState.UNKNOWN
        self.hidden: bool = False
        self.agents: dict[str, _AgentEntry] = {}

    @property
    def effective_state(self) -> StatusState:
        """Highest-priority state across main process + all agents."""
        best = self.state
        best_p = AppController._STATE_PRIORITY.get(best, 999)
        for agent in self.agents.values():
            p = AppController._STATE_PRIORITY.get(agent.state, 999)
            if p < best_p:
                best = agent.state
                best_p = p
        return best
```

**Important:** `effective_state` needs `_STATE_PRIORITY`. Currently it's a class-level dict on `AppController` (line 412). Three steps required:
1. Define `_STATE_PRIORITY` as a module-level constant above both classes
2. Remove `_STATE_PRIORITY` from the `AppController` class body
3. Update `_highest_priority_state` to use bare `_STATE_PRIORITY` (not `self._STATE_PRIORITY`)

The `effective_state` property then references the module-level dict directly.

- [ ] **Step 2: Wire `on_agent_stop` in controller**

In `AppController.__init__`, update the `HookServer` construction:

```python
self._hook_server = HookServer(
    on_hook_event=self._on_hook_event,
    on_session_end=self._on_hook_session_end,
    on_agent_stop=self._on_hook_agent_stop,
    port=config.HOOK_PORT,
)
```

Add the marshalling method:

```python
def _on_hook_agent_stop(self, session_id: str, agent_id: str):
    """Called from hook server thread on SubagentStop."""
    self._root.after(0, self._handle_agent_stop, session_id, agent_id)
```

Add the handler:

```python
def _handle_agent_stop(self, session_id: str, agent_id: str):
    """Remove an agent from a session."""
    pid = self._session_id_to_pid.get(session_id)
    if pid is None:
        return
    entry = self._sessions.get(pid)
    if not entry:
        return
    removed = entry.agents.pop(agent_id, None)
    if removed:
        logger.debug(
            "pid=%d agent=%s removed, %d agents remain",
            pid, agent_id[:12], len(entry.agents),
        )
        self._refresh_ui()
```

- [ ] **Step 3: Update `_on_hook_event` signature and `_apply_hook_state`**

Update `_on_hook_event` to accept and forward agent fields:

```python
def _on_hook_event(
    self, session_id: str, event: str, new_state: StatusState,
    cwd: str = "", agent_id: str = "", agent_type: str = "",
):
    self._root.after(
        0, self._apply_hook_state,
        session_id, event, new_state, cwd, agent_id, agent_type,
    )
```

Update `_apply_hook_state` to handle agents:

```python
def _apply_hook_state(
    self, session_id: str, event: str, new_state: StatusState,
    cwd: str = "", agent_id: str = "", agent_type: str = "",
):
    pid = self._session_id_to_pid.get(session_id)

    # CWD fallback (existing)
    if pid is None and cwd:
        for entry in self._sessions.values():
            if entry.session.cwd == cwd:
                pid = entry.session.pid
                self._session_id_to_pid[session_id] = pid
                break

    if pid is None:
        self._pending_hook_states[session_id] = new_state
        return

    entry = self._sessions.get(pid)
    if not entry:
        return

    if agent_id:
        # Agent event — register or update agent
        agent = entry.agents.get(agent_id)
        if agent is None:
            agent = _AgentEntry(
                agent_id=agent_id, state=new_state, agent_type=agent_type,
            )
            entry.agents[agent_id] = agent
            logger.debug(
                "pid=%d agent=%s registered state=%s",
                pid, agent_id[:12], new_state.value,
            )
        else:
            agent.state = new_state
        self._refresh_ui()
        return

    # Main process event
    # UserPromptSubmit clears all tracked agents (new user turn)
    if event == "UserPromptSubmit" and entry.agents:
        logger.debug("pid=%d clearing %d agents on new prompt", pid, len(entry.agents))
        entry.agents.clear()

    prior = entry.state

    # Intercept idle: transition to ready instead
    if new_state == StatusState.IDLE:
        new_state = StatusState.READY

    if prior == new_state:
        # Only refresh if agents were just cleared (agent count changed)
        if event == "UserPromptSubmit":
            self._refresh_ui()
        return

    entry.state = new_state
    cwd_short = cwd_basename(entry.session.cwd)
    logger.debug(
        "pid=%d project=%s new_state=%s prior_state=%s event=%s",
        pid, cwd_short, new_state.value, prior.value, event,
    )
    self._refresh_ui()
```

- [ ] **Step 4: Update `_refresh_ui` to use effective_state and pass agent count**

```python
def _refresh_ui(self):
    all_entries = self._sorted_entries()
    visible_states = [
        (entry.session, entry.effective_state, entry.container, len(entry.agents))
        for entry in all_entries
        if not entry.hidden
    ]
    self._main_window.update_sessions(visible_states)

    highest = self._highest_priority_state(visible_states)
    if highest != self._tray_state:
        self._tray_state = highest
        update_tray_icon(self._tray_icon, color=self._tray_color_for_state(highest))
```

Update `_highest_priority_state` signature to match:

```python
def _highest_priority_state(
    self,
    session_states: list[tuple[SessionInfo, StatusState, ContainerInfo | None, int]],
) -> StatusState | None:
    if not session_states:
        return None
    best = None
    best_priority = 999
    for _, state, _, _ in session_states:
        p = _STATE_PRIORITY.get(state, 999)
        if p < best_priority:
            best = state
            best_priority = p
    return best
```

- [ ] **Step 5: Update `_remove_session` to clean up agents**

Already handled — when `_sessions.pop(pid)` removes the entry, agents go with it. No change needed, but verify `_remove_session` doesn't need explicit cleanup.

- [ ] **Step 6: Fix integration tests**

Update `tests/test_integration.py` — add `on_agent_stop` to `_make_server`:

```python
def _make_server(self):
    states: dict[str, StatusState] = {}
    ended: list[str] = []
    agent_stopped: list[tuple[str, str]] = []

    def on_hook(session_id, event, state, cwd="", agent_id="", agent_type=""):
        states[session_id] = state

    def on_end(session_id):
        ended.append(session_id)

    def on_agent_stop(session_id, agent_id):
        agent_stopped.append((session_id, agent_id))

    server = HookServer(
        on_hook_event=on_hook,
        on_session_end=on_end,
        on_agent_stop=on_agent_stop,
        port=0,
    )
    return server, states, ended, agent_stopped
```

Update all callers of `_make_server` to unpack 4 values instead of 3.

Also update `test_rapid_tool_calls_stay_working` — it monkey-patches `server._server.on_hook_event` with a wrapper that only accepts `(session_id, event, state, cwd="")`. Update the wrapper signature to `(session_id, event, state, cwd="", agent_id="", agent_type="")`.

- [ ] **Step 7: Run full test suite**

Run: `python -m pytest tests/ -x -q`
Expected: All pass.

---

## Task 3: Debounce State Display Updates (FR-043)

**Files:**
- Modify: `claude_dashboard/controller.py` (`_refresh_ui`)

Auto-wake cycles after agent completion (`UserPromptSubmit` → `Stop` per agent) cause rapid state transitions that flicker the dashboard. Debounce UI updates with a 300ms window. High-priority states (PermissionRequired, AwaitingInput) bypass the debounce.

- [ ] **Step 1: Add debounce to `_refresh_ui`**

Replace direct UI calls with a debounced version. Use `tkinter.after()` with a cancel-and-reschedule pattern:

```python
_DEBOUNCE_MS = 300  # module-level constant

# On AppController.__init__:
self._debounce_id: str | None = None

def _refresh_ui(self):
    """Debounced UI refresh. High-priority states bypass debounce."""
    # Check if any visible session has a high-priority effective state
    all_entries = self._sorted_entries()
    bypass = any(
        entry.effective_state in (StatusState.PERMISSION_REQUIRED, StatusState.AWAITING_INPUT)
        for entry in all_entries
        if not entry.hidden
    )

    if bypass:
        # Cancel pending debounce and refresh immediately
        if self._debounce_id is not None:
            self._root.after_cancel(self._debounce_id)
            self._debounce_id = None
        self._do_refresh_ui(all_entries)
    else:
        # Cancel previous pending refresh, schedule new one
        if self._debounce_id is not None:
            self._root.after_cancel(self._debounce_id)
        self._debounce_id = self._root.after(
            _DEBOUNCE_MS, self._do_refresh_ui_deferred,
        )

def _do_refresh_ui_deferred(self):
    """Called by debounce timer."""
    self._debounce_id = None
    self._do_refresh_ui(self._sorted_entries())

def _do_refresh_ui(self, all_entries):
    """Actual UI refresh logic (extracted from original _refresh_ui)."""
    visible_states = [
        (entry.session, entry.effective_state, entry.container, len(entry.agents))
        for entry in all_entries
        if not entry.hidden
    ]
    self._main_window.update_sessions(visible_states)

    highest = self._highest_priority_state(visible_states)
    if highest != self._tray_state:
        self._tray_state = highest
        update_tray_icon(self._tray_icon, color=self._tray_color_for_state(highest))
```

- [ ] **Step 2: Run full test suite**

Run: `python -m pytest tests/ -x -q`
Expected: All pass. Tests that check immediate state changes may need adjustment if they rely on synchronous refresh — use `root.update()` or `root.after(0, ...)` in test harnesses to flush pending callbacks.

---

## Task 4: UI — Agent Count Indicator

**Files:**
- Modify: `claude_dashboard/ui/main_window.py:186-204` (`update_sessions`)
- Modify: `claude_dashboard/ui/main_window.py:280-366` (`_add_row`, `_update_row`)

The `update_sessions` signature changes to include agent count. The CWD label appends `(+N)` when agents are active.

- [ ] **Step 1: Update `update_sessions` signature**

Change the tuple to include agent count:

```python
def update_sessions(
    self,
    sessions: list[tuple[SessionInfo, StatusState, ContainerInfo | None, int]],
):
```

Update all internal references that unpack the tuple (add `_` or `agent_count` as the 4th element).

- [ ] **Step 2: Update `_add_row` to accept and display agent count**

Add `agent_count: int = 0` parameter. Build the CWD display string:

```python
def _add_row(
    self,
    session: SessionInfo,
    state: StatusState,
    container: ContainerInfo | None = None,
    agent_count: int = 0,
):
    ...
    display_name = cwd_relative_to_home(session.cwd)
    if agent_count > 0:
        display_name = f"{display_name} (+{agent_count})"
    cwd_var = tk.StringVar(value=display_name)
    ...
```

- [ ] **Step 3: Update `_update_row` to accept and display agent count**

Add `agent_count: int = 0` parameter. Update the CWD var:

```python
def _update_row(
    self,
    session: SessionInfo,
    state: StatusState,
    container: ContainerInfo | None = None,
    agent_count: int = 0,
):
    row = self._rows[session.pid]
    ...
    display_name = cwd_relative_to_home(session.cwd)
    if agent_count > 0:
        display_name = f"{display_name} (+{agent_count})"
    row["cwd_var"].set(display_name)
    ...
```

- [ ] **Step 4: Update `update_sessions` body to pass agent count through**

```python
for session, state, container, agent_count in sessions:
    if session.pid in self._rows:
        self._update_row(session, state, container, agent_count)
    else:
        self._add_row(session, state, container, agent_count)
        changed = True
```

And the same for other tuple unpacking sites in `update_sessions` (`current_pids`, `desired_order`, re-order loop).

- [ ] **Step 5: Run full test suite**

Run: `python -m pytest tests/ -x -q`
Expected: All pass. Some grow-up tests may need tuple signature updates.

---

## Task 5: Integration Tests — Agent Lifecycle

**Files:**
- Modify: `tests/test_integration.py`

- [ ] **Step 1: Add agent lifecycle integration test**

```python
def test_agent_lifecycle(self):
    """SubagentStart -> agent tool use -> SubagentStop."""
    server, states, _, agent_stopped = self._make_server()
    server.start()
    sid = "agent-test"

    try:
        # Main starts
        _post_hook(server.port, {"session_id": sid, "hook_event_name": "UserPromptSubmit"})
        assert states[sid] == StatusState.WORKING

        # Agent starts working (PreToolUse with agent_id)
        _post_hook(server.port, {
            "session_id": sid,
            "hook_event_name": "PreToolUse",
            "tool_name": "Bash",
            "agent_id": "agent-1",
            "agent_type": "general-purpose",
        })
        assert states[sid] == StatusState.WORKING

        # Main stops (idle) but agent is still working
        _post_hook(server.port, {"session_id": sid, "hook_event_name": "Stop"})
        assert states[sid] == StatusState.IDLE

        # Agent finishes
        _post_hook(server.port, {
            "session_id": sid,
            "hook_event_name": "SubagentStop",
            "agent_id": "agent-1",
        })
        assert agent_stopped == [(sid, "agent-1")]
    finally:
        server.stop()

def test_agent_permission_request(self):
    """Agent PermissionRequest carries agent_id."""
    server, states, _, _ = self._make_server()
    server.start()
    sid = "agent-perm-test"

    try:
        _post_hook(server.port, {
            "session_id": sid,
            "hook_event_name": "PermissionRequest",
            "tool_name": "Read",
            "agent_id": "agent-2",
            "agent_type": "general-purpose",
        })
        # The states dict captures by session_id, and the hook still maps to PERMISSION_REQUIRED
        assert states[sid] == StatusState.PERMISSION_REQUIRED
    finally:
        server.stop()
```

- [ ] **Step 2: Run full test suite**

Run: `python -m pytest tests/ -x -q`
Expected: All pass.

---

## Task 6: Final Validation

- [ ] **Step 1: Run full test suite with coverage**

Run: `python -m pytest tests/ --cov=claude_dashboard --cov-report=term-missing -q`
Expected: All pass, coverage >= 80%.

- [ ] **Step 2: Run make check**

Run: `make check`
Expected: format, lint, typecheck, test, coverage all pass.

- [ ] **Step 3: Fix any issues**

- [ ] **Step 4: Manual verification**

With the dashboard running and `--debug` enabled on hooks:
1. Spawn a background agent — verify `(+1)` appears on the row and state stays Working
2. Agent completes — verify `(+1)` disappears and state transitions to Ready
3. Spawn 2 background agents — verify `(+2)`, then `(+1)` as each completes
4. Send a new prompt while agents tracked — verify agents cleared
