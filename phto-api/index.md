# phto-api — Engineering Guide

> Zeyaroo's Go backend. Creator-client media collaboration: inquiries → proposals → milestones → escrow-backed payments → media delivery.

This guide is aimed at an engineer joining the project. It covers **what** the system does, **how** it does it, and **why** it was built this way. Each page is a stand-alone section; read top-to-bottom on first pass.

| Section | Description |
|---------|-------------|
| [1. Overview](phto-api/overview.md) | What this service is and what it solves |
| [2. Architecture](phto-api/architecture.md) | Components, data flow, external deps |
| [3. Key Decisions](phto-api/decisions.md) | Technology and design tradeoffs |
| [4. Data Model](phto-api/data-model.md) | Entities, relationships, Mermaid ER diagram |
| [5. API Surface](phto-api/api-surface.md) | Routes, auth, rate limits |
| [6. Infrastructure & Deployment](phto-api/infrastructure.md) | Docker, Woodpecker, Komodo, OCIR |
| [7. Gotchas & Landmines](phto-api/gotchas.md) | Non-obvious behaviors, footguns |
| [8. Technical Debt & Future](phto-api/tech-debt.md) | What's deferred and why |
| [9. What to Expect](phto-api/operations.md) | Failure modes, scaling, on-call |
| [10. Local Development](phto-api/local-dev.md) | Running and debugging locally |

## Repo

The service lives at `phto-api/` in the zeyaroo monorepo. Module path: `github.com/bytkloud/phto`. The legacy module name is preserved — "phto" was the original codename; "zeyaroo" is the user-facing brand.
