# Official Claude Code Tooling

## Session Management CLI Flags

Claude Code provides single-session resume/continuation flags [1]:
- `--continue` / `-c`: Resume most recent conversation
- `--resume` / `-r`: Interactive session picker
- `--fork-session`: Copy existing session history into new session
- `/rename`: Name sessions for later retrieval

No `--list` flag exists. Open feature request at [8].

## Agent Teams [2]

Experimental multi-agent orchestration (not monitoring):
- One session acts as team lead, coordinating 2-5 teammates
- Teammates work in own context windows
- Backends: in-process, tmux split-pane, iTerm2 split-pane
- Communication via JSON inbox files on disk
- Requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

## Subagents [7]

Claude Code can spawn up to 10 subagents in parallel within a single session. These run in isolated contexts and report summaries back. Built-in parallel execution, but scoped within one session rather than cross-session.

## Desktop App [3]

Separate product from CLI with GUI sidebar for multiple sessions. Each session gets its own git worktree.

## Lifecycle Hooks [4] [5]

The extension mechanism claude-dashboard uses:
- SessionStart, Stop, SubagentStop, Notification
- PreToolUse, PostToolUse, PermissionRequest
- Command-based, HTTP, or LLM prompt hooks

## Analytics [6]

Team/Enterprise only. Aggregate metrics (lines of code, sessions, active users). Not real-time per-session state.

## Gaps

- No built-in real-time dashboard for concurrent CLI sessions
- No `--list` flag to enumerate running sessions
- Agent Teams is orchestration, not observability
- Desktop app is a separate product, not a CLI companion
