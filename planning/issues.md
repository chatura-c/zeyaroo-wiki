# Zeyaroo Issues

---

## Issue 1 — Gate cover image upload behind paid plan ✅

**Labels:** `feature`, `billing`

**Description:**
Cover image upload on the creator profile should be restricted to paying users only. Free-tier users should see the upload control as disabled or hidden, with a clear upsell prompt explaining it's a paid feature.

**Acceptance criteria:**
- On `ProfileSettings`, the cover image upload control is disabled/hidden for free users
- A tooltip or inline message explains this is a paid feature with a link to upgrade
- API-side: `PATCH /users/me/profile` should reject cover image updates for non-paying users (guard in `services/user.go`)
- Paid status check should use the same billing/subscription check used elsewhere in the codebase

**Technical notes:**
- Frontend: `phto-ui/src/pages/settings/ProfileSettings.tsx`
- Backend guard: `internal/services/user.go` → `UpdateCreatorProfile`
- Reuse existing subscription/plan check pattern — look for how storage tier or other paid gates are enforced

---

## Issue 2 — Gate profile slug behind paid plan ✅

**Labels:** `feature`, `billing`

**Description:**
Custom profile slugs (vanity URLs for the creator's public profile) should only be available to paying users. Free users should either get an auto-generated identifier or no public profile link at all.

**Acceptance criteria:**
- Slug input in `ProfileSettings` is disabled for free users with an upsell prompt
- API rejects slug updates from non-paying users
- If free users have an auto-generated slug, it should not be shown as editable

**Technical notes:**
- Same billing gate pattern as Issue 1
- Backend: `internal/services/user.go` → slug update logic
- Frontend: `phto-ui/src/pages/settings/ProfileSettings.tsx`

---

## Issue 3 — Slug is set-once; no subsequent updates allowed ✅

**Labels:** `feature`, `ux`

**Description:**
Once a creator sets their slug for the first time, it should be permanent — no future changes allowed. This prevents link rot for clients who have bookmarked or been sent a creator's profile URL.

**Acceptance criteria:**
- After a slug is saved, the input in `ProfileSettings` renders as read-only (not just disabled — show a lock icon or static text)
- A short explanation is shown: e.g. "Your profile URL cannot be changed after it is set."
- API-side: if `slug` is already set on `CreatorProfile`, the update endpoint ignores or rejects any slug change
- No admin override needed for MVP

**Technical notes:**
- Backend: check `CreatorProfile.Slug != ""` before applying slug update in `services/user.go`
- Frontend: check `creatorProfile.slug` from API response; if non-empty, render static text instead of input

---

## Issue 4 — Upload progress survives page navigation ✅

**Labels:** `bug`, `ux`

**Description:**
When a file upload is in progress and the user navigates away, the upload progress dialog disappears. If the user returns to the upload page, there is no indication that an upload is still happening — the upload either silently fails or completes without feedback.

**Steps to reproduce:**
1. Start uploading media in a project
2. Navigate to another page before the upload finishes
3. Navigate back to the project media page
4. Observe: no progress indicator, no way to know upload state

**Expected behavior:**
- Upload continues in the background after navigation
- A persistent indicator (e.g. a global upload tray or header badge) shows ongoing uploads from any page
- On returning to the upload page, in-progress uploads are reflected in the UI

**Acceptance criteria:**
- Upload state lives in a global Zustand store, not local component state
- A persistent upload tray component (bottom-right corner or similar) shows active uploads from anywhere in the app
- On completion or failure, the tray updates accordingly and can be dismissed

**Technical notes:**
- Current upload dialog is likely local state in a media/upload component — needs to be lifted to `stores/uploadStore.ts` (create if not exists)
- The XHR/fetch request itself persists through navigation; only the UI state is lost — fix is UI-only

---

## Issue 5 — First-time user onboarding tour

**Labels:** `feature`, `ux`

**Description:**
New creators have no guidance on what to do after signing up. A lightweight guided tour should walk them through the key actions: setting up their profile, creating their first inquiry/project, and understanding the proposal flow.

**Acceptance criteria:**
- Tour triggers automatically on first login (tracked via a `hasSeenTour` flag on the user record or localStorage)
- Covers at minimum: profile setup, creating an inquiry, and the proposal builder
- Can be dismissed at any step and re-triggered from a Help menu
- Mobile-friendly

**Proposed tour steps:**
1. Welcome — "Here's how Zeyaroo works"
2. Complete your profile (avatar, bio, services)
3. Wait for inquiries or browse the explore page
4. Build a proposal when an inquiry arrives
5. Get paid via milestones

**Technical notes:**
- Consider a lightweight library (e.g. `driver.js`) or build a simple step-by-step modal/tooltip chain
- Store `onboarding_completed_at` on the user model (backend) or use `localStorage` for MVP
- Tour component should be globally mounted in `App.tsx` and controlled by a Zustand store

---

## Issue 6 — Notification click navigates to wrong page ✅

**Labels:** `bug`

**Description:**
Clicking a notification does not navigate to the correct page. Reported specifically for a new inquiry notification — clicking it does not take the user to the inquiry detail page.

**Steps to reproduce:**
1. Receive a new inquiry notification (as a creator)
2. Click the notification in the notification panel
3. Observe: navigation goes to wrong page or does nothing

**Expected behavior:**
- Clicking a notification navigates to the relevant entity (e.g. inquiry detail, proposal, milestone)
- Each notification type has a defined target route

**Acceptance criteria:**
- Notification data includes a `link` or `entity_type` + `entity_id` field that resolves to the correct frontend route
- Notification click handler resolves the route and calls `navigate()`
- Verified working for: new inquiry, new proposal, milestone funded, milestone release request

**Technical notes:**
- Backend: `internal/models/notification.go` — check if `link` field exists; add if not
- Frontend notification panel: `phto-ui/src/components/` — find the notification list/item component and wire up `onClick → navigate(notification.link)`
- Route map needed: `inquiry → /inquiries/:id`, `proposal → /proposals/:id`, etc.

---

## Issue 7 — Proposal builder date field is too wide ✅

**Labels:** `bug`, `ui`

**Description:**
The date field in the proposal builder stretches to fill its full container width, making it look disproportionate compared to other fields. Date inputs should have a fixed or max width.

**Steps to reproduce:**
1. Open the proposal builder
2. Observe the date field (e.g. shoot date or delivery date)
3. It spans the full row width

**Expected behavior:**
- Date field has a constrained width appropriate to its content (e.g. `max-w-48` or `w-fit`)

**Acceptance criteria:**
- Date input in the proposal builder is visually consistent with other narrow inputs
- Fix is scoped to the proposal builder date field only — do not change global date input styles

**Technical notes:**
- File: likely `phto-ui/src/pages/` — find the proposal builder page/component
- Add `max-w-xs` or a specific width class to the date `<input>` wrapper

---

## Issue 8 — Proposal email to client contains localhost URL ✅

**Labels:** `bug`, `critical`

**Description:**
When a proposal is sent to a client, the email contains a `localhost` URL instead of the production/hosted URL. Clients cannot access the proposal via the link in the email.

**Steps to reproduce:**
1. Creator finalizes and sends a proposal to a client
2. Client receives the email notification
3. The proposal link in the email points to `http://localhost:...`

**Expected behavior:**
- Proposal link in the email uses the correct base URL for the environment (e.g. `https://app.zeyaroo.com/proposals/:id`)

**Acceptance criteria:**
- Backend reads base URL from an environment variable (e.g. `APP_BASE_URL` or `FRONTEND_URL`)
- All email templates use this variable for generated links
- `APP_BASE_URL` is set correctly in the Komodo stack environment for dev and prod
- No hardcoded `localhost` in any email template or service

**Technical notes:**
- Backend: search for `localhost` in `internal/` — likely in a notification/email service
- Add `APP_BASE_URL` (or equivalent) to `internal/config/config.go` and `.env.defaults`
- Update Komodo stack config for `zeyaroo-dev` to include the var (and prod stack when ready)
- Also audit any other email links (e.g. invite links, share links) for the same issue

---

## Issue 9 — Improve default milestone template (6-chapter structure)

**Labels:** `feature`, `content`

**Description:**
The default milestone template used when creating a new proposal is too sparse. It should be replaced with a more useful 6-chapter structure that reflects a typical photography/videography engagement lifecycle.

**Proposed default template:**

| # | Milestone | Description |
|---|-----------|-------------|
| 1 | Retainer | Initial deposit to confirm the booking and reserve the date |
| 2 | Pre-Shoot Preparation | Creative brief review, shot list, location scouting confirmation |
| 3 | On-Day Coverage | Day-of shooting — full session as agreed |
| 4 | Social Media Delivery | 50 edited photos delivered for client's social media use |
| 5 | Completed Album | Full edited album delivered — all final images/videos |
| 6 | Final Sign-off | Client reviews and approves all deliverables; remaining balance released |

**Acceptance criteria:**
- When a creator starts a new proposal, the milestones section is pre-populated with this 6-chapter structure
- Each milestone has a name and a default description
- Amounts are left blank (creator fills in)
- Creator can edit, reorder, add, or remove milestones freely
- The template is applied only when there are no existing milestones (i.e. new proposals only)

**Technical notes:**
- This is likely a frontend-only change — a `DEFAULT_MILESTONES` constant in the proposal builder
- File: find the proposal builder page in `phto-ui/src/pages/`
- If milestone templates are stored/served from the backend, add a seed or config there instead
