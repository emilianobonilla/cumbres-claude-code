# Cumbres — Tech Stack

> **Date:** 2026-03-02
> **Status:** Adopted

---

## Overview

Cumbres is built as a full-stack Next.js application deployed on Vercel. Supabase provides the PostgreSQL database, file storage, and authentication in a single platform, minimizing the number of external services. All layers are chosen to minimize operational overhead while keeping the codebase cohesive and type-safe.

---

## Stack

| Layer | Choice | Version |
|---|---|---|
| Framework | Next.js (App Router) | 15 |
| Language | TypeScript | 5 |
| Database | PostgreSQL (Supabase) | — |
| ORM | Drizzle ORM | latest |
| Auth | Supabase Auth + @supabase/ssr | — |
| File storage | Supabase Storage | — |
| UI components | shadcn/ui + Tailwind CSS | — |
| Charts | Recharts | — |
| Excel export | SheetJS (xlsx) | — |
| Google Calendar | googleapis (npm) | — |
| PWA | @ducanh2912/next-pwa | — |
| Deployment | Vercel | — |

---

## Decision Notes

### Next.js 15 (App Router)
Full-stack framework that handles both the React frontend and the API layer (Server Actions + Route Handlers) in a single project. Eliminates the need for a separate backend service and maps naturally to Vercel's deployment model.

### Supabase
Consolidates three infrastructure concerns into one platform:

- **PostgreSQL** — managed database with automatic backups, connection pooling via Supabase's pooler, and a direct connection string for Drizzle.
- **Storage** — S3-compatible file storage for event type documents (max 20 MB) and expense receipts (max 10 MB). Files are stored with auto-generated unique keys to prevent collisions.
- **Auth** — email + password authentication with built-in session management, password reset emails, and Next.js App Router integration via `@supabase/ssr`. The forgot-password flow (time-limited reset links) is handled natively — no custom implementation or external email provider required. Custom SMTP (e.g., Resend) can be wired in later if the built-in limits become a constraint for this small internal team.

### Drizzle ORM
Preferred over Prisma and over the Supabase JS query builder because:
- The financial and reservations reports require non-trivial aggregate queries; Drizzle's SQL-like syntax keeps those readable and easy to optimize.
- Lighter runtime footprint — important for Vercel's serverless functions.
- Schema is defined in TypeScript and migrations are plain SQL files, making them easy to review.
- Connects directly to Supabase Postgres via the connection string (direct or pooler mode).

### Two clients, clean separation
The app uses two clients in parallel, each with a distinct responsibility:
- **`@supabase/supabase-js` + `@supabase/ssr`** — auth operations (sign in, sign out, session management, password reset) and file storage uploads/deletes.
- **Drizzle ORM** — all application data queries (reservations, clients, payments, expenses, reports, etc.).

This is a common and well-supported pattern. Auth and storage stay in the Supabase client; business data stays in Drizzle.

### shadcn/ui + Tailwind CSS
Component library suited to internal dashboards. Components are copied into the project (not installed as a dependency), keeping full control over styling. Tailwind handles responsive layout for desktop and mobile.

### Recharts
React-native charting library used for the financial report (revenue trend, payment breakdown) and reservations report (volume trend, popular days). Integrates cleanly with shadcn/ui's card components.

### SheetJS (xlsx)
Server-side Excel generation for reservation exports (3-sheet .xlsx) and expense exports. Runs inside Next.js Route Handlers — no separate service needed.

### googleapis
Official Google client library for the Google Calendar sync feature. Used server-side only (service account credentials stored in environment variables). Sync failures are non-blocking — the reservation saves regardless and errors are logged for retry.

### @ducanh2912/next-pwa
Best-maintained PWA wrapper for Next.js. Generates the service worker and web manifest needed for home-screen installation on mobile — a hard requirement per the spec.

### Vercel
Native deployment target for Next.js. Provides:
- **Vercel Cron** — for the daily Google Calendar reconciliation job (no separate cron service needed).
- Edge network for static assets and server functions.
- Environment variable management for secrets (Supabase URL/keys, Google service account, etc.).

---

## What Was Deliberately Left Out

| Excluded | Reason |
|---|---|
| Separate backend service | Next.js Server Actions + Route Handlers cover all API needs |
| Redis / job queue | No real-time or heavy async requirements in the current phase |
| Neon | Supabase provides PostgreSQL |
| Cloudflare R2 | Supabase Storage covers file storage needs |
| Auth.js (NextAuth) | Supabase Auth covers email + password, sessions, and password reset |
| Resend | Supabase Auth handles the only outbound email (password reset); revisit if limits become an issue |
| Prisma | Drizzle is lighter and better suited for complex report queries |
| Supabase JS query builder | Drizzle is used for all data queries; the Supabase client is scoped to auth and storage only |
| WhatsApp integration | Deferred to a future phase (see PRD.md §3.9) |
