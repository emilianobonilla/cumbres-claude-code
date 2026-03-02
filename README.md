# Cumbres

Private internal web application for managing a single event venue. Covers the full lifecycle of a booking — from initial reservation through payment collection, client communication, and financial reporting.

## Documentation

| Document | Description |
|---|---|
| [`docs/PRD.md`](docs/PRD.md) | Product Requirements Document |
| [`docs/SPEC.md`](docs/SPEC.md) | Functional Specification (tech-agnostic) |
| [`docs/STORIES.md`](docs/STORIES.md) | User Stories by epic |
| [`docs/STACK.md`](docs/STACK.md) | Tech stack decisions and rationale |
| [`docs/SDLC.md`](docs/SDLC.md) | Branch strategy, CI/CD, release flow, local dev setup |
| [`docs/TEST_STRATEGY.md`](docs/TEST_STRATEGY.md) | Testing philosophy, tools, and coverage targets |
| [`docs/setup/GITHUB.md`](docs/setup/GITHUB.md) | GitHub configuration (branch protection, environments, secrets) |
| [`docs/setup/VERCEL.md`](docs/setup/VERCEL.md) | Vercel project setup and environment variables |
| [`docs/setup/SUPABASE.md`](docs/setup/SUPABASE.md) | Supabase project setup (Auth, Storage, CLI) |
| [`docs/setup/GOOGLE_CLOUD.md`](docs/setup/GOOGLE_CLOUD.md) | Google Cloud setup for Calendar sync |

## Tech Stack

Next.js 15 · TypeScript · Supabase (PostgreSQL + Auth + Storage) · Drizzle ORM · Tailwind CSS · shadcn/ui · Vercel

## Getting Started

See [`docs/SDLC.md`](docs/SDLC.md) for the full local dev setup. Quick start:

```bash
npm install
supabase start        # requires Docker
npm run db:migrate
npm run dev
```

External service configuration: [`docs/setup/`](docs/setup/)
