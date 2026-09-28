# Design Justification — Vacation Rental Platform

## Entities and responsibilities

- **User** — a person who can list properties (as a host) and/or make
  bookings (as a guest). One class instead of separate `Host`/`Guest`
  classes, because on a real platform the same person plays both roles.
- **Property** — a listing owned by a host. Knows how to check its own
  availability for a date range.
- **Booking** — a reservation of one property by one guest for a date
  range. Owns its own lifecycle: confirming, cancelling, and triggering
  a review.
- **Payment** — the payment tied to one booking.
- **Review** — feedback tied to one specific booking/stay.

## Key design decisions

**`Property.is_available(start_date, end_date)` instead of a boolean flag.**
A book in a lending system is either loaned or not — one flag is enough.
A rental property is different: many guests can book the same property
for different, non-overlapping date ranges at the same time. A single
`is_available` boolean would be wrong the moment two guests want
different weeks. So availability is computed from existing bookings for
that date range, not stored as a flag.

**`Booking` owns `confirm_booking()`, `cancel_booking()`, and
`leave_review()`.** These are all actions whose meaning only exists in
the context of one specific reservation, so the Booking is the object
responsible for them, not the User or the Property.

## Relationships and multiplicities

- `User "1" -- "0..*" Property` — a host can list zero or many
  properties; each property belongs to exactly one host.
- `User "1" -- "0..*" Booking` — a guest can make zero or many
  bookings; each booking belongs to exactly one guest.
- `Property "1" -- "0..*" Booking` — a property can be booked many
  times over its lifetime; each booking targets exactly one property.
- `Booking "1" *-- "0..1" Payment` — a payment only makes sense inside
  the context of its booking (composition); a booking may not have a
  payment yet (e.g. pending).
- `Booking "1" *-- "0..1" Review` — a review is intrinsically tied to
  the stay it came from (composition); not every booking gets reviewed.

## Alternatives considered

- **Separate `Host` and `Guest` classes** inheriting from `User`. Rejected:
  it would force every user to pick one role, which doesn't reflect how
  the same person can be both, and it adds an inheritance hierarchy the
  behavior doesn't actually need.
- **A `Platform`/`BookingService` class mediating everything.** Rejected
  as unnecessary — `User.create_booking()` is a clear enough entry point
  without adding a class whose only job is delegating to others.
- **Boolean `is_available` flag on `Property`.** Rejected — see above;
  it breaks as soon as a property has more than one booking ever.

## Trade-offs

- Because `is_available()` recomputes from bookings instead of reading a
  flag, it costs a bit more at read time, but it can never get out of
  sync with reality — no risk of a stale flag if a cancellation isn't
  handled correctly somewhere else.
- Payment and Review are modeled as compositions of Booking, which keeps
  them simple, but means a cancelled/deleted booking would need explicit
  handling for what happens to its payment/review records (e.g. keep the
  payment for refund history even if the booking is cancelled, not
  deleted).

## What we'd simplify or extend if requirements changed

- If properties needed multiple photos/amenities, that's a new
  `Amenity`/`Photo` class associated with `Property` — doesn't touch the
  booking/payment/review flow at all.
- If hosts needed approval workflows (accept/reject a booking request
  before it's confirmed), `Booking.status` already supports that as an
  extra transition, without changing the class structure.
