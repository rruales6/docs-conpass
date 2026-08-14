# docs-conpass — conpass platform docs

**conpass** is a SaaS loyalty & membership platform that delivers merchant programs
directly into **Google Wallet** and **Apple Wallet** — no app to install, no cardboard
cards. Ecuador-first, global-ready, bilingual ES/EN. This repo holds architecture,
decisions, AWS setup, and design reference. Code lives in two sibling repos.

## Repos

| Repo | What |
|---|---|
| **[bke-conpass](https://github.com/rruales6/bke-conpass)** | Backend — AWS Lambda (Python 3.12, ARM64), one SRP FastAPI Lambda per feature, Serverless Framework, Supabase Postgres. Contract-first OpenAPI 3.1. |
| **[fte-conpass](https://github.com/rruales6/fte-conpass)** | Frontend — one unified installable React + Vite **PWA** (role-based: public / merchant / staff / platform-admin), duotone design system, ES/EN. |
| **docs-conpass** (this) | Architecture, decisions, AWS runbooks, design reference. |

Both apps share **one Supabase database** (Postgres + Auth + RLS). Program image assets
(icon / background) and the payment QR live in a public-read **AWS S3** bucket; payment
receipts live in a separate **private** one.

## Contents

- **[STATUS.md](STATUS.md) — live build status & phase progress.**
- [BACKLOG.md](BACKLOG.md) — remaining work, MVP-prioritized (P0 pre-launch → deferred).
- [DECISIONS.md](DECISIONS.md) — fixed architecture & domain decisions.
- [BUILD-PLAN.md](BUILD-PLAN.md) — phases, multi-agent workflow, credential gates.
- [RUNBOOK-PAYMENTS.md](RUNBOOK-PAYMENTS.md) — the manual subscription-payment flow: publish
  the transfer details, verify a receipt, mark paid; plus troubleshooting and known limits.
- [aws/](aws) — least-privilege IAM policies + the [deploy runbook](aws/DEPLOY.md).
- [design-reference/](design-reference) — visual guidelines + UI prototype.

## Status

Phases 1–6 and 8–10 are done and deployed; Phase 7 (hardening) was partly done and the rest,
with all post-phase items, is the MVP-prioritized pool in [BACKLOG.md](BACKLOG.md). See
[STATUS.md](STATUS.md) for detail.

- **Database:** live on Supabase — schema + RLS + Data-API lockdown applied.
- **Backend:** deployed to AWS (prod, us-east-1) — `https://c8glyvxjh7.execute-api.us-east-1.amazonaws.com`. Live end-to-end: onboarding, auth/roles, programs, enrollment, in-store operations (offline + idempotent), Google Wallet (issue/update/revoke), platform-admin, program metrics/redemptions, and V1.1 operation-user management + card personalization (icon/background on S3 → wallet pass). Only birthday automation + notifications remain stubbed (`501`).
- **Frontend:** unified PWA fully wired (public / merchant / cashier / platform-admin journeys) and deployed to the CloudFront test host; cutover to `conpass.cards` (S3 + CloudFront) pending — see BACKLOG P0.
- **Subscriptions:** activation is **manual and receipt-gated** — a merchant transfers to the published account, uploads the receipt (the signup CTA stays disabled until they do, enforced server-side too), and platform-admin verifies it before marking paid. ⚠️ The transfer details start empty and **must be published in `/admin` → Datos de pago before anyone can subscribe** — see [RUNBOOK-PAYMENTS.md](RUNBOOK-PAYMENTS.md).

Core principles (non-negotiable): backend is the authority for balances; single-issuer
wallet model; offline-capable idempotent cashier ops; opaque per-member QR; data
minimization (LOPDP/GDPR). See [DECISIONS.md](DECISIONS.md).
