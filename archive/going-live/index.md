# Zeyaroo — Going Live Roadmap

## How to Use This Document

This is the master orchestration document for the Zeyaroo going-live roadmap.
Each stage is implemented in a dedicated session. At the start of every session,
the agent reads this file **first**, then reads the assigned stage file, then executes.

---

## Agent Instructions (Read Every Session)

You are the **implementing architect** for this stage. Your job is to read, plan,
and delegate — not to directly edit files yourself unless the change is genuinely
trivial (one line, no judgment needed).

---

### Step-by-step workflow

**1. Orient** — read this file in full. Note the current progress state and any
  blockers logged in the session log. Then read the assigned stage file (`N.md`).

**2. Explore the codebase before planning.** The stage files were written before
  the code existed. Reality may differ. Before delegating, use the Explore subagent
  or Grep/Glob directly to verify:
  - Which files already exist vs. what needs to be created
  - How existing patterns are implemented (e.g., how the webhook handler works,
    how GORM models are structured, how TanStack Query mutations are wired up)
  - Any naming inconsistencies between the stage file and the actual code

**3. Plan before delegating.** Produce a concise implementation plan that groups
  tasks by:
  - **Layer** — backend-only, frontend-only, or full-stack (backend + frontend)
  - **Dependency order** — which tasks must ship first before others can start
  - **Risk** — destructive migrations, interface changes, or cross-repo contract
    changes (these need explicit callouts in the subagent prompt)

**4. Delegate using the Agent tool.** Model selection rules:
  - `haiku` — single-file mechanical edits: adding a model field, adding a route
    registration line, writing a small Go struct, updating a Tailwind class.
    Use this aggressively — most individual tasks in a stage are haiku-level.
  - `sonnet` — multi-file judgment tasks: implementing a service + its handler +
    wiring it into the router, building a React page with query hooks, designing
    a state machine or background job.
  - `opus` — only for deep architectural reasoning (e.g., designing a new
    abstraction that several stages will depend on). Rare.

  Write subagent prompts that are **self-contained**: include the relevant file
  paths, the existing patterns to follow (quote a short example), the exact
  interface the task must satisfy, and what a correct result looks like.
  Do not assume the subagent has read this file or the stage file.

**5. Parallelize aggressively.** Backend and frontend tasks for the same stage
  are almost always independent — launch them in the same message. Within
  the backend, model field additions are independent of handler logic. Within
  the frontend, page components are independent of shared UI components.

**6. Verify before marking complete.** After a subagent finishes:
  - Spot-check the files it touched (Read or Grep)
  - Confirm the acceptance criteria from the stage file are met
  - Flag any drift from the cross-cutting architectural decisions below

**7. Commit and update progress.**
  - Each repo (`phto-api`, `phto-ui`, `zeyaroo-e2e`) commits independently
  - Short commit messages, no co-authored-by lines
  - Update the Progress Tracker in this file: `[x]` done, `[~]` partial/blocked
  - Add a row to the Session Log with date, stage, and a one-line summary

---

### Architecture context (non-negotiable)

- **Router**: Chi (`github.com/go-chi/chi/v5`) — not Gin. Some stage files say
  "Gin middleware" — translate to Chi middleware equivalents.
- **Backend layers**: `handlers → services → repository → DB`. Handlers call
  services, services call repositories, repositories call GORM. Never skip layers.
- **Auth**: `middleware.GetUserID(r.Context())` to extract user ID in handlers.
  Admin-only routes use a separate middleware that checks `user.Role == "admin"`.
- **Response helpers**: `respondJSON(w, status, data)` and
  `respondError(w, status, msg)` from `handlers/response.go`.
- **DB**: GORM with SQLite now, PostgreSQL in Stage 9. Always use GORM idioms.
  GORM `AutoMigrate` is called on startup — adding a field to a model is enough.
- **Storage**: `internal/storage` has a `Storage` interface. Use it; never import
  a concrete storage type in handlers or services. Stage 7 swaps the impl to B2.
- **Event bus**: Thumbnails and watermarks go through `EventBus → EventProcessor`,
  not inline in upload handlers. Stage 7 extends this.
- **Frontend stack**: React 19, TypeScript, Vite, TanStack Query, Zustand,
  Tailwind CSS 4. Types live in `src/types/index.ts`. API hooks in `src/api/`.
- **Three git repos**: `phto-api` (backend), `phto-ui` (frontend), `zeyaroo-e2e`.
  API contract changes (adding/removing fields) require updating both `phto-ui`
  and `zeyaroo-e2e` in the same session.

---

## Roadmap Overview

### Phase 1 — The Transactional Engine (Critical Path to Launch)

These three stages must ship before Zeyaroo takes real money. Do them in order;
each builds on the previous.

| Stage | Title | Status |
|-------|-------|--------|
| [1](1.md) | Global & Local Payment Infrastructure | `[x] complete` |
| [2](2.md) | Escrow & Milestone Logic Hardening | `[x] complete` |
| [3](3.md) | Zero-Friction Lifecycle (Guest Mode & Project Claiming) | `[x] complete` |

**Dependency notes:**
- Stage 2 depends on Stage 1's gateway interface and escrow models being in place.
- Stage 3's payout logic requires Stage 1's `PayoutDetails` model.
- Stage 9 (PostgreSQL) is a prerequisite for production but is intentionally deferred
  to Phase 3 — SQLite is fine for early testing.

---

### Phase 2 — Revenue & Growth (Monetization)

| Stage | Title | Status |
|-------|-------|--------|
| [4](4.md) | SaaS Tiering & Monetization Engine | `[x] complete` |
| [5](5.md) | Growth Engine & Viral Loops | `[ ] not started` |
| [6](6.md) | Automated Retention & CRM | `[x] complete` |

**Dependency notes:**
- Stage 4 builds on Stage 1's DodoPayments integration (subscription billing reuses the same gateway).
- Stage 4's feature-gating middleware must be in place before Stage 6 (notification
  preferences respect tier limits).
- Stage 5 and Stage 6 are largely independent of each other and can run in parallel
  if bandwidth allows.

---

### Phase 3 — Infrastructure & Aesthetics (Scalability)

| Stage | Title | Status |
|-------|-------|--------|
| [7](7.md) | Media Processing & Storage Optimization | `[ ] not started` |
| [8](8.md) | The "Luxury Studio" UI/UX Overhaul | `[ ] not started` |
| [9](9.md) | Database & DevOps Hardening | `[ ] not started` |

**Dependency notes:**
- Stage 7 (pre-signed uploads, CDN) is largely independent and can be started any time
  after Phase 1 ships. The storage abstraction (`internal/storage`) already exists.
- Stage 8 is pure frontend and can run fully in parallel with any backend stage.
- Stage 9 (PostgreSQL migration) is the most disruptive — do it before going live
  with real traffic, but after the feature set is stable.

---

### Phase 4 — Trust & Reliability (Operational Excellence)

| Stage | Title | Status |
|-------|-------|--------|
| [10](10.md) | Legal Infrastructure & Dispute Management | `[ ] not started` |
| [11](11.md) | Platform Observability & Trust Signals | `[ ] not started` |
| [12](12.md) | Creator Productivity (AI & Roadmap) | `[~] partial (AI tasks skipped)` |

**Dependency notes:**
- Stage 10's dispute fee logic reuses Stage 1's payment gateway interface.
- Stage 11 (Sentry, health checks) can be partially done early — basic health
  endpoints are useful from day one.
- Stage 12 (AI features) is non-blocking and purely additive.

---

## Cross-Cutting Architectural Decisions

These apply globally and subagents must respect them:

1. **Gateway abstraction first** — Stage 1 defines the `Gateway` interface. All
   subsequent payment logic (escrow release, subscription billing, dispute fees)
   goes through this interface. Never call DodoPayments or PayHere directly from
   business logic.

2. **Feature gating** — Stage 4 introduces `FeatureGate` service methods
   (`CanCreateProject`, `HasStorageSpace`) that return `402 Payment Required`.
   Any feature added in Stages 5–12 that is tier-locked must use this pattern —
   do not add ad-hoc tier checks in handlers.

3. **DB driver abstraction** — GORM handles both SQLite and PostgreSQL. Always
   use GORM idioms; raw SQL is only acceptable for performance-critical queries
   that GORM cannot express. Check `DB_DRIVER` env var, never hardcode.

4. **Storage abstraction** — `internal/storage` has a `Storage` interface. All
   file I/O goes through it. Stage 7 swaps the implementation to Backblaze B2
   pre-signed uploads; handlers must never import a concrete storage type directly.

5. **Event-driven media processing** — thumbnails and watermarks are generated
   via `EventBus → EventProcessor`, not inline in upload handlers. Stage 7
   extends this pipeline to add watermarking and EXIF extraction workers.

6. **No breaking API changes** — the frontend and E2E tests depend on
   `/api/v1/*` contracts. Adding fields to responses is always safe. Removing or
   renaming fields, or changing URL structures, requires updating both `phto-ui`
   and `zeyaroo-e2e` in the same session.

7. **Background scheduler** — Stage 6 introduces a job scheduler (`gocron` or
   equivalent). All subsequent background jobs (Stage 7 media processing health,
   Stage 12 archiving, etc.) must be registered in this same scheduler — do not
   launch ad-hoc goroutines in `main.go`.

8. **Tier config is the source of truth for limits** — Stage 4 defines
   `GetTierConfig(tier)`. Any handler or service that enforces storage, project
   count, or fee percentage must call this function. Never hardcode tier limits.

---

### Inter-Stage Dependency Map

Quick reference for what each stage depends on and what it enables:

| Stage | Hard depends on | Enables / unlocks |
|-------|----------------|-------------------|
| 4 | Stage 1 (Dodo gateway, fee logic) | Tier-locked features in 5, 6, 8, 12 |
| 5 | Stage 4 (vanity slug, tier badge) | Stage 6 (inquiry tracking via referrals) |
| 6 | Stage 4 (tier gate for notifications), Stage 1 (webhook hooks) | Stage 7 (scheduler), Stage 11 (media health) |
| 7 | Stage 6 (scheduler infrastructure) | Stage 10 (GDPR B2 deletion), Stage 11 (media health), Stage 12 (AI pipeline) |
| 8 | Stage 7 (watermark UX, upload tray), Stage 4 (tier badge) | (aesthetic; no hard blockers) |
| 9 | No hard deps, but do after feature set is stable | Stage 10 (account deletion), Stage 11 (health checks) |
| 10 | Stage 1 (dispute fee payment), Stage 7 (B2 deletion) | — |
| 11 | Stage 7 (media worker health), Stage 9 (DB health check) | — |
| 12 | Stage 7 (B2 storage, thumbnail pipeline, metadata column), Stage 6 (scheduler) | — |

---

## Progress Tracker

> Update this section at the end of each session. Format: `[x]` = done, `[~]` = partial/blocked, `[ ]` = not started.

### Stage 1 — Global & Local Payment Infrastructure
- [x] Task 1: Extend models for multi-currency (Project, Milestone)
- [x] Task 2: Payment Gateway Strategy Interface
- [x] Task 3: DodoPayments Checkout Service
- [x] Task 4: Secure storage of Creator Payout Details
- [x] Task 5: Creator Payout Information UI
- [x] Task 6: Regional Gateway Resolver Logic
- [x] Task 7: DodoPayments Webhook Endpoint
- [x] Task 8: Manual Bank Transfer Request UI
- [x] Task 9: Manual Funding Approval Backend
- [x] Task 10: Platform Fee Deduction Logic
- [x] Task 11: Transaction Ledger Table
- [x] Task 12: Frontend Net Payout Display

### Stage 2 — Escrow & Milestone Logic Hardening
- [x] Task 1: Proposal Math Validation (server-side)
- [x] Task 2: Enforce Escrow for "None" type milestones
- [x] Task 3: DB Transactions for fund/release operations
- [x] Task 4: 7-Day Auto-Release Background Job
- [x] Task 5: Client Quality Confirmation Checkbox
- [x] Task 6: Project Deletion Lock on funded milestones
- [x] Task 7: Milestone Sequence Enforcement
- [x] Task 8: Admin Dispute Resolution API
- [x] Task 9: Request Changes reason history
- [x] Task 10: Automatic delivery lock on "Submitted"
- [x] Task 11: Refund Logic Stub for cancelled projects

### Stage 3 — Zero-Friction Lifecycle
- [x] Task 1: Secure Public Access Tokens for proposals
- [x] Task 2: Email-OTP Request Logic
- [x] Task 3: Guest-to-JWT Authentication Handler
- [x] Task 4: Public Proposal Viewing UI
- [x] Task 5: Guest email support in Milestone Payouts
- [x] Task 6: Project Hand-Over State Trigger
- [x] Task 7: "Claim Your Account" Invitation UI
- [x] Task 8: Technical Project Claim Service
- [x] Task 9: Soft-lock funded guest projects (done in Stage 2)
- [x] Task 10: Magic Link copy-to-clipboard tool
- [x] Task 11: Auto-provision "My Deliveries" project

### Stage 4 — SaaS Tiering & Monetization Engine
- [x] Task 1: Add `SubscriptionTier`, `SubscriptionID`, `SubscriptionExpiresAt` to User model
- [x] Task 2: Define tier config struct + `GetTierConfig(tier)` utility (starter/pro/studio limits)
- [x] Task 3: Feature-gating service methods (`CanCreateProject`, `HasStorageSpace`) + 402 handler
- [x] Task 4: Dynamic platform fee deduction in escrow release (look up creator tier at release time)
- [x] Task 5: `BillingSettings.tsx` — plans page with tier cards and "Upgrade" buttons
- [x] Task 6: `POST /billing/subscribe` — Dodo checkout session for subscription plan
- [x] Task 7: Extend Dodo webhook handler for `subscription.created/renewed/cancelled` events
- [x] Task 8: `UpgradeModal.tsx` — reusable 402-triggered modal with upgrade CTA
- [x] Task 9: Tier badge in `Layout.tsx` sidebar (Bronze/Silver/Gold color coding)
- [x] Task 10: Lock custom logo/cover fields behind Pro gate in `ProfileSettings.tsx`
- [x] Task 11: `PATCH /admin/users/:id/tier` — admin override endpoint
- [x] Task 12: Downgrade cleanup background job (reset expired subscriptions to starter)

### Stage 5 — Growth Engine & Viral Loops [skip]
- [ ] Task 1: OpenGraph + Twitter Card meta tags on `SharedGallery.tsx`
- [ ] Task 2: JSON-LD `ProfessionalService` structured data on public creator profiles (GEO)
- [ ] Task 3: "Powered by Zeyaroo" referral footer on `SharedGallery.tsx` and `PublicProposal.tsx`
- [ ] Task 4: `GET /sitemap.xml` — dynamic sitemap of all public creator profiles
- [ ] Task 5: "Share Gallery" / "Share Profile" buttons using Web Share API + copy fallback
- [ ] Task 6: `GrowthStats` event log — track inquiry creation and first payment per inquiry source
- [ ] Task 7: `ReferredByUserID` on User model + `?ref=` param in registration flow
- [ ] Task 8: `slug` (unique) on `CreatorProfile` + vanity URL routing + settings field
- [ ] Task 9: `<link rel="canonical">` on creator profile page (slug takes precedence over UUID)
- [ ] Task 10: `/hire/:id` — standalone "Contact Me" lead capture page (mobile-optimized)
- [ ] Task 11: Log timestamp when ZIP download endpoint is first hit per project (conversion event)
- [ ] Task 12: `GET /public/creators/:id/og-image.png` — dynamic social image (avatar + 3 thumbnails)

### Stage 6 — Automated Retention & CRM
- [x] Task 1: Background scheduler infrastructure in `main.go` (gocron or ticker + job registry)
- [x] Task 2: `internal/services/whatsapp.go` — WhatsApp API integration + `SendMessage()` method; add `whatsapp_number` to User
- [x] Task 3: New inquiry WhatsApp alert — hook into `CreateInquiry`, trigger if creator has number
- [x] Task 4: Unaccepted proposal reminder worker (48h nudge for `negotiating` proposals)
- [x] Task 5: Milestone funded notification — Email + WhatsApp on webhook success for creator
- [x] Task 6: `models.Questionnaire` — `project_id`, `schema` (JSON questions), `responses` (JSON)
- [x] Task 7: Questionnaire builder UI in proposal creation flow (text + multiple-choice rows)
- [x] Task 8: Client-side questionnaire response view on `PublicProposal.tsx` + save endpoint
- [x] Task 9: Post-delivery feedback email worker (7 days after final milestone release)
- [x] Task 10: `NotificationSettings` JSON on User model + settings UI (per-channel opt-out)
- [x] Task 11: `ActivityFeed.tsx` dashboard component — last 10 client events from event log

### Stage 7 — Media Processing & Storage Optimization
- [x] Task 1: `POST /projects/:id/media/upload-url` — generate B2 pre-signed `PutObject` URL (15-min)
- [x] Task 2: Refactor upload UI to direct-to-B2 (`axios.put` to pre-signed URL) + commit endpoint
- [x] Task 3: Standardize B2 key hierarchy: `users/{uid}/projects/{pid}/media/{mid}/{filename}`
- [x] Task 4: `MediaProcessor` background worker — download orig, generate `_thumb.jpg`, upload to B2
- [x] Task 5: Watermarking service — `ApplyWatermark()` producing `web_preview` variant per image
- [x] Task 6: Conditional media URL logic — serve `web_preview` if milestone unpaid, `orig` if paid
- [x] Task 7: Switch `mediaUrl` helper to serve via `media.zeyaroo.com` (Cloudflare[or something similar] CDN CNAME)
- [ ] Task 8: B2 lifecycle rules documentation/script (keep current, delete failed multiparts after 7d)
- [x] Task 9: EXIF extraction in background worker (`rwcarlsen/goexif`) → store in `media.metadata` JSON
- [x] Task 10: `StreamProjectZip(projectID)` — streaming zip of `orig` files only for "Download All"
- [ ] Task 11: FFmpeg video transcoding worker — 720p `.mp4` preview for video uploads
- [x] Task 12: `Cache-Control: max-age=86400` on all signed media URL responses

### Stage 8 — Luxury Studio UI/UX Overhaul [skip]
- [ ] Task 1: Update Tailwind config — `surface-950` deep obsidian; reduce ambient glow in `Layout.tsx`
- [ ] Task 2: Serif font (Lora / Instrument Serif) — apply `font-serif` to proposal + profile headings
- [ ] Task 3: Masonry layout in `SharedGallery.tsx` (CSS columns or react-masonry; responsive 1→3+ cols)
- [ ] Task 4: `CoverImageID` on Proposal model + cinematic hero header on `PublicProposal.tsx`
- [ ] Task 5: `UploadTray.tsx` — fixed bottom-right tray tracking multi-file upload progress
- [ ] Task 6: Premium empty states for Projects, Inquiries, Albums (SVG icon + serif sub-text)
- [ ] Task 7: `MediaSkeleton.tsx` — animated masonry-shaped loading skeletons in gallery views
- [ ] Task 8: Watermarked image hover overlay — "Unlock high-res" badge linked to Pay Milestone
- [ ] Task 9: Confetti burst on proposal acceptance (`canvas-confetti`)
- [ ] Task 10: Glassmorphism nav — `bg-surface-950/80 backdrop-blur-md` in `Layout.tsx` + modals
- [ ] Task 11: `BrandColor` (hex) on `CreatorProfile` — Pro gate; override CSS accent on public pages

### Stage 9 — Database & DevOps Hardening [skip]
- [ ] Task 1: PostgreSQL support in `database.go` — `DB_DRIVER=postgres` branch with `gorm.io/driver/postgres`
- [ ] Task 2: `docker-compose.prod.yml` — `postgres:16-alpine` service, health checks, volumes, networks
- [ ] Task 3: Harden `phto-ui/Dockerfile` — multi-stage, `nginx:stable-alpine`, `try_files` SPA routing, <50MB image
- [ ] Task 4: Structured logging with `slog` — JSON handler in prod; add `user_id`/`project_id` to service logs
- [ ] Task 5: GORM connection pool config — `MaxIdleConns`, `MaxOpenConns`, `ConnMaxLifetime` via env vars
- [ ] Task 6: Non-root user in API Dockerfile (`adduser zeyaroo`, correct upload dir perms, `USER zeyaroo`)
- [ ] Task 7: Rate limiting Chi middleware — 60 req/min general; 5/10min for `/auth/request-otp`
- [ ] Task 8: Secrets audit — verify no keys printed on startup in `config.go`; CI pulls from env, not files
- [ ] Task 9: DB backup script — `pg_dump` + upload to private B2 bucket via rclone; cron container
- [ ] Task 10: Multi-arch CI build — `docker buildx` in `.woodpecker.yml` for `linux/amd64,linux/arm64`
- [ ] Task 11: Deep health check — `SELECT 1` on DB + `HeadBucket` on B2 in `handlers/health.go`; 503 on failure
- [ ] Task 12: Graceful shutdown — `http.Server.Shutdown(ctx)` with 5s timeout on SIGINT/SIGTERM

### Stage 10 — Legal Infrastructure & Dispute Management
- [ ] Task 1: `/terms` and `/privacy` public routes; links in all public page footers
- [ ] Task 2: Terms agreement checkbox on register; add `AgreedToTermsAt` + `TermsVersion` to User model
- [ ] Task 3: Mandatory quality confirmation checkbox in "Release Payment" modal (final milestone)
- [ ] Task 4: Standard Service Agreement text in backend config; link on every public proposal page
- [ ] Task 5: `models.Dispute` — `milestone_id`, `initiator_id`, `reason`, `status`, `resolution_type`, `admin_id`
- [ ] Task 6: Dispute Center UI — evidence upload (max 3 files) accessible from milestones tab
- [ ] Task 7: Dispute review fee — Dodo checkout session required before `Dispute` record is created
- [ ] Task 8: `DELETE /users/me` — soft-delete user + all projects + trigger B2 file deletion for all keys
- [ ] Task 9: IP transfer note on final milestone "Pay" CTA (usage rights language)
- [ ] Task 10: Privacy Policy sub-processor section (Backblaze, Dodo, Resend, Twilio/WhatsApp provider)
- [ ] Task 11: Data minimization audit — remove any non-essential PII fields from User model

### Stage 11 — Platform Observability & Trust Signals
- [ ] Task 1: `@sentry/react` in `phto-ui` — init in `main.tsx`, environment tag, user feedback on crash
- [ ] Task 2: `sentry-go` in `phto-api` — init in `main.go`, Chi middleware for panics + 500s
- [ ] Task 3: Public status page on BetterStack/Cronitor for API + Frontend; link in app footer
- [ ] Task 4: `IsVerified` (bool) + `VerifiedAt` (timestamp) on `CreatorProfile`; include in profile API response
- [ ] Task 5: `VerifiedBadge.tsx` — shield-check icon with tooltip on creator profile + proposal pages
- [ ] Task 6: `GET /public/stats` — cached aggregate: total escrow volume, files delivered, active creators
- [ ] Task 7: "Member since" + completed project count in a "Trust Box" on public creator profile
- [ ] Task 8: Media processing health monitor — alert if any media stuck in "Processing" > 30 minutes
- [ ] Task 9: `models.AuditLog` — IP + User Agent on login; Security tab in Settings showing login history
- [ ] Task 10: "Escrow Explained" popover next to every milestone Pay button (3-step how-it-works)
- [ ] Task 11: Floating feedback/bug report widget (Doorbell.io or HubSpot free)
- [ ] Task 12: `MAINTENANCE_MODE` env var — Chi middleware returning `503` with "We'll be back" when active

### Stage 12 — Creator Productivity (AI & Roadmap)
- [ ] Task 1: `internal/services/ai_vision.go` — `AnalyzeImage()` via Google Vision / GPT-4o-mini; store tags in `media.metadata` [skipped - AI]
- [ ] Task 2: Perceptual hash duplicate detection — `group_id` on media table; API returns grouped stacks [skipped - AI]
- [x] Task 3: Creator culling view UI — fullscreen, keyboard nav (←/→, P=pick, X=reject), filter mode
- [x] Task 4: Client favorites workflow — `is_favorite` on media; heart in SharedGallery; PATCH /media/:id/favorite
- [x] Task 5: `POST /projects/:id/archive` — `is_archived` flag; `POST /projects/:id/restore`; disable high-res when archived
- [ ] Task 6: Smart album auto-generation worker — create system album if >5 media share an AI tag [skipped - AI]
- [x] Task 7: `models.ProjectTemplate` — milestone + questionnaire structure; CRUD API + TemplateSettings.tsx
- [ ] Task 8: Blur/blink detection on thumbnails (local phash/opencv) — "Warning" flag in culling view [skipped - AI]
- [x] Task 9: "Forever Gallery" upsell — $25 one-time Dodo payment sets `expires_at +10yr`, marks project Protected
- [x] Task 10: Color palette extraction from uploads — store top 5 hex codes in project metadata; display in project list
- [x] Task 11: Soft-delete trash for projects — `deleted_at` on Project; 30-day grace; Trash tab in Projects page
- [ ] Task 12: Smart search in project detail — filter media by AI tag or color palette [skipped - AI-dependent]

---

## Session Log

| Date | Stage | Notes |
|------|-------|-------|
| 2026-03-17 | Stage 1 | All 12 tasks complete. CheckoutGateway interface + Dodo impl, GatewayRouter, fee calc, ledger, webhook, manual funding flow, payout settings UI. |
| 2026-03-17 | Stage 2 | All 11 tasks complete. None-type escrow hold, release atomicity w/ payout ledger, 7-day auto-release job, proposal math validation, deletion lock, sequence enforcement, admin dispute resolve API, MilestoneFeedback history, delivery lock on submit, refund stub, frontend quality checkbox. |
| 2026-03-17 | Stage 3 | All 11 tasks complete. AccessToken on Proposal + public endpoints, OTP guest auth (VerificationCode model, request-otp, verify-otp, guest JWT), EscrowTransaction nullable PayerID + GuestEmail, IsReadyForHandover + can_be_claimed, ClaimService (ownership transfer + My Deliveries provision), PublicProposal OTP modal + accept flow, ClaimProjectBanner, copy-link button. Task 9 was already complete from Stage 2. |
| 2026-03-17 | Stage 4 | All 12 tasks complete. SubscriptionTier/ID/ExpiresAt on User model, GetTierConfig utility, FeatureGateService (CanCreateProject + HasStorageSpace) + 402 gate on project Create, dynamic tier-aware fee in ReleaseMilestonePayment, POST /billing/subscribe Dodo checkout, webhook subscription.created/renewed/cancelled events, UpgradeModal, tier badge in Sidebar, Pro gate on branding fields, PATCH /admin/users/:id/tier, DowngradeExpiredSubscriptions cleanup job. |
| 2026-03-17 | Stage 6 | All 11 tasks complete. Scheduler (tasks 1-2 already existed), WhatsApp wired to InquiryService + send on new inquiry, WhatsApp send on milestone funded, Questionnaire model+service+handler+routes (GET/PUT /projects/:id/questionnaire + public responses endpoint), CRMService (proposal reminder 48h + post-delivery feedback 7d workers registered in scheduler), notification settings routes in router, NotificationSettings.tsx page (WhatsApp number + 3×2 checkbox grid), ActivityFeed.tsx dashboard widget, QuestionnaireBuilder.tsx in proposal detail, client questionnaire form on PublicProposal.tsx. |
| 2026-03-17 | Stage 7 | 10/12 tasks complete (skipped T8 docs, T11 FFmpeg). B2 key hierarchy (users/{uid}/projects/{pid}/media/{mid}/{filename}), upload-url alias route, direct-to-B2 frontend refactor (initiate→PUT→complete with multipart fallback), watermark service (ApplyWatermark → _web_preview variant), conditional URL logic (non-owners see web_preview), CDN base URL config (MEDIA_CDN_URL env + mediasign.NewWithBase), EXIF extraction via goexif → media.metadata column, StreamProjectZip streaming handler, Cache-Control 24h TTL on signer. |
| 2026-03-18 | Stage 12 | 7/12 tasks complete (AI tasks T1/2/6/8/12 skipped). CullStatus+IsFavorite on MediaObject, culling view UI (fullscreen keyboard nav), client favorites in SharedGallery, project archive/restore, project soft-delete trash (30d purge job), ProjectTemplate CRUD (service+handler+TemplateSettings.tsx), Forever Gallery $25 Dodo checkout, color palette extraction (algorithmic, stores top-5 hex in project), Trash tab in Projects page. |
