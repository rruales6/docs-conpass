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

The class carries `issuerName`, the device policy, and — since Phase 12 — the
**`classTemplateInfo.cardTemplateOverride`** that decides which rows are drawn on the front
of the card. Everything those rows *say* is still object-level (`hexBackgroundColor`, `logo`,
`heroImage` and every `textModulesData` value are object fields for Generic passes), so a
program appearance change still means touching every card. The class only describes the
**shape**; the object supplies the **content**.

`_ensure_class` therefore **PUTs a class that already exists** instead of only creating on a
404. Create-only would strand every program whose class predates the template — it would keep
rendering the old pass forever. The PUT is idempotent, so re-running costs one request.

## 2. What the holder sees

| Field | Content | Where it renders |
|---|---|---|
| `cardTitle` | merchant name | top row, next to the logo |
| `subheader` | **kind of card** — "Tarjeta de lealtad" / "Pase de membresía" | small kicker above the title |
| `header` | **program name** | the title line |
| `barcode` | opaque per-card token, QR | scanned by the cashier |
| `textModulesData` | the values behind every templated cell | the card front, via the class template |
| `hexBackgroundColor` / `logo` / `heroImage` | program appearance | card background / logo / banner |

The rows on the front, built by `_card_rows` (shape, class) and `_text_modules` (values,
object). Ids are positional and the two must agree — a cell bound to an id the object never
emits renders blank, which reads as a broken pass rather than an absent value:

| Row | Layout | `loyalty_stamps` | `loyalty_points` | `membership_pass` |
|---|---|---|---|---|
| 0 | `oneItem` | `hdr0` SELLOS `●●●○○○○○` — **the row exists only when the program draws a strip** | *(none)* | *(none)* |
| 1 | `threeItems` | `hdr1` PROGRESO `3 / 8` · `hdr2` POR CANJEAR `1` · `hdr3` CANJEADOS `2` | `hdr1` PROGRESO `120 / 200` · `hdr2` POR CANJEAR · `hdr3` CANJEADOS | `hdr1` ESTADO `Activa` · `hdr2` VÁLIDA HASTA `2026-12-31` · `hdr3` INCLUYE … |
| 2 | `oneItem` | `sec0` RECOMPENSA → the program's reward text. **Omitted from the template entirely** when the program defines no reward | | |
| 3 | `twoItems` | `sec1` MIEMBRO → holder name · `sec2` CORREO → member email | | |

Four rules behind that table:

- **The strip gets a row to itself.** At a third of the card width, eight dots wrapped onto a
  second line on a real phone. `_has_strip()` decides, and it deliberately reads only
  *program*-level facts (type and target) — a class template cannot vary per holder, so
  "does this card currently have few enough stamps" is not a question it may ask.
- **The dot strip is for recognition, the figure for reading**, so `PROGRESO` states the pair
  even when the strip is drawn. `stamp_strip()` returns `●○` only when there is a target and
  it is at most `MAX_STAMP_DOTS` (12); past that the strip row disappears and the figures
  carry it alone. The denominator is dropped only when a program has **no target at all**.
- **Claimable and claimed are different numbers.** `POR CANJEAR` is `cards.rewards_available`
  — earned and waiting — which goes *down* on a redemption. `CANJEADOS` is
  `cards.rewards_redeemed`, a separate counter that only ever climbs. A membership has
  nothing to claim, so it keeps the plain three-across row instead.
- **A missing per-card value renders `—`, not blank.** Row shape is a *program* decision, so
  a reward-less program simply has no reward row. Name and email are *per card*, and a class
  template cannot vary by card — so an anonymous enrollment gets a placeholder (D15).

> Cell labels are short on purpose (`POR CANJEAR`, not `PREMIOS POR CANJEAR`). A label that
> wraps in a third-width cell is the same defect as the strip that forced this layout.

Labels come from `_LABELS`, keyed by the card's own language (`cards.language`, captured from
the enroll form — D16), not by the holder's device locale.

## 3. Auth and transport — no Google SDK

Two RS256 JWTs signed with the service-account key via `PyJWT`, REST via `httpx`. Both
libraries already ship in the shared layer, so the integration costs **zero extra
dependencies** and no cold-start weight.

1. **API token** — a self-signed assertion (`scope: wallet_object.issuer`) exchanged at
   Google's token endpoint via the `jwt-bearer` grant. Cached until ~60 s before expiry.
2. **Save link** — a `typ: "savetowallet"` JWT appended to
   `https://pay.google.com/gp/v/save/`.

## 4. The save link and its 1800-character budget

**When the object exists, the link just references it by id (~700 chars).** That is Google's
own mitigation for the URL-length limit, and since Phase 12 it is the primary path rather
than a rescue — which is exactly what lets the card-front layout grow without the link
silently degrading. `issue()` knows the object exists because it just wrote it; `add_link()`
probes with a read, which keeps that `GET` endpoint side-effect-free.

The self-contained form remains for the one case that needs it: a link minted **without**
that assurance has to be able to create the pass on its own, because `issue()`'s REST
pre-creation is best-effort and must never block enrollment.

The constraint it exists for is size. Google: *"The safe length of an encoded JWT is 1800
characters… If the length is over 1800 characters, the save may not work due to truncation by
web browsers."* Measured, for a real program with a logo and a hero image:

| Variant | Phase 11 | Phase 12 (card-front rows) |
|---|---|---|
| **id-reference only (no object)** | 705 | **685** ✅ *(the normal path)* |
| full object + `origins` | 2 241 | — |
| full object, no `origins`, short asset keys | 1 704 ✅ | 2 035 ❌ |
| full object, no images at all | 1 375 ✅ | 1 703 ✅ |

The middle row is the point: the richer pass no longer fits the self-contained form once it
carries images. Nothing was cut to make it fit — the link stopped needing to carry it.

Which is why the payload omits two optional things, both bought back as content budget:

- **`origins`** — only matters to the JS button API; the URL form does not need it.
- **image `contentDescription`** — `cardTitle`/`header` already carry the same text.

and why asset keys are terse: `p/<prog8>/b5c72d2470e.jpg`, not
`programs/<uuid>/background-<32 hex>.jpg` (`assets.py`). Each image URL sits inside the JWT,
so ~85 characters per image is a real cost.

When the object is *not* known to exist and the self-contained payload is over the limit,
`_save_link` keeps it anyway and logs a warning: a possibly-truncated link still beats a
short link pointing at nothing.

## 5. Keeping an installed pass in sync

| Trigger | What happens |
|---|---|
| Enrollment | `issue()` — ensure class (create **or** update), create/refresh object, return the save link |
| Accrue / redeem | `update()` after commit, best-effort |
| `PATCH /programs/{id}` | `sync_program()` once for the class template, then `_push_appearance_to_passes()` for that program's cards |
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
| Balance not visible without expanding | The program's class has no card-front template yet. It is refreshed on the next enrollment or program edit — save the program in the panel |
| A front row renders blank | The class template references a `textModulesData` id the object does not emit. `_card_rows` and `_text_modules` must agree |
| Pass labels are in the wrong language | `cards.language` was captured from the PWA when that customer enrolled (D16). It is per card, not per program |
| A value wraps onto a second line | It is too wide for its cell. Give it a `oneItem` row, as the stamp strip has, or shorten the label |
| `[TEST ONLY]` prefixes the merchant name | Google's own watermark for an issuer account that is not published yet — not something in our payload. It clears when the issuer is approved |
| `NotConfigured` / 503 | `GOOGLE_WALLET_SA_JSON` missing or not minified into the Lambda env (`with_env.py`, 4 KB limit) |
| Push silently does nothing | The edit touched no `_WALLET_VISIBLE` column |

## 7. Verifying against the real API

`backend/tests/test_live_integration.py` runs the whole loop — enroll → issue a real object
→ accrue → push → revoke → clean up — against the live issuer and live Supabase. It is
skipped without secrets, so CI stays offline-safe. Unit tests mock the network and use a
throwaway RSA key.
