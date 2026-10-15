# Getting started with the GlampSys Partner API

This API lets a booking portal pull availability and rates for glamping units,
get a binding price for a specific stay, hold the dates while the guest pays,
and confirm the reservation once payment has cleared.

The full reference is generated from the OpenAPI 3.1 contract and is the source
of truth. This page explains how the pieces fit together.

## Implemented scope

Implementation status below is not a claim that a particular partner account
or credential has been provisioned. Validate your issued sandbox token before
starting development. Confirming reservations is live in the contract; whether
your account may confirm real (production) orders is agreed separately — start
in the sandbox.

| Endpoint | Status |
|---|---|
| `GET /ping` | live |
| `GET /units` | live |
| `GET /ari` | live — availability, indicative rates and daily restrictions |
| `POST /quotes` | live (1.2.0) — binding price, with or without holding the dates |
| `GET /quotes/{quote_id}`, `DELETE /quotes/{quote_id}` | live (1.2.0) — read and release a hold |
| `POST /reservations/confirm` | live (1.3.0) — turn a held quote into a paid reservation |
| `GET /reservations/{partner_reservation_id}`, `GET /reservations` | live (1.3.0) — look up one reservation, reconciliation feed |
| `POST /reservations/{partner_reservation_id}/cancel` | live (1.4.0) — cancel a reservation and release its nights |
| reservation modify, webhooks | planned — in the contract, marked `x-status: planned` |

Planned operations are published early on purpose, so you can build against the
final contract before we deploy them. Their shape will not change without a
version bump.

## Authentication

Every request carries a bearer token:

```
Authorization: Bearer gsp_live_…
```

You receive two separate tokens:

- `gsp_test_…` — **sandbox**. Sees only the units of a demo property. Nothing
  you do with it can affect a real guest, a real calendar or a real channel.
- `gsp_live_…` — **production**. Issued after the acceptance run against the
  sandbox passes.

Never share a token between environments. Tokens can be rotated at any time
without changing the contract; during a rotation both the old and the new token
work for an agreed overlap.

A missing, unknown or revoked token returns `401 UNAUTHORIZED`. A valid token
whose access is closed — your account is paused, or your source IP is outside
the configured allow-list — returns `403 FORBIDDEN`.

Start by calling `GET /ping`. It checks the token and tells you how many units
you can reach, without creating a reservation or holding inventory. Read
requests still update operational token-usage timestamps and rate counters.

## Read-only first run

The base URL is `https://glampsys.cz/api/partner/v1`. Use an individually
issued sandbox token, never a production token, for this acceptance sequence:

1. `GET /ping` — expect `200`, `environment: "sandbox"` and the agreed demo
   unit count.
2. `GET /units` — save the returned `unit_id` or `partner_unit_id`.
3. `GET /ari?unit_id=SANDBOX-1&from=2027-06-11&to=2027-06-13&currency=CZK`
   — replace the alias and dates with your agreed demo mapping and a suitable
   future range. `unit_id` is **required**; repeat it for multiple units
   (`unit_id=SANDBOX-1&unit_id=SANDBOX-POOL`). `unit_ids` is not a parameter.
4. Repeat the same units/ARI request with its `ETag` in `If-None-Match`:
   unchanged data returns `304` without a body.

ARI includes both `from` and `to`, allows at most 366 days and 100 units in
one call, and uses date-only `YYYY-MM-DD` values for local calendar nights.
Each day has `available`, `available_count`, `min_stay`, `arrival_allowed`,
`departure_allowed`, and `price`. A price is
`{"amount_minor": 320000, "currency": "CZK"}`, not a decimal nightly amount.
`price: null` can mean an unsellable night; when EUR is not configured it
also has `price_unavailable_reason: "CURRENCY_NOT_ENABLED"`. Do not replace
that with a made-up currency conversion.

For this phase, call only these three GET endpoints. ARI is not a binding
quote and must not be used to collect payment.

## Identifiers

- `unit_id` (`unit_123`) and `property_id` (`prop_7`) are opaque and stable. Do
  not parse them.
- `partner_unit_id` is **your** identifier for a unit, agreed during
  onboarding. Wherever an endpoint takes a unit, you may send either form.
- A unit that is not mapped to you returns `404 NOT_FOUND`, never `403`, so the
  response does not reveal whether it exists.

## The complete booking flow

```
GET    /units                       once a day — conditions of stay
GET    /ari?unit_id=…&from=…&to=…    every 1–3 h — availability, rates, restrictions
POST   /quotes      hold: false     "check availability" on the listing page
POST   /quotes      hold: true      guest submits the order — dates are held
DELETE /quotes/{quote_id}           guest abandoned checkout — release the dates
POST   /reservations/confirm        after your payment gateway confirms payment
GET    /reservations?updated_since=  reconciliation — what changed on our side
```

1. **Pull conditions and availability.** `GET /units` returns check-in times,
   minimum stay, allowed arrival days, cancellation rules and the local tax.
   `GET /ari` returns one row per unit and day. Send `If-None-Match` with the
   `ETag` you received; an unchanged response comes back as `304` with no body.
2. **Never rebuild the final price from ARI.** ARI prices are indicative.
   The binding price of a stay — promotions, minimum-stay rules, local tax —
   always comes from `POST /quotes`.
3. **Check without holding.** `POST /quotes` with `"hold": false` returns
   `200` with the binding price and real-time availability without blocking
   the dates (`status: "priced"`, `quote_id: null`). Use it on the listing
   page; clicks and bots do not consume inventory and need no
   `Idempotency-Key`. Do not send `guest` with it — that is a `400`.
4. **Hold while the guest pays.** `POST /quotes` with `"hold": true` (and an
   `Idempotency-Key`) returns `201` with a `quote_id`, `status: "held"`,
   `expires_at` and `hold_ttl_seconds`. The hold is serialized against every
   other sale of the same unit, including guests booking on the operator's own
   website: another request for the same dates gets `409 UNAVAILABLE` until
   the hold expires or you release it. You may send the `guest` you already
   know — the operator then sees the name on the hold in their calendar; it is
   never echoed back and is erased 24 hours after the quote ends.
5. **Release a hold you no longer need.** `DELETE /quotes/{quote_id}` frees the
   dates at once and returns the quote with `status: "cancelled"`. It is
   idempotent — releasing an already released or expired quote returns `200`
   with its current status. A confirmed quote cannot be released (`409`).
6. **Confirm after payment.** `POST /reservations/confirm` (with an
   `Idempotency-Key`) turns the held quote into a reservation atomically and
   returns `201` with `reservation_id` and `guest_portal_url`. Send your own
   `partner_reservation_id`, the `guest`, the `consents` you collected
   (`terms` and `gdpr` must be `true`) and the `payment` the guest made:
   GlampSys keeps the quote's price and only checks that the payment is in
   the quote's currency and within your agreed tolerance of
   `total_price_minor` (otherwise `409 PRICE_MISMATCH`). A quote can be
   confirmed once (`409 QUOTE_ALREADY_USED` afterwards).

### Late payment

Your payment can finish after the hold's `expires_at`. Confirm anyway — for
**24 hours after `expires_at`** GlampSys re-checks the stay under the same
lock every other sale takes (unit still yours and active, location not paused,
nights in season, arrival not in the past, dates free). If the stay can still
be sold, the reservation is created at the quoted price. If not, you get
`409 UNAVAILABLE` with `error.details.reason = "HOLD_EXPIRED"`: nothing was
created, refund the guest. After 24 hours, or if you released the quote
yourself with `DELETE`, confirming is `410 QUOTE_EXPIRED`
(`error.details.status` = `expired` / `cancelled`).

The property's owner can pause sales. A pause set while your hold runs covers
the hold; confirming inside it is `409 UNAVAILABLE` with
`error.details.reason = "SALES_PAUSED"` — refund the guest.

### Guest emails after confirmation

GlampSys emails a guest only after confirmation, in `guest.language`, and only
what the email policy agreed for that location allows: a booking confirmation
without any payment details (it names you and your `partner_reservation_id` and
links the online check-in), arrival instructions and other stay emails, or
nothing at all. It never sends a guest of your reservation a payment request,
a reminder or an invoice — you collected the money. If you send your own
confirmation, include `guest_portal_url` from the confirm response; it is the
guest's online check-in.

### Cancelling a reservation

`POST /reservations/{partner_reservation_id}/cancel` with an `Idempotency-Key`
(the body `{ "reason": "guest_request" }` is optional, for our records only)
cancels the reservation and frees its nights at once — in `GET /ari`, on the
property's website and on its other channels. The response is the reservation
as `GET /reservations/{partner_reservation_id}` returns it, with
`status: "cancelled"`, `cancellation_fee_minor` and `cancelled_at`.

- **GlampSys moves no money.** It never refunds the guest and never charges a
  cancellation fee: you collected the payment, you refund the guest.
  `cancellation_fee_minor` is what the property's cancellation policy charges
  for cancelling now, in the reservation's currency — exactly the
  `charge_minor` of the quote's `cancellation_milestones` band that contains
  the moment of cancellation. GlampSys records it as the property's retained
  revenue; you settle it with the property under your agreement.
- **GlampSys does not email the guest about the cancellation** — not even
  under the `full` email policy — and withdraws the arrival and stay emails it
  had planned. Tell the guest yourself.
- Cancelling again (a retry with a new key, or a reservation the property
  already cancelled) returns `200` with the current state and no new fee.
- A stay that is over (`status: "completed"` — the departure day has passed)
  cannot be cancelled: `409 CONFLICT` with
  `error.details.reason = "RESERVATION_COMPLETED"`.
- To change dates or the unit, cancel and confirm a new quote (modification is
  planned).

### Reconciliation

`GET /reservations?updated_since=<ISO 8601>` lists your reservations whose
record changed at or after that instant, oldest change first, in pages of up to
200 (`limit`). Pass `next_cursor` back unchanged as `cursor` until it is
`null`. Remember the largest `updated_at` you processed and start the next run
from it; a reservation can appear twice, so upsert by `reservation_id`.
`GET /reservations/{partner_reservation_id}` returns one reservation with its
current `status` (`confirmed`, `modified`, `cancelled`, `completed`).

### Holds: lifetime and caps

- A hold lasts **`hold_ttl_seconds`** — 1800 seconds (30 minutes) unless
  agreed otherwise for your account (allowed range 300–3600). After
  `expires_at` the quote reads as `expired` and no longer holds the dates;
  the background sweep frees the calendar within about a minute.
- You may have at most **200 active holds per environment** and **3 per
  unit** at once, unless agreed otherwise. Over a cap, `POST /quotes` with
  `hold: true` returns `429 RATE_LIMITED` with
  `error.details.rule = "HOLD_LIMIT"`, `error.details.scope` (`partner` or
  `unit`) and a `Retry-After` until your oldest hold in that scope expires.
  Expired, released and confirmed quotes do not count; `hold: false` never
  counts. Sandbox and production holds are counted separately.

### What a quote contains

- `breakdown` sums to `total_price_minor`: `accommodation` (the nights with
  your markup), `promotion` (an automatic promotion, negative for a
  discount) and `local_tax` (only when your line collects it — then
  `taxes.local_tax.included` is `true` and `collected_by` is `partner`).
- `nightly` shows each night with your markup. It is informational.
- The price follows the group. Where the operator prices by the number and
  ages of guests, the nights include per-guest surcharges for `adults` and
  each child's age on the arrival day — send the real group, a quote for two
  adults is not valid for four.
- `cancellation_milestones` lists every band of the property's cancellation
  policy with the exact instants it applies (`from` / `until`, inclusive, ISO
  8601 UTC) and what cancelling in it would cost in the quote's currency.
- In `EUR` (only on a line with EUR enabled at a fixed rate — otherwise
  `422 BUSINESS_RULE_VIOLATION` with `details.rule = "CURRENCY"`), the rate is
  fixed into `fx_rate` for the life of the quote.

### When a quote is refused

The checks run in a fixed order and the first failing one is returned —
the full table is in the reference under `POST /quotes`. Business rules come
back as `422 BUSINESS_RULE_VIOLATION` with `error.details.rule`: `CURRENCY`,
`CAPACITY`, `BOOKING_WINDOW`, `OUT_OF_SEASON`, `ARRIVAL_DAY`,
`DEPARTURE_DAY` or `MIN_STAY`, plus the rule's limit (for example
`min_nights` or `allowed_weekdays`) so you can tell the guest why.

## Money

All amounts are integers in **minor units** — `125000` is 1 250,00 CZK. There
are no floating-point amounts anywhere in the API. Every amount is accompanied
by an ISO 4217 `currency`.

## Days of the week

Allowed arrival and departure days use ISO 8601: `1` is Monday, `7` is Sunday.
`null` means there is no restriction. An empty list would mean "never", so the
API does not send one.

## Idempotency

Every `POST` that changes state requires an `Idempotency-Key` header (the one
exception is `POST /quotes` with `"hold": false`, which changes nothing).

- Repeating a request with the **same key and the same body** returns the
  stored response, marked with the header `Idempotent-Replayed: true`. It
  never creates a second hold or reservation.
- Reusing a key with a **different body** returns `409 IDEMPOTENCY_KEY_REUSED`.
  If the original request is still running you get the same code with
  `error.details.state = "in_progress"` — wait and retry with the same key.
- A key is 1–255 printable ASCII characters. Keys are separate per environment
  (sandbox and production never share one).
- `5xx` and `429` responses are not stored: retrying with the same key does
  the work again. Everything else is replayed for 48 hours.

If you do not receive a response to `POST /reservations/confirm`, **retry with
the same key**. Do not generate a new key — that is how double bookings happen.

## Errors

Every error has the same envelope:

```json
{
  "error": {
    "code": "QUOTE_EXPIRED",
    "message": "The quote has expired.",
    "request_id": "req_01J…",
    "details": { "quote_id": "q_01J…" }
  }
}
```

Build your integration on `error.code`, not on `message` — the message is for
humans and may change. Log `request_id`; it is also returned in the
`X-Request-ID` header of **every** response, successful ones included, and it
is the fastest way for us to find your request.

Treat unknown fields in responses, and unknown values of enums such as a status,
as valid. We add fields without a version bump.

## Rate limits

Limits are per partner and per class of endpoint, counted per minute:

| Class | Endpoints | Default |
|---|---|---|
| read | all `GET` | 600 / min |
| quote | `POST /quotes` | 300 / min |
| write | `POST /reservations/confirm`, `POST /reservations/{id}/cancel` (and modify when live), `DELETE /quotes` | 600 / min |

The classes are independent. Exhausting `quote` — for example because a bot is
hammering your booking form — does not consume the `write` bucket; write
requests still have their own limit. Over the limit you get
`429 RATE_LIMITED` with a `Retry-After` header; wait that many seconds.
`error.details.class` says which limit it was; a `429` with
`error.details.rule = "HOLD_LIMIT"` is the hold cap described above, not a
request rate.

## Webhooks (planned)

We notify you when a reservation you created is cancelled or modified in
GlampSys, and when availability of a mapped unit changes outside your own
requests. `availability.changed` is a signal to pull `GET /ari` for the affected
range — it does not carry prices.

Every delivery is signed:

```
X-GlampSys-Signature: t=1788950400,v1=5257a869…
```

To verify it:

1. Take `t` and the **raw** request body, exactly as received.
2. Compute `HMAC-SHA256(secret, "<t>.<raw body>")` and hex-encode it.
3. Compare it with `v1` using a constant-time comparison.
4. Reject the delivery if `t` is more than five minutes away from your clock.
   This stops a captured delivery from being replayed later.

Respond with any `2xx` to acknowledge. Other responses are retried after 1 min,
5 min, 30 min, 2 h and 12 h.

## Support

Your integration contact is set up during onboarding. When you report an issue,
include the `request_id`, and for reservations the `partner_reservation_id` and
`quote_id`.
