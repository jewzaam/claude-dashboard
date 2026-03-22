# claude-dashboard

[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/) [![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

A cross-platform dashboard for monitoring and navigating multiple Claude Code sessions.

## Overview

When running many Claude Code sessions in parallel across VS Code instances and terminals, it becomes difficult to know which sessions need attention and to quickly switch between them. OS-native notifications are noisy and not actionable.

Claude Dashboard solves this with a persistent, low-profile Tkinter window that:

- Discovers all running Claude Code sessions automatically
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

## Usage

```bash
claude-dashboard
```

Or run as a module:

```bash
python -m claude_dashboard
```

### Options

| Option | Description |
|--------|-------------|
| `--debug` | Enable debug logging |
| `--quiet` / `-q` | Suppress non-essential output |

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
