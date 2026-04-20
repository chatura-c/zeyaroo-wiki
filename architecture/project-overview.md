# Phto - Project Overview Document

**Document Version:** 1.0
**Last Updated:** February 2026
**Purpose:** Onboarding document for project managers and stakeholders

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Project Scope](#2-project-scope)
3. [Target Users](#3-target-users)
4. [Core Features & Modules](#4-core-features--modules)
5. [Technical Architecture](#5-technical-architecture)
6. [Domain Model & Data Flow](#6-domain-model--data-flow)
7. [Current Implementation Status](#7-current-implementation-status)
8. [Known Gaps & Technical Debt](#8-known-gaps--technical-debt)
9. [Recommended Next Steps](#9-recommended-next-steps)
10. [Risk Assessment](#10-risk-assessment)
11. [Glossary](#11-glossary)

---

## 1. Executive Summary

**Phto** is a creator-client media collaboration platform (SaaS) designed for photographers and videographers. The platform enables:

- **Project Management** for creative work
- **Proposal Negotiation** with full audit trails
- **Milestone-Based Payments** with escrow protection
- **Media Sharing** with watermarking until payment completion

### Business Model

Revenue streams:
1. **Storage Subscriptions** - Creators pay for reserved storage ($0.02/GB/month)
2. **Dynamic Storage** - Clients pay for on-demand storage ($0.10/GB/month)
3. **Transaction Fees** - Potential future revenue from escrow transactions

### Current State

The platform has a **functional MVP** with core workflows implemented end-to-end. Both backend and frontend are operational with the primary user journeys complete. The system is ready for alpha/beta testing with real users.

---

## 2. Project Scope

### In Scope

| Category | Features |
|----------|----------|
| **Authentication** | Email/password, Google OAuth, Facebook OAuth via Firebase |
| **User Roles** | Creator, Client, Admin |
| **Project Lifecycle** | Inquiry → Negotiation → Active → Completed/Cancelled |
| **Proposal System** | Multi-version negotiation with state machine |
| **Payments** | Milestone-based escrow with fund/release/refund/dispute |
| **Media** | Upload, organize (albums), watermark, share publicly |
| **Storage** | Tiered pricing (reserved/dynamic), usage tracking, billing |
| **Billing** | Invoice generation, payment processing, transaction history |

### Out of Scope (Current Phase)

- Mobile applications (iOS/Android)
- Real-time chat/messaging between users
- Video conferencing integration
- AI-powered image editing or enhancement
- Marketplace/discovery features beyond creator directory
- Multi-currency support (USD only)
- White-label/enterprise deployments

---

## 3. Target Users

### Creators (Photographers/Videographers)

**Profile:**
- Professional or semi-professional photographers and videographers
- Need to manage client relationships and project deliveries
- Want protection through milestone-based payments
- Need secure file sharing with watermarking

**Key Jobs-to-be-Done:**
- Showcase portfolio to attract clients
- Negotiate project terms professionally
- Get paid securely before delivering final assets
- Organize and share media with clients

### Clients (Individuals/Businesses)

**Profile:**
- Individuals or businesses hiring creative professionals
- Want transparency in project terms and pricing
- Need assurance of delivery before full payment
- Want easy access to their purchased media

**Key Jobs-to-be-Done:**
- Find and evaluate creators
- Negotiate project terms
- Pay securely with escrow protection
- Access and download final deliverables

---

## 4. Core Features & Modules

### 4.1 Authentication & User Management

| Feature | Status | Description |
|---------|--------|-------------|
| Email/Password Auth | ✅ Complete | Traditional registration and login |
| Google OAuth | ✅ Complete | Sign in with Google via Firebase |
| Facebook OAuth | ✅ Complete | Sign in with Facebook via Firebase |
| Role Selection | ✅ Complete | Users choose Creator or Client at registration |
| Creator Profiles | ✅ Complete | Bio, avatar, social links, service categories |
| Admin Role | ⚠️ Partial | Role exists but admin-specific features limited |

### 4.2 Creator Discovery

| Feature | Status | Description |
|---------|--------|-------------|
| Creator Directory | ✅ Complete | Browse all creators with search |
| Public Profiles | ✅ Complete | View creator info, categories, social links |
| Portfolio Gallery | ✅ Complete | View creator's portfolio media with lightbox |
| Send Inquiry | ✅ Complete | Contact creator (works for guests too) |

### 4.3 Project Management

| Feature | Status | Description |
|---------|--------|-------------|
| Create Projects | ✅ Complete | New project with title, description, dates |
| Project Dashboard | ✅ Complete | View all projects with status filtering |
| Media Management | ✅ Complete | Upload, organize, delete media |
| Album Organization | ✅ Complete | Create albums, assign media |
| Collaborators | ⚠️ Partial | Backend complete, frontend needs integration |
| Ownership Transfer | ✅ Complete | Request/accept/decline transfer workflow |
| "My Studio" | ✅ Complete | Auto-created project for creator portfolio |

### 4.4 Inquiry & Proposal Workflow

| Feature | Status | Description |
|---------|--------|-------------|
| Guest Inquiries | ✅ Complete | Clients can inquire without account |
| Authenticated Inquiries | ✅ Complete | Full inquiry with user association |
| Proposal Creation | ✅ Complete | Creator drafts proposal from inquiry |
| Version History | ✅ Complete | Immutable snapshots of each negotiation step |
| State Machine | ✅ Complete | inquiry → pending → negotiating → finalized/canceled |
| Milestone Definition | ✅ Complete | Up to 10 milestones per proposal |
| Counter-Proposals | ✅ Complete | Client can request changes |
| Accept/Finalize | ✅ Complete | Both parties can accept |
| Create Project | ✅ Complete | Generate project from accepted proposal |

**Proposal State Machine:**

```
        ┌─────────────────────────────────────┐
        │                                     │
        ▼                                     │
   [inquiry] ──► [pending] ──► [negotiating] ─┴──► [finalized]
        │            │              │
        │            │              │
        └────────────┴──────────────┴─────────► [canceled]
```

### 4.5 Escrow & Milestone Payments

| Feature | Status | Description |
|---------|--------|-------------|
| Sequential Funding | ✅ Complete | Milestones must be funded in order |
| Escrow Hold | ✅ Complete | Payments held until work completion |
| Start Work | ✅ Complete | Creator marks milestone as in-progress |
| Complete Work | ✅ Complete | Creator marks work as done |
| Release Funds | ✅ Complete | Client releases payment to creator |
| Auto-Release | ⚠️ Partial | Backend logic exists, scheduling needed |
| Dispute | ⚠️ Partial | Can initiate, resolution workflow incomplete |
| Refund | ✅ Complete | Full refund processing with gateway |
| Watermark Removal | ✅ Complete | Final milestone removes watermarks |
| Project Completion | ✅ Complete | Auto-complete on final milestone release |

**Milestone Status Flow:**

```
[pending] ──► [funded] ──► [in_progress] ──► [completed]
                              │
                              └──► [disputed]
```

### 4.6 Media & Sharing

| Feature | Status | Description |
|---------|--------|-------------|
| File Upload | ✅ Complete | Multipart upload, direct upload |
| Pagination | ✅ Complete | 50 items per page |
| Album Assignment | ✅ Complete | Many-to-many media-album |
| Watermark Flag | ✅ Complete | Track which media needs watermark |
| Shareable Links | ✅ Complete | Public gallery with unique token |
| Password Protection | ✅ Complete | Optional password for shares |
| Share Expiration | ✅ Complete | Optional expiry date |
| Album-Based Sharing | ✅ Complete | Select specific albums per share |
| Lightbox View | ✅ Complete | Full-screen media browsing |
| Download | ✅ Complete | Download from shared gallery |

**Note:** Actual watermark image rendering is not implemented—only the flag tracking exists.

### 4.7 Storage Management

| Feature | Status | Description |
|---------|--------|-------------|
| Free Quota | ✅ Complete | 100MB default free storage |
| Reserved Storage | ✅ Complete | 50GB or 100GB at $0.02/GB/month |
| Dynamic Storage | ✅ Complete | Pay-as-you-go at $0.10/GB/month |
| Usage Tracking | ✅ Complete | Real-time calculation of used bytes |
| Plan Switching | ✅ Complete | Switch between reserved/dynamic |
| Quota Adjustment | ✅ Complete | Increase/decrease reserved quota |
| Proration | ✅ Complete | Mid-cycle changes prorated |
| Usage History | ✅ Complete | Daily audit logs |

### 4.8 Billing & Invoicing

| Feature | Status | Description |
|---------|--------|-------------|
| Invoice Generation | ✅ Complete | Auto-generate from storage usage |
| Line Items | ✅ Complete | Subscription, overage, adjustments |
| Invoice Status | ✅ Complete | draft → finalized → paid/overdue |
| Payment Processing | ✅ Complete | Gateway integration (mock ready) |
| Payment History | ✅ Complete | Transaction log |
| Upcoming Charges | ✅ Complete | Forecast next billing |
| Tax Calculation | ❌ Placeholder | Model field exists, no logic |

### 4.9 Webhook System

| Feature | Status | Description |
|---------|--------|-------------|
| External Sources | ✅ Complete | Register webhook endpoints |
| Auth Methods | ✅ Complete | HMAC-SHA256, API key, Basic auth |
| Event Storage | ✅ Complete | Persist for replay |
| Retry Logic | ✅ Complete | Exponential backoff |
| Background Processing | ✅ Complete | Configurable interval |

---

## 5. Technical Architecture

### 5.1 Technology Stack

| Layer | Technology |
|-------|------------|
| **Backend** | Go 1.22, Chi router, GORM ORM |
| **Frontend** | React 19, TypeScript, Vite |
| **State Management** | Zustand (client), TanStack Query (server) |
| **Styling** | Tailwind CSS 4 |
| **Authentication** | Firebase Auth + JWT fallback |
| **Database** | SQLite (dev), PostgreSQL (prod) |
| **File Storage** | Local filesystem or S3-compatible |
| **Payment Gateway** | Mock (testing), Stripe/PayPal (interfaces ready) |

### 5.2 Backend Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        HTTP Layer                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    Chi Router                        │   │
│  │  /api/v1/auth, /projects, /proposals, /media, etc.  │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│                      Handlers (15 files)                    │
│  Request validation, response formatting, error handling    │
├─────────────────────────────────────────────────────────────┤
│                      Services (16 files)                    │
│  Business logic, state machines, transaction management     │
├─────────────────────────────────────────────────────────────┤
│                     Repository Layer                        │
│  GORM data access, query building                           │
├─────────────────────────────────────────────────────────────┤
│                     Infrastructure                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Database │  │ Storage  │  │ Gateway  │  │  Events  │   │
│  │ (GORM)   │  │ (S3/FS)  │  │ (Pay)    │  │  (Pub)   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 Frontend Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Pages Layer                          │
│  Dashboard, Projects, Proposals, Settings, etc.             │
├─────────────────────────────────────────────────────────────┤
│                     Components Layer                        │
│  UI primitives, feature components, layouts                 │
├─────────────────────────────────────────────────────────────┤
│                      Hooks Layer                            │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │  TanStack Query  │  │   Zustand Store  │                │
│  │  (Server State)  │  │  (Client State)  │                │
│  └──────────────────┘  └──────────────────┘                │
├─────────────────────────────────────────────────────────────┤
│                        API Layer                            │
│  Axios client with interceptors, typed API hooks            │
├─────────────────────────────────────────────────────────────┤
│                     External Services                       │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │   Firebase Auth  │  │   Backend API    │                │
│  └──────────────────┘  └──────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

### 5.4 API Structure

All endpoints under `/api/v1`:

| Resource | Endpoints |
|----------|-----------|
| Auth | `/auth/register`, `/auth/login`, `/auth/refresh`, `/auth/firebase/*` |
| Users | `/users/me`, `/creators`, `/creators/{id}` |
| Projects | `/projects`, `/projects/{id}`, `/projects/{id}/media`, `/projects/{id}/albums` |
| Media | `/media/{id}`, `/media/{id}/download`, `/media/{id}/albums` |
| Albums | `/albums/{id}`, `/albums/{id}/media` |
| Inquiries | `/inquiries`, `/inquiries/creator`, `/inquiries/client` |
| Proposals | `/proposals/{id}`, `/proposals/{id}/versions`, `/proposals/{id}/accept` |
| Milestones | `/milestones/{id}`, `/milestones/{id}/fund`, `/milestones/{id}/release` |
| Storage | `/storage/usage`, `/storage/subscribe`, `/storage/switch-mode` |
| Billing | `/billing/invoices`, `/billing/payments`, `/billing/upcoming` |
| Shares | `/shares/{id}`, `/public/shares/{token}` |

---

## 6. Domain Model & Data Flow

### 6.1 Entity Relationships

```
User (1) ─── (0..1) CreatorProfile
  │
  ├── (many) Project (as owner via ProjectCollaborator)
  │     │
  │     ├── (many) Album ─── (many) MediaObject (via MediaAlbum)
  │     │
  │     ├── (many) MediaObject
  │     │
  │     ├── (many) Share ─── (many) Album (via ShareAlbum)
  │     │
  │     └── (many) ProjectTransferRequest
  │
  ├── (many) Inquiry
  │     │
  │     └── (1) Proposal
  │           │
  │           └── (many) ProposalVersion
  │                 │
  │                 └── (many) Milestone (max 10)
  │                       │
  │                       └── (1) EscrowTransaction
  │                             │
  │                             └── (many) EscrowDispute
  │
  ├── (1) StorageSubscription
  │
  └── (many) Invoice
        │
        ├── (many) InvoiceLineItem
        │
        └── (many) PaymentTransaction
```

### 6.2 Primary User Flows

**Flow 1: Client Finds Creator and Starts Project**

```
1. Client browses /creators
2. Views creator profile and portfolio
3. Sends inquiry (with or without account)
4. System creates Inquiry + Proposal (status: inquiry)
5. Creator sees inquiry in dashboard
```

**Flow 2: Proposal Negotiation**

```
1. Creator creates first ProposalVersion (status → pending)
2. Creator sends proposal to client (status → negotiating)
3. Client reviews, can:
   a. Accept → status: finalized
   b. Counter-propose → new version, status: pending
   c. Cancel → status: canceled
4. Back-and-forth until finalized
```

**Flow 3: Project Execution with Escrow**

```
1. Creator or client creates Project from finalized proposal
2. Milestones copied from proposal
3. Client funds Milestone 1 (escrow holds funds)
4. Creator starts work
5. Creator completes work
6. Client releases funds (creator paid)
7. Repeat for remaining milestones
8. Final milestone release → watermarks removed, project completed
```

**Flow 4: Media Sharing**

```
1. Creator uploads media to project
2. Creates Share with selected albums
3. Optionally sets password and expiration
4. Shares unique URL with client
5. Client accesses gallery (verifies password if set)
6. Downloads media if allowed
```

---

## 7. Current Implementation Status

### 7.1 Backend Completeness

| Module | Lines of Code | Status |
|--------|---------------|--------|
| Models | 14 files | ✅ Complete |
| Handlers | 5,585 lines | ✅ Complete |
| Services | 5,366 lines | ✅ Complete |
| Middleware | ~200 lines | ✅ Complete |
| Storage Providers | 2 (local, S3) | ✅ Complete |
| Payment Gateways | 1 (mock) | ⚠️ Needs Stripe/PayPal |

### 7.2 Frontend Completeness

| Page/Feature | Status |
|--------------|--------|
| Authentication | ✅ Complete |
| Dashboard | ✅ Complete |
| Projects List | ✅ Complete |
| Project Detail | ✅ Complete |
| Media Management | ✅ Complete |
| Albums | ✅ Complete |
| Milestones | ✅ Complete |
| Shares | ✅ Complete |
| Public Gallery | ✅ Complete |
| Inquiries | ✅ Complete |
| Proposals | ✅ Complete |
| Creator Directory | ✅ Complete |
| Creator Profiles | ✅ Complete |
| Settings - Profile | ✅ Complete |
| Settings - Storage | ✅ Complete |
| Settings - Billing | ✅ Complete |

### 7.3 Integration Status

| Integration | Status | Notes |
|-------------|--------|-------|
| Firebase Auth | ✅ Working | Full OAuth flow |
| Local File Storage | ✅ Working | Development mode |
| S3 Storage | ⚠️ Untested | Code present, needs verification |
| Payment Gateway | ⚠️ Mock Only | Stripe/PayPal need implementation |
| Email Notifications | ❌ Not Implemented | No email service integrated |

---

## 8. Known Gaps & Technical Debt

### 8.1 Missing Features

| Feature | Priority | Effort | Notes |
|---------|----------|--------|-------|
| Real watermark rendering | High | Medium | Currently only tracks flag |
| Stripe integration | High | Medium | Interface ready |
| Email notifications | High | Medium | No notification system |
| Admin dashboard | Medium | High | Admin role exists but no UI |
| Dispute resolution workflow | Medium | Medium | Can initiate but no resolution |
| Collaborator UI integration | Medium | Low | Backend done, frontend needs hookup |
| Tax calculation | Low | Low | Placeholder in invoice model |
| Auto-invoice scheduling | Low | Medium | Manual trigger only |

### 8.2 Technical TODOs in Code

| Location | Issue |
|----------|-------|
| `backend/internal/handlers/escrow.go:492` | Missing project access verification |
| `backend/internal/handlers/proposal.go:537` | Milestone update not implemented |
| `backend/internal/handlers/share.go:744` | File serving not fully integrated |
| `backend/internal/handlers/share.go:516` | Share media access verification incomplete |
| `frontend/src/pages/ProjectDetail.tsx:1154` | Collaborators array hardcoded empty |

### 8.3 Security Considerations

| Item | Status | Action Needed |
|------|--------|---------------|
| JWT Secret | ⚠️ Default value | Change in production |
| CORS Origins | ⚠️ Localhost only | Add production domains |
| S3 Credentials | ⚠️ Env vars | Verify secure storage |
| Password Hashing | ✅ bcrypt | N/A |
| Firebase Tokens | ✅ Validated | N/A |

---

## 9. Recommended Next Steps

### 9.1 Immediate Priorities (Before Beta)

1. **Implement real watermarking**
   - Add image processing for watermark overlay
   - Integrate with media upload/download flow

2. **Integrate Stripe payments**
   - Implement PaymentGateway interface for Stripe
   - Add webhook handling for payment events
   - Test full escrow flow with real payments

3. **Add email notifications**
   - Choose email service (SendGrid, AWS SES, etc.)
   - Implement notification triggers for key events:
     - New inquiry received
     - Proposal status changes
     - Milestone funded/completed
     - Payment released

4. **Fix known TODOs**
   - Address access verification gaps
   - Complete share file serving

### 9.2 Short-Term Roadmap (1-2 months)

1. **Admin Dashboard**
   - Dispute resolution interface
   - User management
   - Platform analytics

2. **Enhanced UX**
   - Mobile-responsive improvements
   - Loading states and error handling polish
   - Onboarding flow for new users

3. **Production Hardening**
   - S3 storage validation
   - PostgreSQL testing
   - Rate limiting
   - Security audit

### 9.3 Medium-Term Roadmap (3-6 months)

1. **Advanced Features**
   - Real-time notifications (WebSocket)
   - In-app messaging
   - Calendar integration
   - Contract templates

2. **Analytics & Reporting**
   - Creator earnings dashboard
   - Client spending reports
   - Platform metrics

3. **Mobile App**
   - React Native or native development
   - Core workflows: gallery, messaging, payments

---

## 10. Risk Assessment

### 10.1 Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Payment integration issues | Medium | High | Test thoroughly in sandbox |
| S3 migration problems | Low | Medium | Test with production-like data |
| Database scaling | Low | High | PostgreSQL ready, monitor usage |
| Firebase outages | Low | Medium | JWT fallback exists |

### 10.2 Business Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Competitor features | Medium | Medium | Prioritize differentiating features |
| Low creator adoption | Medium | High | Focus on onboarding and support |
| Payment disputes | Medium | High | Clear terms, good dispute workflow |
| Storage cost overruns | Low | Low | Tiered pricing covers costs |

### 10.3 Operational Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Missing notifications | High | Medium | Prioritize email integration |
| Support load | Medium | Medium | Build help docs, FAQ |
| Watermark bypass | Low | High | Implement server-side rendering |

---

## 11. Glossary

| Term | Definition |
|------|------------|
| **Creator** | A photographer or videographer offering services on the platform |
| **Client** | A person or business hiring creators for projects |
| **Inquiry** | Initial contact from client to creator expressing interest |
| **Proposal** | Formal project terms including deliverables, timeline, and pricing |
| **ProposalVersion** | Immutable snapshot of proposal at a point in negotiation |
| **Milestone** | A phase of project work with associated payment amount |
| **Escrow** | Holding funds securely until work is completed and approved |
| **Watermark** | Protective overlay on media until final payment |
| **My Studio** | Auto-created project for creator's portfolio |
| **Reserved Storage** | Pre-paid storage allocation at lower per-GB rate |
| **Dynamic Storage** | Pay-as-you-go storage at higher per-GB rate |
| **Share** | Public link to view project media (optionally password-protected) |

---

## Appendix A: Key File Locations

### Backend

| Purpose | Path |
|---------|------|
| Entry point | `backend/cmd/server/main.go` |
| Configuration | `backend/internal/config/config.go` |
| All models | `backend/internal/models/` |
| All services | `backend/internal/services/` |
| All handlers | `backend/internal/handlers/` |
| Router setup | `backend/internal/handlers/router.go` |
| Auth middleware | `backend/internal/middleware/` |

### Frontend

| Purpose | Path |
|---------|------|
| Entry point | `frontend/src/main.tsx` |
| Routes | `frontend/src/App.tsx` |
| API hooks | `frontend/src/api/` |
| Pages | `frontend/src/pages/` |
| Components | `frontend/src/components/` |
| Auth store | `frontend/src/stores/authStore.ts` |
| Types | `frontend/src/types/index.ts` |

---

## Appendix B: Environment Configuration

### Backend Environment Variables

```bash
# Server
HOST=0.0.0.0
PORT=8080

# Database
DB_DRIVER=sqlite          # or postgres
DB_DSN=phto.db            # or postgres connection string

# Authentication
JWT_SECRET=change-me-in-production
JWT_EXPIRATION_HOURS=24
FIREBASE_PROJECT_ID=your-project-id
FIREBASE_CREDENTIALS_PATH=./credentials.json  # or
FIREBASE_CREDENTIALS_JSON='{"type":"service_account",...}'

# Storage
STORAGE_TYPE=local        # or s3
STORAGE_PATH=./uploads
S3_BUCKET=your-bucket
S3_REGION=us-east-1
S3_ENDPOINT=https://s3.amazonaws.com
S3_ACCESS_KEY_ID=your-key
S3_SECRET_ACCESS_KEY=your-secret
S3_URL_EXPIRY_MINUTES=15

# Pricing (cents)
STORAGE_RESERVED_PRICE_CENTS_PER_GB=2
STORAGE_DYNAMIC_PRICE_CENTS_PER_GB=10
STORAGE_DEFAULT_FREE_QUOTA_MB=100

# Events
EVENTS_PROCESSING_INTERVAL=30s
EVENTS_MAX_RETRIES=5
```

### Frontend Environment Variables

```bash
VITE_API_URL=http://localhost:8080/api/v1
```

---

*Document maintained by the Phto development team. For questions, contact the engineering lead.*
