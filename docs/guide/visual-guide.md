[README](../../README.md) | [Getting Started](getting-started.md) | **Visual Guide** | [Session Lifecycle](session-lifecycle.md) | [Interactions](interactions.md) | [Settings](settings.md)

# Visual Guide

## Row Layout

Each session row displays, left to right:

```
[Eye Icon] [Status Emoji] [Project Path (+agents)] [Branch] ............. [Container]
```

- **Eye icon** — git working tree status (outer color) + manual flag (pupil color)
- **Status emoji** — current session state
- **Project path** — CWD relative to home, with agent count if any (e.g., `~/source/my-project (+2)`)
- **Branch** — `[branch-name]` if not on trunk; red if merged
- **Container** — `VS Code`, `Term`, `Git Bash`, or `screen`

## Session States

Colors and emojis shown below are defaults. All are customizable via [Settings](settings.md).

| State | Emoji | Color | Meaning |
|-------|-------|-------|---------|
| Working | `🔄` | Gray | Claude is actively processing |
| Ready | `⏸️` | Green | Claude finished; click to clear |
| Idle | `⏸️` | Dark | Waiting for user input |
| Awaiting Input | `❓` | Amber | Claude is asking a question |
| Permission Required | `⚠️` | Amber | Claude needs tool approval |
| Unattached | `👻` | Dark | Previous session, not running |

**Ready vs Idle**: When Claude finishes, the row turns green (Ready). Clicking the row clears it to Idle (dark). This gives you visual confirmation that Claude finished without the status disappearing immediately.

**Priority**: When a session has agents, the row shows the highest-priority state across the main process and all agents. Priority order: Permission Required > Awaiting Input > Working > Ready > Idle.

## Git Status Eye Icon

The eye icon to the left of each row shows git working tree status. The outer ring color indicates the git state; the pupil color indicates a manual flag. Colors are customizable via [Configuration](configuration.md#git-flag-colors).

| Status | Color | Meaning |
|--------|-------|---------|
| Unstaged changes | Green | Modified files not yet staged |
| Staged uncommitted | Amber | Changes staged but not committed |
| Committed not pushed | Red | Local commits not on remote |
| Pushed not merged | Blue | Branch pushed, PR open or pending |
| Manual flag (pupil) | Purple | User-toggled via middle-click |
| Clean | Transparent | No git changes, no flag |

Priority: manual flag > unstaged > staged > committed > pushed.

## Branch Display

- Branch appears as `[branch-name]` after the project path
- Hidden when on the trunk branch (detected from `<remote>/HEAD`)
- **Red text** when the branch has been merged into trunk — a reminder to clean up or rebase

Merge detection uses three strategies: ancestor check, `git cherry` (rebase merges), and `git diff` (squash merges).

## Title Bar

The title bar shows summary information:

```
[Eye] [Chef Kiss] Claude Dashboard (+3)          5h: 42%  7d: 78%  $183.48
```

- **Eye icon** — always green when expanded, hidden when shaded
- **Chef kiss emoji** — project mascot (PNG image with fallback to unicode)
- **Title text** — `Claude Dashboard` with `(+N)` suffix if sessions are hidden
- **5h / 7d** — OAuth usage utilization percentages (if available)
- **Daily cost** — today's total spend across all sessions

### Window Shade

Click the title bar (text, icon, or empty area) to collapse the window to just the title bar. Click again to expand. When shaded:

- Title bar background changes to the highest-priority session state color (amber if something needs attention, green if ready, dark if idle)
- Eye icon hides (redundant with background color)
- Rows update silently in the background

When `grow_up` is enabled, the title bar sits at the bottom of the window, so shading keeps it pinned in place.

## Tray Icon

The system tray icon is an eye shape whose color reflects the highest-priority **actionable** state across all visible sessions:

- **Amber** — at least one session needs permission or has a question
- **Green** — at least one session is ready (finished)
- **Dark gray** — all sessions idle or working

The tray menu includes Toggle (show/hide window), Unhide entries for hidden sessions, Settings, Restart, and Quit.

## Agent Indicators

When a session has active agents (subprocesses), the agent count appears after the path:

```
~/source/my-project (+2) [feature-branch]
```

Agent permission requests are debounced for 5 seconds before surfacing in the UI — most agent permissions are auto-resolved by skills, so this avoids flicker.

## Text Contrast

Text color is automatically computed per row using W3C sRGB contrast ratios. Light text on dark backgrounds, dark text on light backgrounds. The `text_color` setting exists for backward compatibility but is not used.
