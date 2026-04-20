# Zeyaroo — Beta Prep Checklist

Each item is a self-contained task that can be tackled in a single session.
Items are ordered by priority: **P0** = must fix before any public user, **P1** = must fix for a trustworthy beta, **P2** = important but can be done mid-beta.

---

## Subscription Tiers Reference
[i want subscriptions to be flexible per user]
payments can be done same ways as milestone payments
online, offline[via bank transfer] + we should generate a invoice. 
| Tier | Storage | Max Projects | Platform Fee | Price |
|---|---|---|---|---|
| `starter` (default) | 5 GB | 2 | 5% | Free |
| `pro` | 100 GB | Unlimited | 2% | $19/month |
| `studio` | 500 GB | Unlimited | 1% | $49/month |

Defined in `phto-api/internal/services/tiers.go`.
Stored as `users.subscription_tier` (string, default `"starter"`).
Also tracked: `subscription_id` (DodoPayments subscription ID), `subscription_expires_at`.

---

## P0 — Must Fix Before Any Public Access

---

### ~~P0-1 · Remove Eruda Debug Console~~ ✓

**Problem:** `eruda` is imported and initialized on every page load in production. Even though it's hidden with `eruda.hide()`, it still runs initialization code, adds event listeners, and signals to anyone inspecting the page that this is an unfinished product.

**Files:**
- `phto-ui/src/main.tsx` — lines 3, 8–9

**Fix:**
Remove the eruda import and all eruda calls entirely:
```tsx
// DELETE these three lines:
import eruda from 'eruda'
eruda.init()
eruda.hide()
```
Also remove eruda from `package.json` dependencies and run `npm install` to update the lockfile.

**Verify:** Production build should have no reference to `eruda` in the bundle.

---

### ~~P0-2 · Remove /debug Page~~ ✓

**Problem:** The `/debug` route is publicly accessible (no auth guard) and exposes all Vite env vars (`import.meta.env`), build metadata, and internal configuration to any anonymous visitor.

**Files:**
- `phto-ui/src/pages/DebugPage.tsx` — the full page
- `phto-ui/src/App.tsx` — route registration around line 58

**Fix:**
1. Delete `phto-ui/src/pages/DebugPage.tsx`
2. Remove the `/debug` route from `App.tsx`

**Verify:** `GET /debug` should return the app's 404 / catch-all, not the debug page.

---

### ~~P0-3 · Move Resend Credentials Out of todo2.md~~ ✓ (key removed from file — rotate it in Resend dashboard and set RESEND_API_KEY in prod .env)

**Problem:** A live Resend API key is committed in plaintext in `todo2.md` line 26. This key should be rotated and moved to the deployment environment.

```
password - re_evrjdm36_57Gh9YBqpThhCFFtakByf1mz   ← LIVE KEY, DO NOT SHARE
host - smtp.resend.com
port - 465
username - resend
```

**Steps:**
1. Log in to Resend and revoke the key above. Generate a new one.
2. Add `RESEND_API_KEY=<new-key>` to the production `.env` (on the server, not committed).
3. Verify the backend's `RESEND_API_KEY` env var is read and passed to `EmailService`. If the key is absent, the email service falls back to a `noopSender` — confirm by sending a test notification.
4. Remove lines 25–30 from `todo2.md`.

**Verify:** Send a test project invitation or inquiry notification; confirm a real email arrives.

---

### ~~P0-4 · Fix TransferDialog Runtime Crash~~ ✓

**Problem:** `TransferDialog` is rendered with a stub `onSuccess` that throws. Any user who transfers a project will hit an unhandled runtime error.

**File:** `phto-ui/src/pages/ProjectDetail.tsx` around line 553–556:
```tsx
onSuccess={function (): void {
  throw new Error('Function not implemented.');
}}
```

**Fix:**
Replace the stub with the correct success behavior — likely invalidate the project list query and navigate away (since the project is no longer owned by this user after transfer):
```tsx
onSuccess={() => {
  queryClient.invalidateQueries({ queryKey: ['projects'] })
  navigate('/projects')
}}
```
Confirm the exact query key used for the project list in the codebase.

**Verify:** Transfer a project end-to-end; confirm the UI navigates cleanly without throwing.

---

### ~~P0-5 · Wire DodoPayments Subscription Webhook → Tier Upgrade~~ ✓ (already wired — subscription.created/renewed/cancelled all handled in webhook.go)

**Problem:** When a creator subscribes to Pro or Studio, `BillingService.CreateSubscriptionCheckout` creates a DodoPayments checkout and returns a URL. The creator pays, DodoPayments sends a webhook — but it's unclear if `subscription.created` / `subscription.renewed` events from DodoPayments are actually routed to `UserService.UpdateSubscription`.

**Investigation needed first:**
1. Check `phto-api/internal/handlers/webhook.go` — does `receiveDodoPaymentsWebhook` handle `subscription.created` and call `userService.UpdateSubscription(userID, planID, subscriptionID, expiresAt)`?
2. Check the route registration — is there a specific DodoPayments route, or does it go through the generic `POST /api/v1/webhooks/{source_name}`?
3. If the generic route is used, check `WebhookService.ProcessWebhook` → `EventProcessor` — is there a handler registered for the `dodopayments.subscription.created` event type?

**Expected webhook payload from DodoPayments (approximate):**
```json
{
  "event": "subscription.created",
  "data": {
    "subscription_id": "sub_xxx",
    "metadata": {
      "user_id": "<uuid>",
      "plan_id": "pro"
    },
    "expires_at": "2026-05-10T00:00:00Z"
  }
}
```

**If the handler exists and is wired correctly:** Write an integration test that posts a mock `subscription.created` webhook and asserts `users.subscription_tier` is updated in the DB.

**If not wired:** Add a handler in `webhook.go` or the event processor that:
1. Parses `user_id` and `plan_id` from `metadata`
2. Parses `subscription_id` and `expires_at` from the payload
3. Calls `userService.UpdateSubscription(userID, planID, subscriptionID, &expiresAt)`

**`UserService.UpdateSubscription` signature:**
```go
func (s *UserService) UpdateSubscription(userID uuid.UUID, tier string, subscriptionID string, expiresAt *time.Time) error
```

**Verify:** Subscribe to Pro in staging, check `users` table that `subscription_tier = 'pro'` after webhook fires.

---

### ~~P0-6 · Enforce Storage Limits on Upload~~ ✓

**Problem:** `FeatureGateService.HasStorageSpace` exists and is correct, but neither the standard media upload handler (`MediaHandler.Upload`) nor the direct upload initiation handler (`MediaDirectUploadHandler.InitiateUpload`) calls it. Any user on any tier can upload unlimited data.

**Subscription limits to enforce:**
- Starter: 5 GB total across all their projects
- Pro: 100 GB
- Studio: 500 GB

**Files:**
- `phto-api/internal/handlers/media.go` — `MediaHandler` struct (fields: `svc`, `albumSvc`, `projectSvc`, `escrowSvc`, `userSvc`, `signer`)
- `phto-api/internal/handlers/media_direct_upload.go` — `MediaDirectUploadHandler.InitiateUpload`

**Fix:**
1. Add `featureGate *services.FeatureGateService` to both handler structs.
2. Wire via the router (see where `FeatureGate` is passed to `ProjectHandler` for pattern).
3. In `MediaHandler.Upload`: after parsing the multipart form and knowing `fileSize`, call:
   ```go
   if err := h.featureGate.HasStorageSpace(userID, fileSize); err != nil {
       if errors.Is(err, services.ErrStorageLimitReached) {
           w.WriteHeader(http.StatusPaymentRequired)
           json.NewEncoder(w).Encode(map[string]string{"error": "storage_limit_reached"})
           return
       }
       // handle other errors
   }
   ```
4. In `MediaDirectUploadHandler.InitiateUpload`: same check using `req.FileSize` from the request body (the size is declared up-front for direct uploads).

**Frontend handling:**
- The frontend should catch HTTP 402 on upload and show an upgrade prompt.
- Search for where upload errors are handled in the media upload hooks/components.

**Verify:**
- Create a Starter account, upload files until close to 5 GB limit, confirm next upload is rejected with 402.
- Confirm Pro and Studio accounts are not erroneously blocked.

---

## P1 — Must Fix for a Trustworthy Beta
P1 completed - DONE
---

### P1-1 · Add Password Reset Flow

**Problem:** No "Forgot password?" flow exists. Users who forget their password have no recovery path.

**Backend:** Firebase handles password reset via `sendPasswordResetEmail(auth, email)` — no backend changes needed as Firebase handles the email delivery and token verification.

**Frontend changes:**
1. Add a "Forgot password?" link on the login page (`phto-ui/src/pages/Login.tsx` or wherever auth is rendered).
2. Create a `ForgotPassword` page/modal:
   - Input: email address
   - On submit: call `sendPasswordResetEmail(auth, email)` from `firebase/auth`
   - Show confirmation: "If that email is registered, a reset link has been sent."
   - Always show the confirmation message regardless of whether the email exists (prevents enumeration).
3. Add a route: `/forgot-password` (can be a modal on the login page instead of a full page).

**Important UX detail:** The confirmation message must not reveal whether the email exists or not.

**Firebase:** No additional Firebase configuration needed — password reset is enabled by default for Email/Password providers.

**Verify:** Request reset for a real account, receive email, follow link, set new password, log in with new password.

---

### P1-2 · Add Email Verification

**Problem:** New accounts are immediately usable without verifying their email address. This allows throwaway emails and reduces deliverability reputation.

**Decision to make first:** Do you want to *require* verification before using the app, or just *prompt* for it?
- **Recommended for beta:** Prompt with a persistent banner ("Please verify your email — check your inbox") but don't hard-block.
- **Stricter option:** Gate certain actions (creating projects, sending proposals) behind verification.

**Firebase:** Call `sendEmailVerification(auth.currentUser)` after registration.

**Backend changes:**
- The Firebase JWT includes `email_verified: bool`. The backend can check this on protected routes if you want to gate access.
- Optional: add `EmailVerified bool` to the `User` model and sync it from Firebase claims.

**Frontend changes:**
1. In `useFirebaseAuth.ts` or the registration flow: after `createUserWithEmailAndPassword`, call `sendEmailVerification(user)`.
2. Add a `useEmailVerification` hook or check `auth.currentUser.emailVerified`.
3. Show a dismissible banner on the dashboard for unverified users with a "Resend verification email" button.
4. On "Resend" button: call `sendEmailVerification(auth.currentUser)` again (Firebase rate-limits this automatically).

**Verify:** Create new account, confirm verification email arrives, verify the banner disappears after clicking the link and refreshing.

---

### P1-3 · Fix Billing Page — Replace Mock Data with Real API

**Problem:** The main `/billing` page (`phto-ui/src/pages/Billing.tsx`) shows hardcoded mock data for:
- "Total Value Secured", "Available for Payout", "Net Lifetime Earnings" — all from `MOCK_SUMMARY`
- Subscription card — from `MOCK_SUBSCRIPTION`
- Payout method — from `MOCK_PAYOUT`
- Transaction ledger — falls back to `MOCK_TRANSACTIONS` if no real data

The real billing API exists and works (`GET /billing/payments`, `GET /billing/upcoming`). `BillingSettings.tsx` already uses these correctly. But the main Billing page ignores them.

**What backend endpoints exist:**
- `GET /api/v1/billing/payments` — paginated payment transactions
- `GET /api/v1/billing/upcoming` — upcoming charges
- `GET /api/v1/billing/invoices` — invoice list
- `GET /api/v1/users/me` — includes `subscription_tier`, `subscription_expires_at`

**What's missing from the backend (needs to be added):**
The "earnings" summary (Total Value Secured, Available for Payout, Net Lifetime Earnings) requires aggregating completed milestone payments. This doesn't exist as a dedicated endpoint. Options:
1. Add `GET /api/v1/billing/summary` that returns `{ total_secured, available_payout, lifetime_earnings }` computed from the milestones/escrow tables.
2. Or compute it client-side by summing payments data (less accurate but faster to ship).

**Frontend fix:**
1. Replace `MOCK_SUBSCRIPTION` with data from `GET /api/v1/users/me` (`subscription_tier`, `subscription_expires_at`).
2. Replace `MOCK_TRANSACTIONS` with data from `GET /api/v1/billing/payments` (already partially done — `paymentsData.items.length > 0` check suggests this).
3. Replace `MOCK_PAYOUT` with the creator's actual bank/payout details from `GET /api/v1/creators/profile` (fields: `bank_name`, `account_holder_name`, `account_number`, `swift_code` — but these are `json:"-"` currently, need to be exposed on a separate endpoint or creator profile response).
4. For the earnings summary, decide between option 1 or 2 above.

**Note on payout details exposure:** The `CreatorProfile` model marks bank details with `json:"-"`. To expose them to the creator (and only the creator), add a separate `GET /api/v1/creators/profile/payout-settings` endpoint that returns bank details only for the authenticated user.

---

### P1-4 · Wire "Save Forever" Button in Vault

**Problem:** The "Save Forever" button in `phto-ui/src/pages/Vault.tsx` line 203 is a TODO stub — it does nothing.

**Backend is fully implemented:**
- `POST /api/v1/projects/{id}/forever` — `ProjectHandler.CreateForeverGalleryCheckout`
- `ProjectService.CreateForeverGalleryCheckout(projectID, userID)` — creates DodoPayments checkout for project protection
- `ProjectService.MarkProjectProtected(projectID)` — sets `is_protected = true` on the project
- Webhook handler for one-time payment success calls `MarkProjectProtected`

**Frontend fix:**
In `Vault.tsx`, replace the TODO stub:
```tsx
// Replace:
onClick={(e) => { e.stopPropagation(); /* TODO: link to upgrade flow */ }}

// With (approximate):
onClick={async (e) => {
  e.stopPropagation()
  const { checkout_url } = await api.post(`/projects/${project.id}/forever`)
  window.location.href = checkout_url
}}
```

Add a `useMutation` hook via React Query for cleaner loading/error state.

**Verify:** Click "Save Forever" on a vault project, confirm DodoPayments checkout opens, complete payment in test mode, confirm project `is_protected = true` in DB.

---

### P1-5 · Persist Inquiry Pipeline Status

**Problem:** Dragging inquiry cards between kanban columns (New / Qualifying / Proposal Sent / Booking) updates local state only, with an explicit comment: `// TODO: persist via PATCH /api/v1/inquiries/:id/pipeline_status when backend supports it`.

**Backend changes needed:**
1. Add `PipelineStatus string` field to the `Inquiry` model (or `InquiryLead` if that's the actual model name). Valid values: `"new"`, `"qualifying"`, `"proposal_sent"`, `"booking"`.
2. Add DB migration.
3. Add `PATCH /api/v1/inquiries/{id}/pipeline_status` handler:
   ```go
   // Request body: { status: "qualifying" }
   // Auth: must be the creator who owns this inquiry
   // Updates: inquiry.pipeline_status = req.Status
   ```
4. Return the updated inquiry.

**Frontend changes:**
1. In `phto-ui/src/pages/Inquiries.tsx`, after local state update on drag, call the PATCH endpoint.
2. Use `useMutation` from React Query — on error, revert the local state (optimistic update pattern).

**Verify:** Drag a card, refresh the page, confirm the card is in the new column.

---

### P1-6 · Fix Profile Photo Upload for All Tiers

**Problem:** `AvatarImageSelector.tsx` only pulls photos from the user's "My Studio" project. Free-tier (Starter) clients don't have a Studio project, so they effectively can't set a profile photo at all.

**Fix:**
Add a secondary option: "Upload from device" — a standard `<input type="file" accept="image/*">` that uploads directly without needing a project.

**Steps:**
1. Add a "Upload new photo" button/tab alongside the existing media selector.
2. On file select, upload to a dedicated endpoint or to the Studio project if it exists, or to a special `avatar` upload endpoint.
3. Simplest approach: call `POST /api/v1/users/me/avatar` (add this endpoint) that accepts a multipart image, stores it, and updates `users.avatar_url` (or `creator_profiles.avatar_url`).
4. Alternatively, upload to OCI Object Storage directly and update the avatar URL via `PUT /api/v1/users/me` or `PUT /api/v1/creators/profile`.

**Subscription note:** Avatar upload should be free for all tiers. Do NOT run this through `HasStorageSpace` — it's a profile feature, not project storage.

---

### P1-7 · Fix Proposal Builder Cover Photos

**Problem:** Cover photos in the proposal builder are not loading. The backend `PATCH /api/v1/proposals/{id}/cover` endpoint exists and stores a `media_id`, but the signed URL for the cover photo is likely not being included in `proposalToResponse`.

**Debugging steps:**
1. Check `phto-api/internal/services/proposal_service.go` (or wherever `proposalToResponse` is defined) — is the cover `media_id` being resolved to a signed URL?
2. Check the frontend `ProposalBuilder.tsx` — is it reading `proposal.cover_url` or similar field?
3. Check the signing logic — the signer may require the media object to exist in the DB with the correct `project_id` scope.

**Likely fix:** In `proposalToResponse`, if `proposal.CoverMediaID != nil`, fetch the media object and call `signer.SignURL(media.StorageKey, expiry)`, include as `cover_url` in the response.

---

### P1-8 · Currency Display — Replace Hardcoded USD

**Problem:** Two places show hardcoded `$` instead of the user's actual currency:
1. Email notification templates: `formatCurrency` in `phto-api/internal/notifications/templates.go` formats as `$%.2f` always.
2. `todo.md`: "Proposal view: show project currency instead of hardcoded dollar sign" (apparently a separate frontend issue).

**User model:** `users.currency` defaults to `"LKR"` (Sri Lankan Rupee). Projects may have their own currency field.

**Backend fix:**
In `phto-api/internal/notifications/templates.go`:
```go
// Replace:
func formatCurrency(amount float64) string {
    return fmt.Sprintf("$%.2f", amount)
}

// With:
func formatCurrency(amount float64, currency string) string {
    symbol := currencySymbol(currency)
    return fmt.Sprintf("%s%.2f", symbol, amount)
}

func currencySymbol(currency string) string {
    switch currency {
    case "USD": return "$"
    case "LKR": return "Rs."
    case "EUR": return "€"
    case "GBP": return "£"
    default: return currency + " "
    }
}
```
Pass the `currency` from the relevant project or user when constructing notification data.

**Frontend fix:**
Find where the proposal view hardcodes `$` and replace with `proposal.currency` or a `formatCurrency(amount, currency)` utility.

---

### P1-9 · Branding — Replace "phto" with "zeyaroo" in Emails

**Problem:** Email content still uses "phto" as the product name.

**File:** `phto-api/internal/notifications/templates.go` — the `wrapEmailTemplate` function and individual notification message strings.

**Fix:** Search for `"phto"` (case-insensitive) in all notification template files and replace with `"Zeyaroo"`. Also check the `From` address in `EmailService` — should be `noreply@zeyaroo.com` or similar, not a phto domain.

---

### P1-10 · Auto-Complete Non-Deliverable Milestones After Online Payment

**Problem:** When a client pays online (via DodoPayments) for a milestone with `deliverable_type = "none"`, the milestone should auto-complete (no deliverable to review). This works for manual funding (`ApproveManualFunding` checks `DeliverableTypeNone` and auto-completes), but the DodoPayments webhook path (`HandleDodoPaymentsWebhook`) does not.

**File:** `phto-api/internal/services/escrow_service.go` — `HandleDodoPaymentsWebhook`

**Fix:** After successfully recording the payment and updating the milestone to funded, check:
```go
if milestone.DeliverableType == models.DeliverableTypeNone {
    // auto-complete: release escrow, mark milestone complete
    if err := s.completeMilestone(ctx, milestone); err != nil {
        // log but don't fail the webhook
    }
}
```
Make sure this mirrors the logic in `ApproveManualFunding`.

---

### P1-11 · Disable Escrow for Sri Lanka

**Problem:** `todo.md` notes: "Disable Zeyaroo escrow for Sri Lanka". Sri Lankan financial regulations may restrict or complicate holding funds in escrow for domestic transactions. For LKR projects, the workflow should skip escrow and treat payments as direct.

**Decision to make:** What exactly does "disable escrow" mean for the UX?
- Option A: For LKR-currency projects, milestones are marked as "direct payment" — client pays creator directly (bank transfer, etc.) and photographer manually marks as funded.
- Option B: Zeyaroo acts as invoice-only for LKR; escrow is only for foreign-currency projects.

**Implementation sketch (Option A):**
1. Add a `is_escrow_enabled bool` flag on `Project` (derived from currency or explicit toggle).
2. For LKR projects, skip `CreateMilestoneCheckout` / DodoPayments flow.
3. Show "Manual Payment" workflow: client sees bank details, photographer marks as received.
4. The existing `MarkManuallyFunded` handler already exists for this purpose.

**Frontend:** For LKR projects, hide the "Pay with card" button, show bank transfer instructions and a "Mark as paid" button for the photographer.

---

### P1-12 · Post-Cull Media Refresh

**Problem:** After culling or returning photos, the gallery UI shows stale state — the counts and media grid don't reflect the changes until a full page reload.

**File:** `phto-ui/src/pages/` — wherever cull/return mutations are defined.

**Fix:** In the `onSuccess` callback of the cull and return mutations, invalidate the relevant React Query keys:
```tsx
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: ['media', projectId] })
  queryClient.invalidateQueries({ queryKey: ['albums', projectId] })
}
```
Confirm the exact query keys used in the codebase.

---

### P1-13 · Fix Select-All Checkbox in Project Gallery

**Problem:** The top-left checkbox in the project hub gallery does not work. It should select all photos (same as Ctrl+A / Ctrl+click behavior).

**File:** `phto-ui/src/components/MediaGrid.tsx` — the checkbox component.

**Fix:** Wire the checkbox `onChange` to call whatever the multi-select handler is (likely sets a `selectedIds` state to all media IDs when checked, and empties it when unchecked).

---

### P1-14 · Inquiry Category Filtering

**Problem:** The inquiry dialog shows all available service categories, not filtered to the categories the creator has set up in their profile.

**File:** `phto-ui/src/components/InquiryDialog.tsx` (or similar)

**Fix:**
1. Fetch the creator's `service_categories` from their profile (already in `CreatorProfile.ServiceCategories`).
2. Filter the category list shown in the dialog to only those categories, plus an "Other" option.
3. The creator profile endpoint `GET /api/v1/creators/{id}` already returns this data.

---

### P1-15 · Onboarding Wizard — Implement Multi-Step Flow

**Problem:** `OnboardingWizard.tsx` is a stub modal with a single link to `/settings/profile`. The `onComplete` prop is accepted but never called. New creator accounts have no guided setup.

**Recommended steps for the wizard:**
1. **Welcome** — "Welcome to Zeyaroo" with the creator's name
2. **Profile setup** — display name, bio, avatar upload
3. **Service categories** — pick from available categories
4. **Payout details** — bank name, account number, SWIFT (can skip for later)
5. **Done** — "You're ready to start" with CTA to create first project

**Implementation:**
- Implement as a multi-step modal using local state for `currentStep`.
- On step completion, call the relevant PUT endpoints (`PUT /api/v1/creators/profile`).
- Call `onComplete()` prop when wizard is finished or skipped.
- Show it to creators who haven't completed setup (check a `onboarding_completed` flag or check if `display_name` is the default).

---

## P2 — Important, Can Be Done Mid-Beta

---

### ~~P2-1 · Add Payout Request Flow~~ ✓ (POST /billing/payout-request + GET /billing/payout-requests added; frontend shows amount/notes input and pending request status; admin reviews manually)

**Problem:** The "Request Payout" button in the Billing page is purely cosmetic — it only animates locally with no API call.

**Backend foundation exists:** `CreatorProfile` stores bank details (`bank_name`, `account_holder_name`, `account_number`, `swift_code`). `DodoPaymentsGateway.TriggerPayout` exists.

**What needs to be built:**
1. `POST /api/v1/billing/payout-request` — creates a payout request record (new table `payout_requests`) with `amount`, `currency`, `status = "pending"`.
2. Admin reviews and manually approves (for now — can automate later).
3. On approval, call `DodoPaymentsGateway.TriggerPayout`.
4. Notify creator via email/in-app when payout is processed.

**Frontend:**
- The "Request Payout" button calls `POST /api/v1/billing/payout-request`.
- Show payout request status in the billing page.
- "Available for Payout" balance must be computed server-side (released milestone funds minus fees minus previous payouts).

**Note:** This is a significant workflow. For beta, consider a manual process: creator emails support, you process manually. Add the UI flow in parallel.

---

### P2-2 · Admin Panel (Basic)

**Problem:** No admin UI exists. Dispute resolution, tier management, and manual invoice generation all require direct API calls.

**Backend admin endpoints already exist:**
- `PATCH /api/v1/admin/users/{id}/tier` — set subscription tier
- `POST /api/v1/billing/invoices/generate` (admin only) — generate invoice
- Dispute resolution via admin endpoints in the escrow handler

**Minimum admin UI for beta:**
make a new project + git repo for this. let's keep it seperate.
1. Route `/admin` behind `role === "admin"` check.
2. User list with ability to change tier.
3. Dispute list with resolution actions.
4. This can be a simple, unstyled table-based UI — it's internal only.

---

### P2-3 · Separate Firebase Projects (Dev / Prod)

**Problem:** A single Firebase project is used across all environments. Test accounts and production accounts share the same auth namespace.

**Steps:**
1. Create two Firebase projects in the Firebase Console: `zeyaroo-dev` and `zeyaroo-prod`.
2. Update `phto-ui/.env.development` to use `zeyaroo-dev` credentials.
3. Update `phto-ui/.env.production` to use `zeyaroo-prod` credentials.
4. Update the backend `FIREBASE_PROJECT_ID` and `FIREBASE_CREDENTIALS` env vars for each deployment environment.
5. Migrate any existing real accounts to the prod Firebase project if needed.

---

### ~~P2-4 · Formalize Local Development .env~~ ✓ (both .env.example files already exist)

**Problem:** There is no dedicated `.env` file for local development. Developers use production or ad-hoc credentials locally.

**Steps:**
1. Create `phto-api/.env.example` with all required env var keys (no values).
2. Create `phto-ui/.env.example` similarly.
3. Document in README: copy `.env.example` to `.env.local` and fill in dev values.
4. Add `.env.local` and `.env` to `.gitignore` if not already.
5. Set up a dev Resend sandbox, dev DodoPayments test credentials, and dev Firebase project.

---

### ~~P2-5 · Profile Slug / Custom Creator URL~~ ✓ (Slug field added to CreatorProfile model; POST /creators/profile/claim-slug + GET /creators/slug/{slug} + GET /creators/profile/suggest-slug; frontend settings page has claim UI with suggestion button and public URL display)

**Problem:** Creator profiles are accessed by UUID. Creators can't share a clean link like `zeyaroo.com/c/johndoe`.

**Backend changes:**
1. Add `Slug string` to `CreatorProfile` model with a unique index.
2. Add `POST /api/v1/creators/profile/claim-slug` — validates slug format (alphanumeric + hyphens, 3–30 chars), checks uniqueness, sets it.
3. Add `GET /api/v1/creators/:slug` — resolve slug to creator profile (alongside existing UUID route).
4. Auto-suggest a slug on onboarding based on display name.

**Frontend changes:**
- Add slug field to profile settings.
- Show the creator's public URL once claimed.
- Share button generates the slug URL.

**Premium consideration:** `todo2.md` mentions this could be a premium feature (custom domain for Studio tier). The slug itself can be free; custom domains can be Studio-only.

---

### ~~P2-6 · Proposal Date Pickers~~ ✓ (all three use native type="date"; chapter deadline styling improved to match start/end date inputs)

**Problem:** Date fields in the proposal builder (chapter deadlines, logistics dates) use plain text inputs or a `type="date"` with minimal styling rather than a proper date picker.

**File:** `phto-ui/src/components/ProposalBuilder.tsx` around line 179 and logistics section.

**Fix:** Replace date text inputs with a date picker component. Use an existing date picker library already in the project (check `package.json` for `react-day-picker`, `@radix-ui/react-popover`, etc.) or add one. The picker should open on click and return an ISO date string.

---

### ~~P2-7 · WhatsApp Client Notification Preference Gating~~ ✓ (added ProposalWhatsApp field to notificationPrefs; gated HandleProposalVersionSubmitted and HandleBriefRequestSent)

**Problem:** WhatsApp notifications are sent to clients unconditionally:
```go
// TODO: gate by client WhatsApp preference once client notification prefs are built out
```
Line 116 in `phto-api/internal/services/notification_whatsapp_channel.go`.

**Fix:**
1. Extend `NotificationSettings` (currently a JSON string on `User`) to include `whatsapp_enabled bool` for clients.
2. Before sending a WhatsApp message to a client, check `client.NotificationSettings.WhatsAppEnabled`.
3. Default to `true` for backward compatibility (don't silently drop existing notifications).
4. Add a notification preferences UI for clients in their settings.

---

### ~~P2-8 · Verify Image Post-Processing Works End-to-End~~ ✓ (pipeline fully wired — CompleteDirectUpload emits media.uploaded → EventProcessor job queue → ImageProcessingService.ProcessImage generates _thumb/_medium/_web_preview variants; processing_status tracked in DB)

**Problem:** `todo2.md` notes: "are uploaded media being post processed -- create thumbnails etc." — this is an open question, not a confirmed gap.

**Investigation steps:**
1. Upload a photo through the UI.
2. Check the DB `media_objects.processing_status` field — does it update from `"processing"` to `"ready"`?
3. Check OCI Object Storage for `_thumb`, `_medium`, `_web_preview` variant files alongside the original.
4. Check the image processing worker/service — is it running? Check Komodo for the worker container status.

**If broken:**
- Check that the image processing job is triggered after `CompleteDirectUpload`.
- Check worker logs for errors.
- Confirm OCI credentials in the worker's environment.

---

### ~~P2-9 · Manual Milestone Funding UI~~ ✓ (already implemented — canMarkManuallyFunded + Mark as Paid button in MilestoneCard, confirmed wired in MilestoneList)

**Problem:** `todo.md`: "Allow photographer to manually mark milestone as funded". The backend `MarkManuallyFunded` handler exists, but it may not be exposed in the UI.

**Fix:** In `MilestoneList.tsx` or `MilestoneCard.tsx`, add a "Mark as Funded" button visible only to the project owner (creator), only when the milestone is in `pending_payment` status and the project is an LKR/manual-payment project. This is particularly important for the Sri Lanka escrow-disabled flow (P1-11).

---

## Pre-Beta Launch Checklist Summary

| # | Task | Priority | Estimated Complexity |
|---|---|---|---|
| P0-1 | Remove eruda | P0 | XS — 3 line deletion |
| P0-2 | Remove /debug page | P0 | XS — delete file + route |
| P0-3 | Rotate Resend key, move to env | P0 | XS — ops task |
| P0-4 | Fix TransferDialog crash | P0 | S — 5 line fix |
| P0-5 | Wire DodoPayments subscription → tier upgrade | P0 | M — investigate + possibly add handler |
| P0-6 | Enforce storage limits on upload | P0 | M — wire FeatureGate into 2 handlers |
| P1-1 | Password reset flow | P1 | S — Firebase call + minimal UI |
| P1-2 | Email verification | P1 | S — Firebase call + banner UI |
| P1-3 | Billing page real data | P1 | L — needs backend summary endpoint + frontend wiring |
| P1-4 | Wire "Save Forever" button | P1 | S — frontend mutation + checkout redirect |
| P1-5 | Persist inquiry pipeline status | P1 | M — backend field + PATCH endpoint + frontend mutation |
| P1-6 | Profile photo upload for all tiers | P1 | M — new avatar upload endpoint + UI |
| P1-7 | Fix proposal cover photos | P1 | S — debug + fix URL signing |
| P1-8 | Currency display (not hardcoded USD) | P1 | S — formatCurrency utility fix |
| P1-9 | Replace "phto" with "zeyaroo" in emails | P1 | XS — string replacement |
| P1-10 | Auto-complete non-deliverable milestones | P1 | S — add check in webhook handler |
| P1-11 | Disable escrow for Sri Lanka | P1 | L — architectural decision + UI changes |
| P1-12 | Post-cull media refresh | P1 | XS — invalidate query on mutation success |
| P1-13 | Fix select-all checkbox | P1 | XS — wire onChange handler |
| P1-14 | Inquiry category filtering | P1 | S — filter list by creator profile |
| P1-15 | Onboarding wizard | P1 | L — multi-step modal with API calls |
| P2-1 | Payout request flow | P2 | XL — new table, endpoint, admin workflow |
| P2-2 | Admin panel | P2 | L — new route, user/dispute management UI |
| P2-3 | Firebase dev/prod split | P2 | M — Firebase console + env config |
| P2-4 | Formalize local .env | P2 | XS — .env.example files + README |
| P2-5 | Profile slug | P2 | M — DB field + claim endpoint + frontend |
| P2-6 | Proposal date pickers | P2 | S — replace inputs with picker component |
| P2-7 | WhatsApp client preference gating | P2 | S — check pref before sending |
| P2-8 | Verify image post-processing | P2 | S — investigation + fix if broken |
| P2-9 | Manual milestone funding UI | P2 | S — add button in MilestoneCard |
