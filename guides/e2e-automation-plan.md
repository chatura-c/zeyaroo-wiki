# Zeyaroo E2E Test Automation Plan

Goal: move from 18 tests (~15% coverage) to 70+ tests covering all P0 and P1 scenarios
from `e2e-test-scenarios.md`, in 4 phased iterations.

---

## Constraints

| Constraint | Impact |
|-----------|--------|
| SQLite, workers: 1 | All tests sequential — keep each test fast; avoid redundant setup |
| Single shared DB per run | Tests must use unique data (timestamp emails); no cross-test state dependencies |
| Firebase disabled in tests | No Google/Facebook SSO testing; all auth via JWT |
| Local storage backend | No B2/S3 presigned URL testing; direct upload via FormData only |
| Backend event processing = 5s | UI assertions need generous timeouts (10-15s) for async processing |

---

## Phase 1 — Foundation & Core Happy Path

**Target: 30 tests, covers all P0 scenarios**

### 1.1 Expand API Helpers (`helpers/api.ts`)

Add ~25 functions needed across all phases. Do this first so every spec file
has what it needs from day one.

```typescript
// ─── Auth ──────────────────────────────────────────────
refreshToken(token: string): Promise<AuthResult>

// ─── Users ─────────────────────────────────────────────
getCurrentUser(token: string): Promise<UserResult>
updateUser(token: string, data: UpdateUserInput): Promise<UserResult>

// ─── Creator Profile ───────────────────────────────────
getCreatorProfile(token: string): Promise<CreatorProfileResult>
claimSlug(token: string, slug: string): Promise<CreatorProfileResult>
suggestSlug(token: string): Promise<{ suggestion: string }>
getCreatorBySlug(slug: string): Promise<CreatorProfileResult>
getCreatorPortfolio(creatorId: string): Promise<{ media: MediaResult[] }>

// ─── Projects ──────────────────────────────────────────
listProjects(token: string, status?: string): Promise<{ projects: ProjectResult[]; total: number }>
updateProject(token: string, id: string, data: UpdateProjectInput): Promise<ProjectResult>
deleteProject(token: string, id: string): Promise<void>
archiveProject(token: string, id: string): Promise<void>
restoreProject(token: string, id: string): Promise<void>
trashProject(token: string, id: string): Promise<void>
restoreTrashProject(token: string, id: string): Promise<void>
getTrashedProjects(token: string): Promise<ProjectResult[]>
getProjectStorage(token: string, id: string): Promise<StorageResult>

// ─── Proposals ────────────────────────────────────────
listCreatorProposals(token: string): Promise<{ proposals: ProposalResult[] }>
listClientProposals(token: string): Promise<{ proposals: ProposalResult[] }>
cancelProposal(token: string, id: string): Promise<void>
createDirectProposal(token: string, data: DirectProposalInput): Promise<ProposalResult>

// ─── Milestones ───────────────────────────────────────
getProjectMilestones(token: string, projectId: string): Promise<MilestonesSummary>
getMilestoneDetail(token: string, id: string): Promise<MilestoneDetailResult>
updateMilestone(token: string, id: string, data: UpdateMilestoneInput): Promise<MilestoneResult>
startMilestone(token: string, id: string): Promise<MilestoneDetailResult>     // already exists, fix return type
completeMilestone(token: string, id: string): Promise<MilestoneDetailResult>  // already exists, fix return type
releaseMilestone(token: string, id: string): Promise<MilestoneDetailResult>   // already exists, fix return type
unlockMilestone(token: string, id: string): Promise<MilestoneDetailResult>
requestChanges(token: string, id: string, reason: string): Promise<MilestoneDetailResult>
disputeMilestone(token: string, id: string, reason: string): Promise<DisputeResult>
submitDelivery(token: string, id: string): Promise<AlbumResult>
getMilestoneDeliveryAlbum(token: string, id: string): Promise<AlbumResult>

// ─── Media ─────────────────────────────────────────────
uploadMedia(token: string, projectId: string, filePath: string): Promise<MediaResult>
deleteMedia(token: string, id: string): Promise<void>
addMediaToAlbums(token: string, mediaId: string, albumIds: string[]): Promise<void>
removeMediaFromAlbum(token: string, mediaId: string, albumId: string): Promise<void>
updateCullStatus(token: string, mediaId: string, status: string): Promise<void>

// ─── Albums ────────────────────────────────────────────
updateAlbum(token: string, id: string, data: { name?: string }): Promise<AlbumResult>
deleteAlbum(token: string, id: string): Promise<void>
getAlbumMedia(token: string, id: string): Promise<ListMediaResult>

// ─── Shares ────────────────────────────────────────────
createShare(token: string, projectId: string, data: CreateShareInput): Promise<ShareResult>
listShares(token: string, projectId: string): Promise<ShareResult[]>
deleteShare(token: string, id: string): Promise<void>
getPublicShare(token: string): Promise<PublicShareResult>
verifySharePassword(token: string, password: string): Promise<ShareAccessResult>
getShareAlbums(accessToken: string, token: string): Promise<AlbumResult[]>
getShareMedia(accessToken: string, token: string, params?: ShareMediaParams): Promise<ShareListMediaResult>

// ─── Amendments ────────────────────────────────────────
createAmendment(token: string, projectId: string): Promise<AmendmentResult>
addDraftMilestone(token: string, projectId: string, amendmentId: string, data: DraftMilestoneInput): Promise<MilestoneAmendmentResult>
publishAmendment(token: string, projectId: string, amendmentId: string): Promise<AmendmentResult>
applyAmendment(token: string, projectId: string, amendmentId: string): Promise<void>
rejectAmendment(token: string, projectId: string, amendmentId: string, reason?: string): Promise<void>
cancelAmendment(token: string, projectId: string, amendmentId: string): Promise<void>

// ─── Transfer & Handover ──────────────────────────────
initiateTransferRequest(token: string, projectId: string, toUserId: string): Promise<TransferResult>
getPendingTransfers(token: string): Promise<{ transfers: TransferResult[] }>
acceptTransfer(token: string, transferId: string): Promise<void>
declineTransfer(token: string, transferId: string): Promise<void>
initiateHandover(token: string, projectId: string): Promise<HandoverResult>
getPendingHandovers(token: string): Promise<{ handovers: HandoverResult[] }>
acceptHandover(token: string, handoverId: string): Promise<void>
declineHandover(token: string, handoverId: string): Promise<void>

// ─── Notifications ─────────────────────────────────────
listNotifications(token: string): Promise<{ notifications: NotificationResult[]; unread_count: number }>
markNotificationRead(token: string, id: string): Promise<void>
markAllNotificationsRead(token: string): Promise<void>

// ─── Admin ─────────────────────────────────────────────
adminLogin(): Promise<string>    // move from billing.spec.ts
getAdminStats(token: string): Promise<AdminStatsResult>
listAdminUsers(token: string): Promise<{ users: AdminUserResult[] }>
updateAdminUserTier(token: string, userId: string, tier: string): Promise<void>
resolveDispute(token: string, milestoneId: string, action: string, notes: string): Promise<void>
```

Also update existing helpers to return typed data (e.g., `startMilestone`,
`completeMilestone`, `releaseMilestone` currently return `void` — change to
return `MilestoneDetailResult` so tests can assert on status).

### 1.2 Add New Fixtures (`fixtures/index.ts`)

```typescript
interface Fixtures {
  // ─── existing ──────────────────────
  creatorAuth: AuthResult;
  clientAuth: AuthResult;
  creatorPage: Page;
  clientPage: Page;

  // ─── new ───────────────────────────
  adminAuth: AuthResult;        // admin@zeyaroo.com JWT
  adminPage: Page;              // page with admin auth

  // ─── composite flow fixtures ──────
  // Reusable "project with milestones" setup — avoids repeating
  // the 4-step inquiry→version→accept→create-project dance
  projectWithMilestones: {
    creatorAuth: AuthResult;
    clientAuth: AuthResult;
    projectId: string;
    proposalId: string;
    milestones: MilestoneResult[];
  };

  // Project with one funded+started milestone ready for delivery
  fundedMilestone: {
    creatorAuth: AuthResult;
    clientAuth: AuthResult;
    projectId: string;
    milestoneId: string;
    deliveryAlbumId: string;
  };
}
```

**`projectWithMilestones` fixture** — does the full setup:
1. Register creator + client
2. Update creator profile (display_name)
3. Client creates inquiry
4. Creator sends proposal version with 2 milestones (one `media` type, one `none`)
5. Client accepts proposal
6. Creator creates project from proposal

**`fundedMilestone` fixture** — extends `projectWithMilestones`:
7. Client funds first milestone
8. Creator starts milestone

These composite fixtures eliminate the repeated `setupProposalWithProject()`
pattern currently duplicated across `milestone-escrow.spec.ts` and
`creator-client-flow.spec.ts`.

### 1.3 Add Types (`helpers/types.ts`)

Extract all interfaces from `api.ts` into a dedicated types file. Add new
result types for every endpoint group. This keeps `api.ts` clean and lets
spec files import only the types they need.

### 1.4 Spec Files for Phase 1

#### `specs/auth.spec.ts` — Expand (4 → 8 tests)

| # | Test | Type |
|---|------|------|
| 1 | Register + login via JWT | API (existing) |
| 2 | Unauthenticated /dashboard redirect | UI (existing) |
| 3 | Injected creator auth reaches dashboard | UI (existing) |
| 4 | Injected client auth reaches dashboard | UI (existing) |
| 5 | Duplicate email registration returns 409 | API |
| 6 | Wrong password login returns 401 | API |
| 7 | Token refresh works and returns new JWT | API |
| 8 | Creator auto-gets studio project + portfolio album on signup | API |

#### `specs/full-happy-path.spec.ts` — NEW (2 tests)

The single most important spec. Tests the entire business flow in one test.

| # | Test | Type |
|---|------|------|
| 1 | Full flow: register → profile → inquiry → proposal → accept → project → fund → start → deliver → complete → release → handover | API+UI |
| 2 | Full flow with negotiation: client counters → creator re-counters → accept | API+UI |

This test uses the `projectWithMilestones` and `fundedMilestone` fixtures for
the early steps, then drives the delivery + handover via API + page assertions.

#### `specs/proposal-negotiation.spec.ts` — NEW (6 tests)

| # | Test | Type |
|---|------|------|
| 1 | Creator sends proposal, proposal → "negotiating" | API |
| 2 | Client accepts, proposal → "finalized" | API (existing, move from creator-client-flow) |
| 3 | Client counters proposal, proposal → "pending" | API |
| 4 | Creator accepts client version, proposal → "finalized" | API |
| 5 | Cancel proposal in valid state | API |
| 6 | Cancel finalized proposal returns 400 | API |

#### `specs/milestone-escrow.spec.ts` — Expand (2 → 6 tests)

| # | Test | Type |
|---|------|------|
| 1 | Full API-driven lifecycle: fund → complete (none type) | API (existing) |
| 2 | Client funds milestone via UI button | UI (existing) |
| 3 | Full escrow with deliverables: fund → start → upload → submit → release | API |
| 4 | Client releases milestone via UI | UI |
| 5 | Sequential milestone funding enforced | API |
| 6 | Fund already-funded milestone returns 400 | API |

#### `specs/shares.spec.ts` — NEW (5 tests)

| # | Test | Type |
|---|------|------|
| 1 | Creator creates share with password | API |
| 2 | Public gallery: wrong password rejected | UI |
| 3 | Public gallery: correct password → media visible | UI |
| 4 | Public gallery: album filter works | UI |
| 5 | Creator deletes share, gallery returns 404 | API+UI |

#### `specs/handover.spec.ts` — NEW (5 tests)

| # | Test | Type |
|---|------|------|
| 1 | Creator initiates handover after all milestones complete | API |
| 2 | Client accepts handover, ownership transfers | API |
| 3 | Client sees project in vault after handover | UI |
| 4 | Client declines handover | API |
| 5 | Creator's storage usage decreases after handover accepted | API |

---

## Phase 2 — Media, Albums, Amendments

**Target: 50 tests total (20 new), covers remaining P0 + key P1**

### 2.1 Additional API Helpers

```typescript
// ─── Media (extended) ──────────────────────────────────
getDeliveryPreview(token: string, projectId: string): Promise<DeliveryPreviewResult>
getDeliverySummary(token: string, projectId: string): Promise<DeliverySummaryResult>
purgeRejectedMedia(token: string, projectId: string): Promise<{ deleted: number }>
updateFavorite(token: string, mediaId: string, isFavorite: boolean): Promise<void>

// ─── Briefs ────────────────────────────────────────────
createBriefRequest(token: string, projectId: string, data: BriefRequestInput): Promise<BriefResult>
getPublicBrief(token: string): Promise<PublicBriefResult>
submitBriefAnswers(token: string, data: BriefAnswersInput): Promise<void>

// ─── Storage ───────────────────────────────────────────
getStorageUsage(token: string): Promise<StorageUsageResult>

// ─── Billing (extended) ────────────────────────────────
payInvoice(token: string, invoiceId: string, gateway: string): Promise<PaymentResult>
listPayments(token: string): Promise<{ payments: PaymentResult[] }>
requestPayout(token: string, data: PayoutRequestInput): Promise<PayoutRequestResult>
listPayoutRequests(token: string): Promise<PayoutRequestResult[]>
getCreatorEarnings(token: string): Promise<EarningsResult>
```

### 2.2 Spec Files for Phase 2

#### `specs/project-media.spec.ts` — Expand (3 → 8 tests)

| # | Test | Type |
|---|------|------|
| 1 | Create project, title visible | UI (existing) |
| 2 | Upload photo, media appears | UI (existing) |
| 3 | Create album via UI | UI (existing) |
| 4 | Delete media as owner | API |
| 5 | Add media to album, verify in album | API |
| 6 | Remove media from album | API |
| 7 | Cull status: accept/reject via API | API |
| 8 | Purge rejected media | API |

#### `specs/albums.spec.ts` — NEW (4 tests)

| # | Test | Type |
|---|------|------|
| 1 | Rename album, verify updated name | API |
| 2 | Cannot rename system album (Portfolio) | API |
| 3 | Delete custom album | API |
| 4 | Cannot delete milestone delivery album | API |

#### `specs/amendments.spec.ts` — NEW (6 tests)

| # | Test | Type |
|---|------|------|
| 1 | Create draft → add milestones → publish → client applies | API |
| 2 | Client rejects amendment with reason | API |
| 3 | Cannot delete funded milestone from draft | API |
| 4 | Cannot reduce funded milestone amount | API |
| 5 | Creator cancels draft amendment | API |
| 6 | Public amendment page: approve via token | API |

#### `specs/project-lifecycle.spec.ts` — NEW (7 tests)

| # | Test | Type |
|---|------|------|
| 1 | Archive project, hidden from active list | API |
| 2 | Restore archived project | API |
| 3 | Cannot archive project with funded milestone | API |
| 4 | Trash project, appears in trash list | API |
| 5 | Restore trashed project | API |
| 6 | Cannot trash system project | API |
| 7 | Project storage usage increases after upload | API |

#### `specs/transfer.spec.ts` — NEW (4 tests)

| # | Test | Type |
|---|------|------|
| 1 | Initiate transfer, recipient accepts | API |
| 2 | Recipient declines transfer | API |
| 3 | Original owner cannot edit after transfer | API |
| 4 | Pending transfer appears in recipient's list | API |

---

## Phase 3 — Billing, Notifications, Public Pages, Admin

**Target: ~75 tests total (~21 new), covers all P1**

### 3.1 Additional API Helpers

```typescript
// ─── Public Proposal ───────────────────────────────────
getPublicProposal(token: string): Promise<PublicProposalResult>
requestProposalOTP(token: string): Promise<void>
verifyProposalOTP(token: string, code: string): Promise<{ token: string; role: string }>
acceptProposalAsGuest(token: string, proposalToken: string): Promise<void>

// ─── Guest Inquiry ────────────────────────────────────
sendGuestInquiry(creatorId: string, data: GuestInquiryInput): Promise<InquiryResult>

// ─── Admin (extended) ─────────────────────────────────
listAdminDisputes(token: string): Promise<{ disputes: DisputeResult[] }>
listAdminEscrowTransactions(token: string): Promise<{ transactions: EscrowTransactionResult[] }>
listAdminPayoutRequests(token: string): Promise<{ requests: PayoutRequestResult[] }>
updateAdminPayoutRequest(token: string, id: string, status: string): Promise<void>
```

### 3.2 Spec Files for Phase 3

#### `specs/billing.spec.ts` — Expand (6 → 18 tests)

| # | Test | Type |
|---|------|------|
| 1-6 | (existing tests) | |
| 7 | Pay invoice via mock gateway, payment appears in listPayments | API |
| 8 | Release milestone → platform commission invoice auto-created | API |
| 9 | Commission math: creator earnings = milestone amount minus platform % | API |
| 10 | Creator requests payout, appears in creator's payout list | API |
| 11 | Admin processes (approves) creator payout request | API |
| 12 | Creator earnings increase after milestone release, available balance correct | API |
| 13 | Billing page shows earnings + payout request | UI |
| 14 | Plan upgrade changes subscription_tier | API |
| 15 | Storage usage shown on billing settings page | UI |
| 16 | Manual milestone funding → service fee invoice auto-generated | API |
| 17 | Invoice totals correct after admin applies line item + discount | API |
| 18 | Subscription fee invoice created on plan subscribe | API |

> **Test 9 note**: assert `earnings.available_cents == milestone.amount_cents - platform_fee_cents`,
> not just that the value is non-zero. Derive `platform_fee_cents` from the invoice line item so
> the test stays correct if the platform fee % changes.

#### `specs/milestone-dispute.spec.ts` — NEW (3 tests)

| # | Test | Type |
|---|------|------|
| 1 | Either party can dispute funded milestone | API |
| 2 | Admin resolves dispute with release | API |
| 3 | Admin resolves dispute with refund | API |

#### `specs/notifications.spec.ts` — NEW (3 tests)

> **Prerequisite**: `NotificationEventProcessor` must be wired in `services.go` before test 1
> will pass. Tests 2-3 can run independently once a notification exists via any trigger.

| # | Test | Type |
|---|------|------|
| 1 | Inquiry creates notification for creator | API |
| 2 | Mark notification as read | API |
| 3 | Mark all notifications read | API |

#### `specs/public-pages.spec.ts` — NEW (4 tests)

| # | Test | Type |
|---|------|------|
| 1 | Public proposal page renders proposal data | UI |
| 2 | Guest inquiry via creator profile (no auth) | API |
| 3 | Public brief: fill answers + submit | API+UI |
| 4 | Public payment page renders milestone info | UI |

#### `specs/admin.spec.ts` — NEW (4 tests)

| # | Test | Type |
|---|------|------|
| 1 | Admin stats endpoint returns data | API |
| 2 | Admin lists + searches users | API |
| 3 | Admin upgrades user tier | API |
| 4 | Admin lists disputes | API |

---

## Phase 4 — Edge Cases, P2 Scenarios, Polish

**Target: 90+ tests total (15+ new)**

#### `specs/edge-cases.spec.ts` — NEW (5 tests)

| # | Test | Type |
|---|------|------|
| 1 | Client cannot access creator-only endpoint | API |
| 2 | User cannot access another user's project | API |
| 3 | Accept proposal in wrong state returns 400 | API |
| 4 | Create version on finalized proposal returns 403 | API |
| 5 | Validation: empty title, missing email | API |

#### `specs/templates.spec.ts` — NEW (3 tests)

| # | Test | Type |
|---|------|------|
| 1 | Create + list + delete milestone template | API |
| 2 | Create + list + delete brief template | API |
| 3 | Apply template in proposal builder | UI |

#### `specs/notification-settings.spec.ts` — NEW (2 tests)

| # | Test | Type |
|---|------|------|
| 1 | Get + update notification settings | API |
| 2 | Settings page toggles persist | UI |

#### `specs/storage.spec.ts` — NEW (4 tests)

| # | Test | Type |
|---|------|------|
| 1 | Storage usage reflects uploads | API |
| 2 | Upload blocked when over quota | API |
| 3 | Storage usage decreases after media deletion | API |
| 4 | Storage freed after project handover (assert creator usage drops by project size) | API |

---

## Implementation Rules

### Test Structure Pattern

Every test follows the same pattern:

```typescript
import { test, expect } from '../fixtures';
import { helperA, helperB } from '../helpers/api';

test.describe('feature name', () => {
  test('description', async ({ creatorAuth, clientAuth, creatorPage }) => {
    // 1. Setup via API (fast, deterministic)
    const project = await createProject(creatorAuth.token, { ... });

    // 2. Exercise via UI or API
    await creatorPage.goto(`/projects/${project.id}`);

    // 3. Assert on UI or API response
    await expect(creatorPage.getByText(project.title)).toBeVisible({ timeout: 10_000 });
  });
});
```

### Timeout Guidelines

| Context | Timeout |
|---------|---------|
| API calls via `fetch` | Default (5s) — API is local, should be fast |
| UI navigation `page.goto()` | Default (30s) — Vite HMR can be slow |
| UI element visibility after API action | 10-15s — backend event processing interval is 5s |
| UI element visibility on static load | 5s |

### Data Isolation

- **Every test uses `uniqueEmail()`** — no shared user state
- **Composite fixtures create fresh entities** — no cross-test project/milestone reuse
- **No test ordering dependency** — any single spec file can run independently
- **DB wipe only at run start** (via webServer command), not between tests —
  tests are responsible for their own data uniqueness

### Spec File Naming Convention

```
specs/
├── auth.spec.ts                    # Registration, login, tokens
├── full-happy-path.spec.ts         # End-to-end business flow
├── proposal-negotiation.spec.ts    # Proposals, versions, state machine
├── milestone-escrow.spec.ts        # Milestone lifecycle + escrow
├── milestone-dispute.spec.ts       # Disputes + admin resolution
├── project-media.spec.ts           # Upload, culling, albums
├── project-lifecycle.spec.ts       # Archive, trash, storage, cover
├── albums.spec.ts                  # Album CRUD, system album protection
├── shares.spec.ts                  # Share creation + public gallery
├── amendments.spec.ts              # Contract amendment lifecycle
├── handover.spec.ts                # Project handover to client
├── transfer.spec.ts                # Project transfer requests
├── billing.spec.ts                 # Subscriptions, invoices, payouts
├── notifications.spec.ts           # Notification CRUD
├── notification-settings.spec.ts   # Preference toggles
├── public-pages.spec.ts            # Unauthenticated pages (proposal, brief, pay)
├── admin.spec.ts                   # Admin dashboard endpoints
├── storage.spec.ts                 # Storage quotas + mode switching
├── templates.spec.ts               # Milestone + brief templates
├── edge-cases.spec.ts              # Auth, validation, state machine errors
└── creator-client-flow.spec.ts     # DELETE — migrate any unique assertions into proposal-negotiation.spec.ts
```

### When to Use API vs UI

| Scenario | Use |
|----------|-----|
| Setting up prerequisite data (create user, project, milestone) | API |
| Testing business logic / state transitions | API |
| Testing that the UI renders data correctly | UI |
| Testing user interactions (click, type, drag) | UI |
| Testing error feedback in the UI | UI |
| Testing public-facing pages (gallery, proposal, brief) | UI |

### Running Individual Phases

```bash
# Phase 1 specs only
npx playwright test specs/auth.spec.ts specs/full-happy-path.spec.ts specs/proposal-negotiation.spec.ts specs/milestone-escrow.spec.ts specs/shares.spec.ts specs/handover.spec.ts

# Phase 2 adds
npx playwright test specs/project-media.spec.ts specs/albums.spec.ts specs/amendments.spec.ts specs/project-lifecycle.spec.ts specs/transfer.spec.ts

# Phase 3 adds
npx playwright test specs/billing.spec.ts specs/milestone-dispute.spec.ts specs/notifications.spec.ts specs/public-pages.spec.ts specs/admin.spec.ts

# Phase 4 adds
npx playwright test specs/edge-cases.spec.ts specs/templates.spec.ts specs/notification-settings.spec.ts specs/storage.spec.ts
```

---

## CI Integration

The existing OCI DevOps pipeline (`zeyaroo-e2e-tests`) triggers on push to
`zeyaroo-e2e/main`. No CI changes needed — just push the new spec files.

**Expected run times (estimated):**

| Phase | Tests | Est. Runtime |
|-------|-------|-------------|
| Phase 1 | ~31 | ~4 min |
| Phase 2 | ~57 | ~8 min |
| Phase 3 | ~78 | ~12 min |
| Phase 4 | ~90+ | ~14 min |

Majority of time is backend event processing (5s intervals) and UI rendering.
API-only tests run in <100ms each.

---

## File Creation Order

Build infrastructure bottom-up, then spec files by phase:

1. `helpers/types.ts` — all result/input interfaces
2. `helpers/api.ts` — expand with all Phase 1+2 helpers
3. `fixtures/index.ts` — add adminAuth, projectWithMilestones, fundedMilestone
4. `helpers/auth.ts` — no changes needed
5. Phase 1 spec files (6 files)
6. Phase 2 spec files (5 files)
7. Phase 3 spec files (5 files)
8. Phase 4 spec files (4 files)
