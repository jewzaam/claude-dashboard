[README](../../README.md) | [Getting Started](getting-started.md) | [Visual Guide](visual-guide.md) | [Session Lifecycle](session-lifecycle.md) | **Interactions** | [Settings](settings.md)

# Interactions

## Session Rows

### Left-Click

| Target | Action |
|--------|--------|
| Live session | Foreground the VS Code or terminal window containing the session |
| Live session (Ready) | Same as above, plus clear Ready → Idle |
| Ghost session | Open project in VS Code with `tasks.json` written for Claude auto-launch |

Window foregrounding uses platform-specific methods: Win32 API on Windows, Window Calls D-Bus extension on GNOME/Wayland, or `code` CLI fallback for VS Code.

### Double-Click

Toggles between Idle and Ready. From Idle it sets Ready, re-flagging the row for attention; from any other state it clears to Idle. Clearing records the state that was cleared so a stale metric reporting the same value cannot immediately undo it — see [Session Lifecycle](session-lifecycle.md).

### Middle-Click

Toggles the manual flag on the clicked row. The flag appears as a purple pupil in the eye icon and persists across restarts.

### Right-Click

The menu depends on the row type. Every menu opens with the row's path as a disabled header.

| Row type | Items |
|----------|-------|
| Live session | Hide, Clear State |
| Sandbox | Hide, Clear State, Delete Sandbox |
| Ghost | Hide, Dismiss |

Hide removes the row from the dashboard but keeps it in the Sessions menu for unhiding. Dismiss removes a ghost permanently. Delete Sandbox removes the OpenShell sandbox itself.

## Title Bar

### Left-Click (Text / Icon / Empty Area)

Toggle window shade — collapse to title bar only, or expand to show all rows. While shaded, the bar takes the color of the highest-priority session state, so a permission request is still visible. See [Visual Guide](visual-guide.md#window-shade).

### Middle-Click

Toggle ghost session visibility — hide all ghosts, or show them again. Disconnected sandboxes toggle alongside local ghosts. Flagged rows and error sandboxes are never hidden this way. Live hidden sessions are unaffected.

### Right-Click

| Item | Action |
|------|--------|
| Sessions | Submenu with a visibility checkbox per session |
| Open... | Folder picker → open selected folder in VS Code |
| Settings | Open settings dialog |
| Restart | Save state and restart the dashboard process |
| Quit | Save state and exit |

### Drag

| Target | Action |
|--------|--------|
| Title bar or any row | Move the borderless window. A 5-pixel threshold prevents accidental drags on clicks |
| Counts label (right side) | Resize the window horizontally. Width persists to settings |

## Flag Eye Icon

Leftmost element of each row, immediately left of the state emoji.

| Element | Meaning |
|---------|---------|
| Eye outline | Git working tree status — color configurable per state in Settings |
| Pupil | Manual flag, set by middle-click |
| Circular arrow (replaces the eye) | A non-Claude process is running in a VS Code terminal whose directory matches the session |

The arrow takes the same git status color the eye would use, falling back to the row's auto-contrast text color when the tree is clean. The manual flag is not shown while the arrow is displayed.

## System Tray

| Action | Result |
|--------|--------|
| Left-click (or "Toggle" menu item) | Show or hide the dashboard window |
| Unhide: \<session\> | Unhide a specific hidden session |
| Settings | Open settings dialog |
| Restart | Save state and restart |
| Quit | Save state and exit |

The tray menu is rebuilt on each open and includes one "Unhide" entry per hidden session. The icon color reflects the highest-priority state across all sessions.
