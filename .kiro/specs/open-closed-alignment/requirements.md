# Open-Closed Architecture Alignment

## Audit Summary

### Current State vs. Architecture Vision

The main ARCHITECTURE.md describes a **hub-and-spoke** architecture where:
- **Life Manager (Web)** = Intelligence Hub (aggregates data, runs planning, exports to Google Calendar)
- **Life Launcher (Android)** = Simple Display (reads from Google Calendar, displays tasks)

**Critical Finding:** The current implementation has Life Launcher running its own planning algorithm and fetching directly from Google Tasks, rather than reading the pre-computed plan from Google Calendar as the architecture specifies.

---

## Gaps Identified

### Gap 1: Life Launcher Runs Its Own Planner (MAJOR)

**Architecture says:**
> "Life Launcher simply displays the pre-computed plan from Google Calendar."

**Current implementation:**
- `PlannerAlgorithm.kt` duplicates the planning logic from life-manager
- `WidgetViewModel.kt` calls `PlannerAlgorithm.generatePlan()` locally
- Fetches tasks directly from Google Tasks API

**Impact:** 
- Duplicate code to maintain
- Plans can diverge between web and mobile
- Defeats the purpose of hub-and-spoke architecture

### Gap 2: No "Today's Plan" Calendar Export (MAJOR)

**Architecture says:**
> "Export to Google Calendar as 'Today's Plan'"
> "Each task becomes a calendar event"

**Current implementation:**
- Life Manager has sync-engine.ts but doesn't export a "Today's Plan" calendar
- Tasks are synced individually, not as a computed daily plan
- No dedicated calendar for the plan

### Gap 3: Life Launcher Reads Google Tasks, Not Calendar (MAJOR)

**Architecture says:**
> "Life Launcher reads from Google Calendar"
> "Fetch 'Today's Plan' events"

**Current implementation:**
- `GoogleTasksRepository.kt` fetches from Google Tasks API
- `GoogleCalendarRepository.kt` exists but only for imminent event interrupts
- No code to read a "Today's Plan" calendar

### Gap 4: Task Encoding Convention Partially Implemented

**Architecture says:**
```
[!!!] Call Mom (15m)
 │     │        │
 │     └── Task title
 └── Priority: [!!!]=must-do, [!!]=should-do, [!]=nice-to-have
```

**Current implementation:**
- Life Launcher: `parseTaskTitle()` correctly parses this format ✓
- Life Manager: No encoding when exporting tasks to Google

### Gap 5: User Task Selection Not Implemented

**Architecture says:**
> "User picks tasks for today + adds time estimates"
> "User picks tasks, add time estimates, hit 'Plan My Day'"

**Current implementation:**
- Planning algorithm auto-selects tasks based on scoring
- No UI for user to pick specific tasks for today
- No time estimate input per task

### Gap 6: Widget States Need Simplification

**Architecture says (updated design):**
| State | When |
|-------|------|
| NextTask | Tasks available |
| CalendarEvent | User event within 45 min |
| NeedMoreTasks | All tasks done, directs to web app |
| Hidden | Widget disabled |
| SignInRequired | Not authenticated |
| Loading | Fetching data |
| Error | Something went wrong |

**Current implementation:**
- `WidgetState.kt` has: CalendarEvent, NextTask, EnergyCheckIn, Hidden, SignInRequired, Loading, Error
- Has `EnergyCheckIn` which should be removed (energy set in web app only)
- Missing: `NeedMoreTasks` state

---

## Requirements for Alignment

### Requirement 1: Life Manager Exports Daily Plan to Google Calendar

Life Manager must export the computed daily plan to a dedicated Google Calendar.

**Acceptance Criteria:**
- [ ] Create/update a "Life Manager - Today's Plan" calendar in user's Google account
- [ ] Each planned task becomes a calendar event with:
  - Title: `[!!!] Task title (15m)` (encoded format)
  - Description: Domain name, task description
  - Start/End: Time-blocked based on algorithm
  - Custom property: `lifeManagerTaskId` for tracking
- [ ] Clear previous day's plan events before creating new ones
- [ ] Export runs when user generates plan (user-initiated, not scheduled)

### Requirement 2: Life Launcher Reads from "Today's Plan" Calendar

Life Launcher must read the pre-computed plan from Google Calendar instead of running its own planner.

**Acceptance Criteria:**
- [ ] Remove `PlannerAlgorithm.kt` (or keep only for offline fallback)
- [ ] Fetch events from "Life Manager - Today's Plan" calendar
- [ ] Parse task metadata from event title/description
- [ ] Filter to only `Status: pending` events (show next in order)
- [ ] Cache plan for offline access
- [ ] No energy filtering in launcher (energy filtering happens during planning in web app)

### Requirement 3: Task Completion/Skip Syncs Back to Life Manager

When user completes or skips a task in Life Launcher, it must sync back for analytics.

**Acceptance Criteria:**
- [ ] Life Launcher updates calendar event with `Status: completed` or `Status: skipped`
- [ ] Include timestamp: `CompletedAt` or `SkippedAt`
- [ ] Optionally include `ActualDuration` for analytics
- [ ] Life Manager reads status on next sync cycle
- [ ] Completions stored in `task_completions` table
- [ ] Skips stored in `task_skips` table
- [ ] Domain balance recalculated including launcher completions
- [ ] Dashboard shows accurate completion/skip counts

**Why update instead of delete:**
- Preserves record of what was planned vs what happened
- Enables "planned vs actual" analytics
- Skipped tasks are tracked (important for domain balance nudges)
- Life Manager can calculate actual task durations

### Requirement 4: Plan Generation is User-Initiated

Planning requires user input (energy level) and should be an intentional "start my day" ritual.

**Acceptance Criteria:**
- [ ] Plan is generated when user sets energy level in web app
- [ ] Plan exports to Google Calendar immediately after generation
- [ ] No automatic/scheduled plan generation (energy level is required input)
- [ ] User can regenerate plan by setting a new energy level

### Requirement 5: User Works at Own Pace

Life Launcher must let users work through tasks faster than scheduled.

**Acceptance Criteria:**
- [ ] Time blocks are suggestions, not constraints
- [ ] When user completes a task early, immediately show next pending task
- [ ] Don't wait for scheduled start time
- [ ] Filter display to only `Status: pending` events
- [ ] Maintain order from Life Manager's plan (don't re-sort)

### Requirement 6: Add Missing Widget States

Life Launcher must implement simplified widget states.

**Acceptance Criteria:**
- [ ] Remove `EnergyCheckIn` state (energy set in web app only)
- [ ] Add `NeedMoreTasks` state (shown when all tasks done, directs to web app)
- [ ] Keep `CalendarEvent` state for 45-minute lookahead
- [ ] Keep `NextTask`, `Hidden`, `SignInRequired`, `Loading`, `Error` states

### Requirement 7: 45-Minute Event Lookahead

Life Launcher must show upcoming calendar events with progressive urgency.

**Acceptance Criteria:**
- [ ] Check user's primary Google Calendar for events
- [ ] > 45 min away: Show task only
- [ ] 15-45 min away: Show task + event preview below
- [ ] < 15 min away: Event takes over widget entirely
- [ ] User can dismiss event (swipe left) to return to task view
- [ ] Dismissed events don't re-appear until next event
- [ ] Event sources: primary calendar only (NOT the plan calendar)

---

## Requirements for Enhanced Features (Phase 4)

### Requirement 8: Smart Time-Blocking

Life Manager must slot tasks into gaps between calendar events.

**Acceptance Criteria:**
- [ ] Read user's primary calendar before generating plan
- [ ] Find gaps between meetings (minimum 10 min gap)
- [ ] Slot tasks into gaps based on duration
- [ ] Add 5-minute buffer before meetings
- [ ] Handle edge cases: no gaps, all-day events, overlapping events
- [ ] Tasks that don't fit in gaps are scheduled at end of day

### Requirement 9: Skip-Cycling (Eat the Frog + Quick Wins)

Life Launcher must cycle skipped tasks using front/back queue logic. This is purely local state - skipped tasks remain `pending` in the calendar.

**Acceptance Criteria:**
- [ ] Tasks ordered hardest → easiest by Life Manager
- [ ] Swipe left = skip → jump to easiest task (back of queue)
- [ ] Complete easy task → return to hardest (front of queue)
- [ ] Track frontIndex/backIndex locally (SharedPreferences)
- [ ] Skipped tasks stay `Status: pending` in calendar (not updated)
- [ ] Tasks cycle until: completed, snoozed, or plan regenerated
- [ ] Cycle state resets when plan is regenerated

### Requirement 10: Snooze-to-Tomorrow Gesture

Life Launcher must support snoozing tasks to tomorrow.

**Acceptance Criteria:**
- [ ] Swipe down = snooze to tomorrow
- [ ] Update calendar event with `Status: snoozed`, `SnoozedTo: [date]`
- [ ] Remove from local cycle immediately
- [ ] Life Manager reads snooze status on next sync
- [ ] Life Manager reschedules task for snoozed date

### Requirement 11: Widget Tap Behavior

Life Launcher widget must respond to taps contextually.

**Acceptance Criteria:**
- [ ] Tap on task → Open Life Manager web app
- [ ] Tap on "Need More Tasks" → Open Life Manager web app
- [ ] Tap on event → Open event link (Zoom, Meet, location)
- [ ] Tap when no tasks/events → Open Google Calendar for today

---

## Out of Scope (Future)

- Todoist integration
- Google Fit integration
- Custom fitness plans
- Real-time sync (WebSocket/SSE)
- Multi-device energy sync
- User task selection UI (currently auto-selects based on scoring; HOW-IT-WORKS describes future manual selection)

---

## Dependencies

- Google Calendar API (both apps)
- Google Tasks API (Life Manager only after alignment)
- OAuth tokens (both apps)

## Risks

1. **Breaking change for existing users**: Current Life Launcher users fetch from Google Tasks directly. Migration needed.
2. **Offline degradation**: If Life Launcher can't reach Google Calendar, needs fallback.
3. **Sync latency**: Plan changes in Life Manager won't appear instantly in Life Launcher.

## Success Metrics

- Life Launcher shows same tasks as Life Manager web UI
- No duplicate planning code
- Task completion in either app reflects in both within 5 minutes
