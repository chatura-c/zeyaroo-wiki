# phto-api Refactor Plan: Split Large Files

## Goal

Split 15 files over 500 lines into focused, smaller files using the patterns already
established in the codebase (`escrow_payment_request.go`, `notification_amendment_handlers.go`, etc.).

No behavior changes — pure mechanical reorganization.

---

## Sub-packages: Where and Why

### Handlers → sub-packages (recommended)

Handlers have zero inter-handler dependencies. Each handler struct only injects services,
so moving each domain to its own sub-package is clean:

```
internal/handlers/
  escrow/         package escrow
  media/          package media
  proposal/       package proposal
  project/        package project
  share/          package share
  amendment/      package amendment
  ...
  router.go       assembles all sub-packages
```

`router.go` imports each sub-package and wires services into them. The `Handlers` aggregate
struct becomes typed references to each sub-package's `Handler` type.

### Services → file-split only (not sub-packages)

Services hold concrete cross-service references (e.g. `EscrowService` stores
`*AmendmentService`, `*DeliveryService`). Sub-packaging would require extracting interfaces
for every cross-service dependency — a separate, larger refactor. For now: split into
sibling files within the same `services` package.

---

## Conventions

- New files in the same package = same `package handlers` / `package services` declaration
- Struct definition + constructor always stay in the primary file (e.g. `escrow.go`)
- No API changes, no struct changes
- After each session: `go build ./...` to verify

---

## Sessions

Work through these independently in any order. Each is self-contained.

---

### Session 1 — Escrow Service ✅

**File:** `internal/services/escrow.go` (1561 lines)

Split into 4 files (keeping `escrow_payment_request.go` as-is):

| File | What goes there |
|---|---|
| `escrow.go` | Struct, constructor, setters, `toUserID`; core lifecycle: `FundMilestone`, `StartMilestoneWork`, `CompleteMilestoneWork`, `UnlockMilestone`, `RequestMilestoneChanges`, `ReleaseMilestonePayment`, `autoRejectAmendmentsForMilestone`, `ResolveDispute`, `RefundMilestonePayment`, `RefundEscrow`, `DisputeMilestone` |
| `escrow_queries.go` (new) | `GetMilestoneEscrow`, `GetProjectMilestones`, `removeMilestoneWatermarks`, `autoCompleteProject`, `GetMilestoneWithContext`, `GetUserCurrency`, `GetUser`, `getUser`, `GetMilestoneFeedback` |
| `escrow_auto_release.go` (new) | `ExtendMilestoneDeadlines`, `CheckNonPaymentAndExtend`, `AutoReleaseEligibleMilestones`, `releaseMilestonePaymentTx` |
| `escrow_manual_funding.go` (new) | `HandleDodoPaymentsWebhook`, `ApproveManualFunding`, `MarkManuallyFunded`, `SetManualReceiptURL` |

---

### Session 2 — Escrow Handler ✅

**File:** `internal/handlers/escrow.go` (1373 lines)

Option A (file-split only): split into 2 files.
Option B (sub-package): move to `internal/handlers/escrow/` — see sub-package section above.

| File | What goes there |
|---|---|
| `escrow.go` | Struct, constructor, setters, response types + `milestoneToDetailResponse`; core lifecycle handlers: `GetMilestone`, `FundMilestone`, `StartMilestone`, `CompleteMilestone`, `UnlockMilestone`, `RequestChanges`, `ReleaseMilestone`, `DisputeMilestone`, `GetDeliveryAlbum`, `SubmitDelivery`, `GetProjectMilestones`, `GetMilestoneFeedback`, `AdminResolveDispute` |
| `escrow_payment.go` (new) | `RequestManualFunding`, `ApproveManualFunding`, `RequestFunding`, `ResendFundRequest`, `MarkMilestonePaid`, `GetPublicPaymentPage`, `PublicSelectPaymentMethod`, `PublicUploadReceipt`, `MarkManuallyFunded`, `ServeReceipt` |

---

### Session 3 — Media Handler + Media Service

**Files:** `internal/handlers/media.go` (1148 lines), `internal/services/media.go` (760 lines)

#### Handler → 5 files

| File | What goes there |
|---|---|
| `media.go` | Struct, constructor, setters, response types, URL helpers (`mediaURL`, `thumbnailURL`, `buildMediaResponse`), `Upload`, `Get`, `ListByProject`, `List`, `Delete` |
| `media_serve.go` (new) | `Download`, `ServeFile`, `ServeVariant`, `ServeVariantFile` |
| `media_albums.go` (new) | `AddToAlbums`, `RemoveFromAlbum` |
| `media_direct_upload.go` (new) | `InitiateUpload`, `CompleteUpload` |
| `media_delivery.go` (new) | `ListMyMedia`, `GetPortfolioMedia`, `GetCreatorPortfolioMedia`, `splitIDs`, `GetDeliveryPreview`, `DownloadAll`, `UpdateCullStatus`, `UpdateFavorite`, `PurgeRejected` |

#### Service → 3 files

| File | What goes there |
|---|---|
| `media.go` | Struct, constructor, `Upload`, `GetByID`, `ListByProject`, `ListByProjectPaginated`, `ListByAlbum`, `Delete`, `PurgeRejected`, `GetFilePath`, `ResolveVariantPath`, `GetProjectID`, `SetWatermarked`, `AddToAlbum`, `RemoveFromAlbum`, `AddToMultipleAlbums`, `GetProjectStorageSize`, `ListByOwner`, `GetByIDs` |
| `media_direct_upload.go` (new) | `InitiateDirectUpload`, `CompleteDirectUpload` |
| `media_delivery.go` (new) | `GetClientDeliveryAccess`, `GetDeliveryPreviewMedia`, `StreamProjectZip`, `emitUploadEvent`, `UpdateCullStatus`, `UpdateFavorite` |

---

### Session 4 — Proposal Handler + Proposal Service

**Files:** `internal/handlers/proposal.go` (987 lines), `internal/services/proposal.go` (726 lines)

#### Handler → 2 files

| File | What goes there |
|---|---|
| `proposal.go` | Struct, constructor, setters, response types + mapping (`proposalToResponse`, `versionToResponse`), `Get`, `CreateVersion`, `CreateDirect`, `UpdateCoverImage`, `Accept`, `Cancel`, `CreateProject`, `GetVersionHistory`, `ListForCreator`, `ListForClient` |
| `proposal_milestones.go` (new) | `UpdateMilestone`, `GetClientDeliveryProject`, `FundMilestone`, `CompleteMilestone`, `ReleaseMilestone` |

#### Service → 2 files

| File | What goes there |
|---|---|
| `proposal.go` | Struct, constructor, setters, `GetByID`, `GetByInquiryID`, `GetByAccessToken`, `CreateVersion`, `verifyCanCreateVersion`, `processMilestones`, `AcceptAsGuest`, `Accept`, `createProjectInTx`, `Cancel`, `CreateProjectFromProposal`, `GetVersionHistory`, `GetVersion`, `ListByCreator`, `ListByClient` |
| `proposal_milestones.go` (new) | `GetMilestone`, `FundMilestone`, `CompleteMilestone`, `ReleaseMilestone`, `UpdateMilestone`, `UpdateCoverImage` |

---

### Session 5 — Project Handler + Project Service

**Files:** `internal/handlers/project.go` (831 lines), `internal/services/project.go` (603 lines)

#### Handler → 3 files

| File | What goes there |
|---|---|
| `project.go` | Struct, constructor, setters, helpers (`coverURL`, `coverMediaID`, `formatBytes`), `Create`, `Get`, `Update`, `Delete`, `SetCover`, `List`, `AddCollaborator`, `RemoveCollaborator`, `GetStorageSize` |
| `project_transfer.go` (new) | `Transfer`, `InitiateTransfer`, `AcceptTransfer`, `DeclineTransfer`, `GetPendingTransfers` |
| `project_lifecycle.go` (new) | `Archive`, `RestoreFromArchive`, `SoftDelete`, `RestoreFromTrash`, `ListTrashed`, `ForeverGallery` |

#### Service → 3 files

| File | What goes there |
|---|---|
| `project.go` | Struct, constructor, `Create`, `GetByID`, `SetCoverIfEmpty`, `SetCover`, `CanAccess`, `IsOwner`, `Update`, `Delete`, `List`, `AddCollaborator`, `RemoveCollaborator` |
| `project_transfer.go` (new) | `transferOwnership`, `Transfer`, `InitiateTransfer`, `AcceptTransfer`, `DeclineTransfer`, `GetPendingTransfersForUser`, `hasFundedMilestones` |
| `project_lifecycle.go` (new) | `Archive`, `Restore`, `SoftDelete`, `RestoreFromTrash`, `ListTrashed`, `PurgeExpiredTrash`, `IsReadyForHandover`, `CreateForeverGalleryCheckout`, `MarkProjectProtected` |

---

### Session 6 — Share Handler

**File:** `internal/handlers/share.go` (827 lines)

| File | What goes there |
|---|---|
| `share.go` | Struct, constructor, setters, `shareRateLimiter` + helpers, response types + mapping helpers, `Create`, `ListByProject`, `Get`, `Update`, `Delete`, `RefreshExpiration` |
| `share_public.go` (new) | `GetPublic`, `VerifyPassword`, `ListAlbums`, `ListMedia`, `GetMedia`, `ServeMediaFile` |

---

### Session 7 — Amendment Handler + Amendment Service

**Files:** `internal/handlers/amendment.go` (538 lines), `internal/services/amendment.go` (634 lines)

#### Handler → 2 files

| File | What goes there |
|---|---|
| `amendment.go` | Struct, constructor, response types + mapping, `CreateDraft`, `ListAmendments`, `GetAmendment`, `UpdateAmendmentNote`, `AddDraftMilestone`, `UpdateDraftMilestone`, `DeleteDraftMilestone`, `handleAmendmentError` |
| `amendment_lifecycle.go` (new) | `PublishAmendment`, `CancelAmendment`, `ApplyAmendment`, `RejectAmendment`, `GetPublicAmendment`, `ApplyPublicAmendment`, `RejectPublicAmendment` |

#### Service → 2 files

| File | What goes there |
|---|---|
| `amendment.go` | Struct, constructor, setters, `getActiveProposalForProject`, `CreateDraft`, `GetByID`, `GetByToken`, `ListByProject`, `UpdateNote`, `UpdateDraftMilestone`, `AddDraftMilestone`, `DeleteDraftMilestone` |
| `amendment_lifecycle.go` (new) | `PublishAmendment`, `ApplyAmendment`, `RejectAmendment`, `CancelAmendment`, `AutoRejectPendingAmendments`, `assertCreatorOwns`, `recalcDraftTotal`, `validateEscrowSafeguard`, `isFundedOrActive` |

---

### Session 8 — Misc (Notification Email, Router, Storage Service)

#### `internal/services/notification_email_channel.go` (526 lines) → 3 files

| File | What goes there |
|---|---|
| `notification_email_channel.go` | Struct, constructor, `HandleInquiryReceived`, `HandleProposalVersionSubmitted`, `HandleProposalAccepted`, `HandleMilestoneFunded`, `loadMilestoneForNotification` |
| `notification_email_work.go` (new) | `HandleMilestoneWorkSubmitted`, `HandlePaymentReleased`, `HandleCRMProposalReminder`, `HandleCRMDeliveryFeedback` |
| `notification_email_brief.go` (new) | `HandleBriefRequestSent`, `HandleBriefAnswered` |

#### `internal/handlers/router.go` (513 lines) → 2 files

| File | What goes there |
|---|---|
| `router.go` | `Handlers` struct, handler construction, service wiring (all the `SetXxx` calls) |
| `routes.go` (new) | All route registration extracted into `registerRoutes(r chi.Router, h *Handlers, ...)` called from `NewRouter` |

#### `internal/services/storage.go` (503 lines) → 2 files

| File | What goes there |
|---|---|
| `storage.go` | Struct, constructor, `StorageUsage`, `GetUsage`, `calculateUsedStorage`, `calculateProjectStorage`, `LogUsage`, `GetUsageHistory` |
| `storage_subscription.go` (new) | `Subscribe`, `SwitchMode`, `calculateProration`, `AdjustReservedStorage`, `TransferStorageCost` |
