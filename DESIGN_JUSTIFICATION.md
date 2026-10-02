# Design Justification  Vacation Rental Platform

## Entities and responsibilities

- **User** : a person who can list properties (as a host) and/or make
  bookings (as a guest). One class instead of separate `Host`/`Guest`
  classes, because on a real platform the same person plays both roles.
- **Property** : a listing owned by a host. Knows how to check its own
  availability for a date range.
- **Booking** : a reservation of one property by one guest for a date
  range. Owns its own lifecycle: confirming, cancelling, and triggering
  a review. Also knows whether its dates overlap a given range
  (`overlaps`), which `Property.is_available` uses.
- **Review** : feedback tied to one specific booking/stay.

## Key design decisions

**`Property.is_available(start_date, end_date)` instead of a boolean flag.**
A rental property can be booked by many guests for different,
non-overlapping date ranges. A single boolean would be wrong the moment
two guests want different weeks. So availability is computed from the
property's existing bookings (asking each `Booking.overlaps(...)`), not
stored as a flag.

**`Booking` owns `confirm_booking()`, `cancel_booking()`, and
`leave_review()`.** These actions only make sense in the context of one
specific reservation, so Booking is responsible for them, not the User
or the Property.

**Payments are deliberately not modeled.** The scenario says to avoid
modeling payments and external services, so there is no `Payment` class.
Cancelling a booking only changes its status; money handling is out of
scope.

**Rejecting a booking uses `cancel_booking()`.** Both a guest cancelling
and a host rejecting move the booking to the same final state. We did not
add `reject_booking()` to avoid a duplicate method. The trade-off: we
cannot tell later who ended the booking.

**Booking lifecycle uses a `status` field.** A new booking starts as
`pending`. The host then confirms it (`confirmed`) or rejects it
(`cancelled`). A guest can also cancel a booking.


## Relationships and multiplicities

- `User "1" -- "0..*" Property` : a host can list zero or many
  properties; each property belongs to exactly one host.
- `User "1" -- "0..*" Booking` : a guest can make zero or many
  bookings; each booking belongs to exactly one guest.
- `Property "1" -- "0..*" Booking` : a property can be booked many
  times over its lifetime; each booking targets exactly one property.
- `Booking "1" *-- "0..1" Review` : a review is tied to the stay it came
  from (composition); not every booking gets reviewed.

## Alternatives considered

- **Separate `Host` and `Guest` classes** inheriting from `User`.
  Rejected: it forces every user to pick one role and adds an
  inheritance hierarchy the behavior doesn't need.
- **A `Platform`/`BookingService` class mediating everything.** Rejected
  as unnecessary — `User.create_booking()` is a clear entry point
  without a class whose only job is delegating.
- **Boolean `is_available` flag on `Property`.** Rejected — it breaks as
  soon as a property has more than one booking.

## Trade-offs

- `is_available()` recomputes from bookings instead of reading a flag,
  so it costs a bit more at read time, but it can never go stale.
- Leaving out Payment keeps the model focused on the scenario, but the
  platform cannot represent refunds or prices paid; a cancelled booking
  is simply marked `cancelled` (not deleted) so history is kept.
- Review as a composition of Booking keeps the model simple: a review
  cannot exist without its stay.
- `Booking.overlaps()` ignores cancelled bookings, so dates of a rejected
  or cancelled booking become available again.

## What we'd simplify or extend if requirements changed

- Photos/amenities would be a new `Amenity`/`Photo` class associated
  with `Property`, without touching the booking/review flow.
