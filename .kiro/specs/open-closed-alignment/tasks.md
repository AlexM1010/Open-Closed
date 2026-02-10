# Open-Closed Architecture Alignment - Tasks

## Phase 1: Life Manager Export (Foundation)

- [x] 1.1 Create Plan Exporter Service
  - [x] 1.1.1 Create `life-manager/src/server/services/plan-exporter.ts` with PlanExporter class
  - [x] 1.1.2 Implement `getOrCreatePlanCalendar()` - finds or creates "Life Manager - Today's Plan" calendar
  - [x] 1.1.3 Implement `clearTodaysPlanEvents()` - removes stale events before export
  - [x] 1.1.4 Implement `createPlanEvent()` - creates calendar event with encoded task data
  - [x] 1.1.5 Implement `encodeTaskTitle()` - formats `[!!!] Title (15m)`
  - [x] 1.1.6 Implement `encodeTaskDescription()` - includes domain, taskId, category, Status: pending
  - [x] 1.1.7 Add unit tests for encoding functions
  - [x] 1.1.8 Add integration test with mock Google Calendar

- [x] 1.2 Create Completion Reader Service
  - [x] 1.2.1 Create `life-manager/src/server/services/completion-reader.ts`
  - [x] 1.2.2 Implement `getCompletions()` - reads Status from plan calendar events
  - [x] 1.2.3 Implement `parseStatus()` - extracts completed|skipped|pending
  - [x] 1.2.4 Implement `parseTaskId()` - extracts Task ID from description
  - [x] 1.2.5 Implement `parseTimestamp()` - extracts CompletedAt/SkippedAt
  - [x] 1.2.6 Add unit tests for parsing functions

- [x] 1.3 Update Planner Router
  - [x] 1.3.1 Modify `generatePlan` mutation to call `planExporter.exportPlan()` after generating
  - [x] 1.3.2 Add `syncPlan` mutation for manual trigger
  - [x] 1.3.3 Wire up dependencies in tRPC context
  - [x] 1.3.4 Add router tests

- [x] 1.4 Update Sync Engine for Completions
  - [x] 1.4.1 Modify `importFromGoogle()` to call `completionReader.getCompletions()`
  - [x] 1.4.2 Store completions in `task_completions` table with source='launcher'
  - [x] 1.4.3 Store skips in new `task_skips` table for analytics
  - [x] 1.4.4 Update domain balance calculations to include launcher completions
  - [x] 1.4.5 Add integration test

- [x] 1.5 Update Database Schema
  - [x] 1.5.1 Add `plan_exports` table to track export history
  - [x] 1.5.2 Add `task_skips` table (taskId, skippedAt, skippedDate)
  - [x] 1.5.3 Generate migration with `npm run db:generate`
  - [x] 1.5.4 Test migration with `npm run db:migrate`

---

## Phase 2: Life Launcher Simplification

- [x] 2.1 Create Plan Calendar Repository
  - [x] 2.1.1 Create `life-launcher/app/src/main/java/app/lifelauncher/data/google/PlanCalendarRepository.kt`
  - [x] 2.1.2 Implement `fetchTodaysPlan()` - reads from "Life Manager - Today's Plan" calendar
  - [x] 2.1.3 Implement `parseEventToPlannedTask()` - decodes event title/description, extracts status
  - [x] 2.1.4 Implement `completeTask()` - updates event with Status: completed + CompletedAt
  - [x] 2.1.5 Implement `skipTask()` - updates event with Status: skipped + SkippedAt
  - [x] 2.1.6 Add singleton pattern (matches existing repositories)

- [ ] 2.2 Create PlannedTask Data Class
  - [x] 2.2.1 Create `life-launcher/app/src/main/java/app/lifelauncher/domain/PlannedTask.kt`
  - [x] 2.2.2 Include: eventId, taskId, title, domain, priority, durationMinutes, category, status, startTime, endTime
  - [ ] 2.2.3 Add parsing helpers for description metadata

- [ ] 2.3 Update WidgetViewModel (Major Rewrite)
  - [ ] 2.3.1 Remove energy slider logic entirely
  - [ ] 2.3.2 Remove `logEnergy()` function
  - [ ] 2.3.3 Add `PlanCalendarRepository` dependency
  - [ ] 2.3.4 Implement `refreshDisplay()` with 45-min event lookahead
  - [ ] 2.3.5 Filter to only `status: pending` tasks
  - [ ] 2.3.6 Update `completeTask()` to update event status
  - [ ] 2.3.7 Update `skipTask()` to update event status
  - [ ] 2.3.8 Add `dismissCalendarEvent()` to show tasks after event dismissed

- [ ] 2.4 Update Widget States
  - [ ] 2.4.1 Remove `EnergyCheckIn` state
  - [ ] 2.4.2 Add `NeedMoreTasks` state (directs user to web app)
  - [ ] 2.4.3 Update `CalendarEvent` state for 45-min lookahead
  - [ ] 2.4.4 Keep: NextTask, Hidden, SignInRequired, Loading, Error

- [ ] 2.5 Update GoogleCalendarRepository
  - [ ] 2.5.1 Add `getImminentEvent(lookaheadMinutes: Int)` parameter
  - [ ] 2.5.2 Change default lookahead from 15 to 45 minutes
  - [ ] 2.5.3 Exclude events from "Life Manager - Today's Plan" calendar

- [ ] 2.6 Update HomeFragment UI
  - [ ] 2.6.1 Remove energy slider UI
  - [ ] 2.6.2 Add UI for `NeedMoreTasks` state (button to open web app)
  - [ ] 2.6.3 Update event card to show "in X min" or "at HH:MM"
  - [ ] 2.6.4 Test state transitions

---

## Phase 3: Cleanup and Polish

- [ ] 3.1 Remove Duplicate Planning Code
  - [ ] 3.1.1 Delete `life-launcher/app/src/main/java/app/lifelauncher/domain/PlannerAlgorithm.kt`
  - [ ] 3.1.2 Remove planner-related imports from WidgetViewModel
  - [ ] 3.1.3 Remove `completions7d` tracking (now done by Life Manager)
  - [ ] 3.1.4 Verify app still builds

- [ ] 3.2 Remove Energy Slider Components
  - [ ] 3.2.1 Delete `EnergySlider.kt` widget (or keep for web app widget if needed)
  - [ ] 3.2.2 Remove energy-related SharedPreferences
  - [ ] 3.2.3 Remove energy-related UI layouts
  - [ ] 3.2.4 Update documentation

- [ ] 3.3 Simplify Google Tasks Usage
  - [ ] 3.3.1 Remove `fetchTasks()` calls from WidgetViewModel
  - [ ] 3.3.2 Keep `GoogleTasksRepository.kt` only for backward compat / fallback
  - [ ] 3.3.3 Update documentation

- [ ] 3.4 Update Architecture Documentation
  - [ ] 3.4.1 Update `life-manager/ARCHITECTURE.md` with new export flow
  - [ ] 3.4.2 Update `life-launcher/ARCHITECTURE.md` - remove energy slider, add 45-min lookahead
  - [ ] 3.4.3 Update main `ARCHITECTURE.md` with simplified data flow
  - [ ] 3.4.4 Update widget states table

- [ ] 3.5 Add Offline Fallback
  - [ ] 3.5.1 Cache last fetched plan in SharedPreferences
  - [ ] 3.5.2 Show cached plan when offline
  - [ ] 3.5.3 Add "Offline" indicator to widget
  - [ ] 3.5.4 Queue completion/skip updates for sync when back online

- [ ]* 3.6 End-to-End Testing (Manual)
  - [ ]* 3.6.1 Manual test: Set energy in Life Manager web app
  - [ ]* 3.6.2 Manual test: Generate plan, verify export to calendar
  - [ ]* 3.6.3 Manual test: Open Life Launcher, verify plan appears
  - [ ]* 3.6.4 Manual test: Complete task, verify status updated in calendar
  - [ ]* 3.6.5 Manual test: Skip task, verify status updated
  - [ ]* 3.6.6 Manual test: Exhaust tasks, verify "Open Life Manager" prompt
  - [ ]* 3.6.7 Manual test: Verify 45-min event lookahead works
  - [ ]* 3.6.8 Manual test: Verify completion shows in Life Manager analytics

---

## Phase 4: Enhanced Features

- [ ]* 4.1 Smart Time-Blocking in Plan Exporter
  - [ ]* 4.1.1 Read user's primary calendar events for the day before planning
  - [ ]* 4.1.2 Find gaps between meetings
  - [ ]* 4.1.3 Slot tasks into gaps based on duration
  - [ ]* 4.1.4 Respect buffer time (5 min before meetings)
  - [ ]* 4.1.5 Handle edge cases: no gaps, overlapping events, all-day events

- [ ]* 4.2 Skip-Cycling in Life Launcher (Eat the Frog + Quick Wins)
  - [ ]* 4.2.1 Life Manager orders tasks hardest → easiest
  - [ ]* 4.2.2 Implement TaskQueue with frontIndex/backIndex
  - [ ]* 4.2.3 Skip = jump to easiest task (backIndex)
  - [ ]* 4.2.4 Complete easy task = return to hardest (frontIndex)
  - [ ]* 4.2.5 Track showingFront boolean in local state
  - [ ]* 4.2.6 Tasks cycle until: completed, snoozed, or plan regenerated
  - [ ]* 4.2.7 Swipe down = snooze (removes from cycle entirely)

- [ ]* 4.3 Snooze-to-Tomorrow Gesture
  - [ ]* 4.3.1 Add swipe-down gesture recognition
  - [ ]* 4.3.2 Update calendar event with `Status: snoozed`, `SnoozedTo: tomorrow's date`
  - [ ]* 4.3.3 Remove from local cycle immediately
  - [ ]* 4.3.4 Life Manager reads snooze status, reschedules for next day

- [ ]* 4.4 Enhanced User Event Display Flow
  - [ ]* 4.4.1 Always show next task event at top of widget
  - [ ]* 4.4.2 When user event is 45+ min away: task event only
  - [ ]* 4.4.3 When user event is 15-45 min away: task event + user event preview below
  - [ ]* 4.4.4 When user event is <15 min away: user event takes over entirely
  - [ ]* 4.4.5 Complete/skip user event → status added (not deleted) → back to task events
  - [ ]* 4.4.6 Track dismissed user events to not re-show until next one

- [ ]* 4.5 Widget Tap Behavior
  - [ ]* 4.5.1 When showing task event: tap opens Life Manager web app
  - [ ]* 4.5.2 When showing "Need More Tasks": tap opens Life Manager web app
  - [ ]* 4.5.3 When no task events and no user events: tap opens Google Calendar for today
  - [ ]* 4.5.4 When showing user event: tap opens event (Zoom link, location, etc.)

- [ ]* 4.6 User Event Completion Tracking
  - [ ]* 4.6.1 When user completes/skips a user event, add status to event (not delete)
  - [ ]* 4.6.2 Only for events on primary calendar (not plan calendar)
  - [ ]* 4.6.3 Life Manager can read this for analytics
  - [ ]* 4.6.4 Life Manager can read this for "meetings attended" analytics
  - [ ]* 4.6.5 Respect user privacy - only track if opted in

---

## Definition of Done

A task is complete when:
1. Code is written and compiles
2. Unit tests pass (where applicable)
3. Manual testing confirms expected behavior
4. No regressions in existing functionality
5. Code follows project conventions (see steering files)

## Priority Order

1. **Task 1.1** - Plan Exporter (enables everything else)
2. **Task 2.1** - Plan Calendar Repository (Life Launcher can read)
3. **Task 2.3** - WidgetViewModel rewrite (connects the pieces)
4. **Task 2.4** - Widget States (simplified states)
5. **Task 1.2** - Completion Reader (analytics work)
6. **Task 3.1** - Remove duplicate code (cleanup)
7. Everything else

## Estimated Effort

| Phase | Tasks | Estimated Hours |
|-------|-------|-----------------|
| Phase 1 | 5 | 8-12 |
| Phase 2 | 6 | 10-14 |
| Phase 3 | 6 | 6-8 |
| Phase 4 | 6 | 12-16 |
| **Total** | **23** | **36-50** |
