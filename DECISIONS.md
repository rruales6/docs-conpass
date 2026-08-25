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
| D12 | A program's card background is a **colour plus an optional image**, and the colour always applies. Google documents a fallback chain for an unset `hexBackgroundColor` (hero image → logo → a colour Google picks), but a real pass was seen keeping its original green rather than following the new image, so the colour is always sent explicitly instead of left to a guess — and a transparent PNG is what lets that colour show through. The reported "overlap" was a **rendering bug in our panel** (the preview painted the image at 35 % opacity over the colour), not a data problem: the preview now composes them the way a pass does, image on the coloured card. An empty string clears `color` / `iconStorageKey` / `backgroundStorageKey`; an omitted field is left unchanged | user (Phase 11) |
| D13 | **The "Add to Google Wallet" link references the object by id whenever the object is known to exist** (~700 chars), and only carries the full `genericObject` when it does not. Google caps the encoded JWT at **1800 characters** ("the save may not work due to truncation by web browsers"), and a pass with a logo, a hero image and the card-front rows measures ~2 035 — so referencing, which was Phase 11's rescue for oversized payloads, is now the normal path. That is what lets the layout grow without the link silently degrading. The self-contained form still exists for the one case that needs it: `issue()`'s REST pre-creation is best-effort and must never block enrollment, so a link minted without that assurance has to be able to create the pass by itself. The payload still omits `origins` (JS-button-only) and image `contentDescription`. See [RUNBOOK-WALLET.md](RUNBOOK-WALLET.md) | user (Phase 11, revised Phase 12) |
| D14 | **The card front is a template, not just a title.** Phase 11 put the balance in `header` because untemplated `textModulesData` only renders once the holder expands the pass. Phase 12 removes that constraint instead of working around it: the class carries a `classTemplateInfo.cardTemplateOverride` whose rows are drawn on the FRONT, each cell bound to `object.textModulesData['…']` — so `subheader` names the kind of card, `header` is the programme, and the balance gets a labelled row of its own (`SELLOS ●●●○○○○○` · `PROGRESO 3 / 8` · `PREMIOS 1`) alongside the reward and the member. Row shape is a PROGRAM decision and lives on the class; per-card gaps cannot vary it, so those cells always exist and the object fills them with `—`. `update()` remains a **full PUT** — appearance and label edits must reach passes already installed, and only a replace can clear a removed image — and `_ensure_class` **updates a class that already exists** rather than only creating on 404, or every program predating this layout would render the old pass forever | user (Phase 11, revised Phase 12) |
| D15 | **The pass shows the member's name and email**, which narrows B6 — a holder's address now sits in Google's copy of the object and is readable by anyone holding the phone. Taken deliberately: a card nobody can identify is worth less to the merchant at the counter. The **card id stays opaque** — the token is the QR and is never printed as text, so the readable fields identify a person, never authorise anything. Email is READ from `customers` on each wallet path and is never copied onto `cards`: PII keeps one home, so deleting a customer still deletes it | user (Phase 12) |
| D16 | **A pass renders one language, captured at enrollment.** The card stores the UI language the customer had selected when they submitted the enroll form (`cards.language`, default `es`); every label on the pass comes from it. Not Google's `translatedValues`, which follow the holder's device locale — that is not what a merchant in Ecuador means by "this card is in Spanish" | user (Phase 12) |
| D10 | Messaging (WhatsApp/email) **stubbed** behind `NotificationProvider` | user |

## Domain rules (from business model §10)

| # | Rule |
|---|---|
| B1 | **Backend is the authority for balance**; the wallet pass is visualization only |
| B2 | Emission: single-issuer shared cert for Starter/Growth/Pro; Pro gets a dedicated Pass Type ID; Enterprise = BYO-cert |
| B3 | Billing metric = **active installed passes** ("installed and not deleted from wallet") |
| B4 | QR carries an **opaque per-member identifier**; NFC deferred |
| B5 | Cashier operation must work **offline** and be **idempotent**; < 3s per op; per-window fraud limits |
| B6 | Data minimization: a pass needs only an opaque id; PII is optional and post-install — **narrowed by D15**, which puts the holder's name and email on the face of the pass while keeping the id opaque |
| B7 | Tiers: Starter / Growth / Pro / Enterprise — access-account limits & feature gates per tier |
| B8 | Multi-tenant: each merchant (`comercio`) is a tenant; RLS isolates data; platform-admin is cross-tenant |

## Open (deferred, needs credentials or a later decision)

- Payment provider (DEUNA / Stripe / Kushki) — chosen later; interface ready.
- WhatsApp + email provider — chosen later; interface ready.
- Apple Wallet certificates (Pass Type ID, WWDR, `.p12`) — Phase for Apple.
- `novamira` is the user's unrelated marketing site — NOT an integration target (confirmed).
