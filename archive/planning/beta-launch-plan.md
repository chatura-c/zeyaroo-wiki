# Zeyaroo Beta Launch — Implementation Plan

Work is split into 8 sessions, ordered by priority. Each session is independent
and can be completed in a single sitting. Sessions 1–4 are **blockers** for beta.
Sessions 5–6 are **important for quality**. Sessions 7–8 can happen post-launch.

> **For Claude:** When completing a session, log any steps that require manual
> action outside the codebase (Firebase Console, DNS, cloud dashboards, etc.)
> to [`MANUAL_TODO.md`](./MANUAL_TODO.md). Keep entries brief — one bullet per
> step, grouped under a session heading.

---

## Session 1 — Toast System + Error Feedback ✅

**Goal:** Replace all silent `console.error` catches with user-visible toast
notifications so users know when actions succeed or fail.

**Estimated scope:** ~15 files touched, 1 new dependency

### Steps

1. **Install sonner** (lightweight, unstyled-friendly toast library):
   ```bash
   cd phto-ui && npm install sonner
   ```

2. **Add `<Toaster>` to App.tsx** — place it inside `<BrowserRouter>` so it's
   available everywhere:
   ```tsx
   // phto-ui/src/App.tsx
   import { Toaster } from 'sonner';
   // Inside the return, after <BrowserRouter>:
   <Toaster
     position="bottom-right"
     toastOptions={{
       className: 'bg-surface-card border border-border-main text-text-main',
     }}
   />
   ```

3. **Replace `console.error` catches in ProjectDetail.tsx** — this is the worst
   offender with 10+ silent catches. For each catch block at lines 126, 132,
   141, 147, 184, 220, 228, 268, 279, 288, 298, 304, 317:
   ```tsx
   import { toast } from 'sonner';
   // Replace: catch (e) { console.error(e); }
   // With:    catch (e) { toast.error('Failed to delete album'); }
   ```
   Use descriptive messages per action (delete album, create album, upload,
   remove media, set cover, etc.).

4. **Replace `console.error` catches in these files** — same pattern:
   - `pages/Projects.tsx` — lines 314, 325, 336, 346 (create, delete, trash,
     restore)
   - `pages/ProposalBuilder.tsx` — line 840 (send proposal)
   - `pages/ProposalDetail.tsx` — lines 137, 142, 149 (accept, cancel, create
     project)
   - `pages/settings/StorageSettings.tsx` — lines 94, 107, 117 (subscribe,
     switch mode, adjust quota)
   - `pages/settings/BillingSettings.tsx` — line 78 (pay invoice)
   - `components/ShareManager.tsx` — line 45 (delete share)
   - `components/TransferRequestsNotification.tsx` — lines 24, 32 (accept/
     decline transfer)
   - `pages/PublicProposal.tsx` — lines 396, 426

5. **Add success toasts for key mutations** — not every mutation, just the ones
   where users need confirmation:
   - Project created/deleted/trashed/restored
   - Proposal sent/accepted/cancelled
   - Album created/deleted
   - Share created/deleted
   - Settings saved (profile, payout, storage, notifications)
   - Media uploaded (summary: "3 files uploaded")
   - Media deleted

6. **SharedGallery submit catch** — the comment says "Backend endpoint may not
   exist yet" — verify the endpoint exists, then replace the silent catch with a
   toast.

### Verification
- Test each action's success/error state manually
- Temporarily break an API call to verify error toasts appear

---

## Session 2 — Auth Loading State + Console Cleanup ✅

**Goal:** Fix the flash-of-login-redirect on page refresh, and remove all
`console.log` statements from production code paths.

### Part A: Auth Loading State

The problem: `ProtectedRoute` reads `isAuthenticated` from the Zustand store
(persisted to localStorage, rehydrates synchronously). But on hard refresh,
Firebase's `onAuthStateChanged` fires async and may briefly set
`isAuthenticated = false` → `true`, causing a visible redirect flash.

**Files:**
- `phto-ui/src/components/ProtectedRoute.tsx`
- `phto-ui/src/contexts/FirebaseAuthContext.tsx`

1. **Export `loading` from FirebaseAuthContext** (already exposed via context):
   ```tsx
   // Already exists: useFirebaseAuthContext() returns { loading, error }
   ```

2. **Update ProtectedRoute to wait for auth initialization:**
   ```tsx
   // phto-ui/src/components/ProtectedRoute.tsx
   import { useFirebaseAuthContext } from '../contexts/FirebaseAuthContext';

   export function ProtectedRoute({ children, requiredRole }: ProtectedRouteProps) {
     const { isAuthenticated, user } = useAuthStore();
     const { loading } = useFirebaseAuthContext();
     const location = useLocation();

     // Wait for Firebase to finish initializing before redirecting
     if (loading) {
       return (
         <div className="min-h-screen flex items-center justify-center bg-surface-primary">
           <div className="w-6 h-6 border-2 border-gold/30 border-t-gold rounded-full animate-spin" />
         </div>
       );
     }

     if (!isAuthenticated) {
       return <Navigate to="/login" state={{ from: location }} replace />;
     }

     if (requiredRole && user?.role !== requiredRole) {
       return <Navigate to="/dashboard" replace />;
     }

     return <>{children}</>;
   }
   ```

### Part B: Remove Console Statements

3. **Remove `console.log` from production code paths:**

   | File | Line | What | Action |
   |---|---|---|---|
   | `contexts/FirebaseAuthContext.tsx` | 91 | `console.log('Firebase token refreshed')` | Delete the line |
   | `config/app.ts` | 61–66 | Logs full Firebase config with API keys | Delete or gate behind `import.meta.env.DEV && ...` |
   | `config/firebase.ts` | 19 | `console.log('Connected to Firebase Auth Emulator')` | Keep — only fires in emulator mode |
   | `pages/settings/BillingSettings.tsx` | 202 | `console.log('View details for invoice:', ...)` | Replace with navigation or toast (see Session 6) |

4. **Backend: Remove DSN logging from main.go** — line 56 logs the full
   database DSN (contains credentials):
   ```go
   // phto-api/cmd/server/main.go — DELETE these lines:
   log.Printf("[STARTUP] ENV DB_DSN=%s", os.Getenv("DB_DSN"))
   log.Printf("[STARTUP] DB DSN: %s", cfg.Database.DSN)
   ```
   Keep the driver and storage type logs — those are safe.

### Verification
- Hard-refresh a protected page → should see spinner, then content (no flash to
  login)
- Check browser console for stray `console.log` on token refresh (should be
  gone)
- Check server logs on startup for DSN (should be gone)

---

## Session 3 — Rate Limiting + Graceful Shutdown

**Goal:** Protect auth endpoints from brute force and ensure clean deploys.

### Part A: Rate Limiting

**Files to create/modify:**
- `phto-api/internal/middleware/ratelimit.go` (new)
- `phto-api/internal/handlers/routes.go`

1. **Create a simple IP-based rate limiter middleware** using
   `golang.org/x/time/rate` (already an indirect dependency via chi):

   ```go
   // phto-api/internal/middleware/ratelimit.go
   package middleware

   import (
       "net/http"
       "sync"
       "time"
       "golang.org/x/time/rate"
   )

   type IPRateLimiter struct {
       visitors map[string]*visitorEntry
       mu       sync.Mutex
       rate     rate.Limit
       burst    int
   }

   type visitorEntry struct {
       limiter  *rate.Limiter
       lastSeen time.Time
   }

   func NewIPRateLimiter(r rate.Limit, burst int) *IPRateLimiter {
       rl := &IPRateLimiter{
           visitors: make(map[string]*visitorEntry),
           rate:     r,
           burst:    burst,
       }
       go rl.cleanup()
       return rl
   }

   func (rl *IPRateLimiter) getVisitor(ip string) *rate.Limiter {
       rl.mu.Lock()
       defer rl.mu.Unlock()
       v, exists := rl.visitors[ip]
       if !exists {
           limiter := rate.NewLimiter(rl.rate, rl.burst)
           rl.visitors[ip] = &visitorEntry{limiter: limiter, lastSeen: time.Now()}
           return limiter
       }
       v.lastSeen = time.Now()
       return v.limiter
   }

   func (rl *IPRateLimiter) cleanup() {
       for {
           time.Sleep(5 * time.Minute)
           rl.mu.Lock()
           for ip, v := range rl.visitors {
               if time.Since(v.lastSeen) > 10*time.Minute {
                   delete(rl.visitors, ip)
               }
           }
           rl.mu.Unlock()
       }
   }

   func (rl *IPRateLimiter) Limit(next http.Handler) http.Handler {
       return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
           ip := r.RemoteAddr
           if fwd := r.Header.Get("X-Forwarded-For"); fwd != "" {
               ip = strings.Split(fwd, ",")[0]
           }
           if !rl.getVisitor(strings.TrimSpace(ip)).Allow() {
               http.Error(w, `{"error":"rate limit exceeded"}`, http.StatusTooManyRequests)
               return
           }
           next.ServeHTTP(w, r)
       })
   }
   ```

2. **Apply rate limiter to auth routes** in `routes.go`:
   ```go
   // In registerRoutes(), wrap the /auth group:
   authLimiter := middleware.NewIPRateLimiter(rate.Every(time.Second), 10) // 10 req/s burst
   otpLimiter  := middleware.NewIPRateLimiter(rate.Every(2*time.Second), 5) // 5 req/2s

   r.Route("/auth", func(r chi.Router) {
       r.Use(authLimiter.Limit)
       // ... existing auth routes
   })
   ```
   Also apply `otpLimiter` to:
   - `/auth/guest/request-otp`
   - `/auth/guest/verify-otp`
   - `/public/proposals/{token}/otp/request`
   - `/public/proposals/{token}/otp/verify`

3. **Apply rate limiter to public payment/claim endpoints:**
   - `/projects/{id}/claim` — 5 req/min
   - `/public/pay/{token}/*` — 10 req/min

### Part B: Graceful Shutdown

**File:** `phto-api/cmd/server/main.go`

4. **Replace `http.ListenAndServe` with graceful shutdown pattern:**
   ```go
   // Replace lines 200-205 with:
   srv := &http.Server{
       Addr:    addr,
       Handler: router,
   }

   // Start server in goroutine
   go func() {
       log.Printf("Server starting on %s", addr)
       if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
           log.Fatalf("Server failed to start: %v", err)
       }
   }()

   // Wait for interrupt signal
   quit := make(chan os.Signal, 1)
   signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
   <-quit
   log.Println("Shutting down server...")

   // Give active requests 15 seconds to complete
   ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
   defer cancel()

   // Cancel background processors first
   processorCancel()

   if err := srv.Shutdown(ctx); err != nil {
       log.Fatalf("Server forced to shutdown: %v", err)
   }
   log.Println("Server exited")
   ```
   Add imports: `"os/signal"`, `"syscall"`.
   Move `processorCancel` declaration earlier (before the `defer`), or restructure
   so it's accessible in the shutdown block.

### Verification
- Test rate limiting: `for i in $(seq 1 20); do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:PORT/api/v1/auth/login -X POST; done`
  — should see 429 after burst is exceeded
- Test graceful shutdown: start server, make a long request, send SIGTERM —
  request should complete before server exits

---

## Session 4 — Legal + Security Essentials

**Goal:** Add ToS/Privacy pages, fix the 404 page, and configure Firebase
password reset action URL.

### Part A: Terms of Service + Privacy Policy Pages

1. **Create a simple legal pages component** — can be placeholder content for
   beta, but the pages must exist:

   ```tsx
   // phto-ui/src/pages/Legal.tsx
   export function TermsPage() { ... }
   export function PrivacyPage() { ... }
   ```
   Use a simple centered layout with the branding header. Content can be minimal
   placeholder text for beta: "These terms govern your use of Zeyaroo..." etc.
   Mark them clearly as beta/draft.

2. **Add routes in App.tsx:**
   ```tsx
   <Route path="/terms" element={<TermsPage />} />
   <Route path="/privacy" element={<PrivacyPage />} />
   ```

3. **Add links to auth pages** — in `Login.tsx` and `Register.tsx`, add a
   footer line:
   ```tsx
   <p className="text-xs text-text-muted text-center mt-4">
     By continuing, you agree to our{' '}
     <Link to="/terms" className="underline">Terms of Service</Link> and{' '}
     <Link to="/privacy" className="underline">Privacy Policy</Link>.
   </p>
   ```

### Part B: 404 Page

4. **Style the 404 page** in `App.tsx` to use design system tokens:
   ```tsx
   <Route path="*" element={
     <div className="min-h-screen flex items-center justify-center bg-surface-primary">
       <div className="text-center">
         <h1 className="text-4xl font-bold text-text-main">404</h1>
         <p className="text-text-muted mt-2">Page not found</p>
         <Link to="/" className="text-gold hover:underline mt-4 inline-block">
           Go home
         </Link>
       </div>
     </div>
   } />
   ```

### Part C: Password Reset Return URL

5. **Configure Firebase action URL** — in Firebase Console → Authentication →
   Templates → Password Reset, set the "Action URL" to
   `https://zeyaroo.com/__/auth/action` (or whatever your domain is).

   Alternatively, handle it in-app by adding a route:
   ```tsx
   // App.tsx
   <Route path="/auth/action" element={<AuthActionPage />} />
   ```
   Where `AuthActionPage` reads `mode` and `oobCode` from query params and
   handles `resetPassword` mode using `confirmPasswordReset` from Firebase.

   The simpler approach for beta: just ensure Firebase's default action URL
   works and redirects back to `/login` afterward. Configure the "continue URL"
   in Firebase Console to `https://zeyaroo.com/login`.

### Verification
- Visit `/terms`, `/privacy` — pages render
- Auth pages show ToS/Privacy links
- Visit `/nonexistent` — styled 404 with "Go home" link
- Trigger password reset → email arrives → click link → can reset → redirected
  to login

---

## Session 5 — Mobile Responsive Layout

**Goal:** Make the app usable on phones. The sidebar becomes a slide-out drawer
on small screens.

**Files:**
- `phto-ui/src/components/Layout.tsx`
- `phto-ui/src/components/Sidebar.tsx`

### Steps

1. **Add mobile state to Layout.tsx** — track whether the mobile drawer is
   open:
   ```tsx
   const [mobileOpen, setMobileOpen] = useState(false);
   ```

2. **Update Sidebar to accept mobile props:**
   ```tsx
   interface SidebarProps {
     collapsed: boolean;
     onToggleCollapsed: () => void;
     mobileOpen: boolean;
     onMobileClose: () => void;
   }
   ```

3. **In Sidebar, render two modes:**
   - Desktop (`md:` and up): current fixed sidebar behavior, unchanged
   - Mobile (`< md`): overlay drawer with backdrop

   ```tsx
   export function Sidebar({ collapsed, onToggleCollapsed, mobileOpen, onMobileClose }: SidebarProps) {
     // ... existing code ...

     return (
       <>
         {/* Mobile backdrop */}
         {mobileOpen && (
           <div
             className="fixed inset-0 z-40 bg-black/50 md:hidden"
             onClick={onMobileClose}
           />
         )}

         <aside
           className={`
             fixed inset-y-0 left-0 z-50 flex flex-col w-56 bg-surface-muted
             border-r border-border-main transition-transform duration-200
             ${mobileOpen ? 'translate-x-0' : '-translate-x-full'}
             md:translate-x-0
             ${collapsed ? 'md:w-16' : 'md:w-56'}
           `}
         >
           {/* ... existing sidebar content ... */}
         </aside>
       </>
     );
   }
   ```

4. **In Layout.tsx, add mobile hamburger button:**
   ```tsx
   {/* Mobile header bar - only visible on small screens */}
   <div className="md:hidden fixed top-0 left-0 right-0 z-30 h-14 bg-surface-muted
     border-b border-border-main flex items-center px-4">
     <button onClick={() => setMobileOpen(true)}>
       <Menu className="w-5 h-5 text-text-main" />
     </button>
     <span className="ml-3 font-serif text-lg text-text-main">{branding.appName}</span>
   </div>
   ```

5. **Update content margin classes:**
   ```tsx
   <div className={`
     relative z-20 min-h-screen bg-surface-primary
     pt-14 md:pt-0
     ${isAuthenticated ? `md:${contentML}` : ''}
     transition-all duration-200
   `}>
   ```

6. **Close mobile drawer on navigation** — in Sidebar, listen to
   `location.pathname` changes:
   ```tsx
   const location = useLocation();
   useEffect(() => { onMobileClose(); }, [location.pathname]);
   ```

7. **Hide the desktop collapse toggle on mobile** — add `hidden md:flex` to
   the collapse button.

### Verification
- Resize browser to < 768px → hamburger appears, sidebar is hidden
- Tap hamburger → sidebar slides in with backdrop
- Tap a nav link → sidebar closes, page navigates
- Tap backdrop → sidebar closes
- On desktop (>= 768px) → behavior unchanged

---

## Session 6 — Billing Page Cleanup + Client Dashboard

**Goal:** Remove mock data from the billing page and differentiate the dashboard
for client vs creator roles.

### Part A: Billing Page Mock Data

**File:** `phto-ui/src/pages/Billing.tsx`

1. **Replace `MOCK_SUMMARY` with real API data** — create a new API hook or
   extend `usePayments`:
   - `totalEscrow` → sum of funded but unreleased milestones (need backend
     endpoint or derive from existing milestone data)
   - `availablePayout` → same as available payout requests
   - `lifetimeEarnings` → sum of all released payments

   **If the backend endpoints don't exist yet**, replace the animated rings with
   a simpler "Coming soon" card or honest "No data yet" empty state. Do NOT
   show fake numbers to beta users.

2. **Replace `MOCK_SUBSCRIPTION`** — the subscription data exists in
   `/api/v1/storage/usage` and user tier info. Wire it up or remove the
   SubscriptionCard for beta.

3. **Replace `MOCK_PAYOUT`** — the payout method is stored in creator profile
   (bank_name, account_number from PayoutSettings). Wire to
   `useMyCreatorProfile`.

4. **Remove `MOCK_TRANSACTIONS` fallback** — when real payments are empty, show
   an empty state instead of fake data:
   ```tsx
   {transactions.length === 0 ? (
     <div className="text-center py-12 text-text-muted">
       <p>No transactions yet</p>
       <p className="text-sm mt-1">Transactions will appear here when payments are made</p>
     </div>
   ) : (
     /* existing transaction list */
   )}
   ```

5. **Fix hardcoded values:**
   - `"5%"` platform fee → derive from user tier or config
   - `currency: 'USD'` in payout request → use user's currency preference

### Part B: Client Dashboard

**File:** `phto-ui/src/pages/Dashboard.tsx`

6. **Conditionally render dashboard sections based on role:**
   ```tsx
   const { user } = useAuthStore();
   const isCreator = user?.role === 'creator';
   ```

   - **Creator dashboard** (current): revenue ring, inquiry pulse, project
     stacks, studio activity — keep as-is
   - **Client dashboard**: show:
     - "Your Projects" — projects shared with them or where they're the client
     - "Pending Selections" — galleries waiting for their pick/reject
     - "Active Proposals" — proposals where they're the client
     - Remove revenue/earnings/storage rings for clients

7. **Fix the non-functional quick upload button** (line 249) — either wire it
   to open the upload dialog for that project, or remove the button entirely:
   ```tsx
   // Option A: Remove it
   // Delete the upload icon button on hover

   // Option B: Wire it (preferred)
   onClick={(e) => {
     e.preventDefault();
     e.stopPropagation();
     navigate(`/projects/${project.id}?upload=true`);
   }}
   ```

8. **Fix "Studio Activity" section** — rename to "Recent Activity" and consider
   using the notifications API data, which is what it semantically represents.

9. **Fix "Upcoming Milestones" links** — each item should link to
   `/proposals/{proposalId}` not `/inquiries`.

### Part C: BillingSettings Invoice Detail

**File:** `phto-ui/src/pages/settings/BillingSettings.tsx`

10. **Replace `console.log` with a simple detail drawer or navigation:**
    ```tsx
    // Quick fix: navigate to a detail view or show a toast
    onViewDetails={(invoice) => {
      toast.info(`Invoice #${invoice.invoice_number} — ${invoice.status}`);
      // TODO: build invoice detail drawer in a future session
    }}
    ```

### Verification
- Login as creator → see creator dashboard with real data (or honest empty
  states)
- Login as client → see client-appropriate dashboard
- Visit `/billing` → no mock data visible, empty states where data doesn't
  exist yet
- Quick upload button either works or is gone

---

## Session 7 — Inquiry Pipeline Persistence + Vault Polish

**Goal:** Persist Kanban drag state and fix Vault placeholder data.

### Part A: Inquiry Pipeline Status

**Backend:**

1. **Add `pipeline_status` column to inquiries table:**
   ```go
   // phto-api/internal/models/inquiry.go — add field:
   PipelineStatus string `json:"pipeline_status" gorm:"default:new"`
   ```
   GORM auto-migrate will add the column.

2. **Add PATCH endpoint** for updating pipeline status:
   ```go
   // phto-api/internal/handlers/routes.go — under /inquiries:
   r.Patch("/{id}/pipeline-status", h.Inquiry.UpdatePipelineStatus)
   ```

3. **Add handler:**
   ```go
   // phto-api/internal/handlers/inquiry.go
   func (h *InquiryHandler) UpdatePipelineStatus(w http.ResponseWriter, r *http.Request) {
       // Parse ID, decode body { "pipeline_status": "contacted" }
       // Validate allowed values: "new", "contacted", "proposal_sent", "won", "lost"
       // Update via service
   }
   ```

**Frontend:**

4. **Add API function:**
   ```ts
   // phto-ui/src/api/inquiries.ts
   export function useUpdatePipelineStatus() {
     return useMutation({
       mutationFn: ({ id, status }: { id: string; status: string }) =>
         apiClient.patch(`/inquiries/${id}/pipeline-status`, { pipeline_status: status }),
       onSuccess: () => queryClient.invalidateQueries({ queryKey: ['inquiries'] }),
     });
   }
   ```

5. **Wire into Inquiries.tsx drag handler** — replace the TODO at line 460:
   ```tsx
   const updatePipeline = useUpdatePipelineStatus();
   // In onDragEnd:
   updatePipeline.mutate({ id: activeId, status: newColumn });
   ```

### Part B: Vault "Hire Again"

**File:** `phto-ui/src/pages/Vault.tsx`

6. **Wire to actual creator profile data** — each project has a creator (the
   `user_id` / owner). Fetch creator profiles for the displayed projects:
   ```tsx
   // Use the existing creator profile API
   // Map project.user_id → creator profile (display_name, avatar, slug)
   ```
   Or if the project response already includes creator info, use that directly.

7. **If creator data isn't available on the project model**, add it to the
   project response in the backend:
   ```go
   // phto-api/internal/handlers/project.go — in projectToResponse():
   // Include CreatorName, CreatorSlug from the project owner's profile
   ```

### Verification
- Drag an inquiry card to a different column → refresh → card stays in new
  column
- Visit Vault → "Hire Again" shows real creator names and avatars

---

## Session 8 — Post-Beta Polish (Can Wait)

These items improve quality but aren't beta blockers. Tackle when convenient.

### A: Duplicate Route Cleanup
- Remove `/profile` route from App.tsx — `/settings/profile` is the canonical
  path
- Update any links pointing to `/profile` → `/settings/profile`
- In Sidebar.tsx, change the Profile nav link path from `/profile` to
  `/settings/profile`

### B: Remove Duplicate Storage Page
- Delete `phto-ui/src/pages/Storage.tsx` — it's a copy of
  `settings/StorageSettings.tsx` and isn't linked from anywhere

### C: Onboarding Wizard Enhancement
- Replace the simple modal with a 3-step wizard:
  1. Upload avatar + enter display name
  2. Pick service categories
  3. Write bio + add social links
- Show a progress bar; each step saves independently via the existing creator
  profile API
- Mark complete when all 3 steps are done

### D: Real-Time Updates (SSE) [don't do it now - skip]
- Add a `/api/v1/events/stream` SSE endpoint in the backend
- Subscribe to event types: `notification.new`, `media.uploaded`,
  `milestone.status_changed`
- In the frontend, create a `useEventStream` hook that connects on auth and
  dispatches to TanStack Query invalidation
- Replace the 30-second polling in `NotificationBell` with SSE push

### E: Account Deletion [skip for now]
- Backend: `DELETE /api/v1/users/me` — soft-delete user, anonymize PII, cancel
  active subscriptions, queue storage cleanup
- Frontend: Add "Delete Account" button in settings with double-confirmation
  dialog and email verification

### F: Accessibility Pass [skip for now]
- Add `aria-label` to all icon-only buttons in Sidebar.tsx
- Add `aria-live="polite"` to the culling view status region
- Ensure all interactive elements have visible focus rings
- Test with keyboard-only navigation through the main flows

### G: Hardcoded Values
- `"$25"` forever gallery price in ProjectDetail.tsx → fetch from config/tier
- `"5%"` platform fee in Billing.tsx → derive from user tier
- `"30 days"` trash expiry in Projects.tsx → derive from backend config

---

## Session Dependency Map

```
Session 1 (Toast)      ──┐
Session 2 (Auth/Logs)  ──┤── Can run in parallel, no deps
Session 3 (Rate/Shut)  ──┤
Session 4 (Legal/404)  ──┘

Session 5 (Mobile)     ── Independent, can run anytime

Session 6 (Billing/Dashboard) ── Depends on Session 1 (uses toast)

Session 7 (Kanban/Vault) ── Independent

Session 8 (Polish)     ── Independent, post-launch
```

---

## Quick Reference: Files Per Session

| Session | Backend files | Frontend files |
|---|---|---|
| 1 | — | ~15 pages/components + App.tsx |
| 2 | cmd/server/main.go | ProtectedRoute, FirebaseAuthContext, config/app.ts, BillingSettings |
| 3 | middleware/ratelimit.go (new), handlers/routes.go, cmd/server/main.go | — |
| 4 | — | Legal.tsx (new), App.tsx, Login.tsx, Register.tsx + Firebase Console |
| 5 | — | Layout.tsx, Sidebar.tsx |
| 6 | — | Billing.tsx, Dashboard.tsx, BillingSettings.tsx |
| 7 | models/inquiry.go, handlers/inquiry.go, services/inquiry.go, handlers/routes.go | api/inquiries.ts, Inquiries.tsx, Vault.tsx |
| 8 | handlers (SSE), services (account deletion) | Multiple |
