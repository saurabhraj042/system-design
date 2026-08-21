# Introduction to Concurrency

## The Core Idea

Most software runs on machines with multiple CPU cores. Concurrency is how we take advantage of that parallelism — running multiple things at once to improve throughput and responsiveness. But concurrency also introduces a whole category of bugs that don't exist in single-threaded code: race conditions, deadlocks, and starvation. Understanding why these happen is prerequisite knowledge for fixing them.

---

## Processes vs. Threads

**Process:** An isolated running program with its own address space, file descriptors, and resources. The operating system treats processes as independent — they don't share memory unless they explicitly set up shared memory mechanisms.

**Thread:** A unit of execution within a process. Multiple threads in the same process share the same heap (dynamically allocated memory) and global variables. Each thread has its own stack (local variables and function call frames).

```
Process (Python interpreter):
├── Heap (shared by all threads)
│   ├── objects, data structures
│   └── global variables
├── Thread 1 (stack, registers)
├── Thread 2 (stack, registers)
└── Thread 3 (stack, registers)
```

**Why shared heap matters:** When two threads modify the same heap object simultaneously without coordination, results are unpredictable. This is the root cause of most concurrency bugs.

---

## A Note on Python's GIL

Python has a Global Interpreter Lock (GIL) — a mutex that prevents multiple Python threads from executing Python bytecode simultaneously. Only one thread runs at a time.

**Consequences:**
- CPU-bound work (computation): Python threads don't parallelize. Use `multiprocessing` for true parallelism.
- I/O-bound work (network, disk): Python threads do help, because the GIL is released while waiting on I/O. Multiple threads can be "in flight" even if only one runs at a time.

**In LLD interviews:** The concepts (locks, semaphores, condition variables) are the same across all languages. Python's GIL is an implementation detail, but knowing it shows depth. For interview pseudocode, you can use `threading.Lock`, `threading.Semaphore`, etc. — they work correctly because the GIL isn't relevant to the logical correctness of your solution.

---

## Why Concurrency Is Hard

The core problem: threads can interleave in ways you can't predict. The operating system can switch between threads at any point — including in the middle of what looks like a single operation in your source code.

### Example: The Lost Update

```python
# Global counter, two threads each increment it 1000 times
counter = 0

def increment():
    global counter
    for _ in range(1000):
        counter += 1  # NOT atomic — three operations at CPU level

# Run two threads simultaneously
# Expected: counter == 2000
# Actual: counter might be anywhere from 1001 to 2000
```

`counter += 1` looks like one operation but compiles to three:
1. Read current value into register
2. Add 1 to register
3. Write register back to memory

If Thread A reads the value (say, 500), gets preempted, Thread B reads the same value (500), increments to 501, writes back, then Thread A continues with its stale 500, increments to 501, writes back — one increment is lost.

This class of bug is called a **race condition**: the result depends on the unpredictable timing of thread execution.

---

## The Three Problem Categories

Every concurrency problem in LLD interviews falls into one of three categories:

### 1. Correctness — Shared State Corruption

**Problem:** Multiple threads read and write shared state, producing corrupted results.

**Example patterns:**
- Two ticket buyers check availability, both see 1 ticket, both reserve it → double booking
- Two threads both increment a counter → lost updates
- Thread reads an object that another thread is partially writing → torn read

**Tools:** Locks/mutexes, atomic variables, immutable data structures

### 2. Coordination — Threads Waiting on Each Other

**Problem:** Threads need to communicate and hand off work. One thread produces data, another consumes it. Threads need to wait until a resource is ready.

**Example patterns:**
- Worker thread waiting for a task to appear in a queue
- Producer waiting when queue is full, consumer waiting when queue is empty
- Main thread waiting for all workers to finish

**Tools:** Condition variables, blocking queues, semaphores, CountDownLatch

### 3. Scarcity — Limited Resources

**Problem:** Multiple threads compete for a limited pool of resources. Without limits, you exhaust resources (connection pool, thread pool, file handles).

**Example patterns:**
- 200 threads all trying to open database connections at once → exhaustion
- Unlimited queue depth → out-of-memory crash
- N concurrent downloads allowed, rest must wait

**Tools:** Semaphores, resource pools, thread pools with bounded queues

---

## Core Concurrency Primitives

### Locks (Mutexes)

A mutex (mutual exclusion lock) ensures only one thread executes a critical section at a time. Acquiring a locked mutex blocks until the holder releases it.

```python
import threading

lock = threading.Lock()
counter = 0

def increment():
    global counter
    with lock:  # acquire on enter, release on exit (even if exception)
        counter += 1  # only one thread here at a time
```

**Critical rule:** Always acquire the same lock for all accesses to shared state. Protecting writes but not reads still causes races.

### Atomic Variables

Low-level primitives that perform read-modify-write as a single uninterruptible CPU instruction (compare-and-swap). No lock overhead, but only work for single-variable operations.

```python
# Python doesn't have true atomics, but this is the concept:
# CAS pseudocode:
def compare_and_swap(variable, expected, new_value):
    if variable == expected:
        variable = new_value
        return True
    return False
```

Use for counters and flags. Fall back to locks when you need to update multiple fields together as a unit.

### Semaphores

A counting permit. `acquire()` decrements the count (blocks if count is 0). `release()` increments the count.

```python
import threading

# Only 5 threads can do something at once
semaphore = threading.Semaphore(5)

def limited_operation():
    with semaphore:  # blocks if 5 already acquired
        do_expensive_work()
```

Use to limit concurrent access to resources — not to hand out actual resource objects (use a pool for that).

### Condition Variables

Mechanism for threads to wait until a condition is satisfied, then be woken up. Always used together with a lock.

```python
import threading

lock = threading.Lock()
condition = threading.Condition(lock)
queue = []

def producer():
    with condition:
        queue.append(item)
        condition.notify_all()  # wake waiting consumers

def consumer():
    with condition:
        while len(queue) == 0:
            condition.wait()  # releases lock + sleeps atomically
        item = queue.pop(0)
        # process item
```

**Critical:** Use `while` not `if` before checking the condition. Spurious wakeups can occur, and another thread may have consumed the item before this thread runs.

### Blocking Queues

Thread-safe queue where `put()` blocks if full and `get()` blocks if empty. The preferred primitive for producer-consumer coordination — it handles locking and condition signaling internally.

```python
import queue

q = queue.Queue(maxsize=100)  # bounded

def producer():
    q.put(item)  # blocks if queue full

def consumer():
    item = q.get()  # blocks if queue empty
    process(item)
```

In interviews, prefer blocking queues over raw condition variables unless you have a specific reason to go lower level.

---

## Common Concurrency Problems

### Race Condition: Check-Then-Act

A common pattern: check a condition, then act on it — but the condition changes between the check and the action.

```python
# Broken: check and act not atomic
if seat.is_available():  # Thread A checks: available
    # Thread B also checks: available
    seat.reserve(user_a)  # Both threads reserve
    # Thread B also reserves → double booking
```

Fix: hold the lock through both the check and the action.

### Race Condition: Read-Modify-Write

Read a value, compute something, write back — but the value changes between read and write.

```python
# Broken
old = counter.get()   # both threads read 500
new = old + 1         # both compute 501
counter.set(new)      # both write 501 → one increment lost

# Fixed: atomic increment or lock around all three steps
with lock:
    counter += 1
```

### Deadlock

Thread A holds lock 1, waiting for lock 2. Thread B holds lock 2, waiting for lock 1. Neither can proceed.

```python
# Deadlock scenario
def transfer(from_account, to_account, amount):
    with from_account.lock:        # Thread A holds account_1 lock
        with to_account.lock:      # Thread A waiting for account_2 lock
            # Meanwhile Thread B holds account_2, wants account_1
            pass
```

**Fix:** Always acquire multiple locks in the same canonical order (e.g., by account ID):
```python
def transfer(a, b, amount):
    first, second = sorted([a, b], key=lambda acc: acc.id)
    with first.lock:
        with second.lock:
            pass  # no deadlock — consistent ordering
```

### Starvation

A thread never gets to run because other threads keep taking the resources it needs. Common with priority queues where high-priority tasks always jump ahead of low-priority ones.

---

## Decision Tree: Which Primitive?

```
Problem type?
│
├── Correctness (shared state)
│   ├── Single variable → atomic or lock
│   └── Multiple variables together → lock (same lock for all)
│
├── Coordination (threads waiting on each other)
│   ├── Simple task handoff → BlockingQueue
│   └── Complex condition → Condition variable
│
└── Scarcity (limiting access)
    ├── Limit concurrent operations (no actual objects) → Semaphore
    └── Share stateful objects (connections, handles) → Resource pool (blocking queue of objects)
```

---

## Practice Questions

1. You have a class `Counter` with a method `increment()`. Two threads call it 10,000 times each. Explain why the final value might be less than 20,000 and show the fix.
2. A web server has a thread pool of 50 threads. 200 requests arrive simultaneously. Describe what happens to requests 51-200. What primitive should limit the active count?
3. Explain why `while (queue.isEmpty()) { wait(); }` is needed instead of `if (queue.isEmpty()) { wait(); }`.
4. Thread A needs account_1 lock, then account_2 lock. Thread B needs account_2 lock, then account_1 lock. Show the exact execution sequence that produces deadlock, then show the fix.
5. You want to limit an image resizing service to 10 concurrent jobs (to avoid CPU saturation), but there are no "job objects" to pool — just a counter of active jobs. Which primitive is more appropriate: a semaphore or a blocking queue? Why?

## One-Line Summary

Concurrency problems fall into three categories — correctness (shared state corruption), coordination (threads waiting on each other), and scarcity (limited resources) — each solved by a different set of primitives: locks and atomics for correctness, blocking queues and condition variables for coordination, semaphores and pools for scarcity.
