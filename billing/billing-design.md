# Zeyaroo — Billing & Subscription Design

## Overview

Zeyaroo has two revenue streams:

1. **Subscription fees** — monthly recurring from Pro and Studio creators
2. **Platform fees** — a % cut on every milestone payment, regardless of whether the client paid online or offline

Both are collected from the **creator**. Clients only ever pay milestone amounts; Zeyaroo's fee is the creator's cost of using the platform.

---

## Subscription Tiers

Standard tiers are global templates. Custom plans (see below) can override any of these values per user.

| Tier      | Storage | Max Projects | Escrow Fee | Offline Fee | Price (USD) |
|-----------|---------|--------------|------------|-------------|-------------|
| `starter` | 5 GB    | 2            | 8%         | 4%          | Free        |
| `pro`     | 100 GB  | Unlimited    | 4%         | 2%          | $19/month   |
| `studio`  | 500 GB  | Unlimited    | 2%         | 1%          | $49/month   |

- Starter is the default for all new accounts.
- Prices are defined per currency (see [Local Currency Pricing](#local-currency-pricing)).
- **Escrow fee** applies when the client pays online via DodoPayments — Zeyaroo holds and releases the funds, so the fee is higher.
- **Offline fee** applies when the creator manually marks a milestone as funded (bank transfer, cash) — Zeyaroo provides record-keeping and invoicing only, so the fee is lower.
- Fee rates shown above are defaults. Custom plans can set different rates per user.

---

## Billing Cycle

- **30-day rolling cycles** starting from the creator's subscription date (not the 1st of the month).
- **Subscription fee**: billed in advance at cycle start — pay for the next 30 days.
- **Offline milestone platform fees**: billed in arrears at cycle end — fees owed on milestones recorded during the completed cycle.
- One combined invoice per cycle covering both.

Example invoice for a Pro creator (cycle Apr 10 → May 9):

```
Invoice #047 — Cycle: Apr 10 → May 9
────────────────────────────────────────────────────
Subscription — Pro (May 10 → Jun 8)        LKR 5,800

Platform Fees — offline milestones (Apr 10 → May 9)
  Perera Wedding — deposit    LKR 20,000 × 2% (offline) =   LKR 400
  Silva Portfolio — final     LKR 45,000 × 2% (offline) =   LKR 900

                                           Total due:       LKR 7,100
────────────────────────────────────────────────────
Issued: May 10    Due: May 17
```

> **Note:** Escrow fees on online (DodoPayments) milestones are deducted at source — they do not appear on the invoice. Only offline milestone fees are invoiced.

---

## Payment Methods

Both subscriptions and platform fee invoices can be paid:

1. **Online** — via DodoPayments (card/wallet). Handled automatically.
2. **Offline** — bank transfer. Admin manually verifies receipt and marks the invoice as paid.

For the initial Sri Lanka market, offline bank transfer is the primary method. The platform is designed around this being the norm, not the exception.

---

## Grace Period & Capability Limiting

Rather than downgrading a creator's tier for non-payment, their account is capability-limited while their plan and data remain intact.

### Timeline

| Day | Event |
|-----|-------|
| Day 0 | Invoice issued. Due date: Day 7. Grace period ends: Day 37. |
| Day 7 | Due date passes. Reminder notification sent (email + in-app). |
| Day 21 | Warning: "Your account will be limited in 16 days." |
| Day 30 | Warning: "Your account will be limited in 7 days." |
| Day 37 | Grace period ends. Features locked. Persistent banner shown. |

### What Gets Locked

- Creating new projects
- Uploading media
- Sending proposals

### What Stays Available

- View all existing projects and media (read-only)
- Client communication
- Downloading their own work

### Restoration

Features are restored **immediately** on invoice payment — no waiting for the next cycle.

---

## Local Currency Pricing

Subscription plan prices are defined per currency. The creator's billing currency is determined from their profile (`users.currency`) at the time of subscription and does not change mid-subscription.

If a plan has no price defined for the creator's currency, the USD price is used as a fallback.

Example pricing table for Pro:

| Currency | Price     |
|----------|-----------|
| USD      | $19.00    |
| LKR      | LKR 5,800 |
| EUR      | €18.00    |

Prices are stored in the database and managed by admin — not hardcoded.

---

## Custom Plans

Sales (currently just admin) can create a custom subscription plan for a specific creator. Custom plans are **1:1** — one plan, one user.

A custom plan defines the same fields as a standard tier:

| Field            | Description                                      |
|------------------|--------------------------------------------------|
| `name`           | Display name, e.g., "Custom — Kavinda Photography"                |
| `storage_gb`     | Storage limit in GB                                               |
| `max_projects`   | Max active projects (-1 = unlimited)                              |
| `escrow_fee`     | Fee % when client pays online via DodoPayments (higher)           |
| `offline_fee`    | Fee % when creator marks milestone as manually funded (lower)     |
| `prices`         | Per-currency monthly price                                        |
| `price_override` | Optional one-off negotiated price (overrides per-currency price)  |
| `assigned_to`    | The specific user this plan belongs to                            |

**Key point:** Both fee rates are per-plan, meaning they're effectively per-user for custom plans. Changing a creator's fee rates means updating their plan.

Custom plans support different billing cycle lengths (`cycle_days`) but this is not surfaced in the UI yet — it will be available via the admin panel.

---

## Upgrade / Downgrade

### Upgrade (e.g., Starter → Pro)

- Effect is **immediate** — new storage, project limits, and fee rate apply right away.
- Billing: charged from today, billing cycle resets.

### Downgrade (e.g., Pro → Starter)

- Effect at **end of current billing period** — creator keeps their current tier benefits until the cycle they've already paid for ends.
- No refund issued.

### Downgrade Constraints

The system must validate before allowing a downgrade:

| Moving to | Project constraint    | Storage constraint |
|-----------|-----------------------|--------------------|
| Starter   | Must have ≤ 2 active projects | Must be using ≤ 5 GB |
| Pro       | No constraint (unlimited → unlimited) | Must be using ≤ 100 GB |

If a constraint is not met, the downgrade is blocked with a clear message explaining what needs to be reduced first.

---

## Data Model

The current `users.subscription_tier string` is replaced by a proper subscription plan system.

### `subscription_plans`

Stores both standard tier templates and custom per-user plans.

```go
type SubscriptionPlan struct {
    ID             uuid.UUID
    Name           string          // "Pro", "Studio", "Custom — Kavinda Photography"
    Code           string          // "pro", "studio", "custom" — for standard templates
    StorageGB      int             // storage limit; -1 = unlimited
    MaxProjects    int             // -1 = unlimited
    EscrowFee      decimal.Decimal // e.g., 0.04 for 4% — online DodoPayments milestones
    OfflineFee     decimal.Decimal // e.g., 0.02 for 2% — manually funded milestones
    IsTemplate     bool            // true for starter/pro/studio
    AssignedToID   *uuid.UUID      // null for templates, user ID for custom plans
    CreatedByID    uuid.UUID       // admin user who created it
    CreatedAt      time.Time
}
```

### `subscription_plan_prices`

Per-currency prices for each plan.

```go
type SubscriptionPlanPrice struct {
    PlanID     uuid.UUID
    Currency   string          // "USD", "LKR", "EUR"
    Price      decimal.Decimal // price per cycle
    CycleDays  int             // default 30
}
```

### `user_subscriptions`

Tracks the active subscription for each creator.

```go
type UserSubscription struct {
    UserID         uuid.UUID       // one active subscription per user
    PlanID         uuid.UUID
    Currency       string          // billing currency, set at subscription time
    PriceOverride  *decimal.Decimal // null = use plan_prices; set = negotiated one-off
    CycleStart     time.Time
    CycleEnd       time.Time
    Status         string          // "active" | "grace" | "limited" | "cancelled"
    PaymentMethod  string          // "online" | "offline"
    CreatedAt      time.Time
    UpdatedAt      time.Time
}
```

### `invoices`

One invoice per billing cycle per creator.

```go
type Invoice struct {
    ID              uuid.UUID
    UserID          uuid.UUID
    CycleStart      time.Time
    CycleEnd        time.Time
    SubscriptionFee decimal.Decimal
    PlatformFees    decimal.Decimal // sum of offline milestone fees for this cycle
    Total           decimal.Decimal
    Currency        string
    Status          string          // "pending" | "paid" | "overdue" | "waived"
    DueDate         time.Time
    GracePeriodEnd  time.Time       // DueDate + 30 days
    PaidAt          *time.Time
    IssuedAt        time.Time
}
```

### `invoice_line_items`

Individual offline milestone fees that make up `invoices.platform_fees`.

```go
type InvoiceLineItem struct {
    ID          uuid.UUID
    InvoiceID   uuid.UUID
    MilestoneID uuid.UUID
    Description   string          // "Perera Wedding — deposit"
    PaymentMethod string          // "escrow" | "offline"
    Amount        decimal.Decimal // milestone amount
    FeeRate       decimal.Decimal // fee % at time of recording (escrow or offline rate)
    FeeAmount     decimal.Decimal // amount × fee_rate
    Currency      string
    RecordedAt    time.Time
}
```

---

## Migration from Current Schema

1. Seed three `subscription_plans` rows for starter, pro, studio with `is_template = true`.
2. Seed `subscription_plan_prices` for USD (and LKR if defined).
3. For each existing user, create a `user_subscriptions` row pointing to the matching template plan based on their current `subscription_tier` string.
4. Update `FeatureGateService` and all limit checks to read from `user_subscriptions → subscription_plans` instead of switching on `users.subscription_tier`.
5. Keep `users.subscription_tier` as a denormalized cache (updated on subscription changes) for queries that need a quick tier check without a join — but the source of truth is `user_subscriptions`.

---

## Offline Milestone Fee Tracking

When a creator calls `MarkManuallyFunded` on a milestone:

1. Record a pending `invoice_line_item` using the creator's current plan's **`offline_fee` rate**.
2. Set `payment_method = "offline"` on the line item so the rate used is auditable.
3. Associate it with the creator's current open invoice for the cycle (or create one if it doesn't exist yet).
4. The line item is included in the invoice generated at the next cycle end.

When DodoPayments processes a milestone online (escrow flow):

1. The **`escrow_fee` rate** is applied and deducted at source before funds are released to the creator.
2. No invoice line item is created — the fee never passes through the creator's hands.
3. The escrow fee amount is recorded on the milestone for reporting purposes.

The fee rate snapshotted at the time of payment is what counts — if a creator's plan changes mid-cycle, earlier milestones keep the rate that was in effect when they were paid.

---

## Admin Operations

All of the following require admin role:

- Create a custom plan and assign it to a user
- Set per-currency prices on any plan
- Mark an offline invoice as paid
- Waive an invoice (status = "waived", does not trigger capability limiting)
- Manually restore a limited account (e.g., payment confirmed but system hasn't updated yet)
- Change a creator's active subscription plan
