# Coordination

## The Core Idea

Coordination solves the problem of threads that need to communicate with each other — not just avoid corrupting shared state, but actively hand off work, signal when something is ready, and wait efficiently without wasting CPU.

The canonical example: a producer generates tasks, a consumer processes them. The consumer needs to wait when there's nothing to do. The producer needs to wait when the consumer can't keep up. How do you implement that waiting without burning CPU or losing work?

---

## The Anti-Patterns First

Understanding what not to do makes the correct primitives clearer.

### Busy-Waiting

```python
# Broken: burns 100% CPU while waiting
def consumer():
    while True:
        if not queue.isEmpty():
            item = queue.dequeue()
            process(item)
        # else: spin and check again immediately
```

A tight loop that keeps checking a condition uses 100% of one CPU core even when there's nothing to do. On a shared system this starves other threads and spikes CPU metrics.

### Sleep-Polling

```python
# Better than busy-waiting, but still wrong
def consumer():
    while True:
        if not queue.isEmpty():
            item = queue.dequeue()
            process(item)
        else:
            time.sleep(0.1)  # arbitrary delay
```

Introducing a sleep reduces CPU waste but introduces latency: if the producer adds something 1ms after the consumer sleeps for 100ms, the consumer won't see it for another 99ms. And the sleep duration is a guess — too long means latency, too short means wasted CPU.

**The core problem:** both approaches separate the "check condition" step from the "act on condition" step in time. You need a mechanism that makes the check and the wait a single atomic operation.

---

## The Correct Tools

### Condition Variables

A condition variable (CV) lets a thread atomically release a lock and go to sleep, then re-acquire the lock when another thread signals it. This eliminates both the spin and the polling delay.

```python
import threading

lock = threading.Lock()
condition = threading.Condition(lock)
queue = []

def producer(item):
    with condition:
        queue.append(item)
        condition.notify()      # wake one waiting consumer

def consumer():
    with condition:
        while len(queue) == 0:  # WHILE not IF — spurious wakeups exist
            condition.wait()    # atomically: release lock + sleep
        item = queue.pop(0)
    process(item)
```

**Critical rule — use `while`, not `if`:**
- The OS can deliver a spurious wakeup (a wakeup not caused by `notify`)
- Another thread may have consumed the item between `notify` and when this thread actually runs
- Always re-check the condition after waking up

**Separate condition variables for producers and consumers:**

```python
not_empty = threading.Condition(lock)  # consumers wait here
not_full = threading.Condition(lock)   # producers wait here

def producer(item):
    with not_full:
        while len(queue) >= MAX_SIZE:
            not_full.wait()         # producer blocks when queue is full
        queue.append(item)
        not_empty.notify()          # wake a waiting consumer

def consumer():
    with not_empty:
        while len(queue) == 0:
            not_empty.wait()        # consumer blocks when queue is empty
        item = queue.pop(0)
        not_full.notify()           # wake a waiting producer
    process(item)
```

Using one condition variable for both directions causes unnecessary wakeups — a consumer waking another consumer that then finds an empty queue and goes back to sleep.

---

### Blocking Queues (Preferred)

The blocking queue is the preferred primitive for most producer-consumer scenarios. It encapsulates the lock, the condition variables, and the wait/notify logic internally. You just call `put` and `get`.

```python
import queue

q = queue.Queue(maxsize=100)  # bounded queue

def producer():
    q.put(item)    # blocks if queue is full (backpressure)

def consumer():
    item = q.get() # blocks if queue is empty
    process(item)
```

**When to use raw condition variables instead:** When you need more control than put/get provide — for example, a priority queue where wakeup conditions depend on item priority, or when the "condition" is not just "queue is empty" but some complex business invariant.

---

## Backpressure: What Happens When the Queue Is Full?

A bounded queue (`maxsize=N`) solves memory exhaustion, but you must decide what to do when the producer catches up to the consumer:

| Strategy | Implementation | When to use |
|----------|----------------|-------------|
| **Block** | `q.put(item)` — default behavior | Batch processing where the producer can afford to wait |
| **Timeout + reject** | `q.put(item, timeout=0.1)` → return 503 on failure | User-facing services; fail fast with a clear error |
| **Drop + log** | `q.put_nowait(item)` → except Full: log and discard | Lossy analytics, metrics, telemetry where approximate is fine |

```python
# Timeout + reject — HTTP service pattern
try:
    q.put(request, timeout=0.1)
    return "202 Accepted"
except queue.Full:
    return "503 Service Unavailable"

# Drop + log — analytics pattern
try:
    q.put_nowait(event)
except queue.Full:
    metrics.increment("dropped_events")
    logger.warning("Event queue full, dropping event")
```

**Interview default:** block (simple, no data loss). Mention timeout+reject if you're designing a user-facing service.

---

## Graceful Shutdown

When the system shuts down, consumer threads blocked on `q.get()` will wait forever. Two patterns handle this:

### Poll with Timeout

```python
def consumer(shutdown_flag):
    while not shutdown_flag.is_set():
        try:
            item = q.get(timeout=1.0)  # wake up every second to check flag
            process(item)
        except queue.Empty:
            continue  # timeout fired; re-check shutdown_flag
```

The consumer wakes up periodically even when idle, checks the shutdown flag, and exits the loop if set.

### Poison Pill

```python
POISON_PILL = None  # sentinel value

def shutdown():
    # Send one poison pill per consumer thread
    for _ in range(num_consumers):
        q.put(POISON_PILL)

def consumer():
    while True:
        item = q.get()
        if item is POISON_PILL:
            return  # exit cleanly
        process(item)
```

The shutdown path enqueues a sentinel. Each consumer that receives it exits. Simple and deterministic — shutdown happens exactly when all queued work is done.

**Which to use:** Poison pill for clean drain (process remaining items, then exit). Poll with timeout for immediate shutdown or when processing remaining items is unwanted.

---

## Actor Model

The Actor model is an alternative concurrency architecture that eliminates shared mutable state entirely. Each actor has:
- **Private state** — no other actor can read or write it directly
- **Mailbox** — an inbound message queue
- **Message handler** — processes one message at a time (single-threaded per actor)

```python
class ChatRoom:
    def __init__(self, room_id):
        self._room_id = room_id
        self._participants = []
        self._messages = []
        # All mutations happen through the message handler
    
    def handle(self, message):
        if message.type == "JOIN":
            self._participants.append(message.user)
        elif message.type == "SEND":
            self._messages.append(message.content)
            self._broadcast(message.content)
        elif message.type == "LEAVE":
            self._participants.remove(message.user)
```

**Key properties:**
- No locks in business logic — actors are single-threaded within themselves
- State mutation happens by processing one message at a time
- The mailbox is the only concurrency boundary (it's a thread-safe queue)

**When to use:** Many independent stateful entities that communicate by message rather than sharing state. Chat systems (one actor per room), trading systems (one actor per instrument), game servers (one actor per game session), IoT telemetry processing (one actor per device).

**When not to use:** When entities need to coordinate to produce a single result synchronously — the message-passing model adds latency for anything request-response.

---

## Decision Tree

```
Threads need to coordinate?
│
├── One thread hands tasks to another?
│   ├── Need backpressure (producer faster than consumer) → Bounded BlockingQueue
│   ├── Simple task handoff, no backpressure concern → BlockingQueue (unbounded)
│   └── Complex condition (not just empty/full) → Condition variable + lock
│
├── Many independent stateful entities?
│   └── Actor model (one mailbox per entity, no shared state)
│
└── Need to wait for all workers to finish?
    └── CountDownLatch / threading.Barrier
```

**Interview default:** Reach for `BlockingQueue`. It handles the lock, condition variables, and wait logic internally. Only drop to condition variables when you need behavior the blocking queue doesn't expose.

---

## Common Mistakes

| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| `if queue.isEmpty(): wait()` | Spurious wakeups and TOCTOU; another thread may consume item before this runs | `while queue.isEmpty(): wait()` |
| Single condition variable for producers AND consumers | Consumers wake up consumers; wasted context switches | Separate `not_empty` and `not_full` conditions |
| Unbounded queue for user-facing services | Queue fills memory without limit under load | Bound the queue; choose a backpressure strategy |
| No shutdown mechanism for blocked consumers | Threads blocked on `get()` prevent JVM/interpreter from exiting | Poison pill or poll-with-timeout |
| Using sleep-polling as coordination | Arbitrary latency, CPU waste, sleep duration is always a guess | Condition variable or blocking queue |

---

## Practice Questions

1. A producer generates log entries at up to 10,000/sec. Consumers write them to disk at 8,000/sec. What happens over time if the queue is unbounded? What if it's bounded with block strategy? With drop strategy?
2. Why does `condition.wait()` atomically release the lock? What bug would occur if it released the lock first, then slept?
3. You have a worker pool with 10 threads all calling `q.get()` on a shared queue. How many poison pills do you need to send to cleanly shut down all 10 workers?
4. A rate limiter uses a condition variable to make threads wait when a token bucket is empty. The bucket refills every second. Why is `while tokens == 0: wait()` correct, but `if tokens == 0: wait()` might allow too many requests through?
5. You're designing a real-time multiplayer game with 1000 simultaneous game rooms, each with independent state and its own client connections. Compare using shared state (one global lock per room) vs. the Actor model. What changes as you scale from 100 to 10,000 rooms?

## One-Line Summary

Coordination problems — threads waiting on each other — are solved by BlockingQueues (the preferred high-level primitive) or condition variables (for complex conditions); choose a backpressure strategy (block, timeout+reject, or drop) for bounded queues, and always use `while` not `if` when checking wait conditions.
