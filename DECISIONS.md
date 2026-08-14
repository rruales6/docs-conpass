# Fixed decisions

Decisions locked for the build. Sources: user directives + `loyalty-programs`
business model (`docs/modelo-negocio.md` §10) + design (`docs/design-reference`).

## Product / architecture

| # | Decision | Source |
|---|---|---|
| D1 | Single Supabase project (Postgres + Auth + Storage + RLS) shared by all surfaces | user |
| D2 | Backend = AWS Lambda, Python 3.12, ARM64, one **single-responsibility Lambda per feature** | user |
| D3 | FastAPI + Mangum per Lambda; shared code in a Lambda **layer** | user |
| D4 | **Contract-first**: OpenAPI 3.1 in `backend/contracts` is the single source of truth | user |
| D5 | IaC = **Serverless Framework** (`serverless.yml`, per-service functions, python-requirements) | user |
| D6 | Frontend = **one unified PWA** (React + Vite), role-based routing (public / merchant / staff / platform-admin) | user |
| D7 | Auth = **Supabase Auth (GoTrue)**; asymmetric JWT verified **in-Lambda via JWKS URL** (ES256/RS256); RLS for tenancy | derived |
| D7a | Supabase **current key model**: publishable key (browser, anon/authenticated) + secret key (backend, service_role, bypasses RLS). Legacy anon/service_role/HS256-secret NOT used | user |
| D7b | **Data API safety strategy**: frontend uses Supabase only for Auth; all business data via backend Lambdas (secret key). Client roles (anon/authenticated) are REVOKED on all public tables + RLS forced (0004_data_api_grants.sql). Backend DML goes through the Data API (PostgREST) with the secret key | user |
| D7c | Frontend served from **https://conpass.cards**; API on **https://api.conpass.cards** (CORS locked to the frontend origin) | user |
| D8 | Wallet vendors behind `WalletProvider` interface — **Google now, Apple later** with zero feature changes | user |
| D9 | Payments **stubbed** behind `PaymentProvider` (manual proof only for now) | user |
| D9a | A manual-transfer/DEUNA signup **requires a receipt**. Enforced server-side: `POST /merchants` returns `422` without `payment.proofStorageKey`. The disabled CTA in the PWA is UX; the API check is the gate, and both stay | user (Phase 10) |
| D9b | The transfer/DEUNA account details are **DB-backed and admin-editable** (`platform_payment_settings`, a singleton), not config — the bank account or QR can change with no redeploy. One shared row→API mapper (`conpass_common/payment_settings.py`) serves both the public read and the admin write so they cannot drift | user (Phase 10) |
| D9c | Card payments are **not integrated**. The card tab renders (it is in the design) but is `disabled` + *Próximamente*; it must never be able to activate an account until a real processor exists | derived (Phase 10) |
| D11 | **Two S3 buckets, split by audience.** Program images + the payment QR are public-read (they are shown to anonymous visitors). Payment receipts go in a separate **private** bucket — they carry names, account numbers and amounts — written via a public presigned PUT and read only via an admin-signed GET. Receipts must never be moved into the public bucket | user (Phase 10) |
| D10 | Messaging (WhatsApp/email) **stubbed** behind `NotificationProvider` | user |

## Domain rules (from business model §10)

| # | Rule |
|---|---|
| B1 | **Backend is the authority for balance**; the wallet pass is visualization only |
| B2 | Emission: single-issuer shared cert for Starter/Growth/Pro; Pro gets a dedicated Pass Type ID; Enterprise = BYO-cert |
| B3 | Billing metric = **active installed passes** ("installed and not deleted from wallet") |
| B4 | QR carries an **opaque per-member identifier**; NFC deferred |
| B5 | Cashier operation must work **offline** and be **idempotent**; < 3s per op; per-window fraud limits |
| B6 | Data minimization: a pass needs only an opaque id; PII is optional and post-install |
| B7 | Tiers: Starter / Growth / Pro / Enterprise — access-account limits & feature gates per tier |
| B8 | Multi-tenant: each merchant (`comercio`) is a tenant; RLS isolates data; platform-admin is cross-tenant |

## Open (deferred, needs credentials or a later decision)

- Payment provider (DEUNA / Stripe / Kushki) — chosen later; interface ready.
- WhatsApp + email provider — chosen later; interface ready.
- Apple Wallet certificates (Pass Type ID, WWDR, `.p12`) — Phase for Apple.
- `novamira` is the user's unrelated marketing site — NOT an integration target (confirmed).
