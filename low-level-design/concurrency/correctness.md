# Correctness

## The Core Idea

Correctness in concurrency means preventing data corruption when multiple threads access shared state simultaneously. It's the most fundamental problem category — before you can think about coordination or resource management, you need to ensure your shared data stays consistent.

The root cause is always the same: an operation that looks atomic in source code compiles to multiple CPU instructions, and the OS can switch threads between any two instructions.

---

## The Mental Model

Think of a whiteboard in a shared office. One person reads the count (50), walks to their desk to add 10, comes back to write 60 — but while they were away, someone else read the same 50, added 5, and already wrote 55. The first person overwrites with 60. The second update is lost.

This is exactly what happens with `counter += 1` in concurrent code: read, modify, write. The interleaving of these three steps between threads causes lost updates.

---

## The Two Root-Cause Patterns

### Pattern 1: Check-Then-Act

Check a condition, then act on it — but another thread changes the condition between the check and the action.

```python
# Broken ticket booking
if seat.is_available():        # Thread A checks: True
    # Thread B also checks: True at the same moment
    seat.reserve(user_a)       # Thread A reserves
    # Thread B also reserves → double booking
```

**Where it shows up:** Ticket booking, rate limiters, connection pools, LRU cache eviction, parking lot management, download managers.

### Pattern 2: Read-Modify-Write

Read a value, compute something from it, write the result back — but the value changed while you were computing.

```python
# Broken counter
old = counter.get()    # Thread A reads 500
new = old + 1          # Thread A computes 501
counter.set(new)       # Thread A writes 501

# Meanwhile Thread B did the same thing:
# Read 500, computed 501, wrote 501 → one increment lost
```

**Where it shows up:** View counters, bank balances, inventory quantities, metrics aggregation (sum + count), rate limiting counters.

---

## Four Solutions (in order of complexity)

### Solution 1: Coarse-Grained Locking

One lock protects all access to shared state. Any thread that wants to read or write the shared state must hold the lock.

**The critical rule:** Hold the lock through BOTH the check and the update. Releasing between check and update recreates the race.

```python
class TicketSystem:
    def __init__(self):
        self._available = 100
        self._lock = threading.Lock()
    
    def reserve(self, user_id):
        with self._lock:        # acquire lock — only one thread enters
            if self._available <= 0:
                raise Exception("No tickets available")
            self._available -= 1
            return create_booking(user_id)
        # lock released automatically, even if exception thrown
```

**Common mistakes:**
- Using `with self._lock:` for write but forgetting it for read — reads can now observe partially-written state
- Creating a new `Lock()` object inside each method call — each call gets its own lock, which protects nothing
- Releasing the lock between check and action: `with lock: check; release; with lock: update` — back to square one

**When to use:** Human-triggered operations (booking a ticket, submitting an order) where contention is rare and simplicity matters. This is the default for most LLD interviews.

---

### Solution 2: Read-Write Locks

Multiple threads can read simultaneously; writing is exclusive. Nobody can read while a write is in progress; nobody can write while anyone is reading.

```python
import threading

class Cache:
    def __init__(self):
        self._data = {}
        self._lock = threading.RLock()  # Python doesn't have native RWLock; use rwlock library
    
    def get(self, key):
        # shared read — multiple readers can do this simultaneously
        with self._read_lock:
            return self._data.get(key)
    
    def put(self, key, value):
        # exclusive write — blocks all readers and other writers
        with self._write_lock:
            self._data[key] = value
```

**When to use:** When reads dramatically outnumber writes. Rule of thumb: if reads are ≥10x more frequent than writes (e.g., 1000 reads/sec vs. 1 write/sec), a read-write lock improves throughput. If the ratio is closer to 50/50, the overhead of the read-write lock mechanism makes a simple mutex faster.

**Don't use when:** You're tempted to use it just because it "sounds more advanced." Profile first.

---

### Solution 3: Fine-Grained Locking

One lock per resource rather than one global lock. Thread A booking seat 7A and Thread B booking seat 12B can proceed simultaneously without contention.

```python
class SeatMap:
    def __init__(self, seats):
        self._locks = {seat_id: threading.Lock() for seat_id in seats}
        self._seats = {seat_id: True for seat_id in seats}  # True = available
    
    def reserve(self, seat_id, user_id):
        with self._locks[seat_id]:   # only blocks threads competing for THIS seat
            if not self._seats[seat_id]:
                raise Exception(f"Seat {seat_id} not available")
            self._seats[seat_id] = False
            return create_booking(user_id, seat_id)
```

**The deadlock risk:** When you need to acquire multiple locks at once (e.g., transferring between two bank accounts), you can deadlock:
- Thread A holds lock(account_1), waiting for lock(account_2)
- Thread B holds lock(account_2), waiting for lock(account_1)
- Neither proceeds — deadlock

**Fix: always acquire multiple locks in a consistent canonical order:**

```python
def transfer(from_account, to_account, amount):
    # Sort accounts so we always acquire them in the same order
    first, second = sorted([from_account, to_account], key=lambda a: a.id)
    with first.lock:
        with second.lock:
            from_account.balance -= amount
            to_account.balance += amount
```

With consistent ordering, deadlock is impossible: every thread that needs both locks acquires the lower-ID lock first, so no circular wait can form.

**When to use:** Machine-scale throughput where many independent resources exist and contention on a global lock would be the bottleneck. Harder to reason about; don't use unless you need the throughput.

---

### Solution 4: Atomic Variables

CPU-level compare-and-swap (CAS) instructions that read, compare, and write as a single uninterruptible hardware operation. No lock overhead.

**CAS logic:** "If the value at this address is still `expected`, set it to `new_value` and return true. Otherwise, return false (someone else changed it)."

```python
# CAS loop — optimistic concurrency
def atomic_increment(counter_ref, delta):
    while True:
        current = counter_ref.get()
        new = current + delta
        if counter_ref.compare_and_swap(current, new):
            return new  # succeeded
        # Another thread changed it — retry with fresh value
```

**Python reality:** Python's `int` operations are not atomically guaranteed below the GIL. Use `threading.Lock()` to simulate atomics in Python. In Java, `AtomicInteger`/`AtomicLong` give you true hardware atomics.

**Limitation:** Atomics only work for a single variable. If you need to update two fields as a unit (e.g., decrement a counter AND update a timestamp together), you must use a lock. Atomics are just the fast path for the single-variable case.

**When to use:** Simple counters, flags, or sequence numbers where only one variable needs to change and no invariant spans multiple variables.

---

### Solution 5: Thread Confinement

Partition data so each thread owns its own slice and never shares state with other threads. No sharing = no race conditions.

```python
class PartitionedCounter:
    def __init__(self, num_threads):
        # Each thread gets its own counter
        self._counters = [0] * num_threads
    
    def increment(self, thread_id):
        self._counters[thread_id] += 1  # no lock needed — only one thread touches this slot
    
    def total(self):
        return sum(self._counters)  # read by a single thread at query time
```

**Real-world example:** Dragonfly (Redis alternative) uses thread confinement to scale across cores — each thread owns a hash bucket partition, eliminating cross-thread coordination for the common case.

**Tradeoff:** Cross-partition operations (moving work from one partition to another) still require coordination. Works best when operations are naturally partitioned by some key (user ID, shard key).

---

## Decision Tree

```
Is there shared mutable state?
│
├── Single variable (counter, flag)
│   ├── Python → threading.Lock() wrapping the update
│   └── Java/Go → AtomicInteger, sync/atomic
│
└── Multiple variables / invariant spans two+ fields
    └── Must use a lock
        │
        ├── Is contention a concern?
        │   ├── No (human-triggered, low frequency) → Coarse-grained lock
        │   └── Yes (machine-scale throughput)
        │       ├── Can partition by resource? → Fine-grained locking (watch for deadlock)
        │       └── Can partition by thread? → Thread confinement
        │
        └── Read-heavy (reads ≫ writes, 10:1 ratio+) → Read-write lock
```

**Interview default:** Reach for coarse-grained locking. It's correct, simple, and easy to explain. Only upgrade to fine-grained if you identify a specific throughput bottleneck that the coarse lock creates.

---

## Common Bug Patterns

| Bug | Example | Fix |
|-----|---------|-----|
| Check-then-act without lock | if available → reserve (two separate ops) | Hold lock through both check and update |
| Releasing lock between check and update | `with lock: check; release; with lock: update` | Single `with lock:` block for both |
| Inconsistent lock acquisition for multiple locks | Thread A: lock1→lock2, Thread B: lock2→lock1 | Canonical ordering by ID |
| Locking writes but not reads | `set()` is protected; `get()` is not | Protect all accesses, both read and write |
| New lock object per call | `Lock()` inside the method body | Lock is an instance variable |
| Atomic for multi-field invariant | CAS on balance only, timestamp updated separately | Use a lock when multiple fields must change together |

---

## Worked Example: Thread-Safe LRU Cache

LRU cache has multiple correctness hazards: `get()` needs to update the access order (read-modify), `put()` may need to evict the LRU entry (check-then-act on size).

```python
from collections import OrderedDict
import threading

class LRUCache:
    def __init__(self, capacity):
        self._capacity = capacity
        self._cache = OrderedDict()
        self._lock = threading.Lock()  # coarse-grained: all ops under one lock
    
    def get(self, key):
        with self._lock:
            if key not in self._cache:
                return -1
            self._cache.move_to_end(key)  # update LRU order — must be in lock
            return self._cache[key]
    
    def put(self, key, value):
        with self._lock:
            if key in self._cache:
                self._cache.move_to_end(key)
            self._cache[key] = value
            if len(self._cache) > self._capacity:
                self._cache.popitem(last=False)  # evict LRU — check+act in lock
```

Coarse-grained lock here is the right choice: cache access is low-frequency (human page requests), correctness is critical, and the simple design is easy to audit.

---

## Practice Questions

1. A booking system checks `if seat.available()` then calls `seat.reserve()` in two separate method calls, each with its own lock acquisition. Why is this still a race condition? How do you fix it?
2. You have a bank transfer method that needs to debit account A and credit account B. Show the exact sequence of events that produces deadlock when Thread 1 transfers A→B and Thread 2 transfers B→A simultaneously. Then show the fix.
3. Why can't you use a CAS atomic for a rate limiter that checks both "is the count below limit?" and "what time window are we in?" together? What do you use instead?
4. A connection pool has 10 connections. 50 threads start simultaneously. Explain what happens to threads 11-50 with a coarse-grained lock implementation that only allows one thread at a time through `getConnection()`.
5. You're designing a distributed inventory system. Each item has a count. Writes happen 10x per second per item, reads happen 100x per second per item. Is a read-write lock worth the complexity? Justify your answer.

## One-Line Summary

Correctness bugs come from two patterns — check-then-act and read-modify-write — solved by holding a lock through the entire sequence; coarse-grained locking is the interview default, with fine-grained locking or atomics only when you need the throughput.
