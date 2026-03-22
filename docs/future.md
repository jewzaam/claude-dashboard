# Claude Orchestrator — Future Work

> Ideas and features deferred from MVP. Captured from initial voice transcript, 2026-03-22.

## Interactive Features

### Permission Handling in Orchestrator
- Expand a session row to see the pending permission request (the command being requested)
- Accept or deny the permission directly from the orchestrator UI
- Eliminates need to switch to the Claude terminal for permission prompts
- **Open question**: How to communicate approval/denial back to the Claude process (stdin injection? hooks? file-based signaling?)

### Input Injection
- When Claude is waiting for user input, show the last response in an expandable panel
- Allow typing a reply directly in the orchestrator
- Inject that text as if typed in the Claude terminal
- **Open question**: stdin injection is hacky. Alternatives — hooks, file-watch convention, Claude Code API

### Voice Input per Session
- Record button per session row (Start Recording → Pause → Send)
- Uses existing Whisper-based voice capability (`~/source/claude-skill-voice`)
- Transcribes and injects as text input to the target session
- Same injection challenge as Input Injection above

### Sub-Agent Awareness
- The main Claude process can be "done" (idle) while background sub-agents are still running. Current state detection may show the session as idle when significant work is in progress.
- **Why**: Without this, the user has no visibility into ongoing background work. A session that looks idle may actually have 5 agents churning. The user can't make informed decisions about when to interact or wait.
- **Open question**: How are sub-agents represented in the transcript or process list? Are they child processes of claude.exe, or are they API-level constructs invisible to the OS? Need to investigate whether sub-agent count/status is discoverable from the transcript JSONL (sub-agent entries exist in `{sessionId}/subagents/`).
- Desired display: indicator showing "waiting on N agents" with some visual signal that work is happening even though the main prompt is idle.

### Elapsed Time Indicator
- Show how long the current prompt/task has been running, from initial prompt through any sub-agent work.
- **Why**: The user has mental models of how long specific tasks take (e.g., a review skill takes 15-20 minutes). Seeing "14 minutes elapsed" triggers a mental context switch — the user starts preparing to review the output. This is about enabling proactive attention management, not just passive monitoring.
- If the main prompt finishes but agents are still running, the timer should continue (total wall time from prompt to full completion).
- Post-MVP because it requires understanding sub-agent lifecycle (see above) to be accurate.

## UI Enhancements

### Row Priority
- Each row could have a priority level
- Priority affects sort order, colorization, and collapse state
- Higher priority sessions stay visible; lower priority ones auto-collapse

### Stack Direction
- User preference for stack growth direction (up vs down)
- Currently fixed to grow downward

### Sound Alerts
- Port sound capability from d4-timer-w11
- Configurable per-status sounds (e.g., chime when permission needed)

## Technical Investigation Needed

### Session Status Detection
- How does Claude Code expose its current state?
- Options to investigate:
  - Claude Code status line / internal state files
  - Hooks that fire on state changes
  - Polling process state and inferring from behavior
  - Claude Code MCP or API for session metadata

### Process Foregrounding Edge Cases
- `screen` sessions on Linux — can we activate the terminal hosting the screen session?
- Wayland compatibility (xdotool is X11-specific)
- Multiple monitors + virtual desktops on both platforms

### Input/Output Channel to Claude Sessions
- stdin injection feasibility and risks
- Hook-based approach (write file → hook picks it up)
- Claude Code watching a directory for input files
- Any official Claude Code API for session interaction
