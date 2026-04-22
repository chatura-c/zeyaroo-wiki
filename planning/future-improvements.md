# Future Improvements

Ideas and features that are not yet scheduled but worth documenting for future consideration.

Items marked 🍒 are low-hanging fruit — they leverage existing patterns, models, and infrastructure with minimal new code.

---

## 1 — Custom Domain for Studio Portfolio Page 🍒

**Labels:** `feature`, `billing`, `infrastructure`

**Description:**
Allow studios to configure a custom domain name for their public portfolio page, instead of using the default Zeyaroo-hosted path (`<zeyaroo.domain>/c/<slug>`). For example, a studio could use `www.mystudio.com` instead of `app.zeyaroo.com/c/my-studio`.

**Approach options:**

| Option | Example | Pros | Cons |
|--------|---------|------|------|
| **Full custom domain** | `www.mystudio.com` | Maximum brand ownership; professional look | Requires DNS management, TLS cert provisioning, CNAME setup per domain |
| **Subdomain** | `mystudio.zeyaroo.com` | No DNS config from studio; single wildcard cert | Less brand ownership; still looks like a Zeyaroo subdomain |
| **Sub-path** | `zeyaroo.com/mystudio` | Simplest to implement; no DNS or cert work | Same as current slug approach; minimal improvement |

Full custom domain is the most valuable option for studios and aligns with how platforms like Shopify, Notion, and Squarespace handle this.

**High-level implementation sketch (full custom domain):**

1. **DNS:** Studio adds a CNAME record pointing their domain to Zeyaroo (e.g. `CNAME www.mystudio.com → custom.zeyaroo.com`)
2. **Domain verification:** Studio enters their domain in settings; Zeyaroo verifies DNS is correctly pointed (poll or webhook)
3. **TLS:** Auto-provision certs via Caddy/Let's Encrypt (Caddy handles this automatically for on-demand TLS) or use a cert manager
4. **Routing:** When a request arrives on a custom domain, look up which studio it maps to and serve their portfolio page. Could be:
   - **Reverse proxy layer:** Caddy resolves custom domain → Zeyaroo backend, which maps domain → studio
   - **Database lookup:** `CustomDomain` table linking domain → `CreatorProfile`
5. **Frontend:** Portfolio page must render correctly at the custom domain (no hardcoded Zeyaroo base URL assumptions)

**Data model:**

```
CustomDomain
  ├── id
  ├── creator_id        → CreatorProfile
  ├── domain            (e.g. "www.mystudio.com")
  ├── verified_at       (null until DNS verified)
  ├── ssl_provisioned_at
  ├── status            (pending → verified → active → error)
  └── created_at
```

**Billing gate:**
This should be a paid feature. Custom domain configuration is only available on a paid plan (same gate pattern as slug and cover image).

**Edge cases & considerations:**
- Domain takeover prevention: verify the studio actually controls the domain before activating
- Multiple domains per studio? (e.g. both `mystudio.com` and `www.mystudio.com`) — decide on limits
- What happens if DNS breaks or cert expires? Graceful fallback or error page?
- SEO: ensure canonical URL handling so content isn't duplicated across custom domain and Zeyaroo slug path
- Caddy on-demand TLS: `on_demand_tls` directive can auto-provision certs as requests arrive — fits this use case well
- Rate limiting on cert provisioning to avoid Let's Encrypt bans

**Acceptance criteria:**
- Studio can enter a custom domain in profile settings (behind paywall)
- System verifies CNAME is correctly pointed before activation
- TLS cert is auto-provisioned and renewed
- Portfolio page renders at the custom domain with no broken assets or API calls
- Original Zeyaroo slug path still works (redirect or canonical)
- Admin dashboard shows custom domain status per studio

**Technical notes:**
- Backend: new `CustomDomain` model, migration, service + handler
- Frontend: settings page section for domain config, verification status UI
- Infrastructure: Caddy config update for on-demand TLS + custom domain routing
- Current reverse proxy: Caddy on prod VM — ideal for on-demand TLS support

---

## 2 — Portfolio Analytics 🍒

**Labels:** `feature`, `billing`

**Description:**
Give creators insight into how their public portfolio is performing — page views, referral sources, visitor geography. Creators currently have no idea if anyone is looking at their portfolio page.

**Why low-hanging:**
The portfolio page already exists at `/c/:slug`. Adding a view counter is trivial — increment a field on each visit. Referral and geography can be derived from request headers (Referer, CloudFlare/OCI geo headers) without any client-side tracking script.

**Implementation sketch:**
- Add `PortfolioViews` model (or fields on `CreatorProfile`): `total_views`, `views_this_week`, `views_this_month`
- Middleware or handler on the public portfolio route increments the counter
- Referral tracking: log `Referer` header into a `PortfolioReferral` table (referrer, count)
- Geography: if behind CDN/proxy that sets geo headers, log country into `PortfolioGeography` table
- Dashboard widget on creator's home showing top-line numbers + a simple chart
- Gate detailed analytics (referral sources, geography) behind paid plan; free tier sees only total views

**Data model:**

```
PortfolioView
  ├── id
  ├── creator_id
  ├── viewed_at
  ├── referrer          (from Referer header, nullable)
  ├── country           (from geo header, nullable)
  └── user_agent_hash   (for basic dedup, nullable)

CreatorProfile (add fields)
  ├── total_views
  └── views_last_7d     (materialized counter, refreshed periodically)
```

**Acceptance criteria:**
- Each visit to `/c/:slug` records a view
- Creator dashboard shows total views, views this week, top referrers
- Detailed analytics (referral breakdown, geography) gated behind paid plan
- Admin can see aggregate platform-wide portfolio traffic

---

## 3 — Public Reviews & Testimonials

**Labels:** `feature`, `trust`, `discovery`

**Description:**
Allow clients to leave a review after a project is completed. Reviews appear on the creator's public portfolio page, building social proof and driving client trust.

**Implementation sketch:**
- After milestone sign-off (last milestone completed), prompt client to leave a review
- `Review` model linked to `Project` + `CreatorProfile`
- Rating: 1–5 stars + optional text testimonial
- Creator can choose which reviews to display (pin/hide) but cannot delete or edit them
- Reviews render on public portfolio page under the portfolio content
- Reviews also feed into the discovery/marketplace ranking

**Data model:**

```
Review
  ├── id
  ├── project_id        → Project
  ├── creator_id        → CreatorProfile (reviewee)
  ├── client_id        → User (reviewer)
  ├── rating            (1–5)
  ├── testimonial       (text, nullable)
  ├── pinned            (creator chose to highlight)
  ├── hidden            (creator chose to hide)
  └── created_at
```

**Edge cases:**
- Only one review per project per client
- Client must have an actual completed project — no fake reviews
- Creator can hide but not delete (transparency)
- Report/flag mechanism for inappropriate content

**Acceptance criteria:**
- Client receives prompt after project completion
- Review appears on creator's portfolio page
- Creator can pin/hide reviews
- Average rating shown on portfolio header
- Admin can moderate reported reviews

---

## 4 — Team / Studio Accounts

**Labels:** `feature`, `architecture`

**Description:**
Currently one user = one studio. Many real studios have multiple photographers, editors, or admins who need access under the same studio identity. Allow multiple user accounts under a single studio entity.

**Implementation sketch:**
- Introduce `Studio` entity separate from `User`/`CreatorProfile`
- `Studio` has members with roles: `owner`, `admin`, `member`
- `CreatorProfile` links to `Studio` instead of directly to `User`
- Shared portfolio, shared projects, shared billing
- Each member still has their own login; permissions control what they can do

**Data model:**

```
Studio
  ├── id
  ├── name
  ├── slug
  ├── owner_id          → User
  └── created_at

StudioMember
  ├── id
  ├── studio_id         → Studio
  ├── user_id           → User
  ├── role              (owner, admin, member)
  └── joined_at

CreatorProfile (modify)
  ├── studio_id         → Studio (instead of directly on User)
  └── ...
```

**Why NOT low-hanging:**
This is a significant architecture change. The current model assumes `User = Creator = Studio`. Decoupling requires migrating existing data, updating every creator-scoped query, and rethinking permissions. Worth doing but not a quick win.

**Acceptance criteria:**
- Studio owner can invite members via email
- Members can access studio projects, proposals, media
- Portfolio page shows the studio, not just one creator
- Billing is at the studio level, not individual
- Existing single-creator accounts migrate to studio-with-one-member

---

## 5 — Calendar & Booking 🍒

**Labels:** `feature`, `ux`

**Description:**
Add a calendar view for shoot dates, milestone deadlines, and availability. Integrate with iCal so creators can sync Zeyaroo dates with their existing calendar (Google Calendar, Apple Calendar, etc.).

**Why low-hanging:**
Milestones and proposals already have dates (shoot date, delivery date). The data exists — just needs a new view and an iCal feed endpoint.

**Implementation sketch:**
- New `/calendar` page showing month/week view with milestones and key dates
- iCal feed: `GET /api/v1/calendar/ical?token=<creator-token>` — subscribable calendar URL
- Each milestone due date, shoot date, and proposal deadline becomes a calendar event
- Availability blocking: creator marks dates as "busy" or "available" (future — start with just displaying existing dates)

**Data model:**

```
CalendarEvent (or derive from existing data)
  ├── derived from Milestone.due_date, Proposal.shoot_date, etc.
  └── no new table needed for MVP — just render existing dates

CalendarBlock (future)
  ├── id
  ├── creator_id
  ├── date
  ├── status           (busy, available, tentative)
  └── note
```

**Acceptance criteria:**
- Creator sees a calendar view with all their milestone dates and shoot dates
- iCal feed URL that can be subscribed to from external calendars
- Calendar updates automatically when milestones/dates change

---

## 6 — Contract & E-Signature

**Labels:** `feature`, `legal`, `billing`

**Description:**
Proposals outline terms but aren't legally binding contracts. Add a contract layer with e-signature so both creator and client sign off before escrow is funded. Reduces disputes and adds professionalism.

**Implementation sketch:**
- After proposal is finalized, generate a contract document from the proposal data (terms, milestones, amounts)
- Contract includes standard legal clauses (configurable per studio or platform-wide)
- Both parties e-sign (typed name + checkbox acknowledgment, or integration with a provider like DocuSign/HelloSign)
- Signed contract is stored as a PDF and linked to the project
- Optional: make contract signing a prerequisite before first milestone can be funded

**Data model:**

```
Contract
  ├── id
  ├── project_id        → Project
  ├── proposal_id       → Proposal
  ├── pdf_url           (generated contract document)
  ├── status            (draft → sent → signed_by_creator → fully_signed)
  ├── creator_signed_at
  ├── client_signed_at
  └── created_at
```

**Acceptance criteria:**
- Contract auto-generated from finalized proposal
- Both parties can review and sign electronically
- Signed PDF stored and downloadable
- Contract status visible in project workspace

---

## 7 — Client Portal 🍒

**Labels:** `feature`, `ux`, `strategy`

**Description:**
A first-class client experience that runs on two parallel paths — not a funnel that forces account creation. Most clients are one-off and should never be forced to sign up. But returning clients who want the premium experience should get one.

### Two paths, both first-class

| | **Accountless (public pages)** | **Registered (client portal)** |
|---|---|---|
| Who | One-off clients, low-friction | Returning clients, or one-off who want to opt in |
| Access | Magic links (proposal `/p/:token`, payment `/pay/:token`, brief `/brief/:token`, gallery `/gallery/:token`) | Authenticated dashboard |
| Friction | Zero — click link, do the thing | Sign-up, but optional and clearly positioned as an upgrade |
| Data scope | Single project/proposal only | Cross-project view, history, receipts |
| Persistence | No — link is the entry point | Yes — everything in one place |

**The accountless path is permanent, not a stepping stone.** It should stay exactly as frictionless as it is now. The registered path is a premium upgrade, not a requirement.

### What changes from today

**Current problems:**
- Registered clients log in and get the creator app with some things hidden — feels like a hollowed-out version, not a real experience
- Dashboard shows irrelevant creator stats (earnings, storage, active leads)
- Vault (`/vault`) exists but isn't in the sidebar — clients can't discover it
- Client project list is identical to creator's — no client-relevant filtering or layout
- Settings has payout/billing tabs (creator-only) that confuse clients

**What the client portal should be:**
- **ClientDashboard** — active projects with "next action" badges (pending payment, delivery ready, review needed), recent activity feed, quick links to open proposals/payments
- **Client sidebar** — Dashboard, My Projects, My Vault, My Invoices, Settings (notification + payment preferences only)
- **Project list** — client-filtered: show status + what the client needs to do next, not media counts or storage
- **Project detail** — keep existing `ClientDeliveryView` but wrap in a client-aware layout (no "Upload Media" or "Add Album" hints)
- **Vault** — already works, just needs to be in the nav
- **Settings** — payment methods (saved cards via PayHere vaulting), notification preferences, account management

### Implementation sketch

**Phase 1 — Layout & navigation (2–3 days)**
- `ClientLayout` component: client-specific sidebar, header, branding
- Route restructuring: client-role users render under `ClientLayout` instead of `Layout`
- `ClientDashboard` page replacing the creator dashboard for `role === 'client'`
- Add Vault, Invoices to client sidebar

**Phase 2 — Client-specific views (3–5 days)**
- `ClientProjectList` — action-oriented cards ("3 milestones pending", "1 delivery ready", "Payment due")
- `ClientProjectDetail` — wrap `ClientDeliveryView` in cleaner client framing
- `ClientSettings` — payment methods, notifications, account (no payout/storage/billing tabs)
- Invoice/receipt list page for registered clients

**Phase 3 — Cross-project features (later)**
- Rehire flow: "Hire Again" from vault → pre-filled inquiry
- Project timeline view: visual history of milestones, payments, deliveries
- Creator favorites / bookmarks

### Accountless path improvements (separate from portal)

These are independent enhancements to the public pages that benefit both paths:

- **Claim flow smoothing**: guest → registered client transition should feel like upgrading, not starting over. Claimed project data carries over seamlessly.
- **Post-action prompt**: after a client completes a payment or accepts a delivery on a public page, gently show "Create an account to track all your projects in one place" — no pressure, just an option
- **Persistent magic links**: ensure public page tokens don't expire too aggressively for one-off clients who bookmark their proposal/payment page

### Data model

No new backend models needed for Phase 1–2. The existing `User` with `role: 'client'`, project membership, milestones, and invoices already contain everything. Changes are frontend routing + layout.

For Phase 3:
```
ClientFavorite (optional)
  ├── id
  ├── client_id         → User
  ├── creator_id        → CreatorProfile
  └── created_at
```

**Acceptance criteria:**
- Registered client lands on `ClientDashboard` (not creator dashboard)
- Client sidebar shows: Dashboard, Projects, Vault, Invoices, Settings
- Client dashboard shows active projects with next-action indicators
- Vault is accessible from client navigation
- Client settings show payment methods + notifications only (no payout/storage)
- Public pages (proposal, payment, brief, gallery) remain unchanged — zero-friction accountless path preserved
- Post-action prompt on public pages optionally invites account creation
- Guest → registered client claim flow preserves existing project access

---

## 8 — Referral Program 🍒

**Labels:** `feature`, `growth`

**Description:**
Let creators invite other creators or clients to Zeyaroo. Each user gets a referral code; when a referred user signs up and completes an action (first project, first payment), the referrer gets a reward (storage credit, billing credit, or cashback).

**Why low-hanging:**
Just a referral code on the user model + tracking in the signup flow. The post-signup hook system already exists (`AuthService.OnSignup`). Minimal new infrastructure.

**Implementation sketch:**
- Add `referral_code` to `User` (auto-generated, unique)
- Add `referred_by` to `User` (nullable, links to referring user)
- Signup endpoint accepts optional `ref` query param or form field
- `PostSignupHook` records the referral relationship
- Reward triggers on first completed project or first payment (configurable)
- Reward: billing credit or storage credit (simplest — just add to existing billing/storage models)

**Data model:**

```
User (add fields)
  ├── referral_code     (unique, auto-generated on signup)
  └── referred_by_id    → User (nullable)

ReferralReward
  ├── id
  ├── referrer_id       → User
  ├── referred_id       → User
  ├── trigger_event     (first_project, first_payment)
  ├── reward_type       (billing_credit, storage_credit)
  ├── reward_amount
  └── awarded_at
```

**Acceptance criteria:**
- Every user has a shareable referral link
- Referred signups are tracked
- Reward is issued when trigger event occurs
- Creator can see their referral stats in settings

---

## 9 — Data Export & GDPR Compliance

**Labels:** `compliance`, `infrastructure`

**Description:**
Users should be able to export all their data (profile, projects, media metadata, invoices) in a portable format. Required for GDPR compliance (right to data portability) and builds trust.

**Implementation sketch:**
- `GET /api/v1/users/me/export` — triggers an async job that packages user data
- Export includes: profile JSON, projects JSON, invoices JSON, media metadata (not files themselves — too large)
- Delivered as a ZIP download or download link sent via email
- Account deletion endpoint: `DELETE /api/v1/users/me` — soft delete + schedule hard delete after grace period
- Anonymize or delete associated data per data retention policy

**Acceptance criteria:**
- User can request a data export from settings
- Export is generated asynchronously and delivered as downloadable ZIP
- User can request account deletion
- Account deletion removes/anonymizes personal data within defined retention period

---

## 10 — API & Webhooks

**Labels:** `feature`, `integrations`

**Description:**
Expose a public API and webhook system so studios can integrate Zeyaroo with their CRM, accounting software, or CMS. Enables power users and opens up an integration ecosystem.

**Implementation sketch:**
- API keys: generate in settings, scoped permissions (read-only, read-write, per-resource)
- Rate limiting per API key
- Webhook subscriptions: configure endpoint URL + event types (project.created, milestone.funded, payment.received)
- Webhook delivery with retries and event log
- API documentation (OpenAPI/Swagger)

**Data model:**

```
APIKey
  ├── id
  ├── user_id
  ├── name
  ├── key_hash          (store hash, not plaintext)
  ├── permissions       (JSON or enum set)
  ├── last_used_at
  └── created_at

WebhookSubscription
  ├── id
  ├── user_id
  ├── url
  ├── event_types       (JSON array)
  ├── secret            (for payload signing)
  ├── active
  └── created_at

WebhookDelivery
  ├── id
  ├── subscription_id
  ├── event_type
  ├── payload
  ├── response_status
  ├── attempts
  └── delivered_at
```

**Acceptance criteria:**
- User can create API keys with scoped permissions
- User can configure webhook endpoints and event types
- Webhooks are delivered with retries on failure
- API is documented

---

## 11 — PWA & Mobile Optimization 🍒

**Labels:** `feature`, `ux`

**Description:**
Make Zeyaroo installable as a PWA and optimize the mobile experience. Creators are often on-the-go — they need to respond to inquiries, review proposals, and check deliverables from their phone. Push notifications for critical events (new inquiry, milestone funded) would improve responsiveness.

**Why low-hanging:**
Adding a `manifest.json` and service worker is mechanical. Push notifications can leverage the existing notification system + Firebase Cloud Messaging (Firebase is already wired up). The frontend is already responsive with Tailwind — just needs PWA plumbing.

**Implementation sketch:**
- Add `manifest.json` with app name, icons, theme color
- Add service worker for offline caching (workbox or hand-rolled)
- Push notifications: register FCM token on frontend, send via backend when notification event fires
- Optimize critical mobile flows: inquiry response, proposal review, media upload

**Acceptance criteria:**
- Zeyaroo is installable on mobile browsers (Add to Home Screen)
- Push notifications work for key events (new inquiry, proposal, milestone)
- Critical flows work well on mobile viewport
- Offline shows a meaningful fallback (cached shell or message)

---

## Low-Hanging Fruit Summary

These can be shipped quickly because they leverage existing data models, infrastructure, and patterns:

| # | Feature | Why it's easy |
|---|---------|---------------|
| 1 | Custom Domain | Caddy already supports on-demand TLS; just new model + Caddy config |
| 2 | Portfolio Analytics 🍒 | Portfolio page exists; just add view counter + log headers |
| 5 | Calendar & Booking 🍒 | Milestone dates already in DB; new view + iCal endpoint |
| 7 | Client Portal 🍒 | All data exists; new frontend layout scoped to client role |
| 8 | Referral Program 🍒 | Post-signup hook system exists; just add referral code + tracking |
| 11 | PWA & Mobile 🍒 | Firebase already wired; manifest + service worker is mechanical |
