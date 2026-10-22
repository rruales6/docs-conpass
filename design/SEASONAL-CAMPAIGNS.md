# Seasonal campaigns ("campañas de temporada") — design proposal

Status: **APPROVED 2026-10-21** (Phase 14) with the owner decisions recorded under
"Owner decisions" below — where they differ from the original proposal, they win.

## What it is

A seasonal campaign is a **discount card tied to a commemorative date**: Día de la
Madre, San Valentín, Navidad, Black Friday, Fiestas de Quito… The customer saves it to
their wallet. During the campaign's dates the cashier scans it and applies the
discount. After the dates, the pass shows as expired.

It sits beside the two kinds that exist today:

| Kind | What the customer does | Lives |
|---|---|---|
| **Campaña fija** (loyalty_stamps / loyalty_points) | collects stamps/points towards a reward | indefinitely |
| **Campaña de temporada** (new: `seasonal_discount`) | uses a discount, usually once | only between two dates |
| Pase de membresía (unchanged) | shows an active membership | per validity |

## Data model — migration 0013

- `program_type` gets a new value `seasonal_discount`.
- `programs` gets four new columns, all NULL for other kinds:
  - `occasion text` — e.g. "Día de la Madre".
  - `discount_percent integer` (1–100).
  - `valid_from date` and `valid_until date`, with a check that `valid_until >= valid_from`.
- The existing `reward` column holds the benefit text, e.g. "20 % en toda la carta" or
  "2x1 en postres", so a non-percentage offer is possible.
- **No new card columns.** `cards.rewards_redeemed` (migration 0010) counts uses.
- **No new transaction kind.** Applying the discount is a `redeem`, so the existing
  redemptions report and metrics include it for free.

## API (contract-first)

- `ProgramType` gains `seasonal_discount`. `ProgramCreate` and `Program` gain
  `occasion`, `discountPercent`, `validFrom` and `validUntil`.
  - For seasonal campaigns, the server requires both dates and at least one of
    `discountPercent` / `reward`. Otherwise it returns 422.
- **Enrolment** is open only while the campaign is active (decision 3): before
  `validFrom` → 422 `campaign_not_started`, after `validUntil` → 422 `campaign_ended`.
- **Cashier**: `POST /operations/resolve` already returns the program, so the cashier
  sees "Campaña de temporada · 20 % · Día de la Madre · 1–12 may".
  - "Aplicar descuento" calls the existing idempotent `POST /operations/redeem`. It
    stays offline-safe, uses the same queue, and cannot double-count on replay.
  - The server refuses with a reason the cashier screen translates:
    `campaign_not_started`, `campaign_ended`, or `discount_already_used` (when
    `rewards_redeemed >= 1`).
  - Accrual on a seasonal card is refused with 422.
- "Today" is evaluated in **America/Guayaquil**, because no merchant time zone is
  stored yet.

## Wallet pass (Google Generic) — as built

- Kicker: "Campaña de temporada · <occasion>" / "Seasonal offer · <occasion>". The title
  is the campaign name; the card title is the merchant.
- Front rows, using the same Phase 12 template mechanism:
  - Row 1: DESCUENTO `20 %` · VÁLIDA `1–12 may` · ESTADO `Disponible` / `Usado`. A
    text-only offer has no percentage, so that campaign's row 1 is just VÁLIDA · ESTADO.
    Row shape depends only on the campaign, never on the card, because it lives on the
    class.
  - Row 2: BENEFICIO, which is the `reward` text. The row is omitted if empty.
  - Row 3: MIEMBRO · CORREO, the same as other cards.
- The object sets **`validTimeInterval`**, from 00:00 on `validFrom` to 23:59:59 on
  `validUntil`, at UTC−5. Google itself shows the pass as not yet valid or expired, so
  the start and end of a campaign need no push. ESTADO changes only on the one real
  event, applying the discount. The pass is kept, not revoked.

## Panel and metrics

- In the "Nueva campaña" modal, the type toggle gets a third option, **Temporada**. Its
  fields are:
  - occasion, with Ecuador date suggestions: San Valentín 14 feb, Día de la Madre (2nd
    Sunday of May), Día del Padre (3rd Sunday of June), Fiestas de Quito, Navidad,
    Black Friday;
  - discount %;
  - benefit text;
  - from / until dates.
- In "Mis campañas", each seasonal campaign shows a status badge: **Próxima**, **Activa**
  or **Finalizada**.
- Metrics for a seasonal campaign show cards saved, discounts applied, and the share of
  saved cards actually used.
- The abandonment rate does not apply, so it is omitted, as it is for membership passes.

## Owner decisions (2026-10-21)

1. **All plans** get seasonal campaigns — no feature gate. Notifications about them are
   limited by each plan's notification allowance (when notifications ship).
2. **One use per customer, always.** No `usesPerCard` setting; `rewards_redeemed >= 1`
   means used.
3. **Only while the campaign is active.** A single status decides both enrolment and the
   cashier: `active` = the campaign is switched on AND today is within
   [`validFrom`, `validUntil`]. Before the start date the campaign is *upcoming* and
   nobody can save the card yet; after the end date it is *ended*.
4. **Expired cards stop being billed.** A seasonal card whose campaign has ended no longer
   counts as an active card (`merchant_active_passes`).
5. **Offer = percentage and/or free text**, at least one required.

## Size

The work is about the size of Phase 12:

- one migration;
- contract additions;
- programs, enrolment and operations handlers;
- the wallet provider's rows;
- the frontend: modal, list badges, cashier result and enrolment page.

Notifications stay out of scope, since they are still 501.
