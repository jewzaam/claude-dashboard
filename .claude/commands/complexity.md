# Complexity Reduction Skill

Interactive, human-in-the-loop cyclomatic complexity reduction. The model identifies candidates, the human directs simplification.

## Prerequisites

- `make complexity` target must exist (xenon)
- `make test` must pass before starting

## Process

### 1. Check repo state

Run `git status`. If there are uncommitted tracked changes, ask the user (via AskUserQuestion) whether to continue anyway. Default option: "No, stop". If the user says stop, halt.

### 2. Run `make complexity`

Parse xenon output. Each ERROR line is a violation. Present them sorted by grade (D first, then C), with file, function name, and grade.

Recommend the highest-grade (worst) function as the starting candidate.

Wait for the user to confirm or pick a different one.

### 3. Analyze the chosen function

Read the function. For each decision point (if, elif, for, while, except, and, or), state:

- The line number
- What it decides
- Whether this decision still needs to exist — and if you can't tell, say so

Present the list. Do NOT use AskUserQuestion here — just state the analysis and wait for the user to respond with their thoughts on what to simplify.

### 4. Apply feedback

The user will provide direction: dead code to remove, logic to collapse, decisions that are unnecessary, areas that are hard to understand. Apply their feedback as edits.

Run `make test` to verify nothing broke. Run `make complexity` to get the new score.

### 5. Present updated results

Show the new xenon output. If the function dropped a grade, note it. If violations remain, recommend the next candidate.

Do NOT ask "would you like to continue?" as a text question. Use AskUserQuestion with options:

- "Continue with [recommended next function]"
- "Pick a different function"
- "Done for now"

### 6. Loop

Repeat from step 3 with the next function until the user says done or all violations are resolved.

## Rules

- Never extract functions just to lower a score. The goal is removing decisions that don't need to exist.
- Never reformat code (flip if/else, rename operators) to shave points.
- If a function's complexity is genuine — every decision is necessary — say so. Not everything can be simplified.
- The human directs simplification. The model analyzes and executes.
- Do not run `make complexity` inside `make check`. Complexity is a separate, interactive concern.
