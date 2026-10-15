# conpass — build status

_Updated: 2026-10-15_

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
| **7 — Efficiency review + hardening** | ✅ Done (scope trimmed) | Done: deps layer re-slimmed + 512 MB, RLS/Data-API audit passed, CORS fix for `console.conpass.cards`, global loading overlay. Evaluated and backlogged with rationale: SnapStart, edge Lambda authorizer, async wallet push, Secrets Manager, least-privilege IAM, CI + observability. See [BACKLOG.md](BACKLOG.md). |
| **9 — Session controls (V1.1)** | ✅ Done, deployed, verified live | Header **"Cerrar sesión"** on Admin (04), Panel (05) and cashier (06) → signs out and returns to login (07). In the `/demo` sandbox the same control becomes **"← Volver al demo"** (→ `/demo` hub) instead of a logout, so a demo visitor can leave a page without tearing down the shared demo session. Cashier result view (06·B) also gains a **"← Volver a escanear"** back link (the A→B flow was already sequential via the two cashier routes). Frontend-only; built (Node 20 + workbox) + deployed to CloudFront. |
| **10 — Manual-payment activation (02 + 04)** | ✅ **Closed** — done, deployed, verified live | Signup (Función 02) can no longer be activated on trust: the **"Activar cuenta" CTA stays disabled until a transfer receipt is uploaded**, and the page says so in plain language rather than hiding it in a tooltip. The gate is enforced **server-side too** — `POST /merchants` returns `422` when a `deuna`/`manual_transfer` payment arrives without a `proofStorageKey`. The payment card now matches the design's method toggle (**card tab visible but disabled — no processor is integrated**) and shows the **written transfer details** (bank, account type, account number with copy, beneficiary, RUC, contact email) *alongside* the QR, so a payer never depends on the QR alone. Those details are **DB-backed** — `platform_payment_settings`, a singleton table (migration `0008`) — and maintained from Función 04's new **"Datos de pago"** card + modal (all fields + QR upload), so changing the bank account or QR needs no redeploy. Función 04's "Gestionar cuenta" gains **"Ver comprobante"** (short-lived presigned link) right above "Marcar como pagado", plus the upload date; it states plainly when no receipt exists. Receipts go to a **private** bucket `conpass-payment-proofs-prod` (all four public-access blocks on) via a public presigned PUT; only platform-admin can read one back. Row→API mapping lives once in `conpass_common/payment_settings.py`, shared by both Lambdas. 70 backend tests + ruff clean. **Known limit (backlogged):** the proof is a single column written only at signup — no renewal-month upload path, and adding one against that column would overwrite the prior receipt. |

| **11 — Card background + Google image spec** | ✅ Done, deployed, verified live | Creating a program painted an uploaded background image at **35 % opacity over the background colour**, so the two visibly overlapped. The card preview now composes them the way a wallet pass does — the colour *is* the card, the image sits on it as a hero block — rather than blending them. The colour **always applies** and stays a merchant choice: Google documents a fallback (hero image → logo → a colour Google picks) but a real pass was seen keeping its original green, so we always send the colour explicitly rather than leave it to a guess, and a transparent PNG lets that colour show through. An **empty storage key clears a field**, which finally gives the panel a way to **remove** an uploaded image. Both uploads now follow **Google's Generic-pass image spec**: logo PNG 660×660 1:1 (was documented 512×512 — *below* Google's minimum) and hero image PNG 1032×812 ≈5:4 (was 1032×336 JPG, and the picker refused PNG outright — the one format whose transparency lets the colour through). Picked files are dimension-checked in the browser and a mismatch raises a **non-blocking amber warning**, never a block. **Wallet sync fixed** in the same pass: `GoogleWalletProvider.update()` patched only `textModulesData`+`state` and was called only after accrue/redeem, so an **installed pass never picked up a later appearance edit** — it now PUTs the whole object (a replace, so a removed image is genuinely cleared) and a program `PATCH` pushes to that program's installed passes, best-effort and bounded to `WALLET_PUSH_MAX_CARDS` per request. And the **tracked balance is now the pass title** (`header` = "3 / 8 sellos" / "120 / 200 puntos" / "Activa hasta …", programme name moved to `subheader`) — it used to sit in `textModulesData`, which Google only shows once the holder expands the pass. The **save link now carries the full `genericObject`** (per Google's web guide) instead of just an object id, so it can create the pass on its own when the best-effort REST pre-creation didn't happen — see **D13** and [RUNBOOK-WALLET.md](RUNBOOK-WALLET.md) for the 1800-character budget this has to fit in. 86 backend tests + ruff clean. |

| **12 — A pass that reads like a card** | ✅ Done, deployed, verified live | Until now Google's default Generic rendering decided everything, so the pass showed three things at a glance and the rest — holder, reward, validity — hid in the details section the holder has to expand. The class now carries a **`classTemplateInfo.cardTemplateOverride`**, which draws labelled rows on the **front** of the card, each cell bound to one of the object's `textModulesData` entries. So the kicker names the **kind of card** ("Tarjeta de lealtad" / "Pase de membresía"), the title is the **program name**, and the balance moves out of the title into a row of its own: `SELLOS ●●●○○○○○` · `PROGRESO 3 / 8` · `PREMIOS 1` — the strip for recognition and the figure for reading, since one does not replace the other. Below it the **reward** under its own label, and the **member's name and email** (D15 — the card id stays opaque; the token is the QR and is never printed). Past 12 stamps the dots stop being legible, so the row falls back to the count and the target. Row *shape* is a program decision and lives on the class, so a reward-less program simply has no reward row; per-card gaps cannot vary a class template, so those cells always exist and the object fills them with `—`. Three supporting changes made it safe: `_ensure_class` now **updates a class that already exists** (create-only would have stranded every program predating the template, rendering the old pass forever); the **save link references the created object by id** (~685 chars) instead of embedding it, because the richer pass measures ~2 035 against Google's 1 800-character URL limit — the link stopped carrying the object rather than the layout being cut to fit (**D13** revised); and the pass is no longer hardcoded Spanish — `cards.language` (migration `0009`) stores the language the customer had selected when they enrolled and every label follows it (**D16**). The merchant panel's card preview mirrors the new layout. 110 backend tests + ruff clean; the live integration test exercises the whole loop against the **real** Google Wallet API (class + template created, object with the new cells, accrue → PUT → revoke). **Revised after a real-phone test**: the stamp strip wrapped onto a second line because it shared a third-width cell, so it now gets a `oneItem` row of its own and the three short figures sit beneath it — `PROGRESO 3 / 8` · `POR CANJEAR` · `CANJEADOS`. `PROGRESO` carries the value/target pair for points too, which retired the separate `META` cell. `POR CANJEAR` and `CANJEADOS` are **different numbers**: the first is `rewards_available` (earned, still waiting, falls on redemption), the second the new `cards.rewards_redeemed` counter (migration `0010`, backfilled from `transactions`) that only climbs — put on the card row so no wallet path pays a query for it, including the 40-card program push. Whether the strip row exists is a *program* fact (type + target), never a per-holder one, since a class template cannot vary by card. See [RUNBOOK-WALLET.md](RUNBOOK-WALLET.md). |

| **13 — Pricing v2 (subscription + consumption)** | ✅ Done, deployed, verified live | Aligns the product with business model v2.0: Starter **$4.99** / Growth **$14.99** / Pro **$24.99**, with 200 / 1 000 / 5 000 active cards included and $0.044 / $0.034 / $0.024 per extra card, plus 1 000 / 10 000 notifications on Growth / Pro. One catalog in the shared layer (`conpass_common/plans.py`, **D17**) replaces four hardcoded copies of the old $19 / $49 / $149 / $499 list; public `GET /plans` serves it, and the signup page, admin console and merchant panel read it. New `GET /merchants/{id}/usage` gives used / included and the **estimated month** (the merchant panel's new *Su plan* card; admin shows the same per client and in MRR). Inclusions are not caps, so an extra card is billed and never refused. **No program limit on any plan**; operation users stay limited (2 / 5 / unlimited) and the 402 names the tier to upgrade to. Plan features are gated server-side (`402 feature_not_in_plan`) ahead of notifications and birthdays themselves. Pro is back on signup and Enterprise left the product. Migration `0011` drops the per-row price/limit copies and the trigger that never updated them on a plan change. 130 backend tests + ruff clean; frontend typecheck + build clean. Live-verified 2026-10-15: `GET /plans` serves the catalog, migration `0011` applied (the billable-card count now reads real active cards — it was stuck at 0 waiting on `wallet_installed`), `/demo/panel` shows *Su plan* (Growth · 4 / 1 000 cards · $14.99 estimated), signup lists all three plans with the amount due following the selection. The admin console was not browser-checked (needs a password login) — covered by unit tests. |

## Endpoint status (deployed API)

**Live:** `GET /health`, `GET /me`, `GET /demo`, `GET /plans` (public, Phase 13), `GET /merchants/{id}/usage` (Phase 13), `POST /merchants`, `GET /merchants/{id}`,
`GET|POST /merchants/{id}/operation-users`, `GET|POST /programs`,
`GET|PATCH /programs/{id}`, `GET /programs/{id}/metrics`, `GET /programs/{id}/redemptions`,
`POST /programs/{id}/enroll` (+ Google Wallet link),
`GET /cards/{id}`, `GET /cards/{id}/wallet-links`,
`POST /operations/{accrue,redeem,validate-access,resolve}`,
`GET /admin/{clients,stats}`, `PATCH /admin/clients/{id}/subscription`,
`GET /payment-settings` (public), `POST /payment-proofs/upload-url` (public),
`PATCH /admin/payment-settings`, `POST /admin/payment-settings/qr-upload-url`,
`GET /admin/clients/{id}/payment-proof`.

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
130 tests pass (unit + live integration against real Supabase **and** real Google Wallet),
ruff clean. Backend CI workflow disabled per request (`ci.yml.disabled`).

## Recently fixed
- **Program background colour and image overlapped** — the card preview blended an uploaded
  background over the colour at 35 % opacity, which read as an unintended overlap. The preview
  now draws the image *on* the coloured card instead of through it. The first fix attempt made
  the two mutually exclusive on the assumption that Google would derive a background colour from
  the hero image; a real pass proved it does not (it keeps its own default), so the colour went
  back to being a merchant choice that applies alongside the image. See **D12** in
  [DECISIONS.md](DECISIONS.md).

## Known follow-ups

Phase 10 is closed. **All remaining work lives in [BACKLOG.md](BACKLOG.md)**, MVP-prioritized
(P0 pre-launch · P1 soon-after · P2 polish/scale · deferred features).

⚠️ **Launch blocker carried out of Phase 10:** the transfer/DEUNA details ship EMPTY, and signup
deliberately refuses to take money until they are published — **no merchant can subscribe** until
platform-admin fills them in at `/admin` → *Datos de pago*. It is an operational step, not a code
task. See [RUNBOOK-PAYMENTS.md](RUNBOOK-PAYMENTS.md).

Operational notes kept here:
- Manual payment flow (publish details → verify receipt → mark paid), plus troubleshooting:
  [RUNBOOK-PAYMENTS.md](RUNBOOK-PAYMENTS.md).
- Google Wallet passes — object model, what the pass shows, the save-link size budget, how an
  installed pass stays in sync, troubleshooting: [RUNBOOK-WALLET.md](RUNBOOK-WALLET.md).
- Run `backend/scripts/reset_demo.py` to wipe demo-sandbox activity
  (creds `demo-owner@conpass.cards` / `conpass-demo-2026`).
- **Prod data was reset to a clean slate on 2026-08-21** (at the owner's request, to retest from
  scratch): the two non-demo merchants and all programs/cards/customers/transactions were
  deleted, auth users pruned to `admin@conpass.cards` + `demo-owner@conpass.cards`, orphaned
  program images removed from `conpass-program-assets-prod`, and the demo tenant re-seeded with
  one fresh program via `scripts/seed_demo.py`. Google Wallet objects were deliberately left
  alone, so passes saved before the reset now reference cards that no longer exist — they stay
  on the phone unchanged and their update pushes no-op.
