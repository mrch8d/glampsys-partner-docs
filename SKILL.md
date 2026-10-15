---
name: glampsys-partner-api
description: Build or maintain a booking-portal integration with the GlampSys Partner API v1 — glamping unit catalog, availability, indicative rates and stay restrictions (ARI), binding quotes with holds, confirmed reservations paid to the portal, their cancellation and reconciliation, and later modifications and webhooks. Use when writing code that calls https://glampsys.cz/api/partner/v1, maps GlampSys units to portal listings, syncs ARI with ETags, or handles errors, rate limits or webhook signatures from GlampSys.
---

# GlampSys Partner API v1 — integration skill

This file is written for an AI coding assistant working on the partner side of
the integration. It tells you what is live, how to call it safely, and which
mistakes break the integration. It does not replace the contract.

## Sources of truth

Fetch these instead of guessing field names or behaviour:

| What | URL |
|---|---|
| Contract (OpenAPI 3.1) | https://mrch8d.github.io/glampsys-partner-docs/partner-v1.openapi.yaml |
| Guide | https://mrch8d.github.io/glampsys-partner-docs/getting-started.md |
| Changelog | https://mrch8d.github.io/glampsys-partner-docs/CHANGELOG.md |
| Rendered reference | https://mrch8d.github.io/glampsys-partner-docs/ |

If this file and the contract disagree, **the contract wins**. Generate request
and response types from the contract rather than writing them by hand, and
check `info.version` and the changelog when something looks different.

Base URL: `https://glampsys.cz/api/partner/v1`

## What is live

| Operation | Status |
|---|---|
| `GET /ping` | live |
| `GET /units` | live |
| `GET /ari` | live |
| `POST /quotes`, `GET`/`DELETE /quotes/{quote_id}` | live since 1.2.0 |
| `POST /reservations/confirm`, `GET /reservations/{partner_reservation_id}`, `GET /reservations` | live since 1.3.0 |
| `POST /reservations/{partner_reservation_id}/cancel` | live since 1.4.0 |
| `POST /reservations/{id}/modify` | planned |
| webhooks `reservation.cancelled`, `reservation.modified`, `availability.changed` | planned |

Planned operations carry `x-status: planned` in the contract. Calling one today
returns `404 NOT_FOUND` with `error.details.reason = "UNKNOWN_ENDPOINT"`. You may
build against their published shapes, but keep that code behind a feature flag
and do not ship a flow that depends on them until the changelog marks them live.

## Ground rules

1. **The token is a secret.** Read it from an environment variable or a secret
   store (for example `GLAMPSYS_PARTNER_TOKEN`). Never hardcode it, log it,
   commit it, put it in a shared or exported Postman collection, a test
   fixture, an error report or a prompt. `gsp_test_…` is sandbox,
   `gsp_live_…` is production; keep them in separate configurations and make
   test code refuse to run with a `gsp_live_` token.
2. **Money is integer minor units plus a currency.** `{"amount_minor": 320000,
   "currency": "CZK"}` is 3 200 Kč. Never convert to floating point; format
   only at the display edge.
3. **ARI is indicative, never binding.** Never compute a payable total from
   ARI and never charge a guest from it. Promotions, minimum-stay rules and
   the local tax are resolved only by `POST /quotes`.
4. **`price: null` means no price.** The night is not sellable, or the
   requested currency is not enabled (`price_unavailable_reason:
   "CURRENCY_NOT_ENABLED"`). Never substitute an estimate or a conversion.
5. **Dates are local calendar dates** in `YYYY-MM-DD`. Do not push them
   through timezone conversions. A stay from `check_in` to `check_out`
   occupies the nights `check_in` … `check_out − 1`. In `GET /ari` both
   `from` and `to` are inclusive.
6. **Identifiers are opaque.** Store `unit_id`; do not parse it. Your own
   listing id is returned as `partner_unit_id`, and every endpoint that takes
   a unit accepts either form. One portal listing is one GlampSys unit. A unit
   that is not mapped to you returns `404`, never `403`.
7. **Be tolerant of additions.** Unknown fields and unknown enum values are
   valid; GlampSys adds them without a version bump. Do not use strict schemas
   that reject extra keys.
8. **Branch on `error.code`, never on `error.message`.** Log `request_id`; it
   is also in the `X-Request-ID` header of every response.

## Recommended sync design

**Catalog.** Call `GET /units` once a day and on demand, with `If-None-Match`
set to the last `ETag`. `304` means nothing changed. A unit that disappears
from the list has been paused or unmapped: hide the listing, do not delete its
history. Conditions of stay come from here — check-in and check-out times,
`default_min_stay`, `booking_window_months`, season, allowed arrival and
departure weekdays (ISO 8601, `1` = Monday, `null` = no restriction),
cancellation rules, terms links and `local_tax`.

**ARI.** Every 1–3 hours, request each unit from today to the end of its
booking window. Limits per call: at most 366 days and 100 units — split larger
requests. Repeat `unit_id` for several units (`?unit_id=a&unit_id=b`);
`unit_ids` is not a parameter. Keep the `ETag` per distinct request and send it
back; `304` means no change. Each day carries `available`, `available_count`,
`min_stay`, `arrival_allowed`, `departure_allowed` and `price`.

**Filtering search results from cached ARI** (display only — the quote is the
final authority): every night from `check_in` to `check_out − 1` is
`available`; `arrival_allowed` is true on `check_in`; `departure_allowed` is
true on `check_out`; the number of nights is at least `min_stay` of the
`check_in` day; the stay lies inside the booking window and the season from
`GET /units`.

**Errors and retries.** `429` comes with `Retry-After` in seconds — wait that
long. Retry `5xx` and network failures with exponential backoff and jitter. Do
not retry other `4xx`, except `408`. Default limits per partner and minute:
read 600, quote 300, write 600; the classes are independent.

```ts
const BASE = "https://glampsys.cz/api/partner/v1";

export class PartnerApiError extends Error {
  constructor(
    readonly status: number,
    readonly code: string | undefined,
    readonly requestId: string | null,
    readonly retryAfterSeconds?: number,
  ) {
    super(`GlampSys ${status} ${code ?? ""} (request ${requestId ?? "?"})`);
  }
}

export async function fetchAri(
  unitIds: string[],
  from: string, // YYYY-MM-DD, inclusive
  to: string, // YYYY-MM-DD, inclusive, at most 366 days
  etag?: string,
) {
  const query = new URLSearchParams({ from, to, currency: "CZK" });
  for (const id of unitIds) query.append("unit_id", id); // at most 100
  const res = await fetch(`${BASE}/ari?${query}`, {
    headers: {
      Authorization: `Bearer ${process.env.GLAMPSYS_PARTNER_TOKEN}`,
      ...(etag ? { "If-None-Match": etag } : {}),
    },
  });
  const requestId = res.headers.get("X-Request-ID");
  if (res.status === 304) return { changed: false as const };
  if (!res.ok) {
    const body = await res.json().catch(() => null);
    const retryAfter = res.headers.get("Retry-After");
    throw new PartnerApiError(
      res.status,
      body?.error?.code,
      requestId,
      retryAfter ? Number(retryAfter) : undefined,
    );
  }
  return { changed: true as const, etag: res.headers.get("ETag"), data: await res.json() };
}
```

## Booking flow

```
POST   /quotes   hold: false   "check availability" on the listing page — holds nothing   (live)
POST   /quotes   hold: true    guest submits the order — dates held until expires_at       (live)
DELETE /quotes/{quote_id}      guest abandoned the payment — release the dates now         (live)
POST   /reservations/confirm   after your payment gateway confirms the payment             (live)
GET    /reservations?updated_since=…   reconciliation of what changed on GlampSys's side      (live)
```

**Quotes (live since 1.2.0).**

- `hold: false` returns `200`, `status: "priced"`, `quote_id: null`. It holds
  nothing, needs no `Idempotency-Key` and must not carry `guest` (`400`). Use
  it for every "check availability / show price" interaction.
- `hold: true` returns `201`, `status: "held"`, a `quote_id`, `expires_at` and
  `hold_ttl_seconds` (1800 s unless agreed otherwise). It requires an
  `Idempotency-Key`; generate one per checkout attempt and reuse it on retries.
  A competing sale of the same unit — including a guest on the operator's own
  website — gets `409 UNAVAILABLE`; exactly one of two racing requests wins.
- The quote is the price you charge: show the guest `total_price_minor` and
  `breakdown` (they always add up), never a total rebuilt from ARI.
- Send the real group — `adults` and the age of every child on the arrival
  day. Where the operator prices by guests, the group changes the price, so
  re-quote whenever the guest changes the number or ages of guests. ARI
  prices are for the unit's included occupancy and are not an upper bound.
- Business-rule refusals are `422 BUSINESS_RULE_VIOLATION` with
  `error.details.rule` (`CAPACITY`, `BOOKING_WINDOW`, `OUT_OF_SEASON`,
  `ARRIVAL_DAY`, `DEPARTURE_DAY`, `MIN_STAY`, `CURRENCY`) and the limit — map
  them to a message for the guest.
- Active holds are capped (200 per environment, 3 per unit, unless agreed
  otherwise). Over a cap you get `429 RATE_LIMITED` with
  `error.details.rule = "HOLD_LIMIT"` — that is not a request-rate problem, so
  do not just retry in a loop; release holds you no longer need with
  `DELETE /quotes/{quote_id}`.
- After `expires_at` the quote reads as `expired` and no longer holds the
  dates.

**Confirm (live since 1.3.0).**

- Call `POST /reservations/confirm` only after your payment gateway confirmed
  the payment in full. Send the `quote_id`, your own `partner_reservation_id`
  (unique in your account; letters, digits, `.`, `_`, `:`, `-`), the `guest`
  (with `language`), the `consents` you collected (`terms` and `gdpr` must be
  `true`) and the `payment` (`status: "paid"`, `total_price_minor`,
  `currency`, `paid_at`). Store the returned `reservation_id`.
- The reservation gets the **quote's** price. `payment` must be in the quote's
  currency and within your agreed tolerance of its `total_price_minor` —
  otherwise `409 PRICE_MISMATCH`, nothing is created. Charge the guest exactly
  the quoted total and you never hit it.
- If a confirm gets no response, retry with **the same `Idempotency-Key`** —
  a new key is how double bookings happen. A second confirm of the same quote
  with another key is `409 QUOTE_ALREADY_USED`.
- **Payment finished after `expires_at`?** Confirm anyway, within 24 hours of
  `expires_at`. GlampSys re-checks that the stay can still be sold; if it can,
  you get the reservation at the quoted price. If not, `409 UNAVAILABLE` with
  `error.details.reason = "HOLD_EXPIRED"` — nothing was created, refund the
  guest. `409 UNAVAILABLE` with `reason: "SALES_PAUSED"` means the owner
  paused sales over the dates — also refund. After 24 hours, or for a quote
  you released, `410 QUOTE_EXPIRED`.
- Every `409`/`410` on confirm means nothing was created and GlampSys
  charged nothing. Decide refund vs. retry by `error.code`, never by text.
- Guest e-mails: GlampSys writes to the guest only after confirmation and
  only what the email policy agreed for the location allows — possibly a
  confirmation without payment details, stay e-mails, or nothing. It never
  sends payment requests, reminders or invoices for your reservations. Do not
  assume which policy applies; if you send your own confirmation, include
  `guest_portal_url` (the guest's online check-in).
- Reconcile with `GET /reservations?updated_since=<last updated_at you
  processed>`; follow `next_cursor` until it is `null`, upsert by
  `reservation_id` (the feed is at-least-once). `GET
  /reservations/{partner_reservation_id}` gives one reservation's current
  `status` — `confirmed`, `modified`, `cancelled` (with
  `cancellation_fee_minor`) or `completed`.

**Cancel (live since 1.4.0).**

- `POST /reservations/{partner_reservation_id}/cancel` with an
  `Idempotency-Key`; the body `{ "reason": "…" }` is optional (≤ 500
  characters, for GlampSys's records only). The response is the
  `Reservation` with `status: "cancelled"`, `cancellation_fee_minor` and
  `cancelled_at`; the nights are free again in `GET /ari` right away.
- **GlampSys moves no money**: it never refunds the guest and never charges a
  cancellation fee. You refund the guest from your payment.
  `cancellation_fee_minor` (in the reservation's currency) is what the
  property's policy charges for cancelling now — the `charge_minor` of the
  quote's `cancellation_milestones` band containing that moment; you settle
  it with the property under your agreement.
- **GlampSys sends the guest no cancellation e-mail** (under any e-mail
  policy) and withdraws planned arrival and stay e-mails — tell the guest
  yourself.
- Retrying is safe: the same key replays the response; cancelling an already
  cancelled reservation with a new key is `200` with no new fee.
  `409 CONFLICT` with `error.details.reason = "RESERVATION_COMPLETED"` means
  the stay is over. Changing dates = cancel + a new quote and confirm until
  modify is live.

**Modify, webhooks (planned).**

- Webhooks are signed with `X-GlampSys-Signature: t=<unix>,v1=<hex>`:
  compute `HMAC-SHA256(secret, "<t>.<raw body>")` over the raw body, compare in
  constant time, reject `t` more than five minutes off. `availability.changed`
  only tells you to pull `GET /ari` again; it carries no prices.

## Sandbox acceptance checklist

1. `GET /ping` → `200`, `environment: "sandbox"`, the agreed `unit_count`.
2. `GET /units` → the demo units; keep their `unit_id`.
3. `GET /ari` for those units and a future range → per-day rows.
4. The same request with `If-None-Match` → `304` without a body.
5. `POST /quotes` with `hold: false` for a sellable stay → `200`,
   `status: "priced"`, and `breakdown` adds up to `total_price_minor`.
6. `POST /quotes` with `hold: true` and an `Idempotency-Key` → `201`,
   `status: "held"`; `GET /ari` for those nights now shows one unit fewer.
7. The same request with the same key → the same response with
   `Idempotent-Replayed: true`; a second hold of the last free unit → `409`.
8. `DELETE /quotes/{quote_id}` → `status: "cancelled"`; the nights are free
   again in `GET /ari`.
9. A new held quote → `POST /reservations/confirm` with `payment` equal to
   its `total_price_minor` → `201`, `reservation_id`; the same request with
   the same key → the same body with `Idempotent-Replayed: true`.
10. `GET /reservations/{partner_reservation_id}` → `status: "confirmed"`;
    `GET /reservations?updated_since=…` lists it.
11. `POST /reservations/{partner_reservation_id}/cancel` with a new key →
    `200`, `status: "cancelled"`, `cancellation_fee_minor` = the matching
    milestone's `charge_minor`; the nights are free again in `GET /ari`; the
    same call with another key → `200`, same body, no new fee.

When reporting a problem, send the time of the request, the HTTP status,
`error.code` and `X-Request-ID` — never the token and never guest data.
