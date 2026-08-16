# Claude Dashboard

Cross-platform Tkinter dashboard that monitors running Claude Code sessions. Shows session state (working, idle, permission required, awaiting input) via OTEL-based Prometheus metrics.

## Quick Start

```bash
make install-dev    # Install package + dev deps
make check          # test-format, test-lint, test-typecheck, test-unit, test-coverage
make run            # Run the app (logging to file)
make run DEBUG=1    # Run with debug logging (rotated at 2 MB, 1 backup)
python -m claude_dashboard                     # Run with console output
python -m claude_dashboard --debug             # Run with debug logging to console
python -m claude_dashboard --log-file <path>   # Redirect logs to file (append mode)
```

Requires OTEL stack running — see [claude-otel-stack](https://github.com/jewzaam/claude-otel-stack)

## Architecture

- **Session discovery**: Polls `~/.claude/sessions/*.json` every 5s to find running Claude sessions
- **State detection**: Claude Code native OTEL events → OTEL collector → Loki → Prometheus recording rules → dashboard polls Prometheus metrics
- **UI**: Tkinter borderless window with `overrideredirect` (visible on all virtual desktops)
- **Tray**: pystray system tray icon with color reflecting highest-priority session state

### OTEL State Flow

Claude Code emits native OTEL events → OTEL collector exports to Loki → Prometheus recording rules aggregate log entries into session state metrics (`claude_session_state{session_id}`) → dashboard polls Prometheus `/api/v1/query` every 5s → controller updates UI.

Session state metrics aggregate all activity (main + agents) by session_id. No per-agent tracking. State values: `working`, `idle`, `permission_required`, `awaiting_input`, `unknown`.

### Key Files

| File | Purpose |
|------|---------|
| `claude_dashboard/config.py` | Constants, StatusState enum, SandboxPhase enum, EmojiKey type alias, defaults, LOG_FILE, STATE_FILE, PID_FILE, DEFAULT_PROMETHEUS_URL, resolve_prometheus_url() |
| `claude_dashboard/otel_state.py` | Prometheus poller for session state metrics from OTEL recording rules |
| `claude_dashboard/loki.py` | Loki query client for session user prompt extraction (tooltips) |
| `claude_dashboard/controller.py` | Session lifecycle, OTEL poller wiring, UI coordination, session state persistence, PID file lock |
| `claude_dashboard/session.py` | Session discovery, PID validation, CWD helpers, `detect_git_status()`, `detect_merged()`, `detect_upstream()`, `detect_terminal_activity()` |
| `claude_dashboard/file_utils.py` | Atomic JSON file writes (shared by settings + state persistence) |
| `claude_dashboard/ui/main_window.py` | Dashboard window with session rows |
| `claude_dashboard/ui/settings_window.py` | Modal settings editor |
| `claude_dashboard/ui/color_picker.py` | Custom color picker with palette grid, hex entry, live preview |
| `claude_dashboard/tray.py` | System tray icon, dynamic menu with unhide items |
| `claude_dashboard/platform/base.py` | Platform dispatch (ContainerType enum) |
| `claude_dashboard/platform/windows.py` | Win32 window foregrounding |
| `claude_dashboard/platform/linux.py` | Window foregrounding via `window-calls` D-Bus, xdotool fallback |

## Standards

- Python 3.11+, Tkinter, psutil, pystray, Pillow, requests
- black (line-length 100), flake8, mypy
- pytest with 80%+ coverage (currently ~91%)
- Keyword-only args with `*` separator on all public functions
- Settings flow through a single `apply_settings()` path — no duplicating logic between init and update
- All colors/fonts in named constants, not magic strings

## Configuration

- **Prometheus URL**: Prometheus server URL (default `http://localhost:9090`). Configurable via the Settings window (`prometheus_url`, persisted to settings.json). Env var `PROMETHEUS_URL` overrides the saved setting at runtime via `config.resolve_prometheus_url()` (env wins, else settings value).
- **PID_FILE**: Lock file for single-instance enforcement (`~/.claude/claude-dashboard/dashboard.pid`)

## Known Gaps (v0.2)

- **OTEL state latency**: 15-30s delay between Claude Code state change and dashboard update (OTEL SDK flush + recording rule eval + dashboard poll). Accepted tradeoff for sandbox support.
- **User interrupt**: No OTEL event fires when user hits Ctrl+C/Escape. State stays at last value until next interaction.
- **Permission denial**: Denying a tool without feedback text may not fire an OTEL event. State stays at `permission_required` until next interaction.
- **Dashboard starts late**: Sessions show Unknown until Prometheus has their first state metric.

See `docs/state-transitions.md` for the full state machine diagram and gap analysis.

## Loki Query Implementation

### Tooltip XML stripping

`loki.py` `_extract_user_text()` strips all XML blocks from OTEL `user_prompt` event `prompt` field. User text is never wrapped in XML tags; system-injected content always is (`<system-reminder>`, `<task-notification>`, `<command-name>`, etc.). Strip all `<tag>...</tag>` blocks and orphan tags. Do not maintain an exclusion list of known tags — an exclusion list not based on a schema will always fail.

### Cross-stream ordering

Loki `direction=backward` only orders entries within each stream. Structured metadata (like `prompt`) creates unique streams per event. Cross-stream order in the result array is not guaranteed. `_extract_prompts()` sorts results by `observed_timestamp` (descending) before extracting to ensure newest-first ordering. Do not rely on Loki result array order for cross-stream queries.

### Lookback window

`_LOOKBACK_SECONDS = 86400` (24 hours) for tooltip queries. Agent-heavy sessions generate `<task-notification>` auto-wake events that push real user prompts beyond a 1-hour window. A session can have 10+ consecutive task-notification prompts between user inputs.

### Query limit

`limit = len(session_ids) * 20` per query to look past task-notification noise from background agent completions. `_extract_prompts()` skips XML-only prompts and continues to the next result.

## Design Decisions

Decisions recorded here exist because they were non-obvious, caused confusion, or were independently re-proposed as "fixes" during review. They are intentional.

### OTEL state — cleared-state guard

When user clears a session state (context menu, click, double-click), `state_cleared_from` records the state value that was cleared (e.g., `"working"`). OTEL poller rejects updates that report the same state — stale metrics keep reporting what was cleared and get blocked. A genuinely different state (e.g., PERMISSION_REQUIRED after clearing WORKING) is accepted, and `state_cleared_from` resets to `""`. No timestamps or cooldowns — pure state identity comparison. Persisted to `session-state.json` so protection survives dashboard restarts.

### Tooltip dismissal — timers, not crossing events or pointer position

Tkinter runs under XWayland, where the compositor never reports the global pointer position. `winfo_pointerxy()` freezes at the last coordinate the pointer held over one of our own windows — which is inside the row it just left — so **pointer position cannot prove the pointer left a row**, and `<Leave>` is not reliably delivered when the pointer exits a borderless window. Five rules follow, all in `main_window.py`:

1. **`<Motion>` arms the tooltip, not `<Enter>`.** Destroying a tooltip makes X re-evaluate what is under the (frozen) pointer and fire a synthetic `<Enter>` on the row beneath, which re-armed an endless show/hide loop. Real hovering always produces motion; a synthetic crossing never does. `_show_tooltip()` also returns early if one is already pending or shown for that row, so the repeated motion events do not restack timers.
2. **Motion not newer than the last `<Leave>` is discarded** (`_tooltip_leave_time`, compared with a `_TOOLTIP_STALE_MS` window so X timestamp wraparound cannot wedge it). Tk dispatches motion after compression, so the final motion of a pointer leaving the window can arrive *after* the `<Leave>` that dismissed the tooltip and re-arm it. This showed as: grow-down mode, exit across the bottom row's own edge → tooltip always returns. Other edges cross a different widget first, so their last motion is not on a tooltip row.
3. **A pointer reading within `_EDGE_MARGIN_PX` (3 px) of any window border counts as gone.** A pointer that left freezes at its exit point, which is on the outermost pixels; a hover that merely stopped is essentially never parked there. This is what stops the reproducible case — grow-down, exit across the bottom row's own bottom edge — where no other signal distinguishes departure from a stationary hover. Cost: no tooltip while hovering that outer sliver.
4. **`_TOOLTIP_LIFETIME_MS` (4 s) is the only guarantee.** A displayed tooltip always self-destructs; everything else is a best-effort speedup.
5. `_pointer_over_row()` (pre-display guard + 150 ms `_watch_pointer()` poll) cannot prove the pointer is present, only that it is plausibly present.

**Do not re-bind tooltips to `<Enter>` and do not treat pointer coordinates as authoritative.**

### Text color — auto-contrast, not configurable

Text color is computed per-row from the background color using W3C sRGB contrast ratios. The `text_color` setting field exists in the Settings dataclass for backward compat but is unused. Removing the setting from the UI was intentional — users pick status colors, text adapts automatically.

### Platform detection — centralized in config.py

`config.IS_WINDOWS` and `config.IS_LINUX` are the single source of truth. Do not add `platform.system()` or `sys.platform` checks in other modules — import from config instead.

### Ghost eviction — capacity-based, not TTL

Ghost sessions are evicted when total session count exceeds `max_sessions` (default 40). Only non-flagged ghosts are eviction candidates — live sessions and flagged ghosts are never removed. Oldest ghosts (by `last_active` epoch timestamp) are evicted first. Eviction happens lazily on ghost creation and startup. There is no TTL — a ghost with recent activity is always retained if under the cap. **Do not add time-based ghost expiration.**

### Sandbox phase rendering

`SandboxPhase` enum in `config.py` has values READY, ERROR, CREATING, STOPPING, UNKNOWN. All phases flow through discovery — no filtering by openshell phase. Error sandboxes get phase-specific emoji (⚠️ unattached, 🔥 active) via `_sandbox_emoji()` static method in `main_window.py`. Ready+idle sandboxes use default ghost rendering (🏖️). Error sandboxes are excluded from ghost visibility toggle via `_is_error_sandbox()` helper. Ready sandboxes without VS Code toggle with ghosts — they are functionally ghosts.

### Sandbox rendering model

Sandboxes never render as ghosts — always state color background + state emoji. Grey text (`_COLOR_CONTAINER_FG`) when VS Code disconnected, normal contrast when connected. Connection state comes from the `window-calls` D-Bus window scan only — never from OTEL or PID. That extension is a hard dependency: `__main__.main()` calls `window_calls_available()` on Linux and exits with an install message if D-Bus does not answer. Without the check a missing extension silently dims every sandbox row and drops sandbox tooltips (`_update_last_prompts()` skips `unattached` entries), which reads as a rendering bug. `sandbox_connected` field on SessionRow carries VS Code status. `unattached` passed as False for sandboxes in SessionRow (internally still tracked for VS Code detection). Gone from openshell → removed from dashboard entirely (no ghost state). Sandbox Ready→Idle automatically when VS Code disconnects.

### Sandbox removal protection

When `discover_sandbox_sessions()` returns empty but sandbox entries exist, skip removal — likely a transient openshell failure, not all sandboxes vanishing. Protects against spurious removal on openshell errors.

### EmojiKey type alias

`EmojiKey = StatusState | SandboxPhase | None` in `config.py`. `EMOJI_IMAGES` dict uses this type. `EMOJI_SANDBOX_ERROR_ACTIVE` is a standalone `Path` constant (not in the dict) for the fire emoji.

### D-Bus gdbus quote escaping

`_list_windows_dbus()` in `platform/linux.py` must unescape `\\"` → `\"` in gdbus output before JSON parsing. gdbus output can use double-quote wrapper `("...",)` in addition to single-quote `('...',)` — both formats handled. Escaping unified: `\"` → `"` for both quote styles. Window titles containing literal quotes (e.g., music player track names) break JSON parse without this fix. This affects ALL D-Bus window features (sandbox VS Code detection, live session foregrounding).

### Remote sessions — OTEL-only discovery, sticky attention states

Sessions running on another host (same OTEL stack) are discovered purely from
Prometheus state metrics: a `session_id` unmatched by local discovery whose
`host_name` differs from the local hostname becomes a remote row (synthetic
pid, `remote_host` set). The `location` label (display-normalized path from
the recording rules) is used as the row CWD. Rules that follow from having no
local process or working tree:

- **Lifecycle is metric-driven with sticky exceptions.** A remote row whose
  metrics expired is removed — unless its state is PERMISSION_REQUIRED /
  AWAITING_INPUT or it is flagged; those persist until the user dismisses
  (double-click → IDLE → removed next poll). The local stale-WORKING→READY
  flip is NOT applied to remote rows; an absent WORKING remote row is removed
  directly. Do not add TTLs or auto-clear for sticky remote states.
- **Sticky states survive restarts.** Remote entries persist to
  `session-state.json` under a host-qualified key (`host:cwd`) with
  `remote_host`, `cwd`, and `session_id` stored in the value.
  `_restore_remote_from_state()` recreates only sticky/flagged rows on
  startup — everything else is rediscovered from OTEL. The host-qualified key
  prevents a remote project sharing a path with a local one from
  cross-applying hidden/flagged/cleared state.
- **No local behaviors.** Remote rows skip git checks, ghost conversion, CWD
  pruning, terminal-activity scan, and click-to-foreground (left-click still
  clears READY→IDLE). They are never ghosts.
- **Rendering.** Remote rows sort in a block above everything else, by host
  then path. The flag/eye slot background and the right-side truncated host
  label use `color_remote` (settings, default `#0891b2`). Tooltips are
  prefixed with `[host]`. "Show remote sessions" checkbox at the top of the
  title-bar right-click menu toggles visibility (`show_remote` setting);
  entries are still tracked while hidden so sticky states are not lost.
- **Local sandbox race (accepted).** A local sandbox whose telemetry arrives
  before openshell lists it would briefly register as remote; in practice
  local sandbox discovery runs earlier in the same tick and OTEL lags by
  15-30s, so local sandboxes always claim their session_id first.

### Single-instance enforcement

Uses PID file lock (`~/.claude/claude-dashboard/dashboard.pid`) to prevent multiple dashboard instances. Stale PID files (process dead) are removed on startup. Replaces previous port-based lock (hook server socket binding).

## Docs

### Feature Specs (per-feature directories)

| Directory | Content |
|-----------|---------|
| `specs/001-session-dashboard/` | US1-US5 spec, plan, tasks, session detection research |
| `specs/002-hide-apply-autostart/` | US6-US8 spec (hide sessions, Apply button, auto-start) |
| `specs/003-agent-awareness/` | US9 spec, plan, agent hook research |
| `docs/plan-pivot.md` | OTEL-based state detection pivot plan — full cutover from HTTP hooks to OTEL log entries, recording rules, Prometheus metrics, session source configuration (Layer 4) |

### Project-Wide Docs

| File | Content |
|------|---------|
| `docs/state-transitions.md` | Mermaid state machine + known gaps |
| `.specify/memory/constitution.md` | Project principles |
| `docs/future.md` | Deferred features (permission handling, input injection, voice) |
| `docs/spec-future.md` | Deferred user stories (P2/P3) |
| `docs/document-dependencies.md` | Cascade rules for doc updates |
| `docs/raw-prompt.md` | Original voice transcript |

### User Guides

| File | Content |
|------|---------|
| `docs/guide/getting-started.md` | Installation and first-run walkthrough |
| `docs/guide/visual-guide.md` | Annotated screenshots of the dashboard UI |
| `docs/guide/session-lifecycle.md` | How sessions are discovered and tracked |
| `docs/guide/interactions.md` | Click/hover/menu interactions reference |
| `docs/guide/settings.md` | Settings dialog and configuration options |

## Do Not

- Run git write operations (add, commit, push) — user handles git
- Read `~/.claude/ide/*.lock` files (contain auth tokens)
- Add custom command hooks for state detection — OTEL native events provide all signals
- Duplicate settings logic between init and apply paths
- Add `platform.system()` / `sys.platform` checks outside config.py

## Active Technologies

- Python 3.11+ + Tkinter, psutil, pystray, Pillow, requests
- OTEL collector + Loki + Prometheus recording rules (claude-otel-stack)
- JSON settings file (no new storage for this feature)

## Recent Enhancements (v0.2+)

### Logging & Startup

- `--log-file <path>` CLI arg redirects logging to file (append mode)
- `sys.excepthook` captures uncaught stack traces in log files
- Makefile `run` target logs to `~/.claude/claude-dashboard/dashboard.log`
- Single-instance enforcement via PID file lock
- All discovered sessions start visible — nothing is auto-hidden
- Headless runs are not discovered at all: `discover_sessions()` skips `HEADLESS_ENTRYPOINTS` (`sdk-cli`, written by `claude -p` and the Agent SDK). Verified against Claude Code 2.1.226 — the interactive TUI writes `cli`. Unknown entrypoints stay visible so a new one never vanishes silently. OTEL telemetry is emitted by Claude Code itself and is unaffected

### Terminal Activity Detection

`detect_terminal_activity()` in `session.py` scans VS Code terminal child processes (psutil-based, Linux-only). When a non-Claude process runs in a VS Code terminal matching a session CWD, the flag icon (leftmost) changes from the git eye to a circular arrow (`emoji_terminal_activity.png`), tinted with the same git status color the eye would use (`_tint_image()` swaps RGB, keeps alpha — the asset is a single flat color). A clean tree keeps the asset's own yellow. Excluded processes: claude-wrapper.sh, claude, openshell, node, code, pgrep, ps, grep. Checked every ~30s on the sandbox vscode tick.

### UI Interactions

- **Left-click (live)**: Foreground the session's VS Code/terminal window; clear Ready→Idle
- **Left-click (ghost)**: Open in VS Code with tasks.json (auto-launch Claude)
- **Double-click**: Toggle IDLE↔READY (re-flag for attention or clear state)
- **Middle-click**: Toggle manual flag on clicked row
- **Right-click (live)**: Hide, Clear State
- **Right-click (sandbox)**: Hide, Clear State, Delete Sandbox
- **Right-click (ghost local)**: Hide, Dismiss
- **Right-click (title bar)**: Sessions visibility toggles, Open... (folder picker → VS Code), Settings, Restart, Quit
- **Left-click (title bar)**: Window shade toggle — collapse to title bar only; shaded bar uses highest-priority state color
- **Middle-click (title bar)**: Toggle ghost session visibility (hide/show all ghosts) — includes disconnected sandboxes alongside local ghosts; flagged and error sandboxes excluded
- **Drag (counts label)**: Horizontal window resize; width persisted to settings
- **Flag eye icon**: Eye shape left of emoji — outer color = git status, pupil = manual flag (middle-click). Terminal activity changes flag icon from git eye to a circular arrow (`emoji_terminal_activity.png`) when non-Claude process runs in VS Code terminal matching session CWD — arrow is tinted by git status, yellow when clean. Manual flag is not shown while the arrow is displayed
- **Tray menu**: Fully dynamic with "Unhide: (session)" items when sessions are hidden

### State Persistence

- Session state (flagged, hidden, state) saved to `~/.claude/claude-dashboard/session-state.json`
- All persisted across dashboard restarts
- Duplicate-CWD sessions: hidden only persists as true if ALL sessions with that CWD are hidden
- Flag color determined by git status, configurable via 5 `color_flag_*` settings (manual, unstaged, staged, unpushed, unmerged)
- Sandboxes always appear in sessions visibility menu regardless of `unattached` state, so they can be unhidden

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

### OTEL State Detection

- Session state derived from Prometheus metrics (`claude_session_state{session_id}`)
- Metrics aggregated from native Claude Code OTEL events via recording rules
- All activity (main + agents) rolled up by session_id — no per-agent tracking
- Polling interval: 5s (configurable in controller)
- State values: `working`, `idle`, `permission_required`, `awaiting_input`, `unknown`
