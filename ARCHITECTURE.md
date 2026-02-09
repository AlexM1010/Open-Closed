# Open-Closed Architecture

## System Overview

Open-Closed is a dual-platform life management system built on Structural Optimism principles. It consists of two applications that work together through a hub-and-spoke architecture.

```
┌─────────────────────────────────────────────────────────────────┐
│              Life Manager (Web) - Intelligence Hub               │
│                                                                  │
│  Aggregates data from multiple sources:                         │
│  • Google Tasks & Calendar                                      │
│  • Todoist (planned)                                            │
│  • Google Fit (planned)                                         │
│  • Custom fitness plans                                         │
│                                                                  │
│  Responsibilities:                                              │
│  • Pull data from all integrations                             │
│  • Store historical data in SQLite                             │
│  • Run complex planning algorithm                              │
│  • Generate analytics, streaks, guardrails                     │
│  • Export simplified daily plan to Google Calendar             │
└──────────────┬──────────────────────────────────────────────────┘
               │
               │ Exports daily plan
               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Google Calendar/Tasks                         │
│                  (Shared Source of Truth)                        │
│                                                                  │
│  • Stores the computed daily plan                              │
│  • Launcher reads from here                                    │
│  • Enables offline access for launcher                         │
└──────────────┬──────────────────────────────────────────────────┘
               │
               │ Reads daily plan
               ▼
┌─────────────────────────────────────────────────────────────────┐
│            Life Launcher (Android) - Simple Display              │
│                                                                  │
│  Responsibilities:                                              │
│  • Read next task from Google Calendar                         │
│  • Display on home screen widget                               │
│  • Swipe right to complete, left to skip                       │
│  • Energy check-in slider (0-10)                               │
│  • Minimal UI, maximum simplicity                              │
└─────────────────────────────────────────────────────────────────┘
```

## Design Philosophy

### Separation of Concerns

**Life Manager (Web)** = Intelligence Layer
- Complex data aggregation
- Multi-source integration
- Historical analytics
- Planning algorithm
- Desktop-friendly stats dashboard

**Life Launcher (Android)** = Interaction Layer
- Simple, glanceable interface
- Quick task completion
- Energy logging
- No complex logic
- Works offline with cached data

### Why This Architecture?

1. **Scalability**: Add new integrations to life-manager without touching launcher
2. **Simplicity**: Launcher stays lightweight and fast
3. **Flexibility**: Users can use just launcher, just manager, or both
4. **Reliability**: Google Calendar provides offline access and sync
5. **Privacy**: Each user's data stays in their own Google account

## Data Flow

### Daily Planning Flow

```
1. Life Manager wakes up (scheduled job)
   ↓
2. Pull data from all sources:
   - Google Tasks (tasks by domain)
   - Todoist (additional tasks)
   - Google Fit (activity data)
   - SQLite (historical completions, energy logs)
   ↓
3. Run planning algorithm:
   - Filter by energy level
   - Score by priority + domain balance + overdue
   - Select top N tasks for today
   ↓
4. Export to Google Calendar:
   - Create/update "Today's Plan" calendar
   - Each task becomes a calendar event
   - Include metadata (domain, priority, duration)
   ↓
5. Life Launcher reads from Google Calendar:
   - Fetch "Today's Plan" events
   - Display next task on widget
   - User completes → mark done in Google
   ↓
6. Life Manager syncs completions:
   - Pull completed tasks from Google
   - Update SQLite for analytics
   - Recalculate domain balance
```

### Task Completion Flow

```
User swipes right on launcher
   ↓
Launcher marks task complete in Google Calendar
   ↓
Google Calendar syncs (automatic)
   ↓
Life Manager detects completion (next sync cycle)
   ↓
Updates SQLite analytics
   ↓
Recalculates domain balance for next plan
```

## Task Encoding Convention

Both applications understand tasks through a shared encoding format:

```
Task Title Format:
[!!!] Call Mom (15m)
 │     │        │
 │     │        └── Duration in minutes
 │     └── Task title
 └── Priority: [!!!]=must-do, [!!]=should-do, [!]=nice-to-have

Task List/Calendar = Domain:
- "Health" list/calendar
- "Admin" list/calendar
- "Relationships" list/calendar
- "University" list/calendar
- "Creative" list/calendar
```

## Integration Architecture

### Current Integrations

**Google Tasks & Calendar**
- Primary task source
- Shared between both apps
- Free API, reliable

### Planned Integrations

**Todoist**
- Additional task source
- Life Manager pulls tasks
- Exports to Google Calendar for launcher

**Google Fit**
- Activity tracking (steps, workouts)
- Informs energy level adjustments
- Influences task selection

**Custom Fitness Plans**
- Weekly workout schedules
- Adaptive intensity based on energy
- Stored in life-manager SQLite
- Exported as calendar events

### Adding New Integrations

To add a new integration:

1. **Create client service** in `life-manager/src/server/services/`
   ```typescript
   export class NewServiceClient {
     async fetchData(): Promise<Data[]> { ... }
   }
   ```

2. **Add sync metadata table** in `life-manager/src/server/db/schema.ts`
   ```typescript
   export const newServiceSyncMetadata = sqliteTable('new_service_sync', {
     id: integer('id').primaryKey(),
     externalId: text('external_id').notNull(),
     lastSyncedAt: text('last_synced_at'),
   });
   ```

3. **Extend sync engine** in `life-manager/src/server/services/sync-engine.ts`
   ```typescript
   async importFromNewService() {
     const data = await this.newServiceClient.fetchData();
     // Store in SQLite
     // Export relevant items to Google Calendar
   }
   ```

4. **Update planner algorithm** to incorporate new data source

5. **Launcher automatically sees results** through Google Calendar

## Planning Algorithm

The algorithm is deterministic and runs in life-manager:

**Inputs:**
- Tasks from all sources (Google, Todoist, etc.)
- Completions in last 7 days (from SQLite)
- Current energy level (from latest check-in)
- Activity data (from Google Fit)
- Current date and time

**Algorithm:**
1. Aggregate tasks from all sources
2. Filter by energy-based duration cap
3. Score each task:
   ```
   score = priority_weight 
         + domain_neglect_bonus 
         + overdue_bonus
         + fitness_plan_bonus
   ```
4. Select top N tasks (N depends on energy)
5. Export to Google Calendar as "Today's Plan"

**Energy Configuration:**
| Energy | Tasks | Duration Cap | Fitness Intensity |
|--------|-------|--------------|-------------------|
| 0-3    | 2-3   | 15 min max   | Rest day          |
| 4-6    | 3-5   | No cap       | Light workout     |
| 7-10   | 5-6   | No cap       | Full intensity    |

## Technology Stack

### Life Manager (Web)
- **Frontend**: React 18 + Vite + Tailwind CSS
- **Backend**: Express + tRPC
- **Database**: SQLite + Drizzle ORM
- **Testing**: Vitest + fast-check
- **APIs**: Google Calendar, Google Tasks, (Todoist, Google Fit planned)

### Life Launcher (Android)
- **Language**: Kotlin
- **UI**: Android Widgets + Custom Views
- **APIs**: Google Calendar API (via Play Services)
- **Storage**: SharedPreferences (cache only)

## Security & Privacy

### Data Storage
- **User data**: Stored in user's own Google account
- **Analytics**: Stored locally in life-manager SQLite
- **No central database**: Each user runs their own life-manager instance
- **OAuth tokens**: Stored securely (Android Keystore / server environment)

### Privacy Principles
1. User owns their data (in their Google account)
2. Life-manager can be self-hosted
3. No telemetry or tracking
4. Open source (GPLv3)

## Offline Behavior

### Life Launcher
- Caches last fetched plan from Google Calendar
- Shows cached tasks when offline
- Queues completions for sync when back online
- Shows "Offline" indicator

### Life Manager
- Requires internet for integrations
- Can run planning algorithm on cached data
- Syncs when connection restored

## Deployment Options

### Life Manager
1. **Local** (recommended for privacy)
   - Run on user's machine
   - `npm run dev` for development
   - `npm run build && npm start` for production

2. **Self-hosted server**
   - Deploy to personal VPS
   - Railway, Fly.io, or Render

3. **Shared hosting** (future)
   - Multi-tenant with user isolation
   - Each user's data in their Google account

### Life Launcher
- Distributed via APK or Google Play Store
- Each user authenticates with their own Google account
- Reads from their personal Google Calendar

## Future Enhancements

### Planned Features
- [ ] Todoist integration
- [ ] Google Fit integration
- [ ] Custom fitness plan builder
- [ ] Habit tracking
- [ ] Weekly review generator
- [ ] Voice input for task creation
- [ ] Wear OS companion app

### Architectural Improvements
- [ ] Real-time sync (WebSocket/SSE)
- [ ] Conflict resolution UI
- [ ] Multi-device energy sync
- [ ] Backup/restore functionality
- [ ] Export to other calendar apps

## Development Workflow

### Working on Life Manager
```bash
cd life-manager
npm install
npm run db:migrate
npm run dev
```

### Working on Life Launcher
```bash
cd life-launcher
./gradlew assembleDebug
adb install app/build/outputs/apk/debug/app-debug.apk
```

### Testing Integration
1. Set up Google OAuth credentials
2. Run life-manager locally
3. Add tasks via life-manager UI
4. Verify they appear in Google Calendar
5. Open life-launcher on Android
6. Verify tasks appear in widget

## Contributing

See individual project READMEs for contribution guidelines:
- [Life Manager Contributing](./life-manager/README.md)
- [Life Launcher Contributing](./life-launcher/README.md)

## License

Both projects are licensed under [GNU GPLv3](https://www.gnu.org/licenses/gpl-3.0.en.html).
