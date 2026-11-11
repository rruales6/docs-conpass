# Wallet notifications, locales and balance expiry — Phase 16 design

Source of the UX: the owner's design file *Conpass Wallet Pass Platform Design – Conpass UI*
(screen "Panel del comercio · Notificaciones", tabs **Automatizaciones · Envío manual ·
Locales y cercanía · Historial**, plus Función 04 "C · Reglas de envío de notificaciones").
Contract: `backend/contracts/openapi.yaml` (tags `notifications`, `merchants` locations,
`admin` notification-rules). Schema: `backend/db/migrations/0015_notifications.sql`.

## Owner decisions (2026-11-11)

| Topic | Decision |
|---|---|
| Balance expiry | By **inactivity**, a **merchant-wide general parameter** (`balanceExpiryDays`, null = never). Expiry really zeroes stamps/points; rewards already earned are kept. Warned `warnDays` before. |
| Level | Starts at **1** and goes up by one **each time the customer reaches the reward**: `level = 1 + rewards_available + rewards_redeemed` (loyalty campaigns only). No new column. Shown on the pass. |
| Locales | Owner registers locales (name + lat/lng). Cashiers are assigned **one or more** locales; on entering the scan screen a cashier with 2+ picks one; every scan records `transactions.location_id`. Enables the "Clientes de <local>" audience and per-card wallet locations. |
| Personalised reward (unique code per customer) | **Out** — later phase. The composer has no "Recompensa personalizada" type. |
| Membership/access passes | Still parked — never notified. |

## What Google Wallet actually gives us (Generic passes)

* **addMessage** with `messageType: TEXT_AND_NOTIFY` → message on the back of the pass +
  lock-screen push. **Max 3 per pass per 24 h**; a 4th returns QuotaExceeded → we resend it
  as `TEXT` (silent, not billed). Google renders the notification itself.
* `Message.displayInterval` → the wallet hides the message on its own (flash sale closes,
  event over). Links: an `<a href>` in the body; must relate to the program (Google AUP).
* **merchantLocations** (≤10 on the class, ≤10 on the object) → "Estás cerca". Google
  decides radius, dwell time and text; only for holders with notifications on and precise
  always-on location. A reinforcement, not a guaranteed send.
* No lock-screen "event day" highlight on Google Generic (that is Apple `relevantDate`;
  Apple Wallet is not built yet). The UI must not promise it.
* **Our object/class writes are full PUTs** (Phase 11/12: needed to clear images). A PUT
  clears every field it omits — so `messages` and `merchantLocations` are **DB-derived and
  re-sent on every write**. `conpass_common/wallet_sync.py` is the only path that builds a
  pass from rows; feature code never builds `PassContent` by hand anymore.

## Data model (0015)

`merchant_locations`, `operation_user_locations`, `transactions.location_id`,
`cards.enrolled_location_id`, `merchant_notification_settings` (missing row = defaults),
`platform_notification_rules` (singleton), `notifications` (one per manual send; one per
(merchant, automation type, Ecuador day)), `notification_deliveries` (the outbox, one row
per card), `balance_expirations` (audit; deliberately not a transaction kind so an expiry
never counts as "the customer came back"). Views: `card_last_activity`,
`notifiable_cards`, `notification_history_view`. Functions (service_role only):
`notification_audience`, `enqueue_manual_notification`, `claim_notification_deliveries`.

**Notifiable card** = active card of an active loyalty campaign, or of a seasonal campaign
running today (Ecuador). Not membership passes.

## Automations

Defaults (design): balance, ready, near, milestone **on**; expiry on but inert until the
merchant sets `balanceExpiryDays`; inactive **off** (Pro).

Event automations — evaluated **after an accrual commits** (operations service), best-effort
(a failure never fails the cashier operation), **at most one per accrual**, priority
`ready > milestone > near > balance`:

| id | fires when | plan |
|---|---|---|
| ready | the accrual completed ≥1 reward | every plan |
| milestone | the card's accrual count is a multiple of `every` | every plan |
| near | no reward completed AND the visit brought `missing` from above the threshold to `1..threshold` (stamps: `near.stamps`, points: `near.points`) | every plan |
| balance | every accrual | every plan |

Enqueuing an event automation marks that card's still-`pending` **balance/near** deliveries
`superseded` (a stale "le faltan 2" must not go out after the next visit).

Time-based automations — the notifier's **daily run at 08:00 Ecuador** (13:00 UTC):

| id | rule | dedupe_key |
|---|---|---|
| expire (no message) | `balanceExpiryDays` set, balance > 0, `last_at <= now - N days` → stamps/points = 0, row in `balance_expirations`, `push_card` | — |
| expiry | `expiry_on`, N set, balance > 0, `now - N < last_at <= now - (N - warnDays)` | `expiry:<card>:<last_at date>` |
| inactive | `inactive_on` AND plan has `automated_campaigns`, `now - days - 7d < last_at <= now - days` | `inactive:<card>:<last_at date>` |

`last_at` = `card_last_activity.last_at` (last accrual/redeem, enrolment if none).

### Message texts (usted register, es/en by the card's language; header = business name)

`{reward}` = program reward text, fallback "su recompensa" / "your reward".

| id | es | en |
|---|---|---|
| balance | Sumó {n} sello(s). Lleva {held} / {target}: le faltan {missing} para su recompensa: {reward}. *(points: "Sumó {n} puntos…")* | You earned {n} stamp(s). You have {held} / {target}: {missing} to go for {reward}. |
| ready | ¡Completó su tarjeta! Su recompensa le espera: {reward}. Al canjearla, su tarjeta empieza de nuevo. | Card complete! Your reward is waiting: {reward}. Once redeemed, your card starts again. |
| near | Le falta(n) {missing} sello(s)/puntos para su recompensa: {reward}. ¿Pasa hoy? | Only {missing} stamp(s)/points to go for {reward}. Drop by today? |
| milestone | ¡Su visita n.º {n}! Gracias por volver a {merchant}. | Visit number {n}! Thank you for coming back to {merchant}. |
| expiry | Sus {held} sellos/puntos vencen en {d} día(s). Úselos antes de perderlos. | Your {held} stamps/points expire in {d} day(s). Use them before you lose them. |
| inactive | Hace {d} días que no le vemos. Lleva {held} / {target}: su recompensa está cerca. | It's been {d} days. You have {held} / {target}: your reward is close. |

Singular/plural handled properly (1 sello / 2 sellos).

## Manual sends

Types: `general` (every plan), `event` and `flash` (Growth+). Audiences: `all` (every plan),
`near` (merchant thresholds), `inactive` (30 days), `level` (≥ levelMin), `location`
(signed up or scanned there) — Growth+. `url` Growth+. Body ≤150, title ≤60 (defaults to the
business name). `scheduledAt` optional (future, ≤60 days). Flash → message hidden
`flashHours` after send; event → hidden 6 h after `eventAt`.

Flow: POST inserts `notifications` (status `queued`, or `scheduled`) and for "now" calls
`enqueue_manual_notification` (audience expanded in SQL, one round trip). The dispatcher
expands due `scheduled` rows the same way. DELETE cancels a `scheduled` one (409 otherwise).

## Dispatcher (`notifier` Lambda, every minute)

1. Expand due scheduled notifications.
2. Claim due deliveries (`claim_notification_deliveries`, SKIP LOCKED; batches of 200,
   time-boxed), then per delivery:
   * card no longer notifiable → `skipped`;
   * quiet hours on and now in 21:00–08:00 Ecuador → back to `pending`, `not_before` = next 08:00;
   * card already has `cap_per_day` push-`sent` deliveries in the last 24 h → manual:
     `pending` until the oldest of those + 24 h ("conpass guarda las demás para el día
     siguiente"); automation: `skipped` (error `daily_cap`);
   * else `wallet_sync.send_card_message(notify=True)` → `sent`; QuotaExceeded → resend
     silent → `sent_silent`; other error → retry (5 min × attempt) up to 3, then `failed`.
3. Notifications with no `pending`/`sending` deliveries left → `sent`.

Small thread pool (≤8) for wallet calls. Reserved concurrency keeps runs from piling up.

## Billing

`notificationsUsed` = deliveries with status `sent` (push delivered) this calendar month,
UTC (same period as cards). Over the plan's inclusion → billed at the plan rate, never
blocked (D17). Silent and skipped ones are not billed.

## Plan gating (plans.py features)

| Feature | Starter | Growth | Pro |
|---|---|---|---|
| Automations balance/ready/near/milestone/expiry | ✓ | ✓ | ✓ |
| Manual `general` to `all`, no link | ✓ | ✓ | ✓ |
| Segments, `event`, `flash`, link (`manual_campaigns`) | — | ✓ | ✓ |
| Nearby: locales + per-card locations sent to the wallet (`nearby_notifications`) | stored, not sent | ✓ | ✓ |
| Inactivity reminder (`automated_campaigns`) | — | — | ✓ |
| Locales for cashier attribution | ✓ | ✓ | ✓ |

Plan changes (admin PATCH) and every locale change re-sync the merchant's classes.

## Pass changes (wallet specialist)

* Back-of-pass text modules: **NIVEL / LEVEL** (loyalty) and, when the merchant set an
  expiry window and the balance is > 0, **VENCE / EXPIRES** (`last_at + N`, Ecuador date).
  Card-front template unchanged.
* `messages` = the card's last 10 `sent`/`sent_silent` deliveries whose `display_until` has
  not passed (id = delivery id, `displayInterval.end` = display_until).
* Class `merchantLocations` = first 10 active locales (Growth+ else []); object
  `merchantLocations` = enrolment locale + most-visited locales (≤10, per settings,
  Growth+ else []).

## Public demo merchant (owner, 2026-11-11)

The demo has shared public credentials, so it must not become a spam channel:

* **Cards last 7 days.** The notifier's daily run deactivates demo cards enrolled more than
  7 days ago and revokes their Google pass (best-effort). Only recent demo cards are
  notifiable. The seeded sample card (`dedupe_key = demo-sample-card`) is kept — the demo
  cashier pre-fills it, and it has no Google object anyway.
* **At most 3 manual sends per Ecuador day**, any status (scheduled and cancelled count) →
  429 `rate_limited`, `details.reason = demo_daily_limit`.
* **No automations**: nothing is enqueued after an accrual, and the daily job skips demo
  merchants (no expiry warnings, inactivity reminders or balance expiry). Settings stay
  viewable/editable; the PWA says they are illustrative.

Code: `conpass_common/demo.py` (`is_demo_merchant`, `manual_sends_today`, `expire_demo_cards`,
`DEMO_CARD_TTL_DAYS`, `DEMO_DAILY_MANUAL_SENDS`).

## Out of scope / backlog

Personalised reward codes; Apple Wallet (relevantDate, field-change notifications); birthdays
(still Pro automation, still 501); A/B or per-merchant message editing; SMS/WhatsApp.
