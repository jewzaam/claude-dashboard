# Behavioral Specification — Claude Dashboard

## 1. Purpose and Operational Overview

Claude Dashboard is a cross-platform desktop application that monitors running Claude Code sessions and displays their real-time state. It renders a borderless always-on-top Tkinter window with one row per session, a system tray icon reflecting the highest-priority session state, and context menus for session management.

State detection is event-driven: Claude Code command hooks fire a relay script that POSTs JSON to a local HTTP server. The dashboard maps hook events to five session states (`WORKING`, `READY`, `IDLE`, `AWAITING_INPUT`, `PERMISSION_REQUIRED`) and updates the UI accordingly.

Session discovery is poll-based: the dashboard reads `~/.claude/sessions/*.json` on a configurable interval (default 3 seconds) to find new sessions and detect dead ones.

## 2. Startup and Initialization Sequence

1. **Argument parsing** (`__main__.py:40-47`): Four CLI arguments — `--debug` (debug logging), `--quiet`/`-q` (suppress non-essential), `--ttl N` (auto-quit after N seconds, 0=forever), `--log-file <path>` (redirect logs to rotating file).

2. **Logging configuration** (`__main__.py:49-69`):
   - With `--log-file`: `RotatingFileHandler` (2 MB max, 1 backup). Level is `DEBUG` if `--debug`, `WARNING` if `--quiet`, else `INFO`.
   - Without `--log-file`: logs to stderr. Level is `DEBUG` if `--debug`, `WARNING` if `--quiet`, else `INFO`.
   - PIL logger forced to `WARNING` regardless.
   - `sys.excepthook` replaced to route uncaught exceptions through the logger (`__main__.py:27-36`). `KeyboardInterrupt` is excluded and forwarded to `sys.__excepthook__`.

3. **Single-instance check** (`__main__.py:14-24`): Attempts to bind a socket to `127.0.0.1:17384`. If the port is already in use, logs an error and exits with code 1.

4. **DPI awareness** (Windows only, `__main__.py:81-87`): Calls `ctypes.windll.shcore.SetProcessDpiAwareness(1)` (`PROCESS_SYSTEM_DPI_AWARE`). Failure is caught and ignored (older Windows versions).

5. **AppController construction** (`controller.py:192-280`):
   - Loads settings from disk via `load_settings()`.
   - Creates `tk.Tk()` root, immediately withdrawn.
   - Creates `MainWindow` with a `MainWindowCallbacks` struct containing 12 callback functions.
   - Initializes session tracking structures: `_sessions` (PID -> `_SessionEntry`), `_session_id_to_pid` (reverse lookup), `_pending_hook_states` (buffer for pre-discovery events).
   - Initializes trunk cache (CWD -> trunk branch name), fetch tick counter, debounce state.
   - Creates `HookServer` with three callbacks: `_on_hook_event`, `_on_hook_session_end`, `_on_hook_agent_stop`.
   - Syncs OS auto-start with `settings.run_on_startup`.
   - Creates pystray tray icon (not yet running) with four menu callbacks.
   - Creates context menu and ghost-toggle state.
   - Loads saved session state from `session-state.json`.

6. **AppController.run()** (`controller.py:286-321`):
   - Starts hook server (daemon thread).
   - Starts tray icon (daemon thread).
   - If `ttl_seconds > 0`, schedules `_quit` via `root.after`.
   - Fires first `_discovery_tick()`.
   - Enters `root.mainloop()`.
   - On exit (KeyboardInterrupt or quit): stops hook server, stops tray icon.

## 3. Core Lifecycle: Discovery, Polling, Event Processing

### 3.1 Discovery Tick

The discovery tick runs every `poll_interval_seconds * 1000` milliseconds via `root.after` (`controller.py:419-420`). Default interval: 3 seconds (`config.py:40`).

Each tick performs these steps in order:

1. **Discover sessions**: Read all `~/.claude/sessions/*.json` files. Each file contains `pid`, `sessionId`, `cwd`, `startedAt`, `entrypoint`. Sessions are sorted by `cwd_relative_to_home` (case-insensitive).

2. **Register new sessions**: For each discovered session whose PID is alive (validated via `psutil.pid_exists` + process name contains "claude"), if not already tracked, call `_add_session()`.

3. **Ghost dead sessions**: PIDs in `_sessions` but not in the alive set are removed and replaced with ghost entries (synthetic negative PIDs, `unattached=True`). Ghost creation is skipped if another live session still uses the same CWD.

4. **First-tick ghosts**: On the first tick only, `_create_unattached_from_state()` creates ghost entries from the state file for CWDs that have no live session. This provides restart continuity.

5. **Prune stale ghosts**: Ghost entries whose CWD directory no longer exists on disk are removed.

6. **Periodic git fetch**: Every `max(60 // max(poll_interval, 1), 1)` ticks (approximately once per minute at default settings), runs `git fetch <remote>` for all sessions with `PUSHED_NOT_MERGED` status. Fetches are deduplicated by CWD. Timeout: 10 seconds per fetch.

7. **Clear trunk cache**: The trunk branch cache (`_trunk_cache`) is cleared every tick so newly-added remotes are detected.

8. **Git status update**: For all sessions (live and ghost), re-detect branch, git status, and merge status. Git status detection is skipped (result discarded) if it takes longer than 0.5 seconds (`controller.py:400-403`).

9. **Cost and usage update**: Daily cost is re-read every `poll_interval_seconds * 10` ticks (approximately every 30 seconds at default settings, `controller.py:407-409`). Usage limits are re-read every tick.

10. **Refresh UI**: Calls `_refresh_ui()` which is debounced (see section 5).

### 3.2 Session Registration (`_add_session`)

When a new session is discovered (`controller.py:443-495`):

1. Check if CWD matches `ignore_regex` setting — if so, skip silently.
2. Create `_SessionEntry` with initial state `IDLE`.
3. Detect container (VS Code, Terminal, Git Bash, Screen, Unknown) via platform-specific process tree walk.
4. Find the specific window for the session (Windows: title matching by CWD folder name; Linux: deferred to foreground time).
5. Detect git branch (reads `.git/HEAD` directly, no subprocess).
6. Detect git status (subprocess, logs warning if >0.5s on first detection).
7. If an unattached ghost exists for this CWD, replace it — inherit its `flagged` state, force `hidden=False`.
8. Otherwise, apply saved state from the state file (flagged, hidden, state value, agents).
9. If `entrypoint != "cli"` (e.g., `claude -p`), auto-hide the session.
10. Store in `_sessions` dict and register session_id -> PID mapping.
11. Replay any buffered hook state (events that arrived before discovery).

### 3.3 Ghost Session Lifecycle

When a session's PID dies:
- The live session is removed from `_sessions`.
- A ghost entry is created with a synthetic negative PID (decremented from -1).
- The ghost inherits the `flagged` state from the dead session.
- Ghosts appear in the UI with the `color_unattached` background and dim text.
- Ghosts persist across dashboard restarts via the state file.

Ghost cleanup:
- When a new live session matches a ghost's CWD, the ghost is replaced.
- When the ghost's CWD directory is deleted from disk, the ghost is pruned.
- User can dismiss a ghost via right-click context menu.

## 4. State Machines and Transitions

### 4.1 Main Process State Machine

```
                    UserPromptSubmit,
                    PreToolUse (non-AskUserQuestion),
                    PostToolUse
                  +------------------+
                  |                  |
                  v                  |
    +----------+     +---------+    |
    |          |     |         |----+
    |  IDLE *  |---->| WORKING |
    |          |     |         |---->  PERMISSION_REQUIRED
    +----------+     +---------+      (PermissionRequest, non-AskUserQuestion)
         ^               |
         |               |          +---------+
         |               +--------> |         |
         |               Stop/      | AWAITING|
         |               StopFail   | _INPUT  |
         |                          |         |
         |    +--------+            +---------+
         |    |        |  (PreToolUse or PermissionRequest
         +----|  READY |   where tool_name = AskUserQuestion)
   click      |        |
              +--------+
                  ^
                  |
              Stop/StopFail (intercepted: IDLE -> READY)
```

*Notes:*
- `IDLE` is the initial state for new sessions.
- The `IDLE` state is never reached via hook events for the main process — `Stop` and `StopFail` are intercepted to `READY` (`controller.py:672-673`). `READY` is cleared to `IDLE` by a left-click on the row (`controller.py:847-849`).
- `PostToolUseFailure` during `PERMISSION_REQUIRED` is suppressed — state remains `PERMISSION_REQUIRED` (`controller.py:679-691`). This prevents parallel tool failures from clearing a pending permission.
- Same-state transitions are suppressed (`controller.py:675-676`).

### 4.2 Agent State

Agents have the same five states but with different rules:
- There is no `IDLE -> READY` intercept for agents.
- `PERMISSION_REQUIRED` is debounced by 5 seconds before surfacing in the UI.
- Agents are removed on `SubagentStop`, not on state transitions.
- Unmapped events with an `agent_id` (except `SubagentStop`) default to `WORKING`.

### 4.3 Effective State

A session's effective state is the highest-priority state across its main process state and all agent states (`_SessionEntry.effective_state`, `controller.py:156-165`). Priority order: `PERMISSION_REQUIRED` (0) > `AWAITING_INPUT` (1) > `WORKING` (2) > `READY` (3) > `IDLE` (4).

## 5. Event Flows and Processing Rules

### 5.1 Hook Event Flow

See the ASCII data flow in `c4-container.md`. Key processing rules:

1. **Session ID lookup**: Events are matched to sessions by `session_id`. If not found, a CWD-based fallback match is attempted (handles resumed sessions where the session ID may differ from the file, `controller.py:609-615`).

2. **Buffering**: Events for undiscovered sessions are buffered in `_pending_hook_states` and replayed when the session is registered (`controller.py:618-619`, `controller.py:484-487`).

3. **Thread marshaling**: All hook server callbacks use `root.after(0, fn)` to marshal to the Tkinter main thread (`controller.py:576-593`).

### 5.2 UI Refresh Debounce

UI refreshes are debounced at 300ms (`_DEBOUNCE_MS`, `controller.py:68`). When `_refresh_ui()` is called:
- If any non-hidden session has `PERMISSION_REQUIRED` or `AWAITING_INPUT` as its effective state, the debounce is bypassed and `_do_refresh_ui` runs immediately (`controller.py:761-772`).
- Otherwise, a 300ms timer is (re)set. If a previous timer was pending, it is cancelled (`controller.py:773-785`).
- The deferred handler re-reads current state to avoid staleness (`controller.py:787-790`).

### 5.3 Agent Permission Debounce

When an agent enters `PERMISSION_REQUIRED` (`controller.py:648-655`):
- Any existing debounce timer for that agent is cancelled.
- A new 5-second timer is started.
- The event is NOT immediately reflected in the UI.
- If the agent's state changes to anything else within 5 seconds, the timer is cancelled and the new state takes effect immediately.
- If still `PERMISSION_REQUIRED` after 5 seconds, `_flush_agent_perm_debounce` triggers a UI refresh.

## 6. User Interactions

### 6.1 Session Row Interactions

| Input | Target | Condition | Action |
|-------|--------|-----------|--------|
| Left-click | Live session row | State is `READY` | Clears state to `IDLE`, then foregrounds container window |
| Left-click | Live session row | State is not `READY` | Foregrounds container window (re-detects container if missing) |
| Left-click | Ghost session row | Always | Writes `.vscode/tasks.json` if not present, launches `code <cwd>` |
| Double-click | Any session row | `git_status == PUSHED_NOT_MERGED` | Opens PR in browser via `gh pr view --web`; falls back to `gh pr create --web` if no PR exists |
| Middle-click | Any session row | Always | Toggles `flagged` boolean on the session |
| Right-click | Live session row | Always | Context menu: CWD (disabled header), separator, Open PR (if pushed-not-merged), Hide, Clear State |
| Right-click | Ghost session row | Always | Context menu: "Ghost: CWD" (disabled header), separator, Open PR (if pushed-not-merged), Open in VS Code, Hide, Dismiss |

**Single/double-click discrimination** (`main_window.py:1056-1088`): Single-clicks are delayed by 200ms (`_CLICK_DELAY_MS`). If a double-click fires within that window, the pending single-click is cancelled. On Linux, `<ButtonRelease-1>` fires before `<Double-1>`, requiring this delay pattern.

### 6.2 Title Bar Interactions

| Input | Target | Action |
|-------|--------|--------|
| Left-click + release (no drag) | Title icon, emoji, or text | Toggle window shade — collapse to title bar only / expand rows |
| Left-click + drag | Title icon, emoji, or text | Move window (5-pixel drag threshold before movement starts) |
| Middle-click | Title icon, emoji, or text (unshaded, ghosts visible) | Hide all non-flagged ghosts |
| Middle-click | Title icon, emoji, or text (unshaded, ghosts hidden) | Show all ghosts |
| Middle-click | Title icon, emoji, or text (shaded, ghosts hidden) | Unshade window and show all ghosts |
| Middle-click | Title icon, emoji, or text (shaded, ghosts shown) | Unshade window only (ghosts stay shown) |
| Left-click (no drag) | Cost/usage labels | Open 14-day cost history popup |
| Left-click + drag | Cost/usage labels | Horizontal resize (minimum 150px), width persisted to settings |
| Right-click | Any title bar widget | Menu: Sessions (visibility checkboxes), separator, Open... (folder picker -> VS Code), separator, Settings, Restart, Quit |
| Mouse leave | Cost/usage label area | Dismiss cost popup if open |

### 6.3 Title Bar Shade State

When shaded:
- All session rows are hidden (`pack_forget`).
- The title bar background changes to the highest-priority state color, excluding `READY` — only states requiring user action color the shaded bar (`main_window.py:412-413`).
- The eye icon is replaced with a transparent placeholder.
- The emoji image background is recomposited to match the new title bar color.

When unshaded:
- All session rows are restored to their pack order.
- The title bar returns to the default `color_unattached` background.
- The eye icon is restored to its default green.

### 6.4 System Tray

- **Left-click (default action)**: Toggle dashboard window visibility (show/hide).
- **Right-click**: Dynamic menu with: Toggle, separator, "Unhide: (session)" for each hidden session, separator, Settings, Restart, Quit.
- **Icon appearance**: Eye shape (off-center pupil) generated as a PIL image. Outer color reflects the highest-priority actionable state across all non-hidden, non-ghost sessions. `WORKING` is not actionable and is skipped. Gray when no actionable state.

### 6.5 Settings Window

Modal dialog created fresh on each open, destroyed on save or cancel. Position restored from `settings_x`/`settings_y`. Sections:

- **Window**: Always on top (checkbox), Grow upward (checkbox)
- **Rows**: Row height in px (int), Font size in pt (int)
- **Polling**: Poll interval in seconds (int)
- **Startup**: Start on login (checkbox)
- **Filtering**: Ignore CWD regex (text field)
- **Status Colors**: Color swatches for all 5 session states + 5 flag colors + unattached color + window background (each opens `ColorPickerDialog`)

On save: settings are written to disk, `set_run_on_startup` is called, sessions matching the new `ignore_regex` are removed, `MainWindow.apply_settings` is called, UI refreshes.

### 6.6 Cost Popup

Clicking cost/usage labels in the title bar opens a `CostPopup` (`ui/cost_popup.py`):
- Reads daily costs for the last 14 days from `~/.claude/my-claude-stuff-data/session-tracker/<date>/*.json`.
- Each day's cost = `sum(cost.total_cost_usd - _prior_cost)` across all session files for that date.
- Displays a text table with day abbreviation, date, cost, and proportional bar chart.
- Today's row is marked with `*`.
- Shows 14-day total at the bottom.
- Dismissed when the mouse leaves the cost label area.

## 7. Integration Behaviors

### 7.1 Claude Code Sessions

**Discovery**: Reads all `*.json` files in `~/.claude/sessions/`. Expected schema: `{"pid": int, "sessionId": str, "cwd": str, "startedAt": int, "entrypoint": str}`. Files missing `pid` or `sessionId` are skipped. Corrupt JSON is logged as a warning and skipped.

**PID Validation**: `psutil.pid_exists(pid)` + `psutil.Process(pid).name().lower()` must contain "claude". `NoSuchProcess` and `AccessDenied` exceptions return False.

**Hook Events**: Claude Code fires command hooks defined in `~/.claude/settings.json`. The hook config (shipped as `hooks-settings.json`) defines hooks for events including `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`, `UserPromptSubmit`, `Stop`, `StopFailure`, `SubagentStart`, `SubagentStop`, `SessionEnd`, `SessionStart`. Each hook runs `hook_relay.py --debug` which reads the JSON payload from stdin and POSTs it to `http://127.0.0.1:17384/hook`.

### 7.2 VS Code

**Container detection** (Windows): Walk process parent chain via psutil. Match `Code.exe` in process name. Find the "main" VS Code PID by walking up until the parent is not `Code.exe`. Enumerate all windows owned by the main PID via `EnumWindows`, match by CWD folder name in window title.

**Container detection** (Linux): Walk process parent chain. Match `code` or `code-*` in process name. Fallback: check `TERM_PROGRAM=vscode` env var on the bash parent process.

**Ghost session launch**: Write `.vscode/tasks.json` to the CWD (skipped if file exists). The template defines two tasks with the same group `claude-dev`: "claude" (runs `claude` command) and "bash" (runs `exec bash -li`), both set to `runOn: folderOpen`. Then launch `code <folder>` via `shutil.which("code")`.

### 7.3 Git

All git operations use `subprocess.run` with a 2-second timeout (10 seconds for `git fetch`), `capture_output=True`, and `creationflags=config.SUBPROCESS_FLAGS` (Windows: `CREATE_NO_WINDOW`).

| Operation | Command | Called from | Frequency |
|-----------|---------|-------------|-----------|
| Branch detection | Read `.git/HEAD` file directly | `_add_session`, each discovery tick, each hook state change | No subprocess |
| Git status | `git status --porcelain` | `_add_session`, each discovery tick | Every tick |
| Upstream detection | `git remote` + `git symbolic-ref refs/remotes/<remote>/HEAD` | `detect_git_status`, `detect_merged`, `_trunk_branch` | Via callers |
| Unpushed check | `git log --oneline @{u}..HEAD` or `git log --oneline <trunk>..HEAD` | `detect_git_status` | Every tick (if no unstaged/staged) |
| Merge detection | `git merge-base --is-ancestor`, `git cherry`, `git diff --quiet` | `_discovery_tick` | Every tick (if branch exists) |
| Remote fetch | `git fetch <remote>` | `_discovery_tick` | Every ~60 seconds for pushed-not-merged sessions |

### 7.4 GitHub CLI

`gh pr view --web` ��� opens existing PR in browser. Timeout: 10 seconds. If returncode != 0 (no PR found), falls back to `gh pr create --web` (fire-and-forget via Popen). If `gh` is not found (`FileNotFoundError`), logs a warning.

### 7.5 OS Window Manager

**Windows**: `ctypes.windll.user32.SetForegroundWindow(hwnd)`. If the window is minimized, `ShowWindow(hwnd, SW_RESTORE)` first. If `SetForegroundWindow` fails (Windows restricts foreground changes), uses the Alt-key trick: press/release `VK_ALT` around the `SetForegroundWindow` call to acquire foreground permission.

**Linux — VS Code**: `code <cwd>` CLI. VS Code activates the window whose root folder matches `<cwd>`, or opens a new window if none match.

**Linux — Terminals**: `gdbus call --session --dest org.gnome.Shell --object-path /org/gnome/Shell/Extensions/Windows --method org.gnome.Shell.Extensions.Windows.Activate <window_id>`. Window ID is found by `List`-ing all windows via D-Bus, collecting PIDs in the container's process tree, and matching. For VS Code with multiple windows, disambiguates by CWD folder name in window title.

### 7.6 OS Auto-start

**Windows**: Writes `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\ClaudeDashboard` registry value with the startup command. Command uses `pythonw.exe` (falls back to `python.exe` with a warning if `pythonw.exe` not found). Always includes `--debug --log-file <path>`.

**Linux**: Writes `~/.config/autostart/claude-dashboard.desktop` with `Type=Application`, `Name=Claude Dashboard`, `Exec=<startup_cmd>`, `X-GNOME-Autostart-enabled=true`.

Both are toggled in `set_run_on_startup` — called on startup (sync with settings) and on settings save.

## 8. Persistence and Recovery

### 8.1 Settings File

**Path**: `%APPDATA%/claude-dashboard/settings.json` (Windows) or `~/.config/claude-dashboard/settings.json` (Linux).

**Load** (`settings.py:79-108`): Reads JSON, filters to known field names, validates each value's type against the Settings dataclass default (including bool/int discrimination). Invalid fields are silently dropped. Returns defaults if file is missing, corrupt, or unreadable.

**Save** (`settings.py:111-118`): Writes via `atomic_write_json` (tempfile + rename). Logs error on failure.

**Fields**: 26 fields covering window position (x/y for main, settings, color picker), appearance (row_height, row_width, font_size), behavior (always_on_top, grow_up, poll_interval_seconds, run_on_startup), 11 color hex strings, window_bg, text_color (unused — auto-contrast), ignore_regex.

### 8.2 Session State File

**Path**: `~/.claude/claude-dashboard/session-state.json`.

**Structure**: Dict keyed by CWD string. Each entry: `{state: str, hidden: bool, flagged: bool, agents: {agent_id: {state: str, agent_type: str}}}`.

**Save** (`controller.py:1144-1172`): Written on every `_do_refresh_ui` call. For duplicate-CWD sessions, `hidden` is only `True` if ALL sessions with that CWD are hidden. Agent dicts are merged.

**Load** (`controller.py:1134-1142`): Read once at startup. Applied to new sessions in `_apply_saved_state` (flagged, hidden, state, agents). Applied to ghosts in `_create_unattached_from_state`. Buffered hook states override saved state (`controller.py:484-487`).

### 8.3 Atomic Writes

All JSON persistence uses `atomic_write_json` (`file_utils.py:10-33`): Creates parent directories, writes to a tempfile in the same directory, then renames. On any failure, the tempfile is cleaned up and the original file is preserved.

### 8.4 Restart

`_restart()` (`controller.py:1248-1262`): Saves window position, settings, and session state. Stops hook server and tray icon. Calls `root.quit()` then `os.execv(sys.executable, args)` to re-exec the same process with the same arguments. Uses `-m claude_dashboard` to avoid adding the package directory to `sys.path` (which would shadow stdlib `platform`).

## 9. Degradation Behavior

### 9.1 Hook Server Unreachable

`hook_relay.py:83-86`: `urllib.request.urlopen(req, timeout=2)` wrapped in a bare `except Exception`. If the dashboard HTTP server is down or unresponsive, the relay exits silently. No retry. The hook event is lost — the session state will be stale until the next hook event arrives.

### 9.2 Git Subprocess Failures

`session.py:108-123`: `_git_output` wraps `subprocess.run` with `timeout=2`. On `OSError` or `TimeoutExpired`, returns `None`. All callers treat `None` as "no data":
- `detect_git_status`: Returns `GitStatus.CLEAN`.
- `detect_upstream`: Returns `("", "")`.
- `detect_merged`: Returns `False`.

`controller.py:400-403`: If `detect_git_status` takes longer than 0.5 seconds, the result is discarded (the entry keeps its previous `git_status`). This prevents slow git operations (e.g., on network mounts) from blocking the discovery tick.

### 9.3 Git Fetch Failures

`controller.py:381-390`: `git fetch <remote>` has a 10-second timeout. On `OSError` or `TimeoutExpired`, the fetch is silently skipped. The session's merge status will be stale until the next fetch succeeds.

### 9.4 Session File Read Failures

`session.py:53-54`: Individual session files that fail to parse (`JSONDecodeError`) or read (`OSError`) are logged as warnings and skipped. The session is not discovered.

`session.py:34-35`: If the sessions directory does not exist, returns an empty list.

### 9.5 Settings File Failures

`settings.py:81-108`: Returns a `Settings()` with all defaults on any exception during load.

`settings.py:111-118`: Logs an error on write failure. The in-memory settings remain correct; only persistence is lost.

### 9.6 State File Failures

`controller.py:1134-1142`: Returns empty dict on `FileNotFoundError`, `JSONDecodeError`, or `OSError`. Dashboard starts with no saved state — all sessions appear as fresh.

`controller.py:1169-1172`: Logs debug on write failure. State is lost for that save cycle; the next save may succeed.

### 9.7 VS Code Launch Failures

`controller.py:967-985`: `shutil.which("code")` returns `None` if VS Code CLI is not in PATH — logs a warning with installation instructions. `subprocess.Popen` catches `OSError` and logs a warning.

### 9.8 GitHub CLI Failures

`controller.py:1001-1026`: `gh pr view --web` has a 10-second timeout. `FileNotFoundError` (gh not installed) is logged as a warning. `TimeoutExpired` is logged as a warning. Other `OSError` is logged as a warning. No fallback — the PR is not opened.

### 9.9 Window Foregrounding Failures

**Windows** (`platform/windows.py:136-159`): `SetForegroundWindow` returns a boolean. On failure, the Alt-key trick is attempted. If that also fails, returns `False`. The controller logs a warning (`controller.py:860`).

**Linux — D-Bus** (`platform/linux.py:160-189`): `gdbus call` has a 3-second timeout. `FileNotFoundError` (gdbus not found), `TimeoutExpired`, and non-zero return codes all return `False`.

**Linux — VS Code CLI** (`platform/linux.py:242-259`): `code <cwd>` has a 5-second timeout. `FileNotFoundError` and `TimeoutExpired` return `False`.

**Linux fallthrough** (`platform/linux.py:279-284`): If no foreground method succeeds, logs debug and returns `False`.

### 9.10 OS Auto-start Failures

**Windows** (`startup.py:59-77`): Registry operations wrapped in try/except. Logs warning on failure. Dashboard continues running; only auto-start registration is affected.

**Linux** (`startup.py:96-115`): File write wrapped in try/except for `OSError`. Logs warning on failure.

### 9.11 Cost and Usage Data Failures

`controller.py:71-90`: `read_daily_cost()` iterates `*.json` in the tracker directory. Individual file failures (`JSONDecodeError`, `OSError`) are silently skipped. Missing directory returns 0.0.

`controller.py:93-103`: `read_usage_limits()` returns empty dict on any failure.

### 9.12 Tray Icon Failures

`tray.py:107-117`: `update_tray_icon` catches `Exception` and logs debug. The tray icon keeps its previous color.

`controller.py:296-299`: Tray thread crashes are caught and logged. The dashboard continues running without a tray icon.

## 10. Behavioral Nuances

### 10.1 IDLE-to-READY Intercept

The `IDLE` state is unreachable for main process events via hooks. When `map_event_to_state` returns `IDLE` (on `Stop`/`StopFailure`), the controller intercepts it and stores `READY` instead (`controller.py:672-673`). `READY` is a user-actionable state indicating "Claude finished, check results." It is cleared to `IDLE` by left-clicking the row (`controller.py:847-849`). This intercept does not apply to agents — agent `IDLE` is stored as-is.

### 10.2 PostToolUseFailure Suppression During Permission

When the main process state is `PERMISSION_REQUIRED` and a `PostToolUseFailure` event arrives with `new_state=WORKING`, the event is suppressed (`controller.py:679-691`). This handles parallel tool calls where one tool needs permission while others fail — the failures should not clear the permission-required state.

### 10.3 CWD-Based Session Matching

When a hook event's `session_id` is not found in `_session_id_to_pid`, the controller falls back to matching by CWD (`controller.py:609-615`). This handles resumed sessions where Claude Code reuses a CWD but generates a new session ID. The fallback registers the new session ID for future lookups.

### 10.4 Text Color: Auto-Contrast, Not Configurable

`main_window.py:112-123`: Text color is computed per-row from the background color using the W3C sRGB relative luminance formula. The `text_color` field exists in Settings for backward compatibility but is unused — auto-contrast overrides it. Two text colors are defined: `_TEXT_LIGHT` (#f5f0e8, warm white) for dark backgrounds, `_TEXT_DARK` (#1a1520, cool near-black) for light backgrounds.

### 10.5 Git Status Timeout Guard

`controller.py:400-403`: `detect_git_status` is timed with `time.monotonic()`. If the call takes longer than 0.5 seconds, the result is discarded and the session keeps its previous `git_status`. This prevents network-mounted or large repos from blocking the UI update cycle. The guard is also applied during initial session registration (`controller.py:452-459`), where it only logs a warning rather than discarding.

### 10.6 Duplicate-CWD Session Handling

Multiple live sessions can share the same CWD. State persistence handles this: `hidden` is only written as `True` if ALL sessions with that CWD are hidden (`controller.py:1156-1158`). Agent dicts are merged across sessions sharing a CWD (`controller.py:1160`).

### 10.7 Ghost Toggle and Shade Interaction

The middle-click ghost toggle has four-case interaction with the shade state (`main_window.py:520-536`):
- Shaded + ghosts hidden: Unshade and show ghosts (`force_show=True`).
- Shaded + ghosts shown: Unshade only (ghosts stay shown).
- Unshaded + ghosts hidden: Show ghosts.
- Unshaded + ghosts shown: Hide non-flagged ghosts. Flagged ghosts are never hidden by this toggle.

### 10.8 Grow-Up Window Positioning

When `grow_up=True`, the window's bottom edge is anchored. As sessions are added/removed, the window grows upward (y decreases). The bottom-y position is tracked explicitly (`_grow_up_bottom_y`) and saved as the y-coordinate in settings. On restore, the saved y is treated as the bottom edge, and the top-y is computed from `bottom_y - window_height` (`main_window.py:481-482`).

Compositor drift detection: After setting geometry, the actual position is compared to the requested position. If they differ, a warning is logged. This detects Wayland compositors that reposition dock-type windows.

### 10.9 Non-Interactive Session Auto-Hide

Sessions with `entrypoint != "cli"` (e.g., `claude -p` for piped/non-interactive usage) are automatically hidden on discovery (`controller.py:477-479`). They still receive hook events and appear in the Sessions visibility menu.

### 10.10 VS Code Tasks.json Template

Ghost left-click writes `.vscode/tasks.json` only if the file does not exist (`controller.py:168-186`). The template is never updated or overwritten — once written, the file belongs to the user. The template defines two tasks in the same `claude-dev` group (split terminal): `claude` (auto-runs on folder open) and `bash -li` (interactive shell).

### 10.11 Agent Auto-Wake Suppression

`UserPromptSubmit` events do not clear agents (`controller.py:663-667`). This is because auto-wake events (fired after each `SubagentStop`) are indistinguishable from real user prompts and would prematurely clear active agents. Agents are removed only by `SubagentStop` or on PID death.

### 10.12 Shaded Title Bar Color

When shaded, the title bar color reflects the highest-priority state, but `READY` is excluded (`main_window.py:412-413`). The rationale: shaded mode is for "at a glance" awareness — only states requiring user action should color the bar. `READY` means "Claude finished" but doesn't require immediate attention, so the shaded bar remains the default dark color.

### 10.13 Single-Instance Enforcement

The port-binding check in `__main__.py:14-24` uses `SO_REUSEADDR` on the test socket, then attempts to bind to `127.0.0.1:17384`. If binding fails (`OSError`), another instance is running. The socket is closed immediately — it's a probe, not the actual server socket. The hook server creates its own socket later.
