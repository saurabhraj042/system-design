# Dealing with Contention

## The Core Idea

Contention happens when two or more operations race to modify the same piece of data at the same moment. Left unhandled, the result is corrupt state — sold-out inventory that shows negative stock, bank accounts that go below zero, or two people owning the same hotel room. The root cause is always the same: a gap exists between *reading* a value and *writing back* the new one, and another writer sneaks into that gap.

## Mental Model / Analogy

Picture a whiteboard in an office showing "3 parking spots left." Alice reads "3," walks to her car to grab her badge. Bob reads "3" while Alice is away. Both come back and erase "3," write "2." Now the board shows 2 but only one car actually parked. The whiteboard was never the problem — the gap between reading and erasing was.

The fix in all cases below is the same move: **make your write conditional on the value not having changed since you read it.**

---

## How It Works — The 5 Tools

### Tool 1: Conditional Writes

When your safety check can be expressed as a filter on the row you're already updating, you don't need any explicit lock. Pack the condition right into the `WHERE` clause. The database handles the read and the check as a single atomic operation — no gap exists for anyone to slip into.

```sql
-- Sell a product only if stock remains
UPDATE products
SET stock = stock - 1
WHERE product_id = 'p99'
  AND stock > 0;
```

Check how many rows were affected. Zero means someone beat you to it — handle that, don't proceed.

**When to use:** Counter decrements, status flips (`available → reserved`), claiming an unassigned row. Any "if condition, then write" that the DB can evaluate in one statement.

**Limit:** The condition must live entirely in a `WHERE` clause. The moment you need application logic between the read and the write (like "find any two adjacent open seats"), you've outgrown this.

---

### Tool 2: Pessimistic Locking

When you need to read some rows, make a decision in your application code, *then* write back — you need to freeze those rows while you think. `SELECT ... FOR UPDATE` tells the database "I'm about to modify these; hold them for me until I commit."

```sql
BEGIN;

-- Lock available slots in the morning time block
SELECT slot_id FROM calendar_slots
WHERE date = '2026-08-10'
  AND doctor_id = 'd42'
  AND status = 'open'
FOR UPDATE;

-- Application picks the best slot, then claims it
UPDATE calendar_slots
SET status = 'booked', patient_id = 'p77'
WHERE slot_id = 'slot_morning_9am';

COMMIT;
```

The second person running this waits at the `FOR UPDATE` line until the first person commits or rolls back. By then the booked slot is gone and they pick from what remains.

**Common mistakes:**
- Locking too many rows or holding the lock too long (never do a slow network call inside a transaction that holds a lock)
- Deadlocks from inconsistent lock order — always lock rows sorted by a stable key (e.g., ascending ID) across all code paths

**When to use:** Read-decide-write where the decision can't be a `WHERE` predicate; high contention scenarios where conflicts are frequent enough that retry loops would be expensive.

---

### Tool 3: Optimistic Concurrency Control (OCC)

The opposite bet: assume conflicts are rare, don't block anyone, but *detect* a collision at write time. Add a version counter to your row. Read the row and note the version. Write back only if the version still matches what you read. If it doesn't, someone else already wrote — retry.

```sql
-- Step 1: read the item and its version
-- returns: { quantity: 5, version: 17 }

-- Step 2: write back with version check
UPDATE inventory
SET quantity = quantity - 1,
    version  = version + 1
WHERE item_id = 'i55'
  AND version = 17;   -- stale if this is now 18 or higher

-- Zero rows updated = conflict → re-read and retry
```

**The ABA trap:** A value that goes 5 → 4 → 5 looks unchanged to a version check if you're reusing a business value as the version. Use a dedicated, ever-incrementing integer column so the version never rolls back.

**When to use:** Read-decide-write where collisions are *rare* (most e-commerce reads). Better throughput than pessimistic locking when writers don't actually step on each other often.

---

### Tool 4: Serializable Isolation

Sometimes two transactions each read *different* rows, each makes a locally valid decision, but together they break a global rule. Neither transaction touched the same row, so there's no row to lock or version-check. This is called **write skew**.

Example: A rule says a team must always have at least one on-call engineer. Two engineers simultaneously read that the other is still on-call, so each concludes it's safe to clock off. Both commit. Now nobody is on-call.

`SERIALIZABLE` isolation tells the database to track read-write dependencies between concurrent transactions and abort one if they would have been impossible to run sequentially without breaking the rule.

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  SELECT COUNT(*) FROM on_call WHERE team = 'payments' AND active = true;
  -- sees 2, decides it's safe to clock off
  UPDATE on_call SET active = false WHERE engineer_id = 'e1';
COMMIT;
-- If a concurrent transaction made the same decision, one of them gets aborted
```

**Cost:** The DB has to track all reads, not just writes. Aborts throw away work that must be retried. Use only when the conflict genuinely spans rows and you can't collapse the invariant onto a single lockable cell.

---

### Tool 5: Distributed Locks

All four tools above live *inside* a database transaction. A transaction lock dissolves the moment you commit. Sometimes you need to hold exclusive access *across* a wait — like reserving a hotel room for 15 minutes while the user enters payment details.

The trick: stop holding exclusivity as a database lock and start holding it **as data** — a record that says who holds a reservation and when it expires. Any server can check this record.

**Three common implementations:**

| Where the lock lives | How it works | Trade-off |
|---|---|---|
| Redis (SET NX EX) | Atomic set-if-not-exists with TTL | Fast, simple; TTL can expire while holder is still working |
| DB column (reserved_by + expires_at) | Conditional UPDATE in WHERE clause | Consistent with the rest of your data; slower than cache |
| etcd / ZooKeeper | Consensus-based lease with session detection | Strongest guarantees; operationally heavy |

A DB-column reservation looks like:

```sql
UPDATE seats
SET reserved_by = 'user_88',
    reserved_until = NOW() + INTERVAL '15 minutes'
WHERE seat_id = 'A12'
  AND (reserved_by IS NULL OR reserved_until < NOW());
-- 1 row updated = you got it; 0 rows = someone else holds it
```

When `reserved_until` passes, the row automatically reads as free in your WHERE clause — no cleanup job needed.

---

## Choosing the Right Tool

Walk down this list and stop at the first one that fits:

1. **Can your safety rule be a `WHERE` predicate on the row you're writing?** → Conditional write. Simplest, no overhead.
2. **Do you need to read, decide in application code, then write?** → Pessimistic lock. Easy to reason about; use when contention is high.
3. **Same read-decide-write but conflicts are rare?** → OCC with a version column. Better throughput, occasional retry on conflict.
4. **Conflict spans rows that never both get written?** → `SERIALIZABLE` isolation, or materialize the invariant onto one lockable row.
5. **Exclusivity must outlive a single transaction?** → Distributed lock (Redis, DB column, or etcd depending on your durability needs).

---

## Key Trade-offs

| Approach | Latency | Throughput under high contention | Complexity |
|---|---|---|---|
| Conditional write | Lowest | High (no waiting) | Low |
| Pessimistic lock | Low per op, but waiters queue | Serializes writers | Low |
| OCC | Low when no conflict; retry cost on collision | High (no blocking) | Medium |
| Serializable isolation | Medium (tracking overhead) | Low (many aborts) | Medium |
| Distributed lock | Depends on backend | Medium | Medium |

---

## When to Spot This in an Interview

Call it out proactively — don't wait to be asked. When you see:
- Limited supply + multiple buyers (tickets, hotel rooms, flash sale inventory)
- Status transitions that can only happen once (`pending → confirmed`)
- Financial operations on shared balances
- Scheduling where "no conflict" is a cross-row rule

Say something like: *"Multiple users can race to grab the same slot, so I'll use a conditional update that only decrements if supply is still above zero. That closes the race without any explicit lock."*

---

## Common Gotchas

- **Zero rows updated is not an error** — your app must check the affected count and handle it, otherwise follow-up inserts will still fire
- **Don't put slow I/O inside a pessimistic lock** — calling a payment API while holding `FOR UPDATE` serializes every other writer behind that latency
- **Deadlocks need consistent lock ordering** — always acquire locks on multiple rows in the same sorted order (e.g., ascending ID) so circular waits can't form
- **The ABA problem** — a version check on a business value can pass even after a round-trip change; use a dedicated monotonic counter
- **NoSQL stores usually lack `SERIALIZABLE`** — fold cross-row invariants onto one cell you can guard with a conditional write

---

## Practice Questions

1. You're building a flash sale where 500 limited-edition items go on sale simultaneously to 100,000 users. Walk through why a naive read-then-write fails and which tool you'd pick.
2. A calendar booking system must prevent double-booking a meeting room. The selection UI lets users pick any open slot from a visual grid. Why isn't a conditional write sufficient here, and what would you use instead?
3. An auction system lets multiple bidders submit bids concurrently. The only rule: a new bid must be higher than the current max. How would you use OCC here, and what value serves as the "version"?
4. Explain write skew with your own example (not on-call scheduling). What's the minimum change to your schema or query to prevent it?
5. A ride-hailing app needs to assign a driver to exactly one rider at a time. The assignment must be held for 30 seconds while the rider confirms. Which locking approach would you choose and why?

---

## One-Line Summary

Every contention fix is just one thing: make your write conditional on the value not having changed since you read it — and pick the mechanism whose scope matches the gap you're protecting.
