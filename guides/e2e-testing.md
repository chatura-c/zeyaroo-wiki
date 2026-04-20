# Zeyaroo E2E Tests

Playwright test suite covering the core creator→client flow, project/media management, and
milestone/escrow lifecycle. Tests use JWT-only auth (no Firebase) and run against locally started
servers.

## Directory Structure

```
e2e/
├── playwright.config.ts          # Playwright config: webServers, workers, env
├── global-setup.ts               # Wipes test DB + uploads before each run
├── fixtures/
│   ├── index.ts                  # Custom fixtures: creatorAuth, clientAuth, creatorPage, clientPage
│   └── test-image.jpg            # 1×1 JPEG for upload tests
├── helpers/
│   ├── api.ts                    # Typed REST wrappers over /api/v1
│   └── auth.ts                   # injectAuth() — writes JWT into localStorage before page.goto()
└── specs/
    ├── auth.spec.ts              # Register/login, redirect, injected auth
    ├── creator-client-flow.spec.ts  # Profile update, inquiry→proposal→accept
    ├── project-media.spec.ts     # Project create, photo upload, album create
    └── milestone-escrow.spec.ts  # Full escrow lifecycle + UI fund button
```

## Running Tests

```bash
cd e2e
npm install
npx playwright install chromium   # first time only

npm test              # headless, all specs
npm run test:ui       # interactive Playwright UI
npm run test:headed   # watch tests run in browser
npm run test:debug    # step-through debugger
```

## How It Works

### Servers

Playwright starts both servers automatically via `webServer` in `playwright.config.ts`:

| Server | Command | URL |
|--------|---------|-----|
| Backend | `CGO_ENABLED=1 go run ./cmd/server/main.go` | `http://localhost:8080` |
| Frontend | `npm run dev` | `http://localhost:5173` |

`reuseExistingServer: true` locally (skip restart if already running), `false` in CI.

### Test Database

Each run gets a fresh SQLite database:
- `global-setup.ts` deletes `/tmp/zeyaroo-test.db` (+ WAL/SHM files) and `/tmp/zeyaroo-test-uploads`
- GORM `AutoMigrate` recreates the schema on backend startup
- `workers: 1` prevents parallel write conflicts

### Auth Strategy

Firebase auth is disabled in tests via `VITE_DISABLE_FIREBASE_AUTH=true`. The frontend's
`FirebaseAuthContext.tsx` has a guard at the top of its `useEffect` that calls `setLoading(false)`
and returns early when this env var is set — preventing `onAuthStateChanged` from firing with
`null` and wiping injected tokens.

JWT tokens are injected directly into `localStorage` before `page.goto()` via `injectAuth()`:

```ts
await injectAuth(page, auth);  // must be called before page.goto()
await page.goto('/dashboard');
```

This writes to the `auth-storage` Zustand persist key, matching the shape in `authStore.ts`.

### Fixtures

Each test gets fresh users (timestamp-based emails, no shared state):

```ts
import { test, expect } from '../fixtures';

test('example', async ({ creatorAuth, clientAuth, creatorPage, clientPage }) => {
  // creatorAuth / clientAuth: { token, user }
  // creatorPage / clientPage: Page with auth already injected
});
```

### API Helpers

`helpers/api.ts` provides typed wrappers for all test-relevant endpoints:

```ts
registerUser(email, password, role)     → AuthResult
loginUser(email, password)              → AuthResult
updateCreatorProfile(token, data)
createInquiry(clientToken, creatorId, data)  → InquiryResult (includes proposal.id)
createProposalVersion(token, proposalId, data)
acceptProposal(token, proposalId)
createProjectFromProposal(token, proposalId)
getProposal(token, proposalId)          → ProposalResult (includes latest_version.milestones[].id)
createProject(token, data)
fundMilestone / startMilestone / completeMilestone / releaseMilestone
```

## Backend Changes

Two small changes were made to the backend to support deterministic testing:

**`internal/config/config.go`** — added `MockPaymentFailureRate float64` to `EventsConfig`,
read from `MOCK_PAYMENT_FAILURE_RATE` env var (default `0.1`).

**`cmd/server/main.go`** — mock gateway uses `cfg.Events.MockPaymentFailureRate` instead of
hardcoded `0.1`. Tests set it to `0` for deterministic payment outcomes.

## Frontend Changes

**`src/contexts/FirebaseAuthContext.tsx`** — added 3-line guard at the top of `useEffect`:

```ts
if (import.meta.env.VITE_DISABLE_FIREBASE_AUTH === 'true') {
  setLoading(false);
  return;
}
```

Without this, the Firebase auth listener fires with `null` (no Firebase user in tests) and calls
`logout()`, wiping the JWT token injected via `injectAuth()`.

## Spec Overview

| Spec | What it covers |
|------|---------------|
| `auth.spec.ts` | JWT register + login, unauthenticated redirect, injected auth reaches dashboard |
| `creator-client-flow.spec.ts` | Creator profile update visible in /creators; inquiry→proposal→accept flow; creator sees inquiries list |
| `project-media.spec.ts` | Create project, upload photo via file input, create album via UI |
| `milestone-escrow.spec.ts` | API-driven full lifecycle (fund→start→complete→release); client funds milestone via UI button |

## Environment Variables (set by playwright.config.ts)

### Backend
| Variable | Value |
|----------|-------|
| `DB_DRIVER` | `sqlite` |
| `DB_DSN` | `/tmp/zeyaroo-test.db` |
| `JWT_SECRET` | `e2e-test-secret` |
| `JWT_EXPIRATION_HOURS` | `24` |
| `FIREBASE_PROJECT_ID` | _(empty — disables Firebase init)_ |
| `STORAGE_TYPE` | `local` |
| `STORAGE_PATH` | `/tmp/zeyaroo-test-uploads` |
| `EVENTS_PROCESSING_INTERVAL` | `5` |
| `MOCK_PAYMENT_FAILURE_RATE` | `0` |

### Frontend
| Variable | Value |
|----------|-------|
| `VITE_API_URL` | `http://localhost:8080/api/v1` |
| `VITE_DISABLE_FIREBASE_AUTH` | `true` |
