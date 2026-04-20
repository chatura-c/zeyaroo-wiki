# 6. Infrastructure & Deployment

## Environments

Three named environments. All run the same binary; only env vars and the target DB differ.

| Env | Host | DB | Frontend domain | Notes |
|-----|------|----|-----|-------|
| **local** | developer laptop | SQLite (`zeyaroo.db`) | `http://localhost:5173` | `APP_ENV=development` |
| **dev** | `komodo.dev.bytkloud.com` (OCI ARM VM) | Postgres | `https://dev.zeyaroo.com` | Also hosts Woodpecker + Komodo control plane |
| **prod** | second OCI ARM VM | Postgres | `https://zeyaroo.com` | Serves customer traffic |

Both VMs sit behind Caddy (TLS) and OCI Load Balancer. See the infra memory for IPs and OCIDs.

## Image build

### Dockerfile

Two-stage. See [phto-api/Dockerfile](phto-api/Dockerfile).

1. **builder** (`golang:1.24-alpine`) — installs `git`, `gcc`, `musl-dev` for CGO sqlite. Builds with `CGO_ENABLED=1 GOOS=linux GOARCH=arm64` and `-ldflags="-s -w -X main.BuildNumber=$BUILD_NUMBER"`. Output: `./server`.
2. **runtime** (`alpine:3.20`) — adds `ca-certificates` and `tzdata`. Copies binary and `.env.defaults`. Creates non-root `bkloud-user`. Exposes `8080`. `CMD ["./server"]`.

### build_spec files

Two OCI DevOps build specs live in the repo. The current active spec is [build_spec.yml](phto-api/build_spec.yml), which logs into OCIR at `iad.ocir.io` and uses `docker buildx` to push a multi-arch (`linux/amd64,linux/arm64`) image with registry-backed build cache. The older [build_spec.yaml](phto-api/build_spec.yaml) is a local-only build that does not push.

These specs are superseded in practice — CI now runs on Woodpecker, not OCI DevOps.

## CI/CD

**[phto-api/.woodpecker.yml](phto-api/.woodpecker.yml).** Runs on `push` or `manual` to `main` or `develop` (pipeline on `linux/arm64`).

Two steps:

1. **build** — `woodpeckerci/plugin-docker-buildx` pushes to `iad.ocir.io/idjaj5kwnexw/zeyaroo-api:${CI_COMMIT_BRANCH}`. OCIR creds from Woodpecker secrets `ocir_username` / `ocir_password`.
2. **redeploy** — Alpine + `curl` + `jq` hits the Komodo API at `https://komodo.dev.bytkloud.com`. Maps `main` → `zeyaroo-prod` stack, `develop` → `zeyaroo-dev`. Calls `DeployStack`, polls `GetUpdate` every 3 s up to 90 s, fails non-zero if the deploy fails. Creds in secrets `komodo_key` / `komodo_secret`.

**Release = push to main** — there is no staging promotion step. A `main` push produces the `:main` tag and auto-deploys to prod. Dev tracks `develop`. Be careful with `main` merges.

## Infrastructure-as-code

OCI infrastructure (VMs, LB, OCIR, object storage) is managed in a separate repo: `bytkloud-tf`. Not in phto-api. Komodo stacks (Docker Compose definitions) live in an `infra-modules` repo. Neither is covered here — see the `bytkloud-infra` skill.

Manually configured items — not in IaC — that matter for a new deployment:

- **B2 bucket CORS** — run through the checklist at [guides/b2-cors.md](guides/b2-cors.md) before direct uploads will work in a browser.
- **Firebase project** — Google sign-in requires a real Firebase project; credentials path or inline JSON in env. Covered in the firebase projects memory.
- **Resend domain verification** — before prod email sends.

## Config

All config is env vars. Loaded by [internal/config/config.go](phto-api/internal/config/config.go) with precedence: `.env.defaults` → `.env.{development,production}` → `.env.local` → OS env (OS always wins). Godotenv does **not** override already-set vars.

Notable keys (full table in the config file; highlights here):

| Key | Default | Purpose |
|-----|---------|---------|
| `APP_ENV` | `development` | Switches env file, sets DevMode |
| `SERVER_HOST` / `SERVER_PORT` | `0.0.0.0:8080` | Bind |
| `DB_DRIVER` / `DB_DSN` | `sqlite` / `phto.db` | Main DB |
| `{EVENTS,NOTIFICATIONS,WEBHOOKS,STORAGE}_DB_DSN` | fallback main | Per-domain DBs |
| `JWT_SECRET` | `change-me-in-production` | **Must** rotate in prod |
| `FIREBASE_PROJECT_ID` / `FIREBASE_CREDENTIALS_JSON` | `zeyaroo-dev` | Firebase auth (optional) |
| `STORAGE_TYPE` / `S3_BUCKET` / `S3_ENDPOINT` / `S3_REGION` / `S3_ACCESS_KEY_ID` / `S3_SECRET_ACCESS_KEY` | `local` / various | Storage backend |
| `MEDIA_CDN_URL` | `` | Optional CDN prefix for signed URLs |
| `RESEND_API_KEY` / `EMAIL_FROM` | `` / `noreply@phto.app` | Email |
| `DODO_API_KEY` / `DODO_WEBHOOK_SECRET` | `` | DodoPayments |
| `PAYHERE_MERCHANT_ID` / `PAYHERE_SECRET` / `PAYHERE_SANDBOX` | `` / `true` | PayHere (billing invoices) |
| `ADMIN_EMAIL` / `ADMIN_PASSWORD` | `admin@zeyaroo.com` / `admin123` | Seed admin |
| `PLATFORM_DEFAULT_CURRENCY` / `PLATFORM_FEE_PERCENTAGE` | `LKR` / `5.0` | Billing defaults |
| `DEBUG` | `` | Comma-separated module list for applog |

Per-environment sample files: `.env.defaults`, `.env.development`, `.env.production` are committed. `.env.local` is gitignored for secrets.

## Observability

- **Logs.** Custom `applog` package wraps stdlib `log`. Module-based debug toggle via `DEBUG` env var (e.g. `DEBUG=sql,http,events`). All output to stderr. **Not structured** — plain text prefixed with `[module]`. No request-ID injection beyond what `chi.RequestID` provides.
- **Health.** `GET /health` returns `{"status":"ok"}`. No `/ready`. Caddy should probe this.
- **Metrics.** None in-process. The Bytkloud monitoring stack (see infra memory) scrapes at the VM level.
- **Tracing.** None. OpenTelemetry shows up transitively via Firebase SDK imports but is not wired.
- **Error aggregation.** None (no Sentry / Rollbar).

## Graceful shutdown

In [cmd/server/main.go:271](phto-api/cmd/server/main.go#L271), `SIGINT`/`SIGTERM` → stop event processor + scheduler → `http.Server.Shutdown` with a 15 s context. The EventBus is stopped via `defer`. In-flight HTTP requests have up to 15 s to complete; running scheduler jobs are canceled immediately via context.

## Tests and QA

- **Unit tests.** Only two test files: [internal/services/billing_test.go](phto-api/internal/services/billing_test.go) (the richest) and [internal/models/milestone_test.go](phto-api/internal/models/milestone_test.go) (shallow). Billing tests run against `sqlite::memory:` with a `nowFn` injection for deterministic time.
- **Integration tests.** None in this repo.
- **E2E tests.** Live in a separate repo (`e2e/`) — Playwright against phto-ui + phto-api; see [guides/e2e-testing.md](guides/e2e-testing.md).

## Databases on disk

During local runs, four SQLite files are created in the working directory: `phto.db` (legacy default), `zeyaroo.db`, `zeyaroo.db-wal`, `zeyaroo.db-shm`. These are dev artifacts — not committed, never production state. Safe to delete for a fresh DB. Add to `.gitignore` if not already.
