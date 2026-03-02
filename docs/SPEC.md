# Cumbres — Functional Specification (Tech-Agnostic)

> This document describes what the system does and how it behaves, independent of any technology stack.
> It is intended to serve as the basis for rebuilding this system with a different set of tools.

---

## 1. Project Description

**Cumbres** is a private, web-based venue management system for a single event space. It is used exclusively by internal staff — there is no public-facing interface or self-registration.

The system manages the full lifecycle of event reservations: from initial booking through final payment collection, with supporting tools for client records, event package definitions, operating expense tracking, and financial reporting.

**Primary language:** Spanish (Uruguay).
**Timezone:** America/Montevideo.
**Monetary amounts:** Decimal numbers with two decimal places. No currency symbol is displayed (implied local currency).

The application must work on both desktop and mobile browsers and must be installable as a home-screen app on mobile devices (PWA — required, not optional).

---

## 2. Functional Areas

1. Authentication & access control
2. Dashboard (metrics + navigation)
3. Reservations (list, create, view, edit, soft-delete, restore)
4. Clients (list, create, view, edit, delete)
5. Event Types / Packages — "Party Types" (list, create, view, edit, price history, documents, delete)
6. Payments (per reservation: add, edit, delete; includes refund tracking)
7. Expenses (list, create, view, edit, delete, receipt upload, export)
8. Calendar view (with new-reservation shortcut)
9. ~~Automated client messaging — WhatsApp~~ *(deferred to a future phase)*
10. External calendar sync (Google Calendar)
11. Financial report
12. Reservations report
13. Deleted reservations recovery
14. Data export (Excel — reservations and expenses)
15. Shareable links
16. File storage (documents and receipts)
17. User management (Admin only)
18. User profile (all roles)
19. Audit trail (who created/modified each record)

---

## 3. Feature Descriptions

---

### 3.1 Authentication & Access Control

- The application requires login with an email address and password.
- There is no self-registration — staff accounts are created by an administrator.
- All routes except the login page and the password-reset flow require an authenticated session.
- If an unauthenticated user follows a share link, the system stores the intended destination and redirects them to login. After successful login, they are taken to the originally requested page.
- Logout terminates the session and redirects to the login page.

#### Roles

There are two roles: **Admin** and **Staff**.

| Capability | Staff | Admin |
|---|---|---|
| Dashboard | ✓ | ✓ |
| Reservations — list, create, view, edit, soft-delete | ✓ | ✓ |
| Clients — list, create, view, edit, delete | ✓ | ✓ |
| Payments — add, edit, delete | ✓ | ✓ |
| Expenses — list, create, view, edit, delete, export | ✓ | ✓ |
| Calendar view | ✓ | ✓ |
| Export to Excel | ✓ | ✓ |
| Shareable links | ✓ | ✓ |
| User profile (own account) | ✓ | ✓ |
| Event Types — list, view | ✓ | ✓ |
| Event Types — create, edit, delete, manage documents | — | ✓ |
| Expense categories — create, edit, delete | — | ✓ |
| Financial report | — | ✓ |
| Reservations report | — | ✓ |
| Deleted reservations — view & restore | — | ✓ |
| User management — create, edit, deactivate accounts | — | ✓ |

- Every account has exactly one role.
- Role assignment is done by an Admin at account creation time and can be changed later.
- The navigation menu and available actions adapt to the logged-in user's role (restricted items are hidden, not just disabled).

#### First Admin Bootstrap

- On application startup, if the `users` table contains no active Admin accounts, the system automatically creates one using the following environment variables:
  - `BOOTSTRAP_ADMIN_EMAIL` — the email address for the first Admin account
  - `BOOTSTRAP_ADMIN_NAME` — the display name for the first Admin account
  - `BOOTSTRAP_ADMIN_PASSWORD` — the initial password (stored hashed; should be changed after first login)
- If any active Admin account already exists, these variables are ignored — the bootstrap logic does not run.
- If the required variables are missing and no Admin exists, the application should log a clear error and refuse to start.
- This is a one-time operation. Once the first Admin is created via the app's User Management screen, the environment variables can be removed.

#### Forgot Password

- A "Forgot password?" link on the login page initiates a self-service password reset.
- The user enters their email address; the system sends a reset link valid for a limited time (e.g., 1 hour).
- The reset link takes the user to a page where they enter and confirm a new password.
- This flow works for all roles, including Admins, eliminating any single point of failure in account recovery.

---

### 3.2 Dashboard

The dashboard is the landing page after login. It combines quick-navigation shortcuts with at-a-glance operational metrics.

**Metrics displayed:**
- Today's events: list of reservations occurring today (client name, event type, start time if set)
- Upcoming this week: count of pending and confirmed reservations in the next 7 days
- Pending payments: count of confirmed reservations with an outstanding balance (amount due > 0)
- Recent activity: the 5 most recently created or modified reservations

**Navigation shortcuts:** Quick-access cards to the main sections (New Reservation, Clients, Calendar, Expenses).

---

### 3.3 Reservations

Reservations are the central entity of the system. A reservation associates a **client** with an **event type (party type)** on a specific date, with pricing, payment tracking, status, and notes.

#### 3.3.1 Reservation List

- Displays all non-deleted reservations, paginated (20 per page).
- **Default sort:** event date descending (most recent first).
- Each row shows: client name, event date, event type, status badge, payment progress.
- **Filters:**
  - Full-text search by client name (real-time)
  - Date range filter
  - Multi-select filter by status
  - Multi-select filter by event type
- Active filters are shown as removable tags.
- A button to create a new reservation (optionally pre-filled with a date).
- Export all (or filtered) reservations to Excel.

#### 3.3.2 Create Reservation

A single form that creates a reservation and, optionally, records its first payment simultaneously.

| Field | Required | Notes |
|---|---|---|
| Client | Yes | Search/autocomplete by name; must already exist |
| Event Type | Yes | Dropdown; auto-fills Price, Deposit, Guarantee |
| Date | Yes | Date of the event |
| Start Time | No | Optional start time for the event |
| Price | Yes | Pre-filled from event type; editable |
| Deposit | Yes | Required amount to confirm; pre-filled |
| Guarantee | Yes | Security deposit amount; pre-filled |
| Discounts | No | Fixed discount amount; quick-select buttons for 15/20/30/40/50% of price |
| Initial Payment | No | If > 0, creates a payment record automatically |
| Payment Method | Conditional | Visible and required only when Initial Payment > 0; values: cash or bank transfer; default: bank transfer |
| Notes | No | Free text |

**Business rules:**
- Selecting an event type auto-fills Price, Deposit, and Guarantee from the type's current pricing.
- Final price = Price − Discounts.
- If initial payment ≥ Deposit, the reservation is created with status `confirmed`. Otherwise, it starts as `pending`.
- If initial payment > 0, a payment record (type: deposit) is created alongside the reservation.
- **Double-booking warning:** When a date is selected that already has one or more `pending` or `confirmed` reservations, a visible warning is shown listing the conflicting reservations (client name and event type). The user can still proceed and save — the warning does not block submission. Multiple events on the same day are legitimate (e.g., morning and evening slots).

#### 3.3.3 Reservation Detail

Displays all reservation fields plus:
- Client contact information (phone, email, Instagram)
- Payment list: each payment with date, amount, method, type, notes; refunds displayed with a distinct visual treatment (e.g., negative sign, different row color or label)
- Payment progress bar: total paid / final price (as percentage)
- Amount still due: final price − total paid
- Guarantee status: whether the guarantee has been returned to the client, with the date and user who confirmed it
- Audit info: created by/at, last modified by/at
- WhatsApp opt-out status

**Edit mode:** All fields are editable inline, including explicit status selection. The double-booking warning (with conflicting reservation names) applies when the date is changed to one that already has other pending or confirmed reservations.

**Available actions:**
- Save edits
- Add / edit / delete individual payments (via modal)
- Mark guarantee as returned (records timestamp and acting user; action only available on confirmed or cancelled reservations)
- Cancel reservation (confirmation required → sets status to `cancelled`)
- Reopen reservation (available on `cancelled` reservations → sets status back to `pending`; confirmation required)
- Soft-delete reservation (moves it to the deleted reservations page)
- Copy shareable link

#### 3.3.4 Reservation Status Flow

```
pending ──→ confirmed
pending ──→ cancelled
confirmed ──→ cancelled
cancelled ──→ pending
```

- `cancelled` is **not** terminal. A cancelled reservation can be reopened to `pending` (e.g., to correct an accidental cancellation). From `pending`, it can be confirmed again normally.
- **Automatic confirmation:** If a deposit-type payment is added and the total of all deposit payments ≥ `reservation.deposit`, the status is automatically set to `confirmed`.

#### 3.3.5 Soft Delete & Recovery

- Deleting a reservation sets a `deleted` flag and records the timestamp and user. It does not remove data.
- A separate page (Deleted Reservations) lists soft-deleted reservations with deletion metadata and allows restoration.
- **On restoration:** If the reservation had been synced to Google Calendar before deletion, the calendar event is recreated automatically.

---

### 3.4 Clients

Clients are the people or organizations that book the venue.

#### 3.4.1 Client List

- **Default sort:** client name ascending (alphabetical).
- Real-time search by name.
- Each row shows: name, phone, email, Instagram, number of active reservations, last reservation date.

#### 3.4.2 Create / Edit Client

| Field | Required | Validation |
|---|---|---|
| Name | Yes | |
| Phone | No | Uruguayan format (09XXXXXXX) or international (+country code, 8+ digits) |
| Email | No | Valid email format |
| Instagram | No | Free text |
| Notes | No | Internal staff notes (e.g., preferences, observations) |
| Disable WhatsApp by default | No | Boolean; default: false. When true, new reservations for this client are pre-created with `disable_whatsapp_notifications = true`. Staff can still override it per reservation. |

#### 3.4.3 Client Detail

Shows all fields (including notes) and a list of the client's reservations sorted by event date descending. Supports inline editing and deletion.

**Delete restriction:** A client who has active (non-deleted) reservations cannot be deleted.

---

### 3.5 Event Types (Party Types)

Event types define the packages offered by the venue (e.g., "Baby Shower", "Corporate Event"). Each type has a price, a required deposit to confirm, and a security guarantee amount. Price changes are tracked historically.

#### 3.5.1 Event Type List

- **Default sort:** name ascending (alphabetical).
- Real-time name search.
- Each row shows: name, current price, current deposit, current guarantee, active reservation count.

#### 3.5.2 Create / Edit Event Type

| Field | Required | Notes |
|---|---|---|
| Name | Yes | |
| Price | Yes | Decimal |
| Deposit | Yes | Required to confirm a reservation |
| Guarantee | Yes | Security deposit amount, expected to be returned after the event |
| Start Date | Yes | Date from which this pricing is effective |

#### 3.5.3 Event Type Detail

- Inline editing of all fields.
- **Price history:** A new history entry is created whenever price, deposit, or guarantee is modified. The previous values are saved with their effective date range (start_date to the day before the new record's start_date). A table shows all historical pricing.
- **Document attachments:** Files (contracts, menus, etc.) can be uploaded and attached to an event type. Accepted file types: any. Maximum file size: 20 MB. Shows file name, size, and type. Individual documents can be deleted; deleting a document also removes the physical file from storage.

**Delete restriction:** Cannot delete an event type with active reservations.

---

### 3.6 Payment Tracking

Payments belong to a specific reservation. A reservation can have multiple payments.

| Field | Required | Values |
|---|---|---|
| Amount | Yes | Positive decimal (always > 0; the `type` field determines the direction) |
| Date | Yes | Date of payment |
| Method | Yes | `cash` or `bank_transfer` |
| Type | Yes | `deposit` (counts toward confirmation), `regular`, `refund` (money returned to client), or `guarantee` (security deposit collected upfront) |
| Notes | No | Free text |

**Business rules:**
- Total paid = sum of all `deposit` and `regular` payment amounts − sum of all `refund` amounts. `guarantee` payments are **excluded** from this calculation — they are a security deposit, not event income.
- Amount due = final price − total paid.
- Payment progress % = total paid / final price × 100 (capped at 100%, floor at 0%). Guarantee payments do not affect progress.
- Adding a deposit-type payment that brings total deposits ≥ `reservation.deposit` auto-confirms the reservation.
- Refund amounts are always entered as positive numbers — the `refund` type signals that the money flows back to the client.
- Refunds can be recorded at any time, including after a reservation is cancelled — this is the primary use case.
- A refund does not automatically change the reservation status; status changes remain a manual action.
- The system does not block refunds that exceed total paid, but shows a warning when this occurs.
- `guarantee` payments are excluded from the financial report's total income and net income — they appear in the payment list with a distinct visual label.
- The physical return of the guarantee to the client is tracked separately via the reservation's "Mark guarantee as returned" action (which records `guarantee_returned_at` and `guarantee_returned_by`), not as an additional payment record.

Payments are managed via a modal on the Reservation Detail page. Existing payments can be edited or deleted individually.

---

### 3.7 Expenses

Tracks operating costs for the venue.

| Field | Required | Notes |
|---|---|---|
| Date | Yes | |
| Description | Yes | |
| Amount | Yes | Positive decimal |
| Category | No | Selected from a predefined list |
| Related Reservation | No | Links the expense to a specific event |
| Receipt | No | File upload (PDF or image); maximum file size: 10 MB |

**Expense list:**
- **Default sort:** date descending (most recent first).
- Filterable by category (multi-select) and date range.
- Shows total amount for the filtered set.
- Export filtered results to Excel (see Section 3.14).

**Expense detail:** Displays all fields; shows a link/preview for the uploaded receipt if present. Deleting an expense that has a receipt also removes the physical file from storage.

**Default categories (seeded):** Alquiler, Limpieza, Mantenimiento, Servicios, Suministros, Marketing, Seguros, Otros.

---

### 3.8 Calendar View

A monthly calendar showing all reservations as events.

- **Event label:** Client name + event type.
- **Color coding:** Green (confirmed), Orange (pending), Red (cancelled).
- **Navigation:** Previous/next month, and month / week / day view toggles.
- **Interaction — existing event:** Clicking an event navigates to the reservation detail page.
- **Interaction — empty date:** Clicking on an empty date opens the Create Reservation form pre-filled with that date.

---

### 3.9 Automated Client Messaging (WhatsApp)

> **Deferred — not part of the current phase.** This section will be specified in a future iteration.

---

### 3.10 External Calendar Sync (Google Calendar)

Reservation events are automatically mirrored to a configured Google Calendar. No manual action is required.

**Event format:**
- **Title:** Client name + event type name (e.g., "María García — Baby Shower")
- **Description:** Reservation notes (if any) + status (e.g., "Estado: Confirmada")
- **No start time set:** all-day event on the reservation date.
- **Start time set:** timed event starting at the specified time, with a fixed 5-hour duration.

**Sync triggers:**
- **On creation:** A calendar event is created. The returned external event ID is stored on the reservation.
- **On update:** If the reservation has a Google event ID and a relevant field changed (date, start time, status, notes), the calendar event is patched. Irrelevant changes (e.g., updating the sync metadata itself) are ignored.
- **On soft-delete:** The calendar event is deleted.
- **On restore:** If the reservation was synced before deletion, the calendar event is recreated and the new event ID is stored.

**Failure handling:** Sync failures are non-blocking. If the Google Calendar call fails, the reservation is still saved successfully and the error is logged server-side. The sync will be attempted again on the next relevant update to the reservation.

**Reconciliation job (daily, via external cron → secured HTTP endpoint):**
- Runs once per day (same authorization mechanism as any other cron endpoint — shared secret token in `Authorization` header).
- Finds all non-deleted reservations with `status` in `pending` or `confirmed` that either:
  - Have no `google_event_id` (never successfully synced), or
  - Have a `last_synced_at` older than a configurable threshold (default: 24 hours).
- For each such reservation, attempts to create or patch the Google Calendar event.
- Updates `google_event_id` and `last_synced_at` on success; logs failures.
- This ensures reservations that failed their initial sync and were never subsequently updated are eventually reflected in the calendar.

> **Required feature.** Must be fully implemented — not optional.

---

### 3.11 Financial Report (Admin only)

Filtered by a date range (default: current month).

**Displays:**
- Total income: sum of all `deposit` and `regular` payments in the range, minus all `refund` payments in the range
- Total expenses: sum of all expense amounts in the range
- Net income: total income − total expenses
- Charts: payment status breakdown (by count), monthly revenue trend, payment type breakdown (deposit vs. regular vs. refund)
- Itemized payment list for the period (including refunds, visually distinguished)
- Itemized expense list for the period

**Export:** All report data can be exported to Excel.

---

### 3.12 Reservations Report (Admin only)

Filtered by date range. Provides a statistical overview of reservation activity.

**Metrics displayed:**
- Total reservations in range, broken down by status (confirmed, pending, cancelled) with count and percentage
- Reservations by event type: count and total revenue per type
- Reservations by month: trend chart showing volume over time
- Cancellation rate: cancelled / total (%)
- Average payment completion rate: average (total paid / final price) across confirmed reservations
- Average revenue per reservation (final price, excluding cancelled)
- Most popular days of the week for events (bar chart)

---

### 3.13 Data Export (Excel)

#### Reservation Export

Triggered from the Reservation List. Generates a `.xlsx` file with three sheets from all reservations matching the current filters.

**Sheet 1 — Reservations:** client name, phone, email, event type, event date, start time, status, price, deposit, guarantee, discounts, final price, total paid, balance due, payment %, notes, created by (name), created at, last modified by (name), last modified at.

**Sheet 2 — Payments:** all payments for the exported reservations — reservation ID, client name, reservation date, payment date, amount, method (translated), type (translated; refunds clearly labeled), notes.

**Sheet 3 — Summary:** aggregate counts by status, total payment count, total income (net of refunds), export timestamp.

**Filename:** `reservas_{YYYY-MM-DD}.xlsx` or `reservas_filtradas_{YYYY-MM-DD}.xlsx` when filters are active.

#### Expense Export

Triggered from the Expense List. Generates a `.xlsx` file from all expenses matching the current filters (date range, category).

**Columns:** date, description, category, related reservation (client name + date, if linked), amount, receipt (yes/no), created by (name), created at.

**Summary row:** total amount for the exported set, export timestamp.

**Filename:** `gastos_{YYYY-MM-DD}.xlsx` or `gastos_filtrados_{YYYY-MM-DD}.xlsx` when filters are active.

---

### 3.14 Shareable Links

Any reservation, client, or event type can generate a shareable URL. The URL encodes the entity type and its ID. These links are intended for sharing between staff members (e.g., via chat).

- If the recipient is not logged in, they are redirected to login, and then to the intended page after authentication.
- If already logged in, they land directly on the detail page.

---

### 3.15 File Storage

Two categories of files are stored:
1. **Event type documents** — contracts, menus, or other materials attached to an event type. Accepted types: any. Maximum size: 20 MB per file.
2. **Expense receipts** — proof of payment for operating expenses. Accepted types: PDF and images (JPEG, PNG, WebP). Maximum size: 10 MB per file.

Files are stored with auto-generated unique names to prevent collisions. Metadata (name, size, MIME type, URL) is persisted in the database.

**File cleanup on deletion:**
- Deleting an event type document removes both the database record and the physical file from storage.
- Deleting an expense that has a receipt removes both the database record and the physical file from storage.
- Deleting an expense category or soft-deleting a reservation does **not** remove associated expense receipts.

---

### 3.16 User Management (Admin only)

Accessible only to Admin users. Provides a screen to manage all staff accounts.

**User list:** Shows name, email, role, and active/inactive status for every account. Default sort: name ascending.

**Create account:**

| Field | Required | Notes |
|---|---|---|
| Name | Yes | Display name shown in audit trail entries |
| Email | Yes | Used as login credential; must be unique |
| Password | Yes | Set by the Admin at creation time |
| Role | Yes | Admin or Staff |

**Edit account:** Admin can update name, email, role, and password for any account other than their own password (they use the My Profile screen for that).

**Deactivate / reactivate:** An account can be deactivated without being deleted. Deactivated accounts cannot log in. Their audit trail entries (created_by / updated_by) are preserved and continue to show the user's name.

**Constraints:**
- An Admin cannot deactivate their own account.
- There must always be at least one active Admin account.

**Password reset by Admin:** An Admin can set a new password for any other account directly from the edit form (no email required — useful when a staff member is locked out and the forgot-password email is unavailable).

---

### 3.17 User Profile (all roles)

Every logged-in user has access to a "My Profile" screen where they can update their own account without Admin intervention.

**Editable fields:**
- Display name
- Password (requires entering current password to confirm)

Email and role changes are not self-service — they require an Admin action.

---

### 3.18 Audit Trail

Every main entity records who created it and who last modified it, along with timestamps. This information is displayed on each detail page:

- *"Creado por {name} el {date}"*
- *"Modificado por {name} el {date}"*

These fields are set automatically by the system on every write operation — no manual input is required. The user's display name (not email) is used so that audit entries remain readable even if the email changes later.

---

## 4. Data Model

### Entities and Relationships

```
users  (application user accounts)
  - id (unique identifier)
  - name (required — display name used in audit trail)
  - email (required, unique — login credential)
  - password_hash (required)
  - role: 'admin' | 'staff' (required)
  - is_active (boolean, default: true)
  - created_at

clients
  - id
  - name (required)
  - email
  - phone
  - instagram
  - notes (text, nullable — internal staff observations)
  - created_at, updated_at, created_by → users, updated_by → users

party_types  (event packages)
  - id
  - name (required)
  - price (decimal, required)
  - deposit (decimal, required)
  - guarantee (decimal, required)
  - start_date (date, required — pricing effective date)
  - created_at, updated_at, created_by → users, updated_by → users

price_history
  - id
  - party_type_id → party_types
  - price, deposit, guarantee (decimals)
  - start_date (date)
  - end_date (date, nullable — null means currently active)
  - created_at, updated_at, created_by → users, updated_by → users

party_type_documents
  - id
  - party_type_id → party_types
  - name
  - file_url
  - file_size (bytes)
  - mime_type
  - created_at, updated_at, created_by → users, updated_by → users

reservations
  - id
  - client_id → clients (required)
  - party_type_id → party_types (required)
  - date (date, required)
  - start_time (time, nullable)
  - price (decimal, required)
  - deposit (decimal, required)
  - guarantee (decimal, required)
  - discounts (decimal, nullable)
  - notes (text, nullable)
  - status: 'pending' | 'confirmed' | 'cancelled'  (default: pending)
  - deleted (boolean, default: false)
  - deleted_at (timestamp, nullable)
  - deleted_by → users (nullable)
  - guarantee_returned_at (timestamp, nullable — when the guarantee was returned to the client)
  - guarantee_returned_by → users (nullable — who confirmed the return)
  - google_event_id (text, nullable — external calendar event ID)
  - last_synced_at (timestamp, nullable)
  - created_at, updated_at, created_by → users, updated_by → users

payments
  - id
  - reservation_id → reservations (required)
  - amount (decimal, required, > 0 — always a positive number; type field signals direction)
  - date (date, required)
  - method: 'cash' | 'bank_transfer' (required)
  - type: 'deposit' | 'regular' | 'refund' | 'guarantee' (required)
  - notes (text, nullable)
  - created_at, updated_at, created_by → users, updated_by → users

expense_categories
  - id
  - name (required)
  - description (text, nullable)
  - created_at, updated_at, created_by → users, updated_by → users

expenses
  - id
  - date (date, required)
  - description (text, required)
  - amount (decimal, required, > 0)
  - category_id → expense_categories (nullable — SET NULL on category delete)
  - reservation_id → reservations (nullable — SET NULL on reservation soft-delete)
  - receipt_url (text, nullable)
  - created_at, updated_at, created_by → users, updated_by → users

```

> **Note:** `whatsapp_message_templates` and `whatsapp_message_logs` tables are deferred to a future phase.

### Entity Relationship Summary

```
users ──< clients (created_by / updated_by)
users ──< party_types
users ──< reservations (created_by / updated_by / deleted_by / guarantee_returned_by)
users ──< payments
users ──< expenses

clients >──< reservations (one client, many reservations)
party_types >──< reservations (one type, many reservations)
party_types >──< price_history
party_types >──< party_type_documents
reservations >──< payments
reservations >──< expenses (optional link)
expense_categories >──< expenses (optional link)
```

### Key Constraints

| Constraint | Rule |
|---|---|
| Unique email | `users.email` is unique |
| SET NULL | Deleting an expense category sets `expenses.category_id = NULL` |
| SET NULL | Soft-deleting a reservation sets `expenses.reservation_id = NULL` |
| Soft delete | Reservations are never hard-deleted (only `deleted = true`) |
| Client delete block | Clients with active reservations cannot be deleted |
| Party type delete block | Event types with active reservations cannot be deleted |
| Admin continuity | Cannot deactivate the last active Admin account |
| File cleanup | Physical files are removed from storage when their metadata record is deleted |

---

## 5. Access Control

- All data operations require an authenticated session (except the forgot-password flow).
- Data is isolated per installation — all users of the same installation share the same data.
- Role enforcement applies to both UI (restricted items are hidden) and the data layer (operations by unauthorized roles are rejected server-side).
- The User Profile screen (3.17) is accessible to all authenticated roles.
- The Message Log Viewer (3.9.3) is restricted to Admins.

---

## 6. Decisions Log

All open questions from the initial spec have been resolved.

| # | Topic | Decision |
|---|---|---|
| 1 | **User roles** | Two roles: Admin and Staff — permissions table defined in Section 3.1 |
| 2 | **Messaging channel** | WhatsApp (via Meta WhatsApp Cloud API) — deferred to a future phase |
| 3 | **Google Calendar sync** | Required core feature — triggered automatically on reservation changes |
| 4 | **Message scheduling** | External cron service calls a secret-token-secured HTTP endpoint once daily |
| 5 | **Staff account provisioning** | In-app user management screen (Admin only) — Section 3.16 |
| 6 | **Double-booking** | Soft warning — lists conflicting reservations by name but allows saving |
| 7 | **Currency / taxes** | Plain decimal amounts, no currency symbol, no tax logic |
| 8 | **Staff notifications** | None — staff check the app manually |
| 9 | **Cancellation & refunds** | Refund payment type — staff record returns manually; Section 3.6 |
| 10 | **Guarantee tracking** | Guarantee collection recorded as a `guarantee` payment type; excluded from total paid, progress %, and financial income. Physical return tracked via `guarantee_returned_at` flag on the reservation — no additional payment record needed for the return. |
| 11 | **First Admin provisioning** | Env-var bootstrap: on startup, if no active Admin exists, the system creates one from `BOOTSTRAP_ADMIN_EMAIL`, `BOOTSTRAP_ADMIN_NAME`, and `BOOTSTRAP_ADMIN_PASSWORD`. Variables are ignored once an Admin account exists. |
