# conpass — build status

_Updated: 2026-07-18_

Living status of the build. Plan & phase definitions: [BUILD-PLAN.md](BUILD-PLAN.md).
Fixed decisions: [DECISIONS.md](DECISIONS.md).

## Live environments

| Piece | Status | Where |
|---|---|---|
| Database | ✅ live | Supabase (us-east-1) — schema + RLS + Data-API lockdown applied |
| Backend API | ✅ live (prod) | `https://c8glyvxjh7.execute-api.us-east-1.amazonaws.com` — 10 Lambdas + 2 layers |
| Frontend PWA | ✅ live (test host) | `https://d2gwyvyec58l70.cloudfront.net` (S3+CloudFront, HTTPS) |
| Custom domains | ⏳ pending | `conpass.cards` (frontend) + `api.conpass.cards` — awaiting Hostinger CNAME |

AWS account 154320462594 · deploy: `backend/scripts/deploy.sh conpass prod`.

## Phases

| Phase | Status | Notes |
|---|---|---|
| **1 — Foundations** | ✅ Done | Contracts (OpenAPI 3.1), DB schema + RLS + Data-API grants, shared `conpass_common` layer, 10 SRP Lambdas, Serverless v3 infra, codegen, CI. |
| **2 — Auth + tenancy** | ✅ Done, deployed, verified live | Supabase Auth: owner login provisioned on onboarding (temp password, Función 03), operation-user creation (tier-limited), `/me` resolves roles/tenant from the JWT, asymmetric JWKS verification, RLS tenant isolation. Verified end-to-end through the deployed API with a real token. |
| **3 — Core domain** | ✅ Done, verified live | Programs CRUD (tier-limited), customer enrollment / card issuance (opaque QR token, dedupe, welcome bonus), in-store operations (accrue / redeem / validate) with idempotency + fraud window, card read. |
| **4 — Google Wallet** | ⏳ Pending | Needs the `conpass-key.json` service-account file. Blocks: enrollment/cards wallet links + `operations/resolve`. Provider abstraction is Apple-ready. |
| **5 — Unified PWA** | 🟡 In progress | Deployed. Wired to live API: onboarding → activation, login → role redirect, customer enrollment. Remaining screens (merchant panel, cashier scan, admin) render but not yet wired. |
| **6 — Admin + stubs** | ⏳ Pending | Payment + messaging providers stubbed (by design). Platform-admin, program metrics, birthday automation, notifications endpoints return `501`. |
| **7 — Efficiency review + hardening** | ⏳ Pending | Incl. re-slimming the deps layer, optional Lambda authorizer at the edge, SnapStart eval. |

## Endpoint status (deployed API)

**Live:** `GET /health`, `GET /me`, `POST /merchants`, `GET /merchants/{id}`,
`GET|POST /merchants/{id}/operation-users`, `GET|POST /programs`,
`GET|PATCH /programs/{id}`, `POST /programs/{id}/enroll`, `GET /cards/{id}`,
`POST /operations/{accrue,redeem,validate-access}`.

**Stubbed `501` (by phase):** `GET /programs/{id}/metrics` (P6),
`POST /operations/resolve` + `GET /programs/{id}/redemptions` (P3/UI hydrate),
`GET /cards/{id}/wallet-links` (P4), `PUT /programs/{id}/birthday-automation` +
`POST /birthday-cards` (P6), `GET /admin/*` + `PATCH /admin/*` (P6),
`POST /notifications/reminders` (P6).

## Journeys

| Journey | API (deployed) | Browser |
|---|---|---|
| Merchant onboarding → owner login | ✅ verified | ✅ working (activation screen shows temp password) |
| Customer enrollment (issue loyalty card) | ✅ verified | ✅ working |
| Merchant login → role-based routing | ✅ (`/me` roles) | ✅ working |
| Create program (authed) | ✅ verified w/ real token | 🟡 panel screen not wired |
| In-store accrue / redeem / validate | ✅ verified | 🟡 cashier screen not wired |
| Add to Google/Apple Wallet | ❌ Phase 4 | ❌ Phase 4 |

## Quality
17 tests pass (unit + live integration against real Supabase), ruff clean, CI green.

## Known follow-ups
- Wallet integration (P4) needs `conpass-key.json`.
- Wire remaining PWA screens (merchant panel, cashier, admin) — pairs with P6.
- Enrollment route param is named `:merchantSlug` but carries the `programId` (works; rename for clarity).
- Deps layer un-slimmed (`slim:false`) to keep `email-validator` metadata — re-slim in P7.
- Test data uses `@conpasstest.io` (the `.test` TLD is rejected by email validation).
