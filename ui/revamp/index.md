we are doing a massive revamp of the application. make sure you use this index.md as a hub for managing the current state of the revamping work. this is to go into multiple sessions. so clearly update the document with what you have done, what was left behind, what to do next after each checkpoint so we can continue without hiccups or going through the whole lot to figure out where are we.

in this directory [.../revamp] we have a play book with 12 plays. we should carefully plan the implemenation of the this play book.

importantly you should follow the style-guide.md and code-guide.md for best practices.

document your findings that will be needed in general so other sessions do not have to waste time and tokens redoing all that

commit the changes you do

---

## Progress Tracker

### Project Initialization ✅ COMPLETE (2026-03-14)

**Branch:** `main`

**What was done:**
- Initialized new Vite project with React + TypeScript in `zeyaroo-ui`.
- Re-implemented Phase 0 (Design System), Phase 1 (Value Producers), and Phase 2 (Creator Workspace) based on the plan.
- All core pages (`Dashboard`, `ProjectWorkspace`, `PublicProposal`, `SharedGallery`) are functional with mock APIs.
- Established theme architecture (`gallery-white` / `obsidian`) and semantic token system.

---

### Phase 0 — Design System Foundation ✅ COMPLETE (2026-03-14)


**Branch:** `revamp/phase-0-design-system`

**What was done:**
- Replaced all `index.css` with a `data-theme` attribute token system (`gallery-white` + `obsidian`)
- All theming is now CSS-variable only — zero component code changes needed to switch or add themes
- `tailwind.config.js` updated with semantic tokens: `surface-*`, `text-*`, `border-*`, `accent-*`, `status-*`, `gold`
- Typography: Instrument Serif (`font-serif`) + Inter (`font-sans`) via Google Fonts in `index.html`
- Custom radii: `rounded-editorial` (0px, client buttons) + `rounded-workspace` (12px, creator buttons)
- Created `ThemeProvider.tsx` — thin wrapper that sets `data-theme` on its container
- `Layout.tsx` now wrapped in `<ThemeProvider theme="obsidian">` (creator workspace = dark)
- `Button.tsx`: added `context` prop (`editorial` | `workspace`), all tokens updated to semantic
- `Card.tsx`: updated to semantic tokens
- `Input.tsx`: updated to semantic tokens (label is now uppercase tracking-widest per style-guide)
- Installed: `framer-motion`, `@dnd-kit/core`, `@dnd-kit/sortable`, `@dnd-kit/utilities`
- TypeScript: clean (0 errors)

**Known pending (by design):**
- 31 existing page/component files still use old CSS token names (e.g., `bg-surface-950`, `text-text-primary`). These will be fixed page-by-page as each play is implemented.

---

### Phase 1 — Value Producers ✅ COMPLETE (2026-03-14)

**Branch:** `revamp/phase-1-value-producers`

**Play 3 — Digital Invitation** (`/p/:token`) ✅
- New `src/pages/PublicProposal.tsx` — full editorial page
- `CinematicHero` with Framer Motion parallax scroll (70vh, cover image with `useScroll`/`useTransform`)
- `CreatorIntro` — centered avatar + italic serif quote, `whileInView` fade-in
- `DeliverablesSection` — wide cards with newline-split deliverables, staggered entrance
- `VerticalMilestonePath` — timeline with escrow protection tooltips (AnimatePresence expand)
- `ServiceAgreement` — scrollable text + checkbox, Accept locked until agreed
- `StickyProposalAction` — glassmorphic bar, appears after hero scrolls out (IntersectionObserver), full-width on mobile
- New `PublicProposalData` type added to `src/types/index.ts`
- New `usePublicProposal` hook added to `src/api/proposals.ts` (calls `/public/proposals/:token` — backend endpoint TODO)
- Route added to `App.tsx`: `<Route path="/p/:token" element={<PublicProposalPage />} />`

**Play 4 — Shared Gallery** (`/gallery/:token`) ✅
- `SharedGallery.tsx` fully rewritten
- `ThemeProvider theme="gallery-white"` wrap throughout
- `FloatingGalleryNav` — fixed glass pill (top-6), serif project name + item count
- CSS masonry grid: `columns-1 sm:columns-2 lg:columns-3 xl:columns-4`, natural aspect ratios
- Framer Motion per-item stagger: `delay: Math.min((index % 24) * 0.04, 0.6)`
- `WatermarkBadge` — hover-revealed PREVIEW pill on watermarked media
- Improved `Lightbox` — horizontal slide animation (`AnimatePresence` + `custom` direction), scroll position restore on close
- Redesigned `PasswordGate` — gallery-white card, editorial button
- Auto-auth for non-password shares (calls verify with empty password)
- All old token names updated to semantic tokens (`surface-primary`, `text-main`, etc.)
- `GallerySkeleton` with shimmer placeholders

**Known pending (by design):**
- `/public/proposals/:token` backend endpoint does not exist yet — PublicProposal page shows error state until implemented
- "Accept & Pay Deposit" button action not wired (DodoPayments integration = future play)
- "Download All" not in gallery nav (requires milestone payment status check)

---

### Phase 2 — Creator Workspace ✅ COMPLETE (2026-03-14)

**Branch:** `revamp/phase-2-creator-workspace`

**Play 1 — Creator Command Center** (`/dashboard`) ✅
- `Dashboard.tsx` fully rewritten — removed old `StatCard`, `ProjectCard`, quick-actions section
[chatura@archlinux plan]$ cat index.md 
we are doing a massive revamp of the application. make sure you use this index.md as a hub for managing the current state of the revamping work. this is to go into multiple sessions. so clearly update the document with what you have done, what was left behind, what to do next after each checkpoint so we can continue without hiccups or going through the whole lot to figure out where are we.

in this directory [.../revamp] we have a play book with 12 plays. we should carefully plan the implemenation of the this play book.

importantly you should follow the style-guide.md and code-guide.md for best practices.

document your findings that will be needed in general so other sessions do not have to waste time and tokens redoing all that

commit the changes you do

---

### Project Initialization ✅ COMPLETE (2026-03-14)

**Branch:** `main`

**What was done:**
- Initialized new Vite project with React + TypeScript in `zeyaroo-ui`.
- Re-implemented Phase 0 (Design System), Phase 1 (Value Producers), and Phase 2 (Creator Workspace) based on the plan.
- All core pages (`Dashboard`, `ProjectWorkspace`, `PublicProposal`, `SharedGallery`) are functional with mock APIs.
- Established theme architecture (`gallery-white` / `obsidian`) and semantic token system.

---

### Phase 0 — Design System Foundation ✅ COMPLETE (2026-03-14)


**Branch:** `revamp/phase-0-design-system`

**What was done:**
- Replaced all `index.css` with a `data-theme` attribute token system (`gallery-white` + `obsidian`)
- All theming is now CSS-variable only — zero component code changes needed to switch or add themes
- `tailwind.config.js` updated with semantic tokens: `surface-*`, `text-*`, `border-*`, `accent-*`, `status-*`, `gold`
- Typography: Instrument Serif (`font-serif`) + Inter (`font-sans`) via Google Fonts in `index.html`
- Custom radii: `rounded-editorial` (0px, client buttons) + `rounded-workspace` (12px, creator buttons)
- Created `ThemeProvider.tsx` — thin wrapper that sets `data-theme` on its container
- `Layout.tsx` now wrapped in `<ThemeProvider theme="obsidian">` (creator workspace = dark)
- `Button.tsx`: added `context` prop (`editorial` | `workspace`), all tokens updated to semantic
- `Card.tsx`: updated to semantic tokens
- `Input.tsx`: updated to semantic tokens (label is now uppercase tracking-widest per style-guide)
- Installed: `framer-motion`, `@dnd-kit/core`, `@dnd-kit/sortable`, `@dnd-kit/utilities`
- TypeScript: clean (0 errors)

**Known pending (by design):**
- 31 existing page/component files still use old CSS token names (e.g., `bg-surface-950`, `text-text-primary`). These will be fixed page-by-page as each play is implemented.

---

### Phase 1 — Value Producers ✅ COMPLETE (2026-03-14)

**Branch:** `revamp/phase-1-value-producers`

**Play 3 — Digital Invitation** (`/p/:token`) ✅
- New `src/pages/PublicProposal.tsx` — full editorial page
- `CinematicHero` with Framer Motion parallax scroll (70vh, cover image with `useScroll`/`useTransform`)
- `CreatorIntro` — centered avatar + italic serif quote, `whileInView` fade-in
- `DeliverablesSection` — wide cards with newline-split deliverables, staggered entrance
- `VerticalMilestonePath` — timeline with escrow protection tooltips (AnimatePresence expand)
- `ServiceAgreement` — scrollable text + checkbox, Accept locked until agreed
- `StickyProposalAction` — glassmorphic bar, appears after hero scrolls out (IntersectionObserver), full-width on mobile
- New `PublicProposalData` type added to `src/types/index.ts`
- New `usePublicProposal` hook added to `src/api/proposals.ts` (calls `/public/proposals/:token` — backend endpoint TODO)
- Route added to `App.tsx`: `<Route path="/p/:token" element={<PublicProposalPage />} />`

**Play 4 — Shared Gallery** (`/gallery/:token`) ✅
- `SharedGallery.tsx` fully rewritten
- `ThemeProvider theme="gallery-white"` wrap throughout
- `FloatingGalleryNav` — fixed glass pill (top-6), serif project name + item count
- CSS masonry grid: `columns-1 sm:columns-2 lg:columns-3 xl:columns-4`, natural aspect ratios
- Framer Motion per-item stagger: `delay: Math.min((index % 24) * 0.04, 0.6)`
- `WatermarkBadge` — hover-revealed PREVIEW pill on watermarked media
- Improved `Lightbox` — horizontal slide animation (`AnimatePresence` + `custom` direction), scroll position restore on close
- Redesigned `PasswordGate` — gallery-white card, editorial button
- Auto-auth for non-password shares (calls verify with empty password)
- All old token names updated to semantic tokens (`surface-primary`, `text-main`, etc.)
- `GallerySkeleton` with shimmer placeholders

**Known pending (by design):**
- `/public/proposals/:token` backend endpoint does not exist yet — PublicProposal page shows error state until implemented
- "Accept & Pay Deposit" button action not wired (DodoPayments integration = future play)
- "Download All" not in gallery nav (requires milestone payment status check)

---

### Phase 2 — Creator Workspace ✅ COMPLETE (2026-03-14)

**Branch:** `revamp/phase-2-creator-workspace`

**Play 1 — Creator Command Center** (`/dashboard`) ✅
- `Dashboard.tsx` fully rewritten — removed old `StatCard`, `ProjectCard`, quick-actions section
- `AnimatedMetricRing`: SVG ring using `animate()` from framer-motion; animates `strokeDashoffset` from 0 → target on mount (delay configurable)
  - Revenue ring: `var(--color-gold)` (Heritage Gold), mocked at 35%
  - Storage ring: `var(--accent-primary)` (white in obsidian)
- `PulseCard`: large centered number card for inquiry/action count, links to `/inquiries`
- `ProjectStack`: 3-layer Framer Motion card with `initial="idle" whileHover="fan"` variant propagation
  - Outer `div` handles CSS mount animation (`animate-slide-in-right`), inner `motion.div` handles fan variants
  - Cards 2 & 3 fan out (rotate + translate) on hover; top card slightly counter-rotates + scales
  - Rich dark gradient palettes per index slot
  - Status dot + serif project title overlay on top card
- `EmptyStack` silhouette: dashed border + serif italic quote per play1.md spec
- Greeting is time-of-day aware: `Good morning/afternoon/evening, [name].` in `font-serif text-5xl`
- Bottom row: Upcoming Milestones (placeholder panel) + Studio Activity (live inquiry/proposal counts)
- TypeScript: clean (0 errors)

**Play 2 — Project Workspace** (`/projects/:id`) ✅
- `ProjectDetail.tsx` fully rewritten — 3-pane layout: Left sidebar | Center stage | Right inspector
- **Left pane** (`w-52`, sticky): Upload button (accent-primary CTA), Smart Views, Album tree (inline new-album form), Project Storage janitor
- **Center pane**: sticky toolbar (mode pill + filter tag + file count + select-all), bulk action bar, CSS masonry grid (`columns: 3`, natural aspect ratios, `break-inside-avoid`), staggered `animate-scale-in` entrance
  - `inspectedMediaId` state: click = inspect, Shift/Ctrl+Click = multi-select into `selectedMediaIds` Set
  - Hover glass overlay + selection ring `ring-accent-primary` + download button
  - Drag-to-upload: whole center pane is the drop zone with a blur overlay
  - Empty state is a click-to-upload `<label>`
- **Right inspector** (`w-64`): slides in (`animate-slide-in-right`) when `inspectedMedia` is set; thumbnail preview + 5 metadata rows (Filename, Size, Type, Status, Added) + quick actions (Add to Album / Download / Delete)
- Milestones + Shares moved to header icon buttons; render as full-width panels when active tab is selected
- Client delivery view: CSS masonry matching creator grid style, watermark badge, download button
- All old tokens replaced (`surface-800` → `surface-muted`, `accent-400` → `accent-primary`, `font-display` → `font-serif`, etc.)
- TypeScript: clean (0 errors)

**Known pending (by design):**
- Revenue ring is mocked at 35% — wires up when billing API returns escrow totals
- Culling mode (Play 2 spec) stubbed — mode switcher shows "Grid" only; full implementation is a future play
- Keyboard shortcuts for culling (Arrow, Space, X, 1-5) not yet implemented

---

### Phase 3 — Client Experience ✅ COMPLETE (2026-03-16)

**Branch:** `main`

**Play 5 — Selection Room** (extends `/gallery/:token`) ✅
- `SharedGallery.tsx` extended with full selection mode — activates when `share.selection_mode === true`
- `SelectionProgressPill` — fixed top-24, glassmorphic pill showing `N / limit selected` + animated progress bar; turns amber at limit; turns green + check when submitted
- Heart toggle on every masonry item: outline heart on hover, filled heart + check badge when selected; ring-2 selection glow
- In selection mode, clicking the image directly toggles selection (not lightbox) — heart button is a secondary target
- `SelectionTray` — fixed bottom bar; desktop: horizontal thumbnail scroller + "Submit Final Selection" CTA; mobile: collapses to count pill + compact Submit button
- `SelectionDrawer` — mobile full-screen drawer showing 3-col grid of selected items with remove/submit
- `SubmitModal` — confirmation dialog before locking selection
- `QuotaToast` — amber pill toast (3s auto-dismiss) when client exceeds limit
- `ComparisonModal` — side-by-side full-screen overlay; select/deselect each image inline; triggered from Lightbox "Compare" button
- Lightbox extended: "Select" toggle button + "Compare" icon in toolbar when in selection mode; compare hint text when first image queued
- `useSubmitSelection` + `useSaveSelection` mutations added to `src/api/shares.ts`
- `usePublicShare` return type extended with optional `selection_mode`, `selection_limit`, `selection_is_submitted`
- TypeScript: clean (0 errors)

**Known pending (by design):**
- Backend endpoints `POST /public/shares/:token/selection` and `/submit` do not exist yet — UI gracefully degrades (selection still marks locally as submitted)
- "Flight animation" (thumbnail flies into tray on heart tap) — spec's stretch goal, not implemented

**Play 6 — The Vault** (`/vault`) ✅
- New `src/pages/Vault.tsx` — client dashboard, `gallery-white` theme, outside Layout wrapper
- `VaultHeader` — serif greeting with time-of-day (`Good morning/afternoon/evening, [name].`) + "X life events secured · Y GB preserved" summary
- `VaultCard` — `aspect-[4/5]` luxury tile with cover gradient, status badge (`Lifetime Access` / `Expires soon`), amber pulse ring for expiring projects, "Save Forever" CTA on expiring cards, hover lift + shadow
- `FeaturedHero` — single-project layout: 70vh tall hero card (full-bleed), serif title, "Open Gallery" pill button
- `EmptyVault` — serif centered empty state with sparkle icon
- `ClaimSuccessModal` — one-time modal (triggered via `?claimed=1` URL param + sessionStorage gate); staggered sparkle animation + "Your memories are now safe." headline
- `VaultSkeleton` — shimmer skeleton matching header + card grid
- "Discover Creators" / "Hire Again" bottom section (placeholder tiles from project list; links to `/creators`)
- Route `/vault` added to `App.tsx` (protected, outside Layout)
- Exported from `src/pages/index.ts`
- TypeScript: clean (0 errors)

**Known pending (by design):**
- `VaultCard` cover image: shows gradient placeholder until backend provides `cover_image_url` on projects
- "Discover Creators" section pulls creator profiles by `owner_id` — backend endpoint needed to fetch multiple creators by ID
- Project expiry date: heuristic only (event date > 1 year old); real expiry needs backend `expires_at` on Project model
- `ClaimSuccessModal` confetti burst (play6.md stretch) not implemented

---

---

### Phase 4 — Creator Tools ✅ COMPLETE (2026-03-16)

**Branch:** `main`

**Play 7 — Proposal Builder** (`/proposals/:id`) ✅
- `ProposalDetail.tsx` fully rewritten — two modes: Builder (3-col) and Viewer
- **Builder mode** (creator + `canCreateVersion`):
  - Sticky `TotalWatcherBar` — contract total input, "Math Mismatch" amber warning (milestones sum ≠ total), "Send Proposal" CTA
  - Left pane (`w-72`): `LogisticsPane` — client info card (email + event date + location), deliverables textarea, usage rights, revision rounds, start/end date pickers, notes
  - Center pane (flex-1): `MilestoneCanvas` — `SortableMilestoneCard` with `@dnd-kit/sortable` drag handles, title/amount/description/deadline/deliverable-type inline editing, "Add a Phase" dashed-border CTA (max 10), templates dropdown (Wedding / Studio)
  - Right pane (`w-80`, xl+): `MiniProposalPreview` — live client view at 55% scale; gradient cover placeholder, milestone list, total bar; real-time updates as creator types
  - `UnsavedModal` — "Your masterpiece isn't finished. Save as draft?" on back attempt with dirty state
  - Auto-activates builder mode when `status === 'inquiry'` or `status === 'pending'` for creator
- **Viewer mode** (client, finalized, cancelled, or creator reviewing):
  - Status badge + header actions (Accept / Counter-Propose / Cancel / Create Project / View Project)
  - `VersionCard` — semantic tokens, milestones list, deliverables, notes; "Latest" badge on most recent
  - Collapsible version history with Framer Motion height animation
- TypeScript: clean (0 errors)

**Play 8 — Inquiry & Lead Manager** (`/inquiries`) ✅
- `Inquiries.tsx` fully rewritten — Kanban pipeline view + table fallback
- `Command header`: page title + active lead count + proposals-sent count + search bar + view switcher (Pipeline / List)
- **Pipeline view** (default): 4 columns rendered as `DroppableColumn` wrappers
  - `new` (Inbox) / `qualifying` / `proposal_sent` / `booking`
  - Column grouping derived from `inquiry.proposal?.status`
  - `DraggableLeadCard` — uses `useDraggable` + `DragOverlay` for visual ghost on drag
  - `LeadCard` — font-serif title, event date tags (italic "TBD" if no date), location/budget tags, amber "Needs attention" heat indicator (>24h in new/qualifying), green pulse dot (needs action from viewer)
  - `DroppableColumn` — `useDroppable`; highlights with `bg-accent-primary/5` on drag-over
  - `DragOverlay` — rotated + scaled ghost card for premium drag feel
  - Local state (`pipelineState`) tracks drag moves — TODO: persist via `PATCH /api/v1/inquiries/:id/pipeline_status`
- **List view**: `TableView` — animated row list with status badge pills
- `InquiryDetailsDrawer` — Framer Motion slide-in from right; full message, meta tags, date (or "Date TBD" serif italic), client email, proposal status, "View/Send Proposal" CTA → navigates to `/proposals/:id`
- `EmptyState`: serif "Your inbox is a blank canvas. Share your profile link…"
- TypeScript: clean (0 errors)

**Known pending (by design):**
- `PATCH /api/v1/inquiries/:id/pipeline_status` — backend endpoint needed for drag persistence across sessions
- Dragging a lead to "Proposal Sent" column doesn't auto-open builder (Play 8 spec stretch goal)

---

### Phase 5 — Marketplace & Revenue ✅ COMPLETE (2026-03-16)

**Branch:** `main`

**Play 9 — Billing & Revenue Dashboard** (`/billing`) ✅
- `Billing.tsx` fully rewritten — Revenue Dashboard (replaces old tab-based billing page)
- `FinancialSummary` top row: 3 serif count-up blocks (Total Value Secured, Available for Payout, Net Lifetime Earnings) using `animate()` from framer-motion on `useRef<HTMLSpanElement>`
- `SubscriptionCard` (4/12 cols): Studio tier amber badge, next billing date, animated storage meter
- `PayoutMethodCard` (8/12 cols): payout method status, last payout info, animated balance count-down on "Request Payout" click
- `TransactionLedger`: emerald circle for inflows, gray for payouts; hover tooltip on inflow rows shows gross/fee/net breakdown; serif italic empty state
- Uses `usePayments` for live data; falls back to `MOCK_TRANSACTIONS` when empty/loading
- `/billing` route added to `App.tsx` (ProtectedRoute inside Layout); exported as `BillingPage` from `pages/index.ts`

**Known pending (by design):**
- Backend `GET /billing/summary` does not exist — FinancialSummary uses mock data
- "Request Payout" is simulated locally (no real payout API call)
- Subscription plan management / "Change Plan" modal not implemented (future play)

**Play 10 + 12 — Marketplace & Creator Discovery** (`/creators`) ✅
- `Creators.tsx` fully rewritten — magazine-rhythm discovery page
- Atmospheric header: centered `font-serif text-5xl` tagline "Find the eye that sees your story.", pill search bar with `bg-white/50 backdrop-blur-xl`, quick-filter pills (Wedding/Editorial/Film/Commercial)
- `ArtistTile`: `aspect-[3/4] rounded-[2.5rem]` with `group-hover:scale-110 duration-[1500ms]`, bottom gradient overlay, serif name + uppercase category label, eye icon "Quick View" button (hover reveal)
- Featured tiles: every `index % 5 === 2` → `col-span-2 row-span-2` in `grid-flow-dense` 4-col grid; collapses to 1-col on mobile
- `ContextDrawer`: `AnimatePresence` spring slide-in from right, shows avatar/name/bio/categories, "View Profile" + "Send Inquiry" links
- Scroll reveal: `whileInView={{ opacity: 1, y: 0 }}` with `once: true` per tile
- Debounced search (300ms); category filtering client-side on loaded results
- Empty state: serif "No exact matches found, but these artists caught our eye."
- `setInterval` ref wired for future slideshow on hover (clears on mouse leave)

**Play 11 — Public Portfolio / Link-in-Bio** (`/creators/:id`) ✅
- `CreatorProfile.tsx` fully rewritten — gallery-white editorial public profile
- `PortfolioHeader`: centered layout, large circular avatar, `font-serif text-5xl` name, verified shield, bio, social icons cluster
- `ServiceTags`: horizontal scrolling pill row (scrollbar hidden)
- Portfolio grid: CSS masonry (`columns-1 sm:columns-2 lg:columns-3 xl:columns-4`), hidden when < 3 images
- `Lightbox`: `AnimatePresence` + `motion.div` fade/scale, per-image keyed `motion.img` for slide transitions
- Trust footer: "Member since · X Portfolio Items · Available for bookings"
- `StickyInquiryBar` (mobile `md:hidden`): `fixed bottom-0`, `bg-white/70 backdrop-blur-xl`
- Floating desktop card (`hidden md:block fixed right-8 top-28`): summary + CTA
- `PublicInquiryModal`: 3-step `AnimatePresence mode="wait"` flow (Event Type → Date → Contact), progress dots, spring-animated sheet entry

**Known pending (by design):**
- `CreatorProfile` type has no `accepting_inquiries` or `location` fields — floating card shows "Available for bookings" as static copy; trust footer omits "Based in..."
- `useSendCreatorInquiry` mutation called but `/creators/:id/inquire` endpoint needs to be public (no auth header required for guest users)

---

### ALL PLAYS COMPLETE ✅

**All 12 plays implemented.** Final state:
- `BillingSettings.tsx` — not revamped (separate settings page, lower priority)
- Token reference: `src/index.css` (CSS vars) + `tailwind.config.js` (Tailwind aliases)
- Layout.tsx uses `gallery-white` theme (NOT obsidian — this changed from previous sessions)
- Font conventions: `font-serif` (Instrument Serif) for headings, `font-sans` (Inter) for UI
- Button radii: `rounded-editorial` (0px, client) / `rounded-workspace` (12px, creator)