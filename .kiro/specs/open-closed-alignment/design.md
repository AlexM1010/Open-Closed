# Open-Closed Architecture Alignment - Design

## Overview

This design addresses the fundamental architectural drift: Life Launcher currently runs its own planning algorithm instead of reading the pre-computed plan from Google Calendar. This defeats the hub-and-spoke architecture.

**The fix requires changes to both submodules.**

---

## Design Decisions

### Decision 1: Use a Dedicated Google Calendar for the Plan

**Options considered:**
1. Store plan in Google Tasks with special metadata
2. Store plan in a dedicated Google Calendar
3. Store plan in a custom API endpoint

**Chosen: Option 2 - Dedicated Google Calendar**

**Rationale:**
- Google Calendar provides offline sync for free (Android syncs calendars automatically)
- Calendar events have natural start/end times for time-blocking
- Life Launcher already has Google Calendar integration
- No custom server needed for Life Launcher to read the plan

**Calendar name:** `Life Manager - Today's Plan`

### Decision 2: Task Encoding in Calendar Events

Calendar events will encode task metadata in a parseable format:

```
Event Title: [!!!] Call Mom (15m)
Event Description:
  Domain: Relationships
  Task ID: 42
  Priority: must-do
  Category: must-do|want-to|health
```

This allows Life Launcher to parse without API calls.

### Decision 3: Energy Level - Web App Only

**Previous design:** Life Launcher writes energy to calendar, Life Manager reads it.

**New design:** Energy is set only in Life Manager web app.

**Rationale:**
- Energy is a *planning* input, not an *execution* input
- Belongs in the planning tool (web), not execution tool (launcher)
- Forces intentional "start my day" ritual
- Simpler launcher = faster, more focused
- When tasks exhausted → user returns to web app for energy reset + celebration

**Flow:**
1. Morning: User opens Life Manager web app
2. Sets energy level (0-10)
3. Sees generated plan, confetti if yesterday was good
4. Plan exports to Google Calendar
5. User works through tasks in Life Launcher
6. Tasks exhausted → Launcher shows "Open Life Manager for more tasks"
7. User opens web app, sets new energy, gets new batch

**What Life Launcher no longer does:**
- No energy slider
- No writing "Energy: X" events to calendar
- No local energy storage (except for filtering cached plan)

### Decision 4: Event + Task Display (45-Minute Lookahead)

Life Launcher shows both upcoming calendar events AND planned tasks.

**Display logic:**
1. Check for calendar events starting within 45 minutes
2. If event found → Show event card (title, time, "in X min")
3. If no imminent event → Show next pending task
4. User can dismiss event card to see tasks underneath

**Why 45 minutes:**
- Enough time to wrap up current task and prepare
- Not so far ahead that it's distracting
- Matches typical meeting reminder timing

**Event sources:**
- User's primary Google Calendar (manually scheduled events)
- NOT the "Life Manager - Today's Plan" calendar (those are tasks)

### Decision 4: Completion Tracking

**Problem:** Users finish tasks faster (or slower) than estimated. They want to move to the next task immediately, not wait for the scheduled time. Life Manager needs to know what was completed vs skipped for analytics.

**Solution:** Update events with status metadata instead of deleting them.

When Life Launcher marks a task complete:
1. Update the calendar event description with completion status
2. Immediately show the next pending task (ignore scheduled times)
3. Life Manager reads status on next sync → updates SQLite analytics

**Event Status Values:**
- `pending` - Not yet acted on (default)
- `completed` - User swiped right
- `skipped` - User swiped left

**Updated Event Description Format:**
```
Domain: Relationships
Task ID: 42
Category: must-do
Status: completed
CompletedAt: 2026-02-09T09:12:00Z
ActualDuration: 12
```

**Why update instead of delete:**
- Preserves record of what was planned vs what happened
- Enables "planned vs actual" analytics
- Life Manager can calculate actual task durations
- Skipped tasks are tracked (important for domain balance)

**Life Launcher Display Logic:**
- Show next event where `Status: pending` (or no status)
- Ignore scheduled start times for display order
- User works through tasks at their own pace
- Time blocks are suggestions, not constraints

---

## Component Changes

### Life Manager Changes

#### 1. New Service: `plan-exporter.ts`

```typescript
// src/server/services/plan-exporter.ts

export class PlanExporter {
  private calendarClient: GoogleCalendarClient;
  private readonly PLAN_CALENDAR_NAME = 'Life Manager - Today\'s Plan';
  
  /**
   * Export today's plan to Google Calendar
   * Called after generatePlan() or on schedule
   */
  async exportPlan(plan: TodayPlan, userId: number): Promise<void> {
    // 1. Get or create the plan calendar
    const calendarId = await this.getOrCreatePlanCalendar(userId);
    
    // 2. Clear today's existing plan events
    await this.clearTodaysPlanEvents(calendarId, plan.date);
    
    // 3. Create events for each task
    for (const item of plan.items) {
      await this.createPlanEvent(calendarId, item, plan.date);
    }
  }
  
  private async createPlanEvent(
    calendarId: string, 
    item: TodayPlanItem,
    date: string
  ): Promise<void> {
    const { start, end } = this.calculateTimeBlock(item);
    const title = this.encodeTaskTitle(item.task);
    const description = this.encodeTaskDescription(item);
    
    await this.calendarClient.createEvent(oauth2Client, {
      summary: title,
      description,
      start,
      end,
      // Custom property for identification
      extendedProperties: {
        private: {
          lifeManagerTaskId: item.taskId.toString(),
          category: item.category,
        }
      }
    });
  }
  
  private encodeTaskTitle(task: Task): string {
    const priorityMarker = {
      'must-do': '[!!!]',
      'should-do': '[!!]',
      'nice-to-have': '[!]'
    }[task.priority];
    return `${priorityMarker} ${task.title} (${task.estimatedMinutes}m)`;
  }
}
```

#### 2. New Service: `completion-reader.ts`

```typescript
// src/server/services/completion-reader.ts

export interface TaskCompletion {
  taskId: number;
  status: 'completed' | 'skipped';
  timestamp: Date;
  actualDuration?: number;
}

export class CompletionReader {
  private readonly PLAN_CALENDAR_NAME = 'Life Manager - Today\'s Plan';
  
  /**
   * Read completion/skip status from plan calendar events
   * Life Launcher updates events with Status: completed|skipped
   */
  async getCompletions(userId: number, date: string): Promise<TaskCompletion[]> {
    const calendarId = await this.getPlanCalendarId(userId);
    if (!calendarId) return [];
    
    const events = await this.calendarClient.getEventsForDate(oauth2Client, calendarId, date);
    const completions: TaskCompletion[] = [];
    
    for (const event of events) {
      const status = this.parseStatus(event.description);
      if (status === 'completed' || status === 'skipped') {
        const taskId = this.parseTaskId(event.description);
        const timestamp = this.parseTimestamp(event.description, status);
        
        if (taskId && timestamp) {
          completions.push({
            taskId,
            status,
            timestamp,
            actualDuration: this.parseActualDuration(event.description),
          });
        }
      }
    }
    
    return completions;
  }
  
  private parseStatus(description: string | undefined): string | null {
    if (!description) return null;
    const match = description.match(/Status:\s*(\w+)/);
    return match ? match[1] : null;
  }
  
  private parseTaskId(description: string | undefined): number | null {
    if (!description) return null;
    const match = description.match(/Task ID:\s*(\d+)/);
    return match ? parseInt(match[1], 10) : null;
  }
  
  private parseTimestamp(description: string | undefined, status: string): Date | null {
    if (!description) return null;
    const key = status === 'completed' ? 'CompletedAt' : 'SkippedAt';
    const match = description.match(new RegExp(`${key}:\\s*([\\d\\-T:.Z]+)`));
    return match ? new Date(match[1]) : null;
  }
}
```

#### 3. Scheduled Job: `planning-scheduler.ts`

```typescript
// src/server/services/planning-scheduler.ts

export class PlanningScheduler {
  /**
   * Run planning job
   * Called on schedule (default 6 AM) or when energy changes
   */
  async runPlanningCycle(userId: number): Promise<void> {
    // 1. Import from Google (get latest tasks)
    await this.syncEngine.importFromGoogle();
    
    // 2. Read energy level from calendar
    const energy = await this.energyReader.getLatestEnergy(userId) ?? 5;
    
    // 3. Generate plan
    const plan = generatePlan({
      availableTasks: await this.getAvailableTasks(),
      domains: await this.getDomains(),
      completions7d: await this.getCompletions7d(),
      energyLevel: energy,
      currentDate: new Date().toISOString().split('T')[0],
    });
    
    // 4. Export to Google Calendar
    await this.planExporter.exportPlan(plan, userId);
    
    // 5. Log sync
    await this.logSync('export', 'plan', null, 'success', { 
      taskCount: plan.items.length 
    });
  }
}
```

#### 4. Update Router: Add plan export endpoint

```typescript
// src/server/routers/planner.ts

export const plannerRouter = router({
  // Existing
  generatePlan: publicProcedure
    .input(generatePlanSchema)
    .mutation(async ({ input, ctx }) => {
      const plan = generatePlan(/* ... */);
      
      // NEW: Export to Google Calendar
      await ctx.planExporter.exportPlan(plan, ctx.userId);
      
      return plan;
    }),
    
  // NEW: Manual sync trigger
  syncPlan: publicProcedure
    .mutation(async ({ ctx }) => {
      await ctx.planningScheduler.runPlanningCycle(ctx.userId);
      return { success: true };
    }),
});
```

---

### Life Launcher Changes

#### 1. New Repository: `PlanCalendarRepository.kt`

```kotlin
// data/google/PlanCalendarRepository.kt

class PlanCalendarRepository(private val context: Context) {
    
    private val PLAN_CALENDAR_NAME = "Life Manager - Today's Plan"
    
    /**
     * Fetch today's plan from the dedicated calendar
     * This is the PRIMARY data source (not Google Tasks)
     */
    suspend fun fetchTodaysPlan(): List<PlannedTask> = withContext(Dispatchers.IO) {
        val accessToken = authManager.getAccessToken() ?: return@withContext emptyList()
        val service = buildCalendarService(accessToken)
        
        // Find the plan calendar
        val calendars = service.calendarList().list().execute().items
        val planCalendar = calendars.find { it.summary == PLAN_CALENDAR_NAME }
            ?: return@withContext emptyList()
        
        // Fetch today's events
        val now = Instant.now()
        val endOfDay = now.atZone(ZoneId.systemDefault())
            .toLocalDate()
            .plusDays(1)
            .atStartOfDay(ZoneId.systemDefault())
            .toInstant()
        
        val events = service.events()
            .list(planCalendar.id)
            .setTimeMin(DateTime(now.toEpochMilli()))
            .setTimeMax(DateTime(endOfDay.toEpochMilli()))
            .setOrderBy("startTime")
            .setSingleEvents(true)
            .execute()
            .items ?: emptyList()
        
        return@withContext events.mapNotNull { event ->
            parseEventToPlannedTask(event)
        }
    }
    
    private fun parseEventToPlannedTask(event: Event): PlannedTask? {
        val title = event.summary ?: return null
        val (cleanTitle, priority, duration) = parseTaskTitle(title)
        
        // Parse description for metadata
        val description = event.description ?: ""
        val domain = extractDomain(description)
        val taskId = extractTaskId(description)
        val category = extractCategory(description)
        
        return PlannedTask(
            eventId = event.id,
            taskId = taskId,
            title = cleanTitle,
            domain = domain,
            priority = priority,
            durationMinutes = duration,
            category = category,
            startTime = Instant.ofEpochMilli(event.start.dateTime.value),
            endTime = Instant.ofEpochMilli(event.end.dateTime.value)
        )
    }
    
    /**
     * Mark a task as complete by updating the calendar event status
     * We UPDATE instead of DELETE to preserve analytics data
     */
    suspend fun completeTask(eventId: String): Boolean = withContext(Dispatchers.IO) {
        updateTaskStatus(eventId, "completed")
    }
    
    /**
     * Mark a task as skipped by updating the calendar event status
     */
    suspend fun skipTask(eventId: String): Boolean = withContext(Dispatchers.IO) {
        updateTaskStatus(eventId, "skipped")
    }
    
    private suspend fun updateTaskStatus(eventId: String, status: String): Boolean {
        try {
            val accessToken = authManager.getAccessToken() ?: return false
            val service = buildCalendarService(accessToken)
            
            val calendars = service.calendarList().list().execute().items
            val planCalendar = calendars.find { it.summary == PLAN_CALENDAR_NAME }
                ?: return false
            
            // Get existing event
            val event = service.events().get(planCalendar.id, eventId).execute()
            
            // Update description with status
            val now = Instant.now().toString()
            val existingDesc = event.description ?: ""
            val updatedDesc = updateDescriptionStatus(existingDesc, status, now)
            event.description = updatedDesc
            
            // Save back to calendar
            service.events().update(planCalendar.id, eventId, event).execute()
            return true
        } catch (e: Exception) {
            Log.e("PlanCalendarRepo", "Failed to update task status: ${e.message}")
            return false
        }
    }
    
    private fun updateDescriptionStatus(desc: String, status: String, timestamp: String): String {
        // Remove existing status lines if present
        val lines = desc.lines().filter { 
            !it.startsWith("Status:") && 
            !it.startsWith("CompletedAt:") &&
            !it.startsWith("SkippedAt:")
        }
        
        // Add new status
        val statusLine = "Status: $status"
        val timeLine = if (status == "completed") "CompletedAt: $timestamp" else "SkippedAt: $timestamp"
        
        return (lines + statusLine + timeLine).joinToString("\n")
    }
}
```

#### 2. Update WidgetViewModel

```kotlin
// ui/widget/WidgetViewModel.kt

class WidgetViewModel(application: Application) : AndroidViewModel(application) {
    
    private val planCalendarRepository = PlanCalendarRepository.getInstance(application)
    private val googleCalendarRepository = GoogleCalendarRepository.getInstance(application)
    
    // Current plan from Life Manager
    private var todaysPlan: List<PlannedTask> = emptyList()
    
    /**
     * Initialize - fetch plan and check for events
     * NO energy slider - energy is set in web app only
     */
    fun initialize() {
        if (!prefs.lifeManagerEnabled) {
            _widgetState.value = WidgetState.Hidden
            return
        }
        
        viewModelScope.launch {
            refreshDisplay()
        }
    }
    
    /**
     * Main display logic:
     * 1. Check for calendar events within 45 minutes
     * 2. If event found → show event
     * 3. If no event → show next task
     * 4. If no tasks → show "Open Life Manager"
     */
    private suspend fun refreshDisplay() {
        // Step 1: Check for imminent calendar events (45 min lookahead)
        val imminentEvent = googleCalendarRepository.getImminentEvent(lookaheadMinutes = 45)
        
        if (imminentEvent != null) {
            val timeStr = when {
                imminentEvent.isHappening -> "now"
                imminentEvent.minutesUntilStart <= 5 -> "in ${imminentEvent.minutesUntilStart} min"
                else -> "at ${formatTime(imminentEvent.startTime)}"
            }
            _widgetState.value = WidgetState.CalendarEvent(
                id = imminentEvent.id,
                title = imminentEvent.title,
                startTime = timeStr,
                duration = imminentEvent.durationMinutes,
                isHappening = imminentEvent.isHappening
            )
            return
        }
        
        // Step 2: Show next task from plan
        val allEvents = planCalendarRepository.fetchTodaysPlan()
        val pendingTasks = allEvents.filter { it.status == "pending" || it.status == null }
        
        if (pendingTasks.isEmpty()) {
            // No tasks left - direct to web app
            _widgetState.value = WidgetState.NeedMoreTasks(
                message = "All done! Open Life Manager for more tasks.",
                completedToday = allEvents.count { it.status == "completed" }
            )
            return
        }
        
        todaysPlan = pendingTasks
        val nextTask = todaysPlan.first()
        
        _widgetState.value = WidgetState.NextTask(
            id = nextTask.taskId ?: nextTask.eventId.hashCode(),
            title = nextTask.title,
            duration = nextTask.durationMinutes,
            domain = nextTask.domain
        )
    }
    
    /**
     * Complete task - update status in calendar
     */
    fun completeTask(taskId: Int) {
        val task = todaysPlan.firstOrNull() ?: return
        
        viewModelScope.launch {
            planCalendarRepository.completeTask(task.eventId)
            refreshDisplay()
        }
    }
    
    /**
     * Skip task - update status in calendar, move to next
     */
    fun skipTask(taskId: Int) {
        val task = todaysPlan.firstOrNull() ?: return
        
        viewModelScope.launch {
            planCalendarRepository.skipTask(task.eventId)
            refreshDisplay()
        }
    }
    
    /**
     * Dismiss calendar event - show tasks underneath
     */
    fun dismissCalendarEvent() {
        viewModelScope.launch {
            // Re-run display logic but skip event check this time
            val pendingTasks = planCalendarRepository.fetchTodaysPlan()
                .filter { it.status == "pending" || it.status == null }
            
            if (pendingTasks.isEmpty()) {
                _widgetState.value = WidgetState.NeedMoreTasks(
                    message = "All done! Open Life Manager for more tasks.",
                    completedToday = 0
                )
            } else {
                todaysPlan = pendingTasks
                val nextTask = todaysPlan.first()
                _widgetState.value = WidgetState.NextTask(
                    id = nextTask.taskId ?: nextTask.eventId.hashCode(),
                    title = nextTask.title,
                    duration = nextTask.durationMinutes,
                    domain = nextTask.domain
                )
            }
        }
    }
}
```

#### 3. New Data Class: `PlannedTask.kt`

```kotlin
// domain/PlannedTask.kt

/**
 * A task from the Life Manager plan (read from Google Calendar)
 * Different from Task.kt which is parsed from Google Tasks
 */
data class PlannedTask(
    val eventId: String,           // Google Calendar event ID
    val taskId: Int?,              // Life Manager task ID (for tracking)
    val title: String,
    val domain: String,
    val priority: Priority,
    val durationMinutes: Int,
    val category: String,          // must-do, want-to, health
    val status: String?,           // pending, completed, skipped
    val startTime: Instant,
    val endTime: Instant
)
```

#### 4. Update Widget States

```kotlin
// ui/widget/WidgetState.kt

sealed class WidgetState {
    
    /** Calendar event starting soon (within 45 min) or currently happening */
    data class CalendarEvent(
        val id: String,
        val title: String,
        val startTime: String,
        val duration: Int,
        val isHappening: Boolean
    ) : WidgetState()
    
    /** Next task from the plan */
    data class NextTask(
        val id: Int,
        val title: String,
        val duration: Int,
        val domain: String
    ) : WidgetState()
    
    /** All tasks done - direct user to web app */
    data class NeedMoreTasks(
        val message: String,
        val completedToday: Int
    ) : WidgetState()
    
    /** Widget disabled */
    object Hidden : WidgetState()
    
    /** Sign in required */
    object SignInRequired : WidgetState()
    
    /** Loading */
    object Loading : WidgetState()
    
    /** Error with optional cached fallback */
    data class Error(
        val message: String,
        val cachedState: WidgetState? = null
    ) : WidgetState()
}

// REMOVED: EnergyCheckIn - energy is set in web app only
// REMOVED: BalanceWarning - shown in web app dashboard
// REMOVED: AllCaughtUp - replaced by NeedMoreTasks
```

---

## Data Flow (After Changes)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Life Manager (Web)                            │
│                                                                  │
│  1. User sets energy level (0-10) + sees confetti               │
│  2. Pull tasks from Google Tasks                                │
│  3. Run planning algorithm                                      │
│  4. Export plan to "Life Manager - Today's Plan" calendar       │
│  5. Read completion status from calendar events                 │
│  6. Update analytics (completions, skips, domain balance)       │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Google Calendar (Shared State)                      │
│                                                                  │
│  "Life Manager - Today's Plan" calendar:                        │
│    - [!!!] Call Mom (15m) @ 9:00  [Status: completed]           │
│    - [!!] Pay bills (10m) @ 9:15  [Status: pending]             │
│    - [!] Go for walk (30m) @ 9:25 [Status: pending]             │
│                                                                  │
│  Primary calendar (user's events):                              │
│    - Team standup @ 10:00                                       │
│    - Lunch with Sarah @ 12:30                                   │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Life Launcher (Android)                         │
│                                                                  │
│  1. Check primary calendar for events within 45 min             │
│  2. If event → show event card                                  │
│  3. If no event → read plan from "Today's Plan" calendar        │
│  4. Show next pending task                                      │
│  5. On complete/skip → update event status                      │
│  6. When tasks exhausted → "Open Life Manager"                  │
│                                                                  │
│  NO energy slider - energy set in web app only                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Migration Strategy

### Phase 1: Life Manager Export (No Breaking Changes)
1. Add `plan-exporter.ts` to Life Manager
2. Export plan to calendar when user generates plan
3. Life Launcher continues working with Google Tasks (unchanged)

### Phase 2: Life Launcher Reads Calendar
1. Add `PlanCalendarRepository.kt`
2. Update WidgetViewModel to read from calendar first
3. Fall back to Google Tasks if calendar empty (backward compat)

### Phase 3: Remove Duplicate Code
1. Remove `PlannerAlgorithm.kt` from Life Launcher
2. Remove direct Google Tasks fetching (keep for completion sync only)
3. Update documentation

---

## Testing Strategy

### Life Manager
- Unit test: `plan-exporter.ts` encodes tasks correctly
- Integration test: Plan appears in Google Calendar
- Property test: Any valid plan can be exported and re-parsed

### Life Launcher
- Unit test: `PlanCalendarRepository` parses events correctly
- Integration test: Reads plan from test calendar
- Manual test: Complete flow with real Google account

### End-to-End
- Generate plan in Life Manager web UI
- Verify same tasks appear in Life Launcher widget
- Complete task in Life Launcher
- Verify completion reflected in Life Manager analytics


---

## Enhanced Features Design

### Smart Time-Blocking

Life Manager reads the user's primary calendar before generating the plan, then slots tasks into gaps between meetings.

**Algorithm:**
```typescript
async function generateTimeBlockedPlan(tasks: Task[], userId: number): Promise<PlannedTask[]> {
  // 1. Get today's calendar events (meetings, appointments)
  const userEvents = await calendarClient.getTodayEvents(oauth2Client, 'primary');
  
  // 2. Find gaps between events
  const gaps = findGaps(userEvents, workdayStart: '09:00', workdayEnd: '18:00');
  
  // 3. Sort tasks by score (priority + domain neglect + overdue)
  const sortedTasks = scoreAndSort(tasks);
  
  // 4. Fit tasks into gaps
  const scheduled: PlannedTask[] = [];
  for (const task of sortedTasks) {
    const gap = findFittingGap(gaps, task.estimatedMinutes + 5); // 5 min buffer
    if (gap) {
      scheduled.push({
        ...task,
        startTime: gap.start,
        endTime: addMinutes(gap.start, task.estimatedMinutes),
      });
      shrinkGap(gap, task.estimatedMinutes + 5);
    }
  }
  
  return scheduled;
}
```

**Gap Finding Rules:**
- Minimum gap: 10 minutes (shorter gaps are ignored)
- Buffer before meetings: 5 minutes (don't schedule task ending exactly when meeting starts)
- All-day events: Ignored (they don't block time)
- Overlapping events: Treated as single block

### Skip-Cycling (Eat the Frog + Quick Wins)

Life Manager orders tasks **hardest first** (highest score = most important/difficult). When user skips, they get the **easiest task** from the back of the queue. Complete that? Back to the hardest.

**Psychology**: "Eat the frog" - tackle hard tasks when willpower is high. But if you can't face the frog, a quick win builds momentum.

**Local State in Life Launcher:**
```kotlin
data class TaskQueue(
    val tasks: List<PlannedTask>,  // Ordered hardest → easiest by Life Manager
    val frontIndex: Int,           // Points to hardest remaining
    val backIndex: Int,            // Points to easiest remaining
    val showingFront: Boolean      // true = showing hard task, false = showing easy
)

fun onSkip() {
    // Jump to easiest task (back of queue)
    showingFront = false
    showTask(tasks[backIndex])
}

fun onComplete() {
    if (showingFront) {
        // Completed hard task, advance front
        frontIndex++
    } else {
        // Completed easy task, shrink back, return to front
        backIndex--
        showingFront = true
    }
    
    if (frontIndex > backIndex) {
        // Queue exhausted
        showNeedMoreTasks()
    } else {
        showTask(tasks[if (showingFront) frontIndex else backIndex])
    }
}

fun onSnooze() {
    // Remove current task entirely (front or back)
    if (showingFront) frontIndex++ else backIndex--
    showingFront = true  // Always return to front after snooze
    // ... update calendar with snoozed status
}
```

**Example flow:**
```
Queue: [Hard A] [Medium B] [Medium C] [Easy D]
        ↑front                         ↑back

1. Show Hard A
2. User skips → Show Easy D
3. User completes Easy D → Back to Hard A
4. User skips → Show Medium C (new back)
5. User completes Medium C → Back to Hard A
6. User completes Hard A → Show Medium B
7. Done!
```

**Why this works:**
- Start with hardest (willpower highest in morning)
- Skip gives instant relief (easiest task)
- Completing easy task = momentum → try hard one again
- Never stuck in the middle
- Snooze removes task entirely (not cycling forever)

### Snooze-to-Tomorrow Gesture

Three gestures in Life Launcher:
- **Swipe right** = Complete (Status: completed)
- **Swipe left** = Skip (stays in cycle, moves to end)
- **Swipe down** = Snooze to tomorrow (Status: snoozed, removed from cycle)

**Calendar Event Update for Snooze:**
```
Status: snoozed
SnoozedAt: 2026-02-09T14:30:00Z
SnoozedTo: 2026-02-10
```

Life Manager reads this on next sync and reschedules the task for the snoozed date.

### Enhanced Event Display Flow

The widget shows different content based on how close the next calendar event is:

```
Time until event    Widget shows
─────────────────────────────────────────
> 45 min            Task only
15-45 min           Task + event preview below
< 15 min            Event only (takes over)
Event happening     Event only (until dismissed or ended)
```

**Event Preview (15-45 min away):**
```
┌─────────────────────────────┐
│  [!!] Pay bills (10m)       │
│  Admin                      │
├─────────────────────────────┤
│  📅 Team standup in 32 min  │
└─────────────────────────────┘
```

**Event Takeover (<15 min):**
```
┌─────────────────────────────┐
│  📅 Team standup            │
│  in 12 min                  │
│                             │
│  [Join Zoom] [Dismiss]      │
└─────────────────────────────┘
```

**Dismiss behavior:**
- Swipe left or tap "Dismiss" = hide event, return to task view
- Dismissed events don't re-appear until the next event
- If event is currently happening and dismissed, return to task view

### Widget Tap Behavior

| Widget State | Tap Action |
|--------------|------------|
| Showing task | Open Life Manager web app |
| Showing "Need More Tasks" | Open Life Manager web app |
| Showing event | Open event (Zoom link, Maps, etc.) |
| No tasks, no events | Open Google Calendar for today |

**Implementation:**
```kotlin
fun onWidgetTap() {
    when (val state = widgetState.value) {
        is WidgetState.NextTask -> openLifeManagerWebApp()
        is WidgetState.NeedMoreTasks -> openLifeManagerWebApp()
        is WidgetState.CalendarEvent -> openEventLink(state.event)
        is WidgetState.Hidden -> openGoogleCalendar()
        else -> openLifeManagerWebApp()
    }
}

fun openEventLink(event: CalendarEventData) {
    // Try to extract actionable link
    val link = event.conferenceLink  // Zoom, Meet, Teams
        ?: event.location?.let { if (it.startsWith("http")) it else null }
        ?: "https://calendar.google.com/calendar/r/day"
    
    openUrl(link)
}
```
