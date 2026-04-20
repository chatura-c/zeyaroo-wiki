# Zeyaroo Admin Dashboard — Implementation Plan

## Overview

Separate repo (`chatura-c/zeyaroo-admin`) — a standalone admin dashboard for the Zeyaroo platform.
Deployed independently (e.g. `admin.zeyaroo.com` with restricted access).
Dedicated JWT-only auth (no Firebase). Calls the same phto-api backend.

## Tech Stack

| Layer | Choice |
|-------|--------|
| Build | Vite 7 |
| Framework | React 19 + TypeScript 5.9 |
| Routing | React Router v7 |
| Data fetching | Axios + TanStack Query v5 |
| State | Zustand v5 (auth store) |
| UI | shadcn/ui + Tailwind CSS v4 |
| Charts | Recharts |
| Icons | Lucide React |
| Forms | React Hook Form + Zod |
| Tables | TanStack Table (via shadcn Data Table) |
| Date | date-fns |

## Project Structure

```
zeyaroo-admin/
├── public/
│   └── favicon.svg
├── src/
│   ├── api/                    # Axios client + TanStack Query hooks
│   │   ├── client.ts           # Axios instance with JWT interceptor
│   │   ├── queryClient.ts      # Shared QueryClient
│   │   ├── users.ts            # User management hooks
│   │   ├── finance.ts          # Invoice/escrow/payout hooks
│   │   ├── disputes.ts         # Dispute hooks
│   │   ├── projects.ts         # Project overview hooks
│   │   ├── stats.ts            # Dashboard stats hooks
│   │   └── system.ts           # Jobs/events/webhook hooks
│   ├── components/
│   │   ├── ui/                 # shadcn/ui components
│   │   ├── layout/             # Sidebar, Header, AppShell
│   │   ├── data-table/         # Reusable DataTable with filters
│   │   └── charts/             # Revenue, users, storage charts
│   ├── hooks/
│   │   └── use-auth.ts         # Auth hook from Zustand store
│   ├── pages/
│   │   ├── Login.tsx
│   │   ├── Dashboard.tsx       # Overview metrics + charts
│   │   ├── users/
│   │   │   ├── UserList.tsx
│   │   │   └── UserDetail.tsx
│   │   ├── finance/
│   │   │   ├── FinanceOverview.tsx
│   │   │   ├── InvoiceList.tsx
│   │   │   ├── InvoiceDetail.tsx
│   │   │   ├── PayoutList.tsx
│   │   │   └── EscrowList.tsx
│   │   ├── disputes/
│   │   │   ├── DisputeList.tsx
│   │   │   └── DisputeDetail.tsx
│   │   ├── projects/
│   │   │   ├── ProjectList.tsx
│   │   │   └── ProjectDetail.tsx
│   │   └── system/
│   │       ├── JobQueue.tsx
│   │       ├── EventLog.tsx
│   │       └── WebhookSources.tsx
│   ├── stores/
│   │   └── authStore.ts        # Zustand persisted auth
│   ├── types/
│   │   └── index.ts            # API response types
│   ├── config/
│   │   └── app.ts              # Env config (API_URL)
│   ├── App.tsx                 # Router + providers
│   ├── main.tsx
│   └── index.css               # Tailwind + shadcn theme
├── components.json             # shadcn config
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
├── vite.config.ts
├── package.json
├── .env.defaults
└── .env.local                  # gitignored
```

## Backend API Changes (phto-api)

All new admin endpoints under `/api/v1/admin/` with `RequireRole("admin")` middleware.

### New Endpoints (need implementation in phto-api)

| # | Method | Path | Purpose | Progress |
|---|--------|------|---------|----------|
| 1 | `GET` | `/admin/stats` | Platform overview (user counts, revenue, storage, active projects) | ✅ |
| 2 | `GET` | `/admin/users` | List all users (paginated, searchable, filterable) | ✅ |
| 3 | `GET` | `/admin/users/{id}` | Full user detail (profile, subscription, storage, project count) | ✅ |
| 4 | `PATCH` | `/admin/users/{id}` | Update user (role, status) | ✅ |
| 5 | `GET` | `/admin/invoices` | List all invoices across users | ✅ |
| 6 | `GET` | `/admin/invoices/{id}` | View any invoice detail | ✅ |
| 7 | `PATCH` | `/admin/invoices/{id}/status` | Update invoice status (finalize, cancel) | ✅ |
| 8 | `GET` | `/admin/payout-requests` | List all payout requests | ✅ |
| 9 | `PATCH` | `/admin/payout-requests/{id}` | Approve/reject payout (with AdminNote) | ✅ |
| 10 | `GET` | `/admin/escrow/transactions` | List all escrow transactions | ✅ |
| 11 | `GET` | `/admin/disputes` | List all disputes (filterable by status) | ✅ |
| 12 | `GET` | `/admin/projects` | List all projects (paginated, filterable) | ✅ |
| 13 | `GET` | `/admin/projects/{id}` | View any project detail | ✅ |
| 14 | `GET` | `/admin/storage/stats` | Platform storage usage summary | ✅ |
| 15 | `GET` | `/admin/jobs` | Background job queue status | ✅ |
| 16 | `GET` | `/admin/events` | Stored event log | ✅ |

### Existing Endpoints (reuse as-is)

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/auth/login` | JWT login (used for admin auth) |
| `PATCH` | `/admin/users/{id}/tier` | Set user subscription tier |
| `GET` | `/admin/users/{id}/package` | Get user package/limits |
| `PUT` | `/admin/users/{id}/package` | Update user package overrides |
| `POST` | `/milestones/{id}/resolve` | Resolve milestone disputes |
| CRUD | `/webhooks/sources/*` | Webhook source management |
| `PUT` | `/users/{id}/payment-methods` | Admin: set user payment methods |
| `POST` | `/billing/invoices/generate` | Admin: generate invoice |

---

## Implementation Phases

### Phase 1 — Scaffold + Auth + Dashboard

| # | Task | Status |
|---|------|--------|
| 1.1 | Initialize Vite project (React 19, TypeScript) | ✅ |
| 1.2 | Install dependencies (TanStack Query, Zustand, Axios, React Router, Tailwind v4, date-fns) | ✅ |
| 1.3 | Initialize shadcn/ui + add base components (Button, Card, Input, Table, Dialog, Badge, Select, DropdownMenu, Avatar, Separator, Tabs) | ✅ |
| 1.4 | Configure Tailwind v4 theme (dark mode primary, color tokens) | ✅ |
| 1.5 | Create config/app.ts (API_URL from env) | ✅ |
| 1.6 | Create api/client.ts (Axios instance + JWT interceptor + 401 redirect) | ✅ |
| 1.7 | Create api/queryClient.ts (staleTime, retry defaults) | ✅ |
| 1.8 | Create types/index.ts (User, Invoice, EscrowTransaction, Dispute, PayoutRequest, Project, Milestone, Job, StoredEvent, Stats) | ✅ |
| 1.9 | Create stores/authStore.ts (Zustand persist: token, user, isAuthenticated, login, logout) | ✅ |
| 1.10 | Create Login page (email/password form, role=admin gate, error states) | ✅ |
| 1.11 | Create ProtectedRoute component (check isAuthenticated) | ✅ |
| 1.12 | Create layout components (AppShell, Sidebar, Header) | ✅ |
| 1.13 | Set up App.tsx routing (React Router v7, nested routes with layout) | ✅ |
| 1.14 | Create api/stats.ts (useAdminStats hook) | ✅ |
| 1.15 | Create Dashboard page (metric cards for users, projects, revenue, escrow, disputes) | ✅ |
| 1.16 | Add placeholder chart components (Recharts — user growth line, revenue bar, project status donut) | ⬜ (deferred — charts need real data) |
| 1.17 | Implement GET /admin/stats backend endpoint in phto-api | ✅ |

### Phase 2 — User Management

| # | Task | Status |
|---|------|--------|
| 2.1 | Implement GET /admin/users backend (paginated, search, filter by role/tier) | ✅ |
| 2.2 | Implement GET /admin/users/{id} backend (full detail with associated data) | ✅ |
| 2.3 | Implement PATCH /admin/users/{id} backend (role, status updates) | ✅ |
| 2.4 | Create api/users.ts (useUsers, useUser, useUpdateUser, useSetTier, useUpdatePackage hooks) | ✅ |
| 2.5 | Create reusable DataTable component (TanStack Table, server-side pagination, sorting, search input) | ⬜ (using simple Table for now) |
| 2.6 | Create UserList page (data table with search, filters for role/tier, row click to detail) | ✅ |
| 2.7 | Create UserDetail page (profile card, subscription info, storage usage, project count, actions) | ✅ |
| 2.8 | Wire up Change Tier dialog (PATCH /admin/users/{id}/tier) | ✅ |
| 2.9 | Wire up Edit Package dialog (GET/PUT /admin/users/{id}/package) | ✅ |

### Phase 3 — Financial Dashboard

| # | Task | Status |
|---|------|--------|
| 3.1 | Implement GET /admin/invoices backend (paginated, filter by status/user) | ✅ |
| 3.2 | Implement GET /admin/invoices/{id} backend (full invoice detail with line items) | ✅ |
| 3.3 | Implement PATCH /admin/invoices/{id}/status backend (finalize, cancel) | ✅ |
| 3.4 | Implement GET /admin/payout-requests backend (paginated, filter by status) | ✅ |
| 3.5 | Implement PATCH /admin/payout-requests/{id} backend (approve/reject with admin note) | ✅ |
| 3.6 | Implement GET /admin/escrow/transactions backend (paginated, filter by status) | ✅ |
| 3.7 | Create api/finance.ts (useInvoices, useInvoice, useUpdateInvoiceStatus, usePayouts, useUpdatePayout, useEscrowTransactions hooks) | ✅ |
| 3.8 | Create FinanceOverview page (revenue cards, monthly revenue chart, escrow held, pending payouts) | ✅ |
| 3.9 | Create InvoiceList page (data table, status filter, date range, link to detail) | ⬜ (merged into FinanceOverview) |
| 3.10 | Create InvoiceDetail page (invoice info, line items, status change action) | ⬜ |
| 3.11 | Create PayoutList page (data table, approve/reject actions in row) | ⬜ (merged into FinanceOverview) |
| 3.12 | Create EscrowList page (data table, status filter) | ⬜ (merged into FinanceOverview) |

### Phase 4 — Disputes + Projects

| # | Task | Status |
|---|------|--------|
| 4.1 | Implement GET /admin/disputes backend (paginated, filter by status) | ✅ |
| 4.2 | Implement GET /admin/projects backend (paginated, filter by status, search) | ✅ |
| 4.3 | Implement GET /admin/projects/{id} backend (full project detail) | ✅ |
| 4.4 | Create api/disputes.ts (useDisputes hook) | ✅ |
| 4.5 | Create api/projects.ts (useAdminProjects, useAdminProject hooks) | ✅ |
| 4.6 | Create DisputeList page (data table, status filter, link to detail) | ✅ |
| 4.7 | Create DisputeDetail page (dispute info, milestone context, escrow amount, resolve action: release/refund via POST /milestones/{id}/resolve) | ⬜ |
| 4.8 | Create ProjectList page (data table, status filter, search) | ⬜ |
| 4.9 | Create ProjectDetail page (project info, milestones, media count, collaborators) | ⬜ |

### Phase 5 — System Health

| # | Task | Status |
|---|------|--------|
| 5.1 | Implement GET /admin/jobs backend (paginated, filter by status) | ✅ |
| 5.2 | Implement GET /admin/events backend (paginated, filter by status/event type) | ✅ |
| 5.3 | Create api/system.ts (useJobs, useEvents, useWebhookSources hooks) | ✅ |
| 5.4 | Create JobQueue page (data table, status filter, retry count, error details) | ✅ (merged into SystemPages) |
| 5.5 | Create EventLog page (data table, event type filter, payload viewer) | ✅ (merged into SystemPages) |
| 5.6 | Create WebhookSources page (existing CRUD, list/create/edit/delete) | ⬜ |
| 5.7 | Add storage stats to Dashboard page (GET /admin/storage/stats integration) | ⬜ |

---

## Design Decisions

- **Auth**: Dedicated JWT-only login. No Firebase. Admin app calls `/auth/login`, verifies `role=admin` in response, rejects non-admin users.
- **API proxy**: In development, Vite dev server proxies `/api` to phto-api (localhost:8080). In production, Caddy routes `admin.zeyaroo.com/api` to the API.
- **Dark mode only**: Admin dashboard uses a dark theme (zinc/slate base). No light mode toggle.
- **Responsive**: Works on desktop (primary) and tablet. Mobile is secondary.

## Environment Config

### zeyaroo-admin (frontend)

```bash
# .env.defaults
VITE_API_URL=http://localhost:8080/api/v1
VITE_APP_NAME=Zeyaroo Admin
```

```bash
# .env.local (gitignored)
VITE_API_URL=http://localhost:8080/api/v1
```

### phto-api (backend — admin seed)

Add these env vars to auto-seed an admin user on startup:

```bash
ADMIN_EMAIL=admin@zeyaroo.com
ADMIN_PASSWORD=<your-secure-password>
```

If both are set, the server will:
1. Check if any admin user already exists — if yes, skip
2. Check if a user with that email already exists — if yes, promote them to admin
3. Otherwise, create a new admin user with that email/password