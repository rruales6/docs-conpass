# conpass — build status

_Updated: 2026-07-22_

Living status of the build. Plan & phase definitions: [BUILD-PLAN.md](BUILD-PLAN.md).
Fixed decisions: [DECISIONS.md](DECISIONS.md).

## Live environments

| Piece | Status | Where |
|---|---|---|
| Database | ✅ live | Supabase (us-east-1) — schema + RLS + Data-API lockdown applied |
| Backend API | ✅ live (prod) | `https://c8glyvxjh7.execute-api.us-east-1.amazonaws.com` — 10 Lambdas + 2 layers |
| Frontend PWA | ✅ live | `https://console.conpass.cards` (operator console; CNAME/alias → `d2gwyvyec58l70.cloudfront.net`). `/demo` browser-verified. |
| Custom domains | ✅ connected (Phase 8) | Frontend live at `console.conpass.cards`. **Note:** origin must be in the backend CORS allowlist (fixed in Phase 7 — was causing a CORS error on API calls, most visibly `/demo`). |

AWS account 154320462594 · deploy: `backend/scripts/deploy.sh conpass prod`.

## Phases

| Phase | Status | Notes |
|---|---|---|
| **1 — Foundations** | ✅ Done | Contracts (OpenAPI 3.1), DB schema + RLS + Data-API grants, shared `conpass_common` layer, 10 SRP Lambdas, Serverless v3 infra, codegen, CI. |
| **2 — Auth + tenancy** | ✅ Done, deployed, verified live | Supabase Auth: owner login provisioned on onboarding (temp password, Función 03), operation-user creation (tier-limited), `/me` resolves roles/tenant from the JWT, asymmetric JWKS verification, RLS tenant isolation. Verified end-to-end through the deployed API with a real token. |
| **3 — Core domain** | ✅ Done, verified live | Programs CRUD (tier-limited), customer enrollment / card issuance (opaque QR token, dedupe, welcome bonus), in-store operations (accrue / redeem / validate) with idempotency + fraud window, card read. |
| **4 — Google Wallet** | ✅ Done, deployed, verified live | `GoogleWalletProvider` (Generic passes, REST + signed save-link) — zero new deps (PyJWT + httpx). Wired: enrollment issues a pass + returns the "Add to Google Wallet" link (best-effort — never fails enrollment), `GET /cards/{id}/wallet-links`, `POST /operations/resolve` (cashier hydrate), and accrue/redeem reflect balances into the pass best-effort. Proven end-to-end against the real Google Wallet API + live Supabase (issue → update → revoke, self-cleaning). **Deployed to prod (`--force`, 2026-07-18)** — Lambdas carry the SA-JSON env; `/operations/resolve` + `/cards/{id}/wallet-links` live. Provider abstraction stays Apple-ready. |
| **5 — Unified PWA** | 🟡 Mostly done, deployed | Wired to live API: onboarding → activation, login → role redirect, customer enrollment (+ working "Add to Google Wallet" button), **cashier scan → resolve → accrue/redeem** (html5-qrcode camera + manual entry, offline-queued & idempotent, stage-then-confirm stamp batching), and **merchant panel** program create/list + per-program enroll link (copy). Admin dashboard + panel **metrics** stay placeholders until their backends ship (Phase 6, `501`). Plus a **public self-serve `/demo` sandbox**: `GET /demo` returns the most-recent `is_demo` tenant + shared test creds, the PWA silently signs in and reuses the real enroll/panel/cashier journeys under `/demo/*` (verified live). Rebuilt (Node 20) + deployed to the CloudFront test host. |
| **6 — Admin + stubs** | 🟡 Core done, deployed, verified live | **Program metrics** (`GET /programs/{id}/metrics` via `program_metrics_view` — active passes ≈ active cards, wallet-saved callback deferred), **redemptions report** (`GET /programs/{id}/redemptions`), and **platform-admin** (`GET /admin/{clients,stats}` + `PATCH /admin/clients/{id}/subscription` = manual payment activation). Frontend: admin dashboard (stats + client list + confirm-payment) and panel metrics + redemptions wired. Verified end-to-end live (admin token + browser). Payment/messaging **providers stay stubbed** (by design). **Deferred**: birthday automation + `POST /notifications/reminders` (still `501`). Test account: `platform_admin` seeded (`scripts/seed_admin.py`). |
| **8 — V1.1 modals + op-user mgmt + card personalization** | ✅ Done, deployed, verified live | Design V1.1 UI: inline panels → **modals + kebab (⋮) menus**. Admin: per-row ⋮ → "Gestionar cuenta" modal (change plan · mark paid · suspend/reactivate, via `PATCH /admin/clients/{id}/subscription`). Merchant panel: `+`-triggered **create/edit program modal** (+ per-program ⋮ edit/enable-disable via `PATCH /programs/{id}`) and a new **Usuarios de operación** section — add/edit/remove/reset-password (new endpoints `PATCH`/`DELETE`/`POST …/reset-password` on `/merchants/{id}/operation-users/{userId}`; reset returns a fresh temp password). **Card personalization**: color picker + icon/background upload to a **public-read S3 bucket** (`conpass-program-assets-prod`) via presigned PUT (`POST /programs/{id}/appearance-upload-url`); images wired onto the Google Wallet pass as `logo` + `heroImage`. Shared `Modal`/`Menu` primitives added to `@conpass/ui`. 53 backend tests + ruff clean; presign→S3→public-GET verified end-to-end in prod; deployed backend (`--force`) + frontend (CloudFront). Suspension is a status flag (no login-block enforcement this phase). |
| **7 — Efficiency review + hardening** | ⏳ Pending | Incl. re-slimming the deps layer, optional Lambda authorizer at the edge, SnapStart eval. |

## Endpoint status (deployed API)

**Live:** `GET /health`, `GET /me`, `GET /demo`, `POST /merchants`, `GET /merchants/{id}`,
`GET|POST /merchants/{id}/operation-users`, `GET|POST /programs`,
`GET|PATCH /programs/{id}`, `GET /programs/{id}/metrics`, `GET /programs/{id}/redemptions`,
`POST /programs/{id}/enroll` (+ Google Wallet link),
`GET /cards/{id}`, `GET /cards/{id}/wallet-links`,
`POST /operations/{accrue,redeem,validate-access,resolve}`,
`GET /admin/{clients,stats}`, `PATCH /admin/clients/{id}/subscription`.

**Stubbed `501` (deferred):** `PUT /programs/{id}/birthday-automation` +
`POST /birthday-cards` (birthday automation), `POST /notifications/reminders` (messaging).

## Journeys

| Journey | API (deployed) | Browser |
|---|---|---|
| Merchant onboarding → owner login | ✅ verified | ✅ working (activation screen shows temp password) |
| Customer enrollment (issue loyalty card) | ✅ verified | ✅ working |
| Merchant login → role-based routing | ✅ (`/me` roles) | ✅ working |
| Create program (authed) | ✅ verified w/ real token | ✅ wired (panel create + list + enroll link) |
| In-store accrue / redeem / validate | ✅ verified | ✅ wired (scan/manual → resolve → queue) |
| Add to Google Wallet | ✅ live on prod API (issue/update/revoke) | ✅ button wired in enroll flow |
| Add to Apple Wallet | ❌ future (abstraction ready) | ❌ future |

## Quality
54 tests pass (unit + live integration against real Supabase **and** real Google Wallet),
ruff clean. Backend CI workflow disabled per request (`ci.yml.disabled`).

## Recently fixed
- **Cashier accrue double-count (`0→1→3→6`)** — the idempotency store's `put` inserted response
  bodies still holding `UUID`/`datetime` objects, so accrue/redeem 500'd *after* committing, the
  key was never recorded, and the offline queue's retries re-applied the op. Fixed by normalizing
  the stored body to JSON-safe primitives (`idempotency.py`); regression test added. Complementary
  frontend fix: the cashier now awaits the backend and renders the **authoritative** balance.
- **Slow merchant-panel load** — every request called `supabase.auth.getSession()`, serializing
  concurrent fetches on supabase-js's auth lock. The API client now caches the token
  (`onAuthStateChange`) so the panel's loads run in parallel. Also deduped the double `GET /me`
  on login via `AuthContext.refreshIdentity()`.
- **PWA stale-bundle after deploy** — SW now serves document navigations **network-first**
  (`vite.config.ts`: `navigateFallback: null` + a `conpass-app-shell` NetworkFirst rule),
  so a hard load / deep link always fetches the current bundle. Verified: `/demo` loads
  fresh on first load. (Applies going forward once each client picks up this SW.)
- **`apply_migrations.py` silently rolled back every migration** — root cause: a trailing
  `SELECT` on a non-autocommit connection left a txn open, so each `with conn.transaction()`
  became a savepoint that never top-level committed and `close()` rolled it all back (while
  printing "applied"). Fixed: `autocommit=True`, force session port 5432 (not the 6543 txn
  pooler), correct pooler user/ref. Verified persistence with a throwaway migration; all 6
  migrations now tracked.

## Known follow-ups

Phase 8 is complete. **All remaining work now lives in [BACKLOG.md](BACKLOG.md)**, MVP-prioritized
(P0 pre-launch · P1 soon-after · P2 polish/scale · deferred features). Operational note kept
here: run `backend/scripts/reset_demo.py` to wipe demo-sandbox activity
(creds `demo-owner@conpass.cards` / `conpass-demo-2026`).
