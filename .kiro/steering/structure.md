# Project Structure

## Directory Layout

```
src/
├── server/              # Backend (Node.js + tRPC)
│   ├── db/
│   │   ├── schema.ts    # Drizzle schema (single source of truth)
│   │   ├── index.ts     # DB connection + migration runner
│   │   └── seed.ts      # Default data seeder
│   ├── routers/         # tRPC API endpoints (one per domain)
│   ├── services/        # Business logic (pure functions)
│   ├── trpc.ts          # tRPC context + router initialization
│   └── index.ts         # Express server + tRPC adapter
├── client/              # Frontend (React + Vite)
│   ├── components/      # React components
│   ├── hooks/           # Custom React hooks
│   ├── lib/
│   │   └── trpc.ts      # tRPC client setup
│   ├── App.tsx          # Main app component + navigation
│   └── main.tsx         # React entry point
└── shared/
    ├── types.ts         # Shared Zod schemas + TypeScript types
    └── types/           # Additional shared type definitions
```

## Architecture Patterns

### Service Layer Pattern

Business logic lives in `src/server/services/` as pure functions:
- **Input**: Plain objects (no database or HTTP concerns)
- **Output**: Plain objects
- **No side effects**: Services don't touch database or make HTTP calls
- **Testable**: Easy to unit test and property test

Example: `planner.ts` contains the planning algorithm as a pure function.

### tRPC Router Pattern

Routers in `src/server/routers/` handle:
1. Input validation (Zod schemas)
2. Database queries (via Drizzle)
3. Calling service functions
4. Returning results

Routers are thin orchestration layers.

### Shared Validation

All validation schemas live in `src/shared/types.ts`:
- Defined once with Zod
- Used on both client and server
- TypeScript types inferred from schemas
- Ensures client and server stay in sync

### Database Schema

Single source of truth: `src/server/db/schema.ts`
- Drizzle auto-generates TypeScript types
- Migrations generated with `npm run db:generate`
- All tables use snake_case column names
- All dates stored as ISO strings

## Naming Conventions

- **Files**: kebab-case (e.g., `daily-log-form.tsx`)
- **Components**: PascalCase (e.g., `DailyLogForm`)
- **Functions**: camelCase (e.g., `generatePlan`)
- **Database columns**: snake_case (e.g., `created_at`)
- **TypeScript types**: PascalCase (e.g., `TodayPlan`)
- **Constants**: SCREAMING_SNAKE_CASE (e.g., `TASK_PRIORITY`)

## Testing Conventions

- **Unit tests**: `*.test.ts` files co-located with source
- **Property tests**: `*.property.test.ts` files for universal properties
- **Integration tests**: In `__tests__/` directories
- **Test framework**: Vitest with fast-check for property-based testing

## Code Organization Rules

1. **Services are pure**: No database or HTTP calls in service functions
2. **Routers are thin**: Minimal logic, mostly orchestration
3. **Validation at the edge**: All inputs validated with Zod schemas
4. **Types from schemas**: Use `z.infer<typeof schema>` for types
5. **Single schema source**: All schemas in `src/shared/types.ts`
6. **Database types from Drizzle**: Import from schema, don't redefine

## Key Files Reference

| File | Purpose |
|------|---------|
| `src/server/db/schema.ts` | Database schema (all tables) |
| `src/shared/types.ts` | Validation schemas + types |
| `src/server/trpc.ts` | tRPC router registration |
| `src/client/lib/trpc.ts` | tRPC client setup |
| `vite.config.ts` | Proxy `/api` to backend (port 4000) |
| `drizzle.config.ts` | Drizzle ORM configuration |

## Path Aliases

Available in both client and server code:
- `@/*` → `./src/*`
- `@client/*` → `./src/client/*`
- `@server/*` → `./src/server/*`
- `@shared/*` → `./src/shared/*`
