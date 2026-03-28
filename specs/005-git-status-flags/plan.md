# Git Status Flags Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Overload the dashboard flag dot with automatic git status detection — 5 states (manual flag + 4 git states) distinguished by configurable red-graduated colors.

**Architecture:** Add a `GitStatus` enum and a `detect_git_status()` function in `session.py`. The discovery tick calls it per session. The flag dot color is determined by the highest-priority active state (manual flag > git states). Git detection uses `git status --porcelain` and `git log` subprocess calls, gated by `.git/HEAD` existence.

**Tech Stack:** Python 3.11+, subprocess (`git`), Tkinter, existing settings/config patterns.

**Spec:** `specs/005-git-status-flags/spec.md`

---

## Constitution Check

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Cross-Platform First | PASS | `git` subprocess calls are cross-platform |
| II. Low Profile | PASS | Piggybacks on existing discovery tick, no new threads |
| III. Proven Stack | PASS | Uses existing dataclass settings, enum, subprocess patterns |
| IV. Configuration Over Opinion | PASS | All 5 flag colors configurable via settings |
| V. Incremental Delivery | PASS | Layers on existing flag mechanism |
| VI. Simplicity | PASS | Single flag dot with color overloading, no new UI elements |

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `claude_dashboard/config.py` | Modify | Add `GitStatus` enum, replace `DEFAULT_COLOR_FLAGGED` with 5 `DEFAULT_COLOR_FLAG_*` constants |
| `claude_dashboard/session.py` | Modify | Add `detect_git_status()` function |
| `claude_dashboard/settings.py` | Modify | Replace `color_flagged` with 5 `color_flag_*` fields, add migration in `load_settings()` |
| `claude_dashboard/models.py` | Modify | Add `git_status` field to `SessionRow` |
| `claude_dashboard/controller.py` | Modify | Add `git_status` slot to `_SessionEntry`, call `detect_git_status` in tick, pass to UI |
| `claude_dashboard/ui/main_window.py` | Modify | Flag color from git_status/flagged priority, show/hide logic |
| `claude_dashboard/ui/settings_window.py` | Modify | Replace single flagged color picker with 5 pickers |
| `tests/test_session.py` | Modify | Add tests for `detect_git_status()` |
| `tests/test_settings.py` | Modify | Add tests for new color fields and migration |
| `tests/test_agent_tracking.py` | Modify | Update `_serialize_entry` if needed for new slot |

---

## Task 1: GitStatus Enum and Config Constants

**Files:**
- Modify: `claude_dashboard/config.py:40-47` (after StatusState), `claude_dashboard/config.py:65-71` (color constants)

- [ ] **Step 1: Add GitStatus enum to config.py**

After the `StatusState` enum (line 47), add:

```python
class GitStatus(Enum):
    """Git working tree status for flag dot display."""

    CLEAN = "clean"
    PUSHED_NOT_MERGED = "pushed_not_merged"
    COMMITTED_NOT_PUSHED = "committed_not_pushed"
    STAGED_UNCOMMITTED = "staged_uncommitted"
    UNSTAGED_CHANGES = "unstaged_changes"
```

- [ ] **Step 2: Replace DEFAULT_COLOR_FLAGGED with 5 flag color constants**

Replace line 70 (`DEFAULT_COLOR_FLAGGED = "#7c3aed"`) with:

```python
DEFAULT_COLOR_FLAG_MANUAL = "#ef4444"
DEFAULT_COLOR_FLAG_UNSTAGED = "#dc2626"
DEFAULT_COLOR_FLAG_STAGED = "#b91c1c"
DEFAULT_COLOR_FLAG_UNPUSHED = "#991b1b"
DEFAULT_COLOR_FLAG_UNMERGED = "#7f1d1d"
```

- [ ] **Step 3: Verify no import errors**

Run: `python3 -c "from claude_dashboard.config import GitStatus, StatusState"`
Expected: No output (success)

- [ ] **Step 4: Commit**

---

## Task 2: Settings — New Flag Color Fields and Migration

**Files:**
- Modify: `claude_dashboard/settings.py:54` (color_flagged field), `claude_dashboard/settings.py:62-114` (load_settings)
- Test: `tests/test_settings.py`

- [ ] **Step 1: Write failing test for new color fields**

In `tests/test_settings.py`, add to `TestLoadSettings`:

```python
def test_loads_flag_color_fields(self, tmp_settings_path):
    data = {"color_flag_manual": "#ff0000", "color_flag_unstaged": "#cc0000"}
    tmp_settings_path.write_text(json.dumps(data), encoding="utf-8")
    settings = load_settings(path=tmp_settings_path)
    assert settings.color_flag_manual == "#ff0000"
    assert settings.color_flag_unstaged == "#cc0000"
```

- [ ] **Step 2: Write failing test for migration from color_flagged**

```python
def test_migrates_color_flagged_to_color_flag_manual(self, tmp_settings_path):
    data = {"color_flagged": "#purple1"}
    tmp_settings_path.write_text(json.dumps(data), encoding="utf-8")
    settings = load_settings(path=tmp_settings_path)
    assert settings.color_flag_manual == "#purple1"
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `make test`
Expected: FAIL — `color_flag_manual` not a field on Settings

- [ ] **Step 4: Update Settings dataclass**

In `settings.py`, replace `color_flagged` field (line 54) with:

```python
color_flag_manual: str = config.DEFAULT_COLOR_FLAG_MANUAL
color_flag_unstaged: str = config.DEFAULT_COLOR_FLAG_UNSTAGED
color_flag_staged: str = config.DEFAULT_COLOR_FLAG_STAGED
color_flag_unpushed: str = config.DEFAULT_COLOR_FLAG_UNPUSHED
color_flag_unmerged: str = config.DEFAULT_COLOR_FLAG_UNMERGED
```

- [ ] **Step 5: Add migration in load_settings()**

In `load_settings()`, before the `known_fields` filtering (line 80), add:

```python
# Migrate legacy color_flagged → color_flag_manual
if "color_flagged" in raw and "color_flag_manual" not in raw:
    raw["color_flag_manual"] = raw.pop("color_flagged")
elif "color_flagged" in raw:
    del raw["color_flagged"]
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `make test`
Expected: PASS

- [ ] **Step 7: Write roundtrip and dual-key migration tests**

```python
def test_flag_color_roundtrip(self, tmp_settings_path):
    original = Settings(color_flag_manual="#aa0000", color_flag_unmerged="#330000")
    save_settings(original, path=tmp_settings_path)
    loaded = load_settings(path=tmp_settings_path)
    assert loaded.color_flag_manual == "#aa0000"
    assert loaded.color_flag_unmerged == "#330000"

def test_migration_both_keys_prefers_new(self, tmp_settings_path):
    """When both color_flagged and color_flag_manual exist, keep the new one."""
    data = {"color_flagged": "#old", "color_flag_manual": "#new"}
    tmp_settings_path.write_text(json.dumps(data), encoding="utf-8")
    settings = load_settings(path=tmp_settings_path)
    assert settings.color_flag_manual == "#new"
```

- [ ] **Step 8: Run tests, commit**

Run: `make check`

---

## Task 3: Git Status Detection Function

**Files:**
- Modify: `claude_dashboard/session.py` (add `import subprocess`, add detect_git_status after detect_branch)
- Test: `tests/test_session.py`

- [ ] **Step 1: Add `import subprocess` to session.py**

At the top of `session.py`, add `import subprocess` to the imports.

- [ ] **Step 2: Write failing tests for detect_git_status**

In `tests/test_session.py`, add `import subprocess` at the top, then a new class:

```python
from claude_dashboard.config import GitStatus
from claude_dashboard.session import detect_git_status


class TestDetectGitStatus:
    def test_clean_repo(self, tmp_path):
        """Clean git repo returns CLEAN."""
        subprocess.run(["git", "init"], cwd=tmp_path, capture_output=True)
        subprocess.run(["git", "commit", "--allow-empty", "-m", "init"], cwd=tmp_path, capture_output=True)
        result = detect_git_status(cwd=str(tmp_path))
        assert result == GitStatus.CLEAN

    def test_unstaged_changes(self, tmp_path):
        """Untracked file returns UNSTAGED_CHANGES."""
        subprocess.run(["git", "init"], cwd=tmp_path, capture_output=True)
        subprocess.run(["git", "commit", "--allow-empty", "-m", "init"], cwd=tmp_path, capture_output=True)
        (tmp_path / "new_file.txt").write_text("hello")
        result = detect_git_status(cwd=str(tmp_path))
        assert result == GitStatus.UNSTAGED_CHANGES

    def test_staged_uncommitted(self, tmp_path):
        """Staged file returns STAGED_UNCOMMITTED."""
        subprocess.run(["git", "init"], cwd=tmp_path, capture_output=True)
        subprocess.run(["git", "commit", "--allow-empty", "-m", "init"], cwd=tmp_path, capture_output=True)
        (tmp_path / "staged.txt").write_text("hello")
        subprocess.run(["git", "add", "staged.txt"], cwd=tmp_path, capture_output=True)
        result = detect_git_status(cwd=str(tmp_path))
        assert result == GitStatus.STAGED_UNCOMMITTED

    def test_committed_not_pushed(self, tmp_path):
        """Commit on non-default branch with no upstream returns COMMITTED_NOT_PUSHED."""
        subprocess.run(["git", "init"], cwd=tmp_path, capture_output=True)
        subprocess.run(["git", "commit", "--allow-empty", "-m", "init"], cwd=tmp_path, capture_output=True)
        subprocess.run(["git", "checkout", "-b", "feature-x"], cwd=tmp_path, capture_output=True)
        subprocess.run(["git", "commit", "--allow-empty", "-m", "feature work"], cwd=tmp_path, capture_output=True)
        result = detect_git_status(cwd=str(tmp_path))
        assert result == GitStatus.COMMITTED_NOT_PUSHED

    def test_pushed_not_merged(self, tmp_path):
        """Non-default branch with upstream and no divergence returns PUSHED_NOT_MERGED."""
        # Create a bare "remote" and clone it
        bare = tmp_path / "bare.git"
        clone = tmp_path / "clone"
        subprocess.run(["git", "init", "--bare", str(bare)], capture_output=True)
        subprocess.run(["git", "clone", str(bare), str(clone)], capture_output=True)
        # Create initial commit on main
        subprocess.run(["git", "commit", "--allow-empty", "-m", "init"], cwd=clone, capture_output=True)
        subprocess.run(["git", "push", "-u", "origin", "main"], cwd=clone, capture_output=True)
        # Create feature branch, push it (nothing ahead)
        subprocess.run(["git", "checkout", "-b", "feature-y"], cwd=clone, capture_output=True)
        subprocess.run(["git", "commit", "--allow-empty", "-m", "feat"], cwd=clone, capture_output=True)
        subprocess.run(["git", "push", "-u", "origin", "feature-y"], cwd=clone, capture_output=True)
        result = detect_git_status(cwd=str(clone))
        assert result == GitStatus.PUSHED_NOT_MERGED

    def test_non_git_dir(self, tmp_path):
        """Non-git directory returns CLEAN."""
        result = detect_git_status(cwd=str(tmp_path))
        assert result == GitStatus.CLEAN

    def test_unstaged_trumps_staged(self, tmp_path):
        """Both unstaged and staged returns UNSTAGED_CHANGES (higher priority)."""
        subprocess.run(["git", "init"], cwd=tmp_path, capture_output=True)
        subprocess.run(["git", "commit", "--allow-empty", "-m", "init"], cwd=tmp_path, capture_output=True)
        (tmp_path / "staged.txt").write_text("staged")
        subprocess.run(["git", "add", "staged.txt"], cwd=tmp_path, capture_output=True)
        (tmp_path / "unstaged.txt").write_text("unstaged")
        result = detect_git_status(cwd=str(tmp_path))
        assert result == GitStatus.UNSTAGED_CHANGES

    def test_nonexistent_dir(self):
        """Nonexistent directory returns CLEAN."""
        result = detect_git_status(cwd="/nonexistent/path/xyz")
        assert result == GitStatus.CLEAN

    def test_default_branch_clean(self, tmp_path):
        """Default branch with clean tree returns CLEAN."""
        subprocess.run(["git", "init", "-b", "main"], cwd=tmp_path, capture_output=True)
        subprocess.run(["git", "commit", "--allow-empty", "-m", "init"], cwd=tmp_path, capture_output=True)
        result = detect_git_status(cwd=str(tmp_path))
        assert result == GitStatus.CLEAN
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `make test`
Expected: FAIL — `detect_git_status` does not exist

- [ ] **Step 4: Implement detect_git_status**

In `session.py`, after `detect_branch()`, add. Note: uses top-level import of `GitStatus` (add `from claude_dashboard.config import GitStatus` alongside existing imports from config). The `.git` existence check uses `Path(cwd) / ".git"` (not `.git/HEAD`) to support worktrees where `.git` is a file.

```python
def detect_git_status(*, cwd: str) -> GitStatus:
    """Detect git working tree status for flag display.

    Returns the highest-priority git status found. Uses subprocess calls
    to git (read-only). Returns CLEAN for non-git directories.
    """
    # Check .git existence (works for both directories and worktree files)
    git_path = Path(cwd) / ".git"
    if not git_path.exists():
        return GitStatus.CLEAN

    has_unstaged = False
    has_staged = False

    # Working tree + index status
    try:
        result = subprocess.run(
            ["git", "status", "--porcelain"],
            cwd=cwd,
            capture_output=True,
            text=True,
            timeout=2,
        )
        if result.returncode == 0:
            for line in result.stdout.splitlines():
                if len(line) < 2:
                    continue
                index_col = line[0]
                work_col = line[1]
                if line.startswith("??"):
                    has_unstaged = True
                else:
                    if work_col != " ":
                        has_unstaged = True
                    if index_col != " ":
                        has_staged = True
    except (OSError, subprocess.TimeoutExpired):
        return GitStatus.CLEAN

    if has_unstaged:
        return GitStatus.UNSTAGED_CHANGES
    if has_staged:
        return GitStatus.STAGED_UNCOMMITTED

    # Cache branch detection (used for both unpushed and unmerged checks)
    branch = detect_branch(cwd=cwd)

    # Unpushed commits (ahead of upstream)
    has_upstream = False
    try:
        result = subprocess.run(
            ["git", "log", "--oneline", "@{u}..HEAD"],
            cwd=cwd,
            capture_output=True,
            text=True,
            timeout=2,
        )
        if result.returncode == 0:
            has_upstream = True
            if result.stdout.strip():
                return GitStatus.COMMITTED_NOT_PUSHED
    except (OSError, subprocess.TimeoutExpired):
        pass

    # No upstream — check if on non-default branch with commits ahead of default
    if not has_upstream and branch:
        for default in _DEFAULT_BRANCHES:
            try:
                fallback = subprocess.run(
                    ["git", "log", "--oneline", f"{default}..HEAD"],
                    cwd=cwd,
                    capture_output=True,
                    text=True,
                    timeout=2,
                )
                if fallback.returncode == 0 and fallback.stdout.strip():
                    return GitStatus.COMMITTED_NOT_PUSHED
                break  # Found a valid default branch ref
            except (OSError, subprocess.TimeoutExpired):
                continue

    # Pushed but not merged — on non-default branch with upstream
    if branch:
        return GitStatus.PUSHED_NOT_MERGED

    return GitStatus.CLEAN
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `make test`
Expected: PASS

- [ ] **Step 6: Commit**

---

## Task 4: Data Model — SessionEntry and SessionRow

**Files:**
- Modify: `claude_dashboard/controller.py:91-110` (_SessionEntry slots/init)
- Modify: `claude_dashboard/models.py:11-21` (SessionRow)

- [ ] **Step 1: Add git_status to _SessionEntry**

In `controller.py`, add `"git_status"` to `__slots__` (after `"flagged"`):

```python
__slots__ = (
    "session",
    "container",
    "state",
    "hidden",
    "branch",
    "flagged",
    "git_status",
    "agents",
    "unattached",
)
```

In `__init__`, add:

```python
self.git_status: GitStatus = GitStatus.CLEAN
```

Add import at top of controller.py:

```python
from claude_dashboard.config import GitStatus
```

- [ ] **Step 2: Add git_status to SessionRow**

In `models.py`, add `git_status` field after `flagged`:

```python
class SessionRow(NamedTuple):
    """Data passed from controller to UI for each visible session row."""

    session: SessionInfo
    state: StatusState
    container: ContainerInfo | None
    branch: str
    flagged: bool
    git_status: "GitStatus"
    agent_count: int
    unattached: bool
```

Add import:

```python
from claude_dashboard.config import GitStatus
```

- [ ] **Step 3: Update _do_refresh_ui to pass git_status**

In `controller.py:_do_refresh_ui()`, update the `SessionRow` construction (around line 552):

```python
SessionRow(
    session=entry.session,
    state=entry.effective_state,
    container=entry.container,
    branch=entry.branch,
    flagged=entry.flagged,
    git_status=entry.git_status,
    agent_count=len(entry.agents),
    unattached=entry.unattached,
)
```

- [ ] **Step 4: Call detect_git_status in _discovery_tick**

In `controller.py:_discovery_tick()`, after the branch update for existing sessions (line 249), add git status detection:

```python
if session.pid not in self._sessions:
    self._add_session(session)
else:
    self._sessions[session.pid].branch = detect_branch(cwd=session.cwd)
    self._sessions[session.pid].git_status = detect_git_status(cwd=session.cwd)
```

Also update `_add_session` (around line 281) to set initial git status:

```python
entry.branch = detect_branch(cwd=session.cwd)
entry.git_status = detect_git_status(cwd=session.cwd)
```

**FR-063 note**: Unattached (ghost) sessions are created via `_create_unattached_from_state`, NOT via `_add_session` or the discovery tick's live-PID loop. They never hit `detect_git_status` — their `git_status` stays at the `GitStatus.CLEAN` default from `_SessionEntry.__init__`. No explicit guard is needed because the code paths are structurally separate.

Add `detect_git_status` to the imports from `session`:

```python
from claude_dashboard.session import (
    ...,
    detect_git_status,
)
```

- [ ] **Step 5: Run make check**

Expected: May have test failures in test_agent_tracking.py if `_serialize_entry` references changed fields. Fix as needed.

- [ ] **Step 6: Commit**

---

## Task 5: UI — Flag Color from Git Status

**Files:**
- Modify: `claude_dashboard/ui/main_window.py:366-375` (_add_row flag creation), `claude_dashboard/ui/main_window.py:478-488` (_update_row flag update)

- [ ] **Step 1: Add _flag_color helper method**

In `MainWindow`, add a method that returns the flag color based on priority:

```python
def _flag_color(self, row: SessionRow) -> str | None:
    """Return flag dot color based on manual flag and git status priority.

    Returns None if no flag should be shown.
    """
    if row.flagged:
        return self._settings.color_flag_manual
    if row.git_status == GitStatus.UNSTAGED_CHANGES:
        return self._settings.color_flag_unstaged
    if row.git_status == GitStatus.STAGED_UNCOMMITTED:
        return self._settings.color_flag_staged
    if row.git_status == GitStatus.COMMITTED_NOT_PUSHED:
        return self._settings.color_flag_unpushed
    if row.git_status == GitStatus.PUSHED_NOT_MERGED:
        return self._settings.color_flag_unmerged
    return None
```

Add import at top:

```python
from claude_dashboard.config import GitStatus, IS_WINDOWS, StatusState
```

- [ ] **Step 2: Update _add_row to use _flag_color**

Replace the flag creation/packing logic (lines 366-375):

```python
flag_color = self._flag_color(row)
flag_label = tk.Label(
    row_frame,
    text=_FLAG_DOT_CHAR,
    bg=bg,
    fg=flag_color or self._settings.color_flag_manual,
    font=self._font_container,
    anchor=tk.E,
)
if flag_color is not None:
    flag_label.pack(side=tk.RIGHT, padx=(0, 2))
```

- [ ] **Step 3: Update _update_row to use _flag_color**

Replace the flag show/hide logic (lines 478-488):

```python
flag_color = self._flag_color(row_data)
if flag_color is not None:
    row["flag_label"].configure(fg=flag_color)
    if not row["flag_label"].winfo_manager():
        row["cwd_label"].pack_forget()
        row["flag_label"].pack(side=tk.RIGHT, padx=(0, 2))
        row["cwd_label"].pack(side=tk.LEFT, fill=tk.X, expand=True, padx=(2, 0))
else:
    if row["flag_label"].winfo_manager():
        row["flag_label"].pack_forget()
```

- [ ] **Step 4: Run make check**

Expected: PASS

- [ ] **Step 5: Commit**

---

## Task 6: Settings Window — 5 Flag Color Pickers

**Files:**
- Modify: `claude_dashboard/ui/settings_window.py:124-126` (flagged color field)

- [ ] **Step 1: Replace single flagged picker with 5 pickers**

Replace the single flagged color field (lines 124-126):

```python
row = self._add_color_field(
    frame, "\u2b24 Flagged", "color_flagged", settings.color_flagged, row
)
```

With:

```python
row = self._add_color_field(
    frame, "\u2b24 Manual flag", "color_flag_manual", settings.color_flag_manual, row
)
row = self._add_color_field(
    frame, "\u2b24 Unstaged changes", "color_flag_unstaged", settings.color_flag_unstaged, row
)
row = self._add_color_field(
    frame, "\u2b24 Staged uncommitted", "color_flag_staged", settings.color_flag_staged, row
)
row = self._add_color_field(
    frame, "\u2b24 Unpushed commits", "color_flag_unpushed", settings.color_flag_unpushed, row
)
row = self._add_color_field(
    frame, "\u2b24 Unmerged branch", "color_flag_unmerged", settings.color_flag_unmerged, row
)
```

- [ ] **Step 2: Run make check**

Expected: PASS

- [ ] **Step 3: Commit**

---

## Task 7: Controller Cleanup — Remove color_flagged References

**Files:**
- Modify: `claude_dashboard/controller.py` (any remaining `color_flagged` references)

- [ ] **Step 1: Search for remaining color_flagged references**

Run: `grep -rn "color_flagged" claude_dashboard/ tests/`

Fix any remaining references to use `color_flag_manual` or the appropriate new field.

- [ ] **Step 2: Update CLAUDE.md**

Add git status flags to the feature list and update the "Recent Enhancements" section.

- [ ] **Step 3: Run make check**

Expected: PASS — all 166+ tests pass

- [ ] **Step 4: Commit**

---

## Task 8: Performance Guard — Slow Git Detection

**Files:**
- Modify: `claude_dashboard/session.py` (detect_git_status)
- Modify: `claude_dashboard/controller.py` (_discovery_tick)

- [ ] **Step 1: Add timing and caching to detect_git_status**

Wrap the subprocess calls in `detect_git_status` with timing. If total detection time exceeds 500ms, log a warning. The controller caches the previous `git_status` on `_SessionEntry`, so a slow detection can return the cached value.

In `controller.py:_discovery_tick()`, add timing around git status detection:

```python
import time

start = time.monotonic()
new_git_status = detect_git_status(cwd=session.cwd)
elapsed = time.monotonic() - start
if elapsed > 0.5:
    logger.warning(
        "pid=%d git status took %.1fs, using cached value",
        session.pid,
        elapsed,
    )
else:
    self._sessions[session.pid].git_status = new_git_status
```

- [ ] **Step 2: Run make check**

Expected: PASS

- [ ] **Step 3: Final commit**
