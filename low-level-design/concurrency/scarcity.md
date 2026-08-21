# Scarcity

## The Core Idea

Scarcity problems arise when multiple threads compete for limited resources. Unlike correctness (preventing corruption) or coordination (making threads wait on each other), scarcity is about governance: ensuring that the number of active threads, the volume of resource consumption, or the pool of reusable objects stays within safe bounds.

Without scarcity controls, one of three bad things happens:
- Threads exhaust resource pools (all database connections taken, file handles hit OS limits)
- Memory fills up (unbounded queues)
- Throughput collapses (too many concurrent ops cause CPU saturation or I/O contention)

---

## The Primary Tool: Semaphores

A semaphore is a counting permit. It starts with N permits. `acquire()` takes one permit (blocks if count is 0). `release()` returns one permit.

```python
import threading

semaphore = threading.Semaphore(5)  # 5 permits = 5 concurrent operations allowed

def limited_operation():
    with semaphore:          # acquire on enter, release on exit
        do_expensive_work()  # at most 5 threads here at once
```

The `with` statement guarantees release even if an exception is thrown. Never call `acquire()` and `release()` manually — it's too easy to leak permits on exception paths.

**The mental model:** Think of a semaphore as a bouncer at the door with N wristbands. Each thread entering the critical section takes a wristband; each thread leaving returns one. The bouncer blocks threads at the door when all wristbands are out.

---

## Semaphore vs. Resource Pool

A common interview question: "Should I use a semaphore or a connection pool?"

**The answer:** It depends on whether you're counting operations or handing out actual objects.

```
Semaphore: "At most N threads can be in this section simultaneously"
Pool:       "Here is a Connection object — don't use it while someone else is"
```

A semaphore governs *how many* can run concurrently. The threads bring their own resources or create them fresh.

A resource pool *holds actual objects* and lends them out one at a time. The pool ensures each object is used by at most one thread simultaneously and that expensive objects (DB connections, GPU handles) are reused rather than re-created.

"A semaphore limits concurrent ops, but where do the actual Connection objects live? That's what the pool solves."

---

## Resource Pools: Implementation

A blocking queue of actual resource objects is the clean implementation of a pool.

```python
import queue
import threading

class ConnectionPool:
    def __init__(self, size):
        self._pool = queue.Queue(maxsize=size)
        for _ in range(size):
            self._pool.put(create_database_connection())  # pre-warm
    
    def get_connection(self):
        return self._pool.get()    # blocks if no connections available
    
    def return_connection(self, conn):
        self._pool.put(conn)       # returns to pool; wakes waiting thread

# Always use try/finally to guarantee return
def execute_query(sql):
    conn = pool.get_connection()
    try:
        return conn.execute(sql)
    finally:
        pool.return_connection(conn)  # ALWAYS returns, even on exception
```

**The `finally` block is non-negotiable.** If a thread takes a connection and throws an exception without returning it, that connection slot is gone forever. Over time, the pool drains to zero and all threads block indefinitely.

**Why pre-warm?** Creating a connection is expensive (TCP handshake, auth, TLS). Creating all N connections at startup amortizes that cost to initialization time instead of request time.

---

## The Four Scarcity Patterns

### Pattern 1: Limit Concurrent Operations

Use when you want to cap how many threads are doing something simultaneously, but there's no specific object to hand out — just CPU or network bandwidth to protect.

```python
# Cap concurrent image resize jobs to 10
semaphore = threading.Semaphore(10)

def resize_image(path):
    with semaphore:
        return cpu_intensive_resize(path)
```

Use cases: CPU-intensive tasks (image processing, ML inference), concurrent API calls to a rate-limited third party, background job processing.

### Pattern 2: Limit Aggregate Consumption

Use when each operation consumes a variable amount of a resource (memory, bandwidth, disk I/O), and you want to cap total consumption rather than concurrent count.

```python
# Limit total concurrent download bandwidth to 1 GB
# Each download acquires permits proportional to its size (1 permit = 1 MB)
bandwidth_semaphore = threading.Semaphore(1024)  # 1024 MB total

def download(url, size_mb):
    bandwidth_semaphore.acquire(size_mb)   # acquire N permits
    try:
        return fetch(url)
    finally:
        bandwidth_semaphore.release(size_mb)
```

This is less common (Python's `Semaphore` doesn't support acquiring N at once natively — you'd implement it manually), but the pattern is correct for resource accounting.

### Pattern 3: Reuse Expensive Objects

The resource pool pattern above. Use when objects are expensive to create, stateful, and must be exclusive to one thread at a time.

Canonical examples:
- Database connections (TCP connection + auth = hundreds of milliseconds)
- SSL/TLS sessions (cryptographic key exchange)
- GPU computation contexts
- File system handles at OS limits

### Pattern 4: Maximize Utilization

When the bottleneck is idle resources rather than overuse, patterns to maximize throughput:

**Work stealing:** A thread that finishes its queue dequeues from a neighbor's queue rather than becoming idle. Used in Java's ForkJoinPool, Go's goroutine scheduler.

**Batching:** Instead of processing one item at a time, accumulate items for a window (time or count) and process them together. A single database INSERT with 100 rows is typically faster than 100 individual INSERTs.

```python
def batch_writer(batch_queue, db, max_batch=100, max_wait=0.05):
    batch = []
    deadline = time.time() + max_wait
    while time.time() < deadline and len(batch) < max_batch:
        try:
            item = batch_queue.get(timeout=deadline - time.time())
            batch.append(item)
        except queue.Empty:
            break
    if batch:
        db.bulk_insert(batch)
```

**Adaptive sizing:** Dynamic thread pools that grow under load and shrink under idle. Java's `ThreadPoolExecutor` with `corePoolSize` and `maximumPoolSize` does this automatically.

---

## Decision Tree

```
Scarcity problem?
│
├── Cap concurrent operations (no object to hand out)
│   └── Semaphore(N)
│
├── Cap aggregate resource consumption (variable per-op cost)
│   └── Semaphore with N = total budget, acquire(size) per operation
│
├── Reuse expensive stateful objects
│   └── Resource pool (BlockingQueue of actual objects)
│       ├── Always release in finally block
│       └── Pre-warm at startup
│
└── Resources are idle (utilization problem)
    ├── Per-thread batch accumulation → batching
    ├── Threads with empty queues → work stealing
    └── Unpredictable load → adaptive thread pool sizing
```

**Interview default:** If the question is about limiting concurrent access, use a semaphore. If the question is about reusing DB connections or other stateful objects, use a pool (BlockingQueue of objects with a finally block).

---

## Common Mistakes

| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Forgetting `finally` block in pool | One exception leaks a permit or object forever; pool drains over time | Always use `try/finally` or `with` statement |
| Unbounded queues feeding a pool | Queue grows without limit under load → OOM even though pool is bounded | Bound the queue; pick a backpressure strategy |
| Blocking indefinitely on request paths | A slow consumer of pool objects causes the request thread to hang | Use `pool.get(timeout=N)` and return 503 on timeout |
| Using a semaphore when you need the actual objects | Semaphore only controls count; threads still need to create/find the object themselves | Use a pool when you need the object handed to you |
| Creating pool objects lazily (on first use) | Initialization latency at request time; thundering herd when multiple threads init simultaneously | Pre-warm pool at startup |

---

## Worked Example: Rate Limiter with Semaphore

A service can handle at most 100 concurrent requests. Additional requests should wait (backpressure) up to 500ms, then fail fast with a 503.

```python
class ServiceGateway:
    def __init__(self, max_concurrent=100):
        self._semaphore = threading.Semaphore(max_concurrent)
    
    def handle_request(self, request):
        acquired = self._semaphore.acquire(timeout=0.5)
        if not acquired:
            raise Exception("Service at capacity (503)")
        try:
            return process(request)
        finally:
            self._semaphore.release()
```

This is cleaner than a full pool because there's no "connection object" — just a budget of concurrent execution slots.

---

## Practice Questions

1. A connection pool has 10 connections. 50 threads call `get_connection()` simultaneously. Trace what happens to threads 11-50. What exactly blocks them, and what unblocks them?
2. Why is `pool.get(timeout=2.0)` better than `pool.get()` (blocking indefinitely) on a web request path? What happens to thread 11 if threads 1-10 each hold connections for 10 seconds and you use blocking?
3. A semaphore starts with 5 permits. Thread A calls `acquire()` five times without releasing. What happens when Thread B calls `acquire()`? What happens when Thread A calls `release()` once?
4. You want to limit total memory used by in-flight requests to 512 MB. Each request estimates its memory footprint. Why is a simple semaphore (1 permit = 1 MB) not the right tool in Python? How would you implement this correctly?
5. A batch job reads rows from a database, transforms them, and writes to a file. The read step is 2x faster than the write step. How does this cause resource starvation, and what mechanism prevents unbounded memory growth?

## One-Line Summary

Scarcity problems — threads competing for limited resources — are solved by semaphores (limit concurrent count), resource pools (reuse expensive objects via a BlockingQueue with a mandatory `finally` release), or utilization patterns (batching, work stealing, adaptive sizing); the `finally` block is non-negotiable.
