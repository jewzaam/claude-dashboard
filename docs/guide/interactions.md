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
| Live session | Clear State (C) |
| Sandbox | Clear State (C), Delete Sandbox (X) |
| Ghost | Dismiss (D) |

While a menu is open, the letter in parentheses (underlined in the menu) invokes that item. Every right-click menu has these. Delete Sandbox is X, not D, so a ghost-row Dismiss habit cannot delete a sandbox.

Dismiss removes a ghost row. Delete Sandbox removes the OpenShell sandbox itself. There is no Hide: an active session is always visible.

## Title Bar

### Left-Click (Anywhere but the Counts Label)

Toggle search. The title text becomes `/` and typed characters filter rows by path; the filter also reveals rows the ghost toggle conceals. Click again or press Escape to exit. Typing without clicking first also starts a search once the window has focus.

### Middle-Click

Toggle ghost row visibility, same as Show ghost sessions below. While searching, middle-click exits the search instead.

### Right-Click

| Item | Key | Action |
|------|-----|--------|
| Show remote sessions | R | Toggle remote rows |
| Show ghost sessions | G | Toggle ghost rows — see below |
| Open... | O | Folder picker → open selected folder in VS Code |
| Settings | S | Open settings dialog |
| Restart | T | Save state and restart the dashboard process |
| Quit | Q | Save state and exit |

Right-clicking an empty area of the window shows only the two Show toggles.

Show ghost sessions hides or shows all ghosts. Sandboxes with no VS Code window attached toggle alongside local ghosts, as do error sandboxes. Flagged rows stay pinned, and rows with VS Code connected are unaffected. Nothing is remembered per session: attach VS Code to a concealed sandbox and its row returns with the toggle still on.

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

The arrow takes the same git status color the eye would use, falling back to the row's auto-contrast text color when the tree is clean. It has a black outline, like the eye, so it stays visible when the status color matches the row background. The manual flag is not shown while the arrow is displayed.

When a command that ran long enough to be seen (about 5s) finishes, an IDLE row turns READY, so a commit, push, or upload leaves a signal — unless that project's VS Code window has focus, since you are already looking at it. Quick commands like `ls` do not signal. Starting a command clears a READY row to IDLE, since you are working in that session.

## System Tray

| Action | Result |
|--------|--------|
| Left-click (or "Toggle" menu item) | Show or hide the dashboard window |
| Settings | Open settings dialog |
| Restart | Save state and restart |
| Quit | Save state and exit |

The icon color reflects the highest-priority state across all visible sessions.
