# Test Plan: Claude Dashboard

## Strategy

Every functional requirement (FR-xxx) and success criterion (SC-xxx) from the spec is represented in this plan. Tests are either automated (unit tests) or manual (require a display server / live sessions). The plan defines what must be tested — implementation follows the plan.

## Coverage Target

80%+ line coverage of testable modules (`session.py`, `transcript.py`, `settings.py`). Currently at 93%.

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

State detection from transcript entries.

| Test | Type | File | Scenario | Expected State |
|------|------|------|----------|---------------|
| `test_awaiting_input` | Unit | `test_transcript.py` | Last assistant has `stop_reason=end_turn` | AwaitingInput |
| `test_permission_required` | Unit | `test_transcript.py` | Last assistant has `stop_reason=tool_use`, no tool result follows | PermissionRequired |
| `test_permission_required_with_hook_progress` | Unit | `test_transcript.py` | Tool use + hook progress entry, no tool result | PermissionRequired |
| `test_working_after_tool_approved` | Unit | `test_transcript.py` | Tool use + matching tool result | Working |
| `test_working_after_tool_denied_with_text` | Unit | `test_transcript.py` | Tool use + error tool result (denial) | Working |
| `test_working_on_user_prompt` | Unit | `test_transcript.py` | Assistant end_turn followed by user text prompt | Working |
| `test_unknown_for_none_path` | Unit | `test_transcript.py` | No transcript path | Unknown |
| `test_unknown_for_empty_file` | Unit | `test_transcript.py` | Empty transcript file | Unknown |
| `test_unknown_for_no_assistant_entries` | Unit | `test_transcript.py` | Only system entries | Unknown |
| `test_unknown_for_orphaned_tool_result` | Unit | `test_transcript.py` | User tool_result with no assistant entry | Unknown |

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
| Run on Windows 11 | Manual | Full end-to-end: discovery, state detection, navigation, tray |
| Run on Linux (X11) | Manual | Full end-to-end: discovery, state detection, navigation (xdotool) |

### FR-011: Python + Tkinter

Verified by implementation — no separate test needed.

### FR-012: Click-to-Foreground

| Test | Type | Scenario |
|------|------|----------|
| Click row → VS Code window foregrounds | Manual | Click a session row → the VS Code instance containing that session becomes the active window |
| Click row → terminal window foregrounds | Manual | Click a session running in a standalone terminal → that terminal foregrounds |

### FR-013: Process Chain Traversal

| Test | Type | Scenario |
|------|------|----------|
| Verify container detection for VS Code | Manual | Session in VS Code → container type shows "VS Code" |
| Verify container detection for terminal | Manual | Session in standalone terminal → container type shows "Term" |
| Verify process chain walks correctly | Manual | Run `scripts/detect_sessions.py` → chain shows claude → bash → Code.exe |

### FR-014: Virtual Desktop Switching

| Test | Type | Scenario |
|------|------|----------|
| Click row on different virtual desktop | Manual | Session's VS Code is on Desktop 2, dashboard on Desktop 1 → clicking switches to Desktop 2 and foregrounds |

### FR-015: Transcript-Based State Detection

| Test | Type | File | Scenario |
|------|------|------|----------|
| `test_finds_transcript` | Unit | `test_session.py` | CWD-encoded path matches transcript file |
| `test_returns_none_when_not_found` | Unit | `test_session.py` | No matching transcript → None |
| `test_fallback_scan` | Unit | `test_session.py` | Differently-encoded project key → found via scan |
| `test_reads_last_n_lines` | Unit | `test_transcript.py` | 20-line file, max_lines=5 → last 5 entries |
| `test_returns_empty_for_missing_file` | Unit | `test_transcript.py` | File does not exist → empty |
| `test_returns_empty_for_empty_file` | Unit | `test_transcript.py` | Zero-byte file → empty |
| `test_handles_malformed_lines` | Unit | `test_transcript.py` | Mix of valid and invalid JSON → only valid parsed |
| `test_reads_all_when_fewer_than_max` | Unit | `test_transcript.py` | 3-line file, max_lines=10 → all 3 |

### FR-016: Configurable Poll Interval

| Test | Type | Scenario |
|------|------|----------|
| Verify default 5-second poll | Manual | Launch with default settings → status updates every ~5 seconds |
| Verify custom poll interval | Manual | Set `poll_interval_seconds=2` → updates every ~2 seconds |

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
| `test_orders_by_pid` | Unit | `test_session.py` | Sessions with PIDs 300, 100, 200 → ordered 100, 200, 300 |

### FR-024: Denial Flow

| Test | Type | File | Scenario |
|------|------|------|----------|
| `test_working_after_tool_denied_with_text` | Unit | `test_transcript.py` | Tool result with `is_error=true` → Working |

## Success Criteria Traceability

| SC | Description | Validation |
|----|-------------|-----------|
| SC-001 | Sessions appear within one poll cycle | Manual: start session, observe dashboard |
| SC-002 | Foreground within 1 second | Manual: click row, time response |
| SC-003 | Settings survive restart | Unit: `test_roundtrip_preserves_values`; Manual: move window, restart |
| SC-004 | < 1% CPU at idle | Manual: monitor with Task Manager |
| SC-005 | Status updates within one poll cycle | Manual: trigger state change, observe dashboard |
| SC-006 | Status identifiable without reading text | Manual: verify emoji + color alone is sufficient |

## Requirement Interactions

| Interaction | How Tested |
|-------------|-----------|
| FR-001 + FR-017: Discovery filters dead PIDs | `validate_pid` tests cover the filter; `discover_sessions` tests verify parsing. Integration in `controller._tick()` (manual). |
| FR-001 + FR-015: Discovery feeds transcript resolution | `resolve_transcript_path` tested with CWD from session data; integration is in `controller._tick()` (manual). |
| FR-005 + FR-024: State machine handles denial as Working | `test_working_after_tool_denied_with_text` covers the denial-to-Working transition. |
| FR-007 + FR-008: Settings persist window position | `test_roundtrip_preserves_values` covers save/load including `window_x`/`window_y`. |
| FR-004 + FR-005: Emoji and color reflect state | Unit tests verify state detection; manual tests verify UI rendering. |
| FR-012 + FR-013 + FR-014: Click → process chain → foreground + desktop switch | Manual: click row with session on different virtual desktop. |
| FR-022 + FR-005: Tray reflects aggregate attention state | Manual: session enters PermissionRequired → tray icon changes. |

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
| `sample_transcript.jsonl` | Transcript with end_turn state | Available; tests use inline JSONL for precise control |
| `sample_settings.json` | Non-default settings values | `test_loads_valid_settings` via `sample_settings_data` |

## Running Tests

```bash
make test          # Run all tests
make coverage      # Run with coverage report (80% threshold enforced)
make check         # Full suite: format-check, lint, typecheck, test, coverage
```
