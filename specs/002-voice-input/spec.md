# Feature Specification: Voice Input for Dashboard Sessions

**Feature Branch**: `002-voice-input`
**Created**: 2026-03-24
**Status**: Draft
**Input**: User description: "Microphone button in dashboard rows to record voice, transcribe, and send as prompt to that Claude instance. Only visible when session state is ready, idle, or unknown."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Voice Prompt to Idle Session (Priority: P1)

A user sees a Claude session sitting idle in the dashboard. Rather than switching to that terminal and typing, they click a microphone button on that session's row. A recording interface appears. They speak their prompt, finish recording, and the transcribed text is sent as input to that specific Claude session. The session begins working on the spoken prompt.

**Why this priority**: This is the core value proposition — hands-free prompt delivery to a specific session without leaving the dashboard.

**Independent Test**: Can be fully tested by clicking the microphone on an idle session row, speaking a prompt, and verifying the transcribed text arrives as input to that Claude instance.

**Acceptance Scenarios**:

1. **Given** a session row in "idle" state, **When** the user clicks the microphone button, **Then** a voice recording interface appears associated with that session.
2. **Given** an active voice recording, **When** the user finishes recording (presses Done or Enter), **Then** the audio is transcribed and the resulting text is sent as a prompt to the corresponding Claude session.
3. **Given** an active voice recording, **When** the user cancels (presses Cancel or Escape), **Then** no text is sent and the recording interface closes.

---

### User Story 2 - Voice Prompt to Ready Session (Priority: P1)

A user sees a session marked as "ready" (awaiting input after the user previously marked it). They click the microphone button, speak their instructions, and the transcription is delivered as the session's next prompt.

**Why this priority**: "Ready" is a primary target state — the session is explicitly waiting for user input, making voice input a natural fit.

**Independent Test**: Can be tested by marking a session as ready, clicking the microphone, speaking, and verifying the prompt is delivered.

**Acceptance Scenarios**:

1. **Given** a session row in "ready" state, **When** the user clicks the microphone button, **Then** a voice recording interface appears.
2. **Given** a completed transcription for a ready session, **When** the transcription is sent, **Then** the session receives the text as its next prompt and begins processing.

---

### User Story 3 - Microphone Visibility by State (Priority: P2)

The microphone button only appears on session rows where voice input is meaningful — states where the session can accept a new prompt. Users do not see a microphone on sessions that are actively working, awaiting permission, or in other non-input states.

**Why this priority**: Prevents user confusion by only offering voice input when the session can act on it.

**Independent Test**: Can be tested by observing dashboard rows across all session states and verifying the microphone button appears only on rows with qualifying states.

**Acceptance Scenarios**:

1. **Given** a session in "idle" state, **When** the dashboard renders, **Then** the microphone button is visible on that row.
2. **Given** a session in "ready" state, **When** the dashboard renders, **Then** the microphone button is visible on that row.
3. **Given** a session in "unknown" state, **When** the dashboard renders, **Then** the microphone button is visible on that row.
4. **Given** a session in "working" state, **When** the dashboard renders, **Then** the microphone button is NOT visible on that row.
5. **Given** a session in "permission_required" state, **When** the dashboard renders, **Then** the microphone button is NOT visible on that row.
6. **Given** a session transitions from "idle" to "working", **When** the state change is reflected, **Then** the microphone button disappears from that row.

---

### User Story 4 - Recording Feedback and Controls (Priority: P2)

While recording, the user sees visual feedback (recording status, elapsed time) and can pause, resume, cancel, or finish the recording using buttons or keyboard shortcuts.

**Why this priority**: Users need confidence that recording is active and control over the process, but the core send-prompt flow works without polished controls.

**Independent Test**: Can be tested by starting a recording, verifying visual feedback, using pause/resume, and completing or cancelling.

**Acceptance Scenarios**:

1. **Given** an active recording, **When** the user views the recording interface, **Then** they see a visual indicator that recording is in progress and elapsed time.
2. **Given** an active recording, **When** the user presses the pause control, **Then** recording pauses and can be resumed.
3. **Given** an active recording, **When** the user presses Enter or Done, **Then** recording stops and transcription begins.
4. **Given** an active recording, **When** the user presses Escape or Cancel, **Then** recording is discarded with no side effects.

---

### Edge Cases

- What happens when the user clicks the microphone but no microphone hardware is available? The system should display a clear error and not send any text.
- What happens if a session transitions from idle/ready to working while a recording is in progress? The recording should continue, and the transcribed text should still be delivered (the session will queue it).
- What happens if transcription produces empty text (silence or unintelligible audio)? The system should not send an empty prompt and should inform the user.
- What happens if another recording is already in progress for a different session? Only one recording should be active at a time; the microphone button should be disabled on other rows while recording.
- What happens if the voice tooling dependencies are not installed? The microphone button should not appear, or should display an appropriate message when clicked.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Dashboard MUST display a microphone button on session rows whose state is "idle", "ready", or "unknown".
- **FR-002**: Dashboard MUST NOT display a microphone button on session rows whose state is "working", "permission_required", or any other non-qualifying state.
- **FR-003**: Clicking the microphone button MUST open a voice recording interface associated with that specific session.
- **FR-004**: The recording interface MUST provide Done, Cancel, and Pause/Resume controls.
- **FR-005**: The recording interface MUST support keyboard shortcuts: Enter (done), Escape (cancel), Space (pause/resume).
- **FR-006**: Upon completing a recording, the system MUST transcribe the audio to text locally.
- **FR-007**: After successful transcription, the system MUST deliver the transcribed text as a prompt to the specific Claude session associated with that row.
- **FR-008**: The system MUST prevent multiple simultaneous recordings (one recording at a time across all sessions).
- **FR-009**: The system MUST NOT send an empty prompt if transcription produces no text.
- **FR-010**: The microphone button visibility MUST update dynamically when a session's state changes.
- **FR-011**: The system MUST display a clear error if no microphone hardware is detected.

### Key Entities

- **Voice Recording**: A transient audio capture associated with a specific session, with states: recording, paused, transcribing, complete, cancelled.
- **Prompt Delivery**: The mechanism by which transcribed text is injected as input to a running Claude Code session.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can deliver a voice prompt to an idle or ready session within 5 seconds of finishing speaking (transcription + delivery time).
- **SC-002**: The microphone button is visible on 100% of qualifying-state rows and hidden on 100% of non-qualifying-state rows at all times.
- **SC-003**: Users can complete the full voice-to-prompt flow (click, speak, deliver) without switching away from the dashboard window.
- **SC-004**: Transcription errors or hardware failures result in a clear user-facing message with no silent failures.

## Assumptions

- The voice transcription tooling from `claude-skill-voice` (faster-whisper, sounddevice) is available and installed on the user's system.
- The dashboard runs on a system with a working microphone and audio input capability.
- Prompt delivery to a Claude session uses the same mechanism the dashboard already uses or can discover (stdin pipe, session API, or equivalent).
- Only one user interacts with the dashboard at a time (no multi-user concurrency).
- The voice recording interface can reuse or adapt the existing tkinter-based recording GUI from `claude-skill-voice`.
- Local CPU transcription performance is acceptable for the user's hardware (no GPU required).
