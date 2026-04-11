# C4 Context Diagram — Claude Dashboard

## System Context (Level 1)

Shows Claude Dashboard as a single box and every external actor or system it communicates with.

```mermaid
C4Context
    title System Context — Claude Dashboard

    Person(user, "Developer", "Runs Claude Code sessions in terminals and VS Code")

    System(dashboard, "Claude Dashboard", "Cross-platform desktop monitor that shows real-time state of all running Claude Code sessions")

    System_Ext(claude_code, "Claude Code", "Anthropic CLI agent; fires command hooks on state transitions")
    System_Ext(git, "Git", "Local git repositories; branch, status, and merge detection")
    System_Ext(github, "GitHub CLI (gh)", "Opens and creates pull requests in the browser")
    System_Ext(vscode, "VS Code", "IDE; dashboard foregrounds windows and launches new sessions")
    System_Ext(os_tray, "OS System Tray", "Provides persistent tray icon with menu (pystray)")
    System_Ext(os_wm, "OS Window Manager", "Window foregrounding via D-Bus/Win32 APIs")
    System_Ext(os_autostart, "OS Autostart", "XDG .desktop files (Linux) / Registry (Windows)")
    System_Ext(fs_sessions, "Claude Session Files", "~/.claude/sessions/*.json — session discovery source")
    System_Ext(fs_cost, "Cost/Usage Data", "~/.claude/my-claude-stuff-data/ — daily cost and usage limits")

    Rel(claude_code, dashboard, "Hook events via stdin → relay → HTTP POST", "JSON over localhost:17384")
    Rel(dashboard, fs_sessions, "Polls session files", "Every 3s, reads JSON")
    Rel(dashboard, git, "Reads branch, status, merge state", "Subprocess, read-only, 2s timeout")
    Rel(dashboard, github, "Opens/creates PRs", "Subprocess: gh pr view/create --web")
    Rel(dashboard, vscode, "Foregrounds windows, launches sessions", "Subprocess: code <folder>")
    Rel(dashboard, os_tray, "Renders tray icon, builds menu", "pystray API")
    Rel(dashboard, os_wm, "Brings session windows to front", "D-Bus window-calls / Win32 API")
    Rel(dashboard, os_autostart, "Registers/unregisters auto-start", "File write / Registry write")
    Rel(dashboard, fs_cost, "Reads daily cost and usage limits", "JSON file read, every ~30s")
    Rel(user, dashboard, "Clicks, drags, right-clicks rows and title bar")
    Rel(user, claude_code, "Runs sessions in terminals and VS Code")
```

## External Systems Summary

| System | Direction | Protocol | Purpose |
|--------|-----------|----------|---------|
| Claude Code | Inbound | HTTP POST (localhost:17384) | State change events (working, idle, permission, etc.) |
| Session Files | Inbound | File system read | Discover running sessions (PID, CWD, session ID) |
| Cost/Usage Data | Inbound | File system read | Daily cost totals, 5h/7d usage percentages |
| Git | Outbound | Subprocess (read-only) | Branch name, working tree status, merge detection |
| GitHub CLI | Outbound | Subprocess | Open existing PR or create-PR page in browser |
| VS Code | Outbound | Subprocess | Foreground IDE window, launch new sessions |
| OS Window Manager | Outbound | D-Bus / Win32 | Bring terminal/IDE windows to foreground |
| OS System Tray | Outbound | pystray API | Persistent tray icon with state color and menu |
| OS Autostart | Outbound | File / Registry | Register dashboard to start on login |

## Trust Boundaries

```
┌─────────────────────────────────────────────────────────┐
│ Localhost Only                                          │
│                                                         │
│  Claude Code ──stdin──▶ hook_relay.py ──HTTP──▶ Dashboard│
│                                                         │
│  No remote network calls from the dashboard itself.     │
│  GitHub API access is delegated to the `gh` CLI.        │
└─────────────────────────────────────────────────────────┘
```

All dashboard communication is **localhost-only**. The only process that reaches the internet is `gh` (GitHub CLI), invoked as a subprocess for PR operations. The dashboard never makes direct network requests to remote hosts.
