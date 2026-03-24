# Feature Specification: Tray Icon Redesign

**Feature Branch**: `003-tray-icon`
**Created**: 2026-03-24
**Status**: Draft
**Input**: User description: "Tray icon improvements: replace border dot with expressive face (eyes/mouth as state indicator), harsh saturated colors without transitions, left-click toggles dashboard visibility, right-click opens menu"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Face Features as State Indicator (Priority: P1)

A user glances at their system tray and immediately reads session state from the robot face itself. The eyes and mouth change color to reflect the highest-priority session state. No separate status dot, no border circle. The face IS the indicator. At 16x16 or 24x24 tray scale, bold saturated colors on the eyes and mouth read clearly against the neutral head.

**Why this priority**: The status dot with border is illegible at small tray sizes. Moving state indication into the face features (eyes, mouth) that already exist solves visibility and makes the icon more expressive. This is the core ask.

**Independent Test**: Can be fully tested by changing session states and confirming the tray icon's eye and mouth colors update to match, with no status dot present.

**Acceptance Scenarios**:

1. **Given** no active sessions, **When** the dashboard is running, **Then** the robot's eyes and mouth render in a neutral/default color (gray).
2. **Given** one or more sessions in a known state (e.g., working, idle, permission_required, awaiting_input), **When** the tray icon updates, **Then** the eyes and mouth render in the configured color for that state.
3. **Given** multiple sessions in different states, **When** the tray icon updates, **Then** the eyes and mouth reflect the highest-priority state's color.
4. **Given** any session state, **When** the user views the tray icon, **Then** there is no status dot or badge overlay anywhere on the icon.

---

### User Story 2 - Harsh Saturated Colors Without Transitions (Priority: P1)

State colors are bold and saturated. Not pastel, not muted. When state changes, the icon snaps immediately to the new color. No fade, no animation, no transition. The color either is or isn't.

**Why this priority**: Diluted colors defeat the purpose of a glanceable indicator at small scale. Transitions add complexity and delay recognition. Tied with P1 because the face-as-indicator only works if the colors are strong enough to read.

**Independent Test**: Can be tested by triggering state changes and confirming colors are visually distinct, saturated, and change instantaneously.

**Acceptance Scenarios**:

1. **Given** a state change occurs, **When** the tray icon updates, **Then** the new color appears immediately with no intermediate frames or gradual transition.
2. **Given** the default color palette, **When** rendered at the smallest supported tray size, **Then** each state color is visually distinguishable from every other state color.
3. **Given** the user has configured custom state colors, **When** those colors are applied to the face, **Then** they render as specified without any automatic desaturation or blending.

---

### User Story 3 - Left-Click Toggles Dashboard Visibility (Priority: P2)

A user left-clicks the tray icon. If the dashboard window is hidden, it appears. If it's visible, it hides. One click, toggle behavior. The tray icon is a visibility switch.

**Why this priority**: Improves daily workflow by giving the tray icon a direct action instead of requiring the right-click menu to show/hide. Slightly lower than P1 because the current menu-based approach works, just less convenient.

**Independent Test**: Can be tested by left-clicking the tray icon repeatedly and confirming the dashboard alternates between visible and hidden.

**Acceptance Scenarios**:

1. **Given** the dashboard window is hidden, **When** the user left-clicks the tray icon, **Then** the dashboard window becomes visible.
2. **Given** the dashboard window is visible, **When** the user left-clicks the tray icon, **Then** the dashboard window hides.
3. **Given** the dashboard window is visible but behind other windows, **When** the user left-clicks the tray icon, **Then** the dashboard window is brought to the front (treated as "visible" — not hidden and re-shown).

---

### User Story 4 - Right-Click Opens Context Menu (Priority: P2)

Right-clicking the tray icon opens the context menu with settings, quit, and other actions. This is the standard tray convention on both Linux and Windows.

**Why this priority**: Preserves existing menu functionality while relocating it to the expected interaction (right-click). Paired with left-click toggle since together they define the complete click behavior.

**Independent Test**: Can be tested by right-clicking the tray icon and confirming the menu appears with all expected items.

**Acceptance Scenarios**:

1. **Given** the tray icon is visible, **When** the user right-clicks it, **Then** a context menu appears with Settings, Quit, and any other configured actions.
2. **Given** the context menu is open, **When** the user selects an action, **Then** that action executes as expected.

---

### Edge Cases

- What happens when the dashboard process starts but the tray icon hasn't rendered yet? State updates queue and apply once the icon is ready.
- What happens if the platform doesn't distinguish left-click from right-click on tray icons? Fall back to showing the menu on any click.
- What happens when the face is rendered at extremely small sizes (16x16)? Eyes and mouth must still be distinguishable colored regions, even if simplified.
- How does the icon look on both light and dark system tray backgrounds? The neutral head color must provide sufficient contrast for colored eyes/mouth on both.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST render the robot face's eyes and mouth in the color corresponding to the current highest-priority session state.
- **FR-002**: System MUST NOT render a status dot, badge, or any overlay element separate from the face features.
- **FR-003**: System MUST NOT apply border, outline, or stroke to the status indicator elements (eyes/mouth).
- **FR-004**: System MUST change eye and mouth color instantaneously when session state changes — no animation, fade, or transition.
- **FR-005**: Default state colors MUST be high-saturation, bold values that remain distinguishable at 16x16 pixel rendering.
- **FR-006**: Left-click on the tray icon MUST toggle dashboard window visibility (show if hidden, hide if visible).
- **FR-007**: Right-click on the tray icon MUST open the context menu.
- **FR-008**: System MUST respect user-configured custom colors for state indication, applying them to eyes and mouth without modification.
- **FR-009**: The robot head (background shape) MUST remain a neutral, non-state color so that eye/mouth colors stand out.
- **FR-010**: On platforms that do not support distinct left/right click on tray icons, system MUST fall back to showing the menu on any click.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can identify the current session state from the tray icon alone, without hovering or clicking, at the platform's default tray icon size.
- **SC-002**: Each distinct session state produces a visually distinguishable icon — no two states look the same at any supported tray size.
- **SC-003**: Users can show or hide the dashboard with a single left-click on the tray icon, completing the action in under 1 second.
- **SC-004**: State color changes appear on the tray icon within the same update cycle as the state change — no perceptible delay or intermediate state.

## Assumptions

- The robot face design (head shape, antenna) is retained — only the eyes, mouth, and status dot are changing.
- pystray supports distinguishing left-click from right-click on both Linux and Windows (to be verified during planning; fallback defined in FR-010).
- The existing `generate_icon_image()` function in `tray.py` is the sole place where icon rendering occurs — no other code path generates tray icons.
- User-configured colors from settings are already available as RGB tuples at the point where the icon is generated.
- The current Show/Hide menu items are removed from the right-click menu since left-click handles visibility directly.
