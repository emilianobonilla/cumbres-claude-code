# Cumbres — Product Requirements Document

> **Document type:** Product Requirements Document (PRD)
> **Source:** SPEC.md — Cumbres Functional Specification
> **Date:** 2026-02-28
> **Status:** Draft

---

## 1. Overview

### 1.1 Product Summary

**Cumbres** is a private, internal web application for managing a single event venue. It covers the complete lifecycle of a booking — from initial reservation through payment collection, client communication, and financial reporting — replacing manual tracking with a unified, role-aware system.

### 1.2 Problem Statement

Venue staff currently lack a dedicated system to:
- Track reservations and their payment status in one place
- Avoid double-booking without a shared calendar
- Automate client communication (reminders via WhatsApp)
- Produce accurate financial and operational reports

Manual processes (spreadsheets, messaging apps) introduce errors, create blind spots, and slow down day-to-day operations.

### 1.3 Goal

Provide a single, mobile-friendly web application that eliminates manual coordination overhead, reduces payment errors, and gives venue management real-time visibility into operations.

### 1.4 Non-Goals

- Public-facing booking portal — staff only
- Multi-venue support
- Online payment processing (payments are recorded, not collected)
- Customer-facing accounts or self-service

---

## 2. Users

### 2.1 Target Users

All users are internal venue staff. There is no public registration.

| Role | Description |
|---|---|
| **Admin** | Full access — manages accounts, event types, WhatsApp templates, and views financial/reservations reports. There must always be at least one active Admin. |
| **Staff** | Operational access — manages reservations, clients, payments, and expenses. Cannot access financial reports, user management, or WhatsApp template configuration. |

### 2.2 Context

- **Language:** Spanish (Uruguay)
- **Timezone:** America/Montevideo
- **Currency:** Local (no symbol displayed; two decimal places)
- **Device:** Desktop and mobile browsers; must support PWA installation on mobile

---

## 3. Functional Requirements

### 3.1 Authentication & Access Control

**Requirements:**
- Email + password login; no self-registration
- Two roles: Admin and Staff with distinct permission sets
- Forgot password flow via time-limited email link (valid 1 hour)
- Session-aware redirect: unauthenticated users following a deep link are sent to login and forwarded to the original destination after authentication
- All non-auth routes require a valid session
- Role restrictions enforced server-side and reflected in the UI (restricted items hidden, not disabled)

**Priority:** P0 — Prerequisite for all other features

---

### 3.2 Dashboard

**Requirements:**
- Landing page after login
- Displays:
  - Today's events (client name, event type, start time)
  - Upcoming events this week (count of pending + confirmed)
  - Pending payments (count of confirmed reservations with outstanding balance)
  - Recent activity (last 5 created or modified reservations)
  - Quick-access navigation cards: New Reservation, Clients, Calendar, Expenses

**Priority:** P0

---

### 3.3 Reservations

**Requirements:**
- Paginated list (20/page), sortable by event date descending by default
- Filters: full-text search by client name, date range, status (multi-select), event type (multi-select)
- Active filters shown as removable tags
- Create form with optional initial payment; auto-confirms if initial payment ≥ deposit
- Status flow: `pending → confirmed`, `pending → cancelled`, `confirmed → cancelled`, `cancelled → pending`; no state is terminal
- Reopen action available on cancelled reservations (confirmation required → sets status back to `pending`)
- Automatic confirmation when deposit-type payments total ≥ reservation deposit amount
- Double-booking warning (non-blocking) when a date with existing pending/confirmed reservations is selected; multiple events on the same day are legitimate
- Detail page with inline editing, payment list, progress bar, balance due, guarantee status, and audit info
- Guarantee return tracking (timestamp + acting user)
- Soft delete with full recovery (restores Google Calendar event if previously synced)
- Export to Excel (.xlsx, 3 sheets: Reservations, Payments, Summary)
- Shareable link per reservation

**Priority:** P0

---

### 3.4 Clients

**Requirements:**
- List with real-time name search, sorted alphabetically
- Each row: name, phone, email, Instagram, active reservation count, last reservation date
- Create/edit with validation: phone (Uruguayan or international format), email
- Client notes field (internal staff observations)
- Detail view with linked reservations sorted by event date descending
- Delete blocked if client has active reservations
- Shareable link per client

**Priority:** P0

---

### 3.5 Event Types (Party Types)

**Requirements:**
- List with real-time name search and active reservation count per type
- Create/edit: name, price, deposit, guarantee, start date (pricing effective date)
- Auto-fill pricing fields on reservation form when event type is selected
- Price history: every price/deposit/guarantee change creates a history entry with effective date ranges
- Document attachments (any file type, max 20 MB each): upload, list, delete; physical file removed on record deletion
- Delete blocked if event type has active reservations
- Shareable link per event type
- Create/edit/delete restricted to Admin

**Priority:** P0

---

### 3.6 Payments

**Requirements:**
- Payments belong to a reservation; multiple payments per reservation
- Fields: amount (positive decimal), date, method (cash / bank transfer), type (deposit / regular / refund / guarantee), notes
- `guarantee` type = security deposit collected upfront; excluded from total paid, progress %, and financial income; displayed with a distinct label
- Total paid = deposits + regular − refunds (`guarantee` excluded)
- Amount due = final price − total paid
- Progress % = total paid / final price, capped at [0%, 100%]; guarantee payments do not affect progress
- Adding a deposit payment that brings deposit total ≥ required deposit auto-confirms the reservation
- Refunds entered as positive amounts; `refund` type signals direction
- Refunds recordable on cancelled reservations (primary use case)
- Warning shown when refund would exceed total paid; not blocked
- Guarantee return tracked via "Mark guarantee as returned" action on the reservation (not a payment record)
- Managed via modal on reservation detail page

**Priority:** P0

---

### 3.7 Expenses

**Requirements:**
- Track venue operating costs: date, description, amount, category, optional linked reservation, optional receipt
- Receipt upload: PDF or image (JPEG, PNG, WebP), max 10 MB; physical file removed when expense is deleted
- List with date-range and category (multi-select) filters; shows total for filtered set
- Default seeded categories: Alquiler, Limpieza, Mantenimiento, Servicios, Suministros, Marketing, Seguros, Otros
- Expense categories: create, edit, delete (Admin only); deleting a category sets `category_id = NULL` on linked expenses
- Export filtered results to Excel

**Priority:** P0

---

### 3.8 Calendar View

**Requirements:**
- Monthly view showing all reservations as color-coded events:
  - Green = confirmed, Orange = pending, Red = cancelled
- Event label: client name + event type
- Navigation: previous/next month, and month/week/day view toggles
- Click existing event → navigate to reservation detail
- Click empty date → open Create Reservation form pre-filled with that date

**Priority:** P1

---

### 3.9 Automated WhatsApp Messaging

> **Deferred — not part of the current phase.**

---

### 3.10 Google Calendar Sync

**Requirements:**
- Required core feature — must be fully implemented
- Sync is automatic; no manual trigger
- Event format:
  - Title: `{Client Name} — {Event Type}`
  - Description: reservation notes + status
  - No start time → all-day event; start time set → timed event with 5-hour duration
- Sync triggers: reservation created, updated (relevant fields only), soft-deleted, restored
- Failure is non-blocking: reservation saves successfully; error logged; retry on next relevant update
- Stores `google_event_id` on the reservation for subsequent patches/deletes
- **Reconciliation job (daily, via external cron → secured HTTP endpoint):** finds reservations without a `google_event_id` or with a stale `last_synced_at`, and creates/patches their calendar events; logs failures

**Priority:** P0 (marked as required)

---

### 3.11 Financial Report (Admin only)

**Requirements:**
- Filtered by date range (default: current month)
- Displays: total income (deposits + regular − refunds), total expenses, net income
- Charts: payment status breakdown, monthly revenue trend, payment type breakdown
- Itemized payment and expense lists for the period
- Export to Excel

**Priority:** P1

---

### 3.12 Reservations Report (Admin only)

**Requirements:**
- Filtered by date range
- Metrics:
  - Total reservations by status with counts and percentages
  - Reservations by event type (count + revenue)
  - Volume trend chart by month
  - Cancellation rate
  - Average payment completion rate (confirmed reservations)
  - Average revenue per reservation (excluding cancelled)
  - Most popular days of the week (bar chart)

**Priority:** P1

---

### 3.13 Data Export (Excel)

**Requirements:**
- **Reservation export** (.xlsx, 3 sheets): triggered from Reservation List, respects active filters
  - Sheet 1: full reservation data including pricing, payments summary, audit fields
  - Sheet 2: all individual payments for exported reservations
  - Sheet 3: aggregate summary + export timestamp
  - Filename: `reservas_{YYYY-MM-DD}.xlsx` / `reservas_filtradas_{YYYY-MM-DD}.xlsx`
- **Expense export** (.xlsx): triggered from Expense List, respects active filters
  - Columns: date, description, category, linked reservation, amount, receipt flag, audit fields
  - Summary row: total + export timestamp
  - Filename: `gastos_{YYYY-MM-DD}.xlsx` / `gastos_filtrados_{YYYY-MM-DD}.xlsx`

**Priority:** P1

---

### 3.14 Shareable Links

**Requirements:**
- Generate a URL encoding entity type + ID for any reservation, client, or event type
- Unauthenticated recipients redirected to login then forwarded to the intended page
- Intended for staff-to-staff sharing (e.g., via chat)

**Priority:** P2

---

### 3.15 File Storage

**Requirements:**
- Two categories: event type documents (any type, max 20 MB) and expense receipts (PDF/image, max 10 MB)
- Auto-generated unique filenames to prevent collisions
- Metadata (name, size, MIME type, URL) stored in the database
- Physical file removed from storage when the associated database record is deleted
- Soft-deleting a reservation does NOT remove linked expense receipts

**Priority:** P0 (enables event type documents and expense receipts)

---

### 3.16 User Management (Admin only)

**Requirements:**
- List: name, email, role, active status; sorted by name ascending
- Create: name, email, password, role (Admin or Staff)
- Edit: name, email, role, password for any account except own password (handled in My Profile)
- Deactivate / reactivate accounts (deactivated accounts cannot log in; audit trail entries preserved)
- Admin cannot deactivate their own account
- System must always have at least one active Admin
- Admin can set a new password for any other account directly (no email required — useful for lockouts)

**Priority:** P0

---

### 3.17 User Profile (all roles)

**Requirements:**
- Every authenticated user can update their own display name
- Password change requires confirming the current password
- Email and role changes require Admin action

**Priority:** P1

---

### 3.18 Audit Trail

**Requirements:**
- Every main entity records: created_by (user), created_at, updated_by (user), updated_at
- Displayed on each detail page in Spanish ("Creado por {name} el {date}")
- Set automatically on every write; no manual input
- Uses display name (not email) so entries remain readable if email changes

**Priority:** P0

---

## 4. Data Model

### Core Entities

| Entity | Key Fields |
|---|---|
| `users` | id, name, email, password_hash, role, is_active |
| `clients` | id, name, phone, email, instagram, notes |
| `party_types` | id, name, price, deposit, guarantee, start_date |
| `price_history` | id, party_type_id, price, deposit, guarantee, start_date, end_date |
| `party_type_documents` | id, party_type_id, name, file_url, file_size, mime_type |
| `reservations` | id, client_id, party_type_id, date, start_time, price, deposit, guarantee, discounts, notes, status, deleted, guarantee_returned_*, google_event_id, last_synced_at |
| `payments` | id, reservation_id, amount, date, method, type, notes |
| `expense_categories` | id, name, description |
| `expenses` | id, date, description, amount, category_id, reservation_id, receipt_url |
| `whatsapp_message_templates` | *(deferred)* |
| `whatsapp_message_logs` | *(deferred)* |

### Key Constraints

| Constraint | Rule |
|---|---|
| `users.email` | Unique |
| `whatsapp_message_logs (reservation_id, template_id, scheduled_for)` | Unique (deduplication) |
| Reservation delete | Soft delete only (`deleted = true`); hard delete never |
| Cascade on reservation hard delete | `whatsapp_message_logs` cascade |
| Cascade on template delete | `whatsapp_message_logs` cascade |
| Category delete | Sets `expenses.category_id = NULL` |
| Reservation soft-delete | Sets `expenses.reservation_id = NULL` |
| Client delete | Blocked if client has active reservations |
| Party type delete | Blocked if event type has active reservations |
| Admin continuity | Cannot deactivate last active Admin |
| File cleanup | Physical file removed when metadata record is deleted |

---

## 5. Non-Functional Requirements

| Category | Requirement |
|---|---|
| **Language** | All UI copy in Spanish (Uruguay) |
| **Timezone** | America/Montevideo for all date/time display and logic |
| **Responsive design** | Works on desktop and mobile browsers |
| **PWA** | Required — must be installable as a home-screen app on mobile devices (not optional) |
| **Monetary precision** | Two decimal places; no currency symbol |
| **WhatsApp endpoint security** | Cron-called endpoint requires shared secret in `Authorization` header; 401 on failure |
| **File size limits** | Event type documents: 20 MB max; expense receipts: 10 MB max |
| **Google Calendar sync** | Non-blocking — reservation saves must not fail due to calendar sync errors |
| **Role enforcement** | Server-side for all operations; UI hides restricted items |

---

## 6. Feature Priorities

| Priority | Features |
|---|---|
| **P0 — Core** | Authentication, Dashboard, Reservations (full CRUD + status flow), Clients, Event Types, Payments, Expenses, File Storage, Google Calendar Sync, User Management, Audit Trail |
| **P1 — Important** | Calendar View, Financial Report, Reservations Report, Data Export (Excel), User Profile |
| **Deferred** | WhatsApp Messaging (future phase) |
| **P2 — Nice to have** | Shareable Links |

---

## 7. Open Questions

All questions from the original specification have been resolved. See SPEC.md Section 6 (Decisions Log) for the full record.

| # | Topic | Decision |
|---|---|---|
| 1 | User roles | Two roles: Admin and Staff |
| 2 | Messaging channel | WhatsApp (Meta WhatsApp Cloud API) — deferred to a future phase |
| 3 | Google Calendar sync | Required core feature |
| 4 | Message scheduling | External cron → secret-token-secured HTTP endpoint |
| 5 | Staff account provisioning | In-app user management (Admin only) |
| 6 | Double-booking | Soft warning — lists conflicts, does not block save |
| 7 | Currency / taxes | Plain decimal amounts; no currency symbol; no tax logic |
| 8 | Staff notifications | None — staff check the app manually |
| 9 | Cancellation & refunds | `refund` payment type; manual status changes |

---

## 8. Success Metrics

| Metric | Target |
|---|---|
| Zero double-bookings confirmed without a warning | Double-booking warning shown on 100% of conflicting date selections |
| Payment tracking accuracy | Amount due always matches final price − recorded payments |
| Google Calendar parity | Calendar reflects all active reservations within one sync cycle |
| Staff onboarding | New staff accounts created and active within minutes (no email provisioning) |
