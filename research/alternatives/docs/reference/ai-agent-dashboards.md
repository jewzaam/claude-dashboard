# AI Agent Dashboards (Cross-Tool)

## Orchestration Platforms

These go beyond monitoring into task coordination and multi-agent workflows:

| Tool | Agents Supported | Scale | Key Feature | Citation |
|------|-----------------|-------|-------------|----------|
| Stoneforge | Claude, OpenCode, Codex | Medium | Kanban, auto-merge Stewards, event-sourced SQLite | [26] |
| Composio AO | Claude, Codex, Aider | Large (30+ agents) | Plugin architecture, autonomous CI fix | [27] |
| Gas Town | Claude Code | Large (20-30 agents) | Mayor/Polecat/Refinery model, bisecting merge queue | [28] |
| AgentDock | Claude, Cursor | Medium | Web dashboard, tmux + worktrees | [29] |

## Session Multiplexers

| Tool | Agents Supported | Key Feature | Citation |
|------|-----------------|-------------|----------|
| dmux | 11 agent types | Worktree isolation, AI-assisted merge, lifecycle hooks | [30] |
| amux (mixpeek) | Claude Code | Self-healing watchdog, phone/browser access | [31] |
| claude-squad | Claude, Codex, Aider | Homebrew-installable, tmux workspaces | [25] |

## Copilot-Specific

| Tool | Key Feature | Citation |
|------|-------------|----------|
| ghcpCliDashboard | Cross-machine sync, Copilot + Claude | [15] |
| GridWatch | Reads Copilot session data, read-only | Noted in research |

## Cursor-Specific

Cursor has built-in parallel agents (up to 8) using git worktrees. Team dashboard provides aggregate analytics. No external session monitoring tool equivalent [research notes].

## Aider / Cline

Neither has built-in multi-session management. Aider has an open discussion (issue #1839). Cline is developing "enhanced task cards" for multi-instance visibility [research notes].

## Common Patterns

1. **Git worktrees** are the dominant isolation technique across all tools
2. **"Missed permission prompt"** is the #1 pain point cited across the ecosystem [21] [13] [18]
3. **tmux dependency** is near-universal for Linux/macOS tools, limiting Windows support
4. **Most tools are macOS-first** — Windows + Linux cross-platform is rare
