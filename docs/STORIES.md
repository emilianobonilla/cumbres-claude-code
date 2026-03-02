# Cumbres — User Stories

> **Source:** PRD.md
> **Format:** As a [role], I want to [action], so that [benefit].
> **Roles:** Staff · Admin · Any (all authenticated users)

---

## Epic 1 — Authentication & Access Control

**US-101**
As a staff member, I want to log in with my email and password, so that I can access the system securely.
- Login page is the only public route
- Invalid credentials show an error message
- Successful login redirects to the dashboard

**US-102**
As a staff member, I want to reset my password via a link sent to my email, so that I can regain access without Admin intervention.
- "Forgot password?" link is visible on the login page
- Reset link expires after 1 hour
- User must enter and confirm the new password before it is saved

**US-103**
As any authenticated user following a deep link while logged out, I want to be redirected to login and then forwarded to my original destination, so that I don't lose the context of the link I received.
- System stores the intended URL before redirecting to login
- After successful login, user lands on the originally requested page

**US-104**
As an Admin, I want restricted sections to be hidden from Staff (not just disabled), so that the UI doesn't expose functionality that Staff cannot use.
- Navigation items for Admin-only areas are not rendered for Staff users
- Server-side enforcement rejects unauthorized operations regardless of UI state

---

## Epic 2 — Dashboard

**US-201**
As any authenticated user, I want to see today's events on the dashboard, so that I know at a glance what is happening at the venue right now.
- Shows client name, event type, and start time (if set) for each reservation occurring today

**US-202**
As any authenticated user, I want to see how many reservations are coming up this week, so that I can plan ahead.
- Shows count of pending and confirmed reservations in the next 7 days

**US-203**
As any authenticated user, I want to see how many confirmed reservations have an outstanding balance, so that I know which clients still owe money.
- Shows count of confirmed reservations where amount due > 0

**US-204**
As any authenticated user, I want to see the 5 most recently created or modified reservations, so that I can quickly pick up where I left off.
- Each entry shows enough context to navigate to the reservation

**US-205**
As any authenticated user, I want quick-access cards to key sections (New Reservation, Clients, Calendar, Expenses), so that I can reach the most common actions in one tap.

---

## Epic 3 — Reservations

### List

**US-301**
As a staff member, I want to see a paginated list of all active reservations sorted by event date (most recent first), so that I can browse the full booking history.
- 20 reservations per page
- Each row shows: client name, event date, event type, status badge, payment progress

**US-302**
As a staff member, I want to filter reservations by client name, date range, status, and event type, so that I can quickly find the bookings I'm looking for.
- Full-text search on client name is real-time
- Status and event type filters are multi-select
- Active filters appear as removable tags

### Create

**US-303**
As a staff member, I want to create a new reservation by selecting a client, event type, and date, so that a new booking is recorded in the system.
- Client is selected via search/autocomplete (must already exist)
- Event type selection auto-fills price, deposit, and guarantee
- Date is required; start time is optional

**US-304**
As a staff member, I want to apply a discount when creating a reservation, so that I can reflect any agreed price reduction.
- Discount is a fixed amount; quick-select buttons for 15/20/30/40/50% of price
- Final price = price − discounts

**US-305**
As a staff member, I want to record an initial payment at the same time I create a reservation, so that I don't have to open the payment modal immediately after saving.
- If initial payment ≥ deposit, reservation is created with status `confirmed`
- Payment method is required when initial payment > 0; defaults to bank transfer

**US-306**
As a staff member, I want to be warned (but not blocked) when I select a date that already has other pending or confirmed reservations, so that I'm aware of the overlap while still being able to proceed.
- Warning lists conflicting reservations by client name and event type
- Multiple events on the same day are legitimate; warning is informational only

### Detail & Edit

**US-307**
As a staff member, I want to view all details of a reservation on a single page, so that I have full context without switching screens.
- Shows client contact info, payment list, progress bar, balance due, guarantee status, audit info

**US-308**
As a staff member, I want to edit any reservation field inline, so that I can correct mistakes or update details without navigating away.
- Double-booking warning applies when the date is changed to a conflicting one

### Status Management

**US-309**
As a staff member, I want to cancel a reservation (with a confirmation step), so that I can mark it as no longer active.
- Sets status to `cancelled`
- Confirmation dialog required before the status changes

**US-310**
As a staff member, I want to reopen a cancelled reservation (with a confirmation step), so that I can correct an accidental cancellation.
- Sets status back to `pending`
- From `pending`, the reservation can be confirmed again normally

**US-311**
As a staff member, I want a reservation to be automatically confirmed when deposit payments reach the required threshold, so that I don't have to change the status manually.
- Triggered whenever a deposit-type payment is added or edited
- Auto-confirms if sum of deposit payments ≥ reservation's required deposit

### Guarantee

**US-312**
As a staff member, I want to mark the guarantee as returned to the client, so that there is a record of when the security deposit was handed back.
- Records timestamp and the name of the acting user
- Action available only on confirmed or cancelled reservations

### Soft Delete & Recovery

**US-313**
As a staff member, I want to soft-delete a reservation, so that it is removed from the active list without permanently losing the data.
- Sets `deleted = true`; records timestamp and acting user
- Deleted reservations do not appear in the main list

**US-314**
As an Admin, I want to view all soft-deleted reservations and restore any of them, so that I can recover accidentally deleted bookings.
- Restoration recreates the Google Calendar event if one existed before deletion

### Export

**US-315**
As a staff member, I want to export the current reservation list (with active filters applied) to an Excel file, so that I can share or analyze the data outside the system.
- Generates a `.xlsx` file with three sheets: Reservations, Payments, Summary
- Filename reflects the date and whether filters are active

---

## Epic 4 — Clients

**US-401**
As a staff member, I want to search for clients by name in real time, so that I can find a client's record quickly.
- List is sorted alphabetically by default
- Each row shows: name, phone, email, Instagram, active reservation count, last reservation date

**US-402**
As a staff member, I want to create a new client with their contact details, so that I can link them to reservations.
- Required: name
- Optional: phone (validated as Uruguayan or international format), email (validated format), Instagram, notes

**US-403**
As a staff member, I want to add internal notes to a client record, so that I can capture observations that don't fit into structured fields.
- Notes field is free text and visible only to staff

**US-404**
As a staff member, I want to see all of a client's reservations on their profile page, so that I can review their full history in one view.
- Reservations listed sorted by event date descending

**US-405**
As a staff member, I want the system to prevent me from deleting a client who has active reservations, so that I don't accidentally orphan booking records.
- Delete action is blocked with a clear error message if active reservations exist

---

## Epic 5 — Event Types (Party Types)

**US-501**
As an Admin, I want to create an event type package with a name, price, deposit, guarantee, and effective start date, so that staff have accurate pricing when creating reservations.

**US-502**
As an Admin, I want every change to an event type's price, deposit, or guarantee to be recorded with its effective date range, so that I can audit what pricing was in effect for any past reservation.
- Previous values are saved to price history with start and end dates
- A table of all historical pricing is visible on the event type detail page

**US-503**
As any authenticated user, I want selecting an event type on the reservation form to auto-fill the price, deposit, and guarantee fields, so that I don't have to enter standard amounts manually.

**US-504**
As an Admin, I want to attach documents (contracts, menus, etc.) to an event type, so that related materials are stored alongside the package definition.
- Accepted types: any; maximum file size: 20 MB per file
- Deleting a document also removes the physical file from storage

**US-505**
As an Admin, I want the system to prevent me from deleting an event type that has active reservations, so that I don't break existing bookings.
- Delete action is blocked with a clear error message if active reservations exist

---

## Epic 6 — Payments

**US-601**
As a staff member, I want to add a payment to a reservation specifying amount, date, method, and type, so that all money received is accurately tracked.
- Payment types: deposit, regular, refund, guarantee
- Method: cash or bank transfer

**US-602**
As a staff member, I want to record a guarantee payment separately from event payments, so that the security deposit is tracked but does not inflate the event payment progress.
- Guarantee payments are excluded from total paid, amount due, and payment progress %
- Displayed with a distinct label in the payment list

**US-603**
As a staff member, I want to record a refund after a cancellation, so that money returned to the client is accurately reflected in the system.
- Refund amounts are entered as positive numbers; the `refund` type signals direction
- Refunds can be added at any time, including on cancelled reservations

**US-604**
As a staff member, I want to see a payment progress bar and the remaining balance on the reservation, so that I can immediately gauge how much has been paid.
- Progress % = total paid / final price, capped at 100%
- Balance due = final price − total paid

**US-605**
As a staff member, I want to be warned (but not blocked) when a refund would exceed the total amount paid, so that I'm aware of the unusual situation before saving.

**US-606**
As a staff member, I want to edit or delete individual payment records, so that I can correct data entry mistakes.
- Managed via modal on the reservation detail page

---

## Epic 7 — Expenses

**US-701**
As a staff member, I want to record an operating expense with date, description, amount, and category, so that all venue costs are tracked in one place.

**US-702**
As a staff member, I want to upload a receipt (PDF or image) to an expense, so that proof of payment is stored alongside the record.
- Maximum file size: 10 MB
- Physical file is removed from storage when the expense is deleted

**US-703**
As a staff member, I want to optionally link an expense to a specific reservation, so that event-specific costs are easy to identify.

**US-704**
As a staff member, I want to filter expenses by category and date range and see the total for the filtered set, so that I can quickly subtotal costs for a period or area.

**US-705**
As an Admin, I want to create, edit, and delete expense categories, so that the category list stays relevant to the venue's operations.
- Deleting a category does not delete linked expenses; it sets their category to blank

**US-706**
As a staff member, I want to export filtered expenses to Excel, so that I can share cost data outside the system.
- Filename reflects the date and whether filters are active

---

## Epic 8 — Calendar View

**US-801**
As a staff member, I want to see all reservations on a monthly calendar with color coding (green = confirmed, orange = pending, red = cancelled), so that I get a visual overview of venue activity at a glance.

**US-802**
As a staff member, I want to navigate between months and switch between month, week, and day views, so that I can zoom in or out depending on what I need.

**US-803**
As a staff member, I want to click a reservation on the calendar to go directly to its detail page, so that I can access it without searching.

**US-804**
As a staff member, I want to click an empty date on the calendar to open the new reservation form pre-filled with that date, so that I can start a booking directly from the calendar.

---

## Epic 9 — Google Calendar Sync

**US-901**
As a staff member, I want reservations to be automatically synced to the venue's Google Calendar when they are created or updated, so that the external calendar is always up to date without manual effort.
- Title: `{Client Name} — {Event Type}`
- Description: reservation notes + status
- No start time → all-day event; start time set → timed event with 5-hour duration

**US-902**
As a staff member, I want reservation saves to succeed even if the Google Calendar sync fails, so that a third-party API issue never blocks my work.
- Sync errors are logged server-side
- Sync is retried on the next relevant update to the reservation

**US-903**
As a staff member, I want a Google Calendar event to be deleted when a reservation is soft-deleted and recreated when it is restored, so that the external calendar accurately reflects only active reservations.

**US-904**
As an Admin, I want a daily automated job to find and sync any reservations that were never successfully synced or have a stale sync, so that the calendar eventually reflects all bookings even without manual updates.
- Job is triggered by an external cron via a secret-token-secured endpoint
- Logs successes and failures

---

## Epic 10 — Financial Report

**US-1001**
As an Admin, I want to see total income, total expenses, and net income for a selected date range (defaulting to the current month), so that I can assess the venue's financial performance quickly.
- Income = deposits + regular payments − refunds (guarantee excluded)

**US-1002**
As an Admin, I want to see charts showing payment status breakdown, monthly revenue trend, and payment type breakdown, so that I can spot patterns without exporting data.

**US-1003**
As an Admin, I want to see itemized lists of all payments and expenses for the period, so that I can drill into the detail behind the summary numbers.

**US-1004**
As an Admin, I want to export the financial report to Excel, so that I can share it with stakeholders.

---

## Epic 11 — Reservations Report

**US-1101**
As an Admin, I want to see reservation counts broken down by status (confirmed, pending, cancelled) with percentages for a date range, so that I can understand booking volume and health.

**US-1102**
As an Admin, I want to see how many reservations and how much revenue each event type generated in the period, so that I can evaluate which packages are performing best.

**US-1103**
As an Admin, I want to see a monthly volume trend chart, the cancellation rate, average payment completion, and most popular days of the week, so that I can identify operational patterns and opportunities.

---

## Epic 12 — Data Export

**US-1201**
As a staff member, I want to export the current reservation list to a multi-sheet Excel file (Reservations, Payments, Summary), so that I can do offline analysis or share data with others.
- Export respects active filters
- Filename: `reservas_{YYYY-MM-DD}.xlsx` or `reservas_filtradas_{YYYY-MM-DD}.xlsx`

**US-1202**
As a staff member, I want to export the current expense list to an Excel file with a summary row, so that I can share cost data outside the system.
- Export respects active filters
- Filename: `gastos_{YYYY-MM-DD}.xlsx` or `gastos_filtrados_{YYYY-MM-DD}.xlsx`

---

## Epic 13 — Shareable Links

**US-1301**
As a staff member, I want to copy a shareable link to any reservation, client, or event type, so that I can send it to a colleague via chat.

**US-1302**
As a staff member receiving a shared link while logged out, I want to be redirected to login and then forwarded to the intended page, so that I land directly on the right record after authenticating.

---

## Epic 14 — User Management

**US-1400**
As a system operator deploying the application for the first time, I want the first Admin account to be created automatically from environment variables on startup, so that I can access the system without needing a pre-existing account or running manual database commands.
- Reads `BOOTSTRAP_ADMIN_EMAIL`, `BOOTSTRAP_ADMIN_NAME`, and `BOOTSTRAP_ADMIN_PASSWORD` from environment variables
- Creates the Admin account only if no active Admin currently exists; does nothing otherwise
- If the variables are missing and no Admin exists, the application logs a clear error and refuses to start
- Password is stored hashed; the operator should change it after first login via the User Profile screen

**US-1401**
As an Admin, I want to create new staff accounts specifying name, email, password, and role, so that new employees can access the system immediately without waiting for email provisioning.

**US-1402**
As an Admin, I want to edit any staff account's name, email, role, or password, so that I can keep account information up to date.

**US-1403**
As an Admin, I want to deactivate an account without deleting it, so that a departing employee loses access while their name continues to appear correctly in audit trail entries.
- Deactivated accounts cannot log in

**US-1404**
As an Admin, I want to set a new password for any other account directly from the admin panel, so that I can help a locked-out staff member without relying on email.

**US-1405**
As an Admin, I want the system to prevent me from deactivating the last active Admin account, so that the system is never left without an administrator who can log in.

---

## Epic 15 — User Profile

**US-1501**
As any authenticated user, I want to update my own display name, so that my name appears correctly across the system.

**US-1502**
As any authenticated user, I want to change my own password by confirming my current password first, so that I can maintain my account security without Admin involvement.

---

## Epic 16 — Audit Trail

**US-1601**
As any authenticated user, I want to see who created and who last modified each record (with timestamps), so that I can trace changes and know who to follow up with.
- Displayed on each detail page: "Creado por {name} el {date}" / "Modificado por {name} el {date}"
- Set automatically on every write; no manual input required
- Uses display name so entries remain readable even if a user's email changes later
