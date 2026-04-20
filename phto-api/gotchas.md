# 7. Gotchas & Landmines

A catalog of non-obvious behavior. Read before touching any of the named subsystems.

## Auth & tokens

- **`/auth/refresh` is not a refresh flow.** It requires a still-valid JWT and reissues a 24 h token. There is no refresh token. A user that logs out and comes back 25 h later must log in again.
- **Combined auth tries JWT first, Firebase second.** If you mint a JWT with the wrong signing key it will silently fail JWT verification and fall through to Firebase, producing a very confusing 401. Look at `DEBUG=http` + service logs to disambiguate.
- **Share tokens are 1 h, guest OTP tokens are 24 h.** Do not confuse them; the context keys differ (`ShareIDKey` vs `GuestEmailKey`).
- **Guests are authenticated by presence of `role=guest` claim** in the JWT. Do not write code that dispatches on "is there a user row" — the user row may not exist for a guest payer.

## Escrow & payments

- **Two gateway interfaces.** `PaymentGateway` is the legacy synchronous interface used only for `FundMilestone` with the `MockGateway`. `CheckoutGateway` is the redirect-based interface used for real Dodo / Offline payments. Do not confuse the two when adding a new provider.
- **Milestone funding has three code paths** that end in the same `held` escrow row: direct `FundMilestone` (mock only), client payment-link flow via `RequestFunding` → Dodo webhook, and offline manual funding (`MarkManuallyFunded`, `ApproveManualFunding`). Changes to the final state must be mirrored in all three.
- **Auto-release is 7 days from `submitted`.** The `auto-release-milestones` scheduler job runs hourly; it picks up milestones in `submitted` state whose `completed_at` is more than 7 days old, and releases with `ReleaseType = "auto"`. If you change the state model, you also own this job.
- **Releasing a milestone auto-rejects its pending amendments** via `autoRejectAmendmentsForMilestone`. This is async — do not assume amendment state is consistent immediately after release.
- **Platform fee is recalculated at release time** using `ResolveUserLimits`, not stored at funding. If a creator upgrades their tier between fund and release, the new fee applies.
- **Media delivery runs before the payment transaction opens.** File copies are expensive; doing them inside the `gorm.Transaction` would hold locks for minutes. Means a crash between copy and DB commit leaves orphan files (picked up by `cleanup-expired-uploads`).

## Amendments

- **Funded milestones can only be modified upward.** `validateEscrowSafeguard` blocks removal or price-decrease of any milestone that has a held escrow. An amendment that tries both will partially fail; write it as two amendments.
- **Price-increased amendments create `partially_funded` milestones** with `BalanceDueCents` computed. The client must top up via a new payment link. Don't try to release a `partially_funded` milestone — it will be rejected.

## Proposals

- **State machine is enforced by `Proposal.ValidTransitions`** not by the handler. Writing directly to `proposal.status` bypasses validation; always go through `proposal_service.Accept/Cancel/CreateVersion`.
- **`AcceptAsGuest` creates a user account.** The guest email becomes a real `User` row with `role=client`. If the email already exists as a real user, the accept is refused.
- **`CreateProjectFromProposal` runs inside the accept transaction.** Failure rolls back the proposal finalization too. Good — but it means project creation must not call external services (email, Dodo). Those go through EventProcessor.

## Media & storage

- **Direct upload is two-step.** `POST /initiate-upload` creates a pending `MediaObject` + returns a pre-signed PUT URL. Client uploads to B2. `POST /complete-upload` flips the record to `completed`. A client that crashes between steps leaves a pending row cleaned up by the `cleanup-expired-uploads` job after 1 h.
- **Image variants are JPEG regardless of source format.** A PNG upload gets `_thumb.jpg` / `_medium.jpg`. Do not expect format fidelity.
- **Signed URL expiry is TTL-aligned, not per-request.** Every signed URL issued in the same 24 h window has the same `exp` value. Change the TTL and all cached URLs invalidate at once.
- **`cull_status = reject` is soft deletion.** Rejected media is stripped from all albums but the `MediaObject` row and underlying file remain until `PurgeRejected()` is called. Storage usage still counts until purge.
- **Delivery copies media.** On milestone release, `DeliveryService` copies files from creator project to client project. Total bytes are captured on `Delivery.TotalBytes` for audit.

## Storage accounting

- **Two systems track storage.** The `EventBus`-backed `StorageEventHandler` updates counters synchronously on `media.uploaded` / `media.deleted`, then re-publishes to the durable EventProcessor. If you only emit to one, counters will drift.
- **Storage reconciliation runs daily.** The `storage-reconciliation` scheduler job recalculates actual bytes-on-disk and writes corrections. If a user reports a wildly wrong quota, check `StorageUsageLog` and wait for next run — or trigger `cmd/reconcile/main.go` manually.

## Notifications

- **Three channels, three jobs per event.** Email, in-app, and WhatsApp are independent EventProcessor handlers. Do **not** assume a single "send notification" path.
- **Direct email bypass still exists.** Some `EscrowService` methods `go func()` direct email sends outside the EventProcessor. These are fire-and-forget; a Resend outage will drop them silently. Adding new notifications: **do it via the channel pattern**, not via a goroutine.
- **Email notifications are gated by `NotificationSettings` JSON on `User`.** The channel reads the preference before sending. In-app ignores the preference.
- **Stripe event handlers exist but no Stripe gateway is wired.** `payment_event_processor.go` registers handlers for `stripe.payment_intent.succeeded` etc. — legacy, unused. Leave them alone.

## Events & jobs

- **EventProcessor polls every 30 s.** There is no pub/sub push. An event is not dispatched until the next tick.
- **Retries are 2^n minutes, max 5.** Past 5 retries the job goes to `failed` and never runs again. No dead-letter UI — check via `/admin/jobs`.
- **Two backend replicas will double-process.** The job store has no lease/lock column. Never run more than one phto-api replica until this is fixed.

## Router & middleware

- **No tier/plan middleware.** Feature gating lives in services. Do not search for a `RequirePro` middleware — it doesn't exist. Gate in your handler by calling `FeatureGate.CanXxx`.
- **`CombinedOptionalAuthenticate` is used on `/creators/{id}/inquire`.** It populates context if a token is present, otherwise passes through. A guest inquiry will have no `UserIDKey`; your handler must handle both cases.

## Database

- **AutoMigrate at every startup.** Fine for additive changes. **Renames, type changes, and drops silently break the running process.** Write manual SQL embedded in `Migrate()` for those cases, like the `payment_request_token` backfill.
- **Five separate DBs.** `db.Transaction()` will not span them. A write that must be atomic across domains — say, updating a user and inserting a notification — cannot be transactional today.
- **No FK constraints in SQLite config** beyond what GORM generates. Deletion cascades rely on GORM's `OnDelete:CASCADE` tags — and there are a few entities where this is **not** set (e.g., `Delivery` references `Project` without cascade). Dropping a project while deliveries exist will fail at the DB level on Postgres; on SQLite it will silently orphan.

## Build & deploy

- **Dockerfile hardcodes `GOARCH=arm64`.** The container runs only on ARM. If you change this, update the Dockerfile and the Woodpecker config simultaneously.
- **Woodpecker redeploy maps `main` → prod.** There is no confirmation step. A rushed merge ships to production in 2–3 min.
- **`.env.defaults` is copied into the image.** Do not put secrets there.
- **`JWT_SECRET` default is literally `"change-me-in-production"`.** A prod deploy without an override is vulnerable. There is no startup check that enforces a real secret.
