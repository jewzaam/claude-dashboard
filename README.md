# claude-dashboard

[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/) [![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

A cross-platform dashboard for monitoring and navigating multiple Claude Code sessions.

## Overview

When running many Claude Code sessions in parallel across VS Code instances and terminals, it becomes difficult to know which sessions need attention and to quickly switch between them. OS-native notifications are noisy and not actionable.

Claude Dashboard solves this with a persistent, low-profile Tkinter window that:

- Discovers all running Claude Code sessions automatically
- Receives real-time status updates via Claude Code HTTP hooks
- Shows each session's workspace and current status (working, awaiting input, permission required)
- Lets you click a session to bring its containing window (VS Code, terminal) to the foreground
- Minimizes to the system tray with attention indicators
- Persists settings and window position across restarts

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

The dashboard receives session state updates via Claude Code HTTP hooks. Run the following to merge the required hook configuration into `~/.claude/settings.json`:

```bash
make install-hooks
```

This deep-merges `hooks-settings.json` (shipped with the project) into your existing Claude settings. It configures Claude Code to POST hook events to `http://localhost:17384/hook` for the following events: `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PermissionRequest`, `Stop`, and `SessionEnd`.

The hooks are non-blocking — if the dashboard is not running, Claude Code sessions are unaffected. The POST simply fails silently and Claude continues normally.

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
| `--debug` | Enable debug logging |
| `--quiet` / `-q` | Suppress non-essential output |

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
