[README](../../README.md) | **Getting Started** | [Visual Guide](visual-guide.md) | [Session Lifecycle](session-lifecycle.md) | [Interactions](interactions.md) | [Settings](settings.md)

# Getting Started

Claude Dashboard is a persistent, low-profile Tkinter window that monitors running [Claude Code](https://docs.anthropic.com/en/docs/claude-code) sessions. It shows which sessions need attention and lets you switch between them with a click.

## Prerequisites

- Python 3.11+
- Claude Code installed and working
- Linux: GNOME with [Window Calls](https://extensions.gnome.org/extension/4724/window-calls/) extension (for window foregrounding on Wayland)

## Install

### From Git

```bash
pip install git+https://github.com/jewzaam/claude-dashboard.git
```

### Development

```bash
git clone https://github.com/jewzaam/claude-dashboard.git
cd claude-dashboard
make install-dev
```

## Configure Hooks

The dashboard receives session state updates via Claude Code command hooks. This step is **required** for the dashboard to function:

```bash
make install-hooks
```

This does two things:

1. Copies the hook relay script (`hook_relay.py`) to `~/.claude/claude-dashboard/scripts/`
2. Deep-merges hook configuration into `~/.claude/settings.json`

The hooks configure Claude Code to POST events (tool use, permission requests, session end, etc.) to `http://localhost:17384/hook`. If the dashboard isn't running, the POST silently fails and Claude continues normally.

## Launch

```bash
claude-dashboard
```

Or as a module:

```bash
python -m claude_dashboard
```

The dashboard starts an HTTP server on port 17384, discovers running sessions from `~/.claude/sessions/`, and shows a borderless window with one row per session.

### Options

| Option | Description |
|--------|-------------|
| `--debug` | Enable debug logging to stderr |
| `--log-file <path>` | Redirect logs to file (append mode, rotates at 2 MB) |
| `--quiet` / `-q` | Suppress non-essential output |

### Run with Logging (Recommended)

```bash
make run
```

This starts the dashboard with debug logging to `~/.claude/claude-dashboard/dashboard.log`.

## Auto-Start

Enable via the Settings dialog (right-click title bar > Settings). On Linux, this creates an XDG autostart entry.

## Platform Support

| Platform | Window Foregrounding | Requirements |
|----------|---------------------|--------------|
| Windows 11 | Win32 `SetForegroundWindow` API | None |
| Linux/GNOME Wayland | D-Bus via Window Calls extension | [Window Calls](https://extensions.gnome.org/extension/4724/window-calls/) |
| Linux (VS Code fallback) | `code /path` CLI | VS Code `code` command in PATH |

Without the Window Calls extension on Linux, clicking a session row still works for VS Code windows (via the `code` CLI) but cannot foreground terminal windows.

## Key Paths

| Path | Purpose |
|------|---------|
| `~/.claude/sessions/` | Where Claude Code writes session files (read by dashboard) |
| `~/.claude/settings.json` | Claude Code settings (includes merged hook config) |
| `~/.claude/claude-dashboard/session-state.json` | Persisted session state (flags, hidden, agents) |
| `~/.config/claude-dashboard/settings.json` | Dashboard settings (Linux) |
| `~/.claude/claude-dashboard/dashboard.log` | Log file (when using `make run`) |
