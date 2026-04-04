# Claude Dashboard

Cross-platform Tkinter dashboard that monitors running Claude Code sessions. Shows session state (working, idle, permission required, awaiting input) via real-time hook events.

## Quick Start

```bash
make install-dev    # Install package + dev deps
make install-hooks  # Merge hook config into ~/.claude/settings.json
make check          # format-check, lint, typecheck, test, coverage
make run            # Run the app (logging to file)
make run DEBUG=1    # Run with debug logging (rotated at 2 MB, 1 backup)
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
| `claude_dashboard/session.py` | Session discovery, PID validation, CWD helpers, `detect_git_status()`, `detect_merged()`, `detect_upstream()` |
| `claude_dashboard/file_utils.py` | Atomic JSON file writes (shared by settings + state persistence) |
| `claude_dashboard/ui/main_window.py` | Dashboard window with session rows |
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
- pytest with 80%+ coverage (currently ~91%)
- Keyword-only args with `*` separator on all public functions
- Settings flow through a single `apply_settings()` path — no duplicating logic between init and update
- All colors/fonts in named constants, not magic strings

## Known Gaps (v0.2)

- **User interrupt**: No hook fires when user hits Ctrl+C/Escape. State stays at last value until next interaction.
- **Permission denial**: Denying a tool without feedback text may not fire a hook event. State stays at `permission_required` until next interaction.
- **Resumed sessions**: Session ID in `sessions/{PID}.json` may not match what hooks send. CWD-based fallback matching handles this.
- **Dashboard starts late**: Sessions show Unknown until their next hook event fires.

See `docs/state-transitions.md` for the full state machine diagram and gap analysis.

## Design Decisions

Decisions recorded here exist because they were non-obvious, caused confusion, or were independently re-proposed as "fixes" during review. They are intentional.

### Agent lifecycle — no TTL or pruning

Agents can run for arbitrarily long periods (hours). There is no reliable signal to distinguish "still working quietly" from "orphaned after interrupt". A TTL would kill legitimate long-running agents. The only known cause of orphaned agents is user interrupt (Ctrl+C) where `SubagentStop` doesn't fire — and the most common cleanup path (PID death removes the session + all agents) already handles this. The narrow gap (session alive, agent interrupted) is accepted until Claude Code provides a better signal. **Do not add agent TTL, pruning, or timeout-based cleanup.**

### Agent permission debounce (5 seconds)

Agent permission requests are debounced for 5 seconds before surfacing in the UI. Agents are typically run via skills that auto-resolve permission prompts, so most agent permission events are transient noise. If the permission is resolved within 5s, the user never sees it. If still pending after 5s, the UI updates. Main process permission requests are NOT debounced — those always need user action.

### Text color — auto-contrast, not configurable

Text color is computed per-row from the background color using W3C sRGB contrast ratios. The `text_color` setting field exists in the Settings dataclass for backward compat but is unused. Removing the setting from the UI was intentional — users pick status colors, text adapts automatically.

### Platform detection — centralized in config.py

`config.IS_WINDOWS` and `config.IS_LINUX` are the single source of truth. Do not add `platform.system()` or `sys.platform` checks in other modules — import from config instead.

### Hook relay `--debug` flag

The relay script always runs with `--debug` in hooks-settings.json, logging raw payloads to `~/.claude/claude-dashboard/logs/hook-payloads.jsonl`. This is safe because both the relay log and the dashboard log use `RotatingFileHandler` (2 MB, 1 backup).

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
- Add agent TTL, pruning, or timeout-based cleanup (see Design Decisions)
- Add `platform.system()` / `sys.platform` checks outside config.py

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

- **Left-click (live)**: Foreground the session's VS Code/terminal window; clear Ready→Idle
- **Left-click (ghost)**: Open in VS Code with tasks.json (auto-launch Claude)
- **Double-click**: Open PR in browser if branch is pushed-not-merged; falls back to create-PR page if no PR exists
- **Middle-click**: Toggle manual flag on clicked row
- **Right-click (row)**: Hide, Clear State, Open PR (when pushed-not-merged); ghosts also get Open in VS Code, Dismiss
- **Right-click (title bar)**: Sessions visibility toggles, Open... (folder picker → VS Code), Settings, Restart, Quit
- **Left-click (title bar)**: Window shade toggle — collapse to title bar only; shaded bar uses highest-priority state color (excludes "ready" — only action-required states)
- **Middle-click (title bar)**: Toggle ghost session visibility (hide/show all ghosts)
- **Left-click (cost labels)**: 14-day cost history popup; dismisses on mouse leave
- **Drag (cost labels)**: Horizontal window resize; width persisted to settings
- **Flag eye icon**: Eye shape left of emoji — outer color = git status, pupil = manual flag (middle-click)
- **Tray menu**: Fully dynamic with "Unhide: (session)" items when sessions are hidden

### State Persistence

- Session state (flagged, hidden, state) saved to `~/.claude/claude-dashboard/session-state.json`
- All persisted across dashboard restarts
- Duplicate-CWD sessions: hidden only persists as true if ALL sessions with that CWD are hidden
- Flag color determined by git status, configurable via 5 `color_flag_*` settings (manual, unstaged, staged, unpushed, unmerged)

### Git Status Flags

- Eye icon outer color reflects git working tree status
- 5 states: manual flag > unstaged > staged uncommitted > committed not pushed > pushed not merged
- Colors configurable via settings
- Detection runs on each discovery tick via git subprocess calls
- Upstream remote and trunk branch detected dynamically from `<remote>/HEAD` (no hardcoded branch names)

### Merged Branch Detection

- Branch text `[branch-name]` turns bright red when branch has been merged into trunk
- Independent of working tree status — shows red even with staged/unstaged changes
- Three merge strategies: ancestor check, `git cherry` (rebase), `git diff` (squash)
- Periodic `git fetch` for pushed-not-merged sessions (~1/min, rate-limited by tick interval)
- Title bar chef kiss image replaces unicode emoji (falls back if image missing)

### Agent Awareness (US9)

- Dashboard tracks agents per session
- Effective state is highest priority across main + all agents
- Agent count indicator `(+N)` shown when agents active
- Agent state persisted across dashboard restarts
- Agent permission requests debounced 5s before surfacing in UI
