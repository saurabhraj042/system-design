# Movie Ticket Booking

**Difficulty:** Medium | **Concepts:** Orchestrator pattern, derived state, per-resource locking, atomic multi-seat booking, extensibility with holds

---

## Understanding the Problem

Design a movie ticket booking system. Users search for movies, view available showtimes, select seats, and make reservations. Reservations can be cancelled. The system must prevent double-booking, even when multiple users try to book the same seat simultaneously.

---

## Requirements Clarification

**Core questions:**
- Can a user book multiple seats at once? → Yes, and it must be all-or-nothing (atomic).
- What happens if one seat in a multi-seat booking is taken? → Fail the entire booking; book no seats.
- How do we find available seats? → Derive from the list of existing reservations (no separate seat state).
- How are seats identified? → Row letter + number, e.g., "A3", "B15".
- What does cancellation need? → A confirmation ID that routes back to the showtime.

**Final Requirements:**
```
1. Search movies by title (partial match)
2. View available seats for a showtime
3. Book one or more seats atomically (all or nothing)
4. Cancel a reservation by confirmation ID
5. Concurrent bookings from multiple users must be safe
```

---

## Core Entities

Candidate nouns: movie, theater, screen, showtime, seat, reservation, ticket, booking system.

**Seat** — a physical location in a theater. But availability is not intrinsic to the seat itself; it depends on whether a reservation covers it. Not an entity — just a string ID like "A3".

**Screen** — a physical room. Identified by a label within a theater. Not an entity.

**Movie** — has a title and ID. **Entity.**

**Theater** — has an ID and a set of seats. **Entity.**

**Showtime** — a specific movie screening in a specific theater at a specific time. Owns the reservations and therefore knows which seats are available. **Entity.**

**Reservation** — records which seats are booked for which showtime. Has a confirmation ID. Back-references its Showtime for O(1) cancellation routing. **Entity.**

**BookingSystem** — the orchestrator. Routes requests to the right showtime. **Entity.**

| Entity | Responsibility |
|--------|---------------|
| **BookingSystem** | Routes book/cancel/search requests; owns top-level indexes |
| **Showtime** | Owns reservations; derives seat availability; serializes concurrent bookings |
| **Reservation** | Immutable record of which seats are booked + back-reference to showtime |
| **Movie** | Title + ID |
| **Theater** | ID + seat layout |

---

## Class Design

### BookingSystem

```
class BookingSystem:
    - theaters: List<Theater>
    - moviesById: Map<String, Movie>
    - showtimesById: Map<String, Showtime>             // O(1) routing for book/cancel
    - showtimesByMovieId: Map<String, List<Showtime>>  // O(1) routing for search
    - reservationsById: Map<String, Reservation>       // O(1) cancellation routing

    + BookingSystem(theaters, movies, showtimes)
    + searchMovies(title) → List<Showtime>
    + getAvailableSeats(showtimeId) → List<String>
    + book(showtimeId, seatIds) → Reservation
    + cancelReservation(confirmationId) → void
```

**Why `reservationsById` on BookingSystem?** Cancellation arrives with only a confirmation ID. Without this map, finding the reservation would require scanning every showtime's reservation list — O(showtimes × reservations). With the map it's O(1).

**Why `showtimesByMovieId`?** Search takes a movie title, which maps to a movie ID, which maps to a list of showtimes. Without it, search would scan all showtimes and join with movies. The map trades a small amount of memory for O(1) lookups.

### Showtime

```
class Showtime:
    - id: String
    - theater: Theater
    - movie: Movie
    - datetime: DateTime
    - screenLabel: String
    - reservations: List<Reservation>
    - holds: Map<String, SeatHold>        // extensibility: temporary holds

    + book(reservation) → void            // synchronized on this
    + cancel(reservation) → void          // synchronized on this
    + isAvailable(seatId) → boolean
    + getAvailableSeats() → List<String>
    + holdSeats(seatIds, timeoutMs) → String    // extensibility
    + confirmHold(holdId, reservation) → void   // extensibility
```

**Why synchronize on the Showtime, not globally?** Two users booking different showtimes should proceed in parallel. Locking on `this` (the Showtime) gives per-showtime isolation. A global lock would serialize all bookings system-wide — a traffic jam at one movie blocking every other movie.

**Why derive availability from `reservations` rather than tracking a per-seat state?**
Seat availability is *relational* — it depends on the association between a seat and a reservation, not on some intrinsic property of the seat. Storing an `available` flag on a seat would require keeping two sources of truth in sync. Using `reservations` as the single source of truth makes the system self-consistent: if a reservation exists, the seat is taken; if it's cancelled, the seat is free automatically.

### Reservation

```
class Reservation:
    - confirmationId: String
    - showtime: Showtime       // back-reference for O(1) cancellation routing
    - seatIds: List<String>    // immutable copy

    + Reservation(confirmationId, showtime, seatIds)
    + getConfirmationId() → String
    + getShowtime() → Showtime
    + getSeatIds() → List<String>
```

**Why does Reservation hold a back-reference to Showtime?** Cancellation is called with just a `confirmationId`. `BookingSystem.reservationsById` gives the Reservation in O(1). From there, `reservation.getShowtime()` gives the Showtime in O(1), so we can call `showtime.cancel(reservation)` without any search. Without this back-reference, we'd need `reservationsById` to map to both the reservation *and* the showtimeId separately.

### SeatHold (Extensibility)

```
class SeatHold:
    - holdId: String
    - seatIds: List<String>
    - expiresAt: long          // Unix ms

    + SeatHold(holdId, seatIds, expiresAt)
```

Holds represent a temporary reservation during the payment flow. A user selects seats → a hold is created (seats appear taken for 5 minutes) → user pays → hold is confirmed into a real reservation. If payment fails or times out, the hold expires and seats become available again.

---

## Implementation

### `book`

**The atomic challenge:** For multi-seat bookings, we must check *all* seats first, then reserve *all* of them. Any check-then-reserve split creates a window where another thread could grab a seat between our check and our add.

**Solution:** Do everything inside `synchronized(showtime)`:

```
// BookingSystem.book(showtimeId, seatIds):
if showtimeId == null || seatIds == null || seatIds.isEmpty():
    throw IllegalArgumentException("Invalid booking request")

showtime = showtimesById[showtimeId]
if showtime == null:
    throw ShowtimeNotFoundException(showtimeId)

confirmationId = generateUUID()
reservation = Reservation(confirmationId, showtime, copy(seatIds))

showtime.book(reservation)   // throws SeatUnavailableException on conflict

reservationsById[confirmationId] = reservation
return reservation

// Showtime.book(reservation):
synchronized(this):
    seatIds = reservation.getSeatIds()
    for seatId in seatIds:
        if !isValidSeatId(seatId):
            throw InvalidSeatException(seatId)
    for seatId in seatIds:
        if !isAvailable(seatId):
            throw SeatUnavailableException(seatId)
    reservations.add(reservation)
```

**Two-pass validation:** First validate all seat IDs (format check), then check all seats for availability, then add. Never add partially — if seat 3 of 4 is taken, we throw before adding anything.

**Why create the Reservation before calling `showtime.book()`?** The Reservation holds the back-reference to the Showtime. We build it first, pass it to `showtime.book()` which validates and registers it atomically, then register it in `reservationsById`. If `showtime.book()` throws, we never register it in `reservationsById` and the reservation is effectively discarded.

### `isAvailable` and `getAvailableSeats`

```
// Showtime.isAvailable(seatId):
for reservation in reservations:
    if reservation.getSeatIds().contains(seatId):
        return false
return true

// Showtime.getAvailableSeats():
booked = Set<String>()
for reservation in reservations:
    for seatId in reservation.getSeatIds():
        booked.add(seatId)
available = []
for row in 'A' to 'Z':
    for num in 0 to 20:
        seatId = row + num
        if !booked.contains(seatId):
            available.add(seatId)
return available
```

### `cancelReservation`

```
// BookingSystem.cancelReservation(confirmationId):
reservation = reservationsById[confirmationId]
if reservation == null:
    throw ReservationNotFoundException(confirmationId)

showtime = reservation.getShowtime()
showtime.cancel(reservation)
reservationsById.remove(confirmationId)

// Showtime.cancel(reservation):
synchronized(this):
    reservations.remove(reservation)
```

### `searchMovies`

```
// BookingSystem.searchMovies(title):
results = []
for movieId, showtimes in showtimesByMovieId:
    movie = moviesById[movieId]
    if movie.getTitle().toLowerCase().contains(title.toLowerCase()):
        results.addAll(showtimes)
return results
```

### Extensibility: Seat Holds

```
// Showtime.holdSeats(seatIds, timeoutMs):
synchronized(this):
    for seatId in seatIds:
        if !isAvailable(seatId):
            throw SeatUnavailableException(seatId)
    holdId = generateUUID()
    hold = SeatHold(holdId, copy(seatIds), currentTimeMillis() + timeoutMs)
    holds[holdId] = hold
    return holdId

// Showtime.confirmHold(holdId, reservation):
synchronized(this):
    hold = holds[holdId]
    if hold == null:
        throw HoldNotFoundException(holdId)
    if currentTimeMillis() > hold.expiresAt:
        holds.remove(holdId)
        throw HoldExpiredException(holdId)
    holds.remove(holdId)
    reservations.add(reservation)
```

The State pattern is a natural fit here: each seat progresses through AVAILABLE → HELD → BOOKED states, with transitions governed by the Showtime. For a simple implementation without holds, the two-state version (available/booked via the reservations list) is sufficient.

The Observer pattern is useful for showtime cancellations: when a showtime is cancelled, all reservation holders need to be notified. Add a `List<ShowtimeCancellationListener>` to Showtime and call `listener.onShowtimeCancelled(showtime)` for each when the showtime is cancelled.

---

## Verification Trace

Initial state: Showtime S1, theater has seats A1–A3. `reservations=[]`.

**User 1 books A1 and A2:**
```
book("S1", ["A1", "A2"]):
  showtime = showtimesById["S1"]
  reservation = Reservation("R1", showtime, ["A1", "A2"])
  showtime.book(reservation):
    synchronized(showtime):
      isAvailable("A1") → reservations empty → true
      isAvailable("A2") → reservations empty → true
      reservations.add(reservation) → [{R1: A1, A2}]
  reservationsById["R1"] = reservation
  return reservation ✓
```

**User 2 tries to book A2 and A3 simultaneously:**
```
showtime.book(Reservation("R2", showtime, ["A2", "A3"])):
  synchronized(showtime):           // blocks until User 1's lock releases
    isAvailable("A2"):
      reservation R1 contains "A2" → return false
    throw SeatUnavailableException("A2")  // no seats added ✓
```

**User 1 cancels:**
```
cancelReservation("R1"):
  reservation = reservationsById["R1"] → R1
  showtime = reservation.getShowtime() → S1
  showtime.cancel(R1):
    synchronized(showtime): reservations.remove(R1) → []
  reservationsById.remove("R1")
  
// Now A2 is available again
```

---

## Extensibility

### "How would you add recurring showtimes?"

Add a `RecurringSchedule` entity with a recurrence rule (daily, every Friday, etc.) and a `ShowtimeGenerator` that creates Showtime instances from the schedule. The `BookingSystem` would call the generator periodically or on demand. The Showtime class doesn't change.

### "How would you handle seat pricing (premium seats, discounts)?"

Add a `PricingStrategy` interface:
```
interface PricingStrategy:
    calculatePrice(seat, showtime, promo) → long   // cents

class StandardPricing implements PricingStrategy:
    calculatePrice(seat, showtime, promo):
        base = seat.getRow() <= 'E' ? 1500 : 1200  // front rows cost more
        if promo != null: return promo.apply(base)
        return base
```

Inject the strategy into `BookingSystem`. The Reservation stores the price at booking time (prices can change later; the reservation is a point-in-time contract).

---

## What Interviewers Look For

| Level | Bar |
|-------|-----|
| Junior | Theater, Movie, Showtime, Reservation as entities. `book()` checks availability then adds. Returns error if seat taken. |
| Mid-level | Derive availability from reservations (not seat flags). Two-pass validation (check all, then add all). `reservationsById` for O(1) cancel routing. Reservation back-reference to Showtime. Synchronize on Showtime (not globally). |
| Senior | Justify per-showtime lock vs. global. Explain the all-or-nothing atomic window. Mention holds/State pattern for payment flows. Observer for showtime cancellations. Discuss read/write lock upgrade if reads dominate. |

---

## Practice Questions

1. Why validate all seat IDs for format before checking any for availability? What happens if you mix the two passes?
2. The `reservationsById` map lives on `BookingSystem`, but the `reservations` list lives on `Showtime`. Why not put `reservationsById` on `Showtime` too?
3. Two users simultaneously call `book("S1", ["A5"])`. Both enter `showtime.book()` near-simultaneously. One gets the lock first. Trace what the second user sees when it gets the lock — what state has changed and why does it throw?
4. Why store an immutable copy of `seatIds` in the Reservation instead of the original list passed by the caller?
5. The `getAvailableSeats()` method iterates all possible seat IDs (A0–Z20) and checks against the booked set. Why build a booked `Set` first instead of calling `isAvailable(seatId)` for each?

## One-Line Summary

Movie Ticket Booking's core insight is that availability is *derived state* (computed from the reservations list, not stored on seats), booking must be atomic (check all seats then add all, inside a per-showtime synchronized block), and the Reservation holds a back-reference to its Showtime for O(1) cancellation routing without scanning.
