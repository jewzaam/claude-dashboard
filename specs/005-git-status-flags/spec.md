# Feature Specification: Git Status Flags

**Created**: 2026-03-27
**Status**: Draft
**Version**: 0.5.0
**Feature**: 005-git-status-flags
**Dependencies**: `specs/002-hide-apply-autostart/spec.md` (flag toggle, flag persistence)

## User Stories

### User Story 10 — Git Status Flags (Priority: P1)

As a user managing multiple Claude Code sessions across repositories, I want the dashboard flag dot to automatically reflect the git status of each session's working directory, so I can see at a glance which sessions have uncommitted work, unpushed commits, or unmerged branches without leaving the dashboard.

**Why this priority**: The manual flag is already used to track "I need to come back to this session" — but the reason is almost always related to git state (uncommitted changes, unpushed work). Automating this removes a manual step and ensures nothing is forgotten.

**Independent Test**: Have a session with staged-but-uncommitted changes. The flag dot should automatically appear in the appropriate color without any user action. Middle-click should override to manual flag (highest priority).

**Acceptance Scenarios**:

1. **Given** a session's CWD has modified/untracked files not staged in git, **When** the discovery tick runs, **Then** the flag dot appears in the "unstaged changes" color.
2. **Given** a session's CWD has staged but uncommitted changes, **When** the discovery tick runs, **Then** the flag dot appears in the "staged uncommitted" color.
3. **Given** a session's CWD has local commits not pushed to the remote tracking branch, **When** the discovery tick runs, **Then** the flag dot appears in the "unpushed commits" color.
4. **Given** a session's CWD is on a branch that has been pushed but not merged into the default branch, **When** the discovery tick runs, **Then** the flag dot appears in the "unmerged branch" color.
5. **Given** a session has multiple git states simultaneously (e.g., unstaged changes AND unpushed commits), **When** the flag is displayed, **Then** the highest-priority state determines the color.
6. **Given** I middle-click a session row, **When** the flag toggles to manual, **Then** the manual flag color takes precedence over all git states. Middle-clicking again clears the manual flag, reverting to the automatic git state.
7. **Given** a session's CWD is not a git repository, **When** the discovery tick runs, **Then** no automatic flag is shown (flag dot hidden unless manually flagged).
8. **Given** a session's CWD is on the default branch with a clean working tree and nothing unpushed, **When** the discovery tick runs, **Then** no automatic flag is shown.
9. **Given** a session's git state changes (e.g., user commits), **When** the next discovery tick runs, **Then** the flag color updates or the flag disappears accordingly.
10. **Given** the dashboard restarts, **When** sessions are rediscovered, **Then** git status is re-detected from the filesystem (not restored from state file, since git state may have changed while dashboard was down).

---

## Requirements

### Functional Requirements

| ID | Requirement | Source |
|----|-------------|--------|
| FR-050 | Detect git working tree status (unstaged, staged, clean) via `git status --porcelain` subprocess on each discovery tick | AS-1, AS-2, AS-8 |
| FR-051 | Detect unpushed commits via `git log --oneline @{u}..HEAD` subprocess | AS-3 |
| FR-052 | Detect unmerged branch: current branch is not in `_DEFAULT_BRANCHES` and has a remote tracking branch | AS-4 |
| FR-053 | Show highest-priority flag state when multiple are present (manual > unstaged > staged > unpushed > unmerged) | AS-5 |
| FR-054 | Middle-click toggles manual flag, which overrides all git states | AS-6 |
| FR-055 | Skip git status detection for non-git directories | AS-7 |
| FR-056 | Re-detect git status on restart (do not persist to state file) | AS-10 |
| FR-057 | Add `GitStatus` enum to `config.py` and `git_status` field to `_SessionEntry` | Data model |
| FR-058 | Add `git_status` field to `SessionRow` NamedTuple | Data model |
| FR-059 | Add 5 flag color settings (manual, unstaged, staged, unpushed, unmerged) replacing `color_flagged` | Settings |
| FR-060 | Migrate existing `color_flagged` setting to `color_flag_manual` on load | Settings migration |
| FR-061 | Display flag dot colored by highest-priority active state (manual or git) | UI |
| FR-062 | Update settings window with 5 flag color pickers replacing single "Flagged" picker | UI |
| FR-063 | Skip git status detection for unattached (ghost) sessions | Performance |
| FR-064 | Log warning if `git status` takes >500ms and use cached result | Performance |

### Success Criteria

| ID | Criterion |
|----|-----------|
| SC-010 | Git status flags update within one discovery tick (default 3s) of a filesystem change |
| SC-011 | Manual flag always overrides automatic git status regardless of git state |
| SC-012 | Non-git directories never show an automatic flag |
| SC-013 | Dashboard startup does not restore git status from state file — always re-detects |
| SC-014 | All 5 flag colors are independently configurable via settings |

---

## Flag Priority Order

Priority determines which state is shown when multiple are present (highest priority wins):

| Priority | State | Default Color | Rationale |
|----------|-------|---------------|-----------|
| 1 (highest) | Manual flag | `#ef4444` | Explicit user intent — always trumps |
| 2 | Unstaged changes | `#dc2626` | Most at-risk — not in git, could be lost |
| 3 | Staged uncommitted | `#b91c1c` | In index but not permanent |
| 4 | Committed not pushed | `#991b1b` | Safe locally, not shared |
| 5 (lowest) | Pushed not merged | `#7f1d1d` | Safe remotely, workflow reminder |

All colors are configurable via settings. The defaults form a red graduation from bright (high priority) to dark (low priority), using the same flag dot character (U+2B24).

**Color distinguishability note**: The four non-manual colors are intentionally close in hue for aesthetic cohesion. Users who want instant distinguishability at a glance can customize them to more distinct colors via settings. A tooltip on the flag dot showing the status text (e.g., "Unstaged changes") is deferred to a future enhancement.

---

## Git Status Detection

### Detection Method

Git status is detected via subprocess calls on each discovery tick. Direct filesystem reads (as used by `detect_branch()` for `.git/HEAD`) are insufficient for full status detection — index format parsing and tree diffing are complex and fragile.

| State | Command | Interpretation |
|-------|---------|---------------|
| Unstaged changes | `git status --porcelain` | Output contains lines with `??` (untracked) or non-space in column 2 (modified not staged) |
| Staged uncommitted | `git status --porcelain` | Output contains lines with non-space in column 1, excluding `??` |
| Committed not pushed | `git log --oneline @{u}..HEAD` | Non-empty output (commits ahead of upstream) |
| Pushed not merged | Branch detection | Current branch is not in `_DEFAULT_BRANCHES` (`{"main", "master"}`) and has a remote tracking branch |

Both commands are read-only, fast (<50ms each for typical repos), and cross-platform.

**Default branch detection**: Uses the existing `_DEFAULT_BRANCHES = frozenset({"main", "master"})` set from `session.py`. This is a known limitation — repos with non-standard default branches (e.g., `develop`, `trunk`) will incorrectly show "pushed not merged" when on the default branch. This is acceptable for v0.5 and can be improved later by reading `refs/remotes/origin/HEAD`.

**Remote state staleness**: The "pushed not merged" detection reflects the last-known state of the remote tracking branch. The dashboard does not call `git fetch`, so this data may be stale. This is intentional — fetching would be a write-like network operation that violates the constitution's local-only constraint.

### Detection Frequency

Git status runs during the discovery tick (default every 3 seconds):
- Only runs for sessions with live PIDs (not unattached placeholders)
- Both `git status --porcelain` and `git log --oneline @{u}..HEAD` run as a single detection pass
- For N active sessions, this means 2N subprocess calls per tick (e.g., 16 calls for 8 sessions)
- Each call is <50ms for typical repos, so 8 sessions adds ~400ms to the tick — acceptable
- If any single `git` call takes >500ms, log a warning and use the previous cached `GitStatus` value

### Non-Git Directories

Check for `.git/HEAD` existence before running any subprocess. If absent, set `git_status = GitStatus.CLEAN` and skip. No flag shown unless manually set.

---

## Data Model Changes

### New Enum: `GitStatus`

Added to `config.py` alongside `StatusState`:

```python
class GitStatus(Enum):
    CLEAN = "clean"
    PUSHED_NOT_MERGED = "pushed_not_merged"
    COMMITTED_NOT_PUSHED = "committed_not_pushed"
    STAGED_UNCOMMITTED = "staged_uncommitted"
    UNSTAGED_CHANGES = "unstaged_changes"
```

Priority order matches enum declaration (CLEAN is lowest / no flag).

### New/Modified Entities

| Entity | Change | Fields |
|--------|--------|--------|
| `GitStatus` (enum) | New | CLEAN, PUSHED_NOT_MERGED, COMMITTED_NOT_PUSHED, STAGED_UNCOMMITTED, UNSTAGED_CHANGES |
| `_SessionEntry` | Modified | Add `git_status: GitStatus = GitStatus.CLEAN` slot. Ephemeral — not persisted. |
| `SessionRow` | Modified | Add `git_status: GitStatus` field to NamedTuple |
| `Settings` | Modified | Replace `color_flagged` with 5 `color_flag_*` fields |

### `_SessionEntry` Changes

Add `git_status: GitStatus = GitStatus.CLEAN` to slots. This is computed on each discovery tick and is NOT persisted to the state file (git state is re-detected on restart).

### `SessionRow` Changes

Add `git_status: GitStatus` field to the NamedTuple passed to the UI.

### Settings Changes

Add 5 new color settings with defaults:

```python
color_flag_manual: str = "#ef4444"       # bright red
color_flag_unstaged: str = "#dc2626"
color_flag_staged: str = "#b91c1c"
color_flag_unpushed: str = "#991b1b"
color_flag_unmerged: str = "#7f1d1d"     # dark red
```

The existing `color_flagged` setting is replaced by `color_flag_manual`. Migration happens in `load_settings()`: if the loaded JSON contains `color_flagged`, map its value to `color_flag_manual` before field filtering. The old key is removed on next `save_settings()` call (since `dataclasses.asdict` only serializes known fields).

---

## UI Behavior

### Flag Dot Display Logic

The flag dot is shown when either `flagged` (manual) is True OR `git_status` is not CLEAN. The color is determined by the highest-priority active state:

1. If `flagged` is True → use `color_flag_manual`
2. Else if `git_status == UNSTAGED_CHANGES` → use `color_flag_unstaged`
3. Else if `git_status == STAGED_UNCOMMITTED` → use `color_flag_staged`
4. Else if `git_status == COMMITTED_NOT_PUSHED` → use `color_flag_unpushed`
5. Else if `git_status == PUSHED_NOT_MERGED` → use `color_flag_unmerged`
6. Else → hide flag dot

### Middle-Click Behavior

Middle-click toggles the manual flag as today. When manual flag is active, it overrides the git status color. When manual flag is cleared, the flag reverts to the automatic git status color (or hides if git is clean).

### Settings Window

Replace the single "Flagged" color picker with 5 color pickers:

- "Manual flag"
- "Unstaged changes"
- "Staged uncommitted"
- "Unpushed commits"
- "Unmerged branch"

---

## Edge Cases

- **Detached HEAD**: No upstream to compare against. Only unstaged/staged detection applies. `git log @{u}..HEAD` fails — skip committed-not-pushed detection. Pushed-not-merged is also skipped (no branch name).
- **No remote tracking branch**: `git log @{u}..HEAD` fails. Treat as "committed not pushed" if the branch has local commits ahead of the default branch (`git log --oneline main..HEAD` or `master..HEAD`) and the branch is not itself a default branch.
- **Submodules**: Git status of the session CWD only — do not recurse into submodules (`git status` does not recurse by default).
- **Large repos**: If `git status --porcelain` takes >500ms, log a warning and use the previous cached `GitStatus` value for that session.
- **Bare repos**: No working tree. `.git/HEAD` exists but no working tree operations are meaningful. Skip detection (check for `.git/HEAD` AND that the CWD is not inside a bare repo).
- **Worktrees**: `.git` may be a file (not a directory) pointing to the actual git dir. `git status` and `git log` handle this transparently.
- **Multiple sessions same CWD**: Each session independently detects git status. They will show the same flag state since they share the same git repo.

## Clarifications

### Session 2026-03-27 (Voice Transcript)

- Q: Should the flag dot be overloaded with multiple states or should separate indicators be added? → A: Overload the single flag dot. Five states total (manual + 4 git states), distinguished by color.
- Q: What color scheme? → A: Red graduation from bright (manual/highest priority) to dark (pushed-not-merged/lowest priority). All configurable via settings.
- Q: What's the priority order? → A: Manual flag > unstaged > staged uncommitted > committed not pushed > pushed not merged. Follows data-loss risk gradient.
