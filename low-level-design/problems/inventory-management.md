# Inventory Management

**Difficulty:** Medium | **Concepts:** Observer pattern, transfer atomicity, lock ordering, alerts outside locks, Dependency Inversion

---

## Understanding the Problem

Design an inventory management system for a multi-warehouse operation. Warehouses hold stock of various products. Stock can be added, removed, or transferred between warehouses. The system must fire alerts when product stock falls below configured thresholds.

---

## Requirements Clarification

**Core questions:**
- Is Product an entity? → No. A product is just a string ID (SKU). The system manages quantities, not product objects.
- Can one product have multiple alert thresholds? → Yes. A warehouse might alert at both 10 and 5 units remaining.
- Who handles the alert? → An external listener — the system uses Dependency Inversion (interface, not a concrete class).
- What if a transfer can't complete? → Return false (source doesn't have enough). Don't partially complete.
- Thread safety? → Mention it; the tricky case is transfer (two warehouses involved).

**Final Requirements:**
```
1. addStock(warehouseId, productId, quantity)
2. removeStock(warehouseId, productId, quantity) → boolean (false if insufficient)
3. getStock(warehouseId, productId) → int
4. transfer(productId, fromWarehouseId, toWarehouseId, quantity) → boolean
5. getWarehousesWithAvailability(productId, quantity) → List<String>
6. setLowStockAlert(warehouseId, productId, threshold, listener) → void
   - fires listener.onLowStock(warehouseId, productId, currentQuantity) when stock crosses threshold
```

---

## Core Entities

Candidate nouns: warehouse, product, inventory, stock, alert, threshold, transfer, manager.

**Product** — the system only cares about its ID. No state, no behavior relevant here. Not an entity — just a `String` key.

**Stock** — a quantity value. Not an entity.

**AlertConfig** — a threshold + listener pair. Multiple can be registered per product per warehouse. **Value object.**

**AlertListener** — an interface. External code implements it to receive notifications. **Interface.**

**Warehouse** — holds a map of productId → quantity. Manages its own stock. Fires alerts. **Entity.**

**InventoryManager** — the orchestrator. Owns all warehouses. Routes requests. Handles cross-warehouse operations. **Entity.**

| Entity | Responsibility |
|--------|---------------|
| **InventoryManager** | Routes operations to correct warehouse; handles transfers across two warehouses |
| **Warehouse** | Manages per-product quantities; stores alert configs; derives alerts to fire |
| **AlertConfig** | Immutable pair of (threshold, listener) |
| **AlertListener** | Interface for external alert handlers (Dependency Inversion) |

---

## Class Design

### AlertListener (Interface)

```
interface AlertListener:
    + onLowStock(warehouseId, productId, currentQuantity) → void
```

**Why an interface?** Dependency Inversion. The inventory system shouldn't know whether alerts go to email, Slack, a dashboard, or a database. The caller provides an implementation. This makes the inventory library usable in any context without coupling it to a specific notification system.

### AlertConfig (Value Object)

```
class AlertConfig:
    - threshold: int
    - listener: AlertListener

    + AlertConfig(threshold, listener)
    + getThreshold() → int
    + getListener() → AlertListener
```

Immutable value object. No identity — two AlertConfigs with the same threshold and listener are interchangeable.

### Warehouse

```
class Warehouse:
    - id: String
    - inventory: Map<String, Integer>                  // productId → quantity
    - alertConfigs: Map<String, List<AlertConfig>>     // productId → list of (threshold, listener)

    + Warehouse(id)
    + addStock(productId, quantity) → void
    + removeStock(productId, quantity) → boolean       // false if insufficient
    + getStock(productId) → int
    + checkAvailability(productId, quantity) → boolean
    + setLowStockAlert(productId, threshold, listener) → void
    + getAlertsToFire(productId, previousQty, newQty) → List<AlertListener>  // private helper
```

**Why `alertConfigs: Map<String, List<AlertConfig>>`?** A product can have multiple thresholds (alert at 10, alert again at 5). The inner `List<AlertConfig>` stores all registered alerts for a product. The outer map gives O(1) lookup by product ID.

**Why does Warehouse NOT fire alerts directly in `removeStock`?** Firing alerts (calling external listeners) while holding a lock is dangerous: the listener might try to call back into the inventory system, causing a deadlock. Instead, Warehouse exposes `getAlertsToFire()` — a pure computation of which listeners to notify — and the caller (InventoryManager) fires them after releasing the lock.

### InventoryManager

```
class InventoryManager:
    - warehouses: Map<String, Warehouse>

    + InventoryManager(warehouseIds)
    + addStock(warehouseId, productId, quantity) → void
    + removeStock(warehouseId, productId, quantity) → boolean
    + getStock(warehouseId, productId) → int
    + transfer(productId, fromWarehouseId, toWarehouseId, quantity) → boolean
    + getWarehousesWithAvailability(productId, quantity) → List<String>
    + setLowStockAlert(warehouseId, productId, threshold, listener) → void
```

---

## Implementation

### `addStock` and `removeStock`

```
// Warehouse.addStock(productId, quantity):
current = inventory.getOrDefault(productId, 0)
inventory[productId] = current + quantity

// Warehouse.removeStock(productId, quantity):
current = inventory.getOrDefault(productId, 0)
if current < quantity:
    return false
inventory[productId] = current - quantity
return true

// Warehouse.getStock(productId):
return inventory.getOrDefault(productId, 0)
```

`removeStock` is the atomic check-and-remove: it reads current stock and either declines or removes in one operation. No separate "check then remove" — that would be a TOCTOU race.

### Alert Computation

```
// Warehouse.setLowStockAlert(productId, threshold, listener):
config = AlertConfig(threshold, listener)
if !alertConfigs.containsKey(productId):
    alertConfigs[productId] = []
alertConfigs[productId].add(config)

// Warehouse.getAlertsToFire(productId, previousQty, newQty):  [private]
configs = alertConfigs.getOrDefault(productId, [])
toFire = []
for config in configs:
    if previousQty > config.threshold && newQty <= config.threshold:
        toFire.add(config.listener)
return toFire
```

**Alert trigger condition:** `previousQty > threshold && newQty <= threshold` — the quantity crossed the threshold downward. This prevents:
- Firing on every remove (would spam if stock stays below threshold)
- Firing when adding stock (irrelevant direction)
- Firing when stock is already below threshold before this operation

### Transfer — Three Levels of Correctness

**Bad approach — check then transfer (race condition):**
```
if from.checkAvailability(productId, quantity):    // check
    from.removeStock(productId, quantity)          // transfer (race window here)
    to.addStock(productId, quantity)
```
Between `checkAvailability` and `removeStock`, another thread could remove the same stock. The availability check becomes stale. Stock could go negative.

**Good approach — use return value of removeStock:**
```
// InventoryManager.transfer:
if !from.removeStock(productId, quantity):    // atomic: check-and-remove together
    return false
to.addStock(productId, quantity)
return true
```
`removeStock` does the check and removal atomically. If it returns false, nothing was removed. If true, removal already happened — we just need to add to the destination. This is correct for single-threaded scenarios.

**Great approach — explicit locking with canonical ordering (for concurrent scenarios):**

With multiple threads transferring simultaneously, even the "good" approach has issues: concurrent transfers of different products between the same two warehouses could interleave mid-transfer, leaving the system in an intermediate state. More seriously, two transfers going opposite directions (A→B and B→A) could deadlock if each locks A then B and B then A respectively.

```
// InventoryManager.transfer (concurrent-safe):
if quantity <= 0: return false
from = warehouses[fromWarehouseId]
to = warehouses[toWarehouseId]
if from == null || to == null: return false
if fromWarehouseId == toWarehouseId: return false

// Canonical ordering to prevent deadlock
first  = (fromWarehouseId < toWarehouseId) ? from : to
second = (fromWarehouseId < toWarehouseId) ? to   : from

previousQty = from.getStock(productId)

synchronized(first):
    synchronized(second):
        if !from.removeStock(productId, quantity):
            return false
        to.addStock(productId, quantity)

// Fire alerts OUTSIDE the lock
newQty = from.getStock(productId)
alertsToFire = from.getAlertsToFire(productId, previousQty, newQty)
for listener in alertsToFire:
    listener.onLowStock(from.id, productId, newQty)

return true
```

**Canonical lock ordering:** Always lock warehouses in the same order (alphabetical by ID). If Thread 1 wants to transfer A→B and Thread 2 wants to transfer B→A, both will try to lock A first, then B. Thread 1 gets A; Thread 2 blocks waiting for A. When Thread 1 finishes and releases both, Thread 2 proceeds. No deadlock.

**Why fire alerts outside the lock?** The `AlertListener.onLowStock()` call is external code — we can't know what it does. If it tries to acquire a lock on the warehouse (e.g., to check remaining stock), and we're still holding the warehouse lock, deadlock. Fire all alerts after releasing all locks.

### InventoryManager `transfer` (simple version for interviews)

```
transfer(productId, fromWarehouseId, toWarehouseId, quantity):
    if quantity <= 0: return false
    from = warehouses[fromWarehouseId]
    to = warehouses[toWarehouseId]
    if from == null || to == null: return false

    previousQty = from.getStock(productId)
    if !from.removeStock(productId, quantity): return false
    to.addStock(productId, quantity)

    newQty = from.getStock(productId)
    alertsToFire = from.getAlertsToFire(productId, previousQty, newQty)
    for listener in alertsToFire:
        listener.onLowStock(from.id, productId, newQty)
    return true
```

### `getWarehousesWithAvailability`

```
getWarehousesWithAvailability(productId, quantity):
    available = []
    for warehouseId, warehouse in warehouses:
        if warehouse.checkAvailability(productId, quantity):
            available.add(warehouseId)
    return available
```

---

## Verification Trace

Setup: Two warehouses W1 and W2. W1 has 15 units of "SKU-001". Alert set at threshold=10.

**Remove 6 units (15 → 9, crossing threshold):**
```
InventoryManager.removeStock("W1", "SKU-001", 6):
  warehouse = warehouses["W1"]
  previousQty = 15
  warehouse.removeStock("SKU-001", 6):
    current = 15 >= 6 → inventory["SKU-001"] = 9, return true
  newQty = 9
  alertsToFire = warehouse.getAlertsToFire("SKU-001", 15, 9):
    config threshold=10: previousQty(15) > 10 && newQty(9) <= 10 → FIRE
    return [alertListener]
  alertListener.onLowStock("W1", "SKU-001", 9) ✓

State: W1 has 9 units of SKU-001
```

**Remove 3 more units (9 → 6, already below threshold):**
```
previousQty = 9
removeStock succeeds → newQty = 6
getAlertsToFire("SKU-001", 9, 6):
  previousQty(9) > 10? NO → don't fire ✓

// No duplicate alert — threshold already crossed
```

**Transfer 5 units from W1 to W2 (W1: 6 → 1):**
```
transfer("SKU-001", "W1", "W2", 5):
  previousQty = 6
  from.removeStock("SKU-001", 5): current=6 >= 5, set to 1, return true
  to.addStock("SKU-001", 5)
  newQty = 1
  getAlertsToFire("SKU-001", 6, 1):
    previousQty(6) > 10? NO → don't fire (already below threshold)
  return true ✓
```

---

## Extensibility

### "How would you add reservation/hold before transfer?"

Similar to the movie ticket booking hold pattern: reserve stock temporarily, then confirm or release. Add `reserveStock(warehouseId, productId, quantity, timeoutMs) → String reservationId` and `confirmReservation(reservationId)`. The warehouse tracks `reservedInventory` separately from `inventory`, and `checkAvailability` subtracts reservations from on-hand stock.

### "How would you support reorder triggers (auto-purchase when stock falls below threshold)?"

The `AlertListener` interface already supports this cleanly:

```
class AutoReorderListener implements AlertListener:
    - purchaseSystem: PurchaseSystem
    - reorderQuantity: int

    onLowStock(warehouseId, productId, currentQuantity):
        purchaseSystem.createPurchaseOrder(warehouseId, productId, reorderQuantity)
```

Register this listener with the inventory manager. No changes to the inventory system — Dependency Inversion at work.

### "How would you track stock history (audit log)?"

Add an `InventoryEvent` entity and an event log:
```
class InventoryEvent:
    - warehouseId, productId, changeType (ADD/REMOVE/TRANSFER), quantity, timestamp, previousQty, newQty
```

Append to the log inside `addStock` / `removeStock`. This gives you a complete audit trail for reconciliation, debugging, and reporting. For large systems, the event log becomes an event stream (Kafka) that external systems consume.

---

## What Interviewers Look For

| Level | Bar |
|-------|-----|
| Junior | Warehouse has inventory map, manager routes to warehouse. addStock/removeStock/getStock. Basic threshold alert on remove. |
| Mid-level | Product is NOT an entity (just a string key). Multiple alerts per product (List<AlertConfig>). Alert fires only when crossing threshold (not on every remove). Return value of removeStock for atomic check-and-remove. |
| Senior | Transfer atomicity: use removeStock's return value (not check-then-remove). Lock ordering for concurrent transfers (canonical order prevents deadlock). Alerts fired outside lock (prevents deadlock with external code). Observer/Dependency Inversion for alert listeners. |

---

## Practice Questions

1. Why is Product not an entity in this design? What would need to be true about the requirements for Product to become an entity?
2. The "bad" transfer approach calls `checkAvailability()` then `removeStock()`. Describe the exact sequence of operations that two threads could interleave to cause stock to go below zero.
3. `getAlertsToFire` uses `previousQty > threshold && newQty <= threshold`. What would happen if the condition were just `newQty <= threshold`? Construct a scenario where this causes a problem.
4. Two threads simultaneously call `transfer("SKU-001", "W1", "W2", 5)` and `transfer("SKU-001", "W2", "W1", 5)`. Without canonical lock ordering, show the exact interleaving that causes deadlock. Then show why canonical ordering prevents it.
5. `AlertListener.onLowStock()` is called outside the warehouse lock. If the listener calls `inventoryManager.getStock(warehouseId, productId)`, could there be a consistency issue (the stock changed between the alert firing and the listener reading)? Is this acceptable? Why?

## One-Line Summary

Inventory Management's core insight is that Product is just a string key (not an entity), transfer atomicity comes from using `removeStock`'s return value (not a separate availability check), alerts are computed inside the lock but fired outside it (preventing deadlock), and `AlertListener` as an interface applies Dependency Inversion to decouple the inventory system from any specific notification mechanism.
