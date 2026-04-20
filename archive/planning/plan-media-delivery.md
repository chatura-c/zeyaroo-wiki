# Media Delivery & Project Lifecycle — Implementation Plan

## Background

Zeyaroo is a creator-client media collaboration SaaS. The core value prop is milestone-based
escrow: creators stage watermarked previews → clients pay → watermarks lift → clients download.
This plan closes all the gaps in that flow and extends it through project completion and handover.

The audit (April 2026) found that the watermark enforcement is the most critical gap: clients
currently receive original signed URLs before paying, because `GetDeliveryPreview` passes
`useWebPreview=false`. Everything in this plan assumes that is fixed first (Phase 1).

---

## Phase 1 — Fix Watermark Enforcement ✅ COMPLETED

**Goal:** Clients see the watermarked `web_preview` variant when media is staged but unpaid.
After releasing milestone payment, they see and can download the original.

**Tasks:**

1. **Backend — delivery preview serves `web_preview`**
   - `phto-api/internal/handlers/delivery.go` — `GetDeliveryPreview` handler
   - Change `h.buildMediaResponse(&item.Media, false)` → `true` for unpaid items
   - `buildMediaResponse(media, useWebPreview bool)` already exists and returns the correct variant URL when `true`
   - Guard: only use `web_preview` when `item.DeliveryStatus != DeliveryStatusFull`

2. **Backend — share gallery serves `web_preview`**
   - `phto-api/internal/handlers/share.go` — public share media response
   - Share should serve `web_preview` variant for all media by default (share = preview context)
   - Add a `share.allow_originals bool` field (default false) for future "HQ unlock" feature
   - For now, always return `web_preview` URL to the unauthenticated public endpoint

3. **Backend — wire `removeMilestoneWatermarks`**
   - `phto-api/internal/services/escrow_queries.go` — function already written, zero call sites
   - Call it inside each milestone release path (escrow.go, escrow_payment_request.go,
     escrow_manual_funding.go, escrow_auto_release.go) after `autoCompleteProject`
   - Sets `is_watermarked = false` on all MediaObjects in the project

4. **Frontend — lightbox loads original on click (creator & delivered client)**
   - Currently the lightbox uses the `medium` variant everywhere
   - Add click-to-load-original: show `medium` initially, swap to original signed URL on click
   - For creators: always allowed
   - For clients: only if `is_delivered = true` on that item
   - Use the existing `/media/{id}/variant/{type}/file` signed URL endpoint

**Expected outcome:** Clients without payment see blurred/watermarked previews. Payment release
lifts watermarks platform-wide for that project. Clicking an image in the lightbox loads HQ.

---

## Phase 2 — Client Notifications & Send-to-Client Flow (~1 session)

**Goal:** Client is notified when delivery is staged and when their files are ready to download.
Creator has an explicit "send to client" action rather than delivery being invisible.

**Tasks:**

1. **Backend — notification on share creation**
   - `phto-api/internal/services/share.go` — `CreateShare`
   - After share is created, emit a notification to the project's client user (if linked)
   - Use existing notification pipeline (see `notification_system.md`)
   - Event type: `share.created` — email + in-app channel

2. **Backend — notification when delivery is staged**
   - `phto-api/internal/services/delivery.go` — when creator stages media to milestone delivery album
   - Emit `delivery.staged` event to client: "Your preview is ready — review and release payment to download"
   - In-app + email

3. **Backend — notification when final milestone released (files ready)**
   - Inside the milestone release path, after `removeMilestoneWatermarks`
   - Emit `delivery.complete` event to client: "Your files are ready to download"
   - Include a deep link to `ClientDeliveryView` for the project
   - In-app + email

4. **Frontend — "Send Preview to Client" button**
   - Currently share creation is implicit/buried
   - Add explicit CTA in the project delivery view: "Send Preview Link to Client"
   - Opens a dialog: set password, optional expiry, optional message
   - On submit: creates share + copies link to clipboard + shows "Client notified" toast
   - Lives in `phto-ui/src/pages/ProjectDetail.tsx` or a new `DeliveryPanel` component

**Expected outcome:** Creator clicks one button to send preview. Client gets an email + in-app
notification. When they pay and files are ready, they get another notification.

---

## Phase 3 — Client Bulk Download & Delivery Package (~1 session)

**Goal:** Client can download all their delivered files in one ZIP, not file-by-file.

**Tasks:**

1. **Backend — client ZIP download endpoint**
   - Add `GET /api/v1/projects/{id}/client-download-all`
   - Only accessible to the client user linked to the project
   - Only includes media where `DeliveryAccessFull` (i.e. milestone released)
   - Streams a ZIP using the same `StreamProjectZip` logic used for creator download
   - Returns 402 if any milestone is unfunded, 403 if not the project client

2. **Backend — delivery summary endpoint**
   - `GET /api/v1/projects/{id}/delivery-summary`
   - Returns: total file count, total size, download-ready count, pending-payment count
   - Used by the frontend to show "X of Y files ready to download"

3. **Frontend — bulk download UI for clients**
   - `phto-ui/src/pages/ClientDeliveryView.tsx`
   - Add "Download All Delivered Files" button (ZIP)
   - Show delivery summary stats: "42 of 50 files unlocked"
   - Disable/explain the button if no files are unlocked yet
   - Per-file download button remains for individual selections

4. **Frontend — "show a few in HQ" in share gallery**
   - Share gallery currently shows all previews at `web_preview` quality
   - Add `preview_hq_count: number` to the `Share` model (backend + frontend)
   - When set, the first N images in the share serve from `medium` (1920px) instead of `web_preview`
   - Communicates quality to potential clients without giving away originals
   - May be allow the creator to choose which is the default quality of the share.

**Expected outcome:** After final milestone, client sees "Download All (ZIP)" button and gets
everything in one action. Share gallery can tease quality with a configurable HQ sample.

---

## Phase 4 — Project Lifecycle & Expiry (~1 session)

**Goal:** Projects have a real retention policy. Creators know when projects expire.
Background jobs actually enforce expiry.

**Tasks:**

1. **Backend — wire `PurgeExpiredTrash` into cleanup loop**
   - `phto-api/internal/services/project_lifecycle.go` — function exists
   - `phto-api/internal/services/cleanup.go` — `RunCleanupLoop` does not call it
   - Add `PurgeExpiredTrash` to the loop (run daily or every N hours)
   - Purge deletes project + media records + physical files from storage

2. **Backend — enforce `ExpiresAt` on completed projects**
   - Currently `ExpiresAt` is set but never checked
   - Add a `MarkExpiredProjects` job: projects where `status = completed AND expires_at < now`
     get soft-deleted (moved to trash, 30-day grace before purge)
   - Default expiry for completed projects: 90 days after final milestone release
   - Set this in `autoCompleteProject`
   - Exception: `is_protected = true` projects skip this

3. **Backend — API to get project expiry info**
   - Include `expires_at`, `is_protected`, `days_until_expiry` in the project response
   - Used by the frontend vault/expiry warning UI

4. **Frontend — expiry warnings in project list and detail**
   - `phto-ui/src/pages/Vault.tsx` — currently uses a heuristic based on `event_date`
   - Replace with real `expires_at` from the API
   - Show "Expires in 14 days" badge on projects expiring within 30 days
   - Link to "Protect forever" ($25) CTA when expiry is near
   - In project detail header: expiry chip with upgrade prompt

5. **Frontend — client expiry awareness**
   - Client-side view should show "Project accessible until [date]"
   - If project is near expiry, show "Download your files before [date]"

**Expected outcome:** Projects auto-expire 90 days after completion (unless protected).
Creators and clients see real expiry dates and get warned in time to act.

---

## Phase 5 — Project Handover & Storage Freeing (~1 session)

**Goal:** After final delivery, creator can hand over the project to the client's account
and free up their own storage. Client can optionally pay to keep the project alive on
their own storage quota.

**Tasks:**

1. **Backend — flesh out project transfer for post-delivery handover**
   - `phto-api/internal/services/project_transfer.go` — `InitiateTransfer`, `AcceptTransfer` exist
   - Add a `HandoverToClient` service method:
     - Only callable when `IsReadyForHandover` = true (all milestones complete)
     - Changes `project.owner_id` to the client user
     - Recalculates storage accounting: subtract from creator, add to client
     - Sets a `handed_over_at` timestamp on the project
     - Creator's storage is freed
   - Client must have a storage subscription or agree to dynamic billing to accept

2. **Backend — storage accounting on transfer**
   - `phto-api/internal/services/storage.go`
   - `TransferStorageOwnership(projectID, fromUserID, toUserID)` — moves byte totals
   - Called inside `HandoverToClient`
   - Updates both users' `StorageUsed` fields

3. **Backend — handover invitation / acceptance flow**
   - Creator initiates: `POST /projects/{id}/handover` → creates a `ProjectHandover` record
   - Client gets notification: "Photographer has offered to hand over project files to you"
   - Client accepts: `POST /projects/{id}/handover/accept` → triggers `HandoverToClient`
   - Client declines: creator retains ownership, project expires normally

4. **Frontend — handover CTA for creators**
   - In project detail (when all milestones complete): "Hand Over to Client" button
   - Explains: "Transfer ownership to your client. This frees your storage. Client will manage the project."
   - Confirmation dialog with storage impact: "This will free X GB from your account"

5. **Frontend — handover acceptance for clients**
   - Notification in client dashboard: "Your photographer has offered to transfer the project"
   - Accept/decline flow with explanation of what it means (client's storage quota used)
   - After accept: project appears in client's own project list under "My Files"

**Expected outcome:** Creator can cleanly hand off a completed project, free their storage,
and the client takes ownership. Both parties know exactly what's happening.

---

## Implementation Order

```
Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5
```

Each phase is independent enough to ship in isolation once Phase 1 is done.
Phase 1 is a prerequisite for everything — don't ship previews without fixing watermark enforcement.

## Key Files Quick Reference

| File | Relevant to |
|------|-------------|
| `phto-api/internal/handlers/delivery.go` | Phase 1 — fix watermark serving |
| `phto-api/internal/handlers/share.go` | Phase 1, 3 — share serves web_preview |
| `phto-api/internal/services/escrow_queries.go` | Phase 1 — wire removeMilestoneWatermarks |
| `phto-api/internal/services/share.go` | Phase 2 — notification on share |
| `phto-api/internal/services/delivery.go` | Phase 2, 3 — staged notification, bulk download |
| `phto-api/internal/services/cleanup.go` | Phase 4 — wire PurgeExpiredTrash |
| `phto-api/internal/services/project_lifecycle.go` | Phase 4 — expiry enforcement |
| `phto-api/internal/services/project_transfer.go` | Phase 5 — handover |
| `phto-api/internal/services/storage.go` | Phase 5 — storage accounting |
| `phto-ui/src/pages/ClientDeliveryView.tsx` | Phase 3 — bulk download UI |
| `phto-ui/src/pages/ProjectDetail.tsx` | Phase 2, 5 — send preview CTA, handover CTA |
| `phto-ui/src/pages/Vault.tsx` | Phase 4 — real expiry dates |
