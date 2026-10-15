# Changelog — GlampSys Partner API

All notable changes to the Partner API contract are listed here. The contract
follows [Semantic Versioning](https://semver.org/): a new **major** version
removes or renames a field or changes what an endpoint does, a **minor** version
adds something, a **patch** fixes documentation or examples.

We add fields to responses without a version bump. Treat unknown fields and
unknown enum values as valid.

## 1.4.0 — 2026-10-14

### Live

- `POST /reservations/{partner_reservation_id}/cancel` is deployed. It
  cancels a reservation you confirmed and releases its nights at once — in
  `GET /ari`, on the property's own website and on every other channel.
  `Idempotency-Key` is required; the body (`{ "reason": "…" }`) is optional.
  - **The money stays with you.** GlampSys never refunds the guest and never
    charges or collects a cancellation fee. The response's
    `cancellation_fee_minor` is what the property's cancellation policy
    charges for cancelling now, in the reservation's currency, computed
    exactly like the quote's `cancellation_milestones` — cancelling inside a
    milestone band costs that band's `charge_minor`. GlampSys records it as
    the property's retained revenue; settle it with the property under your
    agreement.
  - **The guest gets no cancellation email from GlampSys**, whatever email
    policy the location has with you; pending pre-arrival and stay emails
    are withdrawn. Inform the guest yourself.
  - Idempotent: the same key and body replays the original response;
    cancelling an already cancelled reservation again, with any key,
    returns `200` with its current state and no new fee.
  - A `completed` reservation (departure day has passed) cannot be
    cancelled: `409 CONFLICT` with `details.reason = RESERVATION_COMPLETED`.
    Another partner's reservation, or one from the other environment, is
    `404 NOT_FOUND`. A paused location link or an unmapped unit does not
    block cancelling.
- `GET /reservations/{partner_reservation_id}` (and the
  `GET /reservations` feed) report a cancellation made through the API with
  exactly the `cancellation_fee_minor` the cancel call returned.

### Changed

This operation was `planned` until this release; nothing was live under the
old shape.

- `POST /reservations/{partner_reservation_id}/cancel` returns the full
  `Reservation` (the same schema as `GET /reservations/{partner_reservation_id}`)
  instead of `ReservationCancellationResult`, which is removed together with
  its `cancelled_revenue_received_minor` (it always equalled
  `cancellation_fee_minor`).
- An already cancelled reservation is `200`, not `409`; `409` on cancel
  documents its codes (`CancelConflict`): `CONFLICT`
  (`RESERVATION_COMPLETED`) and `IDEMPOTENCY_KEY_REUSED`.
- `CancelReservationBody.reason` is limited to 500 characters.

## 1.3.0 — 2026-10-14

### Live

- `POST /reservations/confirm` is deployed. It turns a held quote into a
  reservation the guest has already paid you for. GlampSys records it as paid
  outside GlampSys: no charge, no payment request or reminder, no automatic
  invoice. `Idempotency-Key` is required.
  - The price is the quote's price — GlampSys does not re-price at
    confirmation. `payment` (what the guest paid you) must be in the quote's
    currency and within your account's tolerance of `total_price_minor`
    (zero unless agreed otherwise), otherwise `409 PRICE_MISMATCH` with both
    values in `details`.
  - **Late payment.** A quote whose hold has expired can still be confirmed
    for **24 hours after `expires_at`**: GlampSys re-checks, under the same
    lock every other sale takes, that the unit is still yours and active, the
    location is not paused, the nights are in season, arrival is not in the
    past and the dates are free. Free → the reservation is created at the
    quoted price. Not free → `409 UNAVAILABLE` with
    `details.reason = "HOLD_EXPIRED"`; refund the guest. After the 24 hours,
    or for a quote you released with `DELETE /quotes/{quote_id}`,
    `410 QUOTE_EXPIRED` with `details.status` (`expired` / `cancelled`).
  - A sales pause the owner sets during your hold covers the hold; confirming
    inside the pause is `409 UNAVAILABLE` with
    `details.reason = "SALES_PAUSED"`.
  - `partner_reservation_id` is your id for the reservation, unique within
    your account (`409 CONFLICT`, `details.reason =
    DUPLICATE_PARTNER_RESERVATION_ID`, when reused).
  - Guest emails after confirmation follow the email policy agreed per
    location: a confirmation without payment details (naming you and your
    `partner_reservation_id`, linking the online check-in portal) and stay
    emails, only stay emails, or nothing. GlampSys never sends a guest of a
    partner reservation a payment request, reminder or invoice.
    `guest_portal_url` in the response is the same portal link.
- `GET /reservations/{partner_reservation_id}` is deployed. Status
  (`confirmed`, `modified`, `cancelled`, `completed`), dates, guests, the
  confirmed price and, once cancelled, `cancellation_fee_minor` and
  `cancelled_at`.

### Added

- `GET /reservations?updated_since=&cursor=&limit=` — a reconciliation feed
  of your reservations changed at or after `updated_since`, oldest change
  first, paged with an opaque `next_cursor` (`limit` 1–200, default 50).
- `ConfirmReservationBody.partner_reservation_id`, `.payment`
  (`ReservationPayment`) and `.notes`; `guest.nationality` (optional,
  `ReservationGuest`).
- `Reservation.partner_reservation_reference`, `.quote_id`,
  `.partner_unit_id`, `.nights`, `.cancellation_fee_minor`, `.cancelled_at`.
- `409` on confirm documents every code it can carry
  (`ConfirmConflict`): `QUOTE_ALREADY_USED`, `CONFLICT`, `PRICE_MISMATCH`,
  `UNAVAILABLE`, `IDEMPOTENCY_KEY_REUSED`.

### Changed

These operations were `planned` until this release; nothing was live under
the old shape.

- `ConfirmReservationBody`: `quote_id`, `partner_reservation_id`, `payment`
  are required. `partner_reservation_reference` is no longer the reservation
  id — it stays an optional free-text reference.
- `POST /reservations/confirm` never re-prices the stay (the 1.2.0
  description said it would compare against a fresh price).
- `Consents.accepted_at`, `Reservation.created_at`, `.updated_at` are plain
  strings and `Reservation.check_in` / `.check_out` `YYYY-MM-DD` patterns
  instead of `format: date-time` / `format: date` — the same declaration
  change the quote endpoints got in 1.2.0.
- `GET /quotes/{quote_id}` description: an expired quote can still be
  confirmed for 24 hours (see above).

## 1.2.0 — 2026-10-14

### Live

- `POST /quotes` is deployed. It prices a stay exactly like the public booking
  engine — same rates including dynamic pricing, same automatic promotions,
  same local-tax rule — with your line's markup, and with `hold: true` holds
  the dates. A hold is serialized against every other sale of the unit,
  including the operator's own website, so of two requests for the last free
  unit exactly one gets it and the other `409 UNAVAILABLE`.
  - `hold: true` → `201`, `status: "held"`, `quote_id`, `expires_at`,
    `hold_ttl_seconds` (1800 s unless agreed otherwise, 300–3600).
    `Idempotency-Key` required.
  - `hold: false` → `200`, `status: "priced"`, `quote_id: null`; nothing is
    held, no `Idempotency-Key` needed, the hold cap is not consumed.
  - Checks run in a fixed, documented order; business rules return
    `422 BUSINESS_RULE_VIOLATION` with `error.details.rule` (`CURRENCY`,
    `CAPACITY`, `BOOKING_WINDOW`, `OUT_OF_SEASON`, `ARRIVAL_DAY`,
    `DEPARTURE_DAY`, `MIN_STAY`) and that rule's limit.
  - Active holds are capped per environment (200) and per unit (3) unless
    agreed otherwise. Over a cap: `429 RATE_LIMITED` with
    `error.details.rule = "HOLD_LIMIT"`, `error.details.scope`
    (`partner` / `unit`) and `Retry-After`.
  - The price follows the group you send. Where the operator prices by the
    number and ages of guests, the nights include per-guest surcharges for
    `adults` and each child's age on the arrival day — the same as the public
    booking engine for that group. Elsewhere the group affects only capacity
    and the local tax.
- `GET /quotes/{quote_id}` is deployed. A hold past its `expires_at` reads as
  `expired` immediately, even before the background sweep releases it.
- `DELETE /quotes/{quote_id}` is deployed. Releases the hold at once;
  idempotent.

### Added

- `Quote.partner_unit_id`, `Quote.hold_ttl_seconds` and
  `Quote.taxes.local_tax.collected_by` (`partner` / `property_on_site`).
- `CancellationMilestone.from` and `CancellationMilestone.until` — the exact
  instants (ISO 8601 UTC, both inclusive) each band applies. They are computed
  the way the cancellation fee itself is computed: from midnight **UTC** of
  the check-in day, in whole 24-hour days.
- Response header `Idempotent-Replayed: true` on a replayed response.
- Documented the idempotency details that already applied: a key is 1–255
  printable ASCII characters (pattern on `IdempotencyKeyValue`); keys are
  scoped per environment; `409 IDEMPOTENCY_KEY_REUSED` carries
  `details.state = "in_progress"` while the original request is still being
  processed; a replayed error body carries the original `request_id`; `5xx`
  and `429` are never stored.
- `409 UNAVAILABLE` with `details.reason = "PRICE_NOT_CONFIGURED"` when the
  operator has not priced some nights of the stay.
- `partner_reservation_reference` is limited to 255 characters.

### Changed

- `GET /ari` description: the indicative `price` is for the unit's included
  occupancy and excludes promotions. Version 1.1.0 said a quote comes out at
  or below the sum of the ARI nights; with per-guest surcharges (and
  short-stay surcharges) it can also come out above. The values on the wire
  do not change.

These operations were `planned` until this release; nothing was live under
the old shape.

- `DELETE /quotes/{quote_id}` returns the quote in its current state (the same
  `Quote` body as `GET`), not a separate `QuoteCancellationResult`, which is
  removed. Releasing an already expired quote returns `status: "expired"`.
- Hold caps return `429 RATE_LIMITED` (`details.rule = HOLD_LIMIT`), not
  `422 BUSINESS_RULE_VIOLATION`.
- `currency: "EUR"` on a line without EUR at a fixed rate is refused with
  `422 BUSINESS_RULE_VIOLATION` (`details.rule = CURRENCY`) instead of falling
  back to CZK.
- `guest` is optional with `hold: true` and must be absent or `null` with
  `hold: false` (`400 INVALID_REQUEST`). It is never echoed back.
- `CancellationMilestone.days_before_checkin` can be `null` — for the 100 %
  band after the last configured rule, and for the single 0 % band of a
  property without rules.
- `check_in`, `check_out` and `nightly[].date` are declared as `YYYY-MM-DD`
  patterns, and `expires_at` / `created_at` as plain strings, instead of
  `format: date` / `format: date-time` — the same declaration change `GET /ari`
  got in 1.1.0. The values on the wire do not change.

## 1.1.0 — 2026-09-18

### Live

- `GET /ari` is deployed. Availability, an indicative nightly rate and the
  stay restrictions, per unit and per day.

  `available` and `available_count` are the same numbers the GlampSys booking
  engine shows a guest on the web that day — the endpoint runs the same code,
  so a day you can sell here is a day the operator can sell.

  `price` is **indicative**. It is one night starting that day, from the rate
  plan (including dynamic pricing), with your line's markup applied, rounded
  to whole CZK. It deliberately excludes promotions, which depend on the
  length and timing of the real stay: `POST /quotes` is the binding price and
  will come out at or below the sum of the nights you see here.

  The booking window is **not** applied: a day past the operator's booking
  window can still read as available, exactly as it does on the web. `GET
  /units` gives you `booking_window_months` per unit; `POST /quotes` enforces
  it.

### Added

- `GET /ari` takes an optional `currency` query parameter (`CZK` default,
  `EUR`). `EUR` is only honoured on a line that has it enabled with a fixed
  exchange rate; anywhere else every day comes back with a null `price` and
  `price_unavailable_reason: CURRENCY_NOT_ENABLED` instead of being silently
  priced in CZK. Amounts are always minor units — haléře for CZK, cents for
  EUR, never rounded to whole euros.

### Changed

- `AriDay.date` is declared as a `YYYY-MM-DD` pattern instead of
  `format: date`. The value on the wire is unchanged — it was and is a plain
  `"2026-10-01"` string. Only the declaration changed, so that the generated
  validators accept what the endpoint actually sends. `GET /ari` was
  `planned` until this release, so nothing was built against the old
  declaration.

## 1.0.1 — 2026-09-10

### Fixed

- The server URL is `https://glampsys.cz/api/partner/v1`. Version 1.0.0 listed
  `https://api.glampsys.cz/api/partner/v1`, a host that does not exist. Nothing
  else changed: paths, request and response shapes are identical.

## 1.0.0 — 2026-09-10

First published contract.

### Live

- `GET /ping` — checks the token and returns the number of units you can reach.
- `GET /units` — mapped units with their conditions of stay: check-in and
  check-out times, minimum stay, booking window, season, allowed arrival and
  departure days (ISO 8601), cancellation rules, terms and cancellation policy
  links, and the local tax.

### Planned

Published now so you can build against the final shape; each is marked
`x-status: planned` in the contract until it is deployed.

- `GET /ari` — availability, indicative rates and restrictions per unit and day.
- `POST /quotes` — binding price of a stay, with or without holding the dates.
- `GET /quotes/{quote_id}`, `DELETE /quotes/{quote_id}` — read and release a hold.
- `POST /reservations/confirm` — turn a held quote into a reservation.
- `GET /reservations/{partner_reservation_id}` — current state of a reservation.
- `POST /reservations/{partner_reservation_id}/modify` — change dates, unit or
  guests using a new quote.
- `POST /reservations/{partner_reservation_id}/cancel` — cancel, with the
  cancellation fee calculated from the property's rules.
- Webhooks `reservation.cancelled`, `reservation.modified` and
  `availability.changed`, signed with HMAC-SHA256.
