# Multi-Session Claude Code Management: Landscape Analysis

## Question

Does a tool already exist that does what claude-dashboard does — monitor multiple Claude Code CLI sessions with real-time state detection, cross-platform support, and low-overhead desktop presence?

## Answer

**Yes, the space is crowded.** There are 30+ tools addressing some variant of this problem, most created in early 2026. However, none match claude-dashboard's exact niche: a native Python/Tkinter always-on-top desktop widget with hook-based state detection, cross-platform (Windows + Linux/Wayland), and zero browser/Node.js/Electron dependencies.

## The Landscape (March 2026)

### Categories

The tools fall into five categories, each solving slightly different problems:

| Category | Count | Example Tools | Primary UX |
|----------|-------|---------------|------------|
| Web dashboards | 8+ | claude-code-ui [9], Codeman [13], Stoneforge [26] | Browser tab |
| Desktop apps | 4 | Superset [17], cmux [18], Maestro [19], eyes-on-claude-code [16] | Native window |
| TUI tools | 5+ | claude-tmux [20], Termoil [21], claude-squad [25] | Terminal |
| CLI tools | 4 | csm [32], claude-code-monitor [33], ccswitch [34] | Command line |
| VS Code extensions | 4+ | Claude Session Tools [39], Sessions Explorer [38] | IDE sidebar |

### The Core Problem

The single most cited pain point across all tools is the **"missed permission prompt"** — an agent silently waiting for `[Y/n]` approval in a forgotten tab [21] [13] [18]. This is exactly what claude-dashboard's `permission_required` state solves.

The second problem is **"agentmaxxing"** — running as many parallel agents as practical, where the bottleneck shifts from AI capability to human attention management [47].

### Closest Competitors

#### eyes-on-claude-code [16]
**Most architecturally similar.** Tauri desktop app (Rust + JS), menubar/tray icon, hook-based state detection. Sound effects on state changes, always-on-top mode, opacity control. MIT licensed.
- **Differs from claude-dashboard**: Requires tmux for terminal interaction, Tauri instead of Tkinter, primarily macOS, not tested on Windows.

#### cmux [18]
**Most popular** (7,700+ GitHub stars). Native macOS terminal (Swift/AppKit) with GPU-accelerated rendering. Notification rings when agents need attention, embedded browser, socket API.
- **Differs from claude-dashboard**: macOS only, full terminal replacement (not a lightweight overlay), AGPL license.

#### Codeman [13]
**Most feature-rich web dashboard.** Desktop notifications with red blink for permission prompts, yellow for idle. Token/cost tracking, mobile support with QR auth.
- **Differs from claude-dashboard**: Web-based (requires browser), Node.js + tmux dependencies.

#### claude-squad [25]
**Most popular orchestration tool.** Homebrew-installable terminal app managing isolated tmux workspaces. Agent-agnostic.
- **Differs from claude-dashboard**: Orchestration-focused (not just monitoring), tmux-only, no Windows.

### What Anthropic Provides

Anthropic offers no built-in real-time dashboard [1] [8]. The available pieces:
- **Agent Teams** [2]: Experimental multi-agent orchestration (not monitoring)
- **Desktop App** [3]: GUI with session sidebar (separate product from CLI)
- **Hooks** [4] [5]: The extension mechanism claude-dashboard uses
- **Analytics** [6]: Aggregate usage metrics (Team/Enterprise only)

### Gap Analysis: Where claude-dashboard Fits

| Capability | Web dashboards | Desktop apps | TUI tools | **claude-dashboard** |
|-----------|---------------|-------------|-----------|---------------------|
| Always visible (no alt-tab) | No (browser tab) | Some | No (terminal) | **Yes** (always-on-top overlay) |
| Cross-platform (Win + Linux) | Yes (browser) | No (macOS-only) | Partial (tmux) | **Yes** (Tkinter) |
| Zero Node.js/Electron | No | No (Electron/Tauri) | Yes (Rust) | **Yes** (pure Python) |
| Hook-based state detection | Some | eyes-on-claude-code | No (regex/poll) | **Yes** |
| System tray with state color | No | cmux, eyes-on-claude-code | No | **Yes** |
| Click-to-foreground | No | Some | Some (tmux) | **Yes** (window-calls D-Bus / Win32) |
| Lightweight (<1% CPU idle) | Medium | Heavy (Electron) | Light | **Light** |

### Unique Differentiators

1. **Pure Python stack** — No Node.js, no Electron, no Rust toolchain. `pip install` and go.
2. **Cross-platform with native integration** — Win32 `SetForegroundWindow` on Windows, `window-calls` D-Bus on GNOME/Wayland. Not "cross-platform via browser."
3. **Borderless always-on-top overlay** — Visible across all virtual desktops without taking taskbar space. No other tool does this on both Windows and Linux.
4. **Transient Ready state** — Visual attention indicator when Claude finishes, auto-fading to Idle. Not seen in any competing tool.

### What Others Do Better

1. **Worktree management** — dmux [30], Superset [17], Stoneforge [26] handle git worktree lifecycle. claude-dashboard doesn't.
2. **Cost/token tracking** — Codeman [13], csm [32], ccusage [35] track spending. claude-dashboard doesn't.
3. **Mobile access** — Maestro [19], amux [31], Codeman [13] offer phone-based monitoring.
4. **Terminal embedding** — cmux [18], Codeman [13] embed live terminal output in the dashboard.
5. **Orchestration** — Stoneforge [26], Composio AO [27], Gas Town [28] coordinate multi-agent workflows.

## Conclusion

claude-dashboard occupies a genuine niche: **lightweight, cross-platform, always-visible session state monitoring** without requiring a browser, Node.js, Electron, or tmux. The closest competitor (eyes-on-claude-code) is macOS-focused and uses a heavier stack. The most popular tools (cmux, claude-squad) are macOS/tmux-only.

The trade-off is clear: claude-dashboard is intentionally narrow (monitoring, not orchestration) and intentionally lightweight (Tkinter, not Electron). For users who want "what needs my attention right now" visible at all times across Windows and Linux desktops, nothing else fills this exact gap.

## Methodology

Five research sub-agents searched in parallel across official docs, GitHub, HN, community blogs, and VS Code Marketplace. All sources visited in-session via WebSearch. See [citations.md](citations.md) for the full source list.
