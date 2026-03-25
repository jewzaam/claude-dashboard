# Feature Specification: Hide Sessions, Apply Button, Auto-Start

**Created**: 2026-03-23
**Status**: Implemented
**Version**: 0.2.0
**Feature**: 002-hide-apply-autostart
**Dependencies**: `specs/001-session-dashboard/spec.md` (US1-US5, FR-001–FR-028)

## User Stories

### User Story 6 — Hide Sessions (Priority: P2)

As a user with many active Claude sessions, I want to hide specific sessions from the dashboard via a right-click context menu, so I can focus on the sessions that matter without visual clutter.

**Why this priority**: Usability refinement. With many sessions, not all are relevant at all times. Hiding reduces noise without terminating sessions.

**Independent Test**: Right-click the dashboard, open Sessions submenu, uncheck a session. It disappears from the dashboard. Check it again — it reappears. Close and restart the hidden session — it appears visible in the dashboard.

**Acceptance Scenarios**:

1. **Given** the dashboard is showing 3 sessions, **When** I right-click and open the Sessions submenu, **Then** I see 3 checkbutton entries ordered identically to the dashboard rows, each showing the session's display name (CWD relative to `~`), all checked by default.
2. **Given** I uncheck a session in the Sessions submenu, **When** the menu closes, **Then** that session's row is immediately hidden from the dashboard.
3. **Given** a session is hidden, **When** I right-click and open the Sessions submenu, **Then** the hidden session's entry is unchecked. Checking it makes the session visible again.
4. **Given** all sessions are hidden (or no sessions exist), **When** I view the dashboard, **Then** it shows "No visible Claude sessions".
5. **Given** a session is hidden and then terminates, **When** a new session starts with the same CWD, **Then** the new session is visible (hidden state does not carry over).
6. **Given** a session is hidden, **When** the dashboard restarts, **Then** the session is visible (hidden state is not persisted across app restarts).
7. **Given** a session is hidden, **When** the tray icon priority is calculated, **Then** the hidden session is excluded — it does not influence the tray icon color.

---

### User Story 7 — Apply Button in Settings (Priority: P2)

As a user tweaking dashboard colors and dimensions, I want an Apply button in the settings dialog that applies and saves changes without closing the window, so I can iterate on visual settings quickly.

**Why this priority**: Quality-of-life improvement. The current Save-and-close workflow forces reopening the dialog for each tweak, which is tedious when adjusting colors.

**Independent Test**: Open settings, change a color, click Apply. The dashboard updates immediately. The settings window stays open. Change another color, click Apply again. Click Cancel to close without further changes.

**Acceptance Scenarios**:

1. **Given** the settings window is open, **When** I change a color and click Apply, **Then** the dashboard updates with the new color, the change is persisted to disk, and the settings window remains open.
2. **Given** I have clicked Apply, **When** I click Cancel, **Then** the settings window closes. The previously-applied changes remain in effect (they were already persisted).
3. **Given** the settings window is open, **When** I click Save, **Then** the changes are applied, persisted, and the window closes (same as current Save behavior).
4. **Given** the settings window is open, **When** I look at the button bar, **Then** I see three buttons in order: Apply, Save, Cancel.

---

### User Story 8 — Auto-Start on Boot (Priority: P2)

As a user, I want the dashboard to start automatically when I log in to my OS, so I don't have to manually launch it every time I reboot.

**Why this priority**: Convenience. A monitoring tool that requires manual startup defeats the purpose of always-on awareness.

**Independent Test**: Enable "Start on login" in settings. Reboot. The dashboard launches automatically on login.

**Acceptance Scenarios**:

1. **Given** I check "Start on login" in settings and save, **When** I log in to my OS after a reboot, **Then** the dashboard starts automatically.
2. **Given** I uncheck "Start on login" in settings and save, **When** I log in after a reboot, **Then** the dashboard does not start.
3. **Given** I am on Windows, **When** "Start on login" is enabled, **Then** the system writes a registry entry at `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\ClaudeDashboard` using the full path to `pythonw.exe` (no console window).
4. **Given** I am on Linux, **When** "Start on login" is enabled, **Then** the system writes a `.desktop` file to `~/.config/autostart/claude-dashboard.desktop` using the full path to the current Python interpreter (`sys.executable`).
5. **Given** I am on an unsupported platform, **When** I view the settings, **Then** the "Start on login" checkbox is still present but toggling it has no effect (no-op with a log warning).
6. **Given** the setting is enabled, **When** the dashboard starts, **Then** it syncs the OS auto-start mechanism to match the setting (ensures registry/desktop file exists even if manually deleted).

---

## Edge Cases

- **No visible Claude sessions**: Show "No visible Claude sessions" when no sessions exist or all are hidden. Not an error state.
- **Hidden session receives attention state**: A hidden session entering PermissionRequired or AwaitingInput does NOT surface in the tray icon. The user chose to hide it.
- **Auto-start with venv/pyenv**: The auto-start command uses `sys.executable` (absolute path) to ensure the correct Python interpreter is invoked, regardless of shell environment at login time.
- **Registry/desktop file manually deleted**: On each startup and settings save, the OS auto-start mechanism is re-synced. If the registry key or `.desktop` file was manually removed, it is recreated if the setting is enabled.

## Clarifications

### Session 2026-03-23

- Q: Should hidden state be keyed by PID? → A: No. Track on the session's internal entry object. When the entry is destroyed (session ends), the hidden state goes with it. A new session at the same CWD is visible.
- Q: Should hidden sessions affect the tray icon? → A: No. Hidden sessions are excluded from tray priority calculation.
- Q: Does Apply persist to disk? → A: Yes. Apply and Save both persist. The only difference is Save also closes the window.
- Q: Does Cancel revert previously-applied changes? → A: No. If Apply was clicked, those changes are already persisted. Cancel just closes the window.
- Q: Should Linux auto-start use `python3` or `sys.executable`? → A: `sys.executable` (absolute path) to handle venv/pyenv setups correctly.

## Requirements

### Functional Requirements

- **FR-029**: The right-click context menu MUST include a "Sessions" submenu listing all tracked sessions as checkbutton entries, ordered identically to the dashboard rows. Checked = visible (default), unchecked = hidden. Toggling a checkbutton immediately updates visibility. The submenu MUST be rebuilt each time the context menu is posted to reflect current session state.
- **FR-030**: Hidden session state MUST be stored on the session's internal tracking entry (not keyed by PID or CWD). When the session entry is removed (session terminates), the hidden state is destroyed. Hidden state is NOT persisted across app restarts.
- **FR-031**: Hidden sessions MUST be excluded from both the dashboard display and the tray icon priority calculation.
- **FR-032**: The settings dialog MUST have three buttons: Apply, Save, Cancel. Apply gathers form values, persists to disk, applies to dashboard, and keeps the window open. Save is Apply + close. Cancel closes without reverting previously-applied changes but still persists window/picker positions.
- **FR-033**: System MUST support auto-start on OS login. On Windows, use `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\ClaudeDashboard` registry key with `pythonw.exe -m claude_dashboard`. On Linux, use `~/.config/autostart/claude-dashboard.desktop` with `sys.executable -m claude_dashboard`. Both use the full path to the Python interpreter. Unsupported platforms are a no-op with a log warning.
- **FR-034**: The `run_on_startup` setting MUST be persisted in `settings.json` (default: `false`). The OS auto-start mechanism MUST be synced on app startup and on settings save.

### New/Modified Entities

- **Session** (modified): Gains `hidden: bool` attribute (default `False`). Not persisted.
- **Settings** (modified): Gains `run_on_startup: bool` attribute (default `False`). Persisted in `settings.json`.
