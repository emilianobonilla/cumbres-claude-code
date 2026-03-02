# Cumbres — Tech Stack

> **Date:** 2026-03-01
> **Status:** Adopted

---

## Overview

Cumbres is built as a full-stack Next.js application deployed on Vercel, backed by a serverless PostgreSQL database. All layers are chosen to minimize operational overhead while keeping the codebase cohesive and type-safe.

---

## Stack

| Layer | Choice | Version |
|---|---|---|
| Framework | Next.js (App Router) | 15 |
| Language | TypeScript | 5 |
| Database | PostgreSQL (Neon) | — |
| ORM | Drizzle ORM | latest |
| Auth | Auth.js (NextAuth) | v5 |
| File storage | Cloudflare R2 | — |
| Email | Resend | — |
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

### PostgreSQL on Neon
Serverless Postgres with a free tier and automatic scaling. No connection pooling configuration required — Neon handles it. Chosen over Supabase (more than needed) and SQLite (insufficient for concurrent access on PaaS).

### Drizzle ORM
Preferred over Prisma for this project because:
- The financial and reservations reports require non-trivial aggregate queries; Drizzle's SQL-like syntax keeps those readable and easy to optimize.
- Lighter runtime footprint — important for Vercel's serverless functions.
- Schema is defined in TypeScript and migrations are plain SQL files, making them easy to review.

### Auth.js v5 (NextAuth)
Handles session management, the credentials provider (email + password), and the database adapter for storing sessions. The forgot-password flow (time-limited tokens) is implemented on top of it using a custom token table.

### Cloudflare R2
S3-compatible object storage with no egress fees. Used for both event type documents (max 20 MB) and expense receipts (max 10 MB). Files are addressed by auto-generated unique keys to prevent collisions.

### Resend
Transactional email for password reset links. Clean API, React Email for templates, generous free tier.

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
- Environment variable management for secrets (R2 credentials, Google service account, Resend API key, etc.).

---

## What Was Deliberately Left Out

| Excluded | Reason |
|---|---|
| Separate backend service | Next.js Server Actions + Route Handlers cover all API needs |
| Redis / job queue | No real-time or heavy async requirements in the current phase |
| Supabase | Neon + Auth.js covers the same ground with less surface area |
| Prisma | Drizzle is lighter and better suited for complex report queries |
| WhatsApp integration | Deferred to a future phase (see PRD.md §3.9) |
