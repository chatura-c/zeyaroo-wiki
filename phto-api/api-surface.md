# 5. API Surface

All routes are rooted at `/api/v1`. The only exception is `GET /health`, which lives at the router root.

## Auth model

Four token types coexist:

| Token | Issuer | Lifetime | Claims |
|-------|--------|----------|--------|
| Platform JWT | `/auth/login`, `/auth/register`, `/auth/refresh`, `/auth/firebase/*` | 24 h | `UserID`, `Role` |
| Firebase ID token | Firebase client SDKs | per Firebase | Firebase UID |
| Share access JWT | `/public/shares/{token}/verify` | 1 h | `ShareID`, `ShareToken` |
| Guest JWT | `/auth/guest/verify-otp`, `/public/proposals/{token}/otp/verify` | 24 h | `Email`, `Role=guest` |

The middleware `CombinedAuthenticate` tries the presented bearer token as a platform JWT first; if that fails and Firebase is configured, it falls through to `FirebaseAuthMiddleware.Authenticate`, which verifies the Firebase ID token via the Admin SDK and looks up the user by Firebase UID. This means clients can send either token type on any protected route.

**Roles:** `creator`, `client`, `admin`. Admin routes use `RequireRole("admin")` stacked after auth. There is no plan/tier middleware — tier gating is inside the service layer via `FeatureGate` and `tiers.ResolveUserLimits`.

**Admin seed:** `ADMIN_EMAIL` / `ADMIN_PASSWORD` from env vars at first boot (defaults: `admin@zeyaroo.com` / `admin123`).

## Rate limits

Four per-IP token-bucket limiters on the public surface:

| Limiter | Burst | Applies to |
|---------|-------|------------|
| `authLimiter` | 10/s | `/auth/register`, `/auth/login`, `/auth/refresh`, `/auth/firebase/*` |
| `otpLimiter` | 5 / 2 s | OTP request/verify paths |
| `claimLimiter` | 5 / 12 s | `POST /projects/{id}/claim` |
| `payLimiter` | 10 / 6 s | `/public/pay/{token}*` |

In-memory; per-IP; cleaned every 5 min. `X-Forwarded-For` is honored, which matters because Caddy proxies.

**Authenticated routes are not rate-limited.** If you need to protect a handler from a logged-in abuser, add it inline.

## Versioning

`v1` is a URL prefix. There is no negotiation header, no `Accept-Version`. A `v2` prefix would be a fresh mux, introduced when we need a breaking change.

## Public (unauthenticated) routes

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/health` | 200 OK |
| GET | `/api/v1/portfolio-media` | All portfolio media (homepage) |
| GET | `/api/v1/media/{id}/file` | Signed URL media serve |
| GET | `/api/v1/media/{id}/variant/{type}/file` | Signed URL variant serve (`thumb`, `medium`) |
| POST | `/api/v1/creators/{id}/inquire` | Create inquiry (optional auth) |
| GET | `/api/v1/creators`, `/creators/slug/{slug}`, `/creators/{id}` | Creator discovery |
| GET | `/api/v1/creators/{id}/portfolio` | Portfolio media for a creator |
| POST | `/api/v1/projects/{id}/claim` | Guest client creates account and takes project ownership |
| GET/POST | `/api/v1/public/brief/{token}[/respond]` | Brief view / answer |
| GET/POST | `/api/v1/public/pay/{token}[/select-method|/upload-receipt]` | Public payment page |
| GET/POST | `/api/v1/public/amendments/{atoken}[/apply|/reject]` | Amendment view / act |
| GET/POST | `/api/v1/public/proposals/{token}[/otp/...|/accept|/questionnaire/responses]` | Public proposal + guest accept |
| GET/POST | `/api/v1/public/shares/{token}[/verify]` | Share landing page + password |
| GET | `/api/v1/public/shares/{token}/albums|media|media/{id}[/file]` | Gated by share access JWT |
| POST | `/api/v1/webhooks/{source_name}` | Inbound webhook (auth per-provider) |

See [handlers/routes.go](phto-api/internal/handlers/routes.go) for the full list; anything not above requires authentication.

## Authenticated routes

Every non-public path under `/api/v1` uses `CombinedAuthenticate`. Grouped by domain:

| Group | Shape |
|-------|-------|
| `/users`, `/me`, `/users/{id}/notification-settings` | Profile, settings, earnings |
| `/projects[/:id/...]` | CRUD, cover, archive, trash, forever-gallery, collaborators, transfer, handover |
| `/projects/{id}/albums`, `/albums/{id}` | Album CRUD, media listing |
| `/media[/:id]` + `/initiate-upload` + `/complete-upload` | Direct upload, CRUD, cull, favorite, album membership |
| `/proposals[/:id]` | Version history, updates, accept, cancel |
| `/milestones/{id}` | Fund, start, complete, submit, release, dispute, feedback |
| `/inquiries[/:id]` | List, update pipeline |
| `/shares[/:id]` | CRUD, expiration refresh |
| `/amendments[/:id]` | Drafts, publish, cancel, apply, reject |
| `/templates[/:id]` | Template CRUD |
| `/briefs[/:id]` | Brief creation, resend |
| `/questionnaires/{id}` | Schema, responses |
| `/notifications` | List, unread count, mark read |
| `/billing/*` | Invoice, subscription, upgrade, payout |
| `/storage/*` | Usage, subscribe, mode switch |

## Admin routes

Stacked with `RequireRole("admin")`. Used by `zeyaroo-admin`.

- `PATCH /admin/users/{id}/tier`
- `GET/PUT /admin/users/{id}/package`
- `GET /admin/stats`, `/admin/users`, `/admin/creators/{id}/earnings`
- `GET/PATCH/POST /admin/invoices/*` (approve, reject, line items, discount, send)
- `GET/PATCH /admin/payout-requests`
- `GET /admin/escrow/transactions`, `/admin/disputes`
- `POST /milestones/{id}/resolve` (admin dispute resolution)
- `GET /admin/projects`, `/admin/jobs`, `/admin/events`, `/admin/webhook-attempts`, `/admin/storage/stats`
- `GET/POST/PUT/DELETE /webhooks/sources/{id}` (manage webhook source configs)
- **Dev-mode only** (requires `APP_ENV=development` + admin): `/admin/dev/clock[/advance|/reset]`

## Webhooks (inbound)

`POST /api/v1/webhooks/{source_name}` — no JWT middleware, verification is per-provider:

| Source | Verification | Handled events |
|--------|--------------|----------------|
| `dodopayments` | HMAC-SHA256 on `X-Dodo-Signature` / `X-Webhook-Signature` with `DODO_WEBHOOK_SECRET` | `payment.success`, `subscription.created`, `subscription.renewed`, `subscription.cancelled` |
| `payhere` | MD5 signature on form-POST IPN; sandbox mode bypasses | Status code 2 → `MarkInvoicePaidByGateway` on the billing invoice |
| *other* | Looked up via `WebhookSource` DB record; extensible `Authenticator` (`hmac256`, `apikey`, `basic`, `none`) | Raw event stored, EventProcessor fan-out |

Every inbound call is logged to `webhook_attempts` regardless of outcome.

## Response shape

Responses use helpers in [handlers/response.go](phto-api/internal/handlers/response.go):

- Success → `respondJSON(w, 200, data)` — bare JSON payload, no envelope.
- Error → `respondError(w, status, message)` → `{"error": "<Status Text>", "message": "<message>"}`.
- 500 → `respondInternalError(w, message, err)` — includes `err.Error()` only when `DevMode` is true.

## Media URL signing

Media files are served behind HMAC-signed URLs so `<img>` / `<video>` tags can load without an `Authorization` header. URLs look like `/api/v1/media/{id}/file?exp={unix}&sig={hex}`. Expiry is aligned to TTL boundaries (24 h default), so a browser-cached image stays valid for the full window. See [internal/mediasign/signer.go](phto-api/internal/mediasign/signer.go). If `MEDIA_CDN_URL` is set, URLs are absolute CDN paths.
