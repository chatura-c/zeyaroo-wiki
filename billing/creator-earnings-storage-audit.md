# Creator Earnings & Storage — Audit

## Total Earned

### What "earned" should mean
The total money a creator has received from clients via released milestone payments — i.e.
`SUM(creator_payout_cents) WHERE payee_id = <creator> AND status = 'released'` on `escrow_transactions`.

### What the system actually does

**Dashboard.tsx:417** calculates "earned" by summing `total_cents` from the creator's own paid invoices:

```ts
const earned = (paidInvoicesData?.items ?? []).reduce((s, inv) => s + inv.total_cents, 0);
```

Those invoices are **outbound billing** — storage fees, platform service fees, plan subscriptions.
They represent what the creator *owes the platform*, not what they *received from clients*.
The figure shown as "Total Earned" on the dashboard is therefore wrong.

### Where the real data lives

`EscrowTransaction` (models/escrow.go) has:
- `payee_id` — the creator receiving the payment
- `amount_cents` — gross milestone amount
- `platform_fee_cents` — platform's cut
- `creator_payout_cents` — net amount the creator keeps
- `status` — `released` means the milestone was paid out

A correct earnings query:
```sql
SELECT COALESCE(SUM(creator_payout_cents), 0)
FROM escrow_transactions
WHERE payee_id = '<creator_id>'
  AND status = 'released'
```

### What's missing

- No backend endpoint for a creator's total earned (no `GET /users/me/earnings` or similar).
- The admin stats endpoint (`GET /admin/stats`) computes *platform* revenue (`SUM(platform_fee_cents)`),
  not per-creator income.

### Fix required

1. Add a service method `EscrowService.GetCreatorEarnings(creatorID uuid.UUID)` that returns
   total earned, pending (held), and per-period breakdowns.
2. Expose it via a new endpoint (e.g. `GET /users/me/earnings`).
3. Update Dashboard to use that endpoint instead of summing paid invoices.

---

## Storage Usage

### Creator-facing (correct)

`StorageService.GetUsage()` (services/storage.go:37) calculates live bytes from two queries:

- **Explicit ownership:** `media_objects WHERE owner_user_id = <user>`
- **Implicit ownership:** `media_objects JOIN projects WHERE projects.owner_id = <user> AND owner_user_id IS NULL`

Returns `used_bytes`, `free_quota_bytes`, `quota_bytes`, storage type, and price-per-GB.
Exposed via `GET /storage/usage`. Dashboard renders it correctly as a storage ring.

### Admin-facing (stale / incorrect)

`AdminService.GetUser()` (services/admin.go:208) reads `StorageSubscription.used_bytes`:

```go
s.db.Model(&models.StorageSubscription{}).Where("user_id = ?", userID).Select("COALESCE(used_bytes, 0)").Scan(&storageUsed)
```

`StorageSubscription.used_bytes` is a cached column — not guaranteed to match the live calculation.
This means the admin user detail view and the platform-wide storage stat (`GET /admin/storage/stats`)
can show values that diverge from what the creator sees on their own dashboard.

### Fix required

Replace the `StorageSubscription.used_bytes` reads in `AdminService` with calls to
`StorageService.calculateUsedStorage(userID)` (or a new exported wrapper), so admin and creator
see the same number.

---

## Summary of Gaps

| # | Area | Issue | Severity |
|---|------|--------|----------|
| 1 | Creator earnings | No backend endpoint; earnings data doesn't exist per creator | High |
| 2 | Dashboard "earned" | Sums creator's own billing invoices — semantically wrong | High |
| 3 | Admin storage | Uses stale `used_bytes` column instead of live calculation | Medium |
