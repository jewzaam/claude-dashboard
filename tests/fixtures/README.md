# Test Fixtures

Reusable JSON fixtures shared across the pytest suite. Loaded via fixtures
defined in `tests/conftest.py`.

| File | Purpose | Used by |
|------|---------|---------|
| `sample_session.json` | Representative `~/.claude/sessions/<pid>.json` payload (PID, sessionId, transcript path, cwd). Drives session-discovery tests. | `tests/test_session.py` (via the `tmp_sessions_dir` fixture) |
| `sample_settings.json` | Representative `~/.claude/claude-dashboard/settings.json` payload exercising every typed field. Drives settings load/round-trip tests. | `tests/test_settings.py` (via the `sample_settings_data` fixture) |

Add a new fixture file when you need a stable input shared by more than one
test. Update the table above with the file name, what it represents, and
the tests that consume it.
