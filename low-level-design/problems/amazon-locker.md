# Amazon Locker

**Difficulty:** Easy | **Concepts:** Entity identification, bearer tokens, physical state vs. relational state, access control

---

## Understanding the Problem

Amazon Locker is a self-service package pickup system. A delivery driver arrives with a package, deposits it into a compartment, and the system generates an access code. The customer uses that code to open the compartment and retrieve their package. Compartments come in three sizes (small, medium, large). Codes expire after 7 days.

---

## Requirements Clarification

**Core actions:**
- How does size matching work? → Match exactly to compartment size; if no match, reject deposit
- What does the system return after deposit? → The access token code (how it reaches the customer is out of scope)
- What happens to an expired code? → Reject the pickup; package stays until staff removes it
- What if the code is wrong (never existed or already used)? → Return error; distinguish expired from invalid
- What if all compartments of the right size are full? → Return error; no queueing

**Scope:**
- Just the locker operations: deposit and pickup
- Delivery logistics and driver assignment are out of scope
- SMS/email notification of the code is out of scope
- No lockout on failed attempts (just validate the code)

**Final Requirements:**
```
1. Driver deposits package by specifying size (small/medium/large)
   - System assigns available compartment of matching size
   - Opens compartment, returns access token code; error if no space
2. One access token per package; customer can have multiple packages
3. Customer retrieves by entering access token code
   - System validates (not expired, exists), opens compartment, cleans up
4. Tokens expire after 7 days; expired codes rejected with specific error
5. Package stays in compartment until staff physically removes it
6. Staff can call openExpiredCompartments() to handle expired packages

Out of scope: delivery logistics, SMS/email, lockout logic, UI, multi-station, payment
```

---

## Core Entities

Work through the candidate nouns from the requirements:

**Package** — Our system only cares about one attribute: size. Package doesn't maintain state, doesn't enforce rules, has no identity within our system. It's just a size input parameter. **Not an entity.**

**Compartment** — A physical container with a fixed size and a current occupancy state. Clearly an entity.

**Locker** — Someone needs to orchestrate: scan for available compartments, generate codes, tie them to compartments. That's the Locker — the system's single entry point. **Entity.**

**AccessToken** — At first glance this looks like a string field on Compartment. But an access token is a *bearer token* — it represents the right to open a specific compartment, with an expiration. It owns expiry logic and the mapping to a compartment. Worth modeling as its own entity.

| Entity | Responsibility |
|--------|---------------|
| **Locker** | Orchestrator. Holds all compartments and the token lookup map. Handles deposit and pickup. |
| **AccessToken** | Bearer token with a code, expiration timestamp, and a reference to the compartment it unlocks. Enforces expiry logic. |
| **Compartment** | Physical slot. Has a size and tracks its own occupancy state (occupied/available). |

**Key insight:** Package is not an entity because our system only uses one attribute of it — size — and that's simply an input parameter to `depositPackage`. When the system only uses one property of an external entity, demote it to a parameter.

---

## Class Design

### Locker (Orchestrator)

| Requirement | What Locker must track |
|-------------|----------------------|
| "Assigns available compartment of matching size" | All compartments (iterable to find matches) |
| "User retrieves by entering access token" | Map from token code → AccessToken for O(1) lookup |

```
class Locker:
    - compartments: Compartment[]
    - accessTokenMapping: Map<string, AccessToken>

    + Locker(compartments)
    + depositPackage(size) → string | error
    + pickup(tokenCode) → void | error
    + openExpiredCompartments() → void
```

**Why does `depositPackage` only return the token code, not the compartment number?** The driver sees the door open. They don't need the compartment number — they walk up and put the package in the door that's open. We return the code so it can be forwarded to the customer.

**Why does `pickup` return void?** The customer enters their code and the correct door opens. They don't need a compartment number returned — the physical door is their feedback.

### AccessToken

| Requirement | What AccessToken must track |
|-------------|---------------------------|
| "Access token generated and returned" | The actual code string |
| "Tokens expire after 7 days" | An expiration timestamp |
| "System validates and opens compartment" | A reference to the compartment it unlocks |

```
class AccessToken:
    - code: string
    - expiration: timestamp
    - compartment: Compartment

    + AccessToken(code, expiration, compartment)
    + isExpired() → boolean      // now() >= expiration
    + getCompartment() → Compartment
    + getCode() → string
```

AccessToken owns expiration logic because it owns the timestamp. Callers just call `isExpired()` and decide what to do.

### Compartment

```
class Compartment:
    - size: Size
    - occupied: boolean

    + Compartment(size)
    + getSize() → Size
    + isOccupied() → boolean
    + markOccupied() → void
    + markFree() → void
    + open() → void     // triggers physical door unlock mechanism

enum Size:
    SMALL
    MEDIUM
    LARGE
```

**State ownership decision:** `occupied` is physical state that belongs to the Compartment — it describes whether a package is physically present. The `accessTokenMapping` is relational state (which code maps to which compartment) and belongs in the Locker orchestrator.

An alternative would be to track occupancy in a Set in Locker. Both are defensible. "I put `occupied` on Compartment because physical presence is intrinsic to the compartment" is a good answer. What matters is being able to explain the tradeoff.

---

## Implementation

### `depositPackage`

**Happy path:**
1. Find an available compartment of requested size
2. Open it (hardware unlock)
3. Mark it occupied
4. Generate an access token for it
5. Store token in the lookup map
6. Return the token code

**Edge cases:**
- No compartment of requested size available → throw error
- Invalid size parameter → caught when scanning compartments (no match found)

```python
depositPackage(size):
    compartment = getAvailableCompartment(size)
    if compartment == null:
        throw Error("No available compartment of size " + size)
    
    compartment.open()
    compartment.markOccupied()
    accessToken = generateAccessToken(compartment)
    accessTokenMapping[accessToken.getCode()] = accessToken
    
    return accessToken.getCode()

getAvailableCompartment(size):
    for compartment in compartments:
        if compartment.getSize() == size && !compartment.isOccupied():
            return compartment
    return null

generateAccessToken(compartment):
    code = generateRandomCode()    // cryptographically secure; UUID or 6-digit number
    expiration = now() + 7.days()
    return AccessToken(code, expiration, compartment)
```

### `pickup`

**Happy path:**
1. Look up access token by code
2. Validate (not expired)
3. Open compartment
4. Clean up state

**Edge cases:**
- Code is null or empty
- Code doesn't exist in the map (never existed or already used after cleanup)
- Code exists but is expired

```python
pickup(tokenCode):
    if tokenCode == null || tokenCode.isEmpty():
        throw Error("Invalid access token code")
    
    accessToken = accessTokenMapping[tokenCode]
    if accessToken == null:
        throw Error("Invalid access token code")
    
    if accessToken.isExpired():
        throw Error("Access token has expired")
    
    compartment = accessToken.getCompartment()
    compartment.open()
    clearDeposit(accessToken)

clearDeposit(accessToken):
    compartment = accessToken.getCompartment()
    compartment.markFree()
    accessTokenMapping.remove(accessToken.getCode())
```

**Important:** After `clearDeposit`, the token is removed from the map. A second attempt with the same code returns "Invalid access token code" — identical to a code that never existed. This is intentional: once picked up, there's no benefit to distinguishing "already used" from "never existed." If you wanted to distinguish them, you'd need a `usedTokens` set or an `isUsed` flag before removal.

**Expired tokens stay in the map.** Throwing an error leaves the expired token and its compartment exactly as-is. Staff must physically remove the package before the compartment becomes available again.

### `openExpiredCompartments`

```python
openExpiredCompartments():
    for tokenCode, accessToken in accessTokenMapping:
        if accessToken.isExpired():
            compartment = accessToken.getCompartment()
            compartment.open()
    // Staff see which doors opened, remove packages, then call clearDeposit for each
```

We do NOT call `clearDeposit` here because the compartment is still physically occupied — staff needs to remove the package first. Only after staff confirm removal would a cleanup method free the compartment and remove the token.

---

## Verification Trace

Locker has compartments A (SMALL), B (MEDIUM), C (LARGE), all unoccupied. Token map empty.

**Deposit a medium package:**
```
depositPackage(MEDIUM):
  getAvailableCompartment(MEDIUM) → Compartment B (size matches, !isOccupied)
  B.open() → hardware unlocks
  B.markOccupied() → B.occupied = true
  generateAccessToken(B) → AccessToken("ABC123", now+7days, B)
  accessTokenMapping["ABC123"] = accessToken
  return "ABC123"

Result: "ABC123"
State: B.occupied=true, accessTokenMapping={"ABC123" → AccessToken}
```

**Successful pickup:**
```
pickup("ABC123"):
  accessTokenMapping.get("ABC123") → AccessToken exists
  accessToken.isExpired() → now < expiration → false
  accessToken.getCompartment() → B
  B.open() → door unlocks
  clearDeposit(accessToken):
    B.markFree() → B.occupied = false
    accessTokenMapping.remove("ABC123")

State: B.occupied=false, accessTokenMapping={}
```

**Expired pickup (8 days later):**
```
pickup("ABC123"):
  accessTokenMapping.get("ABC123") → AccessToken still exists (wasn't cleared)
  accessToken.isExpired() → now > expiration → true
  throw Error("Access token has expired")

State: B.occupied=true, accessTokenMapping={"ABC123" → AccessToken (expired)}
```
Staff calls `openExpiredCompartments()`, physically removes package, then cleanup runs.

---

## Extensibility

### "How would you handle size fallback?"

Current behavior: if all medium compartments are full, reject the deposit even if large compartments are available. Fallback means small packages can use a larger compartment as a last resort.

Change only `getAvailableCompartment` — the rest of the design is untouched:

```python
getAvailableCompartment(requestedSize):
    sizesInOrder = [SMALL, MEDIUM, LARGE]
    startIndex = sizesInOrder.indexOf(requestedSize)
    for i from startIndex to sizesInOrder.length - 1:
        size = sizesInOrder[i]
        for c in compartments:
            if c.getSize() == size && !c.isOccupied():
                return c
    return null
```

Try exact size first, then move to larger sizes. Never fall back to smaller (package won't fit).

### "How would you handle broken or out-of-service compartments?"

Replace the binary `occupied: boolean` with a three-state enum:

```
enum CompartmentStatus:
    AVAILABLE
    OCCUPIED
    OUT_OF_SERVICE

class Compartment:
    - size: Size
    - status: CompartmentStatus

    + isAvailable() → boolean    // return status == AVAILABLE
    + markOccupied()             // status = OCCUPIED
    + markAvailable()            // status = AVAILABLE
    + markOutOfService()         // status = OUT_OF_SERVICE
    + markInService()            // status = AVAILABLE
```

`getAvailableCompartment` already calls `!c.isOccupied()` — just rename that call to `c.isAvailable()` and the allocation logic automatically skips out-of-service compartments. No other code changes needed.

### "How would you verify the package was actually deposited?"

Current approach: fire-and-forget. We open the compartment, assume the driver deposits the package, and immediately generate the access token.

For verified deposit, split into two operations:

```
class Locker:
    + reserveCompartment(size) → reservationId
    + confirmDeposit(reservationId) → tokenCode
    + cancelReservation(reservationId) → void

enum CompartmentStatus:
    AVAILABLE, RESERVED, OCCUPIED, OUT_OF_SERVICE
```

Flow: driver calls `reserveCompartment` → gets `reservationId` → door opens → places package → calls `confirmDeposit(reservationId)` → gets access token code.

Tradeoff: adds a RESERVED state, requires a reservation mapping, needs timeout logic (if driver reserves but never confirms within ~3 minutes, auto-cancel and free the compartment). Interview scope: single-phase approach is clean and sufficient. Production: two-phase is essential for guaranteeing package presence.

---

## What Interviewers Look For

| Level | Bar |
|-------|-----|
| Junior | Identify the three core entities (Locker, Compartment, AccessToken). Deposit and pickup work for the happy path. Handles expired tokens and full compartments. |
| Mid-level | Recognize that Package isn't an entity (just a size param). Clean three-class model arrived at through reasoning. Separation of concerns clear: Locker orchestrates, AccessToken handles expiry, Compartment manages physical state. Can explain why occupied flag lives on Compartment. |
| Senior | Proactively discusses tradeoffs: where occupancy state lives, lazy vs. eager cleanup of expired tokens, when you'd introduce a Package entity (multiple packages per compartment scenario). Brings up extensibility scenarios (size fallback, maintenance states, two-phase deposit) before being asked. |

---

## Practice Questions

1. Why is Package not modeled as a class in this design? What criterion decides whether a noun becomes an entity or stays as a parameter?
2. After a successful pickup, the access token is removed from the map. A customer then tries to use the same code again and gets "Invalid access token code." Is there a case where you'd want to distinguish "already used" from "never existed"? How would you add that without changing the main flow?
3. The `openExpiredCompartments()` method opens doors but does not call `clearDeposit`. Why not? What happens to the compartment and token after staff remove the package?
4. Trace through what happens if two delivery drivers simultaneously try to deposit medium packages and there's only one medium compartment available. How does the single-threaded design handle this? What would you need to add for concurrent access?
5. Design the extensibility for a scenario where a customer has multiple packages waiting in different compartments. They enter a single master code and all their compartments open. What entities and methods change?

## One-Line Summary

Amazon Locker's key insight is that Package is not an entity (just a size parameter) while AccessToken is (it has identity, expiry behavior, and compartment ownership) — Locker orchestrates, AccessToken controls access, and Compartment manages its own physical state.
