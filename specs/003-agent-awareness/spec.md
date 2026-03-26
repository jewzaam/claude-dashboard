# Feature Specification: Agent Awareness

**Created**: 2026-03-24
**Status**: Spec'd (implementation pending)
**Version**: 0.4.0
**Feature**: 003-agent-awareness
**Dependencies**: `specs/001-session-dashboard/spec.md` (FR-015, FR-026, FR-027 — hook server and event mapping)

## User Stories

### User Story 9 — Agent Awareness (Priority: P1)

As a user running Claude Code sessions that spawn agents (foreground or background), I want the dashboard to reflect the combined state of the main process and all active agents, so I can see when agents need my attention even if the main process appears idle.

**Why this priority**: Without agent awareness, the dashboard shows Idle while agents are actively working, requesting permissions, or waiting for input. This is the primary state inaccuracy — the dashboard lies about what's happening. This was observed live (2026-03-24) and is the motivation for the research documented in `research.md`.

**Independent Test**: Launch a Claude session, spawn a background agent that needs permission. The dashboard should show PermissionRequired even though the main process is idle.

**Acceptance Scenarios**:

1. **Given** a session spawns a background agent, **When** the main process goes idle but the agent is working, **Then** the session row shows Working (not Idle).
2. **Given** a session has an active agent that needs permission, **When** I view the dashboard, **Then** the session row shows PermissionRequired regardless of the main process state.
3. **Given** a session's last active agent completes (`SubagentStop`), **When** no other agents remain, **Then** the session state reverts to the main process's own state. If other agents remain active, the effective state reflects their states.
4. **Given** a session has active agents, **When** I view the session row, **Then** I see a count indicator showing the number of active agents (e.g., "(+2)") to the right of the CWD display name.
5. **Given** agents are tracked from a previous turn, **When** each agent completes, **Then** it is removed individually via `SubagentStop`. Agents are NOT cleared on `UserPromptSubmit` because auto-wake events are indistinguishable from real user prompts. Orphaned agents (no `SubagentStop`) are cleared on PID death.
6. **Given** a hook event arrives with an `agent_id` not yet tracked, **When** the event is NOT `SubagentStop`, **Then** the dashboard registers the agent. If `SubagentStop` is the first and only event for an `agent_id`, no agent is registered.
7. **Given** a session has active agents, **When** the tray icon priority is calculated, **Then** agent states are included in the rollup (an agent needing permission makes the tray orange).

---

## Edge Cases

- **`SubagentStart` unreliability**: This hook sometimes does not fire for background agents. Agent registration must not depend on it. Register on the first hook event carrying an `agent_id` that is not `SubagentStop`.
- **Orphaned agents (interrupt)**: No `SubagentStop` fires for foreground agents on interrupt, or for background agents explicitly stopped by the user. Orphaned agents are cleared on the next `UserPromptSubmit` (new user turn wipes all agents). Also cleared when the parent session's PID dies.
- **Out-of-order agent completion**: Agents can complete in any order regardless of start order. Each completion triggers an auto-wake cycle (`UserPromptSubmit` → `Stop`) on the main session.
- **Auto-wake after agent completion**: After each `SubagentStop`, Claude Code auto-fires `UserPromptSubmit` → `Stop` on the main session. With N agents, expect N cycles. These are not user-initiated.
- **Permission denied on agent (no feedback)**: `SubagentStop` fires — agent is cleaned up. No stuck state (unlike main process denial).
- **Agent denied, then `SubagentStop` is first event**: If the only event for an `agent_id` is `SubagentStop`, do not register the agent — it's already done.

## Clarifications

### Session 2026-03-24 (Spec Clarify — Pass 2)

- Q: Should high-priority state changes (PermissionRequired, AwaitingInput) bypass the debounce? → A: Yes. These display immediately; debounce only suppresses auto-wake noise.
- Q: How should nested agents (agents spawned by other agents) be treated? → A: Flat peers. No hierarchy tracked. Count reflects total active agents regardless of spawn origin.

### Session 2026-03-24 (Spec Clarify — Pass 1)

- Q: How should the dashboard handle auto-wake transitions (rapid UserPromptSubmit → Stop cycles) after agent completion? → A: Debounce state display updates (200-500ms window) to absorb rapid transitions and prevent flickering.
- Q: What format should the active agent count indicator use? → A: Parenthesized format `(+N)` to the right of the CWD display name.
- Q: What is the canonical term — "subagent" or "agent"? → A: "agent" everywhere. Feature title updated from "Subagent Awareness" to "Agent Awareness".

### Session 2026-03-24 (Agent Hook Research)

- Q: Do standard hooks fired by agents carry `agent_id`? → A: Yes. All hooks (PreToolUse, PostToolUse, PermissionRequest) from agents include `agent_id` and `agent_type`. See `research.md`.
- Q: Is `SubagentStart` reliable? → A: No. It sometimes does not fire for background agents. Register agents on first `agent_id`-carrying event (not `SubagentStop`).
- Q: Does the main session `Stop` fire before background agents complete? → A: Yes. Main goes idle while agents continue working.
- Q: What happens on interrupt with active agents? → A: Foreground agents are orphaned (no `SubagentStop`). Background agents can be individually stopped by the user but also no `SubagentStop` observed. Clean up via PID death.
- Q: Does permission denial on an agent fire `SubagentStop`? → A: Yes, agent stops cleanly. Unlike main process denial which fires nothing.
- Q: Does Claude auto-fire events after `SubagentStop`? → A: Yes. Each `SubagentStop` triggers `UserPromptSubmit` → `Stop` on the main session (auto-wake to process agent result).
- Q: How should the displayed state work with agents? → A: Highest priority across main + all agents. See `docs/state-transitions.md` for the effective state rollup.

## Requirements

### Functional Requirements

- **FR-035**: The hook server MUST extract `agent_id` from hook payloads and pass it through to the controller callback. Events without `agent_id` are main-session events.
- **FR-036**: The controller MUST track active agents per session. An agent is registered on the first hook event carrying an `agent_id` that is NOT `SubagentStop`. An agent is removed on `SubagentStop`. If `SubagentStop` is the first and only event for an `agent_id`, the agent is not registered.
- **FR-037**: The displayed state for a session row MUST be the highest-priority state across the main process and all active agents, using the priority order: PermissionRequired > AwaitingInput > Working > Ready > Idle > Unknown. See `docs/state-transitions.md` for the effective state rollup.
- **FR-038**: When the main session fires `Stop` but active agents remain, the effective state MUST reflect the agents' states (e.g., Working), not the main session's Idle/Ready.
- **FR-039**: The session row MUST display an active agent count indicator in the format `(+N)` to the right of the CWD display name when one or more agents are tracked. The indicator disappears when no agents are active.
- **FR-040**: The tray icon priority calculation MUST include agent states in the rollup. A session with an idle main process but a permission-blocked agent contributes PermissionRequired to the tray priority.
- **FR-041**: Agents MUST be cleared when the parent session's PID dies. Agents MUST NOT be cleared on `UserPromptSubmit` because auto-wake events (fired after each `SubagentStop`) are indistinguishable from real user prompts and would prematurely clear active agents. Agents are removed individually via `SubagentStop`.
- **FR-042**: The hook server MUST extract `agent_type` from hook payloads and pass it to the controller. Observed value: `"general-purpose"`. Stored on the agent entry for future use.
- **FR-043**: The controller MUST debounce state display updates (200-500ms window) to absorb rapid auto-wake transitions (`UserPromptSubmit` → `Stop` cycles) that fire after each `SubagentStop`. This prevents visual flickering during agent completion sequences. High-priority states (PermissionRequired, AwaitingInput) MUST bypass the debounce and display immediately.
- **FR-044**: Agents spawned by other agents (nested agents) MUST be tracked as flat peers in the session's agent dict. No parent-child hierarchy is tracked. The `(+N)` count reflects total active agents regardless of spawn origin.

### New/Modified Entities

- **Session** (modified): Gains `agents: dict[str, Agent]` — map of active agents keyed by `agent_id`.
- **Agent** (new): An agent within a session. Attributes: agent_id (hex string), state (StatusState), agent_type (string, e.g. "general-purpose"). Lifecycle: registered on first `agent_id`-carrying hook event (not `SubagentStop`), removed on `SubagentStop` or parent PID death.
- **EffectiveState** (new): The displayed state for a session — the highest-priority state across the main process and all active agents. See `docs/state-transitions.md`.
- **Settings** (modified): No new settings fields for this feature.

## Success Criteria

### Measurable Outcomes

- **SC-007**: When a background agent needs permission while the main session is idle, the dashboard shows PermissionRequired within one hook event (not delayed until next poll)
