# Google Cloud — Setup Guide

Required for the Google Calendar sync feature. A service account is used for server-to-server authentication (no OAuth user flow).

---

## 1. Create a Google Cloud Project

1. Go to [console.cloud.google.com](https://console.cloud.google.com).
2. Click the project dropdown → **New Project**.
3. Name: `cumbres` (or any name you prefer).
4. Click **Create**.

---

## 2. Enable the Google Calendar API

1. In the project, go to **APIs & Services → Library**.
2. Search for **Google Calendar API**.
3. Click **Enable**.

---

## 3. Create a Service Account

1. Go to **APIs & Services → Credentials → Create Credentials → Service Account**.
2. Fill in:
   - Name: `cumbres-calendar-sync`
   - Description: `Server-side Google Calendar sync for Cumbres`
3. Click **Create and Continue**.
4. Skip the optional role and user access steps → **Done**.

---

## 4. Generate a JSON Key

1. In **APIs & Services → Credentials**, click on the service account you just created.
2. Go to the **Keys** tab → **Add Key → Create new key**.
3. Choose **JSON** → **Create**.
4. A `.json` file is downloaded. **Keep it secure — treat it like a password.**

---

## 5. Encode the Key for Environment Variables

The JSON key must be base64-encoded for storage as an environment variable (avoids issues with multi-line JSON in env vars):

```bash
base64 -i path/to/service-account-key.json | tr -d '\n'
```

Copy the output and add it as:

- **GitHub repository secret**: `GOOGLE_SERVICE_ACCOUNT_KEY`
- **Vercel environment variable**: `GOOGLE_SERVICE_ACCOUNT_KEY` (all environments)

In the app, decode it at runtime:

```ts
const key = JSON.parse(
  Buffer.from(process.env.GOOGLE_SERVICE_ACCOUNT_KEY!, "base64").toString("utf-8")
);
```

---

## 6. Share the Google Calendar with the Service Account

The service account needs access to the specific Google Calendar used for venue bookings.

1. Open [calendar.google.com](https://calendar.google.com) with the Google account that owns the venue calendar.
2. Find the calendar → **⋮ → Settings and sharing**.
3. Scroll to **Share with specific people or groups → Add people**.
4. Add the service account email address (format: `name@project-id.iam.gserviceaccount.com`).
   - Found in the JSON key file as `"client_email"`.
5. Set permission to **Make changes to events**.
6. Click **Send**.

Note the **Calendar ID** (found in **Settings → Integrate calendar → Calendar ID**). It will look like `abc123@group.calendar.google.com` or a plain Gmail address for the primary calendar.

Add it as an environment variable:

| Variable | Value |
|---|---|
| `GOOGLE_CALENDAR_ID` | The calendar ID from the step above |

Add `GOOGLE_CALENDAR_ID` to Vercel environment variables (and `.env.local` for local dev).

---

## 7. Local Dev — Disable Calendar Sync

For local development, Google Calendar sync should be a no-op to avoid polluting the real calendar. The app should check for the env var and skip silently if absent:

```ts
if (!process.env.GOOGLE_SERVICE_ACCOUNT_KEY || !process.env.GOOGLE_CALENDAR_ID) {
  console.warn("Google Calendar sync skipped: credentials not configured");
  return;
}
```

Simply omit `GOOGLE_SERVICE_ACCOUNT_KEY` and `GOOGLE_CALENDAR_ID` from `.env.local` to disable sync locally.

---

## Summary of Environment Variables

| Variable | Where to add |
|---|---|
| `GOOGLE_SERVICE_ACCOUNT_KEY` | Vercel env vars + GitHub repo secret |
| `GOOGLE_CALENDAR_ID` | Vercel env vars (in `.env.local` for local dev with sync enabled) |
