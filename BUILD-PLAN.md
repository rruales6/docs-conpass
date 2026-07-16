# Build plan & phases

## Phases

1. **Foundations (offline)** — schema+RLS, OpenAPI contracts, shared layer, provider
   interfaces, serverless config, codegen, CI, PWA shell. *No credentials needed.*
2. **Auth + tenancy** — Supabase Auth wiring, JWT authorizer, roles. *Needs Supabase secrets.*
3. **Core domain Lambdas** — programs, customers, card issuance, accrual, redemption,
   in-store operation (offline+idempotent).
4. **Google Wallet** — `GoogleWalletProvider`, issue/update/add-link. *Needs Google Wallet secrets.*
5. **Unified PWA** — all screens from the design system, ES/EN, QR scanner. *Needs Supabase.*
6. **Stubs + admin** — payment/messaging stubs, platform-admin, birthday automation.
7. **Opus efficiency review + hardening.**

## Multi-agent workflow

Orchestrator (Opus 4.8, me) delegates:

| Role | Model | Work | Why |
|---|---|---|---|
| Architect / contracts | Opus 4.8 | OpenAPI, data model, provider interfaces, serverless skeleton | ripples everywhere |
| DB / RLS | Sonnet → Opus review | migrations + RLS policies | mechanical; security-audited |
| Scaffolder | Sonnet | per-feature Lambda scaffolding, codegen, CI | pure boilerplate |
| Feature-Lambda implementers (parallel) | Sonnet | business logic per Lambda | isolated SRP units |
| Wallet specialist | Opus 4.8 | Google Wallet + abstraction (Apple-ready) | highest-risk |
| Auth/security | Opus design → Sonnet wiring | Supabase Auth, JWT authorizer, RLS | subtle design |
| PWA implementers (parallel) | Sonnet | screens from design system + typed client | grunt UI |
| Efficiency reviewer | Opus 4.8 | whole-repo: cold-start, DRY, SRP, cost, RLS | user's explicit ask |
| Test / CI | Sonnet | unit + Schemathesis, GitHub Actions | routine |

## Credential gates

Work proceeds offline until it hits one of these; then it pauses and asks:

- **Phase 2/5** need `supabase.*` (url, anon_key, service_role_key, jwt_secret, db_url).
- **Phase 4** needs `google_wallet.*` (issuer_id, service_account_json).
- **Deploy** needs `aws.*` (region, profile/keys).
