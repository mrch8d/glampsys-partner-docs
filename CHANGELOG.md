# Changelog — GlampSys Partner API

All notable changes to the Partner API contract are listed here. The contract
follows [Semantic Versioning](https://semver.org/): a new **major** version
removes or renames a field or changes what an endpoint does, a **minor** version
adds something, a **patch** fixes documentation or examples.

We add fields to responses without a version bump. Treat unknown fields and
unknown enum values as valid.

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
