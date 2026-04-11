# Behavioral Specification — Claude Dashboard

## 1. Purpose

Claude Dashboard is a desktop status monitor for Claude Code sessions. It provides at-a-glance visibility into what every running Claude Code instance is doing — working, waiting for input, requesting permission, or finished — and offers one-click navigation to the relevant window.

---

## 2. Operational Overview

The dashboard runs as a single desktop process with a borderless always-on-top window and a system tray icon. It discovers Claude Code sessions by polling the file system and receives real-time state updates via HTTP hooks that Claude Code fires on every state transition.

Sessions appear as colored rows in the window. Each row shows the project directory, git branch, status emoji, and container (VS Code or terminal). The user interacts with rows via mouse clicks to foreground windows, toggle flags, open PRs, or manage visibility.

When a Claude Code session exits, its row becomes a "ghost" — a persistent placeholder that can relaunch Claude in that project directory via VS Code.

---

## 3. Startup

### 3.1 Single-Instance Enforcement

On launch, the dashboard attempts to bind a socket to port 17384. If the port is already in use, the dashboard exits immediately — only one instance can run.

### 3.2 Initialization Sequence

1. Parse CLI arguments: `--debug`, `--ttl <seconds>`, `--log-file <path>`
2. Configure logging (file or stderr, level based on flags)
3. Install global exception handler (routes uncaught exceptions to logger)
4. On Windows: set DPI awareness to prevent blurry rendering
5. Load user settings from disk (or defaults if missing/corrupt)
6. Load saved session state from disk
7. Create hidden Tkinter root window
8. Create main dashboard window (borderless, positioned from saved settings)
9. Create hook HTTP server (not yet started)
10. Sync OS auto-start with settings (write/remove .desktop or Registry entry)
11. Create system tray icon
12. Start hook server on daemon thread
13. Start tray icon on daemon thread
14. Execute first discovery tick
15. Enter Tkinter main loop

### 3.3 Auto-Start

When enabled, the dashboard registers itself to start on OS login:
- **Linux:** Writes an XDG `.desktop` file to `~/.config/autostart/`
- **Windows:** Writes a Registry key to `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`

---

## 4. Session Discovery

### 4.1 Polling

Every `poll_interval_seconds` (default 3s), the dashboard reads all `*.json` files from `~/.claude/sessions/`. Each file represents a Claude Code session.

**Session file fields:**
- `pid` — OS process ID
- `sessionId` — unique session identifier
- `cwd` — working directory
- `startedAt` — Unix timestamp (milliseconds)
- `entrypoint` — how Claude was launched ("cli", "api", etc.)

### 4.2 PID Validation

For each discovered session, the dashboard verifies:
1. The PID exists (`psutil.pid_exists`)
2. The process name contains "claude"

Both checks must pass for a session to be considered alive.

### 4.3 Ignore Regex

Sessions whose CWD matches the `ignore_regex` setting are excluded entirely — never displayed, never tracked. The regex is evaluated on every discovery tick and on settings change.

### 4.4 Auto-Hide

Sessions with `entrypoint != "cli"` (e.g., API-launched sessions) are automatically hidden on first discovery. They still exist in the state map and can be unhidden via the Sessions menu.

### 4.5 Session Registration

When a new session is discovered:
1. Detect container (VS Code, terminal, etc.) from parent process chain
2. Detect git branch from `.git/HEAD`
3. Detect git working tree status
4. Check if a ghost exists for this CWD — if so, replace it (inherit flag state, force unhide)
5. If no ghost, apply saved state from previous run (flags, visibility, state, agents)
6. Replay any buffered hook events that arrived before discovery
7. Add to session map

---

## 5. Session States

### 5.1 State Enum

| State | Meaning | Visual |
|-------|---------|--------|
| WORKING | Claude is actively processing | Gray row |
| READY | Claude finished a turn, awaiting user | Green row |
| IDLE | Baseline inactive state | Dark row |
| AWAITING_INPUT | Claude asked the user a question (AskUserQuestion tool) | Amber/yellow row |
| PERMISSION_REQUIRED | Claude needs tool approval | Amber/yellow row |

### 5.2 State Priority

Lower number = higher priority. Used for effective state calculation and tray icon color.

1. PERMISSION_REQUIRED (0)
2. AWAITING_INPUT (1)
3. WORKING (2)
4. READY (3)
5. IDLE (4)

### 5.3 Effective State

Each session's effective state is the highest-priority state across its main process and all tracked agents. This is what determines the row color and tray icon.

### 5.4 State Transitions

All state changes are driven by hook events, except:
- **IDLE → READY:** The controller intercepts IDLE (from Stop/StopFailure events) and maps it to READY. READY is cleared to IDLE by the user clicking the row
- **Discovery → IDLE:** New sessions start as IDLE until their first hook event
- **Clear State (context menu):** Resets to IDLE and clears all agents

### 5.5 State Transition Guards

- **PostToolUseFailure during PERMISSION_REQUIRED:** Ignored. When Claude fires parallel tool calls, one may need permission while others fail. The permission state must not be overwritten by a failure on a different tool
- **Duplicate state:** If the new state equals the current state, the event is silently dropped (no UI refresh)

---

## 6. Hook Event Processing

### 6.1 Event Chain

```
Claude Code → command hook fires → hook_relay.py reads JSON from stdin
→ POST to http://127.0.0.1:17384/hook → HookServer parses JSON
→ maps event to StatusState → dispatches callback to main thread
→ AppController applies state change → UI refresh
```

### 6.2 Event-to-State Mapping

| Hook Event | Default State | Condition Override |
|------------|---------------|-------------------|
| UserPromptSubmit | WORKING | — |
| PreToolUse | WORKING | AWAITING_INPUT if tool = AskUserQuestion |
| PostToolUse | WORKING | — |
| PostToolUseFailure | WORKING | Suppressed if current state = PERMISSION_REQUIRED |
| PermissionRequest | PERMISSION_REQUIRED | AWAITING_INPUT if tool = AskUserQuestion |
| Stop | READY | Controller intercepts IDLE → READY |
| StopFailure | READY | Controller intercepts IDLE → READY |
| SubagentStart | WORKING | Registers agent entry |
| SubagentStop | (removal) | Removes agent; no state change |
| SessionEnd | (ghost) | Converts live session to ghost |

### 6.3 Session ID Resolution

Hook events arrive with a `session_id`. The controller maintains a reverse lookup map (`session_id → PID`).

**Fallback:** If the session_id is not found (common with resumed sessions), the controller matches by CWD instead and registers the new session_id for future lookups.

**Buffering:** If neither lookup succeeds (session not yet discovered), the event is buffered in `_pending_hook_states` and replayed when the session is registered.

### 6.4 Hook Relay Behavior

The relay script is invoked per-event by Claude Code's hook system. It:
1. Reads the complete JSON payload from stdin
2. POSTs to `http://127.0.0.1:17384/hook` with a 2-second timeout
3. Exits immediately (exit code always 0)

If the dashboard is not running, the POST fails silently. Claude Code is never blocked or affected.

When `--debug` is enabled (default in shipped config), raw payloads are appended to a JSONL log file with ISO 8601 timestamps.

---

## 7. Agent Tracking

### 7.1 Agent Lifecycle

Agents are subprocesses spawned by Claude Code sessions (via the Agent tool). The dashboard tracks them per-session.

- **SubagentStart:** Creates or updates an agent entry with `agent_id`, `state`, `agent_type`
- **SubagentStop:** Removes the agent entry
- **Other events with `agent_id`:** Updates the agent's state

Agents are never explicitly listed in the UI as separate rows. Their state contributes to the session's effective state, and their count is shown as `(+N)` in the row.

### 7.2 Agent Permission Debounce

Agent permission requests are debounced for 5 seconds before affecting the UI. Rationale: agents are typically run via skills that auto-resolve permission prompts, so most agent permission events are transient.

- If the agent sends a non-permission event within 5s, the debounce timer is cancelled
- If the agent is stopped within 5s, the debounce timer is cancelled
- If still pending after 5s, the UI updates to show PERMISSION_REQUIRED

Main process permission requests are **not** debounced — they always update the UI immediately.

### 7.3 Agent Cleanup

Agents are removed individually via SubagentStop. They are **not** cleared on UserPromptSubmit because auto-wake events (fired after each SubagentStop) are indistinguishable from real user prompts and would prematurely clear active agents.

When a session's PID dies, all its agents are implicitly cleaned up (the entire session entry is removed).

**There is no TTL or timeout-based agent cleanup.** This is an intentional design decision — agents can run for hours, and there is no reliable signal to distinguish "still working" from "orphaned."

---

## 8. Ghost Sessions

### 8.1 What Ghosts Are

When a Claude Code session exits (PID dies or SessionEnd hook fires), the dashboard converts it to a "ghost" — an unattached placeholder with a synthetic negative PID. Ghosts persist until:
- The user dismisses them via right-click context menu
- The CWD directory is deleted (pruned on next discovery tick)
- A new live session starts in the same CWD (ghost is replaced)

### 8.2 Ghost Behavior

- Ghost rows show the "unattached" emoji (👻) and use the `color_unattached` background
- Left-click on a ghost: writes `.vscode/tasks.json` (if not present) and launches VS Code
- Right-click on a ghost: Dismiss, Open in VS Code, Hide, Open PR (if pushed-not-merged)
- Ghosts preserve their `flagged` state from the live session

### 8.3 Ghost Visibility Toggle

Middle-clicking the title bar toggles ghost visibility:
- Show: All non-flagged ghosts become visible
- Hide: All non-flagged ghosts become hidden
- **Flagged ghosts are never hidden by this toggle** — the manual flag protects them

**Shade interaction:** If the window is shaded (collapsed) and ghosts are hidden, middle-click both unshades and shows ghosts. If shaded and ghosts are visible, it only unshades.

### 8.4 State Restoration

On startup, the dashboard reads `session-state.json` and creates ghost entries for every CWD that has saved state but no live session. This preserves flags, visibility, and state across dashboard restarts.

---

## 9. Git Integration

### 9.1 Branch Detection

Reads `.git/HEAD` directly (no subprocess). If HEAD points to the trunk branch (detected via `<remote>/HEAD`), the branch name is suppressed — only non-trunk branches are displayed.

### 9.2 Working Tree Status

Detected per-session per-tick via `git status --porcelain` with a 2-second timeout. Priority order:

| Priority | Status | Meaning | Eye Icon Color |
|----------|--------|---------|----------------|
| 1 | UNSTAGED_CHANGES | Working tree modifications | Green |
| 2 | STAGED_UNCOMMITTED | Staged but not committed | Amber |
| 3 | COMMITTED_NOT_PUSHED | Commits ahead of upstream | Red |
| 4 | PUSHED_NOT_MERGED | Branch pushed, not in trunk | Light blue |
| 5 | CLEAN | Nothing to show | (transparent) |

**Performance guard:** If `detect_git_status()` takes longer than 500ms, the result is discarded to avoid blocking the UI thread.

### 9.3 Merge Detection

For sessions with a non-trunk branch, the dashboard checks if the branch has been merged into trunk using three strategies:

1. **Ancestor check:** `git merge-base --is-ancestor <branch> <trunk>` — detects regular merges
2. **Cherry check:** `git cherry <trunk> <branch>` — detects rebase merges (all commits have patch-equivalent)
3. **Diff check:** `git diff --quiet <trunk> <branch>` — detects squash merges (trees identical)

When merged, the branch name text turns red (`#ef4444`).

### 9.4 Upstream Discovery

The dashboard detects the remote and trunk branch dynamically:
1. List all remotes (`git remote`)
2. For each remote, check `git symbolic-ref refs/remotes/<remote>/HEAD`
3. First successful result becomes the upstream (e.g., `origin/main`)

No hardcoded branch names — works with main, master, develop, or any trunk convention.

### 9.5 Periodic Fetch

For sessions with git status PUSHED_NOT_MERGED, the dashboard runs `git fetch <remote>` approximately once per minute (rate-limited by tick counter). This keeps merge detection current.

---

## 10. User Interactions

### 10.1 Row Interactions

| Action | Target | Behavior |
|--------|--------|----------|
| Left-click | Live row | Foreground the session's VS Code/terminal window. If state is READY, clear to IDLE |
| Left-click | Ghost row | Write `.vscode/tasks.json`, launch VS Code |
| Double-click | Any row | Open PR in browser if PUSHED_NOT_MERGED (via `gh pr view --web`); fallback to create-PR page |
| Middle-click | Any row | Toggle manual flag on/off |
| Right-click | Live row | Context menu: Open PR (if applicable), Hide, Clear State |
| Right-click | Ghost row | Context menu: Open PR (if applicable), Open in VS Code, Hide, Dismiss |

### 10.2 Title Bar Interactions

| Action | Behavior |
|--------|----------|
| Left-click | Toggle shade (collapse to title bar only / expand to show rows) |
| Middle-click | Toggle ghost visibility (flagged ghosts exempt) |
| Right-click | Context menu: Sessions submenu (visibility checkboxes), Open..., Settings, Restart, Quit |
| Drag | Move window |

### 10.3 Cost/Usage Labels

| Action | Behavior |
|--------|----------|
| Click (no drag) | Open 14-day cost history popup |
| Horizontal drag | Resize window width (min 150px, persisted to settings) |

### 10.4 System Tray

| Action | Behavior |
|--------|----------|
| Left-click | Toggle main window visibility |
| Right-click | Menu: Toggle, Unhide items for hidden sessions, Settings, Restart, Quit |

---

## 11. Visual Design

### 11.1 Window

- Borderless, always-on-top (configurable)
- Linux: dock window type (visible on all virtual desktops)
- Grow-up mode: window expands upward from a fixed bottom edge
- Background: configurable (`#1a1a1a` default)

### 11.2 Title Bar

- Left side: Eye icon + chef's kiss emoji (PNG, unicode fallback) + "Claude Dashboard"
- Right side: Session counts `{N} (+hidden) [+ghosts]`, daily cost `$X.XX`, 5h/7d usage percentages
- When shaded: background color reflects highest-priority action-required state (excludes READY to keep title minimal)
- When expanded: background color is always `color_unattached`

### 11.3 Session Rows

- Background color determined by effective state (configurable per-state)
- Text color auto-calculated via W3C sRGB contrast — not user-configurable
  - Light backgrounds → dark text (`#1a1520`)
  - Dark backgrounds → light text (`#f5f0e8`)
- Container label (VS Code, Term, etc.) right-aligned in dim gray (`#888888`)
- CWD paths shortened: `git-worktrees/` → `.../`

### 11.4 Eye/Flag Icon

Each row has an eye-shaped icon with two visual channels:
- **Outer eye color:** Git working tree status (green=unstaged, amber=staged, red=unpushed, blue=unmerged)
- **Pupil color:** Manual flag (purple when flagged, transparent when not)
- Both transparent when no git status and no flag

### 11.5 Status Emojis

Each state has a distinct emoji (loaded as PNG from assets):
- Working: 🔄
- Ready: ✅
- Idle: 😐
- Awaiting Input: ❓
- Permission Required: 🔐
- Unattached: 👻

### 11.6 Tray Icon

64x64 eye shape with offset pupil (asymmetric, "looking right"). Color = highest-priority actionable state. Black outline prevents color bleed when downscaled. Only actionable states are reflected (permission, awaiting input, ready, idle) — working is excluded.

---

## 12. UI Refresh and Debouncing

### 12.1 Standard Debounce

State changes are debounced at 300ms. If multiple hook events arrive within that window, only the final state is rendered.

### 12.2 Priority Bypass

If any visible session has PERMISSION_REQUIRED or AWAITING_INPUT as its effective state, the debounce is bypassed and the UI refreshes immediately. This ensures action-required states are visible without delay.

### 12.3 Refresh Scope

Each refresh:
1. Collects all visible (non-hidden) session entries
2. Builds `SessionRow` tuples for the UI
3. Pushes to MainWindow for rendering
4. Updates tray icon color if priority state changed
5. Updates title bar (cost, usage, counts)
6. Saves session state to disk

---

## 13. Settings

### 13.1 Configurable Fields

**Window behavior:** Always on top, grow upward
**Layout:** Row height (px), font size (pt), row width (px)
**Polling:** Interval in seconds
**Startup:** Run on login
**Filtering:** Ignore CWD regex
**Colors:** 12 configurable colors (5 status states, 5 flag states, unattached background, window background)

### 13.2 Settings Flow

1. User opens Settings (via menu or tray)
2. Modal dialog shows all fields with current values
3. Color fields open a color picker (palette grid + hex entry)
4. Apply: saves to disk, applies to UI, keeps dialog open
5. Save: saves to disk, applies to UI, closes dialog
6. Cancel: saves only dialog position, discards changes

### 13.3 Settings Application

When settings are applied:
1. Save to disk (atomic JSON write)
2. Update OS auto-start registration
3. Remove sessions matching new ignore_regex
4. Rebuild UI fonts, colors, and window attributes
5. Recompute geometry
6. Full UI refresh

### 13.4 Persistence

Settings are stored as a JSON file at the platform-specific config path. Unknown fields are stripped on load. Invalid types are skipped with a log warning. Missing or corrupt files fall back to defaults.

**Dialog and picker positions** are saved to settings on close — so they reopen where the user left them.

---

## 14. State Persistence

### 14.1 What's Persisted

Per-CWD snapshot saved to `~/.claude/claude-dashboard/session-state.json`:
- Session state (WORKING, READY, IDLE, etc.)
- Hidden flag (boolean)
- Manual flag (boolean)
- Agent states and types

### 14.2 Write Frequency

Written on every UI refresh (which happens on every hook event and every discovery tick). Atomic write (temp file + rename).

### 14.3 Multi-Session Merge

When multiple live sessions share the same CWD:
- `hidden` is only `true` if ALL sessions with that CWD are hidden
- Agent maps are merged
- Last state wins for the state field

### 14.4 Restore on Startup

On the first discovery tick, for every CWD in saved state that has no live session, a ghost is created with the saved flags and visibility.

---

## 15. Window Foregrounding

### 15.1 Container Detection

The dashboard walks the parent process chain of each Claude Code PID to identify what container it's running in:
- VS Code (code/code.exe)
- Terminal emulators (gnome-terminal, konsole, alacritty, kitty, wezterm, ghostty, etc.)
- Multiplexers (tmux, screen)
- Windows-specific (Windows Terminal, cmd.exe, Git Bash/mintty)

Fallback: checks `TERM_PROGRAM` environment variable.

### 15.2 Foregrounding (Linux/Wayland)

- **VS Code:** Launches `code <cwd>` CLI — this foregrounds the correct VS Code window
- **Terminal:** Uses GNOME Shell `window-calls` D-Bus extension to list windows by PID and activate the matching one

For multi-window VS Code instances, the dashboard disambiguates by matching the CWD folder name against window titles.

### 15.3 Foregrounding (Windows)

- Finds the main VS Code process (the one whose parent is NOT another Code.exe)
- Enumerates all visible windows owned by that PID
- Matches by CWD folder name in window title
- Uses `SetForegroundWindow` with an Alt-key workaround for focus stealing prevention
- Restores minimized windows before foregrounding

---

## 16. VS Code Integration

### 16.1 Tasks.json Auto-Generation

When launching VS Code for a ghost session or via "Open in VS Code," the dashboard writes `.vscode/tasks.json` with two tasks:
1. **claude** — runs `claude` command
2. **bash** — runs `exec bash -li`

Both tasks:
- Are in the same group (`claude-dev`) so VS Code splits them side-by-side
- Have `runOptions.runOn = "folderOpen"` so they start automatically when the folder opens
- Use `isBackground: true` so they don't block VS Code startup

The file is only written if it doesn't already exist (user customizations are preserved).

### 16.2 PR Operations

Double-click on a PUSHED_NOT_MERGED session runs:
1. `gh pr view --web` — opens existing PR in browser
2. If no PR exists (non-zero exit): `gh pr create --web` — opens create-PR page

---

## 17. Shutdown and Restart

### 17.1 Quit

1. Save window position to settings
2. Save settings to disk
3. Stop hook server (join thread with 1s timeout)
4. Stop tray icon
5. Exit Tkinter main loop

### 17.2 Restart

Same as quit, plus:
1. Save session state to disk
2. Re-execute the process via `os.execv(python, ["-m", "claude_dashboard", ...args])`

The next startup reads the saved session state and recreates ghosts for all CWDs that were being tracked.

### 17.3 TTL Mode

When launched with `--ttl <seconds>`, the dashboard auto-quits after that duration. Used for testing.

---

## 18. External Data Sources

### 18.1 Daily Cost

Reads `~/.claude/my-claude-stuff-data/session-tracker/{YYYY-MM-DD}/*.json`. Each file contains `cost.total_cost_usd` and `_prior_cost` (for cross-day sessions). Displayed in the title bar as `$X.XX`. Refreshed every ~30 seconds (10 × poll interval).

### 18.2 Usage Limits

Reads `~/.claude/my-claude-stuff-data/statusline-cache/oauth_usage.json`. Displays 5-hour and 7-day utilization percentages in the title bar. Refreshed on every discovery tick.

---

## 19. Behavioral Nuances

### 19.1 CWD-Based Fallback Matching

When a hook event arrives with a session_id that isn't in the reverse lookup, the controller falls back to matching by CWD. This handles resumed sessions where Claude Code assigns a new session_id but keeps the same working directory.

### 19.2 Hook Event Buffering

Events that arrive before their session is discovered are buffered by session_id. On session registration, buffered events are replayed. This prevents losing the initial state of fast-starting sessions.

### 19.3 Git Status Timeout Guard

If `detect_git_status()` takes longer than 500ms, the result is discarded. This prevents slow git operations (large repos, network mounts) from freezing the UI.

### 19.4 Wayland Position Tracking

Wayland compositors can report stale or zero-value window positions. The dashboard tracks position independently (not relying on `winfo_x/y`) and rejects suspicious values (x=0, y=0, height<=1).

### 19.5 Context Menu State Machine

The context menu tracks whether it's currently open (`_context_menu_open`). If right-click is fired while the menu is open, it closes instead of opening a new one. This prevents menu stacking.

### 19.6 Multiple Sessions Same CWD

When multiple Claude Code sessions run in the same directory:
- Each has its own row
- State persistence merges them: `hidden` is only true if ALL are hidden
- Ghost creation is suppressed while any live session exists for that CWD
- When the last session dies, one ghost is created

### 19.7 Auto-Wake Agent Confusion

When an agent stops, Claude Code fires a UserPromptSubmit event on the main process. The dashboard does NOT clear agents on UserPromptSubmit because this auto-wake is indistinguishable from a real user prompt. Agents are only removed by explicit SubagentStop events.
