# Citations

All sources visited in-session via WebSearch. Numbered sequentially.

## Official Anthropic

- [1] [CLI reference - Claude Code Docs](https://code.claude.com/docs/en/cli-reference) — Session resume/fork flags (`--continue`, `--resume`, `--fork-session`)
- [2] [Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams) — Agent Teams experimental feature, tmux/iTerm2 backends
- [3] [Use Claude Code Desktop](https://code.claude.com/docs/en/desktop) — Desktop app with multi-session sidebar
- [4] [Hooks reference](https://code.claude.com/docs/en/hooks) — Lifecycle hooks (SessionStart, Stop, PreToolUse, etc.)
- [5] [Automate workflows with hooks](https://code.claude.com/docs/en/hooks-guide) — Hook configuration guide
- [6] [Claude Code usage analytics](https://support.claude.com/en/articles/12157520-claude-code-usage-analytics) — Team/Enterprise analytics
- [7] [Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices) — Subagent spawning (up to 10 parallel)
- [8] [Feature Request: CLI session management](https://github.com/anthropics/claude-code/issues/34318) — Open request for `--list` flag

## Third-Party Dashboards (Web-Based)

- [9] [claude-code-ui (KyleAMathews)](https://github.com/KyleAMathews/claude-code-ui) — Real-time web dashboard, Durable Streams, PR/CI tracking
- [10] [claude-session-dashboard (dlupiak)](https://github.com/dlupiak/claude-session-dashboard) — Read-only analytics dashboard, scans ~/.claude
- [11] [claude-code-dashboard (Stargx)](https://github.com/Stargx/claude-code-dashboard) — Lightweight Node.js dashboard, token/cost tracking
- [12] [Claude-Code-Agent-Monitor (hoangsonww)](https://github.com/hoangsonww/Claude-Code-Agent-Monitor) — React/Node.js/WebSocket dashboard with Kanban
- [13] [Codeman (Ark0N)](https://github.com/Ark0N/Codeman) — WebUI with xterm.js, desktop notifications, cost tracking, mobile support
- [14] [multi-agent-dashboard (TheAIuniversity)](https://github.com/TheAIuniversity/multi-agent-dashboard) — React dashboard for 68+ subagents
- [15] [ghcpCliDashboard (JeffSteinbok)](https://github.com/JeffSteinbok/ghcpCliDashboard) — Web dashboard for Copilot CLI + Claude Code sessions

## Third-Party Dashboards (Desktop/Native)

- [16] [eyes-on-claude-code (joe-re)](https://github.com/joe-re/eyes-on-claude-code) — Tauri desktop app, menubar/tray, hook-based state detection
- [17] [Superset](https://github.com/superset-sh/superset) / [superset.sh](https://superset.sh/) — Electron desktop app, 10+ parallel agents, macOS only
- [18] [cmux](https://cmux.com/) / [GitHub](https://github.com/manaflow-ai/cmux) — Native macOS terminal (Swift/AppKit), 7,700+ stars, GPU-accelerated
- [19] [Maestro (RunMaestro)](https://github.com/RunMaestro/Maestro) / [runmaestro.ai](https://runmaestro.ai) — Electron desktop + mobile remote control + CLI

## Third-Party TUI Tools

- [20] [claude-tmux (nielsgroen)](https://github.com/nielsgroen/claude-tmux) — Rust TUI for tmux session management
- [21] [Termoil (fantom845)](https://github.com/fantom845/termoil) — TUI with regex-based permission prompt detection, pane border blink
- [22] [claude-dashboard (seunggabi)](https://github.com/seunggabi/claude-dashboard) — k9s-style TUI for tmux sessions
- [23] [Claude-Central (RezEnayati)](https://github.com/RezEnayati/Claude-Central) — MTA arrival-board style terminal dashboard
- [24] [TmuxCC (nyanko3141592)](https://github.com/nyanko3141592/tmuxcc) — Centralized AI agent monitoring in tmux

## Orchestration Tools

- [25] [claude-squad (smtg-ai)](https://github.com/smtg-ai/claude-squad) — Terminal app, isolated tmux workspaces, Homebrew installable
- [26] [Stoneforge](https://github.com/stoneforge-ai/stoneforge) / [stoneforge.ai](https://stoneforge.ai/) — Web dashboard + runtime, kanban, auto-merge via Stewards
- [27] [Composio Agent Orchestrator](https://github.com/ComposioHQ/agent-orchestrator) — 30+ concurrent agents, plugin architecture
- [28] [Gas Town (steveyegge)](https://github.com/steveyegge/gastown) — Go-based, 20-30 parallel agents, Mayor/Polecat/Refinery model
- [29] [AgentDock (vishalnarkhede)](https://github.com/vishalnarkhede/agentdock) — Web dashboard for parallel agents across repos
- [30] [dmux](https://github.com/standardagents/dmux) / [dmux.ai](https://dmux.ai/) — Node.js CLI, worktree isolation, 11 agent types supported
- [31] [amux (mixpeek)](https://github.com/mixpeek/amux) / [amux.io](https://amux.io/) — Browser/phone management, self-healing watchdog

## CLI Tools

- [32] [claude-sessions-monitor (csm)](https://github.com/yepzdk/claude-sessions-monitor) — CLI + web dashboard, ghost detection, quota visualization
- [33] [claude-code-monitor (onikan27)](https://github.com/onikan27/claude-code-monitor) — CLI + mobile web, macOS only
- [34] [ccswitch](https://www.ksred.com/building-ccswitch-managing-multiple-claude-code-sessions-without-the-chaos/) — Go CLI for worktree management
- [35] [ccusage (ryoppippi)](https://github.com/ryoppippi/ccusage) — Usage analysis from local JSONL files

## VS Code Extensions

- [36] [Claude Code Usage Tracker](https://marketplace.visualstudio.com/items?itemName=YahyaShareef.claude-code-usage-tracker) — Token/cost monitoring in VS Code
- [37] [Claude Session Usage (Gronsten)](https://marketplace.visualstudio.com/items?itemName=Gronsten.claude-session-usage) — Web + Code usage monitoring
- [38] [Claude Sessions Explorer](https://marketplace.visualstudio.com/items?itemName=ShahadIshraq.vscode-claude-sessions) — Browse/resume sessions from sidebar
- [39] [Claude Session Tools (Microwise AI)](https://dev.to/microwise_ai_7459783ef34d70/monitor-multiple-claude-code-sessions-in-real-time-free-vscode-extension-8l2) — Multi-session real-time monitoring

## macOS Menu Bar Apps

- [40] [Usage4Claude](https://github.com/f-is-h/Usage4Claude) — macOS menu bar, quota monitoring
- [41] [ClaudeBar](https://github.com/tddworks/ClaudeBar) — macOS menu bar, multi-service quotas
- [42] [SessionWatcher](https://www.sessionwatcher.com/) — Usage/limits/costs in menu bar

## Tmux Workflow References

- [43] [How to run Claude Code in a tmux popup](https://www.devas.life/how-to-run-claude-code-in-a-tmux-popup-window-with-persistent-sessions/) — Nested tmux popup pattern
- [44] [Claude Code + tmux: Ultimate Terminal Workflow](https://www.blle.co/blog/claude-code-tmux-beautiful-terminal) — Multi-window project sessions
- [45] [GitButler Blog: Parallel Claude Code](https://blog.gitbutler.com/parallel-claude-code) — Managing sessions without worktrees

## Community Discussions

- [46] [HN: Claudedash](https://news.ycombinator.com/item?id=47119339) — Real-time local dashboard for autonomous workflows
- [47] [Product Hunt: How many Claude Codes in parallel](https://www.producthunt.com/p/claude/how-many-claude-codes-do-you-run-in-parallel) — Community reports 3-5 typical
- [48] [awesome-agent-orchestrators](https://github.com/andyrewlee/awesome-agent-orchestrators) — Curated list of 30+ tools
- [49] [Anthropic Hid a Multi-Agent System Inside Claude Code](https://medium.com/@ayeshamughal21/anthropic-hid-a-multi-agent-system-inside-claude-code-someone-found-it-in-the-binary-99217966174e) — Agent Teams discovery

## Desktop Monitoring Libraries

- [50] [psutil](https://github.com/giampaolo/psutil) — Cross-platform process/system monitoring library
- [51] [pystray](https://pypi.org/project/pystray/) — Cross-platform system tray icon library
- [52] [ttkbootstrap](https://github.com/israel-dryer/ttkbootstrap) — Modern Tkinter theming library
