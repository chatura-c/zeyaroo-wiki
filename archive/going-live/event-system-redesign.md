---
name: Event system redesign — fan-out jobs
description: Planned refactor of the event/pub-sub system to use per-handler job rows for independent retry
type: project
---

## Decision
Replace the current broken event system with a fan-out jobs table where each handler gets its own DB row with independent retry state.

**Why:** Current `PersistInternalEvent` stores events as `status=completed` so the processor never picks them up. Even if fixed, shared retry state means one failed handler (e.g. email SMTP down) blocks or pollutes retry for unrelated handlers (e.g. WhatsApp).

**How to apply:** Follow the implementation steps below in a fresh session.

---

## Implementation Steps

### 1. New `jobs` table (migration)
```sql
jobs (
  id uuid PK,
  event_type text,        -- e.g. "inquiry.received"
  handler    text,        -- e.g. "notify_email" (stable identifier)
  payload    jsonb,
  status     text,        -- pending | processing | completed | failed
  retry_count int default 0,
  max_retries int default 5,
  next_retry_at timestamptz,
  last_error  text,
  created_at  timestamptz,
  processed_at timestamptz
)
```
Add index on `(status, next_retry_at)` for the polling query.

### 2. `JobRepository`
CRUD over the jobs table:
- `CreateJobs(jobs []*Job) error`
- `GetPendingJobs(limit int) ([]Job, error)` — fetch where `status=pending AND (next_retry_at IS NULL OR next_retry_at <= now())`
- `MarkCompleted(id)`, `MarkFailed(id, err)`, `IncrementRetry(id, nextRetryAt)`

### 3. Update `EventProcessor` — add handler registry + fan-out

Replace current `handlers map[string][]EventProcessorFunc` with a registry that also tracks handler names:

```go
type handlerEntry struct {
    name    string
    fn      EventProcessorFunc
}
handlers map[string][]handlerEntry
```

Add `Publish(eventType string, payload any) error`:
- Marshal payload to JSON
- Look up registered handlers for eventType
- Insert one job row per handler (batch insert)

Update `RegisterHandler(eventType, handlerName string, fn EventProcessorFunc)` — name is now required (used as stable `handler` column value).

### 4. Update `processPendingEvents` to poll jobs table
- Fetch pending jobs from `jobRepo` instead of `eventRepo`
- Dispatch to handler by looking up `job.handler` in registry
- Independent retry per job row (existing backoff logic applies unchanged)

### 5. Update `services.go` registrations
Add handler names:
```go
eventProcessor.RegisterHandler("inquiry.received", "notify_email",    emailChan.HandleInquiryReceived)
eventProcessor.RegisterHandler("inquiry.received", "notify_inapp",    inAppChan.HandleInquiryReceived)
eventProcessor.RegisterHandler("inquiry.received", "notify_whatsapp", whatsAppChan.HandleInquiryReceived)
```

### 6. Update publishing call sites
Replace `eventStorage.PersistInternalEvent(ev)` with `eventProcessor.Publish("inquiry.received", ev)` in:
- `inquiry.go` (lines 123, 215)
- `proposal.go` (lines 229, 362)
- `storage_event_handler.go` (lines 35, 98)

### 7. Cleanup job
Add a periodic cleanup (can reuse existing ticker or a separate cron) to delete completed jobs older than 7 days.

### 8. Remove / deprecate old paths
- `EventStorageService.PersistInternalEvent` — no longer needed for domain events
- `publishToEventBus` fallback in `event_processor.go` — remove once all events go through jobs
- `EventBus` — can be kept for in-process signals that don't need durability, or removed entirely

---

## Key constraints
- Handler names are stable identifiers — treat like DB column values, rename requires migration
- New handlers only apply to events fired after registration (no automatic backfill)
- Existing `StoredEvent` table / webhook flow is unaffected — webhooks still use the old path
