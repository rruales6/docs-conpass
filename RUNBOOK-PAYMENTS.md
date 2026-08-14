# Runbook — manual payments (Phase 10)

How subscription payments actually work today, what you have to do by hand, and where
every piece lives. Payments are **stubbed by design** (D9) — there is no processor. A
merchant transfers money to a bank account and uploads a receipt; a human confirms it.

> **Before anything works:** the transfer details start EMPTY. See
> [First-time setup](#first-time-setup-required-once) — until you do it, nobody can
> subscribe.

---

## The flow

| Step | Who | Where |
|---|---|---|
| 1. Publish the account to transfer into | platform-admin | `/admin` → **Datos de pago** |
| 2. Fill the signup form, pick a plan | merchant | `/merchants/subscribe` (Función 02) |
| 3. Transfer the money (QR or written details) | merchant | their bank / DEUNA app |
| 4. Upload the receipt | merchant | same page — **this unlocks the CTA** |
| 5. Submit → tenant + owner login created, subscription `pending` | (automatic) | `POST /merchants` |
| 6. Check the money actually arrived | platform-admin | your bank |
| 7. Open the receipt, then mark paid | platform-admin | `/admin` → ⋮ → **Gestionar cuenta** |

Step 5 creates a **working account with `paymentStatus: pending`**. Note what that does
*not* do: pending is a status flag only — the owner can already log in and use the panel.
Marking paid is bookkeeping, not enablement. (Enforcing suspension is a separate backlog
item; see BACKLOG P1 "Suspend enforcement".)

## First-time setup (required, once)

`platform_payment_settings` ships as a single row with every field `null`. While
`accountNumber` is empty the API reports `configured: false`, and the signup page
deliberately refuses to take money: it shows *"Todavía no publicamos los datos de
transferencia"* and leaves the activate button disabled. **No merchant can subscribe
until this is filled in.**

1. Sign in at `https://console.conpass.cards/login` as the platform admin.
2. **Datos de pago** → **Editar**.
3. Fill bank, account type, account number, beneficiary, RUC, contact email; upload the
   DEUNA QR (PNG/JPG); optionally add free-text instructions.
4. Save, then load `/merchants/subscribe` in a logged-out window and confirm the details
   render and the dropzone appears.

Changing the bank account or the QR later is the same screen — **no redeploy, no
migration**. That was the point of putting it in the DB.

## Verifying a payment

⋮ → **Gestionar cuenta** shows **Ver comprobante** when a receipt exists, with its upload
date, directly above **Marcar como pagado**. The link is a presigned S3 GET valid for
**5 minutes** — it opens in a new tab. If it expires, close it and click again; nothing is
broken.

When no receipt exists the modal says so plainly. Treat that as a red flag: every account
created after Phase 10 has one, so a missing receipt means either a pre-Phase-10 signup or
something odd.

## Where everything lives

**Database** (migration `0008_payment_settings.sql`)

- `platform_payment_settings` — **singleton**: `id boolean primary key default true
  check (id)`, so a second row raises instead of silently forking what signup displays.
  Columns: `bank_name`, `account_type` (`savings`|`checking`), `account_number`,
  `beneficiary_name`, `beneficiary_tax_id`, `contact_email`, `instructions`,
  `qr_storage_key`, `updated_at`.
- `subscriptions.payment_proof_key` + `payment_proof_uploaded_at` — the receipt, written
  **only at signup**. See [Known limits](#known-limits).

RLS forced, `anon`/`authenticated` revoked, explicit `service_role` DML — same lockdown as
migrations 0004/0005. Backend-only access.

**S3 — two buckets, different on purpose**

| Bucket | Access | Holds |
|---|---|---|
| `conpass-program-assets-prod` | **public-read** | program icons/backgrounds, and the payment QR under `platform/` — it is shown to anonymous visitors, so it must be public |
| `conpass-payment-proofs-prod` | **private** (all four public-access blocks on) | receipts under `payment-proofs/` — they carry names, account numbers and amounts |

Never move receipts into the public bucket to "simplify" things.

**Endpoints**

| Endpoint | Auth | Notes |
|---|---|---|
| `GET /payment-settings` | public | what the signup page renders |
| `POST /payment-proofs/upload-url` | **public** | presigned PUT; see [Security](#security-properties-and-their-limits) |
| `PATCH /admin/payment-settings` | platform_admin | partial — only fields sent are written; `""` clears one |
| `POST /admin/payment-settings/qr-upload-url` | platform_admin | presign into the **public** bucket |
| `GET /admin/clients/{merchantId}/payment-proof` | platform_admin | 5-min presigned GET; `404` when none |

**Code** (`bke-conpass`)

- `layers/common/python/conpass_common/payment_settings.py` — the row→API mapper, in the
  shared layer **on purpose**: both the public GET and the admin PATCH render the same
  shape, and one copy cannot drift from the other. Change the shape here, nowhere else.
- `layers/common/python/conpass_common/assets.py` — `presign_payment_proof_upload` /
  `_download`, `presign_payment_qr_upload`. `boto3` is imported lazily (it exists only in
  the Lambda runtime, not the dev venv) — keep it that way or the layer stops importing
  in tests.
- `services/merchants/handler.py` — public GET + upload-url, and the onboarding guard.
- `services/admin/handler.py` — the three admin endpoints.
- `tests/test_payment_settings.py` — 16 tests; boto3 is mocked, no network.

**Code** (`fte-conpass`)

- `apps/pwa/src/pages/public/MerchantSignupPage.tsx` — method tabs, transfer details,
  dropzone, CTA gating.
- `apps/pwa/src/pages/admin/AdminDashboardPage.tsx` — `PaymentSettingsSection` +
  `PaymentSettingsModal`, and the proof viewer inside `ManageClientModal`.
- i18n keys under `merchantSignup.*` and `admin.paymentSettings*` / `admin.paymentProof*`
  in **both** `es.json` and `en.json` (non-mirror: Spanish is the source).

## The activation gate

The requirement was "don't let anyone activate without a receipt". That is enforced in
**two independent places**, and both must stay:

1. **UI** — the CTA is disabled until the upload succeeds, and says why in two places
   (a notice above the button, a helper under it). This is UX, not security.
2. **API** — `POST /merchants` returns **422** when `payment.method` is `deuna` or
   `manual_transfer` and `payment.proofStorageKey` is absent. This is the actual gate; it
   holds against anyone calling the API directly.

If you ever add a payment method that legitimately needs no receipt, add it *outside*
`PROOF_REQUIRED_METHODS` in `services/merchants/handler.py` rather than weakening the check.

**Card payments are not integrated.** The card tab exists in the design, so it renders —
but `disabled`, badged *Próximamente*. It cannot select, cannot submit. Enabling it means
integrating a real processor (BACKLOG).

## Security properties, and their limits

What holds:

- Receipts are **not publicly readable** — verified in prod: a direct GET on an uploaded
  object returns `403`. Reading one requires an admin-signed URL.
- The upload key is server-generated and random, so a caller can only write to the path it
  was just handed — it cannot overwrite someone else's receipt or guess one.
- Content type is constrained to PNG/JPEG/PDF and baked into the signature.

What does **not** hold, and you should know it:

- `POST /payment-proofs/upload-url` is **unauthenticated by necessity** — the account does
  not exist yet at upload time. Anyone can mint upload URLs and push ≤ a few MB into the
  private bucket. There is no rate limit on it today.
- Nothing deletes orphaned objects (an upload where the visitor then abandoned the form),
  and the bucket has **no lifecycle rule**. Storage grows monotonically.
- The 5 MB cap is enforced **client-side only**; a presigned PUT does not carry a size
  condition. A direct caller can exceed it.

All three are backlogged together (P1, "Harden the public payment-proof upload"). They are
acceptable for MVP volume — the blast radius is S3 cost, not data exposure — but they are
not acceptable at scale.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Signup shows "Todavía no publicamos los datos de transferencia" | `accountNumber` is empty → do [First-time setup](#first-time-setup-required-once). |
| Upload button fails from the console with a CORS error | The bucket's `CorsConfiguration` is missing the origin. Both buckets must list `console.conpass.cards` (this was actually missing on the assets bucket and fixed in Phase 10). |
| `503 not_configured` from an upload-url endpoint | `PAYMENT_PROOFS_BUCKET` / `PROGRAM_ASSETS_BUCKET` env var missing on the Lambda — redeploy. |
| `422` on `POST /merchants` mentioning `proofStorageKey` | Working as designed: the receipt is required. |
| `PGRST204` / a new column "does not exist" right after a migration | PostgREST hasn't reloaded. `notify pgrst, 'reload schema'` and wait a few seconds. |
| Proof link 403s | The 5-minute presigned GET expired — click **Ver comprobante** again. |
| Admin sees `hasPaymentProof: false` for an old client | Expected for anything created before Phase 10; only signups after it carry a receipt. |

## Known limits

- **No renewal-month uploads.** The receipt is a single column written only at signup, so
  "Gestionar cuenta" keeps showing the *signup* receipt while you decide about a later
  month. Worse, if a renewal-upload path is ever added against that same column, a new
  upload **overwrites** the previous month's receipt irrecoverably. The fix when monthly
  billing gets real: a `payment_proofs` history table plus a "newest" ordering on the admin
  endpoint — the modal already shows only the latest, so no frontend rework. Deliberate
  product decision to ship the single column; see BACKLOG P1.
- **No automatic reconciliation.** Nothing checks your bank. Marking paid is entirely
  manual, and nothing reminds you that a `pending` client has been pending for a month.
- **`mrrUsd` is derived from the tier**, not from money actually received.
