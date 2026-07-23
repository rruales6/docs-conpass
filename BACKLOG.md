# Backlog

Consolidated list of everything not yet built, **prioritized for MVP**. Phase 8 (V1.1
modals, operation-user management, card personalization) is complete and deployed; the
items below are what's left. Nothing here is scheduled — this is the ordered pool to pull
from next.

Priority key:
- **P0 — pre-launch:** must be done before real (paying) merchants use the product.
- **P1 — soon after launch:** correctness/observability/billing-accuracy; not launch-blocking.
- **P2 — polish / scale:** perf, cleanups, scale-only optimizations.
- **Deferred features / Dev notes:** explicitly out of MVP by product decision, or local-only.

---

## P0 — Pre-launch (MVP blockers)

- **Custom domains.** Cut over from the CloudFront test host (`d2gwyvyec58l70.cloudfront.net`)
  to `conpass.cards` (frontend) + `api.conpass.cards` (API): Hostinger CNAMEs, ACM cert
  (us-east-1), CloudFront alias + API Gateway custom domain. Then remove the `cloudfront.net`
  allowances from the API CORS list (`serverless.yml`) and the S3 asset bucket CORS.
- **Least-privilege IAM + rotate secrets.** Replace the temporary AWS admin (`conpass-admin`)
  with the scoped policies already in `docs/aws/`; rotate the seeded demo/platform-admin
  passwords (currently shared test creds) before exposing prod. _(Phase 7 #10.)_

## P1 — Soon after launch

- **Idempotency write atomicity.** A mutation `compute()` commits the balance change BEFORE
  the idempotency record is written (`run_idempotent` in `conpass_common/idempotency.py`), so
  any failure of `store.put` still leaves the op applied but the key unrecorded → a replay
  re-applies it. The acute case (UUID/datetime JSON serialization) is fixed, but the ordering
  is still fragile. Robust fix: write the idempotency record in the SAME DB transaction as the
  commit, or store-first. _(Found during the Phase 8 cashier-accrue debug.)_
- **Google Wallet "pass saved" callback → `cards.wallet_installed`.** Nothing flips
  `wallet_installed` today, so *active installed passes / installs-this-week* (and the billing
  metric) are approximated as active cards. Wire the Wallet save callback to set the flag for
  real. _(Phase 7 #6.)_
- **Suspend enforcement.** `PATCH /admin/.../subscription` can set `paymentStatus=suspended`,
  but it's only a status flag today — a suspended merchant's owner + operation users can still
  log in and operate. Enforce it (block auth / gate the JWT authorizer) once real billing is
  in place. _(Product decision in Phase 8 was flag-only.)_
- **Async wallet push off the cashier path.** The post-commit `provider.update()` on
  accrue/redeem runs synchronously (best-effort). Move it to an async worker (SQS/EventBridge)
  so a slow Google Wallet call can never brush the in-store `<3s` operation budget. _(Phase 7 #5.)_
- **Re-enable backend CI + observability.** CI is disabled (`ci.yml.disabled`). Re-enable, add
  Schemathesis contract fuzzing, CloudWatch alarms, and structured error tracking. _(Phase 7 #11.)_
- **Secrets → AWS Secrets Manager / SSM.** The SA JSON + Supabase secret key are injected as
  Lambda env vars (under the 4 KB limit). Move to a managed, rotatable store. _(Phase 7 #9.)_

## P2 — Polish / scale

- **Parallelize program-create image uploads.** `MerchantPanelPage` uploads a staged icon then
  background sequentially (`~line 411`); both only need the new program id — wrap in
  `Promise.all`. Safe win, only fires when creating a program with both images. _(Phase 8 audit.)_
- **Parallelize the offline-queue flush** (`features/offline-queue/queue.ts`, the `for … await`
  loop). Care needed: multiple queued ops can target the same card (order matters) and could
  trip the per-card rate limiter — needs same-`cardId` ordering + bounded concurrency, not a
  drop-in `Promise.all`. _(Phase 8 audit.)_
- **Fraud-window tuning.** Accrual limit is `MAX_ACCRUALS_PER_WINDOW=5 / 60s` per card
  (`operations/handler.py`). Validate against real cashier throughput; make it tier/config-driven
  if too tight or too loose.
- **Re-expose operation-user station** (caja 1/2 / door-access). Removed from the panel UI per
  request ("don't need right now"); new users default to `caja_1`. Bring the picker back when
  station-based routing is needed.
- **Edge REQUEST authorizer.** Verify the Supabase JWT once at API Gateway instead of in every
  feature Lambda. Deferred: high blast radius, payoff mostly at scale (feature Lambda already
  does a cheap `@lru_cache`'d JWKS verify; a cache miss *adds* a hop). Needs an authorizer
  Lambda + per-route wiring + dual-path `require_identity` + string-serialized roles. _(Phase 7 #2.)_
- **Re-slim the dependency layer.** `pythonRequirements.slim` is `false` (needed so
  `email-validator` keeps its `dist-info` metadata). Re-enable slimming with
  `slimPatternsAppendDefaults: false` + patterns limited to `*.pyc`/`__pycache__` to cut layer
  size / cold-start. _(Phase 7 note.)_
- **Lambda SnapStart.** Poor ROI now: Python SnapStart is x86-only (our Lambdas are arm64), no
  longer free for Python, and needs published versions/aliases (httpApi rewiring). Revisit if
  cold starts become the dominant cost. _(Phase 7 #3.)_
- **PWA "new version available" toast.** Surface a "refresh for the latest" prompt when the
  service worker detects a new build. _(Phase 7 #13.)_
- **Enrollment route param rename.** `:merchantSlug` actually carries the `programId` (works;
  rename for clarity).
- **Demo sandbox nightly reset.** `scripts/reset_demo.py` wipes sandbox activity; wire it to a
  schedule. Demo creds: `demo-owner@conpass.cards` / `conpass-demo-2026`.

## Deferred features (out of MVP by product decision)

- **Birthday automation** — `PUT /programs/{id}/birthday-automation`, `POST /birthday-cards`
  remain `501`.
- **Notifications / reminders** — `POST /notifications/reminders` remains `501`.
- **Real payment + messaging providers.** Payment (activation is manual via admin "Confirmar
  pago") and messaging (WhatsApp/email) are stubbed behind interfaces by design. Real
  integrations (and thus automated billing/dunning) are post-MVP.

## Dev notes (not backlog)

- Test data uses `@conpasstest.io` — the `.test` TLD is rejected by email validation.
- Frontend build needs Node 20 (`nvm use 20` + `corepack pnpm@8.15.5`); newer Node breaks workbox.
- AWS `conpass` profile is a standard `aws login` IAM-user session (not SSO); re-auth with
  `aws login` when it expires before a deploy.
