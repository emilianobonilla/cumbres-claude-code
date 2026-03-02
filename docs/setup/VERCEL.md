# Vercel — Setup Guide

Manual configuration steps for the Vercel project.

---

## 1. Create the Project

1. Go to [vercel.com/new](https://vercel.com/new).
2. Import the GitHub repository.
3. Select the **emilianobonilla** GitHub account and the `cumbres` repo.
4. Framework preset: **Next.js** (auto-detected).
5. Do **not** deploy yet — configure environment variables first.

---

## 2. Build & Output Settings

**Project → Settings → General → Build & Development Settings**

| Setting | Value |
|---|---|
| Build command | `npm run db:migrate && npm run build` |
| Output directory | `.next` (default) |
| Install command | `npm ci` (default) |
| Node.js version | 20.x |

> The build command runs Drizzle migrations automatically on every deploy before the Next.js build. The `DATABASE_URL` env var must be set for this to work.

---

## 3. Disable Automatic Production Deployment

By default, Vercel deploys every push to the production domain. We only want production deployments via GitHub Actions (the `deploy-prod.yml` workflow on release).

**Project → Settings → Git → Production Branch**

- Set "Production Branch" to a **non-existent branch** (e.g., `_production-disabled`) so Vercel never auto-promotes a push to production.
- Vercel will still create Preview deployments for every push to `main` — these serve as the development environment.

Alternatively, under **Git → Ignored Build Step**, add a script that exits 0 only when `VERCEL_ENV` is `preview`:

```bash
[ "$VERCEL_ENV" = "production" ] && exit 0 || exit 1
```

> This second approach allows Vercel to keep the production domain intact while blocking automatic production builds.

---

## 4. Environment Variables

**Project → Settings → Environment Variables**

Add the following variables. Use the **Environment** column to scope each one correctly.

### All environments (Preview + Production)

| Variable | Value |
|---|---|
| `NEXT_PUBLIC_APP_URL` | Preview: `https://<project>.vercel.app` / Prod: your custom domain |

### Preview environment (development)

| Variable | Value source |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Dev Supabase → Settings → API → Project URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Dev Supabase → Settings → API → anon key |
| `SUPABASE_SERVICE_ROLE_KEY` | Dev Supabase → Settings → API → service_role key |
| `DATABASE_URL` | Dev Supabase → Settings → Database → Connection string (Transaction mode, port 6543) |
| `BOOTSTRAP_ADMIN_EMAIL` | Your admin email (used on first boot only) |
| `BOOTSTRAP_ADMIN_NAME` | Your admin name |
| `BOOTSTRAP_ADMIN_PASSWORD` | A strong initial password |
| `GOOGLE_SERVICE_ACCOUNT_KEY` | Base64-encoded JSON from Google Cloud (see `docs/setup/GOOGLE_CLOUD.md`) |

### Production environment

Same keys as above, but pointing to the **prod** Supabase project and prod values.

> `DATABASE_URL` must use **Transaction mode** (port 6543) for Vercel serverless functions to avoid connection exhaustion. Use the **Session mode** (port 5432) connection string only for migrations if needed.

---

## 5. Cron Jobs

**Project → Settings → Cron Jobs**

| Job | Schedule | Path |
|---|---|---|
| Google Calendar reconciliation | `0 3 * * *` (daily at 3 AM UTC) | `/api/cron/calendar-sync` |

> This requires a `CRON_SECRET` env var to protect the endpoint. Add it to environment variables and verify it in the Route Handler.

Add to environment variables:

| Variable | Value |
|---|---|
| `CRON_SECRET` | A long random secret (generate with `openssl rand -hex 32`) |

---

## 6. Custom Domain (production)

**Project → Settings → Domains**

1. Add your custom domain (e.g., `cumbres.yourdomain.com`).
2. Add the DNS record shown by Vercel at your DNS provider.
3. Vercel provisions SSL automatically.
4. Update `NEXT_PUBLIC_APP_URL` in the Production environment to match.

---

## 7. Retrieve Vercel IDs (for GitHub Actions)

After creating the project, retrieve the IDs needed for `VERCEL_ORG_ID` and `VERCEL_PROJECT_ID` GitHub secrets:

```bash
npm i -g vercel
vercel login
vercel link   # links the local project; creates .vercel/project.json
cat .vercel/project.json
```

Output will contain `orgId` and `projectId`. Add them as GitHub repository secrets (see `docs/setup/GITHUB.md`).

> Do not commit `.vercel/project.json` — it's already in `.gitignore`.
