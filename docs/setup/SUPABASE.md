# Supabase — Setup Guide

Two Supabase projects are required: one for development and one for production. Follow these steps for **each project** unless noted otherwise.

---

## 1. Create Projects

1. Go to [supabase.com/dashboard](https://supabase.com/dashboard) and create two projects:
   - `cumbres-dev`
   - `cumbres-prod`
2. Choose the same region for both (e.g., South America — São Paulo).
3. Set a strong database password for each. Store them in a password manager.

---

## 2. Collect Connection Details

For each project, go to **Settings → API** and note:

| Value | Where to find it |
|---|---|
| Project URL | Settings → API → Project URL |
| `anon` public key | Settings → API → Project API keys → anon / public |
| `service_role` secret key | Settings → API → Project API keys → service_role |
| Database connection string | Settings → Database → Connection string → Transaction mode (port 6543) |

These values go into Vercel environment variables and GitHub Environment secrets. See `docs/setup/VERCEL.md` and `docs/setup/GITHUB.md`.

---

## 3. Auth Configuration

**Authentication → Settings**

### Disable public sign-ups

| Setting | Value |
|---|---|
| Enable email confirmations | ❌ (staff accounts are created by admins; no confirmation needed) |
| Enable sign ups | ❌ (all users are created via the admin UI, not self-registration) |

### Email templates — Password Reset

**Authentication → Email Templates → Reset Password**

Customize the email subject and body to match the app's language (Spanish/Uruguay):

**Subject:**
```
Restablecer contraseña — Cumbres
```

**Body (HTML):**
```html
<p>Hola,</p>
<p>Recibimos una solicitud para restablecer la contraseña de tu cuenta en Cumbres.</p>
<p><a href="{{ .ConfirmationURL }}">Restablecer contraseña</a></p>
<p>Este enlace expira en 1 hora. Si no solicitaste el cambio, ignorá este correo.</p>
```

### URL Configuration

**Authentication → URL Configuration**

| Setting | Value |
|---|---|
| Site URL | `https://<your-domain>` (prod) or `https://<project>.vercel.app` (dev) |
| Redirect URLs | Add: `http://localhost:3000/**` (for local dev) |

---

## 4. Storage — Buckets

**Storage → New bucket** (create both buckets in each project)

### `party-type-documents`

| Setting | Value |
|---|---|
| Name | `party-type-documents` |
| Public | ❌ (private; access via signed URLs or service role) |
| File size limit | 20 MB |
| Allowed MIME types | *(leave empty — any file type accepted)* |

### `expense-receipts`

| Setting | Value |
|---|---|
| Name | `expense-receipts` |
| Public | ❌ (private) |
| File size limit | 10 MB |
| Allowed MIME types | `image/jpeg, image/png, image/webp, application/pdf` |

---

## 5. Storage — Access Policies

Since the app accesses Storage using the **service role key** (server-side only), RLS policies on buckets are not strictly required. However, as a defense-in-depth measure, add a policy that blocks all unauthenticated access:

For each bucket: **Storage → [bucket] → Policies → New policy → Custom**

```sql
-- Allow service role full access (server-side)
-- No client-side policies needed; all access goes through Server Actions
```

> If you later need to generate signed URLs client-side, add a policy here. For now, all file operations go through Next.js Server Actions using the service role key.

---

## 6. Database — Connection Pooling

Vercel serverless functions need connection pooling. Supabase provides two connection strings:

| Mode | Port | Use for |
|---|---|---|
| Transaction (pooler) | 6543 | Vercel serverless functions (set as `DATABASE_URL`) |
| Session (direct) | 5432 | Drizzle migrations (if running locally or in CI) |

The build command (`npm run db:migrate && npm run build`) runs in Vercel's build environment, which is **not** a serverless function — it can use either. The Transaction mode URL works for both, so use it everywhere.

---

## 7. Database — Row Level Security

Drizzle connects using the `service_role` key (via `DATABASE_URL`), which bypasses RLS. **Do not enable RLS** on tables unless you add a separate client-side query layer later.

Access control is enforced at the application layer (Next.js middleware + Server Actions), not at the DB layer.

---

## 8. Supabase CLI — Local Dev Setup

Install the CLI and initialize the local project:

```bash
brew install supabase/tap/supabase
supabase login          # authenticates with your Supabase account
supabase init           # creates supabase/ folder in the project root
supabase start          # starts local Postgres + Auth + Storage (Docker required)
supabase status         # prints local URLs and keys to paste into .env.local
```

To link to the remote dev project (enables push/pull of migrations):

```bash
supabase link --project-ref <dev-project-ref>
# Project ref: Settings → General → Reference ID
```

---

## 9. Personal Access Token (for CI)

The `ci.yml` workflow uses the Supabase CLI to start a local Supabase instance. The CLI needs a personal access token:

1. Go to [supabase.com/dashboard/account/tokens](https://supabase.com/dashboard/account/tokens).
2. Create a token named `cumbres-ci`.
3. Add it as the `SUPABASE_ACCESS_TOKEN` GitHub repository secret (see `docs/setup/GITHUB.md`).
