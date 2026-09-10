# Changelog — GlampSys Partner API

All notable changes to the Partner API contract are listed here. The contract
follows [Semantic Versioning](https://semver.org/): a new **major** version
removes or renames a field or changes what an endpoint does, a **minor** version
adds something, a **patch** fixes documentation or examples.

We add fields to responses without a version bump. Treat unknown fields and
unknown enum values as valid.

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
