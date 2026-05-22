# Windows VS Code Foreground — Design

**Date:** 2026-05-21
**Status:** Approved (pending user review of this file)
**Owner:** Naveen Malik

## Problem

Clicking an active session row on Windows often fails to raise the target VS Code window. Symptom: the dashboard window loses focus (so `SetForegroundWindow` did *something*) but the VS Code window does not come forward. Observed when the target window is buried behind other windows on the same virtual desktop. Linux works reliably because it uses the `code <cwd>` CLI, which delegates foregrounding to VS Code itself.

The current Windows path (`claude_dashboard/platform/windows.py:136`):

1. `IsIconic` → `ShowWindow(SW_RESTORE)` if minimized.
2. `SetForegroundWindow(hwnd)` direct call.
3. On failure, alt-key trick to satisfy Windows' foreground-rights heuristic, then retry.

Two failure modes:

- **Foreground-lock denial.** `SetForegroundWindow` returns 0 unless the caller meets one of Microsoft's allowed conditions. The alt-key trick is unreliable — it works often enough to be confusing but fails frequently when the target is buried.
- **Title-match fragility.** `match_window_by_cwd` (`windows.py:87`) enumerates visible windows whose PID matches the main VS Code PID and picks one whose title contains the CWD folder name. VS Code titles vary (`filename - foldername - Visual Studio Code`, `foldername - Visual Studio Code`, Remote/WSL prefixes). The code returns the *last* visible match silently, which can be a helper window or the wrong project.

## Goals

- Make active-session clicks raise the correct Windows VS Code window reliably.
- Do not change Linux behavior — Linux is working.
- Do not change terminal/GitBash behavior on Windows — those work and have no reported issues.
- No new permission prompts, no new external dependencies.

## Non-goals

- VS Code Insiders / `code-insiders` detection. User does not use Insiders. Add later if requested.
- Fixing terminal foregrounding on Windows. Out of scope.
- Rethinking the per-OS dispatch in `platform/base.py`. Already correct — hard branches via `IS_WINDOWS`.

## Design

### Dispatch (unchanged)

`platform/base.py` already hard-branches per OS. No change.

### `foreground_window_windows()` chain (VS Code only)

For `container.container_type == VSCODE` on Windows:

1. **`code <cwd>` via hidden subprocess.**
   - `path = shutil.which("code")` — cached at module level after first call.
   - `subprocess.Popen([path, cwd], creationflags=CREATE_NO_WINDOW, stdout=DEVNULL, stderr=DEVNULL, shell=False)`.
   - Return `True`. Do not `wait()`; `code` returns quickly after handing off to the running VS Code instance, but window activation is async. Trusting the spawn is acceptable because no information from the child process informs the caller, and waiting only adds latency.
2. **Fallback hwnd path.** Used only if `shutil.which("code")` returns `None`. Existing `SetForegroundWindow` + alt-trick logic preserved as-is — same code, just gated behind `code` unavailability. Log INFO once per process lifetime when this fallback is taken.

For `container.container_type in (TERMINAL, GITBASH, SCREEN, UNKNOWN)`: skip step 1, use hwnd path directly. No regression from current behavior.

### Signature change

`platform/base.py:foreground_window()` needs `cwd` passed to the Windows branch. Currently only Linux receives it. Controller already calls `foreground_window(container, cwd=session.cwd)` at `controller.py:970`, so the change is internal to `base.py`.

### Caching

Module-level globals in `windows.py`:

- `_code_cli_path: str | None` — result of `shutil.which("code")`, computed once.
- `_code_cli_resolved: bool` — guards the cache lookup so `None` (not found) is also cached.
- `_code_fallback_logged: bool` — INFO log once per process when fallback path taken.

### What stays

- `detect_container_windows()` — unchanged.
- `match_window_by_cwd()` — unchanged. Still used for terminal containers and for the VS Code hwnd fallback when `code` is not on PATH.
- `_find_main_vscode_pid()` — unchanged.
- The hwnd foregrounding code (`SetForegroundWindow`, alt trick) — unchanged, only its invocation gate changes.

## Error handling and edge cases

| Case | Behavior |
|------|----------|
| `code` not on PATH | `shutil.which` returns `None`, cached, fall through to hwnd path. INFO log once. |
| `code` resolved but `Popen` raises | Caught, fall through to hwnd path same as missing. |
| `code` exits non-zero | Not observed by caller; we don't wait. Acceptable — silent failure no worse than today. |
| CWD with spaces | `Popen` with list args quotes correctly. No shell. |
| VS Code container but no window open | `code <cwd>` opens a new window. Matches Linux behavior. |
| Multiple VS Code windows, same folder name (worktrees) | `code <cwd>` activates the one with that exact folder. More reliable than title substring match. |
| Terminal containers | hwnd path used directly. No regression. |

## Testing

### Unit tests (`tests/test_platform_windows.py`)

- `test_foreground_window_uses_code_cli_when_available` — `shutil.which` returns `code.cmd` path; assert `Popen` called with `[resolved_path, cwd]` and `creationflags=CREATE_NO_WINDOW`; returns True.
- `test_foreground_window_falls_back_when_code_missing` — `shutil.which` returns `None`; assert hwnd path invoked. Verify INFO log emitted once.
- `test_foreground_window_terminal_skips_code_cli` — `container_type=TERMINAL`; assert `Popen` not called; hwnd path used directly.
- `test_code_cli_path_cached` — two calls; `shutil.which` invoked once.

### Linux tests

Unchanged. Linux code path is not touched.

### Manual verification (Windows, user-run)

1. Bury VS Code window behind dashboard and other apps on the same desktop → click row → window raises. Repeat 3× to confirm reliability.
2. Click a session whose container is Windows Terminal → hwnd path still raises that window.
3. Temporarily rename `code.cmd` to simulate missing → click → INFO log emitted, fallback hwnd path runs, behavior matches today.

### Coverage

Target: maintain ~91%. New branches add roughly 8 lines, all reachable via mocks.

## Rollback

Revert the commit. No state migrations, no settings changes, no hook config changes.

## Open questions

None at design time. Insiders support is deferred; can be added with a list of CLI names to probe in `shutil.which`.
