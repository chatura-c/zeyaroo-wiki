# Admin Portal — Project Tracking Features

Feature spec for close project tracking in the Zeyaroo admin dashboard.

## Current State

The admin portal (`zeyaroo-admin/`) already has pages for Dashboard, Users, Finance (invoices, payouts, escrow), Disputes (list only), Projects (list only), and System (jobs, events, webhooks). However, several backend endpoints lack frontend pages, and key tracking capabilities are missing.

---

## Feature 1: Email Audit Log

Every email the platform sends vanishes into the void. The admin should be able to see and review all emails sent.

### Backend

- **New model:** `EmailLog`
  - `id` (UUID)
  - `recipient_email` (string)
  - `recipient_user_id` (nullable UUID, FK to users)
  - `template_name` (string — e.g. `InvoiceIssued`, `ProposalSent`)
  - `subject` (string)
  -body_html` (text — full rendered HTML)
  - `status` (string — `sent`, `failed`, `bounced`)
  - `provider` (string — `resend`, `smtp`, `zeptomail`)
  - `provider_message_id` (string — for tracking/bounces)
  - `sent_at` (timestamp)
  - `related_project_id` (nullable UUID)
  - `related_user_id` (nullable UUID — the other party, not the recipient)

- **Hook into existing notification email channel:** In `notification_email_channel.go`, after every send attempt, create an `EmailLog` record. Capture the rendered HTML body, template name, and result.

- **New admin endpoints:**
  - `GET /admin/emails` — paginated list, filterable by recipient, template, project, status, date range
  - `GET /admin/emails/{id}` — full email detail with rendered HTML body

### Frontend

- **New page:** `EmailLog` — table with recipient, template, subject, status, date. Search/filter by recipient email, template type, project, date range.
- **New page:** `EmailDetail` — renders the full HTML email body in an iframe or sandboxed div, plus metadata (recipient, sent at, status, related project link).
- **Sidebar:** Add "Emails" nav item.

---

## Feature 2: Project Detail Page

Backend endpoint `GET /admin/projects/{id}` already exists. Frontend route and page are missing.

### Frontend

- **New page:** `ProjectDetail` at route `/projects/:id`
- **Sections:**
  - Header: project title, status badge, owner (linked to user detail), event date, currency
  - Milestones: list with status, amounts, deadlines, escrow status
  - Media: count and storage used, link to gallery if applicable
  - Linked entities: proposals, escrow transactions, disputes, invoices (all linked from this project)
  - Collaborators: list of users involved

---

## Feature 3: Activity Timeline / Audit Trail

A chronological feed of everything that happened on a project (and optionally per user). This is the core tracking feature.

### Backend

- **New model:** `ActivityLog`
  - `id` (UUID)
  - `project_id` (nullable UUID, FK)
  - `user_id` (nullable UUID, FK — who triggered it)
  - `actor_role` (string — `creator`, `client`, `admin`, `system`)
  - `action` (string — e.g. `project.created`, `proposal.sent`, `milestone.funded`, `payment.released`, `dispute.opened`, `invoice.sent`, `media.uploaded`, `status.changed`)
  - `description` (string — human-readable summary)
  - `metadata` (JSON — arbitrary payload like old/new status, amounts, etc.)
  - `created_at` (timestamp)

- **Emit from existing services:** Instrument the existing service methods to create `ActivityLog` records. Key touchpoints:
  - `ProjectService`: create, status change, archive
  - `ProposalService`: sent, accepted, countered, canceled
  - `EscrowService`: fund, release, refund, dispute open/resolve
  - `BillingService`: invoice created, sent, paid
  - `MediaService`: upload, delete
  - `NotificationService`: email sent, WhatsApp sent

- **New admin endpoints:**
  - `GET /admin/activities` — paginated list, filterable by project, user, action type, date range
  - `GET /admin/projects/{id}/activities` — activities scoped to a project

### Frontend

- **On ProjectDetail page:** Activity timeline component showing chronological feed with icons per action type, actor name, description, and timestamp.
- **Standalone page:** `ActivityLog` — global activity feed with filters (by project, user, action type, date range). Searchable.

---

## Feature 4: Dispute Resolution UI

Backend has `POST /milestones/{id}/resolve` via `Escrow.AdminResolveDispute`. Frontend has `DisputeList` only.

### Frontend

- **New page:** `DisputeDetail` at route `/disputes/:id`
- **Sections:**
  - Dispute info: initiator, role, reason, status, opened date
  - Escrow context: transaction amount, payer, payee, platform fee
  - Milestone context: title, sequence, project link
  - Resolution history (if any): who resolved, resolution text, date
- **Actions:** "Resolve" button with options: release to creator, refund to client. Text field for admin resolution notes.

---

## Feature 5: Notification History per User

The `notifications` table already stores in-app notifications. Admin should be able to view them per user.

### Backend

- **New admin endpoints:**
  - `GET /admin/users/{id}/notifications` — paginated list of notifications for a user

### Frontend

- **On UserDetail page:** New tab or section showing all notifications sent to this user (type, title, body, read status, created date).

---

## Feature 6: Proposal Tracking

Proposals and their version snapshots are stored but have no admin visibility.

### Backend

- **New admin endpoints:**
  - `GET /admin/proposals` — list all proposals with filters (status, project, creator, client)
  - `GET /admin/proposals/{id}` — proposal detail with version history

### Frontend

- **New page:** `ProposalList` — table with proposal ID, project, creator, client, status, version count, last updated.
- **New page:** `ProposalDetail` — version timeline showing each proposal/counter-proposal with amounts, milestones, terms.
- **Sidebar:** Add "Proposals" nav item.

---

## Feature 7: Dashboard Enhancements

The `PlatformStats` type already has `user_growth` and `revenue_by_month` fields, but the backend returns empty arrays.

### Backend

- **Fix `GetStats()`** in `AdminService` to populate:
  - `user_growth`: user count per month for last 12 months
  - `revenue_by_month`: revenue per month for last 12 months

### Frontend

- **Line chart:** User growth over time on Dashboard page
- **Bar chart:** Revenue by month on Dashboard page
- **Recent activity feed:** Last 10-20 activities on the Dashboard (uses the ActivityLog from Feature 3)

---

## Implementation Priority

| Priority | Feature | Effort | Impact |
|----------|---------|--------|--------|
| 1 | Project Detail Page | Small | High — backend exists, just needs frontend |
| 2 | Email Audit Log | Medium | High — solves the "where did that email go" problem |
| 3 | Activity Timeline | Medium | High — gives full project visibility |
| 4 | Dispute Resolution UI | Small | Medium — existing backend, just needs frontend |
| 5 | Dashboard Enhancements | Small | Medium — charts + recent activity |
| 6 | Notification History | Small | Low — simple read from existing table |
| 7 | Proposal Tracking | Medium | Medium — new endpoints + frontend |

Recommended order: 1 → 4 → 5 → 2 → 3 → 6 → 7
