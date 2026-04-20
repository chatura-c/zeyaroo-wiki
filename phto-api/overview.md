# 1. Overview

## What is phto-api?

phto-api is the Go backend for **Zeyaroo**, a SaaS platform that lets photographers and videographers collaborate with clients on paid media projects. It replaces the ad-hoc email/WhatsApp + bank-transfer + Google Drive workflow that most independent creators use today with a single system that handles discovery, quoting, escrow payments, deliveries, and permanent client galleries.

## The problem it solves

Freelance creators in markets like Sri Lanka face three recurring problems when working with clients:

1. **Getting paid is awkward.** Clients expect a tangible deliverable before paying; creators need funds held before starting work. Cash and bank-transfer flows offer no protection to either side.
2. **Quotes and revisions drift.** Proposals live in emails and WhatsApp messages. There is no authoritative version. Amendments to scope mid-project lead to disputes.
3. **Delivery is ephemeral.** Google Drive links expire, WeTransfer bundles vanish, and clients lose originals. There is no permanent, branded place for the work to live.

phto-api addresses each: **versioned proposals** with a state machine that enforces who can submit in which state; **milestone-based escrow** that holds funds until the client releases them or an auto-release timer fires; and **delivery projects** that transfer media ownership into the client's own account, so the work persists independently of the creator.

## High-level summary

A Go 1.24 HTTP service written on `chi`, with GORM over either SQLite (dev) or Postgres (prod). Business domain spans ~50 tables across five logically separate databases (main, events, notifications, webhooks, storage). It runs behind Caddy on an Oracle Cloud ARM VM, deploys via Woodpecker → OCIR → Komodo, and talks to Backblaze B2 for media, Resend for email, Firebase for Google sign-in, and DodoPayments / PayHere for card and local payments. The codebase is about 35,000 lines of Go, with the thickest logic in `internal/services/` (escrow, proposal negotiation, notifications) and `internal/handlers/` (chi handlers for ~120 routes).

## Who uses it

- **Creators** — photographers and videographers running their own business. They are the paying SaaS customers (Starter / Pro / Studio tiers).
- **Clients** — usually guests. They sign in only if they want an account; most interact via email-linked public tokens (`/public/proposals/{token}`, `/public/pay/{token}`, `/public/shares/{token}`).
- **Admins** — internal Zeyaroo staff, via the separate `zeyaroo-admin` dashboard (Vite + React), which talks to admin-only routes on the same backend.
