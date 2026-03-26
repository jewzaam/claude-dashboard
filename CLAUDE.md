# Claude Dashboard

Cross-platform Tkinter dashboard that monitors running Claude Code sessions. Shows session state (working, idle, permission required, awaiting input) via real-time hook events.

## Quick Start

```bash
make install-dev    # Install package + dev deps
make install-hooks  # Merge hook config into ~/.claude/settings.json
make check          # format-check, lint, typecheck, test, coverage
make run            # Run (logs to ~/.claude/claude-dashboard/dashboard.log)
make run DEBUG=1    # Run with debug logging
python -m claude_dashboard                     # Run with console output
python -m claude_dashboard --debug             # Run with debug logging to console
python -m claude_dashboard --log-file <path>   # Redirect logs to file (append mode)
```

## Architecture

- **Session discovery**: Polls `~/.claude/sessions/*.json` every 5s to find running Claude sessions
- **State detection**: Claude Code command hooks → `hook_relay.py` → POST to localhost:17384 → dashboard HTTP server
- **UI**: Tkinter borderless window with `overrideredirect` (visible on all virtual desktops)
- **Tray**: pystray system tray icon with color reflecting highest-priority session state

### Hook Flow

Claude Code → command hook fires → `~/.claude/claude-dashboard/scripts/hook_relay.py` reads JSON from stdin → POSTs to `http://127.0.0.1:17384/hook` → `hook_server.py` maps event to state → controller updates UI.

HTTP hooks are documented by Claude Code but don't work in practice (tested 2026-03-22). Command hooks with the relay script are the working approach.

### Key Files

| File | Purpose |
|------|---------|
| `claude_dashboard/config.py` | Constants, StatusState enum, defaults, LOG_FILE, STATE_FILE |
| `claude_dashboard/hook_server.py` | HTTP server on port 17384, event→state mapping, SO_REUSEADDR/SO_REUSEPORT |
| `claude_dashboard/controller.py` | Session lifecycle, hook wiring, UI coordination, session state persistence |
| `claude_dashboard/session.py` | Session discovery, PID validation, CWD helpers |
| `claude_dashboard/ui/main_window.py` | Dashboard window with session rows, row right-click context menu |
| `claude_dashboard/ui/settings_window.py` | Modal settings editor |
| `claude_dashboard/ui/color_picker.py` | Custom color picker with palette grid, hex entry, live preview |
| `claude_dashboard/tray.py` | System tray icon, dynamic menu with unhide items |
| `claude_dashboard/platform/base.py` | Platform dispatch (ContainerType enum) |
| `claude_dashboard/platform/windows.py` | Win32 window foregrounding |
| `claude_dashboard/platform/linux.py` | Window foregrounding via `window-calls` D-Bus, xdotool fallback |
| `scripts/hook_relay.py` | Command hook → HTTP POST relay |
| `scripts/install_hooks.py` | Deep-merge hook config into Claude settings |
| `hooks-settings.json` | Hook config shipped with the project |

## Standards

- Python 3.11+, Tkinter, psutil, pystray, Pillow
- black (line-length 100), flake8, mypy
- pytest with 80%+ coverage (currently ~94%)
- Keyword-only args with `*` separator on all public functions
- Settings flow through a single `apply_settings()` path — no duplicating logic between init and update
- All colors/fonts in named constants, not magic strings

## Known Gaps (v0.2)

- **User interrupt**: No hook fires when user hits Ctrl+C/Escape. State stays at last value until next interaction.
- **Permission denial**: Denying a tool without feedback text may not fire a hook event. State stays at `permission_required` until next interaction.
- **Resumed sessions**: Session ID in `sessions/{PID}.json` may not match what hooks send. CWD-based fallback matching handles this.
- **Dashboard starts late**: Sessions show Unknown until their next hook event fires.

See `docs/state-transitions.md` for the full state machine diagram and gap analysis.

## Docs

### Feature Specs (per-feature directories)

| Directory | Content |
|-----------|---------|
| `specs/001-session-dashboard/` | US1-US5 spec, plan, tasks, session detection research |
| `specs/002-hide-apply-autostart/` | US6-US8 spec (hide sessions, Apply button, auto-start) |
| `specs/003-agent-awareness/` | US9 spec, plan, agent hook research |

### Project-Wide Docs

| File | Content |
|------|---------|
| `docs/state-transitions.md` | Mermaid state machine + known gaps |
| `.specify/memory/constitution.md` | Project principles |
| `docs/future.md` | Deferred features (permission handling, input injection, voice) |
| `docs/spec-future.md` | Deferred user stories (P2/P3) |
| `docs/document-dependencies.md` | Cascade rules for doc updates |
| `docs/raw-prompt.md` | Original voice transcript |

## Do Not

- Run git write operations (add, commit, push) — user handles git
- Read `~/.claude/ide/*.lock` files (contain auth tokens)
- Use timeout-based state inference — hooks or nothing
- Duplicate settings logic between init and apply paths

## Active Technologies
- Python 3.11+ + Tkinter, psutil, pystray, Pillow (003-agent-awareness)
- JSON settings file (no new storage for this feature) (003-agent-awareness)

## Recent Enhancements (v0.2+)

### Logging & Startup
- `--log-file <path>` CLI arg redirects logging to file (append mode)
- `sys.excepthook` captures uncaught stack traces in log files
- Makefile `run` target logs to `~/.claude/claude-dashboard/dashboard.log`
- Clean shutdown with `SO_REUSEADDR`/`SO_REUSEPORT` on hook server socket

### UI Interactions
- **Row context menu** (right-click): Hide, Clear agents (if present), Flag/Unflag
- **Flag indicator**: Purple dot (⬤) left of container label. Sticky — only middle-click or context menu toggles
- **Tray menu**: Fully dynamic with "Unhide: <session>" items when sessions are hidden

### State Persistence
- Session state (flagged, state) saved to `~/.claude/claude-dashboard/session-state.json`
- Restored on startup (except hidden — hidden is transient)
- Flag color configurable via `color_flagged` setting

### Agent Awareness (US9)
- Dashboard tracks agents per session
- Effective state is highest priority across main + all agents
- Agent count indicator `(+N)` shown when agents active
- "Clear agents" in row context menu when agents present
