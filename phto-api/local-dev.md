# 10. Local Development

## Prerequisites

- **Go 1.24** or newer.
- **gcc** + **musl-dev** (or equivalent) — `CGO_ENABLED=1` is required for the SQLite driver.
- **Node 20+** and **npm** — only if you also run `phto-ui`.
- A Firebase service-account JSON, or skip Firebase by leaving `FIREBASE_PROJECT_ID` unset (email/password login still works).
- **Optional:** Backblaze B2 credentials if you want to test media uploads against a real bucket. Default is local disk.
- **Optional:** Resend API key for real email. Default is a no-op sender that logs to stderr.

No Docker, Docker Compose, or Make targets are required for local dev. The app self-contains.

## Running

From `phto-api/`:

```bash
go run ./cmd/server
```

On first run:

- `internal/config/config.go` loads `.env.defaults` then `.env.development` then `.env.local` (if present).
- GORM creates `zeyaroo.db` in the working directory with all tables.
- `SeedAdmin` creates the admin user (default: `admin@zeyaroo.com` / `admin123`).
- HTTP server binds `0.0.0.0:8080`.
- 12 scheduler jobs start; EventProcessor starts polling.

Health check: `curl http://localhost:8080/health` → `{"status":"ok"}`.

## Typical dev env overrides

Put these in `.env.local` (gitignored):

```
JWT_SECRET=dev-only-secret-change-me
APP_ENV=development
DB_DSN=zeyaroo.db

# Firebase — copy file path from the firebase_projects memory
FIREBASE_PROJECT_ID=zeyaroo-dev
FIREBASE_CREDENTIALS_PATH=/path/to/zeyaroo-dev-service-account.json

# Email: enable real Resend, or omit for no-op
RESEND_API_KEY=re_xxx
EMAIL_FROM=dev@zeyaroo.com

# Payments: omit for MockGateway
DODO_API_KEY=
DODO_WEBHOOK_SECRET=

# Debug logging modules
DEBUG=http,sql,events
```

## Seeding real data

There is no all-purpose seed script. Options:

1. **Log in as admin** (`admin@zeyaroo.com` / `admin123`) and use the admin dashboard to create a creator.
2. **Register via `/auth/register`** — creates a regular creator/client user.
3. **Run `cmd/migrate-studio`** after creating creators to backfill studio projects:
   ```bash
   go run ./cmd/migrate-studio
   ```
4. **Use the E2E fixtures.** `e2e/` has `injectAuth()` and seed helpers; see [guides/e2e-testing.md](guides/e2e-testing.md).

## Resetting state

SQLite is a single file. Stop the server and:

```bash
rm -f zeyaroo.db zeyaroo.db-shm zeyaroo.db-wal phto.db
rm -rf uploads/   # local storage
```

Restart the server — fresh DB, fresh admin, fresh uploads. If you also want fresh notifications / events / webhooks / storage DBs, set separate DSNs in `.env.local` pointing to different files so you can delete them independently.

## Debugging

### Tracing a request

`DEBUG=http,sql` prints one line per request and every SQL statement to stderr. Combined with `chi`'s Recoverer, panics show a full stack.

### Inspecting state

- `sqlite3 zeyaroo.db` — direct read.
- `/admin/jobs` — event processor queue.
- `/admin/webhook-attempts` — raw inbound webhook log.
- `/admin/events` — stored events (audit).

### Testing scheduled jobs without waiting

In dev mode, advance the fake clock:

```bash
curl -X POST http://localhost:8080/api/v1/admin/dev/clock/advance \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"duration": "48h"}'
```

Scheduler jobs use the real `time.Now()`; to trigger them on demand, restart the server after advancing the clock and wait for the next tick. See [clock.go](phto-api/internal/clock/clock.go).

### Testing webhooks locally

Dodo and PayHere webhooks need a public URL. Either:

- Use `ngrok http 8080` and configure the provider to point there in sandbox mode.
- Manually `curl` a recorded body — copy the `Body` field from a `/admin/webhook-attempts` row and POST it back with the right signature header.

### Running tests

```bash
go test ./...
```

Only the billing and milestone test packages have meaningful tests today. Tests use `sqlite::memory:` via GORM — no external DB required.

### Helpful CLI tools in this repo

- `go run ./cmd/server` — main API
- `go run ./cmd/migrate-studio` — one-shot studio/portfolio backfill
- `go run ./cmd/reconcile --all-projects` — storage reconciliation

## Connecting to the frontend

`phto-ui` expects `VITE_API_URL=http://localhost:8080/api/v1` (or similar). See [guides/frontend-config.md](guides/frontend-config.md).

For JWT-only testing (bypassing Firebase), `VITE_DISABLE_FIREBASE_AUTH=true` makes the UI use the native register/login paths.

## Common dev issues

- **"sqlite3: unable to open database"** — working directory isn't writable, or `zeyaroo.db-wal` lock from a previous crashed process. Delete the WAL files.
- **Panic on startup: `database/sql: unknown driver "sqlite"`** — CGO didn't compile. Install `gcc`/`musl-dev`; check `CGO_ENABLED=1`.
- **Firebase 401 on every request** — token mismatch between UI and backend Firebase project IDs, or expired Firebase service-account JSON path. Unset `FIREBASE_PROJECT_ID` to disable entirely.
- **CORS errors from phto-ui** — the UI origin must match the allowlist in [router.go:60-91](phto-api/internal/handlers/router.go#L60-L91). Localhost ports 3000, 3001, 4173, 5173, 8000, 8080 are allowed by default.
- **WAL files accumulating in Git status** — add `*.db*` to `.gitignore`.
