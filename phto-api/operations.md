# 9. What to Expect

## Common failure modes

### Backend process crashes

The service is a single Go binary on a single VM. If it crashes, the Komodo stack restart policy brings it back within seconds. Active HTTP requests are lost; any in-flight scheduler jobs are re-run on the next tick. In-flight EventProcessor jobs will be retried because they are durable (pending rows in the `jobs` table).

**Symptoms.** `phto-ui` shows network errors; health check goes red; Caddy logs 502s.

**Remediation.** First, `komodo` stack status. If the container is restarting in a loop, `komodo` logs will show the Go panic. The most common cause is a config change missing an env var.

### Event processor backlog

If email or WhatsApp is flaky, jobs pile up in the events DB. The 30 s poll + 100-per-tick batch is the throughput ceiling — ~12,000 jobs/hour. Past that, jobs age.

**Symptoms.** Delayed emails; `/admin/jobs` shows hundreds of pending rows.

**Remediation.** Check the external provider (Resend dashboard, Dodo status). Increase `EVENTS_PROCESSING_INTERVAL` lower, or bump batch size. Worst case, mark stale jobs `failed` manually to drain the queue.

### B2 upload failures

Direct uploads from browsers fail with CORS errors or pre-signed URL expiry.

**Symptoms.** Uploads stuck at 0%; browser console shows CORS violation; `MediaObject` rows stuck in `pending`.

**Remediation.** Re-check the B2 CORS config against [guides/b2-cors.md](guides/b2-cors.md). Pre-signed URLs are valid for `S3_URL_EXPIRY_MINUTES` (default 30); a user who sits on the upload dialog too long will fail. Retrying with a fresh `POST /initiate-upload` works.

### Webhook auth failures

Dodo or PayHere webhooks come in and get rejected.

**Symptoms.** Payments succeed on the provider side but milestones stay `awaiting_funds`; `/admin/webhook-attempts` shows `auth_failed` rows.

**Remediation.** Confirm `DODO_WEBHOOK_SECRET` or `PAYHERE_SECRET` matches the provider. PayHere sandbox mode bypasses signature — check `PAYHERE_SANDBOX` flag. Raw body is stored on every attempt; replay via the webhook source admin UI.

### Database disk full

SQLite on prod (which shouldn't happen, but has) or Postgres hitting quota.

**Symptoms.** Writes start failing with `disk I/O error` or Postgres out-of-space errors.

**Remediation.** For event/job/webhook tables: run `cleanup-completed-jobs` manually, or shorten retention. For media storage: check `StorageUsageLog` — users over quota may have uploaded large originals.

### Stuck milestone

A milestone that won't progress past a state.

**Symptoms.** Client paid, sees confirmation, but `/milestones/{id}` shows `awaiting_funds`.

**Remediation.** Correlate with webhook attempts (`/admin/webhook-attempts`) by `order_id` / metadata. If the webhook never arrived, ask the provider to replay. If it arrived but processing failed, check the relevant job in `/admin/jobs` for the error.

## Scaling limits & bottlenecks

At current architecture, the hard ceilings are:

- **1 replica.** No job lease → duplicate processing on scale-out. See tech-debt.
- **~100 jobs/tick × 2/min = 12,000 jobs/hour** event processing ceiling.
- **Single B2 bucket, single region.** Media upload latency is bounded by trans-Pacific RTT for most clients.
- **SQLite in dev; Postgres single instance in prod.** No read replicas.
- **ARM VM on OCI Free Tier.** 4 OCPU / 24 GB RAM shared between backend, Caddy, and (on dev VM) Woodpecker + Komodo.
- **Unauthenticated routes alone are rate-limited.** An authenticated attacker can hammer any endpoint. No global QPS ceiling.

Practical scale is "a few hundred active creators with a few thousand monthly client interactions." Past that, the above items need addressing.

## Operational runbook

### Alerts

No alerting is wired today. The monitoring stack (Grafana + Loki, see infra memory) can alert on Caddy error rate and VM resource pressure, but product-level alerts — "milestone stuck for >24h", "job failure rate above X" — are not configured.

### Common operations

**Deploy.** Merge to `main` (prod) or `develop` (dev). Woodpecker builds and triggers Komodo redeploy. Watch [https://komodo.dev.bytkloud.com](https://komodo.dev.bytkloud.com) for status.

**Rollback.** Komodo retains prior stack versions. Redeploy the previous version via Komodo UI.

**Force-run a scheduler job.** No CLI today. Restart the service (next tick fires on schedule). Exception: storage reconciliation has `cmd/reconcile` for on-demand runs.

**Manually approve a paid invoice.** `/admin/invoices` → mark paid. Writes a `PaymentTransaction` row.

**Advance dev clock.** Dev-only: `POST /admin/dev/clock/advance` with `{"duration":"24h"}` to test scheduled flows without waiting. Does not work in prod (routes not registered).

**Inspect a webhook.** `/admin/webhook-attempts` shows the full raw body and headers of every inbound webhook call for debugging.

**Reconcile storage.** `go run ./cmd/reconcile --all-projects` or `--project-id <uuid>`. Writes corrections where actual-on-disk differs from cached `TotalStorageBytes`.

### Typical on-call shift

There is no on-call rotation today — Bytkloud is a personal project with a single operator. In practice, the operator responds to:

1. **User reports.** "My payment went through but milestone didn't release." Investigation path: check `/admin/webhook-attempts` first, then `/admin/escrow/transactions` for the milestone, then `/admin/jobs` for event processing failures.
2. **Deploy regressions.** Woodpecker CI passes but a business flow is broken. Revert via Komodo, investigate locally.
3. **Storage cost spikes.** Unexpected B2 bill — run reconciliation, identify the outlier user, confirm their subscription covers it.

The logs are not rich enough to self-diagnose most issues — expect to pair browser devtools, admin dashboard, and direct Postgres queries.

## Graceful degradation

The system is designed so that secondary features fail independently:

| If this breaks | What still works |
|----------------|------------------|
| Resend down | All money flows; in-app notifications; WhatsApp; emails retry |
| WhatsApp down | All of the above minus WhatsApp |
| Dodo down | LKR payments via offline gateway; billing invoices via PayHere; free-tier usage |
| B2 down | API; no media uploads or serving |
| Firebase down | Email/password auth still works; Google sign-in doesn't |
| Admin DB corrupt | Main app runs; admin UI broken |

The system does **not** gracefully degrade if the main DB is down (everything halts) or if the single backend process is down (everything halts).
