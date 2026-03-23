# Citation Audit Report

Audited 9 of 52 citations (sample-based verification).

| Citation | URL Domain | Status | Notes |
|----------|-----------|--------|-------|
| [1] CLI reference | code.claude.com | VERIFIED | `--continue`, `--resume`, `--fork-session` all confirmed |
| [2] Agent Teams | code.claude.com | VERIFIED | Experimental flag, tmux/iTerm2 backends confirmed |
| [9] claude-code-ui | github.com | UNVERIFIED | WebFetch blocked by permission; found via WebSearch |
| [16] eyes-on-claude-code | github.com | UNVERIFIED | WebFetch blocked by permission; found via WebSearch |
| [18] cmux | cmux.com / github.com | UNVERIFIED | WebFetch blocked by permission; found via WebSearch |
| [21] Termoil | github.com | UNVERIFIED | WebFetch blocked by permission; found via WebSearch |
| [25] claude-squad | github.com | UNVERIFIED | WebFetch blocked by permission; found via WebSearch |
| [27] Composio AO | github.com | UNVERIFIED | WebFetch blocked by permission; found via WebSearch |
| [30] dmux | github.com / dmux.ai | UNVERIFIED | WebFetch blocked by permission; found via WebSearch |

## Notes

- 2/9 fully verified (Anthropic docs)
- 7/9 unverified due to WebFetch permission restrictions on GitHub/third-party domains
- All 9 URLs were discovered via WebSearch in-session (not fabricated)
- The WebSearch tool returns page titles and content snippets, which is how claims were sourced
- To fully verify GitHub citations, WebFetch permissions for github.com would need to be granted

**Status: RESOLVED** — consistency review issue (citation [7] unused) was fixed by adding a reference in official-claude-tooling.md.
