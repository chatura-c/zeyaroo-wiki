# Zeyaroo Wiki

> Creator-client media collaboration platform for photographers and videographers.

## Start here

New to the project? Read the **[phto-api Engineering Guide](phto-api/index.md)** — 10 sections covering architecture, data model, APIs, infra, gotchas, and local dev. It is the authoritative reference for the Go backend.

## Sections

| Section | Description |
|---------|-------------|
| **[phto-api Engineering Guide](phto-api/index.md)** | Authoritative backend engineering reference |
| [Architecture — Project Overview](architecture/project-overview.md) | High-level product architecture (kept from the older doc set) |
| [UI](ui/revamp/index.md) | UI design system, revamp plays, style and code guides |
| [Billing](billing/billing-design.md) | Subscription tiers, platform fees, earnings audit |
| [PayHere API](payhere/checkout-api.md) | Vendor API reference for PayHere integration |
| [Guides](guides/e2e-testing.md) | Operational setup: B2, email, E2E, frontend config |
| [Planning](planning/issues.md) | Active issues tracker |
| [Archive](archive/) | Historical planning docs, roadmaps, and early briefs |

## Running this wiki

```bash
cd wiki && python3 -m http.server 3000
```

Then open `http://localhost:3000`. The site is [docsify](https://docsify.js.org/) + Mermaid, no build step.
