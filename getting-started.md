# Getting started with the GlampSys Partner API

This API lets a booking portal pull availability and rates for glamping units,
get a binding price for a specific stay, hold the dates while the guest pays,
and confirm the reservation once payment has cleared.

The full reference is generated from the OpenAPI 3.1 contract and is the source
of truth. This page explains how the pieces fit together.

## Implemented read-only scope (M0/M1)

Implementation status below is not a claim that a particular partner account
or credential has been provisioned. Validate your issued sandbox token before
starting development. The reservation flow below describes planned operations,
not a permission to send orders or payments.

| Endpoint | Status |
|---|---|
| `GET /ping` | live |
| `GET /units` | live |
| `GET /ari` | live — availability, indicative rates and daily restrictions |
| quotes, holds, reservations and webhooks | planned — in the contract, marked `x-status: planned` |

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

## The complete booking flow (writes planned)

```
GET  /units                       once a day — conditions of stay
GET  /ari?unit_id=…&from=…&to=…    every 1–3 h — availability, rates, restrictions
POST /quotes      hold: false     "check availability" on the listing page
POST /quotes      hold: true      guest submits the order — dates are held
POST /reservations/confirm        after your payment gateway confirms payment
```

1. **Pull conditions and availability.** `GET /units` returns check-in times,
   minimum stay, allowed arrival days, cancellation rules and the local tax.
   `GET /ari` returns one row per unit and day. Send `If-None-Match` with the
   `ETag` you received; an unchanged response comes back as `304` with no body.
2. **Never rebuild the final price from ARI.** ARI prices are indicative.
   The binding price of a stay — promotions, minimum-stay rules, local tax —
   always comes from `POST /quotes`.
3. **Check without holding.** `POST /quotes` with `"hold": false` returns the
   binding price and real-time availability without blocking the dates. Use it
   on the listing page; clicks and bots do not consume inventory.
4. **Hold while the guest pays.** `POST /quotes` with `"hold": true` holds the
   dates and returns a `quote_id` with `expires_at`. Another request for the
   same dates gets `409 UNAVAILABLE` until the hold expires or you release it
   with `DELETE /quotes/{quote_id}`.
5. **Confirm after payment.** `POST /reservations/confirm` with the `quote_id`
   turns the hold into a reservation atomically. A quote can be confirmed once.

## Money

All amounts are integers in **minor units** — `125000` is 1 250,00 CZK. There
are no floating-point amounts anywhere in the API. Every amount is accompanied
by an ISO 4217 `currency`.

## Days of the week

Allowed arrival and departure days use ISO 8601: `1` is Monday, `7` is Sunday.
`null` means there is no restriction. An empty list would mean "never", so the
API does not send one.

## Idempotency (planned write operations)

Every `POST` that changes state requires an `Idempotency-Key` header (the one
exception is `POST /quotes` with `"hold": false`, which changes nothing).

- Repeating a request with the **same key and the same body** returns the
  stored response. It never creates a second reservation.
- Reusing a key with a **different body** returns `409 IDEMPOTENCY_KEY_REUSED`.

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
| write | confirm, modify, cancel, `DELETE /quotes` | 600 / min |

The classes are independent. Exhausting `quote` — for example because a bot is
hammering your booking form — does not consume the `write` bucket; write
requests still have their own limit. Over the limit you get
`429 RATE_LIMITED` with a `Retry-After` header; wait that many seconds.

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
