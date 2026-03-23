# Claude Dashboard

Cross-platform Tkinter dashboard that monitors running Claude Code sessions. Shows session state (working, idle, permission required, awaiting input) via real-time hook events.

## Quick Start

```bash
make install-dev    # Install package + dev deps
make install-hooks  # Merge hook config into ~/.claude/settings.json
make check          # format-check, lint, typecheck, test, coverage
python -m claude_dashboard          # Run
python -m claude_dashboard --debug  # Run with debug logging
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
| `claude_dashboard/config.py` | Constants, StatusState enum, defaults |
| `claude_dashboard/hook_server.py` | HTTP server on port 17384, event→state mapping |
| `claude_dashboard/controller.py` | Session lifecycle, hook wiring, UI coordination |
| `claude_dashboard/session.py` | Session discovery, PID validation, CWD helpers |
| `claude_dashboard/ui/main_window.py` | Dashboard window with session rows |
| `claude_dashboard/ui/settings_window.py` | Modal settings editor with color pickers |
| `claude_dashboard/tray.py` | System tray icon |
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

## Known Gaps (v0.1)

- **User interrupt**: No hook fires when user hits Ctrl+C/Escape. State stays at last value until next interaction.
- **Permission denial**: Denying a tool without feedback text may not fire a hook event. State stays at `permission_required` until next interaction.
- **Resumed sessions**: Session ID in `sessions/{PID}.json` may not match what hooks send. CWD-based fallback matching handles this.
- **Dashboard starts late**: Sessions show Unknown until their next hook event fires.
- **Subagents**: Main session may show Idle while subagents work. Future enhancement.

See `docs/state-transitions.md` for the full state machine diagram and gap analysis.

## Docs

| File | Content |
|------|---------|
| `docs/spec.md` | Functional requirements (v0.2.0) |
| `docs/plan.md` | Implementation plan and architecture |
| `docs/state-transitions.md` | Mermaid state machine + known gaps |
| `docs/constitution.md` | Project principles |
| `docs/research-session-detection.md` | Session detection research (transcript → hooks evolution) |
| `docs/future.md` | Deferred features (permission handling, input injection, voice) |
| `docs/document-dependencies.md` | Cascade rules for doc updates |
| `docs/tasks.md` | Original task breakdown (partially stale — reflects initial build, not hook refactor) |

## Do Not

- Run git write operations (add, commit, push) — user handles git
- Read `~/.claude/ide/*.lock` files (contain auth tokens)
- Use timeout-based state inference — hooks or nothing
- Duplicate settings logic between init and apply paths
