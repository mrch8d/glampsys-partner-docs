---
name: glampsys-partner-api
description: Build or maintain a booking-portal integration with the GlampSys Partner API v1 — glamping unit catalog, availability, indicative rates and stay restrictions (ARI), and later binding quotes with holds, reservations and webhooks. Use when writing code that calls https://glampsys.cz/api/partner/v1, maps GlampSys units to portal listings, syncs ARI with ETags, or handles errors, rate limits or webhook signatures from GlampSys.
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
| `POST /quotes`, `GET`/`DELETE /quotes/{quote_id}` | planned |
| `POST /reservations/confirm`, `GET /reservations/{partner_reservation_id}` | planned |
| `POST /reservations/{id}/modify`, `POST /reservations/{id}/cancel` | planned |
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
   the local tax are resolved only by `POST /quotes` (planned).
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

## Planned booking flow (design only, not live)

```
POST /quotes   hold: false   "check availability" on the listing page — holds nothing
POST /quotes   hold: true    guest submits the order — dates held until expires_at
POST /reservations/confirm   after your payment gateway confirms the payment
DELETE /quotes/{quote_id}    guest abandoned the payment — release the dates now
```

- Every state-changing `POST` needs an `Idempotency-Key` header, except
  `POST /quotes` with `"hold": false`. If a confirm gets no response, retry
  with **the same key** — a new key is how double bookings happen.
- Show the guest the quote's `total_price` and `breakdown`; the reservation is
  confirmed against that quote.
- Webhooks are signed with `X-GlampSys-Signature: t=<unix>,v1=<hex>`:
  compute `HMAC-SHA256(secret, "<t>.<raw body>")` over the raw body, compare in
  constant time, reject `t` more than five minutes off. `availability.changed`
  only tells you to pull `GET /ari` again; it carries no prices.

The hold duration, the behaviour when a payment arrives after `expires_at`,
and the split of guest e-mails between the portal and GlampSys are agreed
before this stage goes live and will be written into the contract. Do not
assume them.

## Sandbox acceptance checklist

1. `GET /ping` → `200`, `environment: "sandbox"`, the agreed `unit_count`.
2. `GET /units` → the demo units; keep their `unit_id`.
3. `GET /ari` for those units and a future range → per-day rows.
4. The same request with `If-None-Match` → `304` without a body.

When reporting a problem, send the time of the request, the HTTP status,
`error.code` and `X-Request-ID` — never the token and never guest data.
