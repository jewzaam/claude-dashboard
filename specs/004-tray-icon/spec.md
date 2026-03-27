# Feature Specification: Tray Icon Redesign

**Feature Branch**: `004-tray-icon`
**Created**: 2026-03-24
**Status**: Implemented
**Input**: User description: "Tray icon improvements: solid color state indicator, left-click toggles dashboard visibility, right-click opens menu"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Solid Color Icon as State Indicator (Priority: P1)

A user glances at their system tray and immediately reads session state from the icon color. The entire icon shape fills with the configured color for the highest-priority session state. No badge dot, no border circle. The color IS the indicator. At tray scale, a bold solid shape is more legible than fine detail.

**Why this priority**: The previous status dot with border was illegible at small tray sizes. A solid colored icon shape is the most readable approach at any scale.

**Independent Test**: Can be fully tested by changing session states and confirming the tray icon color updates to match the configured state color.

**Acceptance Scenarios**:

1. **Given** no active sessions, **When** the dashboard is running, **Then** the icon renders in the configured idle color (default gray).
2. **Given** one or more sessions in a known state, **When** the tray icon updates, **Then** the icon renders in the configured color for that state from settings.
3. **Given** multiple sessions in different states, **When** the tray icon updates, **Then** the icon reflects the highest-priority state's configured color.
4. **Given** any session state, **When** the user views the tray icon, **Then** there is no status dot, badge, or overlay — the icon shape itself is the indicator.

---

### User Story 2 - Instant Color Changes (Priority: P1)

When state changes, the icon snaps immediately to the new color. No fade, no animation, no transition.

**Why this priority**: Transitions add complexity and delay recognition. The color either is or isn't.

**Independent Test**: Can be tested by triggering state changes and confirming colors change instantaneously.

**Acceptance Scenarios**:

1. **Given** a state change occurs, **When** the tray icon updates, **Then** the new color appears immediately with no intermediate frames or gradual transition.
2. **Given** the user has configured custom state colors in settings, **When** those colors are applied to the icon, **Then** they render as the exact RGB values specified.

---

### User Story 3 - Left-Click Toggles Dashboard Visibility (Priority: P2)

A user left-clicks the tray icon. If the dashboard window is hidden, it appears. If it's visible, it hides. One click, toggle behavior.

**Why this priority**: Improves daily workflow by giving the tray icon a direct action instead of requiring the right-click menu to show/hide.

**Independent Test**: Can be tested by left-clicking the tray icon repeatedly and confirming the dashboard alternates between visible and hidden.

**Acceptance Scenarios**:

1. **Given** the dashboard window is hidden, **When** the user left-clicks the tray icon, **Then** the dashboard window becomes visible.
2. **Given** the dashboard window is visible, **When** the user left-clicks the tray icon, **Then** the dashboard window hides.
3. **Given** the dashboard window is visible but behind other windows, **When** the user left-clicks the tray icon, **Then** the dashboard window is brought to the front.

---

### User Story 4 - Right-Click Opens Context Menu (Priority: P2)

Right-clicking the tray icon opens a context menu with Settings, Restart, Quit, and unhide actions for hidden sessions. Show/Hide items are removed since left-click handles visibility.

**Why this priority**: Standard tray convention on both Linux and Windows.

**Independent Test**: Can be tested by right-clicking the tray icon and confirming the menu appears with expected items.

**Acceptance Scenarios**:

1. **Given** the tray icon is visible, **When** the user right-clicks it, **Then** a context menu appears with Settings, Restart, Quit, and any unhide items for hidden sessions.
2. **Given** the context menu is open, **When** the user selects an action, **Then** that action executes as expected.

---

### Edge Cases

- What happens when the dashboard process starts but the tray icon hasn't rendered yet? State updates queue and apply once the icon is ready.
- What happens if the platform doesn't distinguish left-click from right-click on tray icons? Fall back to showing the menu on any click (pystray `default=True` on Toggle menu item handles this).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The tray icon MUST be filled entirely with the configured color for the current highest-priority session state.
- **FR-002**: System MUST NOT render a status dot, badge, or any overlay element separate from the icon shape.
- **FR-003**: State colors MUST be read from user settings (hex → RGB conversion), not hardcoded.
- **FR-004**: Color changes MUST be instantaneous — no animation, fade, or transition.
- **FR-005**: Left-click on the tray icon MUST toggle dashboard window visibility (show if hidden, hide if visible), implemented via pystray `MenuItem("Toggle", ..., default=True)`.
- **FR-006**: Right-click on the tray icon MUST open the context menu with Settings, Restart, Quit, and unhide items for hidden sessions.
- **FR-007**: Show/Hide menu items MUST NOT appear in the right-click menu (left-click handles visibility).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can identify the current session state from the tray icon color alone, without hovering or clicking.
- **SC-002**: Each distinct session state produces a visually distinguishable icon color.
- **SC-003**: Users can show or hide the dashboard with a single left-click on the tray icon.
- **SC-004**: State color changes appear on the tray icon within the same update cycle as the state change.

## Clarifications

### Session 2026-03-26

- Q: What icon design should be used? → A: Iterated from robot face → solid color robot → solid color eye. Final: circle with off-center dark pupil (eye looking to the side). Solid state color fill, transparent background.
- Q: What color should the icon use? → A: Read from user settings (same hex colors as dashboard rows), converted to RGB via `_hex_to_rgb`.
- Q: Robot face at tray scale? → A: Too small for fine detail. Simplified to bold geometric shape.

## Assumptions

- pystray's `default=True` on a MenuItem handles left-click on Windows (confirmed — same pattern as d4-timer-w11).
- The existing `generate_icon_image()` function in `tray.py` is the sole place where icon rendering occurs.
- User-configured colors from settings are converted from hex strings to RGB tuples at render time.
- The current Show/Hide menu items are removed from the right-click menu since left-click handles visibility directly.
