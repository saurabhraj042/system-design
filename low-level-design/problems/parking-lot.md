# Parking Lot

**Difficulty:** Easy | **Concepts:** Entity identification, relational vs. intrinsic state, fee calculation, integer money, occupancy tracking

---

## Understanding the Problem

Design a parking lot system. Vehicles drive in, get assigned a spot, receive a ticket, and pay a fee based on time when they leave. Spots come in three sizes. The system must track which spots are occupied and calculate fees correctly.

---

## Requirements Clarification

**Core questions:**
- How does size matching work? → Each vehicle type maps to exactly one spot type (motorcycle → MOTORCYCLE, car → CAR, large → LARGE). No size fallback unless asked as an extension.
- What does the system return when a vehicle enters? → A ticket with a spot assignment and entry time.
- How is the fee calculated? → Hourly rate, rounded up to the nearest hour (5 minutes counts as 1 hour).
- What currency precision? → Store as cents (integer). Never floating-point dollars.
- What if the lot is full? → Throw an error. No waiting queue.

**Final Requirements:**
```
1. Vehicle enters by specifying type (MOTORCYCLE/CAR/LARGE)
   - System finds available spot of matching size
   - Returns ticket; error if no spot available
2. Vehicle exits by presenting ticket ID
   - System computes fee (round up to nearest hour)
   - Frees the spot; returns fee in cents
3. Concurrent entries from multiple entrances must be handled safely
   (extensibility question — mention it; don't implement unless asked)

Out of scope: reservations, payment processing, display boards, queuing
```

---

## Core Entities

Candidate nouns: vehicle, driver, spot, ticket, parking lot, entrance, fee.

**Vehicle** — the system only needs one property from it: what type it is. It doesn't maintain state inside our system. Not an entity — demote to `VehicleType` enum.

**Entrance** — hardware concern. Not an entity.

**Fee** — a calculated value, not something with identity or behavior. Not an entity — just the return value of `computeFee`.

**ParkingSpot** — a physical space with a fixed size and an availability state. **Entity.**

**Ticket** — a record of entry: which spot, which vehicle type, when. Has identity (ticket ID is how you claim your spot). **Entity.**

**ParkingLot** — the orchestrator. Manages all spots, tracks active tickets. **Entity.**

| Entity | Responsibility |
|--------|---------------|
| **ParkingLot** | Finds available spots, manages occupancy, computes fees, handles enter/exit lifecycle |
| **ParkingSpot** | Pure data holder: ID + spot type. No state management. |
| **Ticket** | Records entry: ticket ID, spot ID, vehicle type, entry time |

---

## Class Design

### ParkingLot

| Requirement | What ParkingLot must track |
|-------------|---------------------------|
| "Find available spot of matching size" | All spots (iterable), plus which are occupied |
| "Compute fee on exit" | The ticket (entry time + spot info) |
| "Prevent double-exit" | Active tickets; remove on exit |

```
class ParkingLot:
    - spots: List<ParkingSpot>
    - occupiedSpotIds: Set<String>       // O(1) availability check
    - activeTickets: Map<String, Ticket> // O(1) ticket lookup on exit
    - hourlyRateCents: long

    + ParkingLot(spots, hourlyRateCents)
    + enter(vehicleType) → Ticket
    + exit(ticketId) → long    // returns fee in cents
```

**Why `Map<String, Ticket>` not `List<Ticket>`?** Exit looks up tickets by ID. A Map gives O(1) lookup; a List would require O(n) scan.

**Why `Set<String>` for occupied spots?** During spot search, we check `spot.id not in occupiedSpotIds` for each candidate. O(1) per check via Set membership vs. O(n) scan.

**Why is `hourlyRateCents` a `long` not `double`?** Floating-point can't represent 0.1 exactly in binary — $0.10 becomes 0.1000000000000000055...  After multiplication, rounding errors accumulate. Store money as integer cents. $5.00/hour = 500 cents/hour.

### ParkingSpot

```
class ParkingSpot:
    - id: String
    - spotType: SpotType

    + ParkingSpot(id, spotType)
    + getSpotType() → SpotType
    + getId() → String
```

ParkingSpot is a pure data holder. No occupancy tracking, no state management. Why not an `occupied` flag on ParkingSpot? Because occupancy is *relational* state — it depends on whether a ticket has been issued for this spot, not whether the spot itself changed physically. The ParkingLot tracks the relationship, so it owns the occupancy Set.

**Why two separate enums `SpotType` and `VehicleType` with the same values?**
They're semantically distinct concepts. A motorcycle *is* a vehicle; a MOTORCYCLE spot *is* a space category. A large vehicle can't fit in a car spot, but a car could hypothetically fit in a large spot. Keeping them separate allows you to add `mapVehicleTypeToSpotType()` logic later (e.g., fallback rules) without collapsing unrelated concepts. If they shared an enum and you ever needed to diverge the logic, you'd have to split them then.

### Ticket

```
class Ticket:
    - id: String
    - spotId: String
    - vehicleType: VehicleType
    - entryTime: long           // Unix milliseconds

    + Ticket(id, spotId, vehicleType, entryTime)
    + getId() → String
    + getSpotId() → String
    + getVehicleType() → VehicleType
    + getEntryTime() → long
```

Ticket is a pure data holder. No behavior beyond getters. Entry time is in milliseconds (multiplying `time()` by 1000) for integer arithmetic on duration.

### Enums

```
enum SpotType:   MOTORCYCLE, CAR, LARGE
enum VehicleType: MOTORCYCLE, CAR, LARGE
```

---

## Implementation

### `enter`

**Happy path:**
1. Find available spot for this vehicle type
2. Mark spot as occupied
3. Generate ticket with entry timestamp
4. Store ticket for later exit lookup
5. Return ticket

**Edge cases:**
- No available spot of required type → throw error
- Invalid vehicle type → no matching spot type → same error

```
enter(vehicleType):
    spot = findAvailableSpot(vehicleType)
    if spot == null:
        throw Error("No available spots for vehicle type " + vehicleType)

    occupiedSpotIds.add(spot.getId())

    ticketId = generateUUID()
    entryTime = currentTimeMillis()
    ticket = Ticket(ticketId, spot.getId(), vehicleType, entryTime)
    activeTickets[ticketId] = ticket

    return ticket
```

### `findAvailableSpot`

```
findAvailableSpot(vehicleType):
    requiredSpotType = mapVehicleTypeToSpotType(vehicleType)
    for spot in spots:
        if spot.getSpotType() == requiredSpotType && spot.getId() not in occupiedSpotIds:
            return spot
    return null

mapVehicleTypeToSpotType(vehicleType):
    if vehicleType == MOTORCYCLE: return MOTORCYCLE
    if vehicleType == CAR: return CAR
    if vehicleType == LARGE: return LARGE
    throw Error("Unknown vehicle type")
```

Keep it simple: first-match linear scan. Don't add "prefer spots near the entrance" logic unless the requirement mentions it — that's a YAGNI violation.

### `exit`

**Happy path:**
1. Validate ticket ID (not null, not empty, exists in map)
2. Calculate fee from entry time to now
3. Free the spot (remove from occupiedSpotIds)
4. Remove ticket from activeTickets (prevents double-exit)
5. Return fee

**Edge cases:**
- ticketId is null or empty
- Ticket doesn't exist or was already used

```
exit(ticketId):
    if ticketId == null || ticketId.isEmpty():
        throw Error("Invalid ticket ID")

    ticket = activeTickets[ticketId]
    if ticket == null:
        throw Error("Ticket not found or already used")

    exitTime = currentTimeMillis()
    fee = computeFee(ticket.getEntryTime(), exitTime)

    occupiedSpotIds.remove(ticket.getSpotId())
    activeTickets.remove(ticketId)

    return fee
```

Both "ticket never existed" and "ticket already used" (removed from map on exit) return the same error. We don't distinguish them because once used, a ticket is gone from `activeTickets`. If you wanted to distinguish, add a `usedTickets: Set<String>` — but that's unnecessary complexity for interview scope.

### `computeFee`

```
computeFee(entryTime, exitTime):
    durationMillis = exitTime - entryTime
    durationHours = durationMillis / (1000 * 60 * 60)    // integer division → floor
    if durationMillis % (1000 * 60 * 60) > 0:
        durationHours++                                    // round up to nearest hour

    return durationHours * hourlyRateCents
```

Rounding up: 5 minutes counts as 1 hour. 2.5 hours counts as 3 hours. The modulo check handles this cleanly — no separate "minimum charge" logic needed.

---

## Verification Trace

ParkingLot has spots: A (MOTORCYCLE), B (CAR), C (LARGE). `occupiedSpotIds={}`, `activeTickets={}`. Hourly rate = 500 cents ($5).

**Vehicle enters:**
```
enter(CAR):
  findAvailableSpot(CAR) → requiredType = CAR
    spot A: type=MOTORCYCLE → skip
    spot B: type=CAR, B not in occupiedSpotIds → return B
  occupiedSpotIds.add("B") → {"B"}
  ticket = Ticket("T123", "B", CAR, 1000000)
  activeTickets["T123"] = ticket
  return ticket

State: occupiedSpotIds={"B"}, activeTickets={"T123" → ticket}
```

**Vehicle exits 2.5 hours later:**
```
exit("T123"):
  ticket = activeTickets["T123"] → found
  exitTime = 10000000  (1000000 + 2.5 hours in ms)
  computeFee(1000000, 10000000):
    durationMillis = 9000000
    durationHours = 9000000 / 3600000 = 2 (integer)
    9000000 % 3600000 = 1800000 > 0 → durationHours = 3
    fee = 3 * 500 = 1500 cents
  occupiedSpotIds.remove("B") → {}
  activeTickets.remove("T123") → {}
  return 1500
```

**Double-exit attempt:**
```
exit("T123"):
  activeTickets["T123"] → null (already removed)
  throw Error("Ticket not found or already used") ✓
```

**Lot full attempt:**
```
enter(CAR):
  findAvailableSpot(CAR): all CAR spots in occupiedSpotIds → return null
  throw Error("No available spots for vehicle type CAR") ✓
  State unchanged ✓
```

---

## Extensibility

### "How would you extend this to a multi-floor parking garage?"

Add a `ParkingFloor` entity between `ParkingLot` and `ParkingSpot`. Each floor owns a set of spots; the lot owns floors. Spot IDs encode floor identity: "3-A15" = floor 3, section A, spot 15.

```
class ParkingLot:
    - floors: List<ParkingFloor>
    - activeTickets: Map<String, Ticket>
    - hourlyRateCents: long

class ParkingFloor:
    - floorNumber: int
    - spots: List<ParkingSpot>
    + getAvailableSpotCount(spotType) → int
    + findAvailableSpot(spotType) → ParkingSpot?
```

**Three allocation strategies** (consider using Strategy pattern if needed at runtime):
1. **Fill lower floors first** — simple floor-by-floor iteration. Keeps vehicles closer to entrance/exit.
2. **Balance across floors** — track available count per floor, prefer the floor with most availability to avoid congestion.
3. **Proximity to destination** — mall garages might know which floor connects to which store. User says "food court" → assign floor 4.

The Ticket class doesn't need to change — it stores spot ID, and the floor info is implicit in the spot ID string.

### "How would you add different pricing for different vehicle types?"

Current: one rate for everyone. New: motorcycles $3, cars $5, large $8.

Simple approach — change `hourlyRateCents` from a single value to a map:
```
class ParkingLot:
    - hourlyRates: Map<VehicleType, long>

computeFee(entryTime, exitTime, vehicleType):
    durationHours = calculateDuration(entryTime, exitTime)
    rate = hourlyRates[vehicleType]    // look up per vehicle type
    return durationHours * rate
```

Complex approach (if rules become intricate like surge pricing or discounts): introduce a `PricingStrategy` interface. For three flat rates, the map is simpler — don't over-engineer.

### "How would you handle multiple entrances with concurrent access?"

**The race condition:** Two vehicles enter simultaneously from different entrances. Both call `findAvailableSpot`, both find spot B23 is free (both read before either writes), both mark it occupied, both get tickets for spot B23. Double booking!

The window between checking if a spot is available and adding it to `occupiedSpotIds` is where the race happens — a classic check-then-act problem.

**Simple fix — coarse-grained lock on `enter()`:**
```
enter(vehicleType):
    synchronized(this):
        // entire method body — now atomic
        spot = findAvailableSpot(vehicleType)
        ...
```

Serializes all entrance requests. Simple, correct. For a parking lot with 3–5 entrances and typical traffic, this is the right choice — performance isn't an issue, correctness is.

**Better fix — fine-grained (if truly high concurrency):**
Use atomic check-and-add on `occupiedSpotIds` with retry logic. Only lock at the moment of claiming the spot. Threads can search in parallel; only the claim is serialized.

Async note: for Node.js or other async systems, even single-threaded event loops can have race conditions at each `await` point. In that case, handle at the database layer using `SELECT FOR UPDATE` or distributed locks (Redis). This is System Design territory, not LLD.

---

## What Interviewers Look For

| Level | Bar |
|-------|-----|
| Junior | Spots, tickets, and an orchestrator. `enter()` finds a spot and returns a ticket. `exit()` computes a fee and frees the spot. Basic error handling: reject when full, reject invalid tickets. |
| Mid-level | Vehicle is not a class (just a type enum). Clean separation: ParkingLot orchestrates, ParkingSpot is a data holder, Ticket is immutable. Justify: why activeTickets is a Map, why occupiedSpotIds is a Set, why pricing logic lives in ParkingLot not Ticket. Handle at least one extension (multi-floor or per-type pricing). |
| Senior | Proactively discuss the occupied flag vs. relational Set tradeoff. Two separate enums (SpotType / VehicleType) explained. Integer cents. Recognize race condition with concurrent entrances, present simple lock solution first then mention fine-grained approach. |

---

## Practice Questions

1. Why does `occupied` state live in `occupiedSpotIds` on ParkingLot rather than as a boolean flag on ParkingSpot? What would need to change if you moved it to ParkingSpot?
2. The `computeFee` method uses integer division then rounds up with a modulo check. Why not just compute `Math.ceil(durationMillis / 3600000.0) * hourlyRateCents`? What could go wrong with floating-point?
3. After a successful `exit()`, the ticket is removed from `activeTickets`. A second `exit("T123")` gets "Ticket not found or already used." Should you distinguish these two cases? What would you need to add, and is it worth it?
4. Trace what happens when two threads simultaneously call `enter(CAR)` on a lot with exactly one CAR spot remaining and no synchronization. Show the exact race window and which invariant it violates.
5. Design `getAvailableCount(vehicleType) → int` for the multi-floor extension. Should this be on ParkingLot or ParkingFloor? What data structure change on ParkingFloor makes this O(1) instead of O(spots)?

## One-Line Summary

Parking Lot's key insight is that Vehicle is not an entity (just a type parameter), occupancy is relational state living on the ParkingLot orchestrator as a Set (not a boolean flag on ParkingSpot), fees use integer cents to avoid floating-point money bugs, and the activeTickets Map enables both O(1) exit lookup and automatic double-exit prevention.
