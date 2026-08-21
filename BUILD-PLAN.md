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
8. **V1.1 design update — modals, operation-user management, card personalization** —
   added after a design revision (`design/Conpass_ Wallet Pass Platform DesignV1.1.zip`).
   Admin + merchant-panel inline panels/forms → **modals + kebab (⋮) action menus** (shared
   `Modal`/`Menu` primitives in `@conpass/ui`); **operation-user management** (edit / delete /
   reset-password endpoints on `/merchants/{id}/operation-users/{userId}`); **card
   personalization** — color picker + icon/background upload to a public-read S3 bucket via
   presigned PUT, wired onto the Google Wallet pass as `logo` / `heroImage`.
   *Needs AWS S3 (program-assets bucket, provisioned via `serverless.yml`).*
9. **Session controls (V1.1)** — "Cerrar sesión" on 04/05/06, "← Volver al demo" in the
   sandbox, "← Volver a escanear" in the cashier result view.
10. **Manual-payment activation (02 + 04)** — the subscription page can no longer be
    activated on trust: a manual-transfer **receipt is required before the "Activar cuenta"
    CTA unlocks**, and the page explains why. Función 02 gains the design's card/DEUNA
    method toggle (card disabled — no processor is integrated) and shows the **written
    transfer details** (bank, account, beneficiary, RUC) alongside the QR, so a payer never
    depends on the QR alone. Those details are **DB-backed** (`platform_payment_settings`,
    a singleton) and maintained from Función 04, which also gains a link to inspect each
    client's receipt before marking them paid. Receipts land in a **private** S3 bucket
    (presigned PUT to write, admin-only presigned GET to read).
    *Needs AWS S3 (payment-proofs bucket, provisioned via `serverless.yml`).*
11. **Card background: colour + optional image (and Google's image spec)** — creating a
    program painted an uploaded background image at 35 % opacity *over* the background
    colour, so the two visibly overlapped. The preview now composes them the way a pass
    does — the colour is the card, the image sits on it as a hero block — instead of
    blending them. The colour **always applies** and stays a merchant choice: Google
    documents a fallback (hero image → logo → a colour Google picks) but a real pass was
    observed keeping its original green, so the colour is set explicitly rather than left
    to a guess. Removing an uploaded image is possible for the first time (an empty
    storage key clears the field). Both uploads
    now follow **Google's Generic-pass image spec**: logo PNG 660×660 1:1 (was documented
    512×512, below Google's minimum) and hero image PNG 1032×812 ≈5:4 (was 1032×336 JPG,
    with the picker refusing PNG — the one format whose transparency lets the colour show
    through). Picked files are dimension-checked in the browser, with a non-blocking
    warning. Also fixed two things the pass itself got wrong: an installed pass never
    picked up a later edit (`update()` patched only the balance text, and nothing pushed
    on a program change), and the **tracked balance was buried** in the details section
    the holder has to expand. Now `update()` replaces the whole object — so colour, logo
    and hero image reach passes that are already in someone's wallet, and a removed image
    is actually cleared — a program edit pushes to its installed passes (best-effort,
    bounded), and the **stamps/points balance is the pass title**. *No new credentials.*

> Status is tracked in [STATUS.md](STATUS.md); remaining work is MVP-prioritized in
> [BACKLOG.md](BACKLOG.md). Phases 1–6 and 8–11 are done and deployed; Phase 7 (hardening)
> was partly done and the rest lives in the backlog. Phases 8–11 ran ahead of 7 because they
> came from design revisions and field reports rather than the original plan.

## Multi-agent workflow

Orchestrator (Opus, me) delegates to Sonnet implementers. Model generation tracks whatever
is current — Phases 1–9 ran on Opus 4.8 + Sonnet 4.5; Phase 10 onward runs on **Opus 5 +
Sonnet 5**. The role split below is what matters and does not change.

| Role | Model | Work | Why |
|---|---|---|---|
| Architect / contracts | Opus | OpenAPI, data model, provider interfaces, serverless skeleton | ripples everywhere |
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
- **Phase 8** needs an **AWS S3** program-assets bucket (public-read; created by `serverless.yml`,
  no new secret — uses the existing `aws.*` deploy credentials).
- **Phase 10** needs a second **AWS S3** bucket for payment proofs (**private** — all public
  access blocked; also created by `serverless.yml`, again no new secret).
- **Deploy** needs `aws.*` (region, profile/keys).
