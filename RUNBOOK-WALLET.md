# Runbook — Google Wallet passes

How a conpass loyalty card becomes a pass in someone's Google Wallet, what the pass shows,
how it stays in sync, and what to check when it doesn't. Companion to
[RUNBOOK-PAYMENTS.md](RUNBOOK-PAYMENTS.md).

Everything lives in `backend/layers/common/python/conpass_common/providers/`:
`wallet.py` (vendor-neutral) and `google_wallet.py` (the Google implementation). Feature
Lambdas never import a vendor SDK — they depend on the `WalletProvider` interface, which is
what lets Apple Wallet slot in later (D8).

## 1. The object model

Google splits every pass into a **class** (template) and an **object** (one holder's copy).
We map that straight onto our domain, with IDs derived from our own UUIDs — nothing
Google-side needs to be stored to find a pass again:

| Google | conpass | id |
|---|---|---|
| `genericClass` | program | `{issuerId}.program_{programId}` |
| `genericObject` | card | `{issuerId}.card_{cardId}` |

Issuer `3388000000023155712`, service account `conpass-generic@conpass-502402`. One shared
issuer for every merchant (B2).

The class is intentionally thin — `issuerName` and device policy. **All visible content is
on the object**, because for Generic passes `hexBackgroundColor`, `logo` and `heroImage` are
object-level fields. That is why changing a program's appearance means touching every card,
not one class.

## 2. What the holder sees

| Field | Content | Where it renders |
|---|---|---|
| `cardTitle` | merchant name | top row, next to the logo |
| `subheader` | program name | small label above the title |
| `header` | **the tracked balance** — `3 / 8 sellos`, `120 / 200 puntos`, `Activa hasta …` | the title line |
| `barcode` | opaque per-card token, QR | scanned by the cashier |
| `textModulesData` | holder, reward, validity | details — only after the holder expands the pass |
| `hexBackgroundColor` / `logo` / `heroImage` | program appearance | card background / logo / banner |

`_status_line()` builds the balance text. The balance is the **title**, not a text module:
Google only shows `textModulesData` once the pass is expanded, which is too late for
"how many stamps do I have".

## 3. Auth and transport — no Google SDK

Two RS256 JWTs signed with the service-account key via `PyJWT`, REST via `httpx`. Both
libraries already ship in the shared layer, so the integration costs **zero extra
dependencies** and no cold-start weight.

1. **API token** — a self-signed assertion (`scope: wallet_object.issuer`) exchanged at
   Google's token endpoint via the `jwt-bearer` grant. Cached until ~60 s before expiry.
2. **Save link** — a `typ: "savetowallet"` JWT appended to
   `https://pay.google.com/gp/v/save/`.

## 4. The save link and its 1800-character budget

The JWT embeds the **whole `genericObject`**, per Google's web guide, so the link can create
the pass by itself — the REST pre-creation in `issue()` is best-effort and must never block
enrollment, so the link cannot depend on it having worked.

The constraint is size. Google: *"The safe length of an encoded JWT is 1800 characters… If
the length is over 1800 characters, the save may not work due to truncation by web
browsers."* Measured, for a real program with a logo and a hero image:

| Variant | chars |
|---|---|
| id-reference only (no object) | 705 |
| full object + `origins` | 2 241 |
| full object, no `origins` | 2 033 |
| full, no `origins`, no image alt-text | 1 875 |
| **full, short asset keys** | **1 704** ✅ |
| full, no images at all | 1 375 ✅ |

Which is why the payload omits two optional things, both bought back as content budget:

- **`origins`** — only matters to the JS button API; the URL form does not need it.
- **image `contentDescription`** — `cardTitle`/`header` already carry the same text.

and why asset keys are terse: `p/<prog8>/b5c72d2470e.jpg`, not
`programs/<uuid>/background-<32 hex>.jpg` (`assets.py`). Each image URL sits inside the JWT,
so ~85 characters per image is a real cost.

**Fallback.** If the embedded form still exceeds the limit, `_save_link` falls back to
Google's documented mitigation — reference the already-created object by id (705 chars).
That is only safe when the object exists, hence `object_exists=`; `issue()` knows it does,
and `add_link()` probes with a read (keeping that `GET` endpoint side-effect-free). With no
such assurance it keeps the oversized self-contained link and logs a warning, because a
possibly-truncated link still beats a short link pointing at nothing.

> **Programs whose images predate the short-key format** (long `programs/…` keys) land at
> ~1 875 and therefore fall back. Re-uploading either image in the panel fixes it. Serving
> assets from a short host would fix it for everyone — see BACKLOG.

## 5. Keeping an installed pass in sync

| Trigger | What happens |
|---|---|
| Enrollment | `issue()` — ensure class, create/refresh object, return the save link |
| Accrue / redeem | `update()` after commit, best-effort |
| `PATCH /programs/{id}` | `_push_appearance_to_passes()` — pushes to that program's cards |
| Cleanup / live test | `revoke()` — `state: INACTIVE` (Google objects cannot be hard-deleted) |

`update()` is a **PUT — a full replace**, not a patch. Two reasons: appearance edits have to
reach passes already on phones, and only a replace can *clear* an image the merchant
removed (a PATCH leaves omitted fields alone). The payload comes entirely from DB rows, so
replacing is exactly what `issue()` already does to an existing object.

The program push is **best-effort and bounded** by `WALLET_PUSH_MAX_CARDS` (40) — each card
is a separate API call inside a 15 s Lambda. Cards beyond the cap refresh on their next
operation. It fires only when a column the pass actually shows changed (`_WALLET_VISIBLE`),
so toggling `active` costs nothing. A wallet outage never fails the program edit.

## 6. Troubleshooting

| Symptom | Likely cause |
|---|---|
| Pass shows an old colour / logo / reward | It predates the sync fix, or the program has more than `WALLET_PUSH_MAX_CARDS` cards. Re-save the program, or operate on the card |
| "Add to Google Wallet" opens an error | The link referenced an object that was never created — check the `issue()` warning in CloudWatch |
| Save link looks truncated | Over 1800 chars. Check the warning log; shorten the asset URLs |
| Balance not visible without expanding | Pass was issued before the balance moved to `header` — any update refreshes it |
| `NotConfigured` / 503 | `GOOGLE_WALLET_SA_JSON` missing or not minified into the Lambda env (`with_env.py`, 4 KB limit) |
| Push silently does nothing | The edit touched no `_WALLET_VISIBLE` column |

## 7. Verifying against the real API

`backend/tests/test_live_integration.py` runs the whole loop — enroll → issue a real object
→ accrue → push → revoke → clean up — against the live issuer and live Supabase. It is
skipped without secrets, so CI stays offline-safe. Unit tests mock the network and use a
throwaway RSA key.
