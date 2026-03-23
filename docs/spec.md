# Feature Specification: Claude Dashboard

**Created**: 2026-03-22
**Status**: Draft
**Version**: 0.2.0
**Input**: Voice transcript (see `raw-prompt.md`)

## User Scenarios & Testing

### User Story 1 — Session Awareness Dashboard (Priority: P1)

As a user running multiple Claude Code sessions across terminals and VS Code instances, I want a persistent, always-available dashboard that shows every active session with its workspace and current status, so I can see at a glance what needs my attention without hunting through windows.

**Why this priority**: This is the core value proposition. Without session discovery and display, nothing else matters. Replaces the failed OS notification approach (noisy, ungrouped, not actionable).

**Independent Test**: Launch 3+ Claude sessions in different directories. The dashboard shows all of them with correct CWD and status indicators. Close one session — it disappears or shows as stopped.

**Acceptance Scenarios**:

1. **Given** the dashboard is running and 3 Claude sessions are active, **When** I look at the dashboard, **Then** I see 3 rows, each showing the session's workspace directory and a status indicator.
2. **Given** a Claude session is actively processing, **When** I view its row, **Then** it shows the "working" indicator (🔄) with the corresponding row color.
3. **Given** a Claude session has finished responding (end_turn) and is waiting for the next user prompt, **When** I view its row, **Then** it shows the "idle" indicator (⏸️).
3a. **Given** a Claude session has asked the user a question (via AskUserQuestion tool) and is actively waiting for an answer, **When** I view its row, **Then** it shows the "awaiting input" indicator (❓).
4. **Given** a Claude session has a pending permission prompt, **When** I view its row, **Then** it shows the "permission required" indicator (⚠️).
5. **Given** a Claude session terminates, **When** the next poll cycle runs, **Then** the row disappears (removed, not shown as stopped).
6. **Given** the dashboard cannot determine a session's status, **When** I view its row, **Then** it shows an "unknown" indicator (🤷).
7. **Given** I have configured custom row height, width, and colors in settings, **When** the dashboard renders, **Then** it uses my configured values.
8. **Given** text in a row exceeds the configured fixed width, **When** the row renders, **Then** the text is truncated (not wrapped, not expanding the row width).

---

### User Story 2 — Session Navigation (Priority: P1)

As a user with sessions spread across VS Code instances and virtual desktops, I want to click a session row and have the containing application (VS Code window, terminal) become the foreground window, so I can immediately interact with that session without manually searching.

**Why this priority**: The primary pain point is finding the right session among many. Discovery without navigation solves only half the problem.

**Independent Test**: Run Claude in two different VS Code windows on different virtual desktops. Click each row — the correct VS Code instance comes to the foreground, switching virtual desktops if needed.

**Acceptance Scenarios**:

1. **Given** a Claude session is running inside VS Code (Git Bash) on Windows, **When** I click its row, **Then** that VS Code window becomes the foreground window.
2. **Given** a Claude session is running in a terminal emulator on Linux, **When** I click its row, **Then** that terminal window becomes the foreground window.
3. **Given** the target window is on a different virtual desktop, **When** I click the row, **Then** the OS switches to that desktop and foregrounds the window.
4. **Given** a Claude session is running inside `screen` on Linux, **When** I click its row, **Then** the terminal hosting that screen session is foregrounded. **Note**: We foreground the terminal window only — we do not attempt to switch to a specific `screen` window within the screen session.
5. **Given** two Claude sessions share the same containing process (e.g., two terminals in one VS Code instance), **When** I click either row, **Then** the same VS Code window is foregrounded. Each session remains an independent row.
6. **Given** I press and hold mouse on a row and drag less than 5px, **When** I release, **Then** it is treated as a click and the containing window is foregrounded.
7. **Given** I press and hold mouse on a row and drag 5px or more, **When** I release, **Then** it is treated as a window drag (repositioning the dashboard) and no navigation occurs.

---

### User Story 3 — Persistent Settings & Window Position (Priority: P1)

As a user, I want my dashboard configuration (row dimensions, colors, emoji choices, window position, poll interval) to persist across restarts, so I don't have to reconfigure every time.

**Why this priority**: Fundamental UX requirement. Without persistence, the dashboard becomes a nuisance to use.

**Independent Test**: Configure settings, reposition the window, restart the dashboard. Everything restores.

**Acceptance Scenarios**:

1. **Given** I move the dashboard window to a specific position, **When** I close and reopen it, **Then** it appears at the same position.
2. **Given** I change row height/width/colors/poll interval in settings, **When** I restart, **Then** settings are preserved.
3. **Given** a settings file does not exist, **When** the dashboard starts, **Then** it uses sensible defaults and creates the settings file.
4. **Given** I right-click the system tray icon and select "Settings", **When** the settings window opens, **Then** I can edit all configurable values (row height, width, colors, emojis, poll interval, always-on-top) in a modal dialog. Each status color label MUST display its corresponding emoji next to it for visual correlation (e.g., "🔄 Working", "⏸️ Idle").

---

### User Story 4 — Container Process Indicator (Priority: P2)

As a user, I want each session row to show an icon or indicator of the containing process (VS Code, terminal emulator, etc.) so I know what window will appear when I click the row.

**Why this priority**: Usability — with many sessions, knowing whether clicking will bring up VS Code vs a terminal vs something else helps set expectations. The process tree already provides this data.

**Independent Test**: Run Claude in VS Code and in a standalone terminal. Each row shows the correct container icon.

**Acceptance Scenarios**:

1. **Given** a Claude session is running inside VS Code, **When** I view its row, **Then** I see a VS Code icon or label.
2. **Given** a Claude session is running in a standalone terminal, **When** I view its row, **Then** I see a terminal icon or label.
3. **Given** the containing process cannot be identified, **When** I view its row, **Then** a generic/unknown icon is shown.

---

### User Story 5 — System Tray Persistence (Priority: P2)

As a user, I want the dashboard to minimize to the system tray when closed, and indicate via the tray icon when something needs my attention, so it stays out of the way but remains accessible.

**Why this priority**: The dashboard should be persistent but not take up taskbar space. The d4-timer-w11 reference project already implements this pattern with pystray.

**Independent Test**: Close the dashboard window. It appears in the system tray. A session enters "permission required" state. The tray icon changes to indicate attention needed. Double-click the tray icon to restore the window.

**Acceptance Scenarios**:

1. **Given** the dashboard is open, **When** I close the window, **Then** it minimizes to the system tray and continues running.
2. **Given** any session enters a state requiring attention (PermissionRequired or AwaitingInput), **When** I look at the tray icon, **Then** it shows a notification indicator (dot, badge, or color change). Note: Idle does NOT trigger the attention indicator — only states where Claude is blocked waiting for a user response.
3. **Given** no sessions need attention, **When** I look at the tray icon, **Then** it shows the default/idle icon.
4. **Given** the dashboard is in the tray, **When** I double-click the tray icon, **Then** the dashboard window reopens at its saved position.

---

### Edge Cases

- **Session starts/stops between polls**: The dashboard shows state as of the last poll. If a session started and stopped between two polls, it was never visible. This is acceptable — if the session file was cleaned up, there's nothing to show.
- **Unknown status**: If no hook event has been received for a session, show the "unknown" indicator (🤷). Do not guess.
- **Process foregrounding fails**: Show a brief error or do nothing. Do not crash.
- **Two sessions share the same CWD**: Display as independent rows. Order deterministically by PID (lower PID first). Both rows click-navigate to the same containing window if they share a parent process.
- **No running Claude sessions**: Show an empty dashboard (no rows). Not an error state.
- **Interrupted/cancelled sessions**: When a user interrupts Claude mid-response (e.g., Ctrl+C), no further hook events are sent. The dashboard retains the last known state until the next poll cycle detects the PID is dead and removes the session. It MUST NOT crash or display stale state indefinitely.
- **Lock screen**: No special behavior. The dashboard is not visible on the lock screen.

## Clarifications

### Session 2026-03-22

- Q: Should terminated sessions disappear or show as stopped? → A: Disappear immediately (row removed when PID is dead).
- Q: How does the user edit settings? → A: Modal settings window accessible from tray right-click menu.
- Q: What is the default poll interval? → A: 5 seconds.
- Q: What should the row label show for CWD? → A: Path relative to `~` (e.g., `~/source/claude-dashboard`).
- Q: How is session-tracker metadata (cost, context %, rate limits) displayed? → A: Not displayed in MVP. Read internally for future use.

### Session 2026-03-22 (Live Testing)

- Q: Should Idle and AwaitingInput be distinct states? → A: Yes. Previously merged as a single AwaitingInput state. Live testing revealed they are semantically different: Idle means Claude finished responding (`end_turn`) and is waiting for the next user prompt (⏸️). AwaitingInput means Claude invoked the AskUserQuestion tool and is actively blocked waiting for a user answer (❓). The tray attention indicator fires on AwaitingInput (Claude is blocked) but NOT on Idle (Claude is just done).
- Q: Can the session ID in `sessions/{PID}.json` always be used to find the transcript? → A: No. When a session is resumed, the session ID may not match the transcript filename. A fallback to the most recently modified `.jsonl` in the project directory is required. See FR-025.
- Q: Which states trigger the tray attention indicator? → A: PermissionRequired and AwaitingInput. Not Idle.

## Requirements

### Functional Requirements

- **FR-001**: System MUST discover sessions by reading `~/.claude/sessions/*.json` (PID, CWD, session ID, start time)
- **FR-002**: System MUST display each session as a row in a vertical stack
- **FR-003**: Each row MUST show the session's workspace directory as a path relative to `~` (e.g., `~/source/claude-dashboard`)
- **FR-004**: Each row MUST show a status indicator (emoji + row color)
- **FR-005**: Status states MUST include: Working, Ready, Idle, Awaiting Input, Permission Required, Unknown. Dead sessions are removed (no Stopped state displayed).

  | State | Enum Value | Hook Event Trigger | Emoji | Description |
  |-------|-----------|-------------------|-------|-------------|
  | Working | `Working` | `UserPromptSubmit`, `PreToolUse` (non-AskUserQuestion), `PostToolUse` | 🔄 | Active processing |
  | Ready | `Ready` | `Stop` (intercepted by controller) | ⏸️ | Transient state after Working; auto-transitions to Idle after `ready_seconds` (default 300s). Cancelled by any new activity. Visually distinct color so user notices Claude finished. |
  | Idle | `Idle` | Ready timeout expires | ⏸️ | Finished responding, waiting for next user prompt |
  | Awaiting Input | `AwaitingInput` | `PreToolUse` with tool_name=`AskUserQuestion` | ❓ | Claude asked a question, actively waiting for user answer |
  | Permission Required | `PermissionRequired` | `PermissionRequest` | ⚠️ | Tool approval needed |
  | Unknown | `Unknown` | No hook event received yet | 🤷 | Fallback |
- **FR-006**: Row width MUST be fixed (configurable, not dynamic). Text exceeding width is truncated.
- **FR-007**: System MUST persist settings to a JSON file (row height, width, colors, emojis, window position, poll interval). Settings changes MUST apply live to the dashboard without hiding or repositioning it — the dashboard redraws in place with updated values.
- **FR-008**: System MUST restore window position on restart
- **FR-009**: System MUST support always-on-top mode (user-togglable)
- **FR-010**: System MUST work on Windows 11 and Linux (GNOME/Wayland). On Linux, window foregrounding requires the [Window Calls](https://extensions.gnome.org/extension/4724/window-calls/) GNOME Shell extension, which exposes window listing and activation via D-Bus — the only reliable method on Wayland where direct window manipulation is blocked by design.
- **FR-011**: System MUST use Python and Tkinter
- **FR-012**: Clicking a row MUST bring the containing application window to the foreground. Click vs drag MUST be distinguished: a short click (< 5px of movement) navigates to the session window; a click-hold+drag (>= 5px of movement) moves the dashboard window. This prevents accidental navigation when repositioning the dashboard.
- **FR-013**: System MUST handle parent process chain traversal (Claude → shell → VS Code / terminal)
- **FR-014**: System MUST handle virtual desktop switching when foregrounding. On Windows, `SetForegroundWindow` handles this natively. On Linux/GNOME Wayland, the `window-calls` extension's `Activate` method switches workspaces automatically.
- **FR-015**: System MUST detect session state via command hooks. Claude Code fires command hooks that execute `scripts/hook_relay.py`, which reads the hook payload from stdin and POSTs it to the dashboard's local HTTP server (`POST http://127.0.0.1:17384/hook`). The server maps hook event names to StatusState values using `map_event_to_state()` in `hook_server.py`. No transcript parsing is performed.
- **FR-016**: System MUST poll for session changes at a user-configurable interval (default: 5 seconds)
- **FR-017**: System MUST validate session PIDs are still alive (cross-reference with OS process list) to handle stale session files
- **FR-018**: [DEFERRED] Enriched metadata (model, cost, context window %, rate limits) is not consumed in MVP. If needed later, the dashboard will configure its own statusline hook rather than depending on external tooling.
- **FR-019**: [REMOVED — was session-tracker date rollover, no longer applicable]
- **FR-020**: System MUST NOT read files it does not need access to — in particular, `~/.claude/ide/*.lock` files contain auth tokens and must never be read
- **FR-021**: Each row MUST show the containing process type (VS Code, terminal, etc.) as an icon or label
- **FR-022**: System MUST minimize to system tray on close, with tray icon reflecting aggregate attention state. The Settings window MUST also be visible across all virtual desktops (topmost attribute), matching the dashboard's cross-desktop visibility behavior.
- **FR-023**: Rows MUST be ordered lexicographically by the displayed CWD text (the `~/...` relative path). This gives a stable, predictable order that groups related projects together.
- **FR-024**: System MUST handle the denial flow — when a tool use is rejected, subsequent hook events (PostToolUse or Stop) transition the state accordingly. No special transcript parsing is required.
- **FR-025**: System MUST resolve transcript paths with a fallback strategy — the session ID in `sessions/{PID}.json` does not always match the transcript filename (sessions can be resumed under a new session ID). When the expected transcript file is not found, the system MUST fall back to the most recently modified `.jsonl` file in the project directory (`~/.claude/projects/{project}/`).
- **FR-026**: System MUST run a local HTTP server (`hook_server.py`) on port 17384 to receive hook events relayed from `scripts/hook_relay.py`. The server MUST be non-blocking — if the dashboard is not running, the relay script's POST fails silently and Claude sessions are unaffected. The server accepts POST requests to `/hook` with JSON payloads containing `session_id`, `hook_event_name`, and optional fields (`tool_name`, `tool_input`, `cwd`).
- **FR-027**: System MUST map hook event names to StatusState values: `UserPromptSubmit`→Working, `PreToolUse`→Working (except `AskUserQuestion`→AwaitingInput), `PermissionRequest`→PermissionRequired, `PostToolUse`→Working, `Stop`→Idle, `SessionEnd`→remove session. Unknown events are ignored.
- **FR-028**: System MUST ship a `hooks-settings.json` file and provide `make install-hooks` (backed by `scripts/install_hooks.py`) to deep-merge hook configuration into `~/.claude/settings.json`. This configures Claude Code to fire command hooks that invoke `scripts/hook_relay.py`, which relays events to the dashboard's HTTP server.

### Key Entities

- **Session**: A running Claude Code process. Attributes: PID, CWD, status, parent window handle, container process type, session ID, slug (human-readable name), start time.
- **Settings**: User preferences. Attributes: row height, row width, status colors (per-state), status emojis (per-state), window position (x, y), always-on-top flag, grow-up flag, poll interval, ready duration (seconds), color picker position.
- **StatusState**: Enum of session states: Working, Ready, Idle, AwaitingInput, PermissionRequired, Unknown. (Dead sessions are removed, not displayed.) Ready is a transient state between Working and Idle — it auto-transitions to Idle after `ready_seconds` (default 5 min) unless new activity arrives. Idle and AwaitingInput are distinct — Idle means Claude finished and the ready timer expired; AwaitingInput means Claude asked a question via AskUserQuestion and is actively waiting for an answer.
- **ContainerType**: Enum of containing process types: VSCode, Terminal, GitBash, Screen, Unknown.

## Success Criteria

### Measurable Outcomes

- **SC-001**: All running Claude sessions appear in the dashboard within one poll cycle of starting
- **SC-002**: Clicking a session row foregrounds the correct window within 1 second on both Windows and Linux
- **SC-003**: Settings and window position survive application restart with no data loss
- **SC-004**: The dashboard uses < 1% CPU at idle
- **SC-005**: The dashboard updates status indicators within one hook event of a state change (near real-time, not poll-bound)
- **SC-006**: The user can identify which session needs attention without reading text (via color + emoji alone)
