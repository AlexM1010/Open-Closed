# Tech Stack

## Core Technologies

- **Database**: SQLite + Drizzle ORM (local-first, zero-server)
- **API**: tRPC (end-to-end type safety between client and server)
- **Frontend**: React 18 + Vite + Tailwind CSS
- **Backend**: Express + Node.js
- **Validation**: Zod schemas (shared between client and server)
- **Testing**: Vitest + fast-check (property-based testing)
- **Recurrence**: rrule.js (iCalendar standard)
- **Charts**: Recharts
- **External APIs**: Google Calendar API, Google Tasks API (optional)

## Build System

- **Package Manager**: npm
- **Module System**: ESM (type: "module" in package.json)
- **TypeScript**: Strict mode enabled
- **Bundler**: Vite (frontend), tsx (backend dev server)

## Common Commands

```bash
# Development
npm run dev              # Start both client and server (concurrently)
npm run dev:server       # Start backend only (port 4000)
npm run dev:client       # Start frontend only (port 5173)

# Build
npm run build            # TypeScript compile + Vite build
npm run preview          # Preview production build

# Testing
npm test                 # Run all tests (Vitest)
npm run test:ui          # Visual test UI
npm test -- --watch      # Watch mode

# Database
npm run db:generate      # Generate migration from schema changes
npm run db:migrate       # Apply migrations
npm run db:studio        # Open Drizzle Studio (visual DB browser)
npm run db:seed          # Seed default domains

# Ports
# Backend: 4000
# Frontend: 5173
# Vite proxies /api requests to backend
```

## Key Dependencies

- **@trpc/server** & **@trpc/client**: Type-safe API layer
- **@tanstack/react-query**: Data fetching and caching (used by tRPC)
- **drizzle-orm**: Type-safe SQL query builder
- **better-sqlite3**: SQLite driver
- **googleapis**: Google Calendar/Tasks integration
- **p-retry**: Retry logic for sync operations
- **fast-check**: Property-based testing library
- **zod**: Runtime validation and type inference

## TypeScript Configuration

- **Target**: ES2022
- **Module**: ESNext with bundler resolution
- **Strict mode**: Enabled
- **Path aliases**: `@/*`, `@client/*`, `@server/*`, `@shared/*`
- **JSX**: react-jsx (automatic runtime)
