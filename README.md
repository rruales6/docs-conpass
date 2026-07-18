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

Both apps share **one Supabase database** (Postgres + Auth + Storage + RLS).

## Contents

- **[STATUS.md](STATUS.md) — live build status & phase progress.**
- [DECISIONS.md](DECISIONS.md) — fixed architecture & domain decisions.
- [BUILD-PLAN.md](BUILD-PLAN.md) — phases, multi-agent workflow, credential gates.
- [aws/](aws) — least-privilege IAM policies + the [deploy runbook](aws/DEPLOY.md).
- [design-reference/](design-reference) — visual guidelines + UI prototype.

## Status

- **Database:** live on Supabase — schema + RLS + Data-API lockdown applied.
- **Backend:** deployed to AWS (prod, us-east-1) — `https://c8glyvxjh7.execute-api.us-east-1.amazonaws.com`. Core domain (onboarding, programs, enrollment, in-store operations) working live; auth/wallet/admin phases in progress.
- **Frontend:** PWA scaffolded and building; hosting on `conpass.cards` (S3 + CloudFront) pending.

Core principles (non-negotiable): backend is the authority for balances; single-issuer
wallet model; offline-capable idempotent cashier ops; opaque per-member QR; data
minimization (LOPDP/GDPR). See [DECISIONS.md](DECISIONS.md).
