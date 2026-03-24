# Feature Specification: State Cache — Persist Session State Across Dashboard Restarts

**Feature Branch**: `001-state-cache`
**Created**: 2026-03-24
**Status**: Draft
**Input**: User description: "Persist session state to disk so the dashboard shows last known state on restart instead of 'unknown'. Hooks are always installed, so state information flows continuously — currently lost when the dashboard is not running. The hook relay should write state to a cache file even when the dashboard is down, and the dashboard should read cached state on startup to restore sessions to their last known state."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Dashboard Restart Shows Last Known State (Priority: P1)

As a user running multiple Claude Code sessions, when I restart the dashboard (e.g., after a crash, system reboot, or manual restart), I want each session row to display its last known state (working, idle, ready, permission required, awaiting input) instead of "unknown", so I can immediately see which sessions need attention without waiting for the next hook event.

**Why this priority**: This is the core problem. The "unknown" state after restart provides no actionable information and forces the user to wait for activity before the dashboard becomes useful. Eliminating it is the entire purpose of this feature.

**Independent Test**: Can be fully tested by starting the dashboard with active Claude sessions, noting their states, stopping the dashboard, and restarting it — sessions should display their prior states immediately.

**Acceptance Scenarios**:

1. **Given** three Claude sessions are running with states Working, Idle, and Permission Required, **When** the dashboard is stopped and restarted, **Then** the sessions display Working, Ready (idle→ready interception), and Permission Required respectively.
2. **Given** a session was in the Working state when the dashboard last ran, **When** the dashboard restarts and no new hook events have arrived, **Then** the session displays Working (not Unknown).
3. **Given** the dashboard has never run before (no cached state exists), **When** the dashboard starts for the first time, **Then** sessions display Unknown (graceful fallback).

---

### User Story 2 - State Captured While Dashboard Is Down (Priority: P1)

As a user who sometimes restarts the dashboard while Claude sessions are active, I want state changes that occur while the dashboard is not running to be captured, so the dashboard shows accurate (not stale) state when it starts back up.

**Why this priority**: Without this, cached state would be stale — reflecting the moment the dashboard stopped rather than the current reality. Since hooks fire continuously regardless of dashboard status, capturing state during downtime is essential for accuracy.

**Independent Test**: Can be tested by stopping the dashboard, triggering Claude activity (sending a prompt, completing a task), then restarting the dashboard — the session should reflect the state from the activity that happened while the dashboard was down.

**Acceptance Scenarios**:

1. **Given** the dashboard is not running and a Claude session transitions from Working to Idle, **When** the dashboard starts, **Then** the session displays Ready (idle→ready interception applied).
2. **Given** the dashboard is not running and a session completes (SessionEnd event), **When** the dashboard starts, **Then** the ended session is not restored from cache.
3. **Given** the dashboard is not running and no hook events fire, **When** the dashboard starts, **Then** the session displays whatever state was last cached.

---

### User Story 3 - Stale Cache Entries Are Cleaned Up (Priority: P2)

As a user, I do not want old cached entries from sessions that ended long ago to pollute the dashboard or consume storage indefinitely, so the cache should automatically discard entries that are no longer relevant.

**Why this priority**: Without cleanup, the cache file grows unboundedly and may restore sessions that no longer exist. This is a hygiene concern rather than core functionality, but important for long-term usability.

**Independent Test**: Can be tested by caching state entries, waiting beyond the retention threshold, restarting the dashboard, and verifying old entries are not restored.

**Acceptance Scenarios**:

1. **Given** a cached state entry is older than 24 hours, **When** the dashboard reads the cache on startup, **Then** the stale entry is ignored.
2. **Given** a cached state entry is less than 24 hours old, **When** the dashboard reads the cache on startup, **Then** the entry is used to restore session state.

---

### Edge Cases

- What happens when the cache file is corrupted or contains invalid data? The dashboard should discard the corrupt cache and fall back to Unknown state.
- What happens when multiple Claude sessions write to the cache file concurrently? Writes should not corrupt the file; a lost update from a race condition is acceptable since the next hook event overwrites it.
- What happens when the cache contains an entry for a session ID that doesn't match any discovered session? The entry should be ignored (no phantom sessions created).
- What happens when a cached session ID doesn't match but the CWD does? The entry should be matched by CWD as a fallback (consistent with existing hook event matching for resumed sessions).
- What happens when the dashboard restarts and a session was Working in the cache but has since finished? The dashboard shows Working until the next hook event corrects it. This is better than Unknown because it communicates "this session was active."

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST persist the last known state for each session to a durable cache whenever a state-changing event occurs.
- **FR-002**: The system MUST capture state changes even when the dashboard is not running, since hook events fire regardless of dashboard status.
- **FR-003**: The dashboard MUST read the state cache on startup and restore each discovered session to its last cached state instead of defaulting to Unknown.
- **FR-004**: The dashboard MUST apply the same state interception rules to cached state as it does to live events (e.g., cached Idle becomes Ready).
- **FR-005**: The system MUST match cached entries to sessions by session ID, with CWD-based fallback matching for resumed sessions.
- **FR-006**: The system MUST remove cache entries when a SessionEnd event fires, preventing restoration of ended sessions.
- **FR-007**: The system MUST discard cache entries older than 24 hours on startup to prevent stale data accumulation.
- **FR-008**: The system MUST gracefully handle a missing, empty, or corrupt cache file by falling back to Unknown state for all sessions.
- **FR-009**: Cache writes MUST be atomic (write-then-rename pattern) to prevent file corruption from concurrent writes or crashes.
- **FR-010**: Cache operations MUST be non-blocking — a failure to read or write the cache must not prevent the dashboard or hook relay from functioning normally.

### Key Entities

- **State Cache Entry**: Represents the last known state of a session. Key attributes: session identifier, state value, working directory, timestamp of last update.
- **State Cache File**: A single persistent file containing all cached session state entries. Keyed by session identifier. Located in the dashboard's configuration directory.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: On dashboard restart with active sessions, 100% of sessions with cached state display their last known state instead of Unknown within 1 second of startup.
- **SC-002**: State changes that occur while the dashboard is down are reflected when the dashboard next starts, with no more than one missed transition per session (the final state is captured).
- **SC-003**: Cache entries older than 24 hours are never restored — 0% stale entry restoration rate.
- **SC-004**: A corrupt or missing cache file results in graceful fallback to Unknown state with no errors visible to the user.
- **SC-005**: Cache write operations complete in under 50 milliseconds and do not introduce perceptible delay to hook event processing.

## Assumptions

- Hooks are always installed and fire for every Claude Code event, regardless of whether the dashboard is running.
- The dashboard's configuration directory is writable by both the dashboard process and the hook relay process.
- Session identifiers are stable within a single session's lifetime (but may differ for resumed sessions, which is handled by CWD fallback matching).
- A single cache file is sufficient; the number of concurrent Claude sessions is small enough that file-level concurrency (not record-level) is adequate.
- The 24-hour retention threshold is sufficient for typical usage patterns (dashboard restarts within a workday).
