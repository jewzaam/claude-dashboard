# Review: C4 Validation — Claude Dashboard

## Summary

| Artifact | Confirmed | Critical | Important | Minor |
|----------|-----------|----------|-----------|-------|
| c4-context.md | 12 | 0 | 0 | 0 |
| c4-container.md | 8 | 0 | 0 | 0 |
| c4-component.md | 15 | 0 | 0 | 0 |
| behavioral-spec.md | 28 | 0 | 0 | 0 |
| **Total** | **63** | **0** | **0** | **0** |

No critical, important, or minor findings. All claims were verified through direct code reads during a comprehensive Phase 1 survey (the entire 7,262-line codebase was read) and Phase 3 verification using helper scripts and targeted file:line evidence.

## Critical

No critical issues identified.

## Important

No important issues identified.

## Minor

No minor issues identified.

## Confirmed

### c4-context.md

| # | Claim | Evidence method | Result |
|---|-------|----------------|--------|
| 1 | Hook server binds to 127.0.0.1:17384 | `hook_server.py:171` — `_ReusableHTTPServer(("127.0.0.1", self._requested_port), ...)` | Confirmed |
| 2 | Hook relay POSTs to localhost:17384 | `scripts/hook_relay.py:77-78` — `urllib.request.Request("http://127.0.0.1:17384/hook", ...)` | Confirmed |
| 3 | Session files at ~/.claude/sessions/*.json | `config.py:21` — `SESSIONS_DIR = CLAUDE_HOME / "sessions"` | Confirmed |
| 4 | Git commands with 2s timeout | `session.py:115` — `timeout=2` | Confirmed |
| 5 | Git fetch with 10s timeout | `controller.py:388` — `timeout=10` | Confirmed |
| 6 | gh CLI with 10s timeout | `controller.py:1008` — `timeout=10` | Confirmed |
| 7 | Relay 2s timeout | `scripts/hook_relay.py:83` — `urlopen(req, timeout=2)` | Confirmed |
| 8 | Settings path Windows | `config.py:25` — `_appdata / "claude-dashboard"` | Confirmed |
| 9 | Settings path Linux | `config.py:29` — `_config / "claude-dashboard"` | Confirmed |
| 10 | State file path | `config.py:37` — `STATE_FILE = CLAUDE_HOME / "claude-dashboard" / "session-state.json"` | Confirmed |
| 11 | Cost tracker path | `config.py:134-135` — `SESSION_TRACKER_DIR = STATUSLINE_DATA_DIR / "session-tracker"` | Confirmed |
| 12 | OAuth usage cache path | `config.py:136` — `OAUTH_USAGE_CACHE = STATUSLINE_DATA_DIR / "statusline-cache" / "oauth_usage.json"` | Confirmed |

### c4-container.md

| # | Claim | Evidence method | Result |
|---|-------|----------------|--------|
| 1 | Three threads: main, hook server, tray | `controller.py:289-302` — hook_server.start(), Thread(target=_run_tray, daemon=True).start() | Confirmed |
| 2 | Hook server is daemon thread | `hook_server.py:175` — `Thread(target=..., daemon=True)` | Confirmed |
| 3 | Tray is daemon thread | `controller.py:301` — `threading.Thread(target=_run_tray, daemon=True)` | Confirmed |
| 4 | Cross-thread via root.after(0, fn) | `controller.py:576-593` — all three callbacks use `self._root.after(0, ...)` | Confirmed |
| 5 | SO_REUSEADDR/SO_REUSEPORT | `hook_server.py:141-142` — `allow_reuse_address = True; allow_reuse_port = True` | Confirmed |
| 6 | Default poll interval 3s | `config.py:40` — `DEFAULT_POLL_INTERVAL_SECONDS = 3` | Confirmed |
| 7 | Discovery tick scheduling | `controller.py:419-420` — `self._root.after(tick_ms, self._discovery_tick)` | Confirmed |
| 8 | Cost read frequency | `controller.py:407` — `now - self._daily_cost_last_read >= self._settings.poll_interval_seconds * 10` | Confirmed |

### c4-component.md

| # | Claim | Evidence method | Result |
|---|-------|----------------|--------|
| 1 | Event mapping: PreToolUse + AskUserQuestion = AWAITING_INPUT | `hook_server.py:43-44` | Confirmed |
| 2 | Event mapping: PermissionRequest + AskUserQuestion = AWAITING_INPUT | `hook_server.py:48-49` | Confirmed |
| 3 | Event mapping: unmapped agent event = WORKING | `hook_server.py:52-54` | Confirmed |
| 4 | SubagentStop excluded from working default | `hook_server.py:53` — `event != "SubagentStop"` | Confirmed |
| 5 | IDLE->READY intercept | `controller.py:672-673` | Confirmed |
| 6 | PostToolUseFailure suppression | `controller.py:679-691` — checks `event == "PostToolUseFailure"` | Confirmed |
| 7 | State priority order | `controller.py:50-56` — PERMISSION_REQUIRED:0, AWAITING_INPUT:1, WORKING:2, READY:3, IDLE:4 | Confirmed |
| 8 | Effective state = highest priority across main + agents | `controller.py:156-165` | Confirmed |
| 9 | Agent permission debounce 5s | `controller.py:652` — `self._root.after(5000, ...)` | Confirmed |
| 10 | Git status priority: unstaged > staged > committed > pushed > clean | `session.py:140-176` — early return pattern | Confirmed |
| 11 | Three merge strategies | `session.py:226-258` — merge-base, cherry, diff | Confirmed |
| 12 | Trunk from remote/HEAD | `session.py:179-212` — `git symbolic-ref refs/remotes/<remote>/HEAD` | Confirmed |
| 13 | Tray icon skips WORKING | `controller.py:1313-1315` — `state not in _ACTIONABLE_STATES` where WORKING is not in _ACTIONABLE_STATES | Confirmed |
| 14 | tray.generate_icon_image reused by main_window | `main_window.py:14` — `from claude_dashboard.tray import generate_icon_image` | Confirmed |
| 15 | Platform font: Segoe UI / Noto Sans | `main_window.py:24-29` — conditional on IS_WINDOWS | Confirmed |

### behavioral-spec.md

| # | Claim | Evidence method | Result |
|---|-------|----------------|--------|
| 1 | Four CLI args: --debug, --quiet/-q, --ttl, --log-file | `__main__.py:41-46` — argparse definitions | Confirmed |
| 2 | RotatingFileHandler 2MB, 1 backup | `__main__.py:57-62` — `maxBytes=2 * 1024 * 1024, backupCount=1` | Confirmed |
| 3 | PIL logger forced to WARNING | `__main__.py:69` | Confirmed |
| 4 | Single-instance via port binding | `__main__.py:16-24` — socket bind test | Confirmed |
| 5 | DPI awareness Windows only | `__main__.py:81-87` — `if config.IS_WINDOWS:` | Confirmed |
| 6 | 12 callbacks in MainWindowCallbacks | `ui/main_window.py:63-81` — 12 Callable fields | Confirmed |
| 7 | Hook event buffering | `controller.py:618-619` — `self._pending_hook_states[session_id] = new_state` | Confirmed |
| 8 | Buffer replay on registration | `controller.py:484-487` — `buffered = self._pending_hook_states.pop(...)` | Confirmed |
| 9 | Auto-hide non-cli entrypoint | `controller.py:477-479` — `if session.entrypoint != "cli"` | Confirmed |
| 10 | Ghost synthetic PID decrement from -1 | `controller.py:276` — `self._next_synthetic_pid: int = -1` | Confirmed |
| 11 | Ghost CWD dedup check | `controller.py:535-538` — loop checking existing sessions for same CWD | Confirmed |
| 12 | Ghost prune on missing directory | `controller.py:357-365` — `Path(entry.session.cwd).is_dir()` | Confirmed |
| 13 | Fetch tick interval formula | `controller.py:241` — `max(60 // max(self._settings.poll_interval_seconds, 1), 1)` | Confirmed |
| 14 | Git status >0.5s discard | `controller.py:402-403` — `if elapsed <= 0.5: entry.git_status = new_git_status` | Confirmed |
| 15 | Debounce 300ms | `controller.py:68` — `_DEBOUNCE_MS = 300` | Confirmed |
| 16 | Debounce bypass for permission/awaiting | `controller.py:762-763` — `StatusState.PERMISSION_REQUIRED, StatusState.AWAITING_INPUT` | Confirmed |
| 17 | Click delay 200ms | `main_window.py:56` — `_CLICK_DELAY_MS = 200` | Confirmed |
| 18 | Drag threshold 5px | `main_window.py:733` — `_DRAG_THRESHOLD = 5` | Confirmed |
| 19 | Minimum resize width 150px | `main_window.py:767` — `_MIN_WIDTH = 150` | Confirmed |
| 20 | CWD fallback matching | `controller.py:609-615` — `if pid is None and cwd:` | Confirmed |
| 21 | text_color unused, auto-contrast | `main_window.py:112-123` — `_contrast_text_for_bg`, Settings.text_color not referenced in rendering | Confirmed |
| 22 | Restart via os.execv | `controller.py:1259-1262` — `os.execv(sys.executable, args)` | Confirmed |
| 23 | -m claude_dashboard to avoid shadow | `controller.py:123-124` — comment explains platform shadow | Confirmed |
| 24 | tasks.json written only if not exists | `controller.py:178-182` — `if tasks_path.exists(): return` | Confirmed |
| 25 | Two tasks in claude-dev group | `config.py:97-129` — "claude" and "bash" tasks with same group | Confirmed |
| 26 | Agent auto-wake suppression (no clear on UserPromptSubmit) | `controller.py:663-667` — comment explains, no agents.clear() call | Confirmed |
| 27 | Shaded bar excludes READY | `main_window.py:412-413` — `color.lower() == self._settings.color_ready.lower()` | Confirmed |
| 28 | Duplicate-CWD hidden logic | `controller.py:1156-1158` — `if not entry.hidden: state[cwd]["hidden"] = False` | Confirmed |

## Verification Methods Used

- **Direct file reads**: All 20 source files in `claude_dashboard/` and 1 script (`hook_relay.py`) were read in full during Phase 1.
- **Helper script: find_platform_conditionals.py**: 29 platform conditionals found. All documented in c4-component.md Platform-Conditional Behavior Summary.
- **Helper script: find_external_calls.py**: 41 external call sites found (33 subprocess, 5 urllib, 2 file open, 1 socket). All external systems documented in c4-context.md External Systems Summary and behavioral-spec.md sections 7 and 9.
- **Line-count verification**: 7,262 source lines confirmed via count_source_lines.py. Medium tier confirmed.
- **Superlative scrub**: All four artifacts scanned for "most", "best", "optimal", "always", "never", "correct", "successful", "reliable", "safe". No unquoted superlatives found.
