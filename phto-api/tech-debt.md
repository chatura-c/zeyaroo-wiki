# 8. Technical Debt & Future Improvements

Catalogued by effort. "Straightforward" is a few days of focused work; "harder" is a multi-week refactor; "deferred" is something we chose not to build because it isn't worth the cost yet.

## Straightforward to fix next

- **Replace hardcoded JWT default.** Make the server refuse to start if `APP_ENV=production` and `JWT_SECRET` is the default string. One-line change; prevents a trivial prod vuln.
- **Add `.gitignore` entries for SQLite files.** `*.db`, `*.db-wal`, `*.db-shm` in the repo root. Small churn; fewer "why is the DB broken on my laptop" tickets.
- **Unify the direct-email bypass in `EscrowService`** with the EventProcessor channel pattern. Several places still do `go func() { emailService.SendMilestoneFundedEmail(...) }()`. These should publish an event like every other flow. Session-level refactor.
- **Add a `/ready` endpoint** that checks DB connectivity and B2 reachability. Caddy currently only hits `/health`, so a degraded backend still looks green.
- **Finish the [REFACTOR.md](phto-api/REFACTOR.md) plan.** Sessions 1–2 are done; 3–8 (media, proposal, project, share, amendment, misc splits) remain. Pure mechanical; improves readability without behavior change.
- **Wire `ProjectHandover` into `database.Migrate()`.** The model exists in [internal/models/project_transfer.go](phto-api/internal/models/project_transfer.go) but is not in the migrate list. Either land the handover feature or delete the unused struct.
- **Structured logging.** Swap `applog` for `log/slog` with a JSON handler. One PR. Immediate observability improvement.
- **Admin tool for stuck jobs.** `/admin/jobs` lists jobs; add a "retry" and "force-complete" action so ops doesn't need `psql` to rescue a failed notification.

## Harder but worth doing

- **Proper migration tool.** Introduce `golang-migrate` or Atlas. Required before the first column rename or type change. Cost: write migrations for the current schema, wire into CI and startup, document the workflow. Benefit: we stop being a rename away from downtime.
- **EventProcessor with job leases.** Add a `leased_until timestamp` column + `UPDATE ... WHERE leased_until IS NULL` pattern. Unlocks horizontal scaling of the backend. Also: a push mechanism (NATS or DB `LISTEN/NOTIFY`) to replace the 30 s poll.
- **Split services into sub-packages** (cross-service interfaces). The `handlers/` sub-package split is already planned in REFACTOR.md; the equivalent `services/` split needs interface extraction for cross-service deps (e.g. `EscrowService` holds `*AmendmentService`). Bigger refactor but brings compile-time coupling down.
- **Separate payment/billing module.** Right now escrow, gateways, invoices, payouts, and `payments.Registry` are tangled. Extract into `internal/payments/` as a clean module with its own interfaces. Reduces the EscrowService file-count.
- **End-to-end contract tests for webhooks.** Current test coverage is ~0% for the Dodo IPN + PayHere IPN paths. Ship a test that replays a recorded webhook body with a known signature against a fresh DB.
- **Two-tier cache for signed URLs.** Currently every request re-signs even within the TTL window. A small in-process map would cut CPU when a gallery page loads 100 images.

## Deliberately deferred

- **Observability (tracing / metrics / APM).** At single-digit QPS the value doesn't justify the setup cost. Will be needed before the next scale jump.
- **Cross-DB transactions.** Splitting into five DBs was a deliberate choice; we accepted non-atomic writes across them. Not worth un-splitting.
- **gRPC / typed API client.** Handwritten TypeScript client in `phto-ui`. Generating one from an OpenAPI spec would require adding an annotation layer over chi. Low priority.
- **Horizontal scaling.** Explicitly capped at one replica until job leases land. Single ARM VM is fine for the next 10× of traffic.
- **Multi-region storage.** Everything is in one B2 bucket in `us-east-005`. Sri Lankan clients eat the latency. The CDN prefix (`MEDIA_CDN_URL`) is the planned mitigation; not yet set up.
- **Custom-domain client galleries.** Asked about, not scoped. Requires wildcard cert work on Caddy and changes to the share handler.

## TODO / HACK / FIXME markers worth knowing

These are scattered through the codebase — grep is authoritative. A few notable ones:

- `removeMilestoneWatermarks` in escrow is synchronous and slow on large deliveries. TODO to move into EventProcessor.
- `HandleDodoPaymentsWebhook` treats missing metadata as an unknown event type and swallows. Should fail loud so admin dashboard surfaces it.
- `autoCompleteProject` has a TODO to also clear `ExpiresAt` — currently an auto-completed project with an expiry still gets marked expired the next day.
- `sendPaymentRequestedEmail` has a goroutine that does not surface errors. See direct-email bypass note above.
- Stripe handlers in `payment_event_processor.go` are dead code preserved for future reuse. Either finish integrating Stripe or delete them.

## Out of scope but adjacent

- **Client SDK for creators.** API clients / zapier integrations have been asked about. Not planned.
- **Mobile app.** Creators increasingly want to approve deliveries on phone. Current mobile browser flow works but isn't optimized. Separate project.
- **AI culling.** Picked but not built. See the archived `going-live/culling.md` for the original thinking.
