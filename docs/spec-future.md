# Claude Dashboard — Future User Stories

> Deferred from MVP spec. These require solving input injection and should not be confused with current iteration scope.

## User Story — Permission Handling from Dashboard (Priority: P2)

As a user, I want to expand a session row to see the pending permission request and approve or deny it directly from the dashboard, so I don't have to switch to the terminal.

**Why this priority**: High-value interaction that reduces context switching. Deferred from MVP because it requires solving the input injection problem.

**Independent Test**: A Claude session prompts for tool permission. The dashboard row expands to show the command. User clicks Approve — the session continues.

**Acceptance Scenarios**:

1. **Given** a session has a pending permission prompt, **When** I click to expand the row, **Then** I see the command being requested.
2. **Given** I click "Approve" on the expanded permission, **When** the approval is sent, **Then** the Claude session proceeds and the row collapses back.
3. **Given** I click "Deny," **When** the denial is sent, **Then** the session handles it and the row updates accordingly.

---

## User Story — Input Injection (Priority: P3)

As a user, when Claude is waiting for my input, I want to see the last response and type my reply directly in the dashboard, so I can interact without switching windows.

**Why this priority**: Extends the dashboard from monitoring to full interaction. Complex — requires reliable input channel to Claude processes.

**Independent Test**: Claude is waiting for input. Dashboard shows the last response. User types a reply in the dashboard — it appears in the Claude session.

**Acceptance Scenarios**:

1. **Given** a session is awaiting input, **When** I expand the row, **Then** I see the last response from Claude.
2. **Given** I type a reply and press Send, **When** the input is delivered, **Then** the Claude session processes it as if typed directly.

---

## User Story — Voice Input per Session (Priority: P3)

As a user, I want a record/send button per session that captures voice, transcribes it via Whisper, and sends the transcript as input to that Claude session.

**Why this priority**: Builds on input injection (P3) and existing voice skill (`~/source/claude-skill-voice`). Requires input injection to work first.

**Independent Test**: Click Record on a session row, speak, click Send. The transcript appears as input in the Claude session.

**Acceptance Scenarios**:

1. **Given** a session is awaiting input, **When** I click Record, speak, and click Send, **Then** the transcribed text is delivered as input to that session.

---

## Screen Real Estate

When the dashboard grows to support inline interaction (permission handling, input injection, voice), row expansion will consume significant vertical space. If the number of sessions is high, this will exceed available screen height. Scrollable container or collapsible sections will be needed.
