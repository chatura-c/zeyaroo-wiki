# E2E Implementation Sessions

Ordered implementation plan for growing from 18 → 90+ tests across 10 sessions.
Each session is designed to be completable in one Claude Code conversation.

Reference: `e2e-automation-plan.md` for full test tables.
Reference: `e2e-test-scenarios.md` for scenario detail.

---

## How to run a session

Open `e2e/` in Claude Code and say:
> "Implement Session N from `wiki/guides/e2e-implementation-sessions.md`"

At the end of every session, run `npm test` from `e2e/` to confirm all existing
tests still pass and the new ones are green before moving on.

---

## Session 0 — Infrastructure

**Goal**: All shared plumbing in place so every future session can just write tests.
No new spec files. No test count change.

### Tasks

**`helpers/types.ts`** — create new file
Extract all existing interfaces out of `helpers/api.ts` and add the new ones
needed across the plan. Minimum set to create:

```
AuthResult, InquiryResult, MilestoneResult, MilestoneDetailResult,
ProposalVersionResult, ProposalResult, ProjectResult, CreatorProfileResult,
AlbumResult, MediaResult, ListMediaResult, ShareResult, PublicShareResult,
ShareAccessResult, ShareMediaParams, ShareListMediaResult,
AmendmentResult, MilestoneAmendmentResult, DraftMilestoneInput,
TransferResult, HandoverResult, NotificationResult,
StorageResult, StorageUsageResult,
InvoiceResult, PaymentResult, PayoutRequestResult, EarningsResult,
AdminStatsResult, AdminUserResult, DisputeResult,
BriefResult, PublicBriefResult, BriefAnswersInput, BriefRequestInput,
UpdateUserInput, UpdateProjectInput, UpdateMilestoneInput,
CreateShareInput, DirectProposalInput, GuestInquiryInput,
MilestonesSummary, AlbumResult, DeliveryPreviewResult, DeliverySummaryResult
```

**`helpers/api.ts`** — expand (keep existing functions, add all new ones)

Add every helper listed in the automation plan sections 1.1, 2.1, and 3.1.
Group them with comments matching the plan. Update `startMilestone`,
`completeMilestone`, `releaseMilestone` to return `MilestoneDetailResult`
instead of `void`.

**`fixtures/index.ts`** — add 3 new fixtures

```typescript
// adminAuth: registers/logs in admin using ADMIN_EMAIL / ADMIN_PASSWORD env,
// falls back to POST /admin/login with defaults from the test env.
adminAuth: AuthResult

// adminPage: browser page with admin auth injected
adminPage: Page

// projectWithMilestones: full inquiry→proposal(2 milestones)→accept→project
projectWithMilestones: {
  creatorAuth: AuthResult;
  clientAuth: AuthResult;
  projectId: string;
  proposalId: string;
  milestones: MilestoneResult[];
}

// fundedMilestone: extends projectWithMilestones — funds first milestone, starts it
fundedMilestone: {
  creatorAuth: AuthResult;
  clientAuth: AuthResult;
  projectId: string;
  milestoneId: string;
  deliveryAlbumId: string;  // album ID from getMilestoneDeliveryAlbum
}
```

The `projectWithMilestones` milestone setup: one `media` type + one `none` type,
both with `amount_cents: 10000`.

**`playwright.config.ts`** — already done (JUnit reporter added for CI)

**`package.json`** — already done (`test:report` script added)

### Verify

```bash
cd e2e && npm test   # all 18 existing tests still pass
```

---

## Session 1 — Auth + Full Happy Path

**Goal**: Expand auth coverage and add the most important E2E test.

**Prerequisite**: Session 0 complete.

### Files

**`specs/auth.spec.ts`** — expand 4 → 8 tests

Add to the existing describe block:
- Duplicate email registration returns 409
- Wrong password returns 401
- Token refresh returns new JWT, new token works on `/users/me`
- Creator auto-gets studio project + portfolio album on signup
  (register, then `listProjects` → assert studio project exists;
  `getCreatorProfile` → assert portfolio album exists)

**`specs/full-happy-path.spec.ts`** — new, 2 tests

Test 1: Full flow via `projectWithMilestones` + `fundedMilestone` fixtures for
setup, then drive the rest via API:
upload media → `submitDelivery` → `releaseMilestone` → assert earnings > 0 →
`initiateHandover` → `acceptHandover` → assert project in client vault.
Add one UI assertion: creator page shows project status "handed_over" (or
equivalent).

Test 2: Negotiation flow — no fixture, drive fully via API:
register creator + client → inquiry → creator sends version → client counters
(`createProposalVersion` as client) → creator accepts (`acceptProposal`) →
assert proposal status = "finalized".

### Verify

```bash
cd e2e && npm test specs/auth.spec.ts specs/full-happy-path.spec.ts
```

Expected: 10 tests pass. Total: ~28.

---

## Session 2 — Proposal Negotiation

**Goal**: Dedicated spec for the proposal state machine. Retire `creator-client-flow.spec.ts`.

**Prerequisite**: Session 0 complete.

### Files

**`specs/proposal-negotiation.spec.ts`** — new, 6 tests

1. Creator sends proposal → proposal status = "negotiating"
2. Client accepts → proposal status = "finalized"
3. Client counters → proposal status = "pending"
4. Creator accepts client version → proposal status = "finalized"
5. Cancel proposal in "negotiating" state → status = "canceled"
6. Cancel already-finalized proposal → 400

**`specs/creator-client-flow.spec.ts`** — audit then delete

Read the file, check if any test covers something *not* in proposal-negotiation
or auth. If unique, move the assertion into the appropriate new spec.
Then delete the file.

### Verify

```bash
cd e2e && npm test specs/proposal-negotiation.spec.ts
```

Expected: 6 tests pass. Run full suite and confirm no regressions.
Total: ~34.

---

## Session 3 — Milestone Escrow

**Goal**: Full escrow lifecycle including the `media` deliverable type.

**Prerequisite**: Session 0 complete (`fundedMilestone` fixture needed).

### Files

**`specs/milestone-escrow.spec.ts`** — expand 2 → 6 tests

Refactor existing tests to use `projectWithMilestones` / `fundedMilestone`
fixtures instead of the local `setupProposalWithProject()` helper. Then add:

3. Full escrow with deliverables: use `fundedMilestone` fixture, upload media
   to `deliveryAlbumId`, call `submitDelivery`, call `releaseMilestone` as
   client, assert milestone status = "completed"
4. Client releases via UI: use `fundedMilestone`, navigate to project milestones
   tab as client, click release button, assert "Released" label visible
5. Sequential milestone funding enforced: use `projectWithMilestones` (2
   milestones), try to fund milestone index 1 before index 0 → assert 400
6. Fund already-funded milestone → assert 400

### Verify

```bash
cd e2e && npm test specs/milestone-escrow.spec.ts
```

Expected: 6 tests pass. Total: ~40.

---

## Session 4 — Shares + Handover

**Goal**: Public gallery and project handover flows.

**Prerequisite**: Session 0 + Session 3 (fundedMilestone fixture used by handover).

### Files

**`specs/shares.spec.ts`** — new, 5 tests

1. Creator creates share with password → share listed with token
2. Public gallery: wrong password shows error (UI)
3. Public gallery: correct password → media grid visible (UI)
4. Public gallery: album filter returns only that album's media (API+UI)
5. Creator deletes share → `getPublicShare` returns 404

Use `projectWithMilestones` for setup, upload 1 media file before creating the share.

**`specs/handover.spec.ts`** — new, 5 tests

1. `initiateHandover` after all milestones released → handover created
2. `acceptHandover` as client → project `owner_id` = client id
3. Client sees project in `/vault` (UI)
4. `declineHandover` → project ownership unchanged
5. Creator storage usage decreases after handover accepted (record usage before
   and after `acceptHandover`, assert `after.used_bytes < before.used_bytes`)

Use `fundedMilestone` fixture and complete the milestone first for tests 1-3, 5.

### Verify

```bash
cd e2e && npm test specs/shares.spec.ts specs/handover.spec.ts
```

Expected: 10 tests pass. Total: ~50.

---

## Session 5 — Media + Albums

**Goal**: Media management and album CRUD.

**Prerequisite**: Session 0 complete.

### Files

**`specs/project-media.spec.ts`** — expand 3 → 8 tests

Keep the 3 existing UI tests. Add:
4. Delete media as owner → media no longer in list
5. Add media to album → appears in `getAlbumMedia`
6. Remove media from album → no longer in `getAlbumMedia`
7. Cull status: accept then reject via API, assert status field updates
8. Purge rejected media → `purgeRejectedMedia` returns `deleted > 0`,
   rejected media no longer in list

Use `projectWithMilestones` for setup and a helper to upload test media.

**`specs/albums.spec.ts`** — new, 4 tests

1. Rename album → `getAlbumMedia` returns updated name
2. Cannot rename system album (Portfolio) → 403
3. Delete custom album → 204
4. Cannot delete milestone delivery album → 403

### Verify

```bash
cd e2e && npm test specs/project-media.spec.ts specs/albums.spec.ts
```

Expected: 9 new tests (12 total across both files). Total: ~62.

---

## Session 6 — Amendments + Project Lifecycle

**Goal**: Contract amendments and project state machine (archive/trash).

**Prerequisite**: Session 0 + Session 3 (funded milestone needed for amendment guard tests).

### Files

**`specs/amendments.spec.ts`** — new, 6 tests

1. Create draft → add 2 milestones → publish → client applies → new milestones
   appear in `getProjectMilestones`
2. Client rejects published amendment with reason → status = "rejected",
   project milestones unchanged
3. Cannot delete funded milestone from draft → 422
4. Cannot reduce funded milestone amount → 422
5. Creator cancels draft amendment → status = "cancelled"
6. Public amendment page: navigate to `/p/:token/amendment/:atoken` (UI),
   click approve → amendment status = "applied"

**`specs/project-lifecycle.spec.ts`** — new, 7 tests

1. Archive project → not in active list
2. Restore archived project → back in active list
3. Cannot archive project with funded milestone → 400
4. Trash project → appears in `getTrashedProjects`
5. Restore trashed project → active again
6. Cannot trash system (studio) project → 403
7. `getProjectStorage` bytes increase after uploading media

### Verify

```bash
cd e2e && npm test specs/amendments.spec.ts specs/project-lifecycle.spec.ts
```

Expected: 13 tests pass. Total: ~75.

---

## Session 7 — Transfer + Billing (Accounting)

**Goal**: Project transfer + the full accounting test suite (commission math,
invoice generation, payout lifecycle).

**Prerequisite**: Session 0 + Session 3 (need a completable milestone for earnings tests).

### Files

**`specs/transfer.spec.ts`** — new, 4 tests

1. Initiate transfer, recipient accepts → project `owner_id` updated
2. Recipient declines → ownership unchanged
3. Original owner cannot `updateProject` after accepted transfer → 403
4. Pending transfer appears in `getPendingTransfers` for recipient

**`specs/billing.spec.ts`** — expand 6 → 18 tests

Keep all 6 existing tests. Add:
7. Pay invoice via mock gateway → payment appears in `listPayments`
8. Release milestone → platform commission invoice auto-created for creator
   (`listInvoices` or `listPayments` shows a `commission` line item)
9. Commission math: after release, `earnings.available_cents` ==
   `milestone.amount_cents - commission_invoice.total_cents`
   (assert with exact values, derive platform fee from the invoice)
10. Creator requests payout → appears in `listPayoutRequests`
11. Admin approves creator payout request → `listPayoutRequests` status = "approved"
12. Earnings available balance correct after milestone release
    (separate from test 9 — this one tests `getCreatorEarnings` response shape)
13. Billing page shows earnings total + payout history (UI)
14. Plan upgrade changes `subscription_tier` on user
15. Storage usage shown on billing settings page (UI)
16. Manual milestone funding via `markManuallyFunded` → service fee invoice
    auto-generated
17. Invoice totals correct after admin adds line item + applies discount
18. Subscription fee invoice created when creator subscribes to plan

> **Test 9 implementation note**: call `releaseMilestone`, then fetch the
> commission invoice and the earnings in the same assertion block. Don't
> hardcode the platform fee percentage — read it from the invoice so the test
> survives a fee change.

### Verify

```bash
cd e2e && npm test specs/transfer.spec.ts specs/billing.spec.ts
```

Expected: 16 new tests (22 total across both files). Total: ~91.

---

## Session 8 — Disputes + Notifications + Public Pages + Admin

**Goal**: Admin workflows, dispute resolution, notification CRUD, public unauthenticated pages.

**Prerequisite**: Session 0 + Session 3 (need a funded milestone for disputes).
**Prerequisite**: `NotificationEventProcessor` wired in `phto-api/services.go`
before notification test 1 will pass. If not yet wired, skip test 1 and leave
a `test.skip` with a comment.

### Files

**`specs/milestone-dispute.spec.ts`** — new, 3 tests

1. Creator disputes funded milestone → milestone status = "disputed"
2. Admin resolves dispute with action="release" → status = "completed"
3. Admin resolves dispute with action="refund" → status = "pending" (refunded)

**`specs/notifications.spec.ts`** — new, 3 tests

1. Inquiry triggers notification for creator → `listNotifications` returns it
   with `unread_count > 0` *(skip if NotificationEventProcessor not wired)*
2. Mark notification as read → `unread_count` decreases
3. Mark all notifications read → `unread_count` = 0

**`specs/public-pages.spec.ts`** — new, 4 tests

1. Navigate to `/p/:proposalToken` → proposal title visible on page (UI)
2. `sendGuestInquiry` without auth → inquiry created with null `client_id`
3. Public brief: `createBriefRequest` → navigate to `/brief/:token` → fill
   answers → submit → brief status = "completed" (API+UI)
4. Navigate to public payment page `/pay/:token` → milestone amount visible (UI)

**`specs/admin.spec.ts`** — new, 4 tests

1. `getAdminStats` returns object with user counts and revenue fields
2. `listAdminUsers` returns users; search by email returns subset
3. `updateAdminUserTier` changes user `subscription_tier`
4. `listAdminDisputes` returns disputes after one is created

### Verify

```bash
cd e2e && npm test specs/milestone-dispute.spec.ts specs/notifications.spec.ts specs/public-pages.spec.ts specs/admin.spec.ts
```

Expected: 14 tests pass (13 if notification test 1 skipped). Total: ~105.

---

## Session 9 — Edge Cases, Templates, Storage, Settings

**Goal**: Error paths, quota enforcement, template CRUD, notification settings.

**Prerequisite**: Session 0 complete.

### Files

**`specs/edge-cases.spec.ts`** — new, 5 tests

1. Client calls creator-only endpoint → 403
2. User calls another user's project endpoint → 403
3. Accept proposal in "inquiry" state (before version sent) → 400
4. Create proposal version on finalized proposal → 403
5. Register with missing email field → 400; create project with empty title → 400

**`specs/templates.spec.ts`** — new, 3 tests

1. Create milestone template → list → delete
2. Create brief template → list → delete
3. Apply milestone template in proposal builder UI (navigate to
   `/proposals/:id/builder`, select template, assert chapters populated)

**`specs/notification-settings.spec.ts`** — new, 2 tests

1. `updateNotificationSettings` persists → `getNotificationSettings` returns
   the updated values
2. Settings page toggles: navigate to `/settings/notifications`, toggle one
   off, reload, assert toggle still off (UI)

**`specs/storage.spec.ts`** — new, 4 tests

1. `getStorageUsage` bytes increase after upload
2. Upload blocked when at quota (`updateAdminUserTier` to a tiny quota first,
   or mock the limit)
3. `getStorageUsage` bytes decrease after `deleteMedia`
4. Creator storage freed after `acceptHandover` — record usage before handover,
   accept, assert `used_bytes` decreased by at least the uploaded media size

### Verify

```bash
cd e2e && npm test specs/edge-cases.spec.ts specs/templates.spec.ts specs/notification-settings.spec.ts specs/storage.spec.ts
```

Expected: 14 tests pass. Run full suite.

```bash
cd e2e && npm test
```

Final target: 90+ tests, all green.

---

## Reporting

### Local

After any run, open the HTML report:

```bash
cd e2e && npm run test:report
```

Screenshots of failures are in `test-results/`. Traces are captured on retry.

### CI

On CI (`CI=true`), Playwright writes JUnit XML to `test-results/results.xml`.
The OCI DevOps pipeline can parse this for pass/fail per test. No pipeline
changes needed — just push spec files to trigger the existing
`zeyaroo-e2e-tests` pipeline.

---

## Session Completion Checklist

Before marking a session done:

- [ ] `npm test` passes with no new failures
- [ ] HTML report shows 0 unexpected failures
- [ ] New spec file(s) committed to `e2e/specs/`
- [ ] Any new helpers added to `helpers/api.ts` and types to `helpers/types.ts`
- [ ] No `test.only` left in any file

---

## Progress Tracker

| Session | Status | Tests After |
|---------|--------|-------------|
| 0 — Infrastructure | ✅ | 18 |
| 1 — Auth + Happy Path | ✅ | 26 |
| 2 — Proposal Negotiation | ✅ | ~34 |
| 3 — Milestone Escrow | 🔶 partial (3 pass, 3 skip) | ~37 |
| 4 — Shares + Handover | ✅ (9 pass, 1 skip) | ~47 |
| 5 — Media + Albums | ⬜ | ~62 |
| 6 — Amendments + Project Lifecycle | ⬜ | ~75 |
| 7 — Transfer + Billing | ⬜ | ~91 |
| 8 — Disputes + Notifications + Public + Admin | ⬜ | ~105 |
| 9 — Edge Cases + Templates + Storage + Settings | ⬜ | ~119 |

### Session 3 Notes

3 tests passing, 3 skipped. Skipped tests blocked by two issues documented in `e2e/CLAUDE.md` under "Known Issues":

- **Tests 2 & 4** (UI Pay button): Zustand v5 persist hydration race condition — store hydrates async after React first render, causing API calls with `token: null` → 401 → `logout()` + redirect to `/login`.
- **Test 6** (fund pending media milestone): Backend `fundMilestone` hangs instead of returning 400 for media milestones in pending status.

### Session 4 Notes

9 tests passing, 1 skipped. Skipped test blocked by the same Zustand hydration issue:

- **Test 3** (client sees project in /vault UI): Same Zustand v5 persist hydration race condition.

Discoveries during implementation:

- **Share token `=` padding**: Backend `generateShareToken` uses `base64.URLEncoding` which produces `==` padding. Must use `encodeURIComponent(share.token)` in frontend URL paths (`/gallery/:token`).
- **Upload handler ignores `album_id` form field**: Backend `POST /projects/:id/media` doesn't process the `album_id` form field — media must be linked to albums via separate `POST /media/:id/albums` call.
- **Backend error response shape**: `respondError` returns `{"error": "<HTTP status text>", "message": "<detail>"}` — UI PasswordGate reads `.error` which is the HTTP status text (e.g. "Unauthorized"), not the human-readable message.
- **`getByRole('img')` unreliable in gallery**: Use `page.locator('img[src]').first()` instead for gallery media grid assertions.
