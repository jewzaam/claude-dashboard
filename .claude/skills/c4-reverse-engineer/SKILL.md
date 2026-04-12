---
name: c4-reverse-engineer
description: >
  Reverse-engineer C4 architecture diagrams (Context, Container, Component) and a behavioral
  specification from a codebase. Produces documentation of WHAT the system does — not how it's
  coded. Use this skill whenever someone asks to document system behavior, produce architecture
  diagrams, reverse-engineer a codebase, understand what a project does, create a C4 model, or
  write a behavioral spec. Also use it when preparing for a refactor, client/server split, or
  rewrite where understanding current behavior is a prerequisite. Even if the user doesn't
  mention "C4" specifically — if they want architecture docs or behavioral documentation from
  code, this is the skill.
---

# C4 Reverse-Engineering

Produce C4 architecture diagrams and a behavioral specification from a codebase. No code changes.

**Core principle:** Subagents tell you WHAT to look at. Direct reads tell you what's TRUE.

Subagent exploration produces plausible-looking specs with subtle errors — wrong refresh rates,
wrong event binding targets, configurable-vs-hardcoded confusion. The skill uses subagents for
the survey (shape of the system) then verifies every specific claim with direct reads before
writing it into a deliverable.

## Output

All output goes to `docs/c4/` in the project directory:

| File | Content |
|------|---------|
| `c4-context.md` | L1 — System boundary + all external actors |
| `c4-container.md` | L2 — Internal runtime containers (processes, threads, services) |
| `c4-component.md` | L3 — Component decomposition per container |
| `behavioral-spec.md` | Full behavioral specification |
| `Review-c4-validation.md` | Quality record documenting what was verified |

Diagrams use Mermaid C4 notation (`C4Context`, `C4Container`, `C4Component`). Include ASCII
fallback for the primary data flow diagram. Note in output: "Mermaid C4 may not render in all
viewers."

## Workflow — 7 Phases

### Phase 0: Intake

Before touching code, establish *why* the user wants this spec and *which* artifacts they
need. Different purposes reweight what "important" means and what depth each section
deserves. Running Phase 0 takes ~30 seconds and prevents you from writing the wrong spec
confidently.

Ask the user two questions via `AskUserQuestion` in a single call:

1. **Purpose** — pick one:

   | Purpose | Emphasis |
   |---------|----------|
   | Client/server split or protocol rewrite | Wire formats, state machines, persistence, external system contracts |
   | Refactor preparation | Module boundaries, coupling, cross-module overrides, integration points |
   | Developer onboarding | Vocabulary (enums/constants), entry points, mental model, happy-path data flow |
   | Security/architecture audit | Trust boundaries, external calls, failure modes, degradation paths, authz checks |
   | General architecture documentation | Balanced across all of the above |

2. **Artifacts** — pick a set:

   | Set | Files produced |
   |-----|----------------|
   | Full set (default) | c4-context + c4-container + c4-component + behavioral-spec + review |
   | Diagrams only | c4-context + c4-container + c4-component + review |
   | Spec only | behavioral-spec + review |
   | One level | whichever of L1/L2/L3 the user names + review |

Record both answers and carry them forward:
- The **purpose** biases Phase 5 emphasis: the spec's content is the same, but the depth
  of each section matches what the purpose needs. An onboarding spec spends more words on
  vocabulary and entry points; an audit spec spends more on degradation and trust
  boundaries. All 10 floor sections remain present.
- The **artifact set** determines which Phase 4/5 files you produce. The review artifact
  is always produced — it's the quality record.

**Skip Phase 0 only if the user has already stated both answers in the conversation.**
In that case, acknowledge what you heard and proceed.

### Phase 1: Codebase Survey (direct reads)

Read the project directly to establish context and vocabulary before dispatching subagents.

1. Read README, CLAUDE.md, or equivalent project docs
2. Measure project size to pick the tier. Use `scripts/count_source_lines.py` (see Reference
   Files) — it counts non-test, non-generated, non-vendored lines in the project's primary
   languages. Without the script, estimate by globbing source files and summing `wc -l`
   across them, excluding tests/vendored/generated trees.

   | Tier | Source lines | Strategy |
   |------|--------------|----------|
   | **Small** | < 3,000 | Direct reads for everything. Skip Phase 2 — the Phase 1 survey *is* the exploration. |
   | **Medium** | 3,000 – 15,000 | Hybrid: direct reads for entry points and vocabulary, 2 subagents in Phase 2 for the remainder. |
   | **Large** | > 15,000 | Subagent-heavy: 3 subagents in Phase 2, direct reads only for sampled verification in Phase 3. |

   File count is a poor proxy — a repo with 200 JSON fixtures is not large, and a repo with
   one 3,000-line controller is not small. Count lines, not files. Polyglot repos: sum lines
   across all primary languages (e.g., Python backend + TypeScript frontend). See
   `references/subagent-prompts.md` for polyglot partition guidance.
3. Identify and read entry points — enumerate every CLI arg, HTTP endpoint, event handler
4. Read vocabulary files: enums, constants, settings/config dataclasses, shared types
5. Identify integration points: find file(s) with the most imports from other project modules.
   Read them in full. Common forms: orchestrator/controller, request handler, CLI entry point,
   application factory. There may be more than one
6. For non-obvious behaviors discovered in steps 3–5, check `git log --follow <file>` for the
   commit that introduced them. Commit messages often explain *why* a guard, timeout, or
   fallback exists — context the code alone doesn't reveal.

### Phase 2: Parallel Exploration (tier-scaled)

**Small tier:** Skip Phase 2 entirely. The Phase 1 survey covered the codebase directly; go
to Phase 3 and verify the claims you already have.

**Medium tier:** Dispatch 2 subagents in parallel. Merge A and B into a single
"Behavior + Interfaces" agent; keep C.

**Large tier:** Dispatch 3 subagents in parallel.

Partition by concern, not by file:

| Agent | Scope |
|-------|-------|
| **A — Internal behavior** | Modules, state management, events, data flows, business logic |
| **B — External-facing interfaces** | UI, API, CLI, public API surface, user-visible interactions |
| **C — Dependencies & infrastructure** | External system calls, file I/O, subprocesses, config loading, platform adapters |

How the partition maps to architectures:
- Desktop app: A = controller/model, B = UI/interaction layer, C = scripts/OS integration
- REST API: A = domain logic/services, B = HTTP endpoints/middleware, C = DB/cache/queues
- CLI tool: A = command logic, B = arg parsing/output formatting, C = file system/network
- Library: A = core algorithms, B = public API surface, C = optional dependencies

For polyglot repos, partition by language/stack instead of by concern — one subagent per
major stack (e.g., Python backend, TypeScript frontend, Rust extension). Each subagent still
receives the 7 prompt instructions, specialized for its stack via the framework calibration
table.

Each subagent prompt must include all 7 instructions from `references/subagent-prompts.md`.
These instructions prevent the 9 known failure modes — omitting them produces specs that look
right but have hidden errors.

### Phase 3: Structured Verification (targeted reads + templates)

For each behavioral claim from the subagent reports, verify it against the code using
structured templates. Read `references/verification-templates.md` for the full template
format, trigger-word heuristics, and priority tiers.

**Key concept:** Each template produces a claim AND its evidence simultaneously. Don't extract
claims from subagent prose then verify separately — that rephrasing step introduces errors.
Instead, read the code and fill in the template directly.

**Priority:** Verify every high-risk claim (timing, configurability, scope, cross-module
overrides, entry point inventory). Sample medium-risk (interaction bindings, platform branches,
event suppression, degradation paths). Skip low-risk (enum values, file paths, static labels).

For mechanical work in this phase, use the helper scripts in `scripts/`:
- `find_platform_conditionals.py` — enumerates every `sys.platform`, `platform.system()`,
  `os.name`, and project-specific platform constant. Feeds PLATFORM claims.
- `find_external_calls.py` — lists every `subprocess.`, `urllib`, `requests`, `socket`,
  `open(` call. Feeds DEGRADATION claims and the L1 external systems list.

These scripts give you a deterministic inventory to diff subagent reports against — any
item in the script output that isn't in a subagent report is a gap to close.

### Phase 4: C4 Diagram Generation

Write diagrams from verified claims only. Respect the artifact set from Phase 0 — skip
any level the user didn't request:

- **L1 Context**: The system as a single box. Every external actor/system it communicates with.
  Include a trust boundary note and an external systems summary table
- **L2 Container**: Internal runtime boundaries — processes, threads, scripts, persistent
  stores. Include the threading/concurrency model
- **L3 Component**: Behavioral decomposition per container. Include event-to-state mapping
  tables, data flow summaries

**Mermaid C4 character pitfalls.** The Mermaid C4 plugin (tested on 11.x) is stricter
than standard Mermaid — labels fail to parse with characters that work fine elsewhere.
Keep labels ASCII-plain:

- **No inner parentheses** inside quoted labels. `System_Ext(id, "Foo (Bar)", ...)` breaks
  the parser; use `"Foo - Bar"` or move "(Bar)" into the description string after a dash.
- **No backslashes**. `"HKCU\\...\\Run"` is treated as an escape. Use forward slashes or
  prose: `"HKCU Run key"`.
- **No angle brackets** `< >` in labels — they can be interpreted as HTML tags when the
  rendered SVG is embedded. Use words: `"code cwd"` not `"code <cwd>"`.
- **Avoid pipe character** `|` inside labels — it's a table separator in the surrounding
  markdown and confuses some renderers.
- **Em dashes and smart quotes are fine in titles and descriptions** but some
  md2html/mermaid pipelines mangle them. When in doubt, ASCII.

After writing each diagram, mentally re-scan the labels for these characters. A diagram
that displays "Syntax error in text" is worse than a plain one — the reader can't even
see the structure.

### Phase 5: Behavioral Spec Generation

Skip this phase if the Phase 0 artifact set doesn't include the behavioral spec.

Write the spec from verified claims. Weight each section's depth by the Phase 0 purpose
(onboarding → vocabulary-heavy; audit → degradation-heavy; client rewrite → protocol-heavy).
The structure below is the **floor**, not the ceiling. Every generated spec includes
these 10 sections. Add sections when the system has substantial behavior outside them
(e.g., agent tracking, ghost sessions, state persistence as standalone sections rather
than squeezed into §3 or §8):

1. Purpose and operational overview
2. Startup / initialization sequence
3. Core lifecycle (discovery, polling, event processing)
4. State machines and transitions (with guards)
5. Event flows and processing rules
6. User interactions (every click/action and its effect)
7. Integration behaviors (each external system)
8. Persistence and recovery
9. Degradation behavior — for each external dependency, what happens when it fails
10. Behavioral nuances — the non-obvious behaviors that would surprise someone reading the code

The spec documents what the code DOES, including when it does something surprising (like
ignoring a configurable setting). It does not judge whether the code SHOULD behave differently.

**Describe, don't evaluate.** Behavioral descriptions are mechanical: what the code runs,
what it returns, what it calls next. Never characterize code as "correct", "reliable",
"optimal", "most", "best", or "successful". Those are judgments the skill cannot make —
they rot fast and mislead worst.

| Bad (evaluative)                            | Good (mechanical)                                                                                                    |
|---------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| "Uses `code <cwd>` — most reliable on X"    | "Runs `code <cwd>`. VS Code activates the window whose root folder is `<cwd>`, or opens a new window if none match." |
| "Foregrounds the correct window"            | "Calls `SetForegroundWindow(hwnd)` on the window whose title contains the CWD basename."                             |
| "Safely handles missing files"              | "Catches `FileNotFoundError` and returns an empty dict."                                                             |
| "Always validates input"                    | "Every caller at [file:line, file:line] routes through `_validate()` before write."                                  |

If the source explicitly uses an evaluative word (e.g., a comment saying "// fast path"),
quote it: "the code labels this 'fast path' (file:line)". Don't launder it into your own
voice.

### Phase 6: Cross-Artifact Consistency Check & Review

Run these consistency checks:
- Every external system in L1 appears in the behavioral spec
- Every container in L2 maps to components in L3
- Every component in L3 appears in the data flow summary or behavioral spec — a listed
  component nothing references is either dead code (flag it for the user) or missing
  coverage (trace how it's used and add the missing edge)
- Every state transition has a documented trigger
- Every timing/frequency claim is consistent across all artifacts
- Every "configurable" claim uses the same term (settings field vs constant)
- Every cross-module override in L3 is reflected in the behavioral spec
- Platform-conditional behaviors documented for both platforms
- **Superlative scrub**: grep the artifacts for "most", "best", "optimal", "always",
  "never", "correct", "successful", "reliable", "safe". Each hit is either quoted from
  source (keep, cite file:line) or the model editorializing (rewrite mechanically — see
  Phase 5 Describe-don't-evaluate table). No superlatives survive into the final output
  unquoted.

**Fix-before-delivery:** Fix Critical and Important findings in the artifacts (return to Phases
4–5), then re-run checks. Minor findings are documented but not fixed.

Produce `Review-c4-validation.md` using the format in `references/review-format.md`. The
review documents the post-fix state as a quality record. It MAY also include an optional
"Resolved during generation" section listing issues caught by Phase 6 and fixed before
delivery — this preserves the audit trail without cluttering the main findings list.

## What This Skill Is NOT

- Not a code review — no quality judgments, no hunting for unused functions
- Not a refactoring plan — no code changes proposed
- Not a test plan
- Not documentation of HOW code works — only WHAT it does

The Phase 6 "component in L3 isn't referenced anywhere" check is a doc-consistency check,
not dead-code review: the concern is that the spec listed a component and then never used
it. If the code itself has dead functions, that's for a separate review — this skill stays
mute on it.

## Reference Files

Read these during the phases that need them:

| File | When to read | Content |
|------|-------------|---------|
| `references/subagent-prompts.md` | Phase 2, before dispatching subagents | 7 prompt instructions + framework calibration table |
| `references/verification-templates.md` | Phase 3, before verification | 8 templates, trigger-word heuristics, priority tiers |
| `references/review-format.md` | Phase 6, before writing review | Severity levels, finding format, document layout |
| `references/failure-modes.md` | When diagnosing a verification failure | 9 failure modes with root causes and detection methods |
| `references/example-output.md` | Phase 5, for phrasing calibration | Toy project with a mechanical-phrasing §9 Degradation excerpt and a side-by-side bad/good comparison |

## Scripts

Deterministic helpers for mechanical work. Run them from the project root; each prints
results to stdout.

| Script | When to run | What it does |
|--------|-------------|--------------|
| `scripts/count_source_lines.py` | Phase 1, step 2 | Counts non-test, non-generated source lines per language and recommends a tier. |
| `scripts/find_platform_conditionals.py` | Phase 3, PLATFORM claims | Enumerates every `sys.platform`, `platform.system()`, `os.name`, `IS_*` constant, Go build tag, Rust `cfg(target_os)`, and C/C++ platform ifdef. |
| `scripts/find_external_calls.py` | Phase 3, DEGRADATION claims and L1 external systems | Enumerates subprocess, network, and filesystem calls across Python/Node/Go/Rust/Java/C#. Use `--group` for a category-grouped report. |

Invoke with `python <skill-path>/scripts/<name>.py [project-root]`. Defaults to the
current working directory. All three exclude tests and vendored code by default; pass
`--include-tests` on the finder scripts if you need the full picture.
