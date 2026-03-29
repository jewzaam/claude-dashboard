# claude-dashboard

[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/) [![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

A cross-platform dashboard for monitoring and navigating multiple Claude Code sessions.

## Overview

When running many Claude Code sessions in parallel across VS Code instances and terminals, it becomes difficult to know which sessions need attention and to quickly switch between them. OS-native notifications are noisy and not actionable.

Claude Dashboard solves this with a persistent, low-profile Tkinter window that:

- Discovers all running Claude Code sessions automatically
- Receives real-time status updates via Claude Code command hooks
- Shows each session's workspace, branch, and current status (working, ready, idle, awaiting input, permission required)
- Tracks git working tree status per session (unstaged, staged, unpushed, pushed-not-merged)
- Detects merged branches and highlights them in red so you remember to rebase
- Lets you click a session to bring its containing window (VS Code, terminal) to the foreground
- Double-click to open the PR (or create one) for pushed branches
- Ghost sessions (previous sessions no longer running) persist and can be reopened in VS Code
- Minimizes to the system tray with attention indicators
- Persists settings, flags, hidden state, and window position across restarts

## Installation

### Development

```bash
make install-dev
```

### From Git

```bash
pip install git+https://github.com/jewzaam/claude-dashboard.git
```

## Setup

### Install Hook Configuration

The dashboard receives session state updates via Claude Code command hooks. Run the following to merge the required hook configuration into `~/.claude/settings.json`:

```bash
make install-hooks
```

This deep-merges `hooks-settings.json` (shipped with the project) into your existing Claude settings. It configures Claude Code to POST hook events to `http://localhost:17384/hook` for the following events: `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PermissionRequest`, `Stop`, and `SessionEnd`.

The hooks are non-blocking — if the dashboard is not running, Claude Code sessions are unaffected. The POST simply fails silently and Claude continues normally.

### Linux Dependencies (GNOME/Wayland)

On GNOME with Wayland, the dashboard requires the **[Window Calls](https://extensions.gnome.org/extension/4724/window-calls/)** GNOME Shell extension for window foregrounding. This extension provides D-Bus methods to list and activate windows — the only reliable way to foreground arbitrary windows on Wayland.

**Install via browser**: Visit [Window Calls on GNOME Extensions](https://extensions.gnome.org/extension/4724/window-calls/) and toggle it on.

Without this extension, clicking a session row will fall back to the `code` CLI for VS Code windows (which still works) but cannot foreground terminal windows.

| Platform | Window Foregrounding | Requirements |
|----------|---------------------|--------------|
| Windows 11 | `SetForegroundWindow` Win32 API | None (built-in) |
| Linux/GNOME Wayland | `window-calls` GNOME Shell extension via D-Bus | [Window Calls](https://extensions.gnome.org/extension/4724/window-calls/) |
| Linux (VS Code fallback) | `code /path` CLI | VS Code installed |

## Usage

```bash
claude-dashboard
```

Or run as a module:

```bash
python -m claude_dashboard
```

The dashboard starts a local HTTP server on port 17384 to receive hook events. Session discovery (polling `~/.claude/sessions/*.json`) runs on a configurable interval (default: 5 seconds). State updates arrive in real time via hooks.

### Options

| Option | Description |
|--------|-------------|
| `--debug` | Enable debug logging to console |
| `--log-file <path>` | Redirect logs to file (append mode) |
| `--quiet` / `-q` | Suppress non-essential output |

### Interactions

| Action | Behavior |
|--------|----------|
| **Left-click** (live session) | Foreground the VS Code/terminal window |
| **Left-click** (ghost session) | Open in VS Code with Claude auto-launch |
| **Double-click** | Open PR in browser (or create-PR page if none exists) — only for pushed branches |
| **Middle-click** | Toggle manual flag |
| **Right-click** (row) | Hide, Clear State, Open PR, Open in VS Code, Dismiss |
| **Right-click** (title bar) | Sessions visibility, Open folder in VS Code, Settings, Restart, Quit |

### Visual Indicators

Each session row shows:

- **Eye icon** (left): outer color = git working tree status, pupil = manual flag
- **Status emoji**: current session state (working, idle, permission required, etc.)
- **CWD + branch**: project path and branch name; branch turns **red** when merged into trunk
- **Container label** (right): VS Code, Terminal, etc.

Git status colors on the eye icon:

| Status | Color | Meaning |
|--------|-------|---------|
| Unstaged changes | Green | Modified files not yet staged |
| Staged uncommitted | Amber | Changes staged but not committed |
| Committed not pushed | Red | Local commits not on remote |
| Pushed not merged | Blue | Branch pushed, PR open or pending |
| Manual flag | Purple pupil | User-toggled via middle-click |

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
