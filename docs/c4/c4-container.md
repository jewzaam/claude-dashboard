# C4 Container Diagram — Claude Dashboard

## Container View (Level 2)

Zooms into the "Claude Dashboard" system boundary to show the major runtime containers (processes, threads, scripts) and how they communicate.

```mermaid
C4Container
    title Container Diagram — Claude Dashboard

    Person(user, "Developer", "Interacts with dashboard via mouse")

    System_Boundary(dashboard, "Claude Dashboard") {
        Container(main_loop, "Tkinter Main Loop", "Python, Tkinter", "Event loop: UI rendering, timer callbacks, hook dispatch. Single-threaded.")
        Container(hook_server, "Hook HTTP Server", "Python, http.server", "Daemon thread listening on 127.0.0.1:17384. Receives hook events, dispatches to main thread via root.after(0).")
        Container(tray_icon, "System Tray", "Python, pystray", "Daemon thread. Renders eye icon with state color. Exposes menu: toggle, unhide, settings, restart, quit.")
        Container(state_store, "State Persistence", "JSON files", "Atomic read/write of session-state.json and settings.json. Survives restarts.")
    }

    System_Boundary(hook_chain, "Hook Relay Chain") {
        Container(hook_relay, "Hook Relay Script", "Python script", "Invoked by Claude Code via stdin pipe. Reads JSON, POSTs to localhost:17384. Exits immediately. No return channel.")
    }

    System_Ext(claude_code, "Claude Code", "Fires command hooks on state transitions")
    System_Ext(fs_sessions, "Session Files", "~/.claude/sessions/*.json")
    System_Ext(git_repos, "Git Repositories", "Local .git/ directories")
    System_Ext(gh_cli, "GitHub CLI", "gh pr view/create")
    System_Ext(vscode, "VS Code", "code CLI")
    System_Ext(os_wm, "OS Window Manager", "D-Bus / Win32")
    System_Ext(cost_data, "Cost/Usage Files", "session-tracker, oauth_usage")

    Rel(claude_code, hook_relay, "Pipes hook JSON to stdin", "Per-event invocation")
    Rel(hook_relay, hook_server, "POST /hook", "HTTP, JSON, localhost:17384")
    Rel(hook_server, main_loop, "root.after(0, callback)", "Thread-safe dispatch to main thread")
    Rel(main_loop, state_store, "Read on startup, write on every UI refresh", "Atomic JSON")
    Rel(main_loop, tray_icon, "Update icon color, rebuild menu", "In-process API calls")
    Rel(main_loop, fs_sessions, "Poll every 3s", "File read")
    Rel(main_loop, git_repos, "Branch, status, merge detection", "Subprocess, 2s timeout")
    Rel(main_loop, gh_cli, "Open/create PR", "Subprocess, 10s timeout")
    Rel(main_loop, vscode, "Launch/foreground", "Subprocess")
    Rel(main_loop, os_wm, "Bring windows to front", "D-Bus / Win32 API")
    Rel(main_loop, cost_data, "Read cost and usage", "File read, every ~30s")
    Rel(user, main_loop, "Click, drag, right-click, middle-click")
    Rel(user, tray_icon, "Left-click toggle, right-click menu")
```

## Container Descriptions

### Hook Relay Script
- **Runtime**: Separate Python process, invoked per-event by Claude Code's hook system
- **Lifecycle**: Starts when Claude Code fires a hook, exits after POST (or on failure). Stateless
- **Communication**: Reads stdin (JSON payload), POSTs to `http://127.0.0.1:17384/hook` with 2s timeout
- **Failure mode**: Silent exit — never blocks or errors back to Claude Code
- **Debug mode**: Logs raw payloads to `~/.claude/claude-dashboard/logs/hook-payloads.jsonl` (rotating, 2 MB)

### Hook HTTP Server
- **Runtime**: Daemon thread within the dashboard process
- **Lifecycle**: Starts on dashboard launch, stops on shutdown. Uses `SO_REUSEADDR`/`SO_REUSEPORT` for clean restarts
- **Communication**: Accepts POST /hook (max 64 KB), parses JSON, maps event to StatusState, dispatches callback to main thread
- **Thread safety**: All state mutation happens on the main thread via `root.after(0, fn)` — the server thread never touches session state directly

### Tkinter Main Loop
- **Runtime**: Main thread of the dashboard process
- **Lifecycle**: Entered after all initialization, runs until quit/restart
- **Responsibilities**:
  - Session discovery (periodic timer)
  - Hook event processing (dispatched from server thread)
  - UI rendering (rows, title bar, menus)
  - Git status detection (subprocess calls)
  - User interaction handling (all click/drag events)
  - State persistence (writes session-state.json on every refresh)
  - Settings application
  - Cost/usage reading

### System Tray
- **Runtime**: Daemon thread (pystray's `icon.run()`)
- **Lifecycle**: Starts after main loop begins, stops on shutdown
- **Communication**: Main loop calls `update_icon()` and `rebuild_menu()` to change tray state
- **Menu**: Dynamically rebuilt with per-session "Unhide" items when sessions are hidden

### State Persistence
- **Files**:
  - `~/.claude/claude-dashboard/session-state.json` — session flags, visibility, state, agents
  - `~/.config/claude-dashboard/settings.json` (Linux) or `~/AppData/Roaming/claude-dashboard/settings.json` (Windows) — user preferences
- **Atomicity**: Write to temp file, then rename (POSIX atomic)
- **Frequency**: State file written on every UI refresh; settings written on explicit save

## Threading Model

```
┌──────────────────────────────────────────────────────┐
│ Main Thread (Tkinter mainloop)                       │
│                                                      │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │ UI Rendering │  │ Timer Ticks  │  │ Hook       │  │
│  │ & Events     │  │ (discovery,  │  │ Callbacks  │  │
│  │              │  │  refresh)    │  │ (after(0)) │  │
│  └─────────────┘  └──────────────┘  └────────────┘  │
└──────────────────────┬───────────────────────────────┘
                       │ root.after(0, fn)
┌──────────────────────┴───────────────────────────────┐
│ Daemon Threads                                       │
│  ┌─────────────────┐  ┌────────────────────────────┐ │
│  │ Hook HTTP Server │  │ pystray Tray Icon          │ │
│  │ (port 17384)     │  │ (native OS tray loop)      │ │
│  └─────────────────┘  └────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

All session state is owned by the main thread. Daemon threads only dispatch events inward — they never read or write session state directly.
