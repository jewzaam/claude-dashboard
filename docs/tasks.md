# Tasks: Claude Dashboard

> **Note (2026-03-23)**: This file reflects the original build plan. The hook migration
> (equivalent to Phases 6-8 in scope) happened organically during live testing and is not
> captured as discrete tasks here. See `docs/research-session-detection.md` for the
> transcript-to-hooks evolution and `docs/plan.md` for the current architecture.

- **Input**: `docs/plan.md`, `docs/spec.md`, `docs/research-session-detection.md`
- **Prerequisites**: Plan complete, spec clarified, session detection research verified

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1–US5)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and build tooling

- [ ] T001 Create project directory structure per plan (`claude_dashboard/`, `tests/`, `scripts/`)
- [ ] T002 Create `pyproject.toml` with dependencies (psutil, pystray, Pillow, dev deps)
- [ ] T003 [P] Create `Makefile` with standard targets (check, install-dev, format, lint, typecheck, test, coverage)
- [ ] T004 [P] Create `.gitignore` (venv, __pycache__, .mypy_cache, *.egg-info, settings.json)
- [ ] T005 [P] Create `.github/workflows/` CI configs (test, lint, typecheck, format-check, coverage)
- [ ] T006 Create `claude_dashboard/__init__.py` and `claude_dashboard/__main__.py` (entry point with `--debug`, `--quiet` flags)

**Checkpoint**: `make install-dev && make check` passes (empty test suite)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core data types, settings, and config that all stories depend on

- [ ] T007 Create `claude_dashboard/config.py` — constants, paths (`CLAUDE_HOME`, `SESSIONS_DIR`), defaults
- [ ] T008 Create `claude_dashboard/settings.py` — Settings dataclass, `load_settings()`, `save_settings()` with atomic JSON I/O and validation
- [ ] T009 [P] Create `tests/test_settings.py` — load/save roundtrip, defaults, missing file, invalid data, atomic write
- [ ] T010 [P] Create `tests/fixtures/sample_session.json` — sample session registry entry
- [ ] T011 [P] Create `tests/fixtures/sample_transcript.jsonl` — transcript entries covering all state machine transitions (working, idle, permission pending, denied, unknown)
- [ ] T012 [P] Create `tests/conftest.py` — shared fixtures (tmp_path-based settings dir, sample data loaders)

**Checkpoint**: `make check` passes. Settings can be loaded, saved, and validated.

---

## Phase 3: User Story 1 — Session Awareness Dashboard (Priority: P1) 🎯 MVP

**Goal**: Discover running sessions, detect their state, display as rows in a dashboard

**Independent Test**: Launch 3+ Claude sessions. Dashboard shows all with correct CWD and status.

### Tests for US1

- [ ] T013 [P] [US1] Create `tests/test_session.py` — discover sessions from fixture dir, validate PID check, handle empty dir, handle malformed JSON
- [ ] T014 [P] [US1] Create `tests/test_transcript.py` — state machine tests: working, awaiting input, permission pending, denied with text, denied without text, unknown, empty file, malformed JSONL

### Implementation for US1

- [ ] T015 [US1] Create `claude_dashboard/session.py`:
  - `discover_sessions()` — read `~/.claude/sessions/*.json`, return list of SessionInfo
  - `validate_pid(pid)` — check PID is alive and is `claude.exe`/`claude`
  - `resolve_transcript_path(cwd, session_id)` — construct path from CWD encoding
- [ ] T016 [US1] Create `claude_dashboard/transcript.py`:
  - `detect_state(transcript_path)` — tail last N lines, apply state machine, return StatusState
  - `tail_jsonl(path, *, max_lines=10)` — efficient tail read (seek from end)
  - StatusState enum: Working, AwaitingInput, PermissionRequired, Unknown
- [ ] T017 [US1] Create `claude_dashboard/ui/main_window.py`:
  - Dashboard window with dynamic row grid
  - Each row: CWD (relative to ~), status emoji, status color
  - Fixed row width, text truncation
  - Rows ordered by PID ascending
  - Rows added/removed/updated on each tick
- [ ] T018 [US1] Create `claude_dashboard/controller.py`:
  - AppController: owns root Tk, poll loop via `root.after()`
  - `_tick()` — discover sessions, validate PIDs, detect states, update UI
  - Remove rows for dead PIDs
- [ ] T019 [US1] Wire `__main__.py` to AppController, verify end-to-end with live sessions

**Checkpoint**: Dashboard shows live sessions with correct CWD and status indicators. Rows appear/disappear as sessions start/stop.

---

## Phase 4: User Story 2 — Session Navigation (Priority: P1) 🎯 MVP

**Goal**: Click a session row to foreground the containing application window

**Independent Test**: Click a row → correct VS Code window comes to foreground

### Implementation for US2

- [ ] T020 [US2] Create `claude_dashboard/platform/base.py` — abstract interface: `detect_container(pid)`, `foreground_window(handle)`
- [ ] T021 [US2] Create `claude_dashboard/platform/windows.py`:
  - `detect_container(pid)` — walk parent process chain, identify container type
  - `find_window(cwd, container)` — enumerate main VS Code process windows, match by CWD folder name in title
  - `foreground_window(hwnd)` — `SetForegroundWindow` with Alt-key fallback
  - Port logic from `scripts/detect_sessions.py`
- [x] T022 [P] [US2] Create `claude_dashboard/platform/linux.py` — window-calls GNOME extension via D-Bus, `code /path` fallback (tested on Fedora 42 / GNOME 48.7 / Wayland, 2026-03-23)
- [ ] T023 [US2] Wire click handler in `ui/main_window.py` — row click calls `platform.foreground_window()`
- [ ] T024 [US2] Cache container detection results — only re-detect on new sessions, not every tick

**Checkpoint**: Clicking a row foregrounds the correct VS Code window. Works across virtual desktops.

---

## Phase 5: User Story 3 — Persistent Settings & Window Position (Priority: P1) 🎯 MVP

**Goal**: Settings and window position persist across restarts

**Independent Test**: Configure settings, reposition window, restart. Everything restores.

### Implementation for US3

- [ ] T025 [US3] Wire settings into controller — load on startup, save on exit
- [ ] T026 [US3] Wire window position persistence — save position on close/hide, restore on start
- [ ] T027 [US3] Wire configurable values into UI — row height, width, colors, emojis, poll interval from Settings
- [ ] T028 [US3] Verify settings survive restart with live test

**Checkpoint**: Full P1 MVP complete. Dashboard shows sessions, click navigates, settings persist.

---

## Phase 6: User Story 4 — Container Process Indicator (Priority: P2)

**Goal**: Each row shows an icon or label indicating the containing process

**Independent Test**: Sessions in VS Code show VS Code icon; sessions in terminal show terminal icon

### Implementation for US4

- [ ] T029 [US4] Add ContainerType field to session row display — text label (e.g., "VS Code", "Terminal")
- [ ] T030 [US4] Source process icon from running application (optional — text label is acceptable MVP)

**Checkpoint**: Each row shows what application contains the session.

---

## Phase 7: User Story 5 — System Tray Persistence (Priority: P2)

**Goal**: Minimize to tray on close, tray icon indicates attention state

**Independent Test**: Close window → appears in tray. Session needs permission → tray icon changes.

### Implementation for US5

- [ ] T031 [US5] Create `claude_dashboard/tray.py` — pystray icon, context menu (Show, Settings, Quit)
- [ ] T032 [US5] Wire close button to minimize-to-tray instead of exit
- [ ] T033 [US5] Implement tray icon attention indicator — change icon/color when any session is in PermissionRequired state
- [ ] T034 [US5] Create `claude_dashboard/ui/settings_window.py` — modal dialog for all configurable values
- [ ] T035 [US5] Wire Settings menu item to open settings_window

**Checkpoint**: Full P2 complete. Tray persistence, attention indicator, settings editor.

---

## Phase 8: Polish & Cross-Cutting

**Purpose**: Documentation, CI, quality

- [ ] T036 [P] Create `TEST_PLAN.md` per standards
- [ ] T037 [P] Create `README.md` per standards (badges, install, usage)
- [ ] T038 Run `make check` — ensure all targets pass (format, lint, typecheck, test, coverage ≥ 80%)
- [ ] T039 Verify GitHub workflows pass
- [ ] T040 End-to-end validation: launch dashboard, start/stop sessions, click to navigate, close to tray, edit settings, restart

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies
- **Phase 2 (Foundation)**: Depends on Phase 1
- **Phase 3 (US1)**: Depends on Phase 2 — BLOCKS remaining stories
- **Phase 4 (US2)**: Depends on Phase 3 (needs session rows to click)
- **Phase 5 (US3)**: Depends on Phase 3 (needs UI to persist)
- **Phase 6 (US4)**: Depends on Phase 4 (needs container detection)
- **Phase 7 (US5)**: Depends on Phase 3 (needs controller for tray integration)
- **Phase 8 (Polish)**: Depends on desired stories being complete

### Parallel Opportunities

- T003, T004, T005 can run in parallel (Setup)
- T009, T010, T011, T012 can run in parallel (Fixtures/Tests)
- T013, T014 can run in parallel (US1 tests)
- T022 can run in parallel with T021 (platform stubs)
- Phase 5 (US3) can run in parallel with Phase 4 (US2) after Phase 3
- Phase 7 (US5) can start after Phase 3, parallel with Phase 6

### Implementation Strategy

1. **Phase 1 + 2**: Setup and foundation → `make check` passes
2. **Phase 3**: US1 (dashboard) → live session visibility
3. **Phase 4 + 5** (parallel): US2 (navigation) + US3 (settings) → full P1 MVP
4. **Phase 6 + 7** (parallel): US4 (container icons) + US5 (tray) → P2 complete
5. **Phase 8**: Polish → ship-ready
