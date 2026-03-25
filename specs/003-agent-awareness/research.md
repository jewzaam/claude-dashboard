# Research: Subagent & Background Process Hook Payloads

**Date:** 2026-03-23
**Updated:** 2026-03-24
**Status:** Complete — all tests run, findings documented, architecture decision made

## Purpose

Determine exactly what data arrives in Claude Code hook payloads during subagent and background process scenarios. The dashboard currently ignores `SubagentStart`/`SubagentStop` hooks. Before implementing agent awareness, we need to know whether standard hooks (PreToolUse, PostToolUse, PermissionRequest, Stop, etc.) fired by subagents carry an `agent_id` field that distinguishes them from the main session.

## Architecture Decision

The answer to "do standard hooks carry `agent_id`?" determines the entire implementation:

- **Yes:** Track per-agent state keyed on `(session_id, agent_id)`. Full state rollup across main + agents.
- **No:** Track agent count via `SubagentStart`/`SubagentStop` brackets only. Suppress Idle/Ready when agents remain active. Coarser state inference.

## Instrumentation

`scripts/hook_relay.py` supports payload logging gated behind `--debug`. When `--debug` is passed, it logs raw JSON payloads to `~/.claude/claude-dashboard/logs/hook-payloads.jsonl` via the `logging` module. Without `--debug`, only the HTTP relay runs (no disk I/O overhead).

The `--marker TEXT` flag injects a labeled boundary line into the log (always writes, regardless of `--debug`).

To enable payload capture during testing, the hook command in `~/.claude/settings.json` must include `--debug`. Remember to remove it after testing.

### Using Markers

```bash
python scripts/hook_relay.py --marker "TEST 1: Foreground subagent - START"
# ... perform Claude Code operation ...
python scripts/hook_relay.py --marker "TEST 1: Foreground subagent - END"
```

## Test Protocol

Each test:
1. Inject start marker
2. Perform the described Claude Code operation in a separate terminal
3. Wait for completion
4. Inject end marker

### Test 1: Foreground subagent
**Goal:** Do a foreground subagent's standard hooks carry `agent_id`?
**Prompt (actual):** "Launch a subagent using the Agent tool with this prompt: 'Run `sleep 3` using the Bash tool, then report back.'"
**Look for:** `SubagentStart` fields, whether `PreToolUse`/`PostToolUse`/`Stop` from the agent carry `agent_id`.

### Test 2: Background subagent
**Goal:** Do background agents fire hooks? Event ordering vs main `Stop`.
**Prompt (actual):** "Launch a subagent in the background using the Agent tool with run_in_background set to true. The subagent should run `sleep 5` using Bash. Also tell me 'hello'."
**Look for:** Does main `Stop` fire while agent is working? Do agent hooks carry `agent_id`? Does `SubagentStop` fire?

### Test 3: Multiple concurrent background subagents
**Goal:** Distinct `agent_id` per agent.
**Prompt (actual):** "Launch 3 subagents in parallel using the Agent tool, all with run_in_background. Agent 1 runs `sleep 3`, Agent 2 runs `sleep 4`, Agent 3 runs `sleep 5`. Tell me 'dispatched' after launching them."
**Look for:** Multiple `SubagentStart` with different `agent_id`. Standard hooks carry respective `agent_id`.

### Test 4: Subagent requiring permission
**Goal:** Does `PermissionRequest` from a subagent carry `agent_id`?
**Prompt (actual):** "Launch a subagent using the Agent tool with this prompt: 'Read the file ~/.claude/settings.json'"
**Note:** First attempt used `ls -la` which was pre-allowed. Retried with `Read` on a sensitive path.
**Look for:** `PermissionRequest` event fields.

### Test 5: User interrupts while subagent running
**Goal:** What happens to agent hooks when the user interrupts (Ctrl+C/Escape) while a subagent is active?
**Prompt (actual):** "Launch a subagent using the Agent tool with this prompt: 'Run `sleep 30` using Bash then say done.'"
**Action:** Interrupted (Escape) while agent was running.
**Observation:** Foreground agent was orphaned — no way to access it after interrupt.
**Look for:** Does `SubagentStop` fire? Is the agent orphaned with no stop event?

### Test 7: Main idle while background subagent works
**Goal:** Confirm idle-while-agents-active gap and event ordering.
**Prompt (actual):** "Launch a subagent in the background using the Agent tool with run_in_background. The subagent should run `sleep 10` using Bash then say done. After launching it, just tell me 'dispatched' and stop."
**Look for:** Main `Stop` fires, then agent hooks continue, then `SubagentStop`.

### Test 8: Permission denied, no feedback (main process)
**Goal:** What hooks fire when the user denies a tool without typing feedback?
**Prompt (actual):** "Read the file ~/.claude/settings.json" (main process, no agent)
**Action:** Denied permission without feedback text.
**Look for:** Does any hook fire on denial?

### Test 8b: Permission denied, no feedback (agent)
**Goal:** Same as Test 8 but through a subagent.
**Prompt (actual):** "Launch a subagent using the Agent tool with this prompt: 'Read the file ~/.claude/settings.json'"
**Action:** Denied permission without feedback text.
**Look for:** Does the agent get a `SubagentStop`? Does the main session continue?

### Test 8c: Permission denied, with feedback (agent)
**Goal:** Does providing denial feedback text change hook behavior?
**Prompt (actual):** Same as 8b.
**Action:** Denied permission with feedback text "just say hi then stop".
**Look for:** Any difference from 8b.

---

## Findings

**The critical question is answered: YES, standard hooks from subagents carry `agent_id`.** This means we can track per-agent state.

### Key Fields Discovered

| Field | Present on | Values |
|-------|-----------|--------|
| `agent_id` | All hooks fired by a subagent (PreToolUse, PostToolUse, PermissionRequest, SubagentStart, SubagentStop) | Hex string, unique per agent (e.g. `a41d1801a7ca`) |
| `agent_type` | All hooks with `agent_id` | `"general-purpose"` (only value observed) |
| `agent_transcript_path` | `SubagentStop` only | Full path to agent transcript |
| `permission_suggestions` | `PermissionRequest` events | Present on both main and agent permission requests |
| `prompt` | `UserPromptSubmit` events | The user's prompt text |

### Test 1: Foreground subagent

Event sequence:
```
UserPromptSubmit          (main)
PreToolUse  tool=Agent    (main — about to spawn agent)
SubagentStart             agent_id=a41d1801a7ca
PreToolUse  tool=Bash     agent_id=a41d1801a7ca  <- standard hook carries agent_id
PostToolUse tool=Bash     agent_id=a41d1801a7ca
SubagentStop              agent_id=a41d1801a7ca
PostToolUse tool=Agent    (main — agent returned)
Stop                      (main)
```

**Conclusion:** Foreground subagent hooks carry `agent_id`. The main session wraps the agent lifecycle with `PreToolUse tool=Agent` / `PostToolUse tool=Agent`. `SubagentStart` fires after `PreToolUse tool=Agent`, `SubagentStop` fires before `PostToolUse tool=Agent`. No `Stop` event fires for the agent — only `SubagentStop`.

### Test 2: Background subagent

Event sequence:
```
UserPromptSubmit          (main)
PreToolUse  tool=Agent    (main)
PostToolUse tool=Agent    (main — returns immediately for background)
Stop                      (main — goes idle while agent works)
PreToolUse  tool=Bash     agent_id=ae02be8a2a4d  <- agent fires AFTER main Stop
PostToolUse tool=Bash     agent_id=ae02be8a2a4d
SubagentStop              agent_id=ae02be8a2a4d
```

**Conclusion:** Main session `Stop` fires BEFORE background agent completes. This confirms the idle-while-agents-active gap. Agent hooks fire independently after main `Stop`. No `SubagentStart` event captured (may have been missed due to timing, or not fired for background agents — need to verify).

**Note:** `SubagentStart` was NOT observed for this background agent. Only `SubagentStop`. This may be a bug or a timing issue. Test 3 shows `SubagentStart` for background agents, so it does fire — likely the event was between markers.

### Test 3: Multiple concurrent background subagents

Event sequence:
```
UserPromptSubmit          (main)
PreToolUse  tool=Agent    (main)
PostToolUse tool=Agent    (main)
SubagentStart             agent_id=a45def17737e
PreToolUse  tool=Agent    (main)
PostToolUse tool=Agent    (main)
SubagentStart             agent_id=ad38f67f0c6d
PreToolUse  tool=Agent    (main)
PostToolUse tool=Agent    (main)
SubagentStart             agent_id=ad0ceeae78c0
PreToolUse  tool=Bash     agent_id=a45def17737e
PreToolUse  tool=Bash     agent_id=ad38f67f0c6d
Stop                      (main — idle, all 3 agents still running)
PreToolUse  tool=Bash     agent_id=ad0ceeae78c0
PostToolUse tool=Bash     agent_id=a45def17737e
PostToolUse tool=Bash     agent_id=ad38f67f0c6d
SubagentStop              agent_id=a45def17737e
PostToolUse tool=Bash     agent_id=ad0ceeae78c0
SubagentStop              agent_id=ad38f67f0c6d
SubagentStop              agent_id=ad0ceeae78c0
Stop                      (main — after all agents done)
```

**Conclusion:** Each agent gets a distinct `agent_id`. `SubagentStart` fires for all three. Main `Stop` fires while agents are still running. Agents complete asynchronously. A second main `Stop` fires after all agents complete (possibly from the user acknowledging the results).

### Test 4: Subagent requiring permission

Event sequence:
```
PreToolUse  tool=Agent    (main)
SubagentStart             agent_id=ab3cf9dd7199
PreToolUse  tool=Read     agent_id=ab3cf9dd7199
PermissionRequest         agent_id=ab3cf9dd7199, tool=Read  <- carries agent_id!
PostToolUse tool=Read     agent_id=ab3cf9dd7199
SubagentStop              agent_id=ab3cf9dd7199
PostToolUse tool=Agent    (main)
Stop                      (main)
```

**Conclusion:** `PermissionRequest` from a subagent carries `agent_id`. The dashboard can distinguish which agent needs permission.

### Test 5: User interrupts while subagent running

Event sequence:
```
PreToolUse  tool=Agent    (main)
SubagentStart             agent_id=ad3a18bd5646
PreToolUse  tool=Bash     agent_id=ad3a18bd5646
(interrupt — no more events)
```

**Conclusion:** No `SubagentStop`, no `Stop`, no `PostToolUse`. The agent is orphaned — no cleanup hook fires. The dashboard must handle this case with a timeout or by clearing agents when the parent session's PID dies.

User observation: foreground agent is gone after interrupt, no way to access it. Background agents can be individually stopped.

### Test 7: Main idle while background subagent works

Event sequence:
```
PreToolUse  tool=Agent    (main)
SubagentStart             agent_id=a94f12e70ada
PostToolUse tool=Agent    (main)
Stop                      (main — idle)
PreToolUse  tool=Bash     agent_id=a94f12e70ada  <- agent works after main Stop
PostToolUse tool=Bash     agent_id=a94f12e70ada
SubagentStop              agent_id=a94f12e70ada
```

**Conclusion:** Confirms Test 2 findings. Clear sequence: main goes idle, agent continues, agent finishes.

### Test 8: Permission denied, no feedback (main process)

Event sequence:
```
PreToolUse  tool=Read     (main)
PermissionRequest         tool=Read  (main)
(no more events — no PostToolUse, no Stop)
```

**Conclusion:** Denying without feedback fires NO follow-up hook. State stays at `PermissionRequired` forever (until next interaction). This is a known gap — confirmed.

### Test 8b: Permission denied, no feedback (agent)

Event sequence:
```
PreToolUse  tool=Agent    (main)
SubagentStart             agent_id=a4a83fdd575f
PreToolUse  tool=Read     agent_id=a4a83fdd575f
PermissionRequest         agent_id=a4a83fdd575f, tool=Read
SubagentStop              agent_id=a4a83fdd575f  <- agent stops after denial
PostToolUse tool=Agent    (main)
Stop                      (main)
```

**Conclusion:** Agent denied without feedback still gets a `SubagentStop`. The agent gives up and the main session continues. Better than the main-process case — the agent lifecycle is clean.

### Test 8c: Permission denied, with feedback (agent)

Event sequence: identical to 8b — `SubagentStop` fires, main continues with `PostToolUse tool=Agent` → `Stop`.

**Conclusion:** Feedback text doesn't change the hook behavior. Agent stops cleanly either way.

---

## Round 3: Retests

### Test 2b: Single background agent — SubagentStart confirmation

Event sequence:
```
UserPromptSubmit          (main)
PreToolUse  tool=Agent    (main)
PostToolUse tool=Agent    (main)
Stop                      (main)
PreToolUse  tool=Bash     agent=af2009862927
PostToolUse tool=Bash     agent=af2009862927
SubagentStop              agent=af2009862927
UserPromptSubmit          (main — auto, processing agent result)
Stop                      (main)
```

**Conclusion:** `SubagentStart` DID NOT fire, again. But agent hooks carry `agent_id` and `SubagentStop` fires. `SubagentStart` is unreliable for background agents.

### Test 3b: Single background agent — second Stop pattern

Event sequence:
```
UserPromptSubmit          (main)
PreToolUse  tool=Agent    (main)
PostToolUse tool=Agent    (main)
SubagentStart             agent=ab61424e8882
Stop                      (main)
PreToolUse  tool=Bash     agent=ab61424e8882
PostToolUse tool=Bash     agent=ab61424e8882
SubagentStop              agent=ab61424e8882
UserPromptSubmit          (main — auto)
Stop                      (main)
```

**Conclusion:** `SubagentStart` fired this time (same prompt as 2b). Confirms unreliability. After `SubagentStop`, the main session auto-fires `UserPromptSubmit` → `Stop` to process the agent result. This is NOT user-initiated.

### Test 3c: Two background agents — second Stop timing

Event sequence:
```
UserPromptSubmit          (main)
PreToolUse  tool=Agent    (main)
PostToolUse tool=Agent    (main)
SubagentStart             agent=a61cac9ae2b6
PreToolUse  tool=Agent    (main)
SubagentStart             agent=ab18d3d549b5
PostToolUse tool=Agent    (main)
Stop                      (main — idle, both agents running)
PreToolUse  tool=Bash     agent=a61cac9ae2b6
PreToolUse  tool=Bash     agent=ab18d3d549b5
PostToolUse tool=Bash     agent=a61cac9ae2b6
PostToolUse tool=Bash     agent=ab18d3d549b5
SubagentStop              agent=ab18d3d549b5   <- sleep 6 finished first (out of order)
UserPromptSubmit          (main — auto)
Stop                      (main)
SubagentStop              agent=a61cac9ae2b6   <- sleep 3 finished second
UserPromptSubmit          (main — auto)
Stop                      (main)
```

**Conclusion:** Each `SubagentStop` triggers its own `UserPromptSubmit` → `Stop` cycle. Agents can complete out of order. The main session wakes up once per completed agent.

### Test 3d: Three background agents — confirm pattern

Event sequence:
```
UserPromptSubmit          (main)
[3x PreToolUse/PostToolUse tool=Agent + SubagentStart for each]
[3x PreToolUse tool=Bash with distinct agent_ids]
Stop                      (main — idle, all 3 running)
PostToolUse tool=Bash     agent=a09ce55ee5a3
SubagentStop              agent=a09ce55ee5a3
UserPromptSubmit          (main — auto)
PostToolUse tool=Bash     agent=ac149d7105d4
Stop                      (main)
SubagentStop              agent=ac149d7105d4
UserPromptSubmit          (main — auto)
PostToolUse tool=Bash     agent=acee2d7fe3b0
Stop                      (main)
SubagentStop              agent=acee2d7fe3b0
UserPromptSubmit          (main — auto)
Stop                      (main)
```

**Conclusion:** Pattern confirmed with 3 agents. Each agent completion triggers a main `UserPromptSubmit` → `Stop` cycle. Final `Stop` is the last one after all agents done.

### Test 5b: User explicitly stops background agent then interrupts main

Event sequence:
```
UserPromptSubmit          (main)
PreToolUse  tool=Agent    (main)
PostToolUse tool=Agent    (main)
SubagentStart             agent=a553b6bc9cd7
PreToolUse  tool=Bash     (main — sleep 10 for timing)
PreToolUse  tool=Bash     agent=a553b6bc9cd7
PostToolUse tool=Bash     (main)
Stop                      (main)
UserPromptSubmit          (main — resumed after user stopped agent)
(interrupt — no more events)
```

**Conclusion:** No `SubagentStop` observed when user explicitly stops a background agent. The main process resumed (`UserPromptSubmit`) but was then interrupted by the user — no `Stop` fires. Explicit agent stop does NOT reliably fire `SubagentStop`.

### Test 9: Permission denial recovery — new prompt clears stuck state

Event sequence:
```
UserPromptSubmit          (main)
PreToolUse  tool=Read     (main)
PermissionRequest         tool=Read  (main)
UserPromptSubmit          (main — new prompt after denial)
Stop                      (main)
```

**Conclusion:** A new `UserPromptSubmit` clears the stuck `PermissionRequired` state. The dashboard can rely on `UserPromptSubmit` to transition out of any stuck state.

---

## Methodology Notes

### Round 1 failure (2026-03-24)

All tests ran but captured zero hook payloads — only markers appeared in the log. Root cause: hooks in `~/.claude/settings.json` point to the *installed* copy at `~/.claude/claude-dashboard/scripts/hook_relay.py`, not the source copy at `~/source/claude-dashboard/scripts/hook_relay.py`. The source copy had logging; the installed copy did not.

**Fix:** After modifying the source relay, must also `cp` it to `~/.claude/claude-dashboard/scripts/hook_relay.py`. Added this to the instrumentation step.

### Round 1 observations (pre-payload, from user interaction)

- **Test 5 (interrupt with agents):** Interrupting the main process does NOT kill subagents. Each agent had to be stopped individually. After all agents were stopped, the main process resumed. This is significant — orphaned agents are a real state management concern. Note: this behavior differed in round 2 where the foreground agent was simply orphaned with no way to access it.
- **Test 4 (permissions):** `ls -la` was pre-allowed, so no permission prompt fired. Retry used `Read ~/.claude/settings.json` which did prompt.
- **Test prompts too slow:** Agent tasks that read many files take too long. Round 2 used `sleep N` for predictable timing.

### Round 2 observations

- **Installed vs source relay:** Must copy source relay to `~/.claude/claude-dashboard/scripts/` after changes.
- **Foreground vs background interrupt behavior:** Foreground agents are killed and orphaned on interrupt (no SubagentStop). Background agents survive and can be individually stopped by the user.
- **Test 8 variants (8, 8b, 8c):** Permission denial behavior differs between main process (no follow-up hook at all) and agents (SubagentStop fires cleanly). Feedback text has no effect on hook behavior.

---

## Conclusions

### Architecture Decision: Per-Agent State Tracking (Path A)

Standard hooks carry `agent_id`. We can track per-agent state keyed on `(session_id, agent_id)`.

### Data Model

- `_AgentEntry`: `agent_id`, `state`, `agent_type`
- `_SessionEntry` gains `agents: dict[str, _AgentEntry]`
- `effective_state` property: highest priority across `self.state` + all `agents[*].state`

### Agent Registration

`SubagentStart` is **unreliable** — it sometimes does not fire for background agents (observed in Tests 2, 2b; fired in 3b with identical prompt). The dashboard MUST NOT depend on `SubagentStart` for agent registration.

**Rule:** Register an agent on the first hook event carrying an `agent_id` that is NOT `SubagentStop`. Remove on `SubagentStop`. If `SubagentStart` fires, it's just another registering event — no special handling.

### State Transitions

| Event | agent_id present? | Action |
|-------|-------------------|--------|
| Any hook (not `SubagentStop`) | yes, new agent_id | Register new `_AgentEntry` with state from event |
| `PreToolUse` | yes, known agent | Update agent state (WORKING or AWAITING_INPUT) |
| `PostToolUse` | yes | Update agent state to WORKING |
| `PermissionRequest` | yes | Update agent state to PERMISSION_REQUIRED |
| `SubagentStop` | yes | Remove `_AgentEntry` |
| `Stop` (main) | no | Update main state. If agents remain active, effective_state still reflects them |
| `UserPromptSubmit` | no | Update main state to WORKING (also clears stuck PermissionRequired) |
| Any standard hook | no | Update main state (existing behavior) |

### Auto-Wake Pattern

After each `SubagentStop`, Claude Code auto-fires `UserPromptSubmit` → `Stop` on the main session to process the agent result. With N background agents, expect N such cycles. The dashboard should handle these transitions normally — they'll briefly flash main to WORKING then back to IDLE/READY, which is correct.

### Edge Cases to Handle

1. **Orphaned agents (interrupt):** No `SubagentStop` fires for foreground agents on interrupt. No `SubagentStop` observed for explicitly-stopped background agents either. Cleared by `UserPromptSubmit` (new user turn wipes all agents). Also cleared when parent session PID dies.
2. **Main idle + agents active:** `effective_state` handles this — Working agents override main Idle.
3. **Permission denied (main process):** No follow-up hook. State stays at PermissionRequired until next `UserPromptSubmit`. Known gap, unchanged.
4. **Permission denied (agent):** `SubagentStop` fires. Agent is cleaned up. No gap.
5. **Out-of-order agent completion:** Agents can complete in any order regardless of start order. Each completion triggers its own main-session wake cycle. The dashboard must handle interleaved events correctly.
6. **`SubagentStart` unreliability:** Do NOT depend on it. Register agents on first `agent_id`-carrying event that is not `SubagentStop`.

### Possible Enhancement: Interrupt Detection via Foreground Agent Wrapper

If the dashboard ran all of its own work through a foreground subagent, an interrupt (Ctrl+C/Escape) would kill that agent's PID. The dashboard could detect the PID death during the next poll cycle and transition the session state accordingly — solving the "no hook fires on interrupt" gap that currently leaves sessions stuck in their last state. This would require the hook relay or dashboard to spawn a monitored foreground agent, but the payoff is reliable interrupt detection without depending on a hook that doesn't exist.

### What NOT to do

- Do NOT depend on `SubagentStart` for agent registration — it is unreliable. Use first `agent_id`-carrying event (not `SubagentStop`).
- Do NOT use `PreToolUse tool=Agent` / `PostToolUse tool=Agent` for agent lifecycle — these are main-session events that bracket the spawn.
- Do NOT expect agents to fire `Stop` — agents get `SubagentStop`, not `Stop`.
- Do NOT expect `SubagentStop` on interrupt or explicit user stop — orphaned agents must be cleaned up via PID death detection.
