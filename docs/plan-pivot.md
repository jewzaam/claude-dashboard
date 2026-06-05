# Plan: OTEL-Based State Detection — Full Cutover

## Problem

Dashboard state detection uses an HTTP hook server (`localhost:17384`) that
only local sessions can reach. Sandbox sessions have no path to fire hooks
back. The dashboard contains event→state mapping logic, state machine
transitions, and agent lifecycle management — complexity that belongs in the
observability layer, not the UI.

## Solution

Full cutover to OTEL-based state. Three layers:

```text
All Claude Sessions (local + sandbox)
    → command hook fires
    → relay script emits OTEL log event to collector
    → collector → Loki

OTEL Stack (recording rules)
    → evaluates hook events from Loki
    → maintains current state metric per session
    → Prometheus: claude_session_state{session_id, project, host_name}

Dashboard
    → polls Prometheus for session state metrics
    → pure display layer — no state machine, no event mapping
```

## What Gets Removed from Dashboard

| Component | Current Role | After |
|---|---|---|
| `hook_server.py` | HTTP server on :17384, event→state mapping | **Deleted** |
| `hook_relay.py` | stdin → HTTP POST relay | **Replaced** by OTEL relay |
| `hooks-settings.json` | Points hooks at HTTP relay | **Updated** to OTEL relay |
| `map_event_to_state()` | Event→StatusState mapping | **Moved** to recording rules |
| `_handle_hook_event()` | Controller state machine | **Replaced** by metric read |
| `_pending_hook_states` | Buffered states for unknown sessions | **Removed** — metric exists before dashboard discovers session |
| Agent state tracking | Per-agent_id state in controller | **Moved** to recording rules |
| Agent debounce (5s) | Permission debounce for agents | **Moved** to recording rules |
| `_STATE_PRIORITY` | Effective state = max(main, agents) | **Moved** to recording rules |
| `HOOK_PORT` config | Port 17384 | **Removed** |

## What Stays in Dashboard

| Component | Role |
|---|---|
| Session discovery | PID polling for local, openshell for sandbox |
| UI rendering | Tkinter rows, emoji, colors, interactions |
| Git status detection | Unchanged |
| Settings/persistence | Unchanged |
| Tray icon | Reads state from metric instead of internal state |
| Context menus | Unchanged |
| VS Code detection | Unchanged (D-Bus window matching) |

---

## Layer 1: OTEL Hook Relay (replaces hook_relay.py)

### Script: `scripts/hook_relay_otel.py`

All sessions (local + sandbox) use this. Command hook fires, script reads
JSON from stdin, emits OTLP log event to collector.

**Transport**: OTLP HTTP (`:4318/v1/logs`). JSON payload, no protobuf, no
SDK dependency. Just `urllib.request`.

**Event structure** (OTLP log record):

```json
{
  "resourceLogs": [{
    "resource": {
      "attributes": [
        {"key": "service.name", "value": {"stringValue": "claude-dashboard-hooks"}},
        {"key": "host.name", "value": {"stringValue": "<hostname>"}},
        {"key": "sandbox.name", "value": {"stringValue": "<SANDBOX_NAME or empty>"}}
      ]
    },
    "scopeLogs": [{
      "logRecords": [{
        "timeUnixNano": "<epoch_ns>",
        "body": {"stringValue": "<raw hook JSON>"},
        "attributes": [
          {"key": "event_name", "value": {"stringValue": "hook_event"}},
          {"key": "hook_event_name", "value": {"stringValue": "PreToolUse"}},
          {"key": "tool_name", "value": {"stringValue": "Bash"}},
          {"key": "session_id", "value": {"stringValue": "<uuid>"}},
          {"key": "agent_id", "value": {"stringValue": "<uuid or empty>"}},
          {"key": "agent_type", "value": {"stringValue": "general-purpose"}},
          {"key": "cwd", "value": {"stringValue": "/path/to/project"}}
        ]
      }]
    }]
  }]
}
```

**Env vars consumed**:

- `OTEL_EXPORTER_OTLP_ENDPOINT` — collector address (already set by claude-wrapper)
- `SANDBOX_NAME` — injected by openshell (empty for local sessions)

**Error handling**: fire-and-forget with 1s timeout. Hook must not block
Claude Code. If collector unreachable, event lost — same as today when
dashboard is down.

### Hook Configuration

Same events as today, different script:

```json
{
  "hooks": {
    "UserPromptSubmit": [{"command": "python3 <path>/hook_relay_otel.py"}],
    "PreToolUse": [{"command": "python3 <path>/hook_relay_otel.py"}],
    "PostToolUse": [{"command": "python3 <path>/hook_relay_otel.py"}],
    "Stop": [{"command": "python3 <path>/hook_relay_otel.py"}],
    "SubagentStart": [{"command": "python3 <path>/hook_relay_otel.py"}],
    "SubagentStop": [{"command": "python3 <path>/hook_relay_otel.py"}],
    "PermissionRequest": [{"command": "python3 <path>/hook_relay_otel.py"}]
  }
}
```

---

## Layer 2: Recording Rules (state derivation in OTEL stack)

State machine logic moves from `hook_server.py` / `controller.py` to
Prometheus recording rules that evaluate Loki data.

### State Derivation Logic

Recording rules run on a schedule (e.g., every 5s) and maintain a gauge
metric per session.

**Approach A: Loki ruler + Prometheus remote write**

Loki recording rules can evaluate LogQL and write metrics to Prometheus.

```yaml
# loki-rules.yaml
groups:
  - name: claude-session-state
    interval: 5s
    rules:
      # Last hook event per session in the last 60s
      - record: claude_session_last_event
        expr: |
          last_over_time(
            {service_name="claude-dashboard-hooks"}
            | unwrap event_sequence
            [60s]
          ) by (session_id, hook_event_name, tool_name, agent_id)
```

**Approach B: Prometheus recording rules over OTEL metrics**

If hook events are also emitted as OTEL metrics (counters), Prometheus
rules can derive state from counter changes.

**Approach C: Stateful service**

Small sidecar that subscribes to Loki tail API, maintains state machine
in memory, exposes Prometheus gauge. Most flexible but most complex.

**Recommendation**: Start with **Approach A** (Loki recording rules) if Loki
ruler supports the needed operations. Fall back to **Approach C** if the
state machine (READY = Stop + no subsequent activity, agent priority
aggregation) is too complex for pure LogQL.

### State Encoding

Prometheus gauge per session:

```text
claude_session_state{session_id="abc", project="/home/nmalik/source/foo", host_name="...", sandbox_name="..."} = 3
```

State values (numeric for Prometheus, dashboard maps to enum):

| Value | State |
|---|---|
| 0 | Unknown |
| 1 | Working |
| 2 | Ready |
| 3 | Idle |
| 4 | Permission Required |
| 5 | Awaiting Input |

### Agent Handling in Recording Rules

Agent effective state requires tracking per-agent_id state and computing
max priority. Two options:

1. **Per-agent metric** — recording rule emits
   `claude_agent_state{session_id, agent_id}`, dashboard reads both
   session + agent metrics and computes effective state (keeps some logic
   in dashboard)

2. **Effective state metric** — recording rule computes
   `claude_session_effective_state` that already incorporates agent priority
   (dashboard is truly stateless)

Option 2 is the goal but may be hard in pure LogQL. Option 1 is a pragmatic
intermediate step.

### READY State

READY = `Stop` event fired AND no subsequent `UserPromptSubmit` within N
seconds. This is a time-windowed absence check — feasible in LogQL:

```text
# Pseudo-LogQL: session is READY if last event was Stop and age < 120s
# and no UserPromptSubmit after it
```

If too complex for recording rules, alternative: the relay script adds a
synthetic `Ready` event N seconds after `Stop` (timer in a small daemon).
Or dashboard keeps READY logic as the one exception.

---

## Layer 3: Dashboard State Read

### Polling

On each discovery tick (~5s), dashboard queries Prometheus:

```text
curl http://localhost:9090/api/v1/query?query=claude_session_state
```

Returns all active sessions with their current state. Dashboard maps
numeric value → StatusState enum, updates entries.

### Session Correlation

The metric carries `session_id`, `project`, `host_name`, `sandbox_name`.

For local sessions: match by `session_id` (same as hook-based today).

For sandbox sessions: match by `sandbox_name` → dashboard's
`sandbox-{name}` session_id. Or match by `project` → CWD in manifest.

### Controller Changes

```python
def _update_states_from_otel(self):
    """Poll Prometheus for session states."""
    response = requests.get(
        "http://localhost:9090/api/v1/query",
        params={"query": "claude_session_state"},
        timeout=2,
    )
    for result in response.json()["data"]["result"]:
        session_id = result["metric"]["session_id"]
        state_value = int(float(result["value"][1]))
        state = _NUMERIC_TO_STATE[state_value]
        # find session entry by session_id or sandbox_name
        # update entry.state
```

---

## Before vs After: Complete Comparison

| Aspect | Before (HTTP Hooks) | After (OTEL Full Cutover) |
|---|---|---|
| State source | HTTP POST to localhost:17384 | Prometheus metric from recording rules |
| State logic location | Dashboard (hook_server.py, controller.py) | OTEL stack (recording rules) |
| Sandbox support | None (can't reach localhost) | Full — same path as local |
| Latency | <100ms | 5-30s (export + rule eval + poll) |
| States: WORKING | ✅ | ✅ |
| States: READY | ✅ | ✅ (recording rule with time window) |
| States: IDLE | ✅ | ✅ |
| States: PERMISSION_REQUIRED | ✅ | ✅ |
| States: AWAITING_INPUT | ✅ | ✅ |
| Agent tracking | ✅ per agent_id | ✅ if recording rules handle agent priority |
| Offline resilience | Works without OTEL | Requires OTEL stack running |
| Dashboard complexity | State machine + HTTP server + agent tracking | Pure metric read |
| Components removed | — | hook_server.py, hook_relay.py, state machine logic |
| Components added | — | hook_relay_otel.py, recording rules, Prometheus poller |
| External dependency | None | OTEL stack (collector + Loki + Prometheus) |

## Benefits Over HTTP Hooks

1. **Durable state transition history** — Every state change becomes a
   queryable event in the OTEL time-series. Hook-based state was ephemeral
   (in-memory only, lost on dashboard restart). OTEL gives:
   - Session state timeline (when did it go READY → WORKING → IDLE)
   - Bottleneck detection (sessions stuck in PERMISSION_REQUIRED)
   - Work pattern analysis (session duration, idle time distribution)
   - Agent lifecycle duration and termination patterns

2. **Sandbox support** — Same path for all session types (local + sandbox).
   HTTP hooks only worked for local sessions.

3. **Separation of concerns** — State derivation logic belongs in the
   observability layer, not the UI. Dashboard becomes pure display.

## Risks

1. **OTEL stack dependency** — dashboard is non-functional for state without
   it. Mitigation: degrade gracefully to Unknown state, sessions still
   visible with git status and interactions.

2. **Recording rule complexity** — agent priority aggregation and READY
   state detection may exceed LogQL capabilities. Mitigation: small stateful
   sidecar as escape hatch.

3. **Export lag** — 5-30s delay for state changes. Accepted by user.

4. **Stale metrics** — if a session dies without Stop, last state persists
   in Prometheus until series expires. Need staleness handling (e.g.,
   recording rule that sets Unknown if no events for 5 minutes).

---

## Layer 4: Session Source Configuration

Today the dashboard hardcodes two session sources (local PID discovery,
openshell sandbox discovery) with hardcoded actions (foreground window,
open VS Code). With OTEL as the single state source, sessions become rows
in a query result. The dashboard doesn't need to know *how* to find
sessions — it needs to know *what to do with them*.

### Concept: Session Source Definitions

Each source is a named configuration block:

```yaml
sources:
  local:
    query: 'claude_session_state{sandbox_name=""}'
    match_by: session_id        # correlate metric → dashboard entry
    discovery: pid              # how to find these sessions initially
    icon: null                  # use state emoji (default)
    actions:
      click: foreground_window  # built-in: D-Bus window activate
      double_click: open_pr     # built-in: open PR in browser
      right_click:
        - Hide
        - Clear State
        - Open PR

  sandbox:
    query: 'claude_session_state{sandbox_name!=""}'
    match_by: sandbox_name      # correlate metric → sandbox-{name}
    discovery: openshell         # how to find these sessions initially
    icon: beach                  # 🏖️ when idle/ghost
    actions:
      click: code_remote         # open VS Code to sandbox folder
      double_click: open_pr
      right_click:
        - Hide
        - Clear State
        - Delete Sandbox
        - Open PR

  # Future example: remote k8s sessions
  k8s:
    query: 'claude_session_state{host_name=~"pod-.*"}'
    match_by: host_name
    discovery: none              # OTEL-only discovery, no local PID
    icon: cloud                  # ☁️
    actions:
      click: ssh_connect         # ssh into pod
      right_click:
        - Hide
        - View Logs
```

### What This Enables

1. **OTEL-only session discovery** — sources with `discovery: none` exist
   purely because OTEL reports state for them. No PID file, no openshell
   CLI. Dashboard renders a row because a metric exists.

2. **Pluggable actions** — click behavior is per-source. Local session
   foregrounds a window. Sandbox opens VS Code remote. Future source
   could SSH, open a URL, or run a custom command.

3. **New sources without code changes** — add a YAML block, define the
   query and actions, dashboard renders it.

4. **Action as command template** — custom actions could be command
   strings with variable substitution:

   ```yaml
   actions:
     click: "ssh {{host_name}} -t 'screen -r claude'"
     double_click: "xdg-open https://grafana/d/session?var-session={{session_id}}"
   ```

### Discovery vs State

Two distinct concerns that are currently tangled:

| Concern | Current | After |
|---|---|---|
| **Does this session exist?** | PID file / openshell list | Source-specific discovery + OTEL metrics |
| **What state is it in?** | Hook HTTP server | Prometheus metric (same for all sources) |

With OTEL-only discovery (`discovery: none`), a session exists *because the
metric exists*. No local discovery needed. This is how remote/k8s/CI sessions
would work — the dashboard never directly accesses the host, it reads OTEL.

For local and sandbox sessions, discovery still happens via PID/openshell
(faster startup, git status, etc.), but state comes from OTEL.

### Session Lifecycle with OTEL Discovery

```text
1. Prometheus metric appears → dashboard creates row
2. State updates via metric polling
3. Metric goes stale (no events for N minutes) → row transitions to ghost
4. Metric series expires → row removed (or kept as ghost if state persisted)
```

No PID to poll. No process to validate. Session exists iff metric exists.

### Built-in Action Types

| Action | Behavior |
|---|---|
| `foreground_window` | D-Bus window activate (existing) |
| `code_remote` | `code <folder>` to open VS Code (existing for sandboxes) |
| `open_pr` | Open PR in browser if pushed-not-merged (existing) |
| `ssh_connect` | SSH to host_name |
| `open_url` | Open URL with variable substitution |
| `run_command` | Execute shell command with variable substitution |
| `none` | No action |

### Available Variables for Templates

From the Prometheus metric labels:

| Variable | Source |
|---|---|
| `{{session_id}}` | Metric label |
| `{{project}}` | Metric label (CWD path) |
| `{{host_name}}` | Metric label |
| `{{sandbox_name}}` | Metric label |
| `{{cwd}}` | Dashboard's resolved CWD for the row |
| `{{branch}}` | Git branch from dashboard |

### Configuration Location

`~/.claude/claude-dashboard/sources.yaml` — separate from settings.json
(which is UI preferences). Sources are operational config, not preferences.

Default ships with `local` and `sandbox` sources. User adds custom sources.

---

## Open Questions

1. What `host.name` do sandbox Claude sessions report in OTEL?

2. Can `SANDBOX_NAME` be injected by openshell into the container env?

3. Does Loki ruler in the current stack support recording rules with
   remote write to Prometheus?

4. Is the agent effective state calculation feasible in pure LogQL, or
   should we keep that one piece in the dashboard?

5. Should READY state be handled by recording rules (time-windowed
   absence check) or kept as dashboard-side logic?

6. What's the Prometheus series TTL / staleness threshold? Sessions that
   die need their state metric to expire or be set to Unknown.
