# 3. Key Decisions & Rationale

Each decision is recorded as **Decision → Why → Tradeoff accepted**. Where a decision is being revisited, that is called out.

## Go 1.24 with chi

**Decision.** The backend is written in Go using the chi v5 router. No Gin, Echo, Fiber, or gRPC.

**Why.** chi leans on `net/http` primitives and has a tiny surface area — middleware is just `http.Handler` wrapping. That keeps the handler package readable without learning a framework DSL, and keeps the dependency graph shallow. Go itself is a fit for a long-running server with many concurrent long-poll/webhook paths, and it produces a single statically linked binary that lands on the OCI ARM VM with minimal fuss.

**Tradeoff.** chi gives no generator, no OpenAPI inference, no dependency injection container. Handlers are wired by hand in `router.go` and `routes.go`. Adding a new domain is a mechanical chore rather than a command.

## GORM over raw SQL

**Decision.** Persistence goes through GORM. Every model embeds `Base` with UUID, `CreatedAt`, `UpdatedAt`.

**Why.** The domain has ~50 tables and is still moving weekly. Writing raw `sqlx` or `squirrel` queries would double the churn cost and produce no runtime benefit at current scale (single-digit QPS). GORM handles migrations via `AutoMigrate`, which is cheap enough that migration infrastructure (Atlas/goose) isn't yet worth its weight.

**Tradeoff.** GORM has footguns — silent `Updates` behavior, N+1 preload traps, generated SQL that is rarely the optimal shape. We live with these at current scale. The billing service has the only non-trivial query optimization so far.

## Dual database driver: SQLite dev, Postgres prod

**Decision.** Developers run against SQLite by default (`DB_DRIVER=sqlite`, `DB_DSN=zeyaroo.db`). Production is Postgres.

**Why.** Zero-friction local setup. Clone, `go run ./cmd/server`, get a working server. No Docker Compose dance for the DB. The schema is simple enough that GORM-generated DDL is portable.

**Tradeoff.** A small number of behaviors differ between SQLite and Postgres — notably case-sensitivity, concurrent writes, and JSON column semantics. A few integration issues have been caught late because the local SQLite DB masked them. The SQLite WAL files (`zeyaroo.db-wal`, `zeyaroo.db-shm`) land in the working directory and can confuse newcomers.

## Five separate logical databases

**Decision.** The config exposes five DSNs: main, events, notifications, webhooks, storage. All default to the main DSN if the corresponding env var is empty.

**Why.** Events and webhook-attempt tables are high-churn and would otherwise dominate the main DB's backup size and autovacuum time. Keeping them separable lets us move them to a cheaper Postgres (or even a different engine) later without code changes.

**Tradeoff.** Services must carry a `DBSet` struct rather than a single `*gorm.DB`. Cross-database transactions are not supported — writes that span (e.g.) a user mutation and a notification row cannot be atomic.

## Dual auth: native JWT + Firebase

**Decision.** Clients can authenticate via either a platform-issued JWT (HS256, 24 h) or a Firebase ID token. `CombinedAuthenticate` tries JWT first, falls back to Firebase.

**Why.** Firebase provides Google sign-in without building OAuth from scratch. Native JWT is needed for email/password signup, guests, share access, and any environment without Firebase configured.

**Tradeoff.** Every request does two parses in the worst case. There is no central session store, which means invalidation is impossible — a stolen token is valid for up to 24 h. There is no refresh token; `/auth/refresh` just reissues a 24 h token if the presented one is still valid (it is a **rolling** token, not a true refresh flow).

## In-process durable event processor (not a queue)

**Decision.** The event/job system is a GORM table polled every 30 s by the same Go process that serves HTTP.

**Why.** Kafka / NATS / Redis Streams would triple the operational surface. At current scale (<100 jobs/hour), a DB-backed queue with exponential backoff is sufficient and has the operational property we most want: every job is visible in the admin dashboard as a row.

**Tradeoff.** No horizontal scaling — running two backend replicas would cause duplicate job execution unless we add a locking column. Poll latency is 30 s by default. This is fine today; it will not be fine at 10× load.

## Escrow as a milestone state machine, not a separate ledger service

**Decision.** Escrow logic lives inside `EscrowService` on the same process, mutating the `milestones` and `escrow_transactions` tables in the main DB with `gorm.Transaction`.

**Why.** Money correctness is the product's load-bearing property. Keeping it in-process with direct DB transactions means every state transition is atomic and reasonable — no eventual consistency, no sagas, no compensating transactions. The `TransactionLedger` append-only table gives us an auditable trail.

**Tradeoff.** Business rules are spread across `escrow.go`, `escrow_payment_request.go`, `escrow_manual_funding.go`, `escrow_auto_release.go`, `escrow_queries.go`. The split is mechanical, not clean. Diagnosing a "stuck" milestone requires reading several files.

## Gateway routing by currency, not by user choice

**Decision.** `GatewayRouter` dispatches to `OfflineGateway` (bank transfer / cash) for LKR and to `DodoPayments` for everything else. There is no gateway picker in the UI.

**Why.** Card acquiring in Sri Lanka requires PayHere or similar; DodoPayments handles cards internationally. The split is enforced here so the UI doesn't have to know.

**Tradeoff.** Adding a new currency means code change. Admins can override via `UserPackage.PaymentMethods` JSON for negotiated deals.

## Three independent notification channels

**Decision.** Email, in-app, and WhatsApp are three separate handler classes, each registered on every domain event. A failure in one channel does not block the others.

**Why.** Per-channel user preferences. Resend / WhatsApp outages should not cascade. Retries are per-job, so a WhatsApp template misconfiguration doesn't cause emails to be re-sent.

**Tradeoff.** Three handlers means three SQL lookups of the same object per event (each channel hydrates its own). The duplication is tolerated for now; a shared `eventpayload` cache is on the backlog.

## GORM AutoMigrate instead of a migration tool

**Decision.** Schema is created by `database.Migrate*` functions calling `db.AutoMigrate(...)` at every startup.

**Why.** We are not at a scale where a cold DB migration takes measurable time. AutoMigrate covers additive changes (new columns, new tables, new indexes) without ceremony. For the one destructive case in history — removing an index on `payment_request_token` — we embedded the backfill in the migrate function (see [internal/database/database.go:127](phto-api/internal/database/database.go#L127)).

**Tradeoff.** Column renames, type changes, and column drops are **not safe** under AutoMigrate. When we hit that case, we will need to introduce a migration tool. Deferred deliberately.

## OCI Free Tier ARM VM + Komodo + Woodpecker

**Decision.** The runtime is an ARM64 VM on OCI Free Tier. Caddy terminates TLS. Komodo runs Docker Compose stacks. Woodpecker runs CI.

**Why.** Cost. Bytkloud is a personal project; OCI's Free Tier offers a capable ARM VM for $0. Komodo provides a clean stack-redeploy API; Woodpecker is self-hostable and integrates with GitHub. This stack is documented in the user-level infra memory.

**Tradeoff.** Single VM = single point of failure. No auto-scaling. ARM-only builds (caught in the Dockerfile's `GOARCH=arm64`). When we outgrow Free Tier, this becomes a migration project.

## Direct upload to B2 via signed PUT URLs

**Decision.** Browsers upload media directly to B2 using `media_direct_upload` pre-signed URLs. phto-api never proxies the bytes.

**Why.** The backend is a single instance. Proxying multi-GB wedding shoots through it would saturate the VM's bandwidth.

**Tradeoff.** B2 CORS must be configured manually via the [b2-cors guide](guides/b2-cors.md). The upload completion is client-driven, so a client that uploads and then crashes leaves a pending `MediaObject` row — cleaned up by the `cleanup-expired-uploads` job.
