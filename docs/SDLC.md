# SDLC — Cumbres

Defines the full development lifecycle: branching strategy, environments, CI/CD workflows, release process, local dev setup, and commit conventions.

---

## Branch Strategy

```
main          ← single integration branch; protected; requires PR + CI
feature/*     ← feature work   → PR to main
fix/*         ← bug fixes      → PR to main
chore/*       ← maintenance    → PR to main
docs/*        ← docs updates   → PR to main
hotfix/*      ← urgent fixes   → PR to main
```

**Rules:**
- `main` is always deployable.
- No direct pushes to `main`; all changes via PR.
- Branch naming must match the prefixes above (enforced by `label-pr.yml`).
- Delete branches after merge.

---

## Environments

| Environment | Trigger | Approval | Supabase |
|---|---|---|---|
| `development` | Push to `main` (via Vercel Git integration) | None | Dev project |
| `production` | GitHub Release published | 1 manual approval | Prod project |

- Vercel's **automatic production deployment** is disabled. Production is only deployed via the `deploy-prod.yml` workflow on `release published`.
- Every merge to `main` auto-deploys to the Vercel **Preview/Dev** environment via the standard Vercel Git integration.
- GitHub Environments store scoped secrets:
  - `development`: dev-tier Supabase URL + anon key
  - `production`: prod-tier Supabase URL + anon key + required reviewer

---

## GitHub Repository Secrets

| Secret | Purpose |
|---|---|
| `SUPABASE_ACCESS_TOKEN` | Supabase CLI authentication in CI |
| `SUPABASE_URL_DEV` | Dev Supabase project URL |
| `SUPABASE_ANON_KEY_DEV` | Dev Supabase anon key |
| `SUPABASE_URL_PROD` | Prod Supabase project URL |
| `SUPABASE_ANON_KEY_PROD` | Prod Supabase anon key |
| `GOOGLE_SERVICE_ACCOUNT_KEY` | Google Calendar sync (JSON, base64-encoded) |
| `VERCEL_TOKEN` | Vercel CLI token for `deploy-prod.yml` |
| `VERCEL_ORG_ID` | Vercel org identifier |
| `VERCEL_PROJECT_ID` | Vercel project identifier |

> `DATABASE_URL` is set directly as a Vercel environment variable (not in GitHub Actions), since migrations run during the Vercel build step (`npm run db:migrate && npm run build`).

---

## CI/CD Workflows

### `ci.yml` — Continuous Integration

Runs on every PR targeting `main`.

**Jobs (sequential, fail-fast):**

1. `typecheck` — `tsc --noEmit`
2. `lint` — `eslint . --max-warnings 0`
3. `unit-tests` — `vitest run --coverage`
4. `e2e-tests` — starts local Supabase via CLI, runs `playwright test`, tears down

**Environment:** Node 20, ubuntu-latest

**Caching:**
- `node_modules` via `actions/cache` (key: `package-lock.json` hash)
- Playwright browsers cached separately (key: Playwright version)

### `deploy-prod.yml` — Production Deployment

Triggers:
- `release: published` (automatic on GitHub Release)
- `workflow_dispatch` with a `tag` input (for manual re-deploys of any prior release)

Requires `production` GitHub Environment (manual approval gate).

Steps:
1. Checkout at the release tag
2. Install Vercel CLI
3. Pull Vercel environment variables
4. Run `vercel --prod --token $VERCEL_TOKEN`

> Migrations run automatically during the Vercel build (`npm run db:migrate && npm run build`), using the `DATABASE_URL` Vercel environment variable.

### `label-pr.yml` — Auto-label PRs

Runs on PR open/edit. Labels PRs based on branch prefix:

| Branch prefix | Label |
|---|---|
| `feature/` | `feature` |
| `fix/` | `bug` |
| `chore/` | `chore` |
| `docs/` | `documentation` |
| `hotfix/` | `hotfix` |

---

## PR Process

1. Create a branch from `main` using the correct prefix.
2. Open a PR to `main` — the PR template will guide the description.
3. CI must pass (all 4 jobs).
4. At least 1 review approval required (configure in branch protection).
5. Squash-merge preferred; the merge commit message must follow Conventional Commits.
6. Delete the branch after merge.

---

## Release Flow

1. Ensure `main` is in a releasable state (CI green, tested in dev).
2. Create a GitHub Release:
   - Tag: `vX.Y.Z` (semver)
   - Target: `main`
   - Write release notes (list of merged PRs / features)
3. Publishing the release triggers `deploy-prod.yml`.
4. Approve the production deployment in the GitHub Environments UI.
5. Monitor Vercel deployment logs and verify the app post-deploy.

---

## Rollback Strategy

### Rolling back a Release (production)

**Option A — Instant Vercel rollback (preferred; seconds, no code change)**

Vercel retains every production deployment. Rollback via:
- Dashboard: Deployments → select a prior deployment → Promote to Production
- CLI: `vercel rollback`

This does NOT roll back DB migrations — only the application code.

**Option B — Code rollback (new patch release)**

1. `git revert <merge-commit>` on main
2. Publish a new GitHub Release (patch bump, e.g. `v1.0.1`)
3. CI runs, prod deploy workflow triggers

Use when Vercel rollback alone isn't sufficient (e.g., config or env var changes).

**Database migrations — no automatic rollback.**

Drizzle does not support rollbacks. Rules:
- Always write **backwards-compatible** (additive) migrations: add columns/tables; never drop or rename in the same release as code that depends on the change.
- For **destructive** schema changes (drop column, rename):
  1. **Release 1**: code handles both old and new schema; apply additive migration.
  2. **Release 2**: apply the destructive migration once all traffic uses the new schema.
- If a bad migration reaches production, write a new **forward migration** to correct it and release immediately.

### Rolling back a merged PR (development)

Every merged PR on GitHub has a **Revert** button. The revert PR goes through normal CI and, once merged, auto-deploys to dev. Recovery is typically very fast.

---

## Commit Conventions

All commits must follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

**Types:** `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `perf`, `ci`, `build`, `revert`

**Examples:**
```
feat(reservations): add double-booking warning
fix(payments): correct deposit auto-confirm threshold
chore: update dependencies
docs: add SDLC and test strategy
```

Enforced via **commitlint** + **husky** (added after scaffolding — see below).

---

## Post-Scaffolding Config Files

These files require `package.json` / `npm install` and must be created right after project scaffolding.

### Install dev dependencies

```bash
npm install -D husky commitlint @commitlint/config-conventional lint-staged
npx husky init
```

### `commitlint.config.ts`

```ts
import type { UserConfig } from "@commitlint/types";

const config: UserConfig = {
  extends: ["@commitlint/config-conventional"],
  rules: {
    "type-enum": [
      2,
      "always",
      ["feat", "fix", "chore", "docs", "refactor", "test", "perf", "ci", "build", "revert"],
    ],
    "subject-case": [2, "always", "lower-case"],
    "header-max-length": [2, "always", 100],
  },
};

export default config;
```

### `.husky/commit-msg`

```sh
#!/bin/sh
npx --no -- commitlint --edit "$1"
```

### `.husky/pre-commit`

```sh
#!/bin/sh
npx lint-staged
```

### `lint-staged` config (add to `package.json`)

```json
"lint-staged": {
  "*.{ts,tsx}": [
    "eslint --max-warnings 0",
    "tsc --noEmit --skipLibCheck"
  ]
}
```

---

## Local Dev Setup

### Prerequisites

- Node.js 20+
- Docker Desktop (for local Supabase)
- Supabase CLI: `brew install supabase/tap/supabase`
- Vercel CLI (optional): `npm i -g vercel`

### First-time setup

```bash
# 1. Clone the repo
git clone git@github.com:<org>/cumbres.git
cd cumbres

# 2. Install dependencies
npm install

# 3. Copy env file and fill in values
cp .env.example .env.local

# 4. Start local Supabase (Docker required)
supabase start
# Outputs local URLs and keys — copy them to .env.local

# 5. Run DB migrations
npm run db:migrate

# 6. Seed dev data (if seed script exists)
npm run db:seed

# 7. Start the dev server
npm run dev
```

### Useful commands

| Command | Description |
|---|---|
| `npm run dev` | Start Next.js dev server |
| `npm run build` | Production build |
| `npm run typecheck` | TypeScript check (`tsc --noEmit`) |
| `npm run lint` | ESLint with zero warnings |
| `npm run test` | Vitest unit tests (watch mode) |
| `npm run test:run` | Vitest unit tests (single run) |
| `npm run test:coverage` | Vitest with coverage report |
| `npm run test:e2e` | Playwright E2E tests |
| `npm run db:migrate` | Run Drizzle migrations |
| `npm run db:generate` | Generate Drizzle migration files |
| `npm run db:seed` | Seed local DB with fixtures |
| `npm run db:studio` | Open Drizzle Studio |
| `supabase start` | Start local Supabase stack |
| `supabase stop` | Stop local Supabase stack |
| `supabase status` | Show local Supabase URLs and keys |

### Environment variables (`.env.local`)

```bash
# Supabase (from `supabase status` for local dev)
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Database (direct Postgres connection for Drizzle)
DATABASE_URL=

# Google Calendar
GOOGLE_SERVICE_ACCOUNT_KEY=   # base64-encoded JSON

# Admin bootstrap (first startup only)
BOOTSTRAP_ADMIN_EMAIL=
BOOTSTRAP_ADMIN_NAME=
BOOTSTRAP_ADMIN_PASSWORD=

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## What Is Not Set Up Yet (post-scaffolding)

| Item | Requires |
|---|---|
| `commitlint.config.ts` | Node/npm |
| `.husky/` hooks | `npm install` |
| `vitest.config.ts` | App scaffolding |
| `playwright.config.ts` | App scaffolding |
| `supabase/` folder | `supabase init` |
| `drizzle.config.ts` | App scaffolding |
| `.env.example` | App scaffolding |
