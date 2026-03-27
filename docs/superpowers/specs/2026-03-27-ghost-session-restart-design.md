# Ghost Session Restart via VS Code

**Date:** 2026-03-27
**Status:** Draft

## Problem

Ghost sessions (unattached placeholders from dead PIDs) currently dismiss on left-click with no way to relaunch Claude in that working directory. Users must manually navigate to the directory and start a new session.

## Design

### Context Menu for Ghost Sessions

Replace the current left-click-to-dismiss behavior. Ghost session rows get:

- **Left-click**: No-op (prevents accidental dismissal)
- **Middle-click**: Toggle flag (unchanged)
- **Right-click**: Per-row context menu with:
  - **Dismiss** — removes the ghost row (current single-click behavior, relocated here)
  - **Open in VS Code** — creates a VS Code tasks.json (if needed) that auto-launches `claude` on folder open, then runs `code <cwd>`

Live session right-click behavior is unchanged (window-level session visibility checklist).

#### Right-Click Routing

The current right-click handler is window-level (`_on_right_click_event` passes `(x, y)` to the controller with no row context). To support per-row context menus for ghosts:

- Each row's `<Button-3>` binding passes the `SessionInfo` to a new `_on_row_right_click` callback (same pattern as `_on_row_click`).
- The controller checks `entry.unattached`: if true, shows the ghost context menu (Dismiss / Open in VS Code). If false, shows the existing visibility checklist menu.
- The window-level `_on_right_click_event` binding remains on the title bar / non-row areas for the visibility menu.

### VS Code Tasks.json Lifecycle

The dashboard manages a `.vscode/tasks.json` file to auto-launch Claude when VS Code opens the project folder. This starts a **fresh** Claude session in the ghost's working directory (not a resume of the dead session).

**Note on `runOn: folderOpen` behavior:** While the tasks.json is present, VS Code will auto-launch `claude` every time the folder is opened — not just the first time. This is why cleanup is important. If cleanup is delayed (dashboard not running, session never registers), the user will see Claude auto-launch on every folder open until the file is removed.

#### Template

A static, project-independent `tasks.json`:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Start Claude Code",
      "type": "shell",
      "command": "claude",
      "presentation": {
        "reveal": "always",
        "panel": "new"
      },
      "runOptions": {
        "runOn": "folderOpen"
      }
    }
  ]
}
```

This template is identical across all projects — no project-specific content. The feature intentionally does not merge into existing `tasks.json` files; it either owns the whole file or leaves it alone.

#### Step 1: Menu Click ("Open in VS Code")

1. Compute the expected hash of the template content.
2. Check if `<cwd>/.vscode/tasks.json` exists.
   - **Does not exist**: Create `.vscode/` if needed, write the template, store the hash in session state keyed by CWD.
   - **Exists and hash matches template**: Take ownership (ours from a previous attempt). Store hash in session state.
   - **Exists and hash does not match**: User's custom file. Leave it alone. Do not store a hash. VS Code opens without auto-launch.
3. Run `code <cwd>` via `subprocess.Popen` (fire-and-forget, no blocking).

If `code` is not found (`FileNotFoundError`), log a warning: "VS Code 'code' command not found — install it via Command Palette > Shell Command: Install 'code' command in PATH". No crash.

#### Step 2: Session Registration (Cleanup)

Cleanup triggers in `_add_session` when a new session is discovered for a CWD that has a stored `vscode_tasks_json_hash`. This covers both hook-based registration and file-polling discovery.

1. Re-read and hash the file at `<cwd>/.vscode/tasks.json`.
2. Compare against the stored hash.
   - **Match**: Delete the file. If `.vscode/` is now empty, delete the directory.
   - **No match**: Leave the file (user modified it between write and registration).
3. Remove the hash entry from session state regardless of outcome.

#### Hash Storage

Add a `vscode_tasks_json_hash` field to the per-CWD session state dict:

```json
{
  "/home/user/source/my-project": {
    "state": "idle",
    "hidden": false,
    "flagged": false,
    "agent_count": 0,
    "vscode_tasks_json_hash": "a1b2c3d4..."
  }
}
```

- Persisted in `~/.claude/claude-dashboard/session-state.json` (existing file).
- Survives dashboard restarts — cleanup happens whenever the session eventually registers.
- Hash is removed after cleanup decision (match or no-match), so it never accumulates.
- All hash operations (write, read, delete) run on the Tkinter main thread (UI callbacks and `_add_session` both run on main thread; hook events are marshaled via `_root.after(0, ...)`).

#### Orphaned Hash Pruning

On dashboard startup, during `_load_session_state`, prune any `vscode_tasks_json_hash` entries where the CWD directory no longer exists on disk. This prevents unbounded accumulation from deleted projects.

### Edge Cases

| Scenario | Behavior |
|----------|----------|
| `.vscode/tasks.json` exists with non-matching hash (user content) | Skip write, open VS Code without auto-launch |
| Dashboard wrote file, user edits it before session registers | Hash mismatch on cleanup → leave file, remove hash |
| Dashboard crashes after writing file, before session registers | Hash persisted → cleanup happens on next dashboard run when session registers |
| Dashboard wrote file, VS Code never opened / session never registers | Hash stays in state; file actively launches claude on every VS Code folder open until manually deleted or session registers |
| Multiple ghost sessions for same CWD | Same hash, same file — idempotent |
| `code` command not found | Log warning with install instructions, no crash |
| CWD directory deleted after hash stored | Pruned on next dashboard startup |

### Files Changed

| File | Change |
|------|--------|
| `controller.py` | Ghost click → no-op; new `_open_in_editor()` method; tasks.json write/cleanup in `_add_session`; hash in state persistence; orphaned hash pruning on load; per-row right-click dispatch |
| `ui/main_window.py` | New `_on_row_right_click` callback; per-row `<Button-3>` passes `SessionInfo`; ghost context menu (Dismiss + Open in VS Code); non-row areas keep window-level visibility menu |
| `config.py` | `VSCODE_TASKS_JSON_TEMPLATE` constant, `VSCODE_TASKS_JSON_HASH` precomputed constant |

### Cross-Platform

The implementation uses `subprocess.Popen(["code", cwd])` and standard `pathlib` / `hashlib` operations — no platform-specific code. The `code` CLI is available on Linux, macOS, and Windows. File path handling uses `pathlib.Path` throughout, which normalizes separators per platform.

### Testing

Tests must be robust against accidental file system side effects:
- All file operations (tasks.json write, delete, hash) must use temporary directories, never real project paths.
- Verify existing test infrastructure blocks real file system edits (conformance tests).
- Cover: write when no file exists, skip when hash mismatches, take ownership when hash matches, cleanup on session register, cleanup skip on hash mismatch, orphaned hash pruning, `code` not found error path.

### Not In Scope

- Configurable editor command setting (future enhancement — hardcode `code` for now)
- Terminal emulator launch option
- Auto-detecting whether Claude Code VS Code extension is installed
- Session resume (`claude --resume`) — this launches a fresh session
- Merging into existing `tasks.json` files with user-defined tasks
