# Elevator

**Difficulty:** Medium | **Concepts:** Simulation vs. hardware, state machine, direction-aware scheduling, SCAN algorithm, separate enum types

---

## Understanding the Problem

Design an elevator control system. Multiple elevators serve a multi-floor building. People press hall call buttons (up/down) to request an elevator, then press destination buttons inside the cab. The system must decide which elevator to dispatch and how each elevator should move.

---

## Requirements Clarification

**Core questions:**
- Simulation or real hardware? → Simulation: expose a `step()` method that advances the system one tick. No real motor controllers or timer callbacks.
- How many elevators, how many floors? → 3 elevators, 10 floors (0–9)
- Who handles destination floor requests? → Passengers press inside the cab; that's a second call to `requestElevator` with `type=DESTINATION`
- What if no elevator is available? → Return false
- Do we need to handle door open/close timing? → No, assume instantaneous

**Clarification that matters: simulation vs. hardware**

Real elevators run callbacks from motor controllers and floor sensors. For an LLD interview, you almost always want the simulation model: a `step()` method that the caller invokes repeatedly. This makes the system predictable and testable without real hardware dependencies.

**Final Requirements:**
```
1. Hall call: user presses UP or DOWN button on a floor (PICKUP_UP, PICKUP_DOWN)
2. Destination: user presses a floor button inside cab (DESTINATION type)
3. System dispatches the best available elevator for hall calls
4. Controller.step() advances all elevators one tick
5. Each elevator moves one floor per tick, stopping at floors with matching requests
6. Only hall calls of DESTINATION type are rejected at the controller level
   (destinations are added directly to an elevator already serving that rider)
```

---

## Core Entities

Candidate nouns: building, floor, elevator, button, passenger, controller, request.

**Floor** — no state beyond a number. Not an entity, just an integer.

**Passenger** — we never track individual people. Not an entity.

**Button** — hardware detail. Not an entity.

**Request** — a floor number paired with a type. It has identity (two requests to the same floor for the same direction are duplicates). **Entity.**

**Elevator** — has position, direction, and a set of pending requests. Encapsulates its own movement logic. **Entity.**

**ElevatorController** — coordinates multiple elevators, dispatches hall calls. **Entity.**

| Entity | Responsibility |
|--------|---------------|
| **ElevatorController** | Receives hall calls, selects best elevator, delegates `step()` to each elevator |
| **Elevator** | Tracks position/direction, accepts requests, moves one floor per tick |
| **Request** | Floor + type pair, used as a value in Elevator's request set |

---

## Class Design

### ElevatorController

```
class ElevatorController:
    - elevators: List<Elevator>

    + ElevatorController()              // creates 3 Elevator instances
    + requestElevator(floor, type) → boolean
    + step() → void
```

**`requestElevator`** — only accepts hall calls (PICKUP_UP, PICKUP_DOWN). DESTINATION requests are invalid here because they apply to a specific elevator the passenger is already inside.

**`step`** — delegates to each elevator. The controller doesn't know how elevators move; that's encapsulated in `Elevator.step()`.

```
step():
    for e in elevators:
        e.step()
```

### Elevator

```
class Elevator:
    - currentFloor: int          // starts at 0
    - direction: Direction       // UP, DOWN, or IDLE
    - requests: Set<Request>     // deduplication via Set

    + Elevator()
    + addRequest(request) → boolean
    + step() → void
    + getCurrentFloor() → int
    + getDirection() → Direction
```

**Why `Set<Request>` not `List`?** Multiple hall calls to the same floor in the same direction are identical — pressing the UP button on floor 3 twice doesn't create two stops. The Set deduplicates automatically (Request must implement equals/hashCode comparing floor + type).

### Request

```
class Request:
    - floor: int
    - type: RequestType

    + Request(floor, type)
    + getFloor() → int
    + getType() → RequestType
    // must override equals() and hashCode() for Set deduplication
```

### Enums

```
enum Direction:
    UP, DOWN, IDLE

enum RequestType:
    PICKUP_UP       // hall call: passenger going up
    PICKUP_DOWN     // hall call: passenger going down
    DESTINATION     // in-cab button: floor number pressed
```

**Why separate `RequestType` from `Direction`?** Hall calls carry directional intent (PICKUP_UP, PICKUP_DOWN) which the elevator uses to decide whether to stop. If you stored only a floor number, the elevator couldn't tell whether to stop going UP or only on the way DOWN. DESTINATION requests apply in both directions — the rider is already aboard.

**Why does `IDLE` live in `Direction` but not `RequestType`?** IDLE is an elevator state, not a type of request. A request asking an elevator to go idle makes no sense.

---

## Implementation

### `requestElevator`

```
requestElevator(floor, type):
    if floor < 0 || floor > 9:
        return false
    if type == DESTINATION:
        return false    // hall-call only at the controller level

    request = Request(floor, type)
    best = selectBestElevator(request)
    return best.addRequest(request)
```

### Elevator Selection: Three Tiers

**Bad — nearest only:**
Always pick the elevator with the smallest distance to the requested floor. Ignores direction: sends an elevator going to floor 9 back to floor 3 to pick someone up, abandoning its current passengers.

**Good — direction-aware basic:**
Prefer elevators already moving in the same direction that haven't passed the floor yet. Fall back to idle elevators. Fall back to any elevator going the other direction.

**Great — SCAN (the recommended approach):**
Elevators sweep in one direction until no more requests lie ahead, then reverse. Like a record needle sweeping across vinyl — efficient, no starvation. Score each elevator by how many ticks it would take to reach the requested floor given its current direction and requests.

```
selectBestElevator(request):
    // Priority: same-direction in-path > idle > opposite direction
    best = null
    bestScore = MAX_VALUE

    for elevator in elevators:
        score = computeScore(elevator, request)
        if score < bestScore:
            best = elevator
            bestScore = score

    return best
```

Mentioning Strategy pattern here is appropriate: `selectBestElevator` defines an interface; you can swap in "minimize wait time," "minimize energy," or "load balance" by plugging in a different implementation.

### `Elevator.step()` — SCAN Algorithm

Four edge cases must be handled in order:

**Case 1 — Nothing to do:**
```
if requests.isEmpty():
    direction = IDLE
    return
```
Without this, an idle elevator with no requests falls through and starts drifting.

**Case 2 — IDLE with requests, pick a direction:**
```
if direction == IDLE:
    // Find nearest request; ties broken by lower floor number
    nearest = null
    minDistance = MAX_VALUE
    for req in requests:
        distance = abs(req.getFloor() - currentFloor)
        if distance < minDistance || (distance == minDistance && req.getFloor() < nearest.getFloor()):
            minDistance = distance
            nearest = req
    direction = (nearest.getFloor() > currentFloor) ? UP : DOWN
```

**Do not** use `requests.iterator().next()` — a HashSet has non-deterministic iteration order. Always use a deterministic tiebreaker.

**Case 3 — Check if we should stop at current floor:**
```
pickupType = (direction == UP) ? PICKUP_UP : PICKUP_DOWN
pickupRequest = Request(currentFloor, pickupType)
destinationRequest = Request(currentFloor, DESTINATION)

if requests.contains(pickupRequest) || requests.contains(destinationRequest):
    requests.remove(pickupRequest)
    requests.remove(destinationRequest)
    // Note: PICKUP_DOWN at this floor is NOT removed if we're going UP
    // It survives and will be serviced on the return trip going DOWN
    if requests.isEmpty():
        direction = IDLE
    return  // stopped this tick — don't move
```

**Stopping is direction-aware:** only stop for PICKUP_UP and DESTINATION when going UP. A PICKUP_DOWN at floor 6 while going UP is ignored — the elevator will serve it on the return trip. This is intentional real-world behavior.

**Case 4 — Reverse if no requests ahead:**
```
if !hasRequestsAhead(direction):
    direction = (direction == UP) ? DOWN : UP
    return  // don't move this tick; next tick checks for a stop at current floor in new direction
```

**Case 5 — Move:**
```
if direction == UP:
    currentFloor++
else if direction == DOWN:
    currentFloor--
```

**Full `step()` puts it together:**
```
step():
    // Case 1
    if requests.isEmpty():
        direction = IDLE
        return

    // Case 2
    if direction == IDLE:
        nearest = null; minDistance = MAX_VALUE
        for req in requests:
            distance = abs(req.getFloor() - currentFloor)
            if distance < minDistance || (distance == minDistance && req.getFloor() < nearest.getFloor()):
                minDistance = distance; nearest = req
        direction = (nearest.getFloor() > currentFloor) ? UP : DOWN

    // Case 3
    pickupType = (direction == UP) ? PICKUP_UP : PICKUP_DOWN
    if requests.contains(Request(currentFloor, pickupType)) || requests.contains(Request(currentFloor, DESTINATION)):
        requests.remove(Request(currentFloor, pickupType))
        requests.remove(Request(currentFloor, DESTINATION))
        if requests.isEmpty(): direction = IDLE
        return

    // Case 4
    if !hasRequestsAhead(direction):
        direction = (direction == UP) ? DOWN : UP
        return

    // Case 5
    if direction == UP: currentFloor++
    else: currentFloor--
```

### `hasRequestsAhead`

```
hasRequestsAhead(dir):
    for request in requests:
        if dir == UP && request.getFloor() > currentFloor:
            return true
        if dir == DOWN && request.getFloor() < currentFloor:
            return true
    return false
```

**Checks ANY request regardless of type.** The elevator must *travel* toward all requests, but only *stop* for matching ones. An elevator going UP will travel toward PICKUP_DOWN on floor 6 (to eventually reach its destination at floor 8), but won't stop there until reversing.

Boundary handling falls out naturally: on floor 9 going UP, `hasRequestsAhead(UP)` returns false (no requests above 9), so Case 4 reverses to DOWN. No explicit `if currentFloor == 9` needed. Adding that check would create a bug: if someone presses floor 9 while we're at floor 9 going up, we'd force DOWN and skip the stop.

### `addRequest`

```
addRequest(request):
    if request.getFloor() < 0 || request.getFloor() > 9:
        return false
    if request.getFloor() == currentFloor:
        return true  // already here — no-op
    return requests.add(request)
```

The Set handles deduplication. The "already here" check avoids a useless entry.

---

## Verification Trace

Elevator at floor 3, direction=UP, requests = {Request(5, PICKUP_UP), Request(7, DESTINATION)}.

```
Tick 0: floor=3, dir=UP, requests={5-PICKUP_UP, 7-DEST}
  Case 3: Request(3, PICKUP_UP) not in set, Request(3, DEST) not in set → no stop
  Case 4: hasRequestsAhead(UP)? Yes (5, 7 > 3) → don't reverse
  Case 5: move UP → floor=4

Tick 1: floor=4 → no stop, move UP → floor=5

Tick 2: floor=5, dir=UP
  Case 3: Request(5, PICKUP_UP) in set! Remove it. requests={7-DEST}, not empty, stay UP
  Return (stopped this tick)

Tick 3: floor=5, dir=UP
  Case 4: hasRequestsAhead(UP)? Yes (7 > 5) → don't reverse
  Case 5: move UP → floor=6

Tick 4: floor=6 → no stop, move UP → floor=7

Tick 5: floor=7, dir=UP
  Case 3: Request(7, DEST) in set! Remove. requests={}, go IDLE
  Return (stopped this tick)

Tick 6: floor=7, dir=IDLE, requests={} → Case 1: stay IDLE
```

Now someone on floor 2 presses DOWN. Controller adds Request(2, PICKUP_DOWN) to this elevator.

```
Tick 7: floor=7, dir=IDLE, requests={2-PICKUP_DOWN}
  Case 2: nearest = floor 2, dist=5, direction = DOWN (2 < 7)
  Case 3: Request(7, PICKUP_DOWN) not in set → no stop
  Case 4: hasRequestsAhead(DOWN)? Yes (2 < 7) → don't reverse
  Case 5: move DOWN → floor=6

Ticks 8-11: move DOWN... floor=5,4,3,2

Tick 11: floor=2, dir=DOWN
  Case 3: Request(2, PICKUP_DOWN) in set! Remove. requests={}, go IDLE
  Return
```

---

## Extensibility

### "How would you add priority floors or an express elevator?"

Express elevators only stop at designated floors (e.g., 0, 5, 9).

```
class ElevatorController:
    - elevators: List<Elevator>
    - expressElevator: Elevator    // tracks the express one

class Elevator:
    - isExpress: boolean
    - expressFloors: Set<int> = {0, 5, 9}
```

In `addRequest`, an express elevator rejects requests to non-express floors:
```
addRequest(request):
    if isExpress && !expressFloors.contains(request.getFloor()):
        return false
    // ... rest as normal
```

In `selectBestElevator`, prefer the express elevator when the requested floor is in the express set and it's idle. Regular elevators handle all other floors. The `Request` class doesn't change — it still stores floor and type; the elevator validates which floors it accepts.

### "How would you add undo to cancel a floor request?"

Add a matching `removeRequest` to Elevator:
```
removeRequest(request):
    requests.remove(request)    // that's it
```

If the elevator already passed the floor and served the stop, the request is already gone — `remove` is a no-op. If the elevator hasn't arrived yet, the request is removed and the elevator will skip that floor. The movement logic in `step()` doesn't change at all.

### "What if multiple hall calls arrive concurrently?"

The single-threaded simulation is safe. In a real concurrent system, two problems arise:

1. Two hall calls hit `requestElevator` simultaneously, both call `selectBestElevator`, both see the same idle elevator, both dispatch to it → one gets double-booked.
2. `step()` is iterating `requests` while `addRequest` inserts into the same set → ConcurrentModificationException.

**Simple fix — coarse-grained lock:**
```
requestElevator(floor, type):
    lock.acquire()
    ...
    lock.release()

step():
    lock.acquire()
    ...
    lock.release()
```

Correct and easy to explain. Tradeoff: hall calls block while step() runs.

**Elegant fix — concurrent queue:**
```
addRequest(request):
    pendingRequests.enqueue(request)    // thread-safe queue

step():
    while !pendingRequests.isEmpty():
        activeRequests.add(pendingRequests.dequeue())
    // all logic uses activeRequests
```

Writers touch only the queue; `step()` touches only the active set. Complete separation — no contention, no lock needed.

---

## What Interviewers Look For

| Level | Bar |
|-------|-----|
| Junior | Identify Elevator + ElevatorController. Basic up/down movement. Dispatch by nearest elevator. Handle invalid floor numbers. `step()` simulation model. |
| Mid-level | Direction-aware stopping (PICKUP_DOWN stays on UP trip). Clean state machine: IDLE ↔ UP ↔ DOWN with correct transitions. Recognize naive nearest-elevator has problems. Separate RequestType and Direction enums. |
| Senior | Proactively clarify simulation vs. hardware. SCAN algorithm with correct edge cases (stop-and-return tick, no explicit boundary checks). Priority tier dispatch. Concurrent access discussion with lock ordering or queue separation. |

---

## Practice Questions

1. Why does the elevator return immediately after stopping at a floor (Case 3) rather than also checking direction and potentially moving in the same tick? What bug would occur if you moved after stopping?
2. An elevator is at floor 4 going DOWN with Request(6, PICKUP_UP) and Request(2, DESTINATION). Trace step-by-step what happens. Does the elevator stop at floor 6 on the way down? Why or why not?
3. `hasRequestsAhead` checks ANY request regardless of type. If you changed it to only check requests matching the current direction, what specific scenario would break?
4. Two elevators: E1 at floor 1 going UP, E2 at floor 8 going DOWN. Hall call at floor 5, PICKUP_UP. What does your scoring function return for each elevator, and which wins? Is floor 5 in E1's path or E2's path for the UP request?
5. The `requests` field is a `Set<Request>`. Two calls: `addRequest(Request(5, PICKUP_UP))` then `addRequest(Request(5, PICKUP_DOWN))`. Does the Set deduplicate both? Should it? What does the second request represent, and what happens to it on the way up?

## One-Line Summary

Elevator's core insight is the SCAN algorithm: sweep in one direction until no requests remain ahead, then reverse — with direction-aware stopping (only pick up passengers going your way) and a `step()` simulation model that handles five cases: empty requests, IDLE initialization, floor stop, direction reversal, and movement.
