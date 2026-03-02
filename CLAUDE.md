# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project

Cumbres is a staff-only internal venue management app (Uruguay). Two roles: **Admin** and **Staff**. All UI text is in Spanish. Timezone: `America/Montevideo`. No currency symbol — plain decimals only.

Full requirements: `docs/PRD.md` · Detailed behavior: `docs/SPEC.md` · Stack rationale: `docs/STACK.md` · User stories: `docs/STORIES.md` · SDLC: `docs/SDLC.md` · Testing: `docs/TEST_STRATEGY.md`

---

## Commands

```bash
npm run dev           # Next.js dev server
npm run build         # Production build
npm run typecheck     # tsc --noEmit
npm run lint          # ESLint, zero warnings allowed

npm run test          # Vitest watch mode
npm run test:run      # Vitest single run
npm run test:coverage # Vitest with V8 coverage report
npm run test:e2e      # Playwright (requires local Supabase running)

npm run db:generate   # Generate Drizzle migration files from schema changes
npm run db:migrate    # Apply pending migrations
npm run db:seed       # Seed local DB with fixtures
npm run db:studio     # Open Drizzle Studio

supabase start        # Start local Supabase stack (Docker required)
supabase stop         # Stop local Supabase stack
supabase status       # Print local URLs and keys for .env.local
```

Run a single Vitest test file: `npx vitest run src/lib/__tests__/payments.test.ts`
Run a single Playwright spec: `npm run test:e2e -- tests/e2e/reservations.spec.ts`

---

## Architecture

### Two clients, strict separation

The app uses two Supabase clients side-by-side — never mix their responsibilities:

- **`@supabase/supabase-js` + `@supabase/ssr`** — Auth (sign in/out, sessions, password reset) and Storage (file uploads/deletes). Used in server components and Server Actions via the SSR helper.
- **Drizzle ORM** — all application data queries (reservations, clients, payments, expenses, reports). Connects directly to Supabase Postgres via `DATABASE_URL`.

### Data flow (App Router)

```
Browser → Next.js Server Component / Server Action
            ├─ Auth/session check via Supabase SSR client
            ├─ Data queries via Drizzle (direct Postgres)
            └─ File operations via Supabase Storage client
```

No API routes for internal data — use Server Actions. Route Handlers only for webhooks and the cron endpoint (`/api/cron/calendar-sync`).

### Access control

Enforced **server-side only** — in middleware and Server Actions. Restricted items are **hidden** in the UI, never just disabled. Staff cannot: manage event types, manage expense categories, view financial/reservations reports, view deleted reservations, or manage users.

### Key paths (planned, post-scaffolding)

```
src/
  app/              Next.js App Router pages and layouts
  components/       shadcn/ui-based UI components
  lib/              Pure business logic (payment calcs, status transitions, utils)
  actions/          Next.js Server Actions (one file per domain)
  db/               Drizzle schema, client, and query helpers
supabase/           Local Supabase config (supabase init)
tests/
  e2e/              Playwright specs
  fixtures/         Seed SQL / seed scripts for E2E
```

### Google Calendar sync

Non-blocking. If sync fails, the reservation still saves and a retry happens on next update. A daily reconciliation cron job runs at 3 AM UTC via Vercel Cron → `/api/cron/calendar-sync`.

---

## Core Business Rules

These rules have tests — don't change the logic without updating them:

- **Payment total** = deposit + regular − refunds. Guarantee payments are **excluded**.
- **Progress %** = total paid / final price, capped 0–100%. Guarantee excluded.
- **Auto-confirm**: when deposit payments total ≥ required deposit amount, status flips to `confirmed` automatically.
- **Status flow**: `pending` ↔ `confirmed` ↔ `cancelled`. No status is terminal; all transitions are allowed.
- **Double-booking**: non-blocking warning only — never prevents saving.
- **Reservations**: soft-delete only (`deleted_at`); never hard-delete.
- **Users**: at least one active Admin must always exist. Enforce this before deactivating or downgrading any user.

---

## Migrations

- Write only **additive** (backwards-compatible) migrations. Never drop or rename a column in the same release as the code change that depends on it.
- Drizzle has no rollback support. A bad migration is fixed by a new forward migration.
- Migrations run automatically during the Vercel build: `npm run db:migrate && npm run build`.

---

## Testing

- **Unit** (Vitest): pure functions in `src/lib/`. No DB, no I/O.
- **Integration** (Vitest + local Supabase): Server Actions with real DB calls.
- **E2E** (Playwright, Chromium only): critical user flows. Requires `supabase start` first.
- Coverage target: 80% lines/functions for `src/lib/` and `src/actions/`.
- Test data should use realistic Spanish-language Uruguayan context (names, dates in Montevideo timezone, plain decimal amounts).

---

## Agents

Two project-scoped agents in `.claude/agents/`:

- **`test-writer`** — invoke after implementing a feature or fixing a bug to write tests following `docs/TEST_STRATEGY.md`.
- **`docs-sync-agent`** — invoke after significant feature completion or business rule changes to keep `docs/` in sync with the codebase.

---

## Commits and PRs

- Conventional Commits enforced via commitlint + husky (added post-scaffolding).
- All changes via PR to `main`; branch naming: `feature/*`, `fix/*`, `chore/*`, `docs/*`, `hotfix/*`.
- Squash-merge only. CI must be green (typecheck → lint → unit tests → E2E).
- PR title must follow Conventional Commits format.
