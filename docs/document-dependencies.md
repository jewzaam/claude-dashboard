# Document Dependencies

Changes to higher-order documents require reviewing and updating all lower-order documents. If a higher-order change does not produce a lower-order change, verify this is correct — don't assume.

## Dependency Order (highest to lowest)

```
docs/constitution.md                  ← Project principles and constraints
    ↓
specs/NNN-feature/spec.md             ← What we're building (user stories, requirements, success criteria)
    ↓
docs/spec-future.md                   ← Deferred user stories (P2/P3)
    ↓
specs/NNN-feature/research.md         ← Technical findings informing the spec
    ↓
specs/NNN-feature/plan.md             ← How we're building it (architecture, structure, dependencies)
    ↓
specs/NNN-feature/tasks.md            ← Execution sequence (task breakdown, phases, checkpoints)
    ↓
source code                           ← Implementation
```

### Feature Spec Directories

| Directory | Feature |
|-----------|---------|
| `specs/001-session-dashboard/` | US1-US5, FR-001–FR-028, SC-001–SC-006 |
| `specs/002-hide-apply-autostart/` | US6-US8, FR-029–FR-034 |
| `specs/003-agent-awareness/` | US9, FR-035–FR-042, SC-007 |

## Cascade Rules

| If this changes... | Review and update... |
|---------------------|----------------------|
| `docs/constitution.md` | All documents below it |
| `specs/NNN/spec.md` | Corresponding `plan.md`, `tasks.md`, source code |
| `docs/spec-future.md` | Nothing (deferred scope, no downstream impact yet) |
| `specs/NNN/research.md` | Corresponding `spec.md` (if findings change requirements), `plan.md`, `tasks.md` |
| `specs/NNN/plan.md` | Corresponding `tasks.md`, source code |
| `specs/NNN/tasks.md` | Source code (execution order only) |
| Source code (via review) | Trace change up to appropriate spec level, then cascade down |

## When Code Review Changes Behavior

1. Identify the change
2. Determine the highest-order document it affects
3. Update that document
4. Cascade downward through all dependent documents
5. If no downstream changes result, verify that's correct
