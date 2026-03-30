[README](../../README.md) | [Getting Started](getting-started.md) | [Visual Guide](visual-guide.md) | **Session Lifecycle** | [Interactions](interactions.md) | [Settings](settings.md)

# Session Lifecycle

## Discovery

The dashboard polls `~/.claude/sessions/*.json` at a configurable interval (default: 3 seconds). Each file contains a session's PID, session ID, working directory, and start time.

A session is considered alive if its PID exists and the process name contains "claude". Dead sessions are converted to ghosts.

## State Flow

Sessions start in an unknown state and transition based on hook events from Claude Code:

```
                          ┌─────────────────────────┐
                          │                         │
    ┌──────────┐     ┌────▼────┐     ┌──────────┐   │
    │ Unknown  ├────►│ Working ├────►│  Ready   │───┘
    └──────────┘     └──┬──┬───┘     └────┬─────┘
         prompt         │  │              │ click
                        │  │         ┌────▼─────┐
                        │  │         │   Idle   │
                        │  │         └──────────┘
                   ┌────▼──┴───┐
                   │Permission │
                   │ Awaiting  │
                   └───────────┘
```

### Hook Events

Claude Code fires command hooks that the dashboard's relay script POSTs to `localhost:17384`:

| Event | Mapped State | Notes |
|-------|-------------|-------|
| `UserPromptSubmit` | Working | User typed a prompt |
| `PreToolUse` | Working (or Awaiting Input for `AskUserQuestion`) | Tool about to run |
| `PostToolUse` | Working | Tool finished |
| `PermissionRequest` | Permission Required (or Awaiting Input for `AskUserQuestion`) | Needs approval |
| `Stop` | Ready | Claude finished (intercepted from Idle → Ready) |
| `SessionEnd` | — | Session removed, ghost created |
| `SubagentStart` | — | Agent registered (unreliable; agents also register on first event with an agent ID) |
| `SubagentStop` | — | Agent removed from session |

### Ready vs Idle

When Claude finishes (`Stop` event), the state goes to **Ready** (green), not Idle. This persists until the user clicks the row, which clears it to Idle. The distinction exists so you can see at a glance which sessions have new results you haven't looked at yet.

## Ghost Sessions

When a session's PID dies or a `SessionEnd` hook fires, the dashboard converts it to an **unattached ghost**:

- Synthetic negative PID (never conflicts with real PIDs)
- Ghost emoji (`👻`) instead of status emoji
- Dimmed text color
- Left-click opens the project in VS Code (with a `.vscode/tasks.json` template that auto-launches Claude)
- Right-click offers: Open in VS Code, Open PR, Hide, Dismiss

Ghosts persist across dashboard restarts via `session-state.json`. Dismissing a ghost removes it permanently.

### Why Ghosts Exist

When you run many Claude sessions across different projects, sessions end throughout the day. Ghosts keep those projects visible so you can:

- Quickly reopen a project you were working on
- See the git status of finished work (pushed? merged?)
- Remember which projects had active work today

## State Persistence

Session state is saved to `~/.claude/claude-dashboard/session-state.json` on every UI refresh. This preserves across dashboard restarts:

- **Hidden state** — which sessions you've hidden from view
- **Flagged state** — manual flags you've toggled
- **Session state** — last known state (ready, idle, etc.)
- **Agent state** — active agents and their states

On startup, the dashboard creates unattached placeholders from the state file for any CWD without a live session.

## Agents

Sessions can have multiple agents (subprocesses spawned by Claude). The dashboard tracks each agent's state independently:

- Agents appear as a count suffix on the project path: `~/project (+2)`
- The row's displayed state is the highest-priority state across main process + all agents
- Agent permission requests are debounced 5 seconds before surfacing (most are auto-resolved by skills)
- Agents are removed when `SubagentStop` fires or the parent session dies

## Auto-Hidden Sessions

Sessions with a non-CLI entrypoint (e.g., `claude -p` for programmatic use) are automatically hidden. They still exist and receive hook events, but don't appear in the dashboard unless unhidden via the Sessions menu.

## Session Filtering

The `ignore_regex` setting filters sessions by CWD. Sessions whose CWD matches the regex are excluded entirely from the dashboard (not just hidden — they don't appear in menus or state files).
