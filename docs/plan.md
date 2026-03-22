# Implementation Plan: Claude Dashboard

- **Date**: 2026-03-22
- **Spec**: `docs/spec.md`
- **Constitution**: `docs/constitution.md`
- **Standards**: `~/source/standards/`
- **Version**: 0.1.0

## Summary

A cross-platform Python/Tkinter dashboard that discovers running Claude Code sessions via `~/.claude/sessions/*.json`, detects their state by tailing transcript JSONL files, displays them as a configurable vertical stack of rows, and enables click-to-foreground navigation by walking the process tree to the containing application window.

## Technical Context

- **Language/Version**: Python 3.11+
- **Primary Dependencies**: Tkinter (stdlib), psutil, pystray, Pillow
- **Storage**: JSON settings file (`%APPDATA%/claude-dashboard/settings.json` on Windows, `~/.config/claude-dashboard/settings.json` on Linux)
- **Testing**: pytest, pytest-cov (80%+ coverage target per standards)
- **Target Platform**: Windows 11, Linux (X11; Wayland deferred)
- **Project Type**: Desktop application (single-user utility)
- **Performance Goals**: < 1% CPU at idle, poll cycle completes in < 100ms
- **Constraints**: No elevated privileges, no network calls, read-only access to Claude CLI files
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

This project is self-contained. All data dependencies are core Claude CLI files that exist on any Claude Code installation:

- `~/.claude/sessions/*.json` — session registry (PID, CWD, session ID)
- `~/.claude/projects/{project}/{sessionId}.jsonl` — session transcripts

No dependency on user-configured hooks, scripts, or external tooling. If future features need additional data from Claude (e.g., statusline metadata), the dashboard will implement its own hooks. If a hook is not configured, functionality must degrade gracefully — never break.

## Project Structure

### Source Code

```text
claude_dashboard/
├── __init__.py
├── __main__.py              # Entry point (--debug, --quiet flags)
├── config.py                # Constants, defaults
├── settings.py              # Settings dataclass, JSON load/save
├── controller.py            # Main dashboard (poll loop, state management)
├── session.py               # Session discovery, state detection
├── transcript.py            # Transcript JSONL parser, state machine
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
├── test_transcript.py       # Transcript parsing, state machine tests
├── test_settings.py         # Settings persistence tests
└── fixtures/
    ├── sample_session.json
    ├── sample_transcript.jsonl
    └── sample_settings.json

scripts/
└── detect_sessions.py       # Standalone diagnostic tool (already written)

.github/
└── workflows/               # CI: test, lint, typecheck, format, coverage

Makefile                     # check, install-dev, format, lint, typecheck, test, coverage
TEST_PLAN.md                 # Testing strategy per standards
pyproject.toml
README.md
```

### Key Files

| File | Responsibility | Pattern From d4-timer-w11 |
|------|---------------|---------------------------|
| `controller.py` | Central dashboard: owns root Tk, poll loop, coordinates subsystems | `controller.py` — AppController |
| `settings.py` | Dataclass + atomic JSON I/O with validation | `settings.py` — Settings dataclass |
| `session.py` | Read `sessions/*.json`, validate PIDs, resolve transcript paths | `api.py` — external data fetch |
| `transcript.py` | Tail JSONL, apply state machine, return StatusState | New (no equivalent) |
| `ui/main_window.py` | Dynamic row grid, status indicators, click handlers | `ui/main_window.py` — countdown grid |
| `ui/settings_window.py` | Modal dialog for editing settings | `ui/settings_window.py` — modal |
| `tray.py` | System tray icon with context menu, attention indicator | `tray.py` — pystray integration |
| `platform/windows.py` | `SetForegroundWindow`, process tree via psutil, `EnumWindows` | `startup.py` — Windows-specific |

## Architecture

### Threading Model (from d4-timer-w11)

```
Main Thread (Tkinter)
├── root.mainloop()
├── _tick() every 5000ms (configurable)
│   ├── discover_sessions()      # Read sessions/*.json
│   ├── validate_pids()          # psutil.pid_exists
│   ├── detect_states()          # Tail transcripts
│   ├── detect_containers()      # Walk process trees (cached)
│   └── update_ui()              # Add/remove/update rows
└── UI event handlers
    ├── row click → foreground_window()
    └── tray menu → show/hide/settings/quit
```

Single-threaded. The poll operations are all local file reads and process queries — fast enough to run on the main thread within a single tick. No worker threads needed for MVP.

### State Machine (from research-session-detection.md)

```
┌─────────────────────────────────────────────────────┐
│ Tail last N lines of transcript JSONL               │
│                                                     │
│ Find last type="assistant" entry:                   │
│   stop_reason="end_turn" ──────────► AwaitingInput  │
│   stop_reason="tool_use"                            │
│     + no subsequent tool result ──► PermissionReq   │
│     + subsequent tool result ─────► Working         │
│                                                     │
│ Last entry is type="user" (prompt) ► Working        │
│                                                     │
│ Cannot determine ─────────────────► Unknown         │
│                                                     │
│ PID not alive ────────────────────► (remove row)    │
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

### Transcript Path Resolution

The transcript path is constructed directly from `sessions/{PID}.json` data — no external dependencies:

```
Session file (sessions/{PID}.json)
  → cwd (e.g., "C:\Users\user\source\claude-dashboard")
  → sessionId (e.g., "5fda2c96-...")

Encode CWD as project key:
  → Replace path separators and colons with dashes
  → e.g., "C--Users-user-source-claude-dashboard"

Construct transcript path:
  → ~/.claude/projects/{project_key}/{sessionId}.jsonl
```

No dependency on session-tracker or any external data source.

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

1. `session.py` — read `sessions/*.json`, validate PIDs, construct transcript paths
2. `transcript.py` — parse JSONL, implement state machine
3. Tests for both with fixture data
4. Verify against live sessions using `detect_sessions.py` patterns

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
| Transcript format changes in Claude CLI update | State detection breaks | Pin to known transcript schema; validate on parse; graceful fallback to Unknown |
| `sessions/` directory behavior changes | Session discovery breaks | Fallback to process enumeration via psutil |
| VS Code window title format changes | Window matching breaks | Fuzzy match; fallback to foregrounding any Code.exe window |
| Wayland on Linux lacks xdotool | Foregrounding fails on Wayland | Detect compositor; degrade gracefully with error message |
| Large transcript files slow polling | Tick exceeds poll interval | Read only last N bytes (seek to end), not full file |
