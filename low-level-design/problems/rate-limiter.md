# Rate Limiter

**Difficulty:** Medium | **Concepts:** Strategy pattern, Token Bucket, Sliding Window Log, per-key locking, lazy initialization, retryAfterMs

---

## Understanding the Problem

Design a rate limiter library. API endpoints need protection against abuse: too many requests from one client in a time window should be rejected. Different endpoints may have different rules (some use token bucket, some use sliding window). The limiter must be thread-safe and return useful information to the caller (how many requests remain, when to retry).

---

## Requirements Clarification

**Core questions:**
- Is this a library (in-process) or a service (network call)? → Library. In-process, no network.
- Does each endpoint need its own algorithm and limits? → Yes. Per-endpoint config.
- What if an endpoint has no specific config? → Use a default limiter.
- What does "rejected" look like? → Return a result object (allowed=false + retryAfterMs). Don't throw exceptions.
- What's the unit of limiting? → Per client ID (user ID or API key) per endpoint.
- Thread safety? → Yes, multiple threads share one RateLimiter instance.

**Final Requirements:**
```
1. allow(clientId, endpoint) → RateLimitResult
   - allowed: boolean
   - remaining: int (requests left in current window)
   - retryAfterMs: long? (null if allowed; ms until next allowed request if rejected)
2. Per-endpoint algorithm selection (Token Bucket or Sliding Window Log)
3. Default limiter for unconfigured endpoints
4. Thread-safe across concurrent callers
5. No background threads (lazy refill on each allow() call)
```

---

## Core Entities

Candidate nouns: limiter, request, client, endpoint, token, bucket, window, log, config.

**Request** — not an entity. Just the act of calling `allow()`.

**TokenBucket** — state for one client under the token bucket algorithm: current tokens + last refill time. **Value object.**

**RequestLog** — state for one client under the sliding window log algorithm: a queue of timestamps. **Value object.**

**RateLimitResult** — the result of `allow()`. Immutable. **Value object.**

**Limiter** — the interface. One implementation per algorithm. **Interface + implementations.**

**LimiterFactory** — reads config maps and produces the right Limiter. **Factory.**

**RateLimiter** — the orchestrator. Routes `allow()` to the correct per-endpoint Limiter. **Entity.**

| Entity | Responsibility |
|--------|---------------|
| **RateLimiter** | Routes allow() to the right Limiter by endpoint; falls back to default |
| **Limiter** | Per-algorithm allow() logic; owns per-key state |
| **LimiterFactory** | Creates the right Limiter from a config map |
| **RateLimitResult** | Immutable result: allowed, remaining, retryAfterMs |
| **TokenBucket** | Per-client state for Token Bucket algorithm |
| **RequestLog** | Per-client state for Sliding Window Log algorithm |

---

## Class Design

### RateLimiter

```
class RateLimiter:
    - limiters: Map<String, Limiter>   // endpoint → limiter
    - defaultLimiter: Limiter

    RateLimiter(endpointConfigs, defaultConfig):
        factory = LimiterFactory()
        for config in endpointConfigs:
            endpoint = config["endpoint"]
            limiters[endpoint] = factory.create(config)
        defaultLimiter = factory.create(defaultConfig)

    allow(clientId, endpoint) → RateLimitResult:
        limiter = limiters.getOrDefault(endpoint, defaultLimiter)
        return limiter.allow(clientId)
```

### Limiter Interface

```
interface Limiter:
    + allow(key) → RateLimitResult
```

`key` is typically `clientId` — it's what we're rate-limiting per. The endpoint is already baked into which Limiter instance is called.

### RateLimitResult

```
class RateLimitResult:
    - allowed: boolean
    - remaining: int
    - retryAfterMs: Long?     // null if allowed

    + RateLimitResult(allowed, remaining, retryAfterMs)
```

### LimiterFactory

```
class LimiterFactory:
    create(config) → Limiter:
        type = config["type"]
        if type == "token_bucket":
            return TokenBucketLimiter(
                capacity = config["capacity"],
                refillRatePerSecond = config["refillRatePerSecond"]
            )
        if type == "sliding_window":
            return SlidingWindowLogLimiter(
                maxRequests = config["maxRequests"],
                windowMs = config["windowMs"]
            )
        throw IllegalArgumentException("Unknown limiter type: " + type)
```

---

## Algorithm 1: Token Bucket

**Mental model:** A bucket holds tokens. Each request consumes one token. Tokens refill at a constant rate up to the bucket's capacity. The bucket never overflows above capacity.

```
class TokenBucketLimiter implements Limiter:
    - capacity: int
    - refillRatePerSecond: double
    - buckets: Map<String, TokenBucket>    // per-key, lazy-created

    allow(key):
        bucket = getOrCreateBucket(key)
        synchronized(bucket):
            now = currentTimeMillis()
            elapsed = now - bucket.lastRefillTime
            tokensToAdd = (elapsed * refillRatePerSecond) / 1000.0
            bucket.tokens = min(capacity, bucket.tokens + tokensToAdd)
            bucket.lastRefillTime = now

            if bucket.tokens >= 1.0:
                bucket.tokens -= 1.0
                return RateLimitResult(true, floor(bucket.tokens), null)
            else:
                tokensNeeded = 1.0 - bucket.tokens
                retryAfterMs = ceil((tokensNeeded * 1000.0) / refillRatePerSecond)
                return RateLimitResult(false, 0, retryAfterMs)

    getOrCreateBucket(key):
        return buckets.computeIfAbsent(key, _ → TokenBucket(capacity, currentTimeMillis()))

class TokenBucket:
    - tokens: double          // double because fractional accumulation is valid
    - lastRefillTime: long
```

**Key design decisions:**

**Why `tokens: double` not `int`?** Refill is proportional to elapsed time. If 500ms passes at a rate of 2 tokens/sec, that's exactly 1.0 token — fine. But if 250ms passes, that's 0.5 tokens. Storing as integer would truncate to 0, losing the partial accumulation. The next 250ms would also add 0 and the user would never get a token despite two 250ms windows having passed. Double preserves fractional accumulation correctly.

**Why no background refill thread?** Background threads add complexity: they need lifecycle management (start, stop, handle errors), and they'd wake up constantly even when no requests are coming. Lazy refill — compute how many tokens to add when a request arrives based on elapsed time — is equivalent mathematically and avoids background threads entirely.

**Why synchronize on `bucket` not on `this`?** Two threads with different client IDs should not block each other. Locking on `this` (the limiter instance) would serialize all clients. Locking on the per-key bucket lets clients proceed in parallel unless they share the same key.

**Why `computeIfAbsent` instead of `get` + `put`?** Under concurrent access, two threads might both see `get() → null` and both create a new bucket, losing one. `computeIfAbsent` is atomic on `ConcurrentHashMap` — exactly one thread creates the bucket, the other gets the same one back.

**RetryAfterMs formula:** If bucket has 0.3 tokens and capacity is 1, we need 0.7 more tokens. At 2 tokens/sec, that takes 0.7/2 = 0.35 seconds = 350ms.

---

## Algorithm 2: Sliding Window Log

**Mental model:** Keep a log of every request's timestamp. To check if a new request is allowed, count how many timestamps fall within the last `windowMs`. Evict timestamps that have aged out before counting.

```
class SlidingWindowLogLimiter implements Limiter:
    - maxRequests: int
    - windowMs: long
    - logs: Map<String, RequestLog>    // per-key, lazy-created

    allow(key):
        log = getOrCreateLog(key)
        synchronized(log):
            now = currentTimeMillis()
            cutoff = now - windowMs

            // Evict timestamps older than the window
            while !log.timestamps.isEmpty() && log.timestamps.peek() < cutoff:
                log.timestamps.poll()

            if log.timestamps.size() < maxRequests:
                log.timestamps.add(now)
                return RateLimitResult(true, maxRequests - log.timestamps.size(), null)
            else:
                oldestTimestamp = log.timestamps.peek()
                retryAfterMs = (oldestTimestamp + windowMs) - now
                return RateLimitResult(false, 0, retryAfterMs)

    getOrCreateLog(key):
        return logs.computeIfAbsent(key, _ → RequestLog())

class RequestLog:
    - timestamps: Queue<long>     // LinkedList: O(1) front removal + back addition
```

**Key design decisions:**

**Why `Queue<long>` not `List<long>`?** Eviction removes from the front (oldest timestamps first). List removal from the front is O(n) (shifts all elements). Queue's `poll()` is O(1). Addition to the back is also O(1) for LinkedList. This makes each `allow()` call amortized O(k) where k is the number of evicted timestamps — eviction work is proportional to the cleanup done, not total history size.

**RetryAfterMs formula:** The oldest timestamp in the log is the one that will "age out" first. When it falls outside the window (`oldestTimestamp + windowMs < now`), we get one free slot. `retryAfterMs = (oldestTimestamp + windowMs) - now` tells us exactly when that happens.

**Memory concern:** Unlike Token Bucket (fixed size), Sliding Window Log grows proportional to the number of requests in the window. For `maxRequests=1000`, the log holds up to 1000 entries per client. Eviction during `allow()` bounds it, but a burst of 1000 requests stays in memory for `windowMs` milliseconds.

---

## Token Bucket vs. Sliding Window Log

| Property | Token Bucket | Sliding Window Log |
|----------|-------------|-------------------|
| Memory per client | O(1): just two numbers | O(maxRequests): a queue of timestamps |
| Burst allowance | Yes: unused tokens accumulate | No: window is absolute |
| Precision | Approximation (refill at check time) | Exact (tracks every request) |
| Best for | APIs where bursts are okay | APIs needing strict per-window limits |
| retryAfterMs | Time to earn 1 token at refill rate | When oldest entry exits the window |

A rate-limited API that charges per message might prefer Sliding Window Log (exact billing). A gaming API where players naturally burst at login might prefer Token Bucket (absorbs the burst, then steady state).

---

## Thread Safety Deep Dive

Three layers of concurrent access:

**Layer 1: Creating state for a new key.** Two threads see the same key for the first time simultaneously. `ConcurrentHashMap.computeIfAbsent()` guarantees exactly one bucket/log is created — both threads get the same object back.

**Layer 2: Reading and updating per-key state.** Once the bucket/log exists, all operations on it (refill, check, decrement) must be atomic together. A thread checking `tokens >= 1` then decrementing is a race window — another thread could drain the last token between check and decrement. `synchronized(bucket)` makes check + decrement one atomic step.

**Layer 3: Multiple keys in parallel.** By locking on the per-key object (not `this`), threads for different clients execute concurrently. Only threads sharing the same client ID contend.

```
// In practice, use ConcurrentHashMap for the outer map:
private final ConcurrentHashMap<String, TokenBucket> buckets = new ConcurrentHashMap<>();

// And synchronized on the per-key object for the inner ops:
synchronized(bucket):
    // refill + check + decrement
```

---

## Memory Management

Per-key state grows unboundedly if clients keep cycling (new client IDs per request). Two approaches:

**LRU eviction:** Wrap `buckets` in an LRU cache with a max size. Least-recently-seen clients are evicted first. When evicted, a new token bucket (full) is created on their next request — effectively resetting their limit, which is acceptable since long-inactive clients are unlikely to be abusers.

**Background scanner:** A periodic thread scans `buckets` and removes entries not accessed in the last N minutes. Simpler than LRU but adds background thread complexity. Use a `lastAccessTime` field on each bucket for the scanner to check.

---

## Extensibility: Fixed Window Counter

```
class FixedWindowCounterLimiter implements Limiter:
    - maxRequests: int
    - windowMs: long
    - counters: Map<String, WindowCounter>

    allow(key):
        counter = getOrCreateCounter(key)
        synchronized(counter):
            now = currentTimeMillis()
            windowStart = (now / windowMs) * windowMs   // floor to window boundary
            if counter.windowStart != windowStart:
                counter.windowStart = windowStart
                counter.count = 0
            if counter.count < maxRequests:
                counter.count++
                return RateLimitResult(true, maxRequests - counter.count, null)
            else:
                retryAfterMs = (windowStart + windowMs) - now
                return RateLimitResult(false, 0, retryAfterMs)
```

Fixed Window is simpler but has a boundary problem: a burst of `maxRequests` at 11:59:59 and another `maxRequests` at 12:00:01 are both within limit, but the two-second window between them sees 2× the intended traffic. Sliding Window Log doesn't have this problem.

---

## Verification Trace

Setup: Token Bucket, capacity=3, refillRate=1/sec. Client "alice".

**Three requests in quick succession (t=0, t=10ms, t=20ms):**
```
allow("alice") at t=0:
  bucket = new TokenBucket(tokens=3, lastRefill=0)
  elapsed=0, tokensToAdd=0, tokens=3
  tokens(3) >= 1 → tokens=2
  return RateLimitResult(allowed=true, remaining=2, retryAfterMs=null) ✓

allow("alice") at t=10:
  elapsed=10ms, tokensToAdd=10*1/1000=0.01, tokens=min(3, 2+0.01)=2.01
  tokens(2.01) >= 1 → tokens=1.01
  return RateLimitResult(allowed=true, remaining=1, ...) ✓

allow("alice") at t=20:
  elapsed=10ms, tokensToAdd=0.01, tokens=min(3, 1.01+0.01)=1.02
  tokens(1.02) >= 1 → tokens=0.02
  return RateLimitResult(allowed=true, remaining=0, ...) ✓
```

**Fourth request at t=30ms (bucket nearly empty):**
```
allow("alice") at t=30:
  elapsed=10ms, tokensToAdd=0.01, tokens=0.03
  tokens(0.03) < 1 → rejected
  tokensNeeded = 1 - 0.03 = 0.97
  retryAfterMs = ceil(0.97 * 1000 / 1) = 970ms
  return RateLimitResult(allowed=false, remaining=0, retryAfterMs=970) ✓
```

---

## What Interviewers Look For

| Level | Bar |
|-------|-----|
| Junior | A single in-memory store mapping clientId to a count/token. Basic allow/reject. Handle one algorithm. |
| Mid-level | Limiter interface with multiple implementations. Factory pattern for algorithm selection. Per-key locking (not global). Lazy initialization with computeIfAbsent. RateLimitResult with retryAfterMs. |
| Senior | Justify double tokens (fractional accumulation). Queue<long> for O(1) sliding window eviction. Three-layer thread safety analysis. Memory management (LRU/scanner). Token Bucket vs. Sliding Window trade-offs articulated clearly. |

---

## Practice Questions

1. Token Bucket uses `synchronized(bucket)` not `synchronized(this)`. Explain exactly what race condition would occur without synchronization, using a specific example with two threads and initial tokens=1.
2. Sliding Window Log evicts timestamps at the start of `allow()`. Could you instead evict on a background thread? What correctness issue would that introduce?
3. A Token Bucket has capacity=10, refillRate=2/sec. The bucket is empty. What is `retryAfterMs`? Now derive the general formula.
4. `computeIfAbsent` is used instead of `get` + `putIfAbsent`. Could you use `putIfAbsent` correctly? Show the correct pattern and explain why `computeIfAbsent` is cleaner.
5. An endpoint gets requests from 10,000 distinct clientIds per hour. Token Bucket has 2 longs (16 bytes) per entry; Sliding Window Log has up to `maxRequests=100` timestamps (800 bytes per entry). Calculate the memory difference and explain when this matters.

## One-Line Summary

Rate Limiter's key insight is using the Strategy pattern (Limiter interface + Factory) for algorithm selection, lazy per-key initialization with ConcurrentHashMap.computeIfAbsent(), synchronized per-key (not globally) for concurrent correctness, and returning retryAfterMs so callers know exactly when to retry rather than backoff-and-guessing.
