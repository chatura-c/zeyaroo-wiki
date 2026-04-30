# Pre-Beta Launch Issues

Issues found during pre-beta audit (April 2026). Each is scoped to fit a single working session. Tackle P0 before any public beta.

- **P0** — must-fix before public beta (security or revenue/correctness) [COMPLETED]
- **P1** — should-fix before public beta (user-visible gaps, support load)
- **P2** — post-beta (defer; tracked here so we don't lose them)

---

## P0 — Must-fix Before Public Beta

---

### Issue B1 — Refuse to boot with default JWT secret in non-dev mode

**Labels:** `critical`, `security`

**Description:**
`JWT_SECRET` falls back to the literal string `"change-me-in-production"` when the env var is unset. If the var is missing in production deploy, the server boots with a known secret and anyone can forge admin/user tokens.

**Acceptance criteria:**
- In `cmd/server/main.go`, after config load: if `cfg.JWT.Secret == ""` or `cfg.JWT.Secret == "change-me-in-production"` AND `!cfg.Server.DevMode`, call `log.Fatal` with a clear message
- Dev mode behavior unchanged (default still works locally)
- Add a startup log line confirming the secret is set (without printing it)

**Technical notes:**
- Default lives at `internal/config/config.go:186`
- Check goes in `cmd/server/main.go` near where the auth middleware is built (~line 268)

---

### Issue B2 — Webhook signature verification must fail closed

**Labels:** `critical`, `security`

**Description:**
Dodo webhook handler currently checks signatures only when the secret is configured (`if h.dodoWebhookSecret != ""`). If the secret env var isn't set the entire signature check is skipped — webhooks are accepted unsigned. Same trap exists for any future webhook source.

**Acceptance criteria:**
- If `dodoWebhookSecret` is empty, the handler returns 503/500 with a clear error (do not process the payload)
- Same posture for any other webhook source we add (audit `webhook.go` for similar `if secret != ""` patterns)
- Startup logs whether each webhook secret is configured (no values)

**Technical notes:**
- `internal/handlers/webhook.go:121-129` (Dodo)
- Also re-check the PayHere webhook path (`receivePayHereWebhook`)

---

### Issue B3 — Add replay protection to Dodo webhooks

**Labels:** `security`, `webhooks`

**Description:**
Webhook handler verifies the HMAC body signature but has no timestamp window or transaction-ID dedup. A captured webhook can be replayed any number of times — useful to a malicious user who watched a real `payment.success` go through.

**Acceptance criteria:**
- Reject webhooks whose `timestamp` header is older than 5 minutes (or whatever Dodo recommends)
- Persist `transaction_id` of every accepted webhook; reject duplicates within the dedup window (24 h is fine)
- Existing `webhook_attempts` table can store the dedup state — verify schema, add a unique index if missing

**Technical notes:**
- `internal/handlers/webhook.go:115-178`
- `webhook_attempts` model is in `internal/models/webhook.go`
- Dodo signs both the body and a timestamp header — confirm the exact header name from their docs (likely `X-Dodo-Timestamp`)

---

### Issue B4 — Reject Dodo webhooks with missing currency instead of defaulting to USD

**Labels:** `bug`, `security`

**Description:**
`receiveDodoPaymentsWebhook` defaults the currency to `"USD"` when the webhook omits it. Sri Lankan payments are in LKR — a malformed webhook would book an LKR payment as USD, corrupting ledger entries and creator earnings.

**Acceptance criteria:**
- If `payload.Data.Currency` is empty, return 400 with a clear error
- No silent default

**Technical notes:**
- `internal/handlers/webhook.go:155-158`
- Same fix probably wanted in PayHere handler — check

---

### Issue B5 — Tighten CORS allowed origins

**Labels:** `security`

**Description:**
CORS currently allows every `*.bytkloud.com` subdomain. If any bytkloud subdomain is ever compromised or repointed it can hit the API with credentials. Production should allow only the specific frontend origins we ship.

**Acceptance criteria:**
- Allowed origins are a fixed list pulled from config (e.g. `CORS_ALLOWED_ORIGINS=https://app.zeyaroo.com,https://zeyaroo.com`)
- Localhost still allowed when `cfg.Server.DevMode` is true
- No wildcard subdomain match in production

**Technical notes:**
- `internal/handlers/router.go:62-87`
- Add `CorsAllowedOrigins []string` to config

---

### Issue B6 — `ResendMilestonePaymentRequest` sends the "completed" email template

**Labels:** `bug`, `email`

**Description:**
The function name says payment-request but the implementation calls `SendMilestoneCompletedEmail`. Clients get a "your work is ready, please pay" email even when the resend was for a milestone that wasn't in that state. Confusing.

**Steps to reproduce:**
1. Creator triggers "resend payment request" on a milestone in `awaiting_funds`
2. Client receives an email with completed-delivery framing

**Acceptance criteria:**
- If milestone is in `awaiting_funds`, send `SendPaymentRequestedEmail`
- If milestone is in `submitted` (creator delivered, awaiting client release), send `SendMilestoneCompletedEmail`
- Use `notifications.BypassDedup` so the resend actually goes out

**Technical notes:**
- `internal/services/escrow.go:627-671`
- `ResendFundRequest` (`escrow_payment_request.go:445`) already sends the right template for the awaiting-funds case — this resend route may even be redundant; consider deleting

---

### Issue B7 — Invoice emails always render amounts as USD

**Labels:** `bug`, `billing`, `i18n`

**Description:**
`buildInvoiceLineItemsHTML` and the invoice email use a hardcoded `$%.2f` format. LKR invoices show as `$5800.00` instead of `LKR 5,800.00`. Customer-visible bug for the entire SL launch market.

**Acceptance criteria:**
- Format the line-item amount and total according to the invoice's `Currency` field
- USD: `$1,234.56`
- LKR: `LKR 1,234.56` (or local convention)
- Use a small helper `formatMoneyCents(amountCents int64, currency string) string` and apply it everywhere invoice/email money is rendered

**Technical notes:**
- `internal/services/billing.go:277` and `:313`
- Check email templates in `notifications/` for any other hardcoded `$`

---

### Issue B8 — `MarkInvoicePaidByGateway` hardcodes "payhere"

**Labels:** `bug`, `billing`

**Description:**
The function looks up the pending payment transaction with `gateway_name = "payhere"`. Dodo-paid invoices won't match and the update silently fails to find the row.

**Acceptance criteria:**
- Take a `gatewayName` parameter; caller passes the actual gateway
- Or remove the gateway filter entirely if there can only be one pending tx per invoice

**Technical notes:**
- `internal/services/billing.go:943-963`
- Audit the call sites — webhook handler is the likely caller

---

### Issue B9 — Public payment page promises "escrow protection" for media milestones that release funds immediately

**Labels:** `bug`, `legal`, `ux`

**Description:**
For media-type milestones, `FundMilestone` releases funds immediately on payment (no hold). The public payment page still tells the client "Funds held securely by Zeyaroo until you approve the delivery. Full escrow protection." Misleading; could be a legal liability if a dispute happens.

**Acceptance criteria:**
- When `data.deliverable_type === 'media'` (or a server-provided `release_mode === 'immediate'` flag), show different copy: "Direct payment to creator" or similar
- Add the deliverable type / release mode to `PublicPaymentPageData`
- Or, alternative: change media milestone behavior to actually hold escrow until client confirms (bigger change — not in scope for this issue)

**Technical notes:**
- Frontend: `phto-ui/src/pages/PublicPayment.tsx:115-117`
- Backend: `internal/services/escrow_payment_request.go:194-211` (add deliverable type to `PublicPaymentPageData`)
- Logic that releases media payments immediately: `internal/services/escrow.go:294-368`

---

### Issue B10 — `ProposalAcceptedEvent` not emitted when creator accepts a client counter-proposal

**Labels:** `bug`, `notifications`

**Description:**
`ProposalService.Accept` only publishes the event when the *client* accepts. When the *creator* accepts a client's counter-proposal the proposal is finalized (status → finalized, project auto-created) but no event fires — the client never gets notified that their counter was accepted.

**Acceptance criteria:**
- Event fires for both acceptance paths (creator and client)
- Event includes who accepted (so the notification template can address the correct recipient)
- Both creator and client receive an in-app notification + email when the proposal is finalized

**Technical notes:**
- `internal/services/proposal.go:330-379` — `Accept` method
- Currently the event is gated by `if !isCreatorAccepting` (line 371)
- May need a new event variant or a field on the existing event

---

### Issue B11 — Verify auto-release cron is actually running

**Labels:** `audit`, `escrow`

**Description:**
`internal/services/escrow_auto_release.go` exists, but the audit didn't confirm it is scheduled. If it's defined but not started, escrowed funds for `submitted` media-free milestones will sit forever when the client never clicks release.

**Acceptance criteria:**
- Confirmed: in `cmd/server/main.go` (or wherever services are started) the auto-release loop is launched as a goroutine on a sensible interval
- Add a startup log line for the schedule
- Add a healthcheck/admin endpoint that reports last-run timestamp, count released
- If not scheduled, schedule it; ship a small admin endpoint to trigger it manually

**Technical notes:**
- `internal/services/escrow_auto_release.go`
- `cmd/server/main.go` background-task wiring

---

### Issue B12 — `ForceCompleteMilestone` creates no escrow record and no commission

**Labels:** `critical`, `billing`, `audit`

**Description:**
A creator can mark a stuck `submitted` or `awaiting_funds` milestone as `completed` via this endpoint with no escrow record written and no platform-commission line item created. For offline-paid milestones this is a revenue leak (we never invoice the creator's commission) and we have no audit trail of how the milestone became "paid".

**Acceptance criteria:**
- `ForceCompleteMilestone` is restricted to a narrower set of states or requires explicit confirmation:
  - For `submitted` (creator delivered, escrow already held): allow — equivalent to client-release; ensure escrow is moved to `released` and the payout ledger entry is written
  - For `awaiting_funds`: refuse, OR require the creator to also pass `payment_method` (offline) and an `ack_amount_cents`, then route through `MarkMilestonePaid`
- Every force-complete writes an audit row (admin-visible) with the reason
- For offline force-complete, the platform commission line item is added to the creator's draft invoice (same path `MarkMilestonePaid` uses)

**Technical notes:**
- `internal/services/escrow.go:674-734`
- Reuse `BillingService.AddOfflineCommissionLineItem`
- Add a model for force-complete audit (or reuse an existing event/audit table)

---

### Issue B13 — Sequential milestone ordering is inconsistent

**Labels:** `bug`, `escrow`

**Description:**
`RequestFunding` allows milestone N if milestone N-1 is in `awaiting_funds` (treated as funded). But `MarkMilestonePaid` only allows N when N-1 is `funded` or `completed`. Result: a creator can request funding on milestone 2 while milestone 1 is only in `awaiting_funds`, and then get stuck because mark-paid won't accept it.

**Acceptance criteria:**
- Pick one definition of "previous milestone is settled" and apply it to all gates: `RequestFunding`, `FundMilestone`, `MarkMilestonePaid`
- Recommended: only `funded` or `completed` counts as settled. `awaiting_funds` does NOT.

**Technical notes:**
- `internal/services/escrow_payment_request.go:108-119` (RequestFunding)
- `internal/services/escrow_payment_request.go:546-555` (MarkMilestonePaid)
- `internal/services/escrow.go:188-196` (FundMilestone)

---

## P1 — Should-fix Before Public Beta

---

### Issue B14 — Dispute resolution admin UI - [SKIP]

**Labels:** `feature`, `admin`

**Description:**
Backend has `ListDisputes`, `ResolveDispute` (release/refund), and the route is wired. Admin dashboard has no page that lists disputes and lets admin act on them. A single dispute on launch day will require manual DB intervention.

**Acceptance criteria:**
- New admin page `/disputes` listing open disputes with: project, milestone, both parties' contact, reason, escrow amount
- Detail view shows the dispute reason, milestone history, any feedback messages, links to download delivery files
- Two action buttons: "Release to creator" and "Refund client" — each opens a notes input then calls `POST /admin/milestones/{id}/resolve`
- Both parties get an email notification when admin resolves

**Technical notes:**
- Backend endpoints already exist
- Admin frontend: `zeyaroo-admin/` — model after the existing `DisputeList` page if present, expand if not
- Dispute resolution emails: add to `notifications/email/` if not already there

---

### Issue B15 — Default 6-chapter milestone template (also tracked in `issues.md` #9) [SKIP]

**Labels:** `feature`, `content`, `ux`

**Description:**
Open in `issues.md` as Issue 9. Pre-populate new proposals with the 6-chapter structure (Retainer / Pre-Shoot / On-Day / Social Delivery / Album / Final Sign-off). Major first-time creator UX win.

**Pointer:** see `wiki/planning/issues.md#issue-9`. Implementation likely a frontend constant in `ProposalBuilder.tsx`.

---

### Issue B16 — First-time creator onboarding tour (also tracked in `issues.md` #5) ✅

**Labels:** `feature`, `ux`

**Description:**
Open in `issues.md` as Issue 5. Lightweight guided tour for new creators on first login.

**Pointer:** see `wiki/planning/issues.md#issue-5`. Use `localStorage` flag for MVP.

---

### Issue B17 — Stop reordering milestones by deadline ✅

**Labels:** `bug`, `ux`

**Description:**
`processMilestones` sorts new-version milestones chronologically by deadline and pushes deadline-less milestones to the end. This silently rewrites the creator's intended ordering. After Issue B15 ships, the 6-chapter template would get its order scrambled.

**Acceptance criteria:**
- Preserve the order the creator submitted
- If we want chronological sorting, expose it as an explicit "Sort by deadline" toggle in the builder
- Sequence numbers reflect the submitted order

**Technical notes:**
- `internal/services/proposal.go:273-294`
- The `processMilestones` function can become a no-op (or be deleted)

---

### Issue B18 — Audit and close TODOs flagged in architecture doc ✅

**Labels:** `audit`, `security`

**Description:**
The Feb 2026 architecture doc (`wiki/architecture/project-overview.md` §8.2) lists code TODOs:
- `handlers/escrow.go:492` — missing project access verification
- `handlers/proposal.go:537` — milestone update not implemented (likely now done — verify)
- `handlers/share.go:744` — file serving not fully integrated
- `handlers/share.go:516` — share media access verification incomplete

**Acceptance criteria:**
- Each TODO is either: (a) confirmed fixed and the TODO comment removed, or (b) reproduced and a follow-up issue created

**Technical notes:**
- Walk each line; line numbers will have drifted since Feb. Use `git blame` to find current locations.

---

### Issue B19 — Invoice number entropy is too low (32-bit random suffix) ✅

**Labels:** `bug`, `billing`

**Description:**
`generateInvoiceNumber` uses 4 random bytes. Collision probability (birthday paradox) becomes meaningful at ~65k invoices. The DB doesn't enforce uniqueness on invoice number either.

**Acceptance criteria:**
- Bump to 8 random bytes (or use a sequence per year)
- Add a unique index on `invoices.invoice_number`
- On insert collision, retry once with a fresh suffix

**Technical notes:**
- `internal/services/billing.go:221-225`

---

### Issue B20 — Add expiry on payment request tokens ✅

**Labels:** `security`, `escrow`

**Description:**
Public payment tokens never expire. If forwarded or scraped from a leaked email, the milestone title, amount, and creator name remain readable forever.

**Acceptance criteria:**
- Add `payment_request_token_expires_at` to milestone model (default: 30 days from creation)
- `GetPublicPaymentPage` returns 404/410 for expired tokens
- Creator can re-issue (regenerate token + reset expiry) via `ResendFundRequest`

**Technical notes:**
- `internal/models/milestone.go`
- `internal/services/escrow_payment_request.go:214-264`

---

### Issue B21 — Add account-level rate limit on auth attempts ✅

**Labels:** `security`

**Description:**
Auth routes have an IP rate limit (10/s) but no account-level lockout. An attacker rotating IPs can brute-force a single account's password.

**Acceptance criteria:**
- After N (e.g. 10) failed login attempts for the same email within 15 min, return 429 / 423 with a clear message
- Counter resets on successful login or after the window expires
- Counter is per-email, not per-IP

**Technical notes:**
- `internal/handlers/auth.go` — `Login` and `FirebaseLogin`
- Could be in-memory (sync.Map) for MVP; move to Redis later

---

## P2 — Post-Beta (deferred, tracked here so we don't lose them)

These are real but not blocking a controlled beta launch. Re-evaluate after first 50 users.

---

### Issue B22 — Proposal version diff view

**Labels:** `feature`, `ux`

When a client/creator opens version 3 they should see what changed vs version 2 (added/removed/modified milestones, deliverables, amounts, dates). Currently they see only the raw version. Negotiation UX gap.

**Implementation sketch:** dedicated component that takes two `ProposalVersion`s and renders a side-by-side or unified diff. No backend change needed.

---

### Issue B23 — Visual milestone timeline UI

**Labels:** `feature`, `ux`

Render the milestone state machine (pending → funded → in_progress → submitted → completed) as a visual timeline on the project detail and public proposal pages. Currently it's only status badges. Hard for non-technical users to track progress.

---

### Issue B24 — PDF invoices

**Labels:** `feature`, `billing`

Currently invoices are HTML only. Sri Lankan creators may need PDF for tax records. Use a server-side renderer (e.g. `gofpdf` or `chromedp` with the existing HTML email template).

---

### Issue B25 — Tax calculation

**Labels:** `feature`, `billing`

`Invoice.TaxCents` is a placeholder. When we have paying users in tax-jurisdictions this becomes urgent. SL VAT rules + per-country logic.

---

### Issue B26 — Sent email log + resend in admin

**Labels:** `feature`, `admin`

`sent_email` model exists. Admin should be able to:
- Search by recipient or subject
- See if a specific email actually went out
- Resend a single email by ID

---

### Issue B27 — Webhook retry button in admin

**Labels:** `feature`, `admin`

`ListWebhookAttempts` returns the data. Admin needs a "Retry" button per attempt.

---

### Issue B28 — Manual capability restoration endpoint

**Labels:** `feature`, `admin`, `billing`

Per `wiki/billing/billing-design.md` — admin should be able to manually restore a capability-limited account (e.g. payment confirmed but system hasn't updated yet). No endpoint exists.

---

### Issue B29 — User impersonation / "view as user"

**Labels:** `feature`, `admin`, `support`

Common support need. Generate a short-lived JWT for any user from the admin panel, with a banner indicating impersonation. Audit all impersonation actions.

---

### Issue B30 — `PlatformStats.user_growth` and `revenue_by_month` always empty

**Labels:** `bug`, `admin`

Memory note: fields exist in `PlatformStats` type but `AdminService.GetStats` doesn't populate them. Cosmetic on the dashboard.

---

### Issue B31 — Custom plan creation endpoint

**Labels:** `feature`, `billing`, `admin`

`wiki/billing/billing-design.md` describes 1:1 custom plans. `AdminUpdatePackage` exists but no `POST /admin/users/{id}/custom-plan` flow with the full schema (storage, fees, prices). Build when we want to actually offer custom plans to specific creators.

---

### Issue B32 — Billing-design.md vs implementation drift

**Labels:** `tech-debt`, `billing`, `docs`

The doc describes a clean cycle (one combined invoice = subscription + offline platform fees). The code emits separate invoices for storage subscriptions, plan subscriptions, and standalone draft invoices for offline commissions. Either update the doc to match reality or refactor the code. Either way: pick one and align before custom plans ship.

---

## How to Use This File

- Issues are prefixed `B` (beta) to distinguish from `issues.md` numbering
- Each issue is sized for one working session
- Update labels and add a `✅` to the heading when done (matches `issues.md` style)
- Reorder priorities as we learn from beta users
