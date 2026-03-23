# Test Plan: Claude Dashboard

## Strategy

Every functional requirement (FR-xxx) and success criterion (SC-xxx) from the spec is represented in this plan. Tests are either automated (unit tests) or manual (require a display server / live sessions). The plan defines what must be tested — implementation follows the plan.

## Coverage Target

80%+ line coverage of testable modules (`session.py`, `transcript.py`, `hook_server.py`, `settings.py`, `platform/linux.py`). Currently at ~94%.

## Requirement Traceability

### FR-001: Session Discovery

Discover sessions by reading `~/.claude/sessions/*.json`.

| Test | Type | File | Scenario |
|------|------|------|----------|
| `test_discovers_session_from_fixture` | Unit | `test_session.py` | Valid session file parsed correctly (PID, session ID, CWD) |
| `test_returns_empty_for_missing_dir` | Unit | `test_session.py` | Sessions dir does not exist |
| `test_returns_empty_for_empty_dir` | Unit | `test_session.py` | Sessions dir exists but has no files |
| `test_skips_malformed_json` | Unit | `test_session.py` | Invalid JSON in session file |
| `test_skips_missing_fields` | Unit | `test_session.py` | Session file with falsy PID or session ID |

### FR-002: Row Display

Each session displayed as a row in a vertical stack.

| Test | Type | Scenario |
|------|------|----------|
| Verify rows appear for each discovered session | Manual | Launch app with 3+ sessions running → 3+ rows visible |
| Verify row is added when new session starts | Manual | Start a Claude session while dashboard is running → new row appears within one poll cycle |
| Verify row is removed when session terminates | Manual | Terminate a Claude session → row disappears within one poll cycle |

### FR-003: CWD Display (relative to ~)

| Test | Type | File | Scenario |
|------|------|------|----------|
| `test_strips_home_prefix` | Unit | `test_session.py` | Forward-slash CWD under home → `~/...` |
| `test_strips_home_prefix_backslash` | Unit | `test_session.py` | Backslash CWD under home → `~/...` |
| `test_preserves_non_home_paths` | Unit | `test_session.py` | CWD not under home → unchanged |
| Verify CWD displays as `~/...` in UI | Manual | Row shows `~/source/project` not full path |

### FR-004: Status Indicator

Each row shows emoji + row color.

| Test | Type | Scenario |
|------|------|----------|
| Verify emoji matches session state | Manual | Working session shows 🔄, idle shows ❓, permission pending shows ⚠️ |
| Verify row background color matches state | Manual | Each state has a distinct background color |

### FR-005: Status States

State detection from HTTP hook events.

| Test | Type | File | Scenario | Expected State |
|------|------|------|----------|---------------|
| `test_user_prompt_submit` | Unit | `test_transcript.py` | `UserPromptSubmit` event | Working |
| `test_pre_tool_use_regular` | Unit | `test_transcript.py` | `PreToolUse` with tool_name="Bash" | Working |
| `test_pre_tool_use_read` | Unit | `test_transcript.py` | `PreToolUse` with tool_name="Read" | Working |
| `test_pre_tool_use_ask_user_question` | Unit | `test_transcript.py` | `PreToolUse` with tool_name="AskUserQuestion" | AwaitingInput |
| `test_permission_request` | Unit | `test_transcript.py` | `PermissionRequest` event | PermissionRequired |
| `test_post_tool_use` | Unit | `test_transcript.py` | `PostToolUse` event | Working |
| `test_stop` | Unit | `test_transcript.py` | `Stop` event | Idle |
| `test_session_end_returns_none` | Unit | `test_transcript.py` | `SessionEnd` event | None (handled as removal) |
| `test_unknown_event_returns_none` | Unit | `test_transcript.py` | Unknown event name | None |
| `test_progress_returns_none` | Unit | `test_transcript.py` | `SubagentStart` event | None |

### FR-006: Fixed Row Width

Row width is fixed and configurable. Text exceeding width is truncated.

| Test | Type | Scenario |
|------|------|----------|
| Verify row width does not change with long CWD paths | Manual | Session with a very long CWD path → text truncated, row width unchanged |
| Verify row_width setting controls width | Manual | Change `row_width` in settings → dashboard width changes on restart |

### FR-007: Settings Persistence

| Test | Type | File | Scenario |
|------|------|------|----------|
| `test_returns_defaults_when_file_missing` | Unit | `test_settings.py` | No settings file → defaults |
| `test_loads_valid_settings` | Unit | `test_settings.py` | Valid JSON → correct field values |
| `test_ignores_unknown_fields` | Unit | `test_settings.py` | Extra fields in JSON → ignored |
| `test_returns_defaults_on_invalid_json` | Unit | `test_settings.py` | Malformed JSON → defaults |
| `test_returns_defaults_on_non_object_json` | Unit | `test_settings.py` | JSON array instead of object → defaults |
| `test_skips_wrong_type_values` | Unit | `test_settings.py` | String in int field → field uses default |
| `test_accepts_none_for_optional_fields` | Unit | `test_settings.py` | `null` for window_x/y → accepted |
| `test_save_creates_file` | Unit | `test_settings.py` | Save creates file with correct content |
| `test_save_creates_parent_dirs` | Unit | `test_settings.py` | Save creates intermediate directories |
| `test_roundtrip_preserves_values` | Unit | `test_settings.py` | Save then load → identical settings |
| `test_save_overwrites_existing` | Unit | `test_settings.py` | Second save overwrites first |

### FR-008: Window Position Restore

| Test | Type | Scenario |
|------|------|----------|
| Verify position persists | Manual | Move window to a specific position, close and reopen → same position |
| `test_roundtrip_preserves_values` | Unit | `test_settings.py` | `window_x`/`window_y` survive save/load cycle |

### FR-009: Always-on-Top

| Test | Type | Scenario |
|------|------|----------|
| Verify always-on-top behavior | Manual | With `always_on_top=True`, dashboard stays above other windows |
| Verify toggle works | Manual | Change setting to `False`, restart → dashboard can go behind other windows |

### FR-010: Cross-Platform (Windows 11 + Linux)

| Test | Type | Scenario |
|------|------|----------|
| Run on Windows 11 | Manual | Full end-to-end: discovery, hook-based state detection, navigation, tray |
| Run on Linux (GNOME/Wayland) | Manual | Full end-to-end: discovery, hook-based state detection, navigation (window-calls D-Bus) |

### FR-011: Python + Tkinter

Verified by implementation — no separate test needed.

### FR-012: Click-to-Foreground

| Test | Type | Scenario |
|------|------|----------|
| Click row → VS Code window foregrounds | Manual | Click a session row → the VS Code instance containing that session becomes the active window |
| Click row → terminal window foregrounds | Manual | Click a session running in a standalone terminal → that terminal foregrounds |

### FR-013: Process Chain Traversal

| Test | Type | File | Scenario |
|------|------|------|----------|
| `test_detects_vscode_container` | Unit | `test_linux.py` | Parent chain includes "code" → ContainerType.VSCODE |
| `test_detects_terminal_container` | Unit | `test_linux.py` | Parent chain includes terminal emulator → ContainerType.TERMINAL |
| `test_detects_screen_container` | Unit | `test_linux.py` | Parent chain includes "screen" → ContainerType.SCREEN |
| `test_detects_tmux_container` | Unit | `test_linux.py` | Parent chain includes "tmux" → ContainerType.TERMINAL |
| `test_fallback_to_term_program_env` | Unit | `test_linux.py` | No known parent but TERM_PROGRAM=vscode → ContainerType.VSCODE |
| `test_unknown_container` | Unit | `test_linux.py` | No recognizable parent → ContainerType.UNKNOWN |
| Verify container detection for VS Code | Manual | Session in VS Code → container type shows "VS Code" |
| Verify container detection for terminal | Manual | Session in standalone terminal → container type shows "Term" |
| Verify process chain walks correctly | Manual | Run `scripts/detect_sessions_linux.py` → chain shows claude → bash → code → systemd |

### FR-013a: Linux Window Matching (D-Bus)

| Test | Type | File | Scenario |
|------|------|------|----------|
| `test_list_windows_dbus_parses_json` | Unit | `test_linux.py` | Mock gdbus returning JSON window list → parsed correctly |
| `test_list_windows_dbus_empty_on_failure` | Unit | `test_linux.py` | gdbus returns error → empty list |
| `test_list_windows_dbus_gdbus_not_found` | Unit | `test_linux.py` | gdbus not installed → empty list |
| `test_activate_window_dbus_success` | Unit | `test_linux.py` | gdbus Activate returns 0 → True |
| `test_activate_window_dbus_failure` | Unit | `test_linux.py` | gdbus Activate returns nonzero → False |
| `test_find_window_id_matches_by_pid` | Unit | `test_linux.py` | Window list contains matching PID → returns window ID |
| `test_find_window_id_disambiguates_by_title` | Unit | `test_linux.py` | Multiple windows for same PID, CWD folder in title → correct window |
| `test_find_window_id_no_match` | Unit | `test_linux.py` | No windows match container PID → returns 0 |
| `test_foreground_window_linux_dbus` | Unit | `test_linux.py` | window-calls available → uses D-Bus activation |
| `test_foreground_window_linux_code_fallback` | Unit | `test_linux.py` | window-calls unavailable, VS Code container → falls back to `code /path` |
| `test_foreground_window_linux_no_method` | Unit | `test_linux.py` | No method available, non-VS Code → returns False |

### FR-014: Virtual Desktop Switching

| Test | Type | Scenario |
|------|------|----------|
| Click row on different virtual desktop | Manual | Session's VS Code is on Desktop 2, dashboard on Desktop 1 → clicking switches to Desktop 2 and foregrounds |

### FR-015: Hook-Based State Detection

State detection via HTTP hooks posted to the dashboard's local server.

| Test | Type | File | Scenario |
|------|------|------|----------|
| `test_receives_hook_event` | Unit | `test_hook_server.py` | POST a hook event → callback fires with correct session ID and state |
| `test_permission_request_event` | Unit | `test_hook_server.py` | PermissionRequest hook → PERMISSION_REQUIRED state |
| `test_session_end_fires_callback` | Unit | `test_hook_server.py` | SessionEnd hook → on_session_end callback fires |
| `test_empty_body_returns_400` | Unit | `test_hook_server.py` | POST with invalid body → 400 response |

### FR-016: Configurable Poll Interval

| Test | Type | Scenario |
|------|------|----------|
| Verify default 5-second poll | Manual | Launch with default settings → session discovery polls every ~5 seconds |
| Verify custom poll interval | Manual | Set `poll_interval_seconds=2` → discovery polls every ~2 seconds |

### FR-017: PID Validation

| Test | Type | File | Scenario |
|------|------|------|----------|
| `test_alive_claude_process` | Unit | `test_session.py` | PID alive, name contains "claude" → True |
| `test_alive_non_claude_process` | Unit | `test_session.py` | PID alive, name is "python.exe" → False |
| `test_dead_pid` | Unit | `test_session.py` | PID does not exist → False |
| `test_access_denied` | Unit | `test_session.py` | psutil raises AccessDenied → False |

### FR-018: [DEFERRED]

Not tested — enriched metadata is not consumed in MVP.

### FR-019: [REMOVED]

Not applicable.

### FR-020: No Unnecessary File Access

| Test | Type | Scenario |
|------|------|----------|
| Verify no reads of `~/.claude/ide/*.lock` | Code review | Grep codebase for `ide/` path access — must not exist |

### FR-021: Container Type Label

| Test | Type | Scenario |
|------|------|----------|
| Verify VS Code label | Manual | Session in VS Code → row shows "VS Code" label |
| Verify Terminal label | Manual | Session in terminal → row shows "Term" label |
| Verify Unknown label | Manual | Unrecognized container → no label (blank) |

### FR-022: System Tray

| Test | Type | Scenario |
|------|------|----------|
| Close window → tray icon appears | Manual | Close dashboard window → icon in system tray, app still running |
| Double-click tray → window reopens | Manual | Double-click tray icon → dashboard window reappears at saved position |
| Permission pending → tray icon changes | Manual | A session enters PermissionRequired state → tray icon changes color to orange |
| No attention needed → default icon | Manual | All sessions idle or working → tray icon is default blue |
| Tray menu: Show, Settings, Quit | Manual | Right-click tray → menu with three items, each works |

### FR-023: Row Ordering

| Test | Type | File | Scenario |
|------|------|------|----------|
| `test_orders_by_cwd_lexicographically` | Unit | `test_session.py` | Sessions with CWDs zebra, alpha, middle → ordered alpha, middle, zebra |

### FR-024: Denial Flow

Denial flow is implicit in hook-based detection — when a user denies a tool, Claude continues processing and the next hook event (PostToolUse or Stop) updates the state accordingly. No special transcript parsing is required.

### FR-026: Hook Server

| Test | Type | File | Scenario |
|------|------|------|----------|
| `test_receives_hook_event` | Unit | `test_hook_server.py` | POST to /hook with valid JSON → 200 response, callback fires |
| `test_permission_request_event` | Unit | `test_hook_server.py` | PermissionRequest event → maps to PERMISSION_REQUIRED |
| `test_session_end_fires_callback` | Unit | `test_hook_server.py` | SessionEnd event → on_session_end callback fires |
| `test_empty_body_returns_400` | Unit | `test_hook_server.py` | Invalid POST body → 400 error |

### FR-027: Hook Event to State Mapping

| Test | Type | File | Scenario |
|------|------|------|----------|
| `test_user_prompt_submit` | Unit | `test_transcript.py` | UserPromptSubmit → Working |
| `test_pre_tool_use_regular` | Unit | `test_transcript.py` | PreToolUse (Bash) → Working |
| `test_pre_tool_use_ask_user_question` | Unit | `test_transcript.py` | PreToolUse (AskUserQuestion) → AwaitingInput |
| `test_permission_request` | Unit | `test_transcript.py` | PermissionRequest → PermissionRequired |
| `test_post_tool_use` | Unit | `test_transcript.py` | PostToolUse → Working |
| `test_stop` | Unit | `test_transcript.py` | Stop → Idle |
| `test_session_end_returns_none` | Unit | `test_transcript.py` | SessionEnd → None (handled separately) |
| `test_unknown_event_returns_none` | Unit | `test_transcript.py` | Unknown event → None |
| `test_progress_returns_none` | Unit | `test_transcript.py` | SubagentStart → None |

## Success Criteria Traceability

| SC | Description | Validation |
|----|-------------|-----------|
| SC-001 | Sessions appear within one poll cycle | Manual: start session, observe dashboard |
| SC-002 | Foreground within 1 second | Manual: click row, time response |
| SC-003 | Settings survive restart | Unit: `test_roundtrip_preserves_values`; Manual: move window, restart |
| SC-004 | < 1% CPU at idle | Manual: monitor with Task Manager |
| SC-005 | Status updates within one hook event | Manual: trigger state change, observe dashboard updates immediately |
| SC-006 | Status identifiable without reading text | Manual: verify emoji + color alone is sufficient |

## Requirement Interactions

| Interaction | How Tested |
|-------------|-----------|
| FR-001 + FR-017: Discovery filters dead PIDs | `validate_pid` tests cover the filter; `discover_sessions` tests verify parsing. Integration in `controller._tick()` (manual). |
| FR-001 + FR-015: Discovery feeds hook server session tracking | Session discovery provides session metadata; hook server receives real-time state updates. Integration is in `controller._tick()` (manual). |
| FR-005 + FR-024: Denial flow handled by subsequent hook events | When a tool is denied, the next hook event (PostToolUse or Stop) updates state. No special-case logic needed. |
| FR-007 + FR-008: Settings persist window position | `test_roundtrip_preserves_values` covers save/load including `window_x`/`window_y`. |
| FR-004 + FR-005: Emoji and color reflect state | Unit tests verify event-to-state mapping; manual tests verify UI rendering. |
| FR-012 + FR-013 + FR-014: Click → process chain → foreground + desktop switch | Manual: click row with session on different virtual desktop. |
| FR-022 + FR-005: Tray reflects aggregate attention state | Manual: session enters PermissionRequired → tray icon changes. |
| FR-015 + FR-026: Hook server drives state detection | Hook events POST to the local HTTP server, which maps them to StatusState and updates session state. |

## Project Key Encoding (Internal)

| Test | Type | File | Scenario |
|------|------|------|----------|
| `test_windows_path` | Unit | `test_session.py` | `C:\Users\user\source\x` → `C--Users-user-source-x` |
| `test_linux_path` | Unit | `test_session.py` | `/home/user/source/x` → `home-user-source-x` |
| `test_forward_slash_path` | Unit | `test_session.py` | `C:/Users/user/source/x` → `C--Users-user-source-x` |

## Fixtures

| File | Purpose | Used By |
|------|---------|---------|
| `sample_session.json` | Valid session registry entry | `test_discovers_session_from_fixture` via `tmp_sessions_dir` |
| `sample_settings.json` | Non-default settings values | `test_loads_valid_settings` via `sample_settings_data` |

## Running Tests

```bash
make test          # Run all tests
make coverage      # Run with coverage report (80% threshold enforced)
make check         # Full suite: format-check, lint, typecheck, test, coverage
```
