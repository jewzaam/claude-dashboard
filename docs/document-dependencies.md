# Document Dependencies

Changes to higher-order documents require reviewing and updating all lower-order documents. If a higher-order change does not produce a lower-order change, verify this is correct — don't assume.

## Dependency Order (highest to lowest)

```
constitution.md          ← Project principles and constraints
    ↓
spec.md                  ← What we're building (user stories, requirements, success criteria)
    ↓
spec-future.md           ← Deferred user stories (P2/P3)
    ↓
research-session-detection.md  ← Technical findings informing the spec
    ↓
plan.md                  ← How we're building it (architecture, structure, dependencies)
    ↓
tasks.md                 ← Execution sequence (task breakdown, phases, checkpoints)
    ↓
source code              ← Implementation
```

## Cascade Rules

| If this changes... | Review and update... |
|---------------------|----------------------|
| `constitution.md` | All documents below it |
| `spec.md` | `plan.md`, `tasks.md`, source code |
| `spec-future.md` | Nothing (deferred scope, no downstream impact yet) |
| `research-session-detection.md` | `spec.md` (if findings change requirements), `plan.md`, `tasks.md` |
| `plan.md` | `tasks.md`, source code |
| `tasks.md` | Source code (execution order only) |
| Source code (via review) | Trace change up to appropriate spec level, then cascade down |

## When Code Review Changes Behavior

1. Identify the change
2. Determine the highest-order document it affects
3. Update that document
4. Cascade downward through all dependent documents
5. If no downstream changes result, verify that's correct
