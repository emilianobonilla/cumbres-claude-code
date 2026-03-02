# GitHub — Setup Guide

Manual configuration steps required in the GitHub repository after pushing the initial code.

---

## 1. Branch Protection — `main`

**Settings → Branches → Add branch ruleset** (or classic protection rule)

| Setting | Value |
|---|---|
| Branch name pattern | `main` |
| Require a pull request before merging | ✅ |
| Required approving reviews | 1 |
| Dismiss stale reviews when new commits are pushed | ✅ |
| Require status checks to pass | ✅ |
| Required status checks | `Typecheck`, `Lint`, `Unit Tests`, `E2E Tests` |
| Require branches to be up to date | ✅ |
| Do not allow bypassing the above settings | ✅ |
| Allow force pushes | ❌ |
| Allow deletions | ❌ |

> Status check names must match the `name:` field of each job in `ci.yml` exactly. They will only appear in the dropdown after the first CI run.

---

## 2. Environments

**Settings → Environments → New environment**

### `development`

| Setting | Value |
|---|---|
| Name | `development` |
| Deployment branches | `main` only |
| Protection rules | None |

Secrets to add (values from dev Supabase project):

| Secret | Value source |
|---|---|
| `SUPABASE_URL_DEV` | Supabase project → Settings → API → Project URL |
| `SUPABASE_ANON_KEY_DEV` | Supabase project → Settings → API → anon / public key |

### `production`

| Setting | Value |
|---|---|
| Name | `production` |
| Deployment branches | `main` only |
| Required reviewers | Add yourself (or the admin) |
| Protection rules | Wait timer: 0 min (reviewers are enough) |

Secrets to add (values from prod Supabase project):

| Secret | Value source |
|---|---|
| `SUPABASE_URL_PROD` | Supabase project → Settings → API → Project URL |
| `SUPABASE_ANON_KEY_PROD` | Supabase project → Settings → API → anon / public key |

---

## 3. Repository Secrets

**Settings → Secrets and variables → Actions → New repository secret**

| Secret | Value source |
|---|---|
| `SUPABASE_ACCESS_TOKEN` | [supabase.com/dashboard/account/tokens](https://supabase.com/dashboard/account/tokens) — create a personal access token |
| `GOOGLE_SERVICE_ACCOUNT_KEY` | Base64-encoded JSON key file from Google Cloud (see `docs/setup/GOOGLE_CLOUD.md`) |
| `VERCEL_TOKEN` | Vercel → Account Settings → Tokens → Create token |
| `VERCEL_ORG_ID` | `vercel env pull` output, or Vercel project settings |
| `VERCEL_PROJECT_ID` | `vercel env pull` output, or Vercel project settings |

> `SUPABASE_URL_DEV/PROD` and `SUPABASE_ANON_KEY_DEV/PROD` are scoped to Environments (step 2), not here.

---

## 4. Labels

The `label-pr.yml` workflow auto-creates labels on first use. You can also create them manually for a cleaner setup:

**Issues → Labels → New label**

| Label | Color |
|---|---|
| `feature` | `#0075ca` |
| `bug` | `#d73a4a` |
| `chore` | `#e4e669` |
| `documentation` | `#0075ca` |
| `hotfix` | `#b60205` |

---

## 5. Vercel Auto-Deploy (disable production auto-deploy)

This is done in Vercel settings, not GitHub — but it's related: GitHub pushes to `main` trigger Vercel Preview deployments automatically (dev environment). Production is only deployed via the `deploy-prod.yml` workflow. See `docs/setup/VERCEL.md` for how to disable the automatic production deployment in Vercel.

---

## 6. General Repository Settings

**Settings → General**

| Setting | Value |
|---|---|
| Default branch | `main` |
| Allow merge commits | ❌ |
| Allow squash merging | ✅ |
| Allow rebase merging | ❌ |
| Automatically delete head branches | ✅ |
| Default commit message (squash) | Pull request title and description |
