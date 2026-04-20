# 4. Data Model

## Core entities

The domain breaks into six clusters. Listed with the entities they contain and the one-line purpose of each.

### Identity & ownership
- `User` — platform account (creator / client / admin)
- `CreatorProfile` — 1:1 public profile for creators (slug, bio, studio project link)
- `UserPackage` — per-user limit overrides (admin deals)

### Work container
- `Project` — central workspace; soft-deletable, archivable, can be a system "studio" project or a delivery project
- `ProjectCollaborator` — composite-PK join table for access control
- `Album` — named media grouping inside a project (milestone album, delivery album, system album)
- `MediaObject` — uploaded photo/video; M2M with Album via `media_albums`
- `Questionnaire` — 1:1 client-facing form per project
- `BriefRequest` / `BriefPoint` — structured shot-list questionnaire

### Negotiation
- `Inquiry` — first client contact (kanban pipeline status)
- `Proposal` — head record per inquiry; has a state machine
- `ProposalVersion` — immutable snapshot per negotiation round
- `Milestone` — work segment with payment and deliverable on a version
- `ProjectAmendment` — proposed mid-project change; references draft and previous versions
- `ProjectTransferRequest` / `ProjectHandover` — creator↔creator and creator→client ownership moves

### Money
- `EscrowTransaction` — held/released/refunded funds per milestone
- `EscrowDispute` — admin-managed dispute
- `TransactionLedger` — append-only double-entry ledger
- `Invoice` / `InvoiceLineItem` / `PaymentTransaction` — platform SaaS billing
- `PayoutRequest` — creator withdrawal
- `PaymentMethod` / `BillingAddress` — stored for billing

### Delivery & sharing
- `Delivery` — immutable audit record per released milestone (maps creator media → client media)
- `Share` — public password-protected gallery link; M2M with Album via `share_albums`
- `Review` — post-project rating
- `MilestoneFeedback` — comment thread on milestone submission (soft-deletable)

### Infrastructure (separate DBs)
- `StoredEvent`, `Job` — durable event/job store (events DB)
- `Notification` — in-app notification row (notifications DB)
- `WebhookSource`, `WebhookAttempt` — inbound webhook config and log (webhooks DB)
- `StorageSubscription`, `StorageUsageLog`, `StoragePlanChange` — storage billing (storage DB)
- `VerificationCode` — email OTP

## ER diagram

```mermaid
erDiagram
  User {
    uuid ID PK
    string FirebaseUID UK
    string Email UK
    string Role
    string SubscriptionTier
    int64 FreeStorageQuotaBytes
    string Currency
    datetime SubscriptionExpiresAt
  }
  CreatorProfile {
    uuid ID PK
    uuid UserID FK
    string Slug UK
    string DisplayName
    uuid StudioProjectID FK
  }
  UserPackage {
    uuid UserID PK
    int64 StorageBytes
    int MaxProjects
    float64 FeePercent
    int64 MonthlyPriceCents
    string PaymentMethods
  }
  Project {
    uuid ID PK
    uuid OwnerID FK
    string Status
    bool IsSystem
    bool IsDeliveryProject
    bool IsArchived
    uuid SourceProposalID FK
    uuid CoverMediaID FK
    datetime ExpiresAt
    datetime DeletedAt
  }
  ProjectCollaborator {
    uuid ProjectID PK
    uuid UserID PK
    string Role
  }
  Album {
    uuid ID PK
    uuid ProjectID FK
    bool IsSystem
    bool IsMilestoneAlbum
    bool IsDeliveryAlbum
    bool IsLocked
    uuid MilestoneID FK
  }
  MediaObject {
    uuid ID PK
    uuid ProjectID FK
    uuid UploaderID FK
    string UploadStatus
    string ProcessingStatus
    string CullStatus
    bool IsWatermarked
    int64 FileSize
  }
  MediaAlbum {
    uuid MediaID PK
    uuid AlbumID PK
  }
  Inquiry {
    uuid ID PK
    uuid ClientID FK
    uuid CreatorID FK
    string GuestEmail
    string PipelineStatus
  }
  Proposal {
    uuid ID PK
    uuid InquiryID FK
    uuid ProjectID FK
    int CurrentVersion
    string Status
    string AccessToken UK
  }
  ProposalVersion {
    uuid ID PK
    uuid ProposalID FK
    int Version
    uuid AuthorID FK
    string AuthorRole
    int64 TotalAmount
  }
  Milestone {
    uuid ID PK
    uuid ProposalVersionID FK
    int Sequence
    string Status
    int64 AmountCents
    int64 BalanceDueCents
    string DeliverableType
    bool IsFinal
    string PaymentRequestToken UK
  }
  EscrowTransaction {
    uuid ID PK
    uuid MilestoneID FK
    uuid PayerID FK
    uuid PayeeID FK
    string Status
    int64 AmountCents
    int64 PlatformFeeCents
    int64 CreatorPayoutCents
    string GatewayName
  }
  EscrowDispute {
    uuid ID PK
    uuid EscrowTransactionID FK
    uuid InitiatorID FK
    string Status
  }
  TransactionLedger {
    uuid ID PK
    uuid UserID FK
    uuid MilestoneID FK
    string Type
    string Status
    int64 AmountCents
  }
  ProjectAmendment {
    uuid ID PK
    uuid ProjectID FK
    uuid ProposalID FK
    uuid DraftVersionID FK
    uuid PreviousVersionID FK
    string Status
    string AmendmentToken UK
  }
  Share {
    uuid ID PK
    uuid ProjectID FK
    uuid CreatorID FK
    string Token UK
    string PasswordHash
    bool IsActive
    int PreviewHqCount
  }
  ShareAlbum {
    uuid ShareID PK
    uuid AlbumID PK
  }
  Delivery {
    uuid ID PK
    uuid MilestoneID FK
    uuid AlbumID FK
    uuid SourceProjectID FK
    uuid ClientProjectID FK
    uuid ClientID FK
    uuid CreatorID FK
    int64 TotalBytes
  }
  MilestoneFeedback {
    uuid ID PK
    uuid MilestoneID FK
    uuid AuthorID FK
    datetime DeletedAt
  }
  Invoice {
    uuid ID PK
    uuid UserID FK
    string InvoiceNumber UK
    string Status
    int64 TotalCents
    datetime DueAt
  }
  InvoiceLineItem {
    uuid ID PK
    uuid InvoiceID FK
    string Type
    int64 AmountCents
  }
  PaymentTransaction {
    uuid ID PK
    uuid InvoiceID FK
    uuid UserID FK
    string Status
    int64 AmountCents
  }
  PayoutRequest {
    uuid ID PK
    uuid UserID FK
    string Status
    int64 AmountCents
  }
  StorageSubscription {
    uuid ID PK
    uuid UserID FK
    string Type
    int64 QuotaBytes
    int64 UsedBytes
  }
  StorageUsageLog {
    uuid ID PK
    uuid UserID FK
    datetime Date
    int64 UsedBytes
  }
  BriefRequest {
    uuid ID PK
    uuid ProjectID FK
    string Token UK
    string Status
  }
  BriefPoint {
    uuid ID PK
    uuid BriefRequestID FK
    string Type
    int Order
  }
  Questionnaire {
    uuid ID PK
    uuid ProjectID FK
  }
  Notification {
    uuid ID PK
    uuid UserID FK
    string Type
    bool Read
  }

  User ||--o| CreatorProfile : "has"
  User ||--o| UserPackage : "overrides"
  User ||--o{ Project : "owns"
  User ||--o{ ProjectCollaborator : "participates"
  User ||--o{ Inquiry : "client of"
  User ||--o{ Inquiry : "creator of"
  User ||--o{ Share : "creates"
  User ||--o{ Invoice : "billed"
  User ||--o{ PayoutRequest : "requests"
  User ||--o{ Notification : "receives"
  User ||--o{ StorageSubscription : "has"
  User ||--o{ StorageUsageLog : "logs"
  CreatorProfile }o--o| Project : "studio_project"

  Project ||--o{ Album : "contains"
  Project ||--o{ ProjectCollaborator : "has"
  Project ||--o{ Proposal : "scoped by"
  Project ||--o{ Share : "shared via"
  Project ||--o{ BriefRequest : "has"
  Project ||--o| Questionnaire : "has"

  Inquiry ||--o| Proposal : "generates"
  Proposal ||--o{ ProposalVersion : "versioned by"
  ProposalVersion ||--o{ Milestone : "defines"

  Milestone ||--o{ EscrowTransaction : "funded by"
  Milestone ||--o{ TransactionLedger : "ledgered"
  Milestone ||--o{ MilestoneFeedback : "commented"
  Milestone ||--o| Delivery : "delivered via"

  EscrowTransaction ||--o{ EscrowDispute : "disputed"

  Album }o--o{ MediaObject : "media_albums"
  Share }o--o{ Album : "share_albums"

  Invoice ||--o{ InvoiceLineItem : "itemized"
  Invoice ||--o{ PaymentTransaction : "paid via"

  BriefRequest ||--o{ BriefPoint : "contains"

  ProjectAmendment }o--|| ProposalVersion : "draft"
  ProjectAmendment }o--|| ProposalVersion : "previous"
  ProjectAmendment }o--|| Project : "amends"
```

## State machines

Enum values are defined in [internal/models](phto-api/internal/models/). Statuses are strings, not DB enums — validation is in the service layer.

### Project
`inquiry` → `negotiation` → `active` → `completed` | `cancelled`

### Proposal
```mermaid
stateDiagram-v2
  [*] --> inquiry
  inquiry --> negotiating
  inquiry --> canceled
  negotiating --> pending
  pending --> negotiating
  pending --> finalized
  negotiating --> finalized
  negotiating --> canceled
  pending --> canceled
  finalized --> [*]
  canceled --> [*]
```

### Milestone
```mermaid
stateDiagram-v2
  [*] --> pending
  pending --> awaiting_funds : creator requests funding
  pending --> funded : admin/direct fund
  awaiting_funds --> partially_funded : price rose via amendment
  awaiting_funds --> funded : fully paid
  partially_funded --> funded : top-up paid
  funded --> in_progress : creator starts work
  in_progress --> submitted : creator submits
  submitted --> completed : client releases OR 7d auto-release
  submitted --> disputed : client disputes
  disputed --> completed : admin resolves → release
  disputed --> pending : admin resolves → refund (rare)
  completed --> [*]
```

### Escrow
`pending` → `held` → `released` | `refunded` | `disputed`

### Amendment
`draft` → `pending_approval` → `applied` | `rejected` | `cancelled`

### Invoice
`draft` → `finalized` → `pending_approval` → `paid` | `overdue` | `canceled`

## Migration strategy

- **No external migration tool.** GORM `AutoMigrate` runs on all five DBs at every startup, via `database.Migrate`, `MigrateEvents`, `MigrateNotifications`, `MigrateWebhooks`, `MigrateStorage`. Foreign keys are disabled during the call to avoid ordering issues.
- **Additive changes are safe.** New tables, columns, and indexes "just work" after a redeploy.
- **Destructive changes are not safe.** Column rename / type change / drop → write the SQL by hand. For that reason there is one embedded backfill in `Migrate()`: the `payment_request_token = NULL` update at [database.go:127](phto-api/internal/database/database.go#L127), run before the new unique index was added.
- **Seeds.** `SeedAdmin()` creates the admin user once at boot from `ADMIN_EMAIL` / `ADMIN_PASSWORD`. No other seed data.
- **One-shot backfill script** lives at [cmd/migrate-studio/main.go](phto-api/cmd/migrate-studio/main.go) — creates "My Studio" project and "Portfolio" album for legacy creator profiles. Run manually.

## Multi-tenancy & scoping

There is **no schema-per-tenant**. All users share the same tables. Scoping is by `OwnerID` / `CreatorID` at the query level.

- **Projects** scope by `OwnerID` plus presence in `project_collaborators`.
- **Albums / MediaObjects / BriefRequests / Questionnaires / Shares** transitively scope through `ProjectID`.
- **Inquiries** scope by `CreatorID`; `ClientID` is nullable so guests can inquire via `GuestEmail`.
- **Guests** appear in two tables: `Inquiry.ClientID IS NULL` and `EscrowTransaction.PayerID IS NULL`. Both have `GuestEmail` text fallbacks. Guest sessions are OTP-driven with a 24 h JWT where `role = guest`.

## Soft delete, archive, trash

| Entity | Mechanism |
|--------|-----------|
| `Project` | `gorm.DeletedAt` + `IsArchived` + `ExpiresAt` + `IsProtected` |
| `MilestoneFeedback` | `gorm.DeletedAt` |
| `MediaObject` | `CullStatus = reject` acts as trash; `PurgeRejected()` hard-deletes |
| `Album` | `IsLocked` freezes (used by delivery albums) |
| `Share` | `IsActive` + `ExpiresAt` for deactivation without deletion |

The `purge-trash` scheduler job hard-deletes soft-deleted projects past retention. Deletion cascade order matters — see [Gotchas](phto-api/gotchas.md).
