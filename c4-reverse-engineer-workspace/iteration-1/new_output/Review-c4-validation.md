# Review: C4 Validation — Claude Dashboard

## Summary

| Artifact | Confirmed | Critical | Important | Minor |
|----------|-----------|----------|-----------|-------|
| c4-context.md | 12 | 0 | 0 | 0 |
| c4-container.md | 10 | 0 | 0 | 1 |
| c4-component.md | 14 | 0 | 0 | 0 |
| behavioral-spec.md | 31 | 0 | 0 | 2 |
| **Total** | **67** | **0** | **0** | **3** |

## Critical

No critical issues identified.

## Important

No important issues identified.

## Minor

### M0: `text_color` Settings field is dead

**Artifact:** behavioral-spec.md
**Section:** §6.6 Settings fields not exposed in the dialog
**Verdict:** Documented nuance; not a defect in the spec.

**Spec says:** "`text_color` (`settings.py:55`) is a persisted field but is not built into `SettingsWindow._build_ui` ... The `MainWindow._add_row` function computes per-row text color via `_contrast_text_for_bg(bg)` ..., ignoring the persisted `text_color` value."

**Code says:** `settings.py:55` defines `text_color: str = config.DEFAULT_TEXT_COLOR`. Grep for `text_color` in `claude_dashboard/` turns up only this one definition — no read sites in `ui/` (confirmed via `Grep text_color path=claude_dashboard`). `CLAUDE.md` "Design Decisions" section calls this out explicitly: "`text_color` setting field exists in the Settings dataclass for backward compat but is unused."

**Evidence:** Grep confirmation + direct read of `ui/main_window.py:113-124` showing auto-contrast logic; no other reader exists.

**Impact:** Anyone implementing a new UI client from this spec might expect `text_color` to be honored. The spec flags this explicitly under §6.6 and §10.11, so no extra edit is needed. Keeping the note as a Minor so reviewers know the spec confirmed the design-decision.

### M1: `DEFAULT_COLOR_BRANCH_MERGED` has no matching Settings field

**Artifact:** behavioral-spec.md
**Section:** §6.5 Settings modal
**Verdict:** MISSING

**Spec says:** Lists all color fields exposed in the settings dialog (12 fields) but does not explicitly document that `config.DEFAULT_COLOR_BRANCH_MERGED` (`config.py:87`) has no corresponding `Settings` dataclass field.

**Code says:** `config.py:87` defines `DEFAULT_COLOR_BRANCH_MERGED = "#ef4444"`. `ui/main_window.py:908-910` reads `config.DEFAULT_COLOR_BRANCH_MERGED` directly (as a module constant) in `_branch_color`, never through `self._settings`. `settings.py:15-59` has no `color_branch_merged` field.

**Evidence:** Full read of `settings.py`; grep for `color_branch_merged` in `claude_dashboard/` returns only the single-line constant in `config.py`.

**Impact:** The merged-branch color is not user-configurable via the Settings dialog even though the rest of the color scheme is. A reader expecting parity between `DEFAULT_COLOR_*` constants and `Settings` fields would miss this. Small — the main spec covers branch-merged display in §10.11 implicitly.

### M2: CLAUDE.md says "every 5 s" but default poll is 3 s

**Artifact:** behavioral-spec.md
**Section:** §3.2 Discovery tick
**Verdict:** Out-of-scope; project docs claim, not the generated spec.

**Spec says:** "`DEFAULT_POLL_INTERVAL_SECONDS = 3` (`config.py:40`)" — matches the code.

**Code says:** `config.py:40`: `DEFAULT_POLL_INTERVAL_SECONDS = 3`.

**Evidence:** Direct read of `config.py:40`. Note that `CLAUDE.md` ("Session discovery: Polls `~/.claude/sessions/*.json` every 5s") is stale — not a defect in the generated spec, but flagged here so the user knows to update the project's own CLAUDE.md at their discretion.

**Impact:** None on the generated spec. Flagged so the user can decide whether to refresh CLAUDE.md.

## Confirmed

### c4-context.md (12)

| Claim | Evidence method |
|-------|----------------|
| Port `17384` binding on `127.0.0.1` only | `__main__.py:19`, `hook_server.py:165` direct read |
| `SUBPROCESS_FLAGS = CREATE_NO_WINDOW` on Windows | `config.py:17` direct read |
| External callers (git, gh, code, gdbus, Win32, winreg, XDG) | `find_external_calls.py --group` output diffed against context table |
| Settings path branches by platform | `config.py:24-31` direct read |
| Session state file location | `config.py:37` direct read |
| psutil is used for PID + process-tree | `session.py:10,59-68`; `platform/windows.py:20`; `platform/linux.py:17` direct reads |
| Hook relay posts to `127.0.0.1:17384/hook` | `scripts/hook_relay.py:77-83` direct read |
| `install_hooks.py` deep-merges hook config | `scripts/install_hooks.py:50-78` direct read |
| pystray provides tray icon | `tray.py` direct read |
| statusline-data paths | `config.py:134-136` direct read |
| User interactions are Tkinter-event-driven | `ui/main_window.py` direct read |
| hook_relay runs in a Claude-Code-spawned subprocess, not in-process | `hooks-settings.json` registered commands + `scripts/hook_relay.py` relay logic |

### c4-container.md (10)

| Claim | Evidence method |
|-------|----------------|
| Tk main + two daemon threads (hook, tray) | `controller.py:289-321`, `hook_server.py:169`, `controller.py:301` direct reads |
| `_ReusableHTTPServer` has `allow_reuse_address/port = True` | `hook_server.py:132-136` direct read |
| `HookServer.stop` is idempotent | `hook_server.py:173-181` direct read |
| Discovery ticks via `root.after(tick_ms, ...)` | `controller.py:419-420` direct read |
| `_DEBOUNCE_MS = 300` | `controller.py:69` direct read + usage at 777, 783 |
| Agent permission debounce is 5000 ms | `controller.py:652` direct read |
| Startup sequence (argparse → logging → excepthook → single-instance → DPI) | `__main__.py:39-93` direct read |
| Settings store path + atomic write | `settings.py`, `file_utils.py:10-33` direct reads |
| State store written on every refresh | `controller.py:828` direct read |
| `_restart` uses `os.execv` with `-m claude_dashboard` args | `controller.py:1248-1262`, `118-125` direct reads |

### c4-component.md (14)

| Claim | Evidence method |
|-------|----------------|
| Event-to-state mapping table | `hook_server.py:20-55` direct read, every branch covered |
| `AskUserQuestion` → `AWAITING_INPUT` for both `PreToolUse` and `PermissionRequest` | `hook_server.py:42-50` direct read |
| Unknown events with `agent_id` → `WORKING` except `SubagentStop` | `hook_server.py:53-54` direct read |
| `_STATE_PRIORITY` values | `controller.py:51-57` direct read |
| `_ACTIONABLE_STATES` excludes `WORKING` | `controller.py:60-67` direct read |
| `_highest_priority_state` skips unattached and non-actionable | `controller.py:1297-1320` direct read |
| `effective_state` combines main + agents | `controller.py:156-166` direct read |
| Per-row data flow tick order | `controller.py:323-420` full direct read |
| Platform split — Windows branches | `find_platform_conditionals.py` output diffed against c4-component |
| Platform split — Linux branches | Same script output |
| Linux `find_window_for_session` is a no-op | `platform/base.py:71-73` direct read |
| Windows `match_window_by_cwd` uses `EnumWindows` + title match | `platform/windows.py:87-133` direct read |
| `detect_container_linux` walks process tree + TERM_PROGRAM fallback | `platform/linux.py:46-108` direct read |
| Linux VS Code foreground uses `code <cwd>` CLI | `platform/linux.py:242-284` direct read |

### behavioral-spec.md (31)

| Claim | Evidence method |
|-------|----------------|
| `--debug`, `--quiet`, `--ttl`, `--log-file` are the only CLI flags | `__main__.py:41-47` direct read (full enumeration) |
| Single-instance guard binds 127.0.0.1:17384 | `__main__.py:14-24` direct read |
| DPI awareness call is Windows-only | `__main__.py:81-87` + `find_platform_conditionals.py` output |
| `_discovery_tick` re-arms unconditionally after exception | `controller.py:413-420` direct read (re-arm is outside try) |
| Tick order: discover → ghost dead → first-tick replay → prune → fetch → trunk-cache clear → per-entry git → cost → usage → refresh | `controller.py:329-414` sequential read |
| Fetch tick interval = `max(60 // max(poll, 1), 1)` | `controller.py:241` direct read |
| Fetch loop dedupes by cwd | `controller.py:371-378` direct read |
| `detect_git_status > 0.5s` in discovery tick discards result | `controller.py:399-403` direct read |
| `_add_session` first-read > 0.5s keeps result and warns | `controller.py:452-460` direct read |
| Ghost synthetic PID starts at -1 and decrements | `controller.py:275-276, 512-513` direct reads |
| `_find_unattached_pid` + `_create_unattached_from_state` | `controller.py:497-530` direct read |
| `session.discover_sessions` reads `sessionId`, `pid`, `cwd`, `startedAt`, `entrypoint` | `session.py:30-56` direct read |
| `validate_pid` requires `"claude"` in proc name | `session.py:59-68` direct read |
| Auto-hide `entrypoint != "cli"` | `controller.py:477-479` direct read |
| Buffered hook states drain in `_add_session` | `controller.py:484-487` direct read |
| `_apply_hook_state` session-id-to-pid fallback via cwd | `controller.py:605-615` direct read |
| `IDLE → READY` rewrite is main-process only | `controller.py:672-673` + agent branch exit at 661 |
| `PostToolUseFailure` suppression rule | `controller.py:682-691` direct read |
| Agent permission debounce 5000 ms | `controller.py:648-655` direct read |
| Refresh debounce bypass only for PERMISSION_REQUIRED/AWAITING_INPUT | `controller.py:761-765` direct read |
| `_do_refresh_ui_deferred` re-reads entries | `controller.py:787-790` direct read |
| `_save_session_state` duplicate-cwd rule (hidden only if all hidden) | `controller.py:1155-1158` direct read |
| `git status --porcelain` parsing rules | `session.py:126-137` direct read |
| `detect_upstream` iterates `git remote` then `git symbolic-ref` | `session.py:179-212` direct read |
| `detect_merged` three strategies | `session.py:215-264` direct read |
| Every git subprocess has `timeout=2` (fetch is 10) | `session.py:116, 193, 232, 243, 257`; `controller.py:386` direct reads |
| `_open_pr` returncode-0 uses view, nonzero uses create | `controller.py:999-1026` direct read |
| `write_vscode_tasks_json` skips if file exists | `controller.py:180-186` direct read |
| `set_run_on_startup` Windows/Linux branches + unknown platform | `startup.py:40-47` direct read |
| `_launch_vscode` resolves `code` via `shutil.which` | `controller.py:967-985` direct read |
| `_ignored` uses `re.match`, not `re.search`, and catches `re.error` | `controller.py:433-441` direct read |

## Resolved during generation

These issues were caught by Phase 6 checks and fixed before delivery. They are listed here for audit traffic.

### R0: Evaluative phrasing — "best-effort `_tray_icon.stop()`"

- **Where:** behavioral-spec.md §2, startup step 6.
- **Caught by:** Phase 6 superlative/evaluation scrub (grep for `best-effort`).
- **Fix:** Replaced with mechanical description citing `controller.py:315-320` — "`self._tray_icon.stop()` inside a bare `try/except Exception: pass`."

### R1: Evaluative phrasing — "The relay never blocks the Claude Code CLI"

- **Where:** behavioral-spec.md §7.2 (hook relay behavior).
- **Caught by:** Superlative scrub (`never`) + evaluation scrub.
- **Fix:** Replaced with mechanical description — the relay uses `urlopen(req, timeout=2)`, so the blocking wait is capped at 2 s per fire; any exception is caught and only logged under `--debug`.

### R2: Evaluative phrasing — "load-bearing for correct operation"

- **Where:** behavioral-spec.md §10 intro.
- **Caught by:** Evaluation scrub (`correct`, `load-bearing`).
- **Fix:** Rewritten to "not obvious from a single-pass read of any one file. Each is referenced elsewhere in the codebase or by a code comment that explains the motivation."

### R3: Evaluative phrasing — "intentionally avoids stale data"

- **Where:** behavioral-spec.md §5.3.
- **Caught by:** Phase 6 scrub for judgment verbs (`intentionally`).
- **Fix:** Replaced with a direct quote of the source comment at `controller.py:788`.

### R4: Evaluative phrasing — "intentionally uses `-m claude_dashboard`"

- **Where:** behavioral-spec.md §10.18.
- **Caught by:** Evaluation scrub (`intentionally`).
- **Fix:** Replaced with mechanical description of `build_restart_args` return value + direct quote of the docstring.

### R5: Cross-check — every L3 component has a data-flow reference

- **Where:** c4-component.md component map (17 components listed).
- **Caught by:** Phase 6 dead-component check — traced every component name in c4-component.md through the spec.
- **Fix:** No dead components found. All 17 components (`entry`, `controller`, `session_mod`, `platform_base`, `platform_win`, `platform_lnx`, `settings_mod`, `file_utils`, `startup`, `config_mod`, `models_mod`, `main_window`, `settings_window`, `color_picker`, `cost_popup`, `tray_mod`, `hook_server_mod`) are referenced in at least one spec section with a file:line citation.

### R6: Cross-check — every L1 external system appears in the spec

- **Where:** c4-context.md external systems table (15 entries).
- **Caught by:** Phase 6 consistency check.
- **Fix:** All 15 entries traced to a spec section (§5-9). In particular, every subprocess call from `find_external_calls.py --group` (33 sites across `controller.py`, `session.py`, `platform/linux.py`) is represented by at least one mention.

### R7: Cross-check — platform conditionals symmetric

- **Where:** c4-component.md Platform split table.
- **Caught by:** Phase 6 check + `find_platform_conditionals.py` output.
- **Fix:** All 20 `IS_WINDOWS`/`IS_LINUX` usages inside `claude_dashboard/` are covered by the Platform split table. The 8 `platform.system()` hits in `scripts/detect_sessions.py` were ignored because `scripts/` is diagnostic tooling, not runtime app code per `CLAUDE.md`. The single `sys.platform` use in `startup.py:47` is referenced in §7.9 as the format argument to the "unsupported platform" warning.

## Phase 3 verification tally

Templates applied during Phase 3 (see `references/verification-templates.md`):

| Template | Claims verified |
|----------|-----------------|
| TIMING | 6 (poll interval, debounce, agent debounce, fetch tick interval, cost refresh modulus, 14-day cost popup range) |
| CONFIGURABLE | 4 (poll_interval_seconds, ignore_regex, every color field, row_width drag persistence) |
| SCOPE | 4 (`IDLE→READY` main-process-only, `PostToolUseFailure` guard scope, auto-hide on `entrypoint != "cli"`, debounce-bypass scope) |
| CROSS_MODULE | 4 (hook_server callbacks → controller via root.after; `_trunk_cache` lifetime; `_on_cost_click` cross-privacy attribute; `_apply_saved_state` vs buffered hook state ordering) |
| ENTRY_POINT | 3 (CLI args `--debug`/`--quiet`/`--ttl`/`--log-file`; POST `/hook` contract; hooks-settings.json registered event list) |
| PLATFORM | 13 (every row of the Platform split table in c4-component.md) |
| DEGRADATION | 25+ (every row of the §9 degradation table in behavioral-spec.md has a file:line citation for the catch/log behavior) |
| BINDING | 10 (row left-click/double-click/middle-click/right-click; title bar drag/shade/ghost-toggle/right-click; cost label drag/click/leave; window-level right-click) |

Low-risk claims (enum values, file path strings, static labels) accepted from Phase 1 direct reads without running templates.
