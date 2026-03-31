[README](../../README.md) | [Getting Started](getting-started.md) | [Visual Guide](visual-guide.md) | [Session Lifecycle](session-lifecycle.md) | **Interactions** | [Settings](settings.md)

# Interactions

## Session Rows

### Left-Click

| Target | Action |
|--------|--------|
| Live session | Foreground the VS Code or terminal window containing the session |
| Live session (Ready) | Same as above, plus clear Ready → Idle |
| Ghost session | Open project in VS Code with Claude auto-launch |

Window foregrounding uses platform-specific methods: Win32 API on Windows, Window Calls D-Bus extension on GNOME/Wayland, or `code` CLI fallback for VS Code.

### Double-Click

Opens the GitHub PR for the session's branch (if pushed and not merged):

- First tries `gh pr view --web` to open an existing PR
- If no PR exists, falls back to `gh pr create --web` (GitHub's create-PR page)
- Requires the [GitHub CLI](https://cli.github.com/) (`gh`) installed

### Middle-Click

Toggles the manual flag on the clicked row. The flag appears as a purple pupil in the eye icon and persists across restarts.

### Right-Click (Live Session)

| Item | Action |
|------|--------|
| Open PR | Open PR in browser (only shown for pushed-not-merged branches) |
| Hide | Hide row from dashboard |
| Clear State | Reset to Idle, clear all agents |

### Right-Click (Ghost Session)

| Item | Action |
|------|--------|
| Open PR | Open PR in browser (only shown for pushed-not-merged branches) |
| Open in VS Code | Write tasks.json and launch VS Code |
| Hide | Hide from dashboard |
| Dismiss | Remove ghost permanently |

## Title Bar

### Left-Click (Text / Icon / Empty Area)

Toggle window shade — collapse to title bar only, or expand to show all rows. See [Visual Guide](visual-guide.md#window-shade) for details.

### Left-Click (Cost / Usage Labels)

Open a popup showing daily cost for the last 14 days with proportional bars. The popup dismisses when the mouse leaves the cost label area.

### Middle-Click

Toggle ghost (unattached) session visibility. Middle-click to hide all ghosts, middle-click again to show them. Live hidden sessions are not affected.

### Right-Click

| Item | Action |
|------|--------|
| Sessions | Submenu with visibility checkbox per session |
| Open... | Folder picker → open selected folder in VS Code |
| Settings | Open settings dialog |
| Restart | Save state and restart the dashboard process |
| Quit | Save state and exit |

### Drag

Click and drag anywhere on the title bar (or any row) to move the borderless window. A 5-pixel movement threshold prevents accidental drags from triggering on clicks.

## System Tray

| Action | Result |
|--------|--------|
| Left-click (or "Toggle" menu item) | Show or hide the dashboard window |
| Unhide: \<session\> | Unhide a specific hidden session |
| Settings | Open settings dialog |
| Restart | Save state and restart |
| Quit | Save state and exit |

The tray menu dynamically includes "Unhide" entries for each hidden session.
