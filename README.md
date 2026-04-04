# claude-dashboard

[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/) [![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black) [![Mutation Testing](https://github.com/jewzaam/claude-dashboard/actions/workflows/mutation.yml/badge.svg?branch=main)](https://github.com/jewzaam/claude-dashboard/actions/workflows/mutation.yml)

**Docs:** [Getting Started](docs/guide/getting-started.md) | [Visual Guide](docs/guide/visual-guide.md) | [Session Lifecycle](docs/guide/session-lifecycle.md) | [Interactions](docs/guide/interactions.md) | [Settings](docs/guide/settings.md)

A cross-platform dashboard for monitoring and navigating multiple Claude Code sessions.

## Why

When running many Claude Code sessions in parallel across VS Code instances and terminals, it becomes difficult to know which sessions need attention and to quickly switch between them. OS-native notifications are noisy and not actionable.

Claude Dashboard shows a persistent, low-profile window with one row per session. Each row shows the session's state (working, ready, permission required), git status, branch, and container. Click a row to bring its window to the foreground.

## Quick Start

```bash
pip install git+https://github.com/jewzaam/claude-dashboard.git
make install-hooks   # Required — configures Claude Code command hooks
claude-dashboard     # Or: make run (with debug logging)
```

See [Getting Started](docs/guide/getting-started.md) for full setup instructions, platform requirements, and CLI options.

## Documentation

| Guide | Description |
|-------|-------------|
| [Getting Started](docs/guide/getting-started.md) | Installation, hook setup, launch, platform support |
| [Visual Guide](docs/guide/visual-guide.md) | Row layout, status colors/emojis, git flags, title bar, tray icon |
| [Session Lifecycle](docs/guide/session-lifecycle.md) | Discovery, state transitions, ghosts, agents, persistence |
| [Interactions](docs/guide/interactions.md) | Click behaviors, menus, shade toggle, drag, tray |
| [Settings](docs/guide/settings.md) | All settings, colors, emojis, filtering, window behavior |

## Third-Party Assets

Status emoji images are from [Noto Color Emoji](https://github.com/googlefonts/noto-emoji) by Google, licensed under the [Apache License 2.0](https://github.com/googlefonts/noto-emoji/blob/main/LICENSE).

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
