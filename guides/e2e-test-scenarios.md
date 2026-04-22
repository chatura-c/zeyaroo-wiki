# Zeyaroo E2E Test Scenarios — Complete Coverage Plan

Organized by **user journey**, each scenario traces from beginning to end across
both API and UI layers. Marked with priority (P0 = critical path, P1 = important,
P2 = nice-to-have) and current coverage status.

---

## Existing Coverage (18 tests)

| Spec | Scenarios |
|------|-----------|
| `auth.spec.ts` | JWT register+login, unauthenticated redirect, injected creator/client auth |
| `creator-client-flow.spec.ts` | Profile update visible in /creators; inquiry→proposal→accept via API; creator sees inquiries |
| `project-media.spec.ts` | Create project, upload photo, create album via UI |
| `milestone-escrow.spec.ts` | API-driven lifecycle with `deliverable_type:'none'`; client funds via UI button |
| `billing.spec.ts` | Subscribe, idempotency, admin line-item+discount, send finalizes, pending invoice, banner, suspension |

---

## 1. Authentication & Registration

### 1.1 JWT Registration — New User (P0) ✅ Partially covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | POST `/auth/register` with role=creator | 200, token returned, user.role = "creator" |
| 2 | POST `/auth/login` with same credentials | 200, same user returned |
| 3 | POST `/auth/register` with same email | 409 conflict |

**UI variant**: Register page → pick creator → fill form → submit → redirects to /dashboard

### 1.2 JWT Registration — Role Assignment (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Register with role="creator" | user.role = "creator" |
| 2 | Register with role="admin" | user.role = "admin" |
| 3 | Register with role="client" (or any other) | user.role = "client" |
| 4 | Register with no role field | user.role = "client" (default) |

### 1.3 Token Refresh (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Register user, get token | token valid |
| 2 | POST `/auth/refresh` with Bearer token | New token returned, user data matches |
| 3 | Use new token for `/users/me` | 200 OK |

### 1.4 Unauthenticated Access (P0) ✅ Covered (redirect only)

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Navigate to `/dashboard` without auth | Redirects to `/login` |
| 2 | Navigate to `/projects` without auth | Redirects to `/login` |
| 3 | GET `/users/me` without Bearer token | 401 |

### 1.5 Wrong Credentials (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | POST `/auth/login` with wrong password | 401 |
| 2 | POST `/auth/login` with non-existent email | 401 |

### 1.6 Guest OTP Auth (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | POST `/auth/guest/request-otp` with email | 200, OTP sent |
| 2 | POST `/auth/guest/verify-otp` with correct code | 200, JWT with role="guest" |
| 3 | POST `/auth/guest/verify-otp` with wrong code | 401 |
| 4 | POST `/auth/guest/request-otp` 6 times in 2 seconds | 429 rate-limited |

### 1.7 Post-Signup Hooks (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Register creator | `GET /users/me` returns subscription_tier="starter" |
| 2 | Register creator | Creator's "My Studio" project auto-created (visible in GET `/projects`) |
| 3 | Register creator | "Portfolio" album exists in studio project |
| 4 | Register creator | CreatorProfile auto-created |
| 5 | Register client | No studio project, no CreatorProfile |

---

## 2. Creator Profile

### 2.1 Profile Setup via /profile Page (P0) ❌ Not covered (API-only)

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator navigates to `/profile` | Page loads with form |
| 2 | Fill display_name, bio, upload avatar | PUT `/creators/profile` called |
| 3 | Add social URLs, select service categories | Profile saved |
| 4 | Navigate to `/creators/:id` | Public profile shows name, bio, avatar, categories |

### 2.2 Slug Claim (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | GET `/creators/profile/suggest-slug` | Returns suggestion based on display_name |
| 2 | POST `/creators/profile/claim-slug` with "my-studio" | 200, slug set |
| 3 | GET `/creators/slug/my-studio` | Returns profile |
| 4 | Another creator claims same slug | 409 conflict |
| 5 | Creator on starter tier tries to claim slug | 403 (paid plan required) |

### 2.3 Cover Photo — Paid Feature Gate (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Starter creator tries to upload cover photo | 403 |
| 2 | Admin upgrades creator to pro | PUT `/admin/users/:id/tier` |
| 3 | Creator uploads cover photo | 200 |

### 2.4 Public Creator Directory (P1) ✅ Partially (list only)

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | GET `/creators` without auth | Returns list with display_name, slug, avatar, categories |
| 2 | Navigate to `/creators` page | Grid of creator tiles renders |
| 3 | Click creator tile | Navigates to `/creators/:id` with full profile |
| 4 | Search by name | Results filter |
| 5 | Filter by category | Results filter |

### 2.5 Portfolio Media (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Upload media to studio project | Media in Portfolio album |
| 2 | GET `/creators/:id/portfolio` | Returns portfolio media |
| 3 | Public profile page shows portfolio grid | Grid renders with lightbox |

---

## 3. Inquiry → Proposal → Negotiation (Core Business Flow)

### 3.1 Client Sends Inquiry via Creator Profile (P0) ❌ Not covered (API-only)

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Client navigates to `/creators/:id` | Creator profile loads |
| 2 | Click "Inquire" → fill 3-step modal | POST `/inquiries` called |
| 3 | Inquiry created | Response includes inquiry + linked proposal (status=inquiry) |
| 4 | Client navigates to `/inquiries` | Own inquiry visible in list view |

### 3.2 Creator Sees Inquiries in Kanban (P0) ✅ Partially (list only)

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator navigates to `/inquiries` | Kanban board with 4 columns renders |
| 2 | Drag inquiry from "New Inbox" to "Qualifying" | PATCH pipeline_status called, card moves |
| 3 | Drag to "Proposal Sent" | Status updated |
| 4 | Switch to List view toggle | Shows flat list instead |

### 3.3 Creator Sends Proposal via Builder (P0) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator clicks "Send Proposal" on inquiry | Navigates to `/proposals/:id/builder` |
| 2 | Builder loads with 3-column layout | Logistics, Chapters, Preview visible |
| 3 | Fill deliverables, usage_rights, revision_rounds, timeline | Form fields populated |
| 4 | Add 3 chapters (milestones) with title, amount, deadline | Chapters appear in sortable list |
| 5 | Drag-reorder chapters | Order updates |
| 6 | Add questionnaire tab | Questions appear in preview |
| 7 | Set cover image from portfolio | Cover set |
| 8 | Click "Send" | POST `/proposals/:id/versions` called, proposal → "negotiating" |

### 3.4 Client Views Public Proposal via Token (P0) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Navigate to `/p/:token` | Proposal renders with milestones, creator info |
| 2 | Answer questionnaire questions | Responses saved |
| 3 | Agree to service agreement checkbox | Accept button enables |
| 4 | Request OTP → verify OTP | Guest JWT obtained |
| 5 | Accept proposal | Proposal → "finalized" |
| 6 | Creator sees proposal as finalized on `/proposals/:id` | Status updated |

### 3.5 Client Counters Proposal (Negotiation Loop) (P0) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Client on `/p/:token` clicks "Refine the Vision" | CompareView diff editor opens |
| 2 | Client edits milestone amounts, adds notes | Changes reflected in comparison |
| 3 | Client submits counter | Proposal → "pending" (client turn) |
| 4 | Creator sees pending proposal on `/proposals/:id` | CompareView shows client's version vs theirs |
| 5 | Creator accepts client version | POST `/proposals/:id/accept`, proposal → "finalized" |
| 6 | OR creator counters again | Proposal → "negotiating", back to client |

### 3.6 Proposal Cancellation (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator cancels proposal in "inquiry" state | 200, proposal → "canceled" |
| 2 | Client cancels proposal in "negotiating" state | 200, proposal → "canceled" |
| 3 | Cancel already finalized proposal | 400 invalid state |

### 3.7 Direct Proposal Creation (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator calls POST `/proposals/direct` with client_email | Inquiry + proposal created in one step |
| 2 | Proposal appears in creator's list | Status = "inquiry" |

### 3.8 Create Project from Finalized Proposal (P0) ✅ Partially (API-only)

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator clicks "Create Project" on finalized proposal | POST `/proposals/:id/create-project` |
| 2 | Project created with milestones | GET `/projects/:id` returns project with proposal_id |
| 3 | Navigate to `/projects/:id` | Project detail page renders with milestones |

### 3.9 Proposal Version History (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Create proposal with 3 versions | Each version immutable |
| 2 | GET `/proposals/:id/versions` | 3 versions returned, each with author_role |
| 3 | UI shows expand/collapse version timeline | All versions visible |

### 3.10 Guest Inquiry (No Auth) (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | POST `/creators/:id/inquire` without JWT with guest_email | Inquiry created with guest_email |
| 2 | Inquiry has no client_id | client_id is null |
| 3 | Creator sees inquiry in list | Shows with client_name + guest_email |

---

## 4. Project Lifecycle

### 4.1 Create Project (P0) ✅ Partially (API-only)

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator clicks "New Project" on `/projects` | Modal opens |
| 2 | Fill title, description, location | POST `/projects` called |
| 3 | Navigate to `/projects/:id` | Project detail renders with title |

### 4.2 Update Project (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | PUT `/projects/:id` with new title, description | 200, updated |
| 2 | UI shows updated title | Title reflects change |

### 4.3 Archive & Restore (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | POST `/projects/:id/archive` on active project | 200, status="archived" |
| 2 | Project not in active list | Archived project hidden from main list |
| 3 | POST `/projects/:id/restore` | 200, status="active" |
| 4 | Project returns to active list | Visible again |
| 5 | Archive project with funded milestone | 400 (cannot archive) |

### 4.4 Trash & Restore (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | POST `/projects/:id/trash` | 200, soft-deleted |
| 2 | Project not in active list | Hidden |
| 3 | GET `/projects/trash` | Trashed project listed |
| 4 | POST `/projects/:id/restore-trash` | 200, restored |
| 5 | Trash system project | 403 forbidden |
| 6 | Trash project with funded milestone | 403 forbidden |

### 4.5 Project Storage (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | GET `/projects/:id/storage` | Returns bytes, formatted_size |
| 2 | Upload media, re-check storage | Bytes increase |

### 4.6 Forever Gallery Protection (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | POST `/projects/:id/forever` | Returns Dodo checkout_url |
| 2 | After payment, GET `/projects/:id` | is_protected = true, expires_at = null |

### 4.7 Collaborator Management (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | POST `/projects/:id/collaborators` with user_id | 204 |
| 2 | Collaborator can GET `/projects/:id` | 200 |
| 3 | DELETE `/projects/:id/collaborators/:userId` | 204 |
| 4 | Removed collaborator gets 403 on GET `/projects/:id` | |

---

## 5. Project Transfer & Handover

### 5.1 Transfer Request Flow (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Owner POST `/projects/:id/transfer-request` | 200, transfer created |
| 2 | Recipient GET `/transfer-requests/pending` | Transfer listed |
| 3 | Recipient POST `/transfer-requests/:id/accept` | Ownership transferred |
| 4 | Original owner can no longer edit project | 403 |
| 5 | New owner GET `/projects/:id` | owner_id = recipient |

### 5.2 Transfer Request Decline (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Recipient POST `/transfer-requests/:id/decline` | 204, ownership unchanged |

### 5.3 Handover Flow (P0) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Complete all milestones in project | All milestones status="completed" |
| 2 | Creator clicks "Hand Over to Client" | POST `/projects/:id/handover`, handover created |
| 3 | Client GET `/handovers/pending` | Handover listed |
| 4 | Client POST `/handovers/accept` | Ownership transferred to client |
| 5 | Project appears in client's vault | `/vault` shows handed-over project |
| 6 | Creator's storage freed | Storage usage decreases |

### 5.4 Handover Decline (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Client POST `/handovers/decline` | 204, ownership unchanged |

### 5.5 Project Claim (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Guest delivery project with can_be_claimed=true | |
| 2 | POST `/projects/:id/claim` with email+password | Account created + ownership transferred |
| 3 | New user can access project | GET `/projects/:id` returns 200 |

---

## 6. Milestones & Escrow (Full Lifecycle)

### 6.1 Full Escrow Lifecycle with Deliverables (P0) ❌ Not covered

The most important untested flow. Currently only `deliverable_type: 'none'` is tested.

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Create proposal with milestones having `deliverable_type: "media"` | Proposal version created |
| 2 | Accept proposal → create project | Project with milestones |
| 3 | Client POST `/milestones/:id/fund` | Milestone → "funded", escrow held |
| 4 | Creator POST `/milestones/:id/start` | Milestone → "in_progress" |
| 5 | Creator uploads media to delivery album | Media in milestone album |
| 6 | Creator POST `/milestones/:id/submit-delivery` | Album locked, milestone → "submitted" |
| 7 | Client POST `/milestones/:id/release` | Escrow released, milestone → "completed" |
| 8 | GET `/projects/:id/milestones` summary | All amounts calculated correctly |

### 6.2 Milestone — Request Changes (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Milestone in "submitted" state | |
| 2 | Client POST `/milestones/:id/request-changes` with reason | Milestone → "pending", feedback created |
| 3 | GET `/milestones/:id/feedback` | Change request visible |
| 4 | Creator can re-upload and re-submit | Milestone → "submitted" again |

### 6.3 Milestone — Unlock (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Milestone in "submitted" state | |
| 2 | Creator POST `/milestones/:id/unlock` | Milestone → "pending", album unlocked |
| 3 | Creator can add/remove media | |

### 6.4 Milestone Dispute (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Milestone in "funded" or later state | |
| 2 | Either party POST `/milestones/:id/dispute` with reason | Milestone → "disputed", dispute created |
| 3 | GET `/admin/disputes` | Dispute listed |
| 4 | Admin POST `/milestones/:id/resolve` action="release" | Milestone → "completed", escrow released |
| 5 | OR admin resolves with action="refund" | Milestone → "pending", escrow refunded |

### 6.5 Sequential Milestone Funding (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Project with 3 milestones | All pending |
| 2 | Fund milestone 1 | Milestone 1 → funded |
| 3 | Try to fund milestone 3 before milestone 2 | 400 (previous not funded) |
| 4 | Fund milestone 2, then 3 | Both funded |

### 6.6 Manual/Offline Funding Flow (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator POST `/milestones/:id/request-funding` with offline options | Payment page created |
| 2 | GET `/public/pay/:token` | Payment page data returned |
| 3 | Client POST `/public/pay/:token/select-method` with method="offline" | Offline selected |
| 4 | Client uploads receipt via POST `/public/pay/:token/upload-receipt` | Receipt uploaded |
| 5 | Creator POST `/milestones/:id/approve-manual` | Milestone → funded |
| 6 | OR Creator POST `/milestones/:id/mark-paid` | Milestone → funded |

### 6.7 Milestone — Mark Manually Funded (Cash) (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator POST `/milestones/:id/mark-manually-funded` | Milestone → "funded" |
| 2 | Service fee invoice auto-generated | GET `/billing/invoices` includes offline_service_fee line item |

### 6.8 Milestone Update (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | PATCH `/milestones/:id` with new title, amount | 200, updated |
| 2 | Try to update funded milestone | 400 |
| 3 | Try to update as client (not creator) | 403 |

---

## 7. Media Management

### 7.1 Upload Media via UI (P0) ✅ Partially (basic upload only)

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator on project media tab | Upload area visible |
| 2 | Drop/select image file | Upload begins |
| 3 | Media appears in grid | Thumbnail visible |
| 4 | First image auto-sets as project cover | Cover image set |

### 7.2 Direct Upload to B2 (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | POST `/projects/:id/media/initiate` | Returns upload_url, media_id |
| 2 | PUT to upload_url with file | 200 (B2 accepts) |
| 3 | POST `/media/:mediaId/complete` | Media upload_status = "completed" |

### 7.3 Media Culling (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Navigate to `/projects/:id/cull` | Full-screen culling view |
| 2 | Click "Pick" on image | PATCH cull_status → "accept" |
| 3 | Press X to reject | PATCH cull_status → "reject" |
| 4 | Filter by "Picks" | Only accepted images shown |
| 5 | Navigate back to project | Rejected media in rejected view |
| 6 | DELETE `/projects/:id/media/rejected` | Rejected media purged |

### 7.4 Media Favorites (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | PATCH `/media/:id/favorite` with is_favorite=true | Media marked |
| 2 | Filter by favorites | Only favorites shown |

### 7.5 Delete Media (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | DELETE `/media/:id` as project owner | 204 |
| 2 | Media no longer in list | |
| 3 | Try delete as non-owner | 403 |

### 7.6 Media Download (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | GET `/media/:id/download` as creator | Full original file |
| 2 | GET `/media/:id/download` as client (undelivered) | 403 (not staged) |
| 3 | GET `/media/:id/download` as client (milestone submitted) | Preview only |
| 4 | GET `/media/:id/download` as client (milestone released) | Full original |
| 5 | GET `/media/:id/variant/thumbnail` | Thumbnail variant |

### 7.7 Bulk Download (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | GET `/projects/:id/media/download-all` as owner | ZIP stream |
| 2 | GET `/projects/:id/client-download-all` as client | ZIP stream |

### 7.8 Delivery Preview & Summary (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Client GET `/projects/:id/delivery-preview` | Staged media with milestone info |
| 2 | Client GET `/projects/:id/delivery-summary` | Delivery stats |

### 7.9 Add/Remove Media from Albums (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | POST `/media/:id/albums` with album_ids | 204 |
| 2 | Media appears in album | GET `/albums/:id/media` includes it |
| 3 | DELETE `/media/:id/albums/:albumId` | 204, removed from album |

---

## 8. Albums

### 8.1 Create Album via UI (P0) ✅ Covered

### 8.2 Update Album (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | PUT `/albums/:id` with new name | 200, updated |
| 2 | Try to rename system album (Portfolio) | 403 |

### 8.3 Delete Album (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | DELETE `/albums/:id` as owner | 204 |
| 2 | Try to delete system album | 403 |
| 3 | Try to delete milestone delivery album | 403 |

### 8.4 List Media in Album (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | GET `/albums/:id/media` | Paginated media list |

---

## 9. Shares & Galleries

### 9.1 Create Share (P0) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator on Shares tab clicks "Create Share" | Dialog opens |
| 2 | Select albums, set password, configure preview_hq_count | POST `/projects/:id/shares` called |
| 3 | Share created with token, URL generated | Share URL displayed |
| 4 | GET `/projects/:id/shares` | New share listed |

### 9.2 Public Gallery — Password Protected (P0) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Navigate to `/gallery/:token` | Password gate shown |
| 2 | Enter wrong password | Error message |
| 3 | Enter correct password | share_access_token stored, media grid loads |
| 4 | Browse albums, view media | Albums filter, media paginated |
| 5 | First N images at medium quality, rest at web_preview | |
| 6 | Open lightbox | Full image view |
| 7 | 6 wrong password attempts | Rate-limited (5/hour) |

### 9.3 Public Gallery — Selection Mode (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Enter selection mode | Checkboxes on images |
| 2 | Select images (up to limit) | Selection tray shows count |
| 3 | Submit selection | POST selection submitted |
| 4 | Creator sees submitted selections | |

### 9.4 Share Update & Delete (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | PUT `/shares/:id` with new password | 200 |
| 2 | DELETE `/shares/:id` | 204 |
| 3 | Try to update share as non-creator | 403 |

### 9.5 Share Expiration Refresh (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | POST `/shares/:id/refresh-expiration` | Expires_at extended by 30 days |

### 9.6 Share — Expired/Inactive (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Access expired share at `/gallery/:token` | 410 gone |
| 2 | Access inactive share | 404 |

---

## 10. Amendments

### 10.1 Full Amendment Lifecycle (P0) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator creates draft amendment | POST `/projects/:id/amendments`, status="draft" |
| 2 | Add draft milestones | POST `.../amendments/:aid/milestones` |
| 3 | Modify draft milestone amount | PATCH `.../milestones/:mid` |
| 4 | Delete a draft milestone | DELETE `.../milestones/:mid` |
| 5 | Publish amendment | POST `.../publish`, status="pending_approval" |
| 6 | Client applies amendment | POST `.../apply`, current version bumped |
| 7 | New milestones appear in project | GET `/projects/:id/milestones` includes new ones |

### 10.2 Amendment — Client Rejects (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Client POST `.../reject` with reason | 204, amendment → "rejected" |
| 2 | Project milestones unchanged | |

### 10.3 Amendment — Cannot Remove Funded Milestone (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Try to delete funded milestone from draft | 422 |
| 2 | Try to reduce funded milestone amount | 422 |

### 10.4 Public Amendment Review (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Navigate to `/p/:token/amendment/:atoken` | Side-by-side diff renders |
| 2 | New milestones show green badge | |
| 3 | Removed milestones show red badge | |
| 4 | Price changes show up/down badges | |
| 5 | Click "Approve" | Amendment applied |
| 6 | OR click "Decline" with reason | Amendment rejected |

### 10.5 Amendment Cancel (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator POST `.../cancel` on draft amendment | 204, amendment → "cancelled" |

---

## 11. Briefs & Questionnaires

### 11.1 Brief Request Lifecycle (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator POST `/projects/:id/brief` with questions | Brief created, token generated |
| 2 | Navigate to `/brief/:token` | Questions render (text + choice types) |
| 3 | Client fills answers and submits | POST `/public/brief/:token/respond`, status → "completed" |
| 4 | Creator resends completed brief | 409 |

### 11.2 Questionnaire in Proposal (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator saves questionnaire schema | PUT `/projects/:id/questionnaire` |
| 2 | Client responds via public proposal | POST `/public/proposals/:token/questionnaire/responses` |
| 3 | Creator views responses | GET `/projects/:id/questionnaire` |

---

## 12. Storage

### 12.1 Storage Usage Check (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | GET `/storage/usage` | Returns type, quota, used, price_per_gb |
| 2 | Upload media, re-check | Used bytes increase |

### 12.2 Storage Subscription & Mode Switch (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | POST `/storage/subscribe` with type="reserved" | Subscription created |
| 2 | POST `/storage/switch-mode` to "dynamic" | Mode switched, proration returned |
| 3 | POST `/storage/adjust-quota` for reserved | Quota adjusted |

### 12.3 Upload Blocked When Over Quota (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Fill storage to limit | |
| 2 | Try to upload media | 402 storage limit |

---

## 13. Billing & Payments

### 13.1 Subscribe to Plan (P0) ✅ Covered (subscription + idempotency)

### 13.2 Invoice Lifecycle (P1) ✅ Partially (admin line-item + discount + send)

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Subscribe to plan → draft invoice created | |
| 2 | Admin adds line item + discount | Totals correct |
| 3 | Admin sends invoice → finalized | GET pending invoice returns it |
| 4 | User POST `/billing/invoices/:id/pay` | Payment processed |
| 5 | GET `/billing/payments` | Payment listed |

### 13.3 Payout Request (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Creator POST `/billing/payout-request` | Payout request created |
| 2 | GET `/billing/payout-requests` | Request listed |
| 3 | Admin PATCH `/admin/payout-requests/:id` | Status updated |

### 13.4 Creator Earnings (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Release milestone escrow to creator | |
| 2 | GET `/users/me/earnings` | Returns earnings breakdown |
| 3 | Navigate to `/billing` | Earnings dashboard shows total |

### 13.5 Plan Upgrade (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | POST `/billing/upgrade` with higher plan_id | Plan upgraded |
| 2 | User subscription_tier updated | |

### 13.6 Billing Addresses (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | POST `/billing/addresses` | Address created |
| 2 | GET `/billing/addresses` | Address listed |
| 3 | DELETE `/billing/addresses/:id` | 204 |

### 13.7 Pending Invoice Banner (P0) ✅ Covered

### 13.8 Suspended Creator Cannot Create Project (P1) ✅ Partially (API-only)

---

## 14. Notifications

### 14.1 Notification CRUD (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Trigger notification (e.g., inquiry received) | |
| 2 | GET `/notifications/` | Notification listed with unread_count > 0 |
| 3 | GET `/notifications/unread-count` | Count matches |
| 4 | PATCH `/notifications/:id/read` | Marked as read |
| 5 | POST `/notifications/mark-all-read` | All read |
| 6 | DELETE `/notifications/:id` | Removed |

### 14.2 Notification Bell in UI (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Navigate to any page with new notification | Bell shows badge count |
| 2 | Click bell | Dropdown shows notifications |
| 3 | Click notification | Navigates to relevant page |

### 14.3 Notification Settings (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Navigate to `/settings/notifications` | Toggles visible |
| 2 | Toggle inquiry_email off | PATCH `/users/me/notification-settings` called |
| 3 | Re-check settings | Setting persists |

---

## 15. Templates

### 15.1 Milestone Template CRUD (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Navigate to `/templates/milestones` | Empty list |
| 2 | Create template with name + milestones JSON | POST `/templates` |
| 3 | Edit template | PUT `/templates/:id` |
| 4 | Delete template | DELETE `/templates/:id` |
| 5 | Apply template in proposal builder | Chapters populated from template |

### 15.2 Brief Template CRUD (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Navigate to `/templates/briefs` | |
| 2 | Create brief template | |
| 3 | Apply in brief request creation | |

---

## 16. Admin Dashboard

### 16.1 Admin Login & Stats (P1) ❌ Not covered (E2E)

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | POST `/admin/login` | Admin JWT returned |
| 2 | GET `/admin/stats` | Stats object with user counts, revenue, etc. |

### 16.2 Admin User Management (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | GET `/admin/users` | User list |
| 2 | GET `/admin/users/:id` | Full detail |
| 3 | PATCH `/admin/users/:id` with name change | Updated |
| 4 | PATCH `/admin/users/:id/tier` to "pro" | Tier upgraded |
| 5 | PUT `/admin/users/:id/package` | Custom package set |

### 16.3 Admin Invoice Management (P1) ✅ Partially

### 16.4 Admin Dispute Resolution (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Create milestone dispute | |
| 2 | GET `/admin/disputes` | Dispute listed |
| 3 | POST `/milestones/:id/resolve` action="release" | Dispute resolved |

### 16.5 Admin Escrow Overview (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | GET `/admin/escrow/transactions` | Escrow transaction list |

### 16.6 Admin Payment Method Config (P2) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | PUT `/admin/users/:id/payment-methods` | Methods set |
| 2 | PUT `/admin/users/:id/milestone-funding` | Escrow/offline config set |

---

## 17. Vault (Client)

### 17.1 Vault Shows Handed-Over Projects (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Complete handover flow | Project transferred to client |
| 2 | Client navigates to `/vault` | Handed-over project visible |
| 3 | Client can download all media | Full original access |
| 4 | "Save Forever" option for expiring project | Checkout URL provided |

---

## 18. Edge Cases & Error Scenarios

### 18.1 Unauthorized Access (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Client tries to access creator-only endpoint | 403 |
| 2 | User tries to access another user's project | 403 |
| 3 | Non-admin tries admin endpoints | 403 |

### 18.2 Validation Errors (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Register with missing email | 400 |
| 2 | Create project with empty title | 400 |
| 3 | Fund milestone with negative amount | 400 |
| 4 | Claim slug with invalid characters | 400 |

### 18.3 Proposal State Machine Violations (P1) ❌ Not covered

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Accept proposal in "inquiry" state | 400 |
| 2 | Create version on finalized proposal | 403 "Not your turn" |
| 3 | Cancel already canceled proposal | 400 |

### 18.4 Duplicate Actions (P2) ✅ Partially (billing idempotency only)

| Step | Action | Assertion |
|------|--------|-----------|
| 1 | Accept already accepted proposal | 400 |
| 2 | Fund already funded milestone | 400 |
| 3 | Create project from proposal that already has one | 400 |

---

## 19. Cross-Cutting / Integration Scenarios

### 19.1 Full Happy Path: Creator → Client → Delivery → Handover (P0) ❌ Not covered as single E2E flow

| Step | Action |
|------|--------|
| 1 | Creator registers, sets up profile |
| 2 | Client registers, finds creator in directory |
| 3 | Client sends inquiry |
| 4 | Creator creates proposal with milestones (deliverable_type: media) |
| 5 | Client accepts proposal via public page |
| 6 | Creator creates project from proposal |
| 7 | Client funds first milestone |
| 8 | Creator starts milestone, uploads media |
| 9 | Creator submits delivery |
| 10 | Client releases payment |
| 11 | Repeat for all milestones |
| 12 | Creator initiates handover |
| 13 | Client accepts handover |
| 14 | Project appears in client vault |
| 15 | Creator earnings updated |

### 19.2 Full Happy Path with Negotiation Loop (P0) ❌ Not covered

| Step | Action |
|------|--------|
| 1-4 | Same as 19.1 |
| 5 | Client counters proposal (edits milestones) |
| 6 | Creator reviews counter, re-counters |
| 7 | Client accepts revised proposal |
| 8-15 | Same as 19.1 |

### 19.3 Full Happy Path with Amendment (P1) ❌ Not covered

| Step | Action |
|------|--------|
| 1-8 | Same as 19.1 through first milestone funded |
| 9 | Creator drafts amendment (add new milestone, modify amount) |
| 10 | Creator publishes amendment |
| 11 | Client approves amendment |
| 12 | New milestones appear in project |
| 13 | Continue with funding remaining milestones |

### 19.4 Full Happy Path with Sharing (P1) ❌ Not covered

| Step | Action |
|------|--------|
| 1-9 | Same as 19.1 through media uploaded |
| 10 | Creator creates share with albums + password |
| 11 | Client opens gallery link, enters password |
| 12 | Client browses media, selects favorites |
| 13 | Client submits selection |
| 14 | Creator sees client's selection |

---

## Coverage Summary

| Domain | P0 Tests | P1 Tests | P2 Tests | Existing Coverage |
|--------|----------|----------|----------|-------------------|
| Auth & Registration | 2 | 4 | 0 | ~40% |
| Creator Profile | 1 | 3 | 2 | ~20% |
| Inquiry → Proposal | 4 | 3 | 2 | ~25% |
| Project Lifecycle | 1 | 3 | 3 | ~10% |
| Transfer & Handover | 1 | 1 | 2 | 0% |
| Milestones & Escrow | 1 | 4 | 2 | ~15% |
| Media Management | 1 | 4 | 3 | ~15% |
| Albums | 0 | 2 | 1 | ~20% |
| Shares & Galleries | 2 | 1 | 2 | 0% |
| Amendments | 1 | 2 | 1 | 0% |
| Briefs & Questionnaires | 0 | 1 | 1 | 0% |
| Storage | 0 | 2 | 2 | 0% |
| Billing & Payments | 2 | 2 | 2 | ~35% |
| Notifications | 0 | 1 | 2 | 0% |
| Templates | 0 | 0 | 2 | 0% |
| Admin | 0 | 3 | 2 | ~10% |
| Vault | 0 | 1 | 0 | 0% |
| Edge Cases | 0 | 3 | 1 | ~5% |
| **Total** | **16** | **40** | **28** | **~15%** |

### Priority Implementation Order

1. **19.1** — Full happy path end-to-end (single test covering the entire creator→client→delivery→handover flow)
2. **6.1** — Full escrow lifecycle with deliverables
3. **9.1 + 9.2** — Share creation + public gallery
4. **3.4** — Client views public proposal via token
5. **5.3** — Handover flow
6. **10.1** — Full amendment lifecycle
7. **18.1 + 18.3** — Authorization and state machine edge cases
8. Everything else by P0 → P1 → P2

### API Helpers Needed (not in existing helpers/api.ts)

To implement these scenarios, the following API helpers should be added:

```typescript
// Auth
refreshToken(token)
requestGuestOTP(email)
verifyGuestOTP(email, code)

// Users
getCurrentUser(token)
updateUser(token, data)
getNotificationSettings(token)
updateNotificationSettings(token, data)
getCreatorEarnings(token)

// Creator Profile
claimSlug(token, slug)
suggestSlug(token)
getCreatorBySlug(slug)
getCreatorPortfolio(creatorId)

// Projects
updateProject(token, projectId, data)
deleteProject(token, projectId)
archiveProject(token, projectId)
restoreProject(token, projectId)
trashProject(token, projectId)
restoreTrashProject(token, projectId)
getTrashedProjects(token)
getProjectStorage(token, projectId)
setProjectCover(token, projectId, mediaId)
foreverGallery(token, projectId)
addCollaborator(token, projectId, userId)
removeCollaborator(token, projectId, userId)

// Transfer & Handover
initiateTransferRequest(token, projectId, toUserId)
getPendingTransfers(token)
acceptTransfer(token, transferId)
declineTransfer(token, transferId)
initiateHandover(token, projectId)
getPendingHandovers(token)
acceptHandover(token, handoverId)
declineHandover(token, handoverId)
claimProject(projectId, email, password)

// Proposals
listCreatorProposals(token, status)
listClientProposals(token, status)
createDirectProposal(token, data)
cancelProposal(token, proposalId)
getProposalVersions(token, proposalId)
setProposalCover(token, proposalId, mediaId)

// Milestones
updateMilestone(token, milestoneId, data)
startMilestone(token, milestoneId)
completeMilestone(token, milestoneId)
unlockMilestone(token, milestoneId)
releaseMilestone(token, milestoneId)
disputeMilestone(token, milestoneId, reason)
requestChanges(token, milestoneId, reason)
submitDelivery(token, milestoneId)
getMilestoneDeliveryAlbum(token, milestoneId)
getMilestoneFeedback(token, milestoneId)
requestFunding(token, milestoneId, options)
requestManualFunding(token, milestoneId, receiptFile)
approveManualFunding(token, milestoneId)
markMilestonePaid(token, milestoneId)
markManuallyFunded(token, milestoneId, note?)

// Media
deleteMedia(token, mediaId)
getMediaDownload(token, mediaId)
getMediaVariant(token, mediaId, type)
addMediaToAlbums(token, mediaId, albumIds)
removeMediaFromAlbum(token, mediaId, albumId)
updateCullStatus(token, mediaId, status)
updateFavorite(token, mediaId, isFavorite)
completeUpload(token, mediaId)
purgeRejectedMedia(token, projectId)
getDeliveryPreview(token, projectId)
getDeliverySummary(token, projectId)
getClientDownloadAll(token, projectId)
getDownloadAll(token, projectId)

// Albums
updateAlbum(token, albumId, data)
deleteAlbum(token, albumId)
getAlbumMedia(token, albumId)

// Shares
createShare(token, projectId, data)
listShares(token, projectId)
updateShare(token, shareId, data)
deleteShare(token, shareId)
refreshShareExpiration(token, shareId)
getPublicShare(token)
verifySharePassword(token, password)
getShareAlbums(token)
getShareMedia(token, params)

// Amendments
createAmendment(token, projectId)
listAmendments(token, projectId)
publishAmendment(token, projectId, amendmentId)
cancelAmendment(token, projectId, amendmentId)
applyAmendment(token, projectId, amendmentId)
rejectAmendment(token, projectId, amendmentId, reason?)
addDraftMilestone(token, projectId, amendmentId, data)
updateDraftMilestone(token, projectId, amendmentId, milestoneId, data)
deleteDraftMilestone(token, projectId, amendmentId, milestoneId)

// Briefs
createBriefRequest(token, projectId, data)
getPublicBrief(token)
submitBriefAnswers(token, answers)

// Storage
getStorageUsage(token)
subscribeStorage(token, data)
switchStorageMode(token, data)
adjustStorageQuota(token, quota)

// Billing
getInvoice(token, invoiceId)
payInvoice(token, invoiceId, gatewayName)
listPayments(token)
getUpcomingCharges(token)
requestPayout(token, data)
listPayoutRequests(token)

// Notifications
listNotifications(token)
getUnreadCount(token)
markNotificationRead(token, id)
markAllNotificationsRead(token)
deleteNotification(token, id)

// Templates
listTemplates(token)
createTemplate(token, data)
updateTemplate(token, id, data)
deleteTemplate(token, id)

// Admin
adminLogin(email, password)
getAdminStats(token)
listAdminUsers(token, params)
getAdminUser(token, userId)
updateAdminUser(token, userId, data)
updateUserTier(token, userId, tier)
updateUserPackage(token, userId, data)
updateUserPaymentMethods(token, userId, methods)
updateUserMilestoneFunding(token, userId, data)
listAdminInvoices(token, params)
getAdminInvoice(token, invoiceId)
updateAdminInvoiceStatus(token, invoiceId, status)
adminApproveInvoice(token, invoiceId)
adminRejectInvoice(token, invoiceId, reason?)
adminAddLineItem(token, invoiceId, data)
adminApplyDiscount(token, invoiceId, data)
adminSendInvoice(token, invoiceId)
listAdminPayoutRequests(token, params)
updateAdminPayoutRequest(token, id, data)
listAdminEscrowTransactions(token, params)
listAdminDisputes(token, params)
listAdminProjects(token, params)
getAdminProject(token, projectId)
listAdminJobs(token, params)
listAdminEvents(token, params)
resolveDispute(token, milestoneId, action, notes)
```
