# Tasks: Agent Awareness

**Input**: Design documents from `/specs/003-agent-awareness/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US9 only for this feature)
- Include exact file paths in descriptions

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Extract shared constants needed by multiple classes

- [X] T001 Move `_STATE_PRIORITY` from `AppController` class body to module-level constant in `claude_dashboard/controller.py`
- [X] T002 Update `_highest_priority_state` to reference module-level `_STATE_PRIORITY` in `claude_dashboard/controller.py`

**Checkpoint**: Existing tests still pass with refactored constant location

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Hook server changes that all agent tracking depends on

**⚠️ CRITICAL**: Agent tracking cannot begin until hook server passes `agent_id` through

- [X] T003 [P] Write tests for `agent_id` extraction in `tests/test_hook_server.py` (test_agent_event_passes_agent_id, test_main_event_has_empty_agent_id)
- [X] T004 [P] Write test for `SubagentStop` callback in `tests/test_hook_server.py` (test_agent_stop_fires_callback)
- [X] T005 Update existing test callbacks to accept `agent_id=""`, `agent_type=""` kwargs and add `on_agent_stop` param in `tests/test_hook_server.py`
- [X] T006 Extract `agent_id` and `agent_type` from payloads in `claude_dashboard/hook_server.py` (`_HookHandler.do_POST`)
- [X] T007 Pass `agent_id` and `agent_type` through `on_hook_event` callback in `claude_dashboard/hook_server.py`
- [X] T008 Add `on_agent_stop` callback for `SubagentStop` events in `claude_dashboard/hook_server.py` (`HookServer.__init__`)
- [X] T009 Run `python -m pytest tests/test_hook_server.py -x -v` — all pass (existing + 3 new)

**Checkpoint**: Hook server passes agent fields through; SubagentStop fires dedicated callback

---

## Phase 3: User Story 9 — Agent Awareness (Priority: P1) 🎯 MVP

**Goal**: Dashboard reflects combined state of main process + all active agents

**Independent Test**: Launch a Claude session, spawn a background agent that needs permission. Dashboard shows PermissionRequired even though main process is idle.

### Implementation for User Story 9

- [X] T010 [US9] Add `_AgentEntry` class with `agent_id`, `state`, `agent_type` slots in `claude_dashboard/controller.py`
- [X] T011 [US9] Add `agents: dict[str, _AgentEntry]` to `_SessionEntry.__slots__` and `__init__` in `claude_dashboard/controller.py`
- [X] T012 [US9] Add `effective_state` property to `_SessionEntry` using module-level `_STATE_PRIORITY` in `claude_dashboard/controller.py`
- [X] T013 [US9] Wire `on_agent_stop` callback in `AppController.__init__` HookServer construction in `claude_dashboard/controller.py`
- [X] T014 [US9] Add `_on_hook_agent_stop` marshalling method and `_handle_agent_stop` handler in `claude_dashboard/controller.py`
- [X] T015 [US9] Update `_on_hook_event` signature to accept `agent_id=""`, `agent_type=""` in `claude_dashboard/controller.py`
- [X] T016 [US9] Update `_apply_hook_state` to register/update agents when `agent_id` present in `claude_dashboard/controller.py`
- [X] T017 [US9] Add `UserPromptSubmit` agent clearing logic (no `agent_id` = clear all agents) in `claude_dashboard/controller.py`
- [X] T018 [US9] Update `_refresh_ui` to use `effective_state` and pass `len(entry.agents)` in `claude_dashboard/controller.py`
- [X] T019 [US9] Update `_highest_priority_state` to accept 5-tuple with agent count in `claude_dashboard/controller.py`
- [X] T020 [US9] Add debounce to `_refresh_ui` with 300ms window and high-priority bypass (FR-043) in `claude_dashboard/controller.py`
- [X] T021 [US9] Update `update_sessions` signature to accept 5-tuple with agent count in `claude_dashboard/ui/main_window.py`
- [X] T022 [US9] Update `_add_row` to accept and display `(+N)` agent count in `claude_dashboard/ui/main_window.py`
- [X] T023 [US9] Update `_update_row` to accept and display `(+N)` agent count in `claude_dashboard/ui/main_window.py`
- [X] T024 [US9] Update all tuple unpacking in `update_sessions` body for 5-element tuples in `claude_dashboard/ui/main_window.py`
- [X] T025 [US9] Fix integration tests — update `_make_server` to return 4 values, update callers in `tests/test_integration.py`
- [X] T026 [US9] Fix `test_rapid_tool_calls_stay_working` monkey-patch signature in `tests/test_integration.py`
- [X] T027 [US9] Run `python -m pytest tests/ -x -q` — all pass (116 passed)

**Checkpoint**: Agent awareness fully functional — effective state, agent count, debounce, tray rollup

---

## Phase 4: Integration Tests — Agent Lifecycle

**Purpose**: Verify agent lifecycle scenarios end-to-end at the hook server level

- [X] T028 [US9] Add `test_agent_lifecycle` integration test (register → tool use → SubagentStop) in `tests/test_integration.py`
- [X] T029 [US9] Add `test_agent_permission_request` integration test (agent PermissionRequest with agent_id) in `tests/test_integration.py`
- [X] T030 Run `python -m pytest tests/ -x -q` — all pass (116 passed)

**Checkpoint**: Agent lifecycle integration tests verify end-to-end hook → state flow

---

## Phase 5: Final Validation

**Purpose**: Full quality gate before merge

- [X] T031 Run `python -m pytest tests/ --cov=claude_dashboard --cov-report=term-missing -q` — coverage 89% (>= 80%)
- [X] T032 Run `make lint` and `make typecheck` — both pass (format-check skipped: black DLL issue)
- [X] T033 No issues found
- [ ] T034 Manual verification: spawn background agent, verify `(+1)` and state Working
- [ ] T035 Manual verification: agent completes, verify `(+1)` disappears, state transitions to Ready
- [ ] T036 Manual verification: spawn 2 agents, verify `(+2)` then `(+1)` as each completes
- [ ] T037 Manual verification: send new prompt with agents tracked, verify agents cleared

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — refactoring only
- **Foundational (Phase 2)**: Depends on Phase 1 (module-level constant needed for tests)
- **User Story 9 (Phase 3)**: Depends on Phase 2 (hook server must pass agent_id)
- **Integration Tests (Phase 4)**: Depends on Phase 3 (agent tracking must be implemented)
- **Final Validation (Phase 5)**: Depends on all previous phases

### Within Phase 3

- T010-T012: Agent data model (sequential — each builds on previous)
- T013-T017: Controller wiring (sequential — callback → handler → state logic)
- T018-T020: UI data flow (sequential — refresh → priority → debounce)
- T021-T024: UI display (can parallel with T018-T020 if interfaces agreed)
- T025-T026: Test fixes (after all production code)

### Parallel Opportunities

- T003 and T004: Independent test files, different test methods
- T021-T024 can begin once T018-T019 define the 5-tuple interface

---

## Implementation Strategy

### MVP First (User Story 9 Only)

1. Complete Phase 1: Setup (refactor constant)
2. Complete Phase 2: Foundational (hook server agent_id extraction)
3. Complete Phase 3: User Story 9 (agent tracking + UI)
4. **STOP and VALIDATE**: Run test suite, verify agent awareness works
5. Complete Phase 4: Integration tests
6. Complete Phase 5: Final validation + manual testing
