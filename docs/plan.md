# Implementation Plan: Claude Dashboard

- **Date**: 2026-03-22
- **Spec**: `docs/spec.md`
- **Constitution**: `docs/constitution.md`
- **Standards**: `~/source/standards/`
- **Version**: 0.1.0

## Summary

A cross-platform Python/Tkinter dashboard that discovers running Claude Code sessions via `~/.claude/sessions/*.json`, detects their state via HTTP hooks from Claude Code, displays them as a configurable vertical stack of rows, and enables click-to-foreground navigation by walking the process tree to the containing application window.

## Technical Context

- **Language/Version**: Python 3.11+
- **Primary Dependencies**: Tkinter (stdlib), psutil, pystray, Pillow
- **Storage**: JSON settings file (`%APPDATA%/claude-dashboard/settings.json` on Windows, `~/.config/claude-dashboard/settings.json` on Linux)
- **Testing**: pytest, pytest-cov (80%+ coverage target per standards)
- **Target Platform**: Windows 11, Linux (X11; Wayland deferred)
- **Project Type**: Desktop application (single-user utility)
- **Performance Goals**: < 1% CPU at idle, poll cycle completes in < 100ms
- **Constraints**: No elevated privileges, localhost HTTP only (port 17384 for hook events), read-only access to Claude CLI files
- **Scale/Scope**: 1-10 concurrent sessions, single user

## Constitution Check

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Cross-Platform First | Pass | Platform-specific code isolated in `platform/` modules |
| II. Low Profile | Pass | System tray persistence, no unsolicited noise |
| III. Proven Stack | Pass | Python + Tkinter + psutil, patterns from d4-timer-w11 |
| IV. Configuration Over Opinion | Pass | Settings dataclass with JSON persistence, modal editor |
| V. Incremental Delivery | Pass | P1 stories are independently testable MVP |
| VI. Simplicity | Pass | No abstractions beyond what's needed |

## Data Independence

This project depends on core Claude CLI files and a user-installed hook configuration:

- `~/.claude/sessions/*.json` — session registry (PID, CWD, session ID) — core CLI, always present
- `~/.claude/settings.json` — must include hook configuration installed via `make install-hooks`
- `hooks-settings.json` — ships with the project, defines the hook endpoints
- `scripts/install_hooks.py` — deep-merges hook config into user's Claude settings

The hooks are non-blocking. If the dashboard is not running, Claude sessions are unaffected — the HTTP POST fails silently. If hooks are not installed, the dashboard still discovers sessions but cannot determine their state (all sessions show Unknown).

## Project Structure

### Source Code

```text
claude_dashboard/
├── __init__.py
├── __main__.py              # Entry point (--debug, --quiet flags)
├── config.py                # Constants, defaults
├── settings.py              # Settings dataclass, JSON load/save
├── controller.py            # Main dashboard (poll loop, state management)
├── session.py               # Session discovery, PID validation
├── transcript.py            # StatusState enum, map_event_to_state()
├── hook_server.py           # HTTP server receiving hook events (port 17384)
├── tray.py                  # System tray icon (pystray)
├── platform/
│   ├── __init__.py
│   ├── base.py              # Abstract interface for platform operations
│   ├── windows.py           # Window foregrounding, process tree (Win32)
│   └── linux.py             # Window foregrounding, process tree (X11)
└── ui/
    ├── __init__.py
    ├── main_window.py       # Dashboard window with session rows
    └── settings_window.py   # Modal settings editor

tests/
├── conftest.py
├── test_session.py          # Session discovery tests
├── test_transcript.py       # StatusState enum, hook event mapping tests
├── test_hook_server.py      # Hook HTTP server tests (4 tests)
├── test_settings.py         # Settings persistence tests
└── fixtures/
    ├── sample_session.json
    └── sample_settings.json

scripts/
├── detect_sessions.py       # Standalone diagnostic tool (already written)
└── install_hooks.py         # Deep-merge hooks-settings.json into ~/.claude/settings.json

hooks-settings.json          # Hook configuration for Claude Code

.github/
└── workflows/               # CI: test, lint, typecheck, format, coverage

Makefile                     # check, install-dev, install-hooks, format, lint, typecheck, test, coverage
TEST_PLAN.md                 # Testing strategy per standards
pyproject.toml
README.md
```

### Key Files

| File | Responsibility | Pattern From d4-timer-w11 |
|------|---------------|---------------------------|
| `controller.py` | Central dashboard: owns root Tk, poll loop, coordinates subsystems | `controller.py` — AppController |
| `settings.py` | Dataclass + atomic JSON I/O with validation | `settings.py` — Settings dataclass |
| `session.py` | Read `sessions/*.json`, validate PIDs | `api.py` — external data fetch |
| `transcript.py` | StatusState enum, `map_event_to_state()` | New (no equivalent) |
| `hook_server.py` | HTTP server on port 17384, receives hook events, dispatches to callbacks | New (no equivalent) |
| `ui/main_window.py` | Dynamic row grid, status indicators, click handlers | `ui/main_window.py` — countdown grid |
| `ui/settings_window.py` | Modal dialog for editing settings | `ui/settings_window.py` — modal |
| `tray.py` | System tray icon with context menu, attention indicator | `tray.py` — pystray integration |
| `platform/windows.py` | `SetForegroundWindow`, process tree via psutil, `EnumWindows` | `startup.py` — Windows-specific |

## Architecture

### Threading Model

```
Main Thread (Tkinter)
├── root.mainloop()
├── _tick() every 5000ms (configurable)
│   ├── discover_sessions()      # Read sessions/*.json
│   ├── validate_pids()          # psutil.pid_exists
│   ├── detect_containers()      # Walk process trees (cached)
│   └── update_ui()              # Add/remove/update rows
└── UI event handlers
    ├── row click → foreground_window()
    └── tray menu → show/hide/settings/quit

Hook Server Thread (HTTP, daemon)
├── HookServer on port 17384
├── POST /hook → parse JSON → map_event_to_state()
│   ├── on_hook_event(session_id, event, state)  # State updates
│   └── on_session_end(session_id)               # Session removal
└── Non-blocking: if dashboard down, Claude unaffected
```

The poll loop handles session discovery and PID validation. State detection is event-driven via the hook server — no transcript parsing. The hook server runs on a daemon thread and dispatches callbacks to the main thread for UI updates.

### State Machine (hook event driven)

```
┌─────────────────────────────────────────────────────┐
│ HTTP POST /hook from Claude Code                    │
│                                                     │
│ Hook Event:                                         │
│   UserPromptSubmit ────────────────► Working         │
│   PreToolUse (regular tool) ───────► Working         │
│   PreToolUse (AskUserQuestion) ────► AwaitingInput   │
│   PermissionRequest ───────────────► PermissionReq   │
│   PostToolUse ─────────────────────► Working         │
│   Stop ────────────────────────────► Idle            │
│   SessionEnd ──────────────────────► (remove row)   │
│                                                     │
│ No hook received yet ──────────────► Unknown         │
│                                                     │
│ PID not alive (poll) ──────────────► (remove row)   │
└─────────────────────────────────────────────────────┘
```

### Settings Dataclass

```python
@dataclass
class Settings:
    # Window
    window_x: int | None = None
    window_y: int | None = None
    always_on_top: bool = True

    # Rows
    row_height: int = 32
    row_width: int = 400

    # Poll
    poll_interval_seconds: int = 5

    # Status indicators (emoji)
    emoji_working: str = "🔄"
    emoji_awaiting_input: str = "❓"
    emoji_permission_required: str = "⚠️"
    emoji_unknown: str = "🤷"

    # Status colors (row background)
    color_working: str = "#1a3a5c"
    color_awaiting_input: str = "#1a4a2a"
    color_permission_required: str = "#5c4a1a"
    color_unknown: str = "#3a3a3a"

    # General
    window_bg: str = "#1a1a1a"
    text_color: str = "#e0e0e0"
```

### Container Detection & Window Matching (verified)

1. Walk parent PIDs via `psutil` until hitting a known application name
2. For VS Code: all windows owned by the main process (parent=explorer.exe), not renderers
3. Match CWD folder name against window titles
4. `SetForegroundWindow` with Alt-key fallback on Windows
5. Linux: `xdotool` or `wmctrl` (to be implemented)

See `scripts/detect_sessions.py` for working implementation.

### Hook Configuration

The dashboard ships `hooks-settings.json` which configures Claude Code to POST hook events to `http://localhost:17384/hook`. The `scripts/install_hooks.py` script deep-merges this into `~/.claude/settings.json`.

Hook events: `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PermissionRequest`, `Stop`, `SessionEnd`.

Port 17384 is fixed (not configurable). The hooks are non-blocking — if the dashboard is not running, the POST fails silently and Claude sessions are unaffected.

## Coding Standards (from ~/source/standards/)

- **Named parameters**: All optional parameters use keyword-only args (`*` separator)
- **Formatting**: black
- **Linting**: flake8
- **Type checking**: mypy
- **Testing**: pytest, 80%+ coverage, TEST_PLAN.md required
- **CLI flags**: `--debug` (DEBUG logging), `--quiet` (suppress non-essential output)
- **Logging**: stderr for debug/warnings, stdout for results
- **Makefile**: `check` (default), `install-dev`, `format`, `lint`, `typecheck`, `test`, `coverage`
- **GitHub workflows**: test (3.10-3.14), lint, typecheck, format-check, coverage
- **No Git LFS**

## Dependencies

```toml
[project]
requires-python = ">=3.11"
dependencies = [
    "psutil>=5.9",
    "pystray>=0.19",
    "Pillow>=10.0",
]

[project.optional-dependencies]
dev = [
    "pytest",
    "pytest-cov",
    "black",
    "flake8",
    "mypy",
]

[project.scripts]
claude-dashboard = "claude_dashboard.__main__:main"
```

## Build Sequence

### Phase 1 — Session Discovery & State Detection (P1, no UI)

1. `session.py` — read `sessions/*.json`, validate PIDs
2. `transcript.py` — StatusState enum, `map_event_to_state()`
3. `hook_server.py` — HTTP server on port 17384, hook event dispatch
4. `hooks-settings.json` + `scripts/install_hooks.py` — hook configuration installer
5. Tests for all with fixture data and live HTTP tests

### Phase 2 — Dashboard UI (P1)

1. `config.py` + `settings.py` — defaults, persistence, atomic JSON
2. `ui/main_window.py` — Tkinter window with dynamic row grid
3. `controller.py` — poll loop, wire session discovery to UI
4. `__main__.py` — entry point with `--debug` and `--quiet` flags

### Phase 3 — Navigation & Container Detection (P1)

1. `platform/windows.py` — process tree, window matching, foregrounding
2. `platform/linux.py` — stub with xdotool/wmctrl (basic implementation)
3. Wire click handlers in `ui/main_window.py`

### Phase 4 — System Tray & Settings (P2)

1. `tray.py` — pystray icon, context menu (Show/Hide, Settings, Quit)
2. `ui/settings_window.py` — modal dialog for all configurable values
3. Tray icon attention indicator (aggregate session state)

### Phase 5 — Container Indicator (P2)

1. Add container type icon/label to each row
2. Source icons from running process or use text labels

## Risk Register

| Risk | Impact | Mitigation |
|------|--------|------------|
| Hook event format changes in Claude CLI update | State detection breaks | Validate hook payloads; unknown events are ignored; graceful fallback to Unknown |
| `sessions/` directory behavior changes | Session discovery breaks | Fallback to process enumeration via psutil |
| VS Code window title format changes | Window matching breaks | Fuzzy match; fallback to foregrounding any Code.exe window |
| Wayland on Linux lacks xdotool | Foregrounding fails on Wayland | Detect compositor; degrade gracefully with error message |
| Port 17384 already in use | Hook server fails to start | Log error clearly; dashboard still works for session discovery but state is Unknown |
| Hooks not installed | No state updates received | Dashboard shows Unknown for all sessions; `make install-hooks` documented in README |
