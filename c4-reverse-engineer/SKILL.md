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

## Workflow — 6 Phases

### Phase 1: Codebase Survey (direct reads)

Read the project directly to establish context and vocabulary before dispatching subagents.

1. Read README, CLAUDE.md, or equivalent project docs
2. Glob all source files, group by directory. Count source files (excluding tests, vendored
   deps, generated code) to select the verification tier:
   - **<30 files**: direct reads for everything
   - **30–100 files**: hybrid (direct reads for key files, subagents for the rest)
   - **100+ files**: subagent-heavy with verification subagents
3. Identify and read entry points — enumerate every CLI arg, HTTP endpoint, event handler
4. Read vocabulary files: enums, constants, settings/config dataclasses, shared types
5. Identify integration points: find file(s) with the most imports from other project modules.
   Read them in full. Common forms: orchestrator/controller, request handler, CLI entry point,
   application factory. There may be more than one

### Phase 2: Parallel Exploration (3 subagents)

Dispatch 3 subagents in parallel. Partition by concern, not by file:

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

**Priority:** Verify all high-risk claims (timing, configurability, scope, cross-module
overrides, entry point inventory). Sample medium-risk (interaction bindings, platform branches,
event suppression, degradation paths). Skip low-risk (enum values, file paths, static labels).

### Phase 4: C4 Diagram Generation

Write diagrams from verified claims only:
- **L1 Context**: The system as a single box. Every external actor/system it communicates with.
  Include a trust boundary note and an external systems summary table
- **L2 Container**: Internal runtime boundaries — processes, threads, scripts, persistent
  stores. Include the threading/concurrency model
- **L3 Component**: Behavioral decomposition per container. Include event-to-state mapping
  tables, data flow summaries

### Phase 5: Behavioral Spec Generation

Write the spec from verified claims. Structure:

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

### Phase 6: Cross-Artifact Consistency Check & Review

Run these consistency checks:
- Every external system in L1 appears in the behavioral spec
- Every container in L2 maps to components in L3
- Every state transition has a documented trigger
- Every timing/frequency claim is consistent across all artifacts
- Every "configurable" claim uses the same term (settings field vs constant)
- Every cross-module override in L3 is reflected in the behavioral spec
- Platform-conditional behaviors documented for both platforms

**Fix-before-delivery:** Fix Critical and Important findings in the artifacts (return to Phases
4–5), then re-run checks. Minor findings are documented but not fixed.

Produce `Review-c4-validation.md` using the format in `references/review-format.md`. The
review documents the post-fix state — it's a quality record, not a punch list of known defects.

## What This Skill Is NOT

- Not a code review — no quality judgments, no dead code flagging
- Not a refactoring plan — no code changes proposed
- Not a test plan
- Not documentation of HOW code works — only WHAT it does

## Reference Files

Read these during the phases that need them:

| File | When to read | Content |
|------|-------------|---------|
| `references/subagent-prompts.md` | Phase 2, before dispatching subagents | 7 prompt instructions + framework calibration table |
| `references/verification-templates.md` | Phase 3, before verification | 8 templates, trigger-word heuristics, priority tiers |
| `references/review-format.md` | Phase 6, before writing review | Severity levels, finding format, document layout |
| `references/failure-modes.md` | When diagnosing a verification failure | 9 failure modes with root causes and detection methods |
