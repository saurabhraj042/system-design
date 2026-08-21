# Logging Service

**Difficulty:** Medium | **Concepts:** N×M class explosion, Formatter/Sink separation, per-destination locking, async writes, hierarchical loggers

---

## Understanding the Problem

Design a logging library. Application code calls `logger.log(level, message)`. The logger routes log records to one or more destinations (console, file, network). Each destination may format records differently and filter by minimum log level. The system must be thread-safe and extensible to new formats and sinks.

---

## Requirements Clarification

**Core questions:**
- Who creates the Logger? → Application code; the library provides it.
- What log levels exist? → DEBUG < INFO < WARN < ERROR. Each destination has a minimum level.
- Can a single Logger write to multiple destinations? → Yes.
- Can different destinations use different formats? → Yes — that's the key design challenge.
- Do we need async logging? → Mention it as an extensibility concern; basic implementation is synchronous.
- What happens when a sink fails (disk full, network down)? → Swallow the error with a diagnostic to stderr; never crash the caller.

**Final Requirements:**
```
1. Logger.log(level, message) routes to all attached destinations
2. Each destination has a minimum level filter (drops records below it)
3. Each destination formats the record with its own Formatter
4. Each destination writes to its own Sink (console, file, etc.)
5. Thread-safe: multiple threads calling log() concurrently must not interleave output
6. Sink failures must not propagate to the caller
```

---

## Core Entities

Candidate nouns: logger, message, level, destination, format, sink, file, console, record.

**Message** — the raw string. Not an entity.

**Record** — a structured capture of one log event: timestamp, level, message, thread name. Immutable. **Entity (value object).**

**Logger** — the entry point. Holds a list of destinations. Routes records to each. **Entity.**

**Destination** — one output channel. Owns a Formatter and a Sink. Applies the level filter. **Entity.**

**Formatter** — converts a LogRecord to a formatted string (plain text, JSON). Pure function (no state). **Interface.**

**Sink** — writes a formatted string somewhere (console, file). Has state (file handle, connection). **Interface.**

| Entity | Responsibility |
|--------|---------------|
| **Logger** | Captures timestamp and thread, creates LogRecord, fans out to destinations |
| **Destination** | Filters by level, formats the record, writes to its sink with a lock |
| **Formatter** | Converts LogRecord → String (stateless) |
| **Sink** | Writes String to an output (stateful) |
| **LogRecord** | Immutable value: timestamp, level, message, threadName |

---

## Class Design

### Logger

```
class Logger:
    - destinations: List<Destination>
    - name: String

    + Logger(name, destinations)
    + log(level, message) → void
    + debug(message) → void     // convenience
    + info(message) → void
    + warn(message) → void
    + error(message) → void
```

### Destination

```
class Destination:
    - formatter: Formatter
    - minLevel: LogLevel
    - sink: Sink
    - lock: Lock

    + Destination(formatter, minLevel, sink)
    + write(record) → void
```

### Interfaces

```
interface Formatter:
    + format(record) → String

interface Sink:
    + write(formatted) → void
    + close() → void
```

### LogRecord

```
class LogRecord:
    - timestamp: long       // Unix milliseconds — immutable
    - level: LogLevel
    - message: String
    - threadName: String

    + LogRecord(timestamp, level, message, threadName)
    // getters only; no setters
```

### Implementations

```
class PlainTextFormatter implements Formatter:
    format(record):
        return record.timestamp + " [" + record.level + "] [" + record.threadName + "] " + record.message

class JsonFormatter implements Formatter:
    format(record):
        return jsonEncode({
            "timestamp": record.timestamp,
            "level": record.level,
            "thread": record.threadName,
            "message": record.message
        })

class ConsoleSink implements Sink:
    write(formatted): stdout.println(formatted)
    close(): // no-op

class FileSink implements Sink:
    - fileWriter: FileWriter

    FileSink(filePath):
        fileWriter = openFile(filePath, mode=APPEND)   // open once in constructor

    write(formatted):
        fileWriter.append(formatted + "\n")

    close():
        fileWriter.close()
```

### Enums

```
enum LogLevel:
    DEBUG = 0, INFO = 1, WARN = 2, ERROR = 3
    // ordinal comparison: record.level >= minLevel
```

---

## The N×M Problem

Naive approach: create one class per combination — `ConsoleJsonDestination`, `FilePlainTextDestination`, `ConsoleJsonDestination`, etc.

With 3 sink types and 3 format types, that's 9 classes. Add one format → 3 more classes. Add one sink → 3 more classes. The number of classes grows multiplicatively. This is the Bridge pattern problem.

**Solution: separate Formatter from Sink.** Each is an independent interface. A Destination composes one Formatter and one Sink. Adding a new format requires one class; adding a new sink requires one class. The number of classes grows additively, not multiplicatively.

```
// Before separation: 3 sinks × 3 formats = 9 classes
ConsoleJsonDestination, ConsolePlainTextDestination, ConsoleXmlDestination
FileJsonDestination,    FilePlainTextDestination,    FileXmlDestination
NetworkJsonDestination, NetworkPlainTextDestination, NetworkXmlDestination

// After separation: 3 + 3 = 6 classes
JsonFormatter, PlainTextFormatter, XmlFormatter    // 3 formatters
ConsoleSink, FileSink, NetworkSink                 // 3 sinks
// Any combination via Destination(formatter, sink)
```

---

## Implementation

### `Logger.log`

**Where should the timestamp be captured?** At the Logger, not inside `Destination.write()`. If each Destination captured its own timestamp, the same logical event would have slightly different timestamps across destinations (the first destination captures earlier than the last). Capturing once at `Logger.log()` entry gives all destinations the same timestamp for the same log call.

**Where should the thread name be captured?** Same reasoning: capture at Logger.log() on the calling thread's `Thread.currentThread().name`. If captured inside Destination.write(), it would reflect the destination's worker thread name (in async mode) — wrong.

```
log(level, message):
    if message == null: return      // silent no-op for null

    record = LogRecord(
        currentTimeMillis(),
        level,
        message,
        Thread.currentThread().name
    )
    for destination in destinations:
        destination.write(record)
```

### `Destination.write`

**Three design decisions in one method:**

1. **Filter before format.** Formatting is CPU work (string building, JSON serialization). Don't format a record you're going to drop.
2. **Format outside the lock.** Formatters are pure functions (stateless, no shared data). Formatting inside the lock would block other threads from writing to this destination while we're doing CPU work. Format first, then lock for the actual I/O.
3. **Swallow sink failures.** The caller should never see an exception because their log statement failed. Write a diagnostic to stderr and continue.

```
write(record):
    if record.level < minLevel: return          // drop quietly

    formatted = formatter.format(record)        // pure function, no lock needed

    lock.acquire()
    try:
        sink.write(formatted)
    catch Exception as e:
        stderr.write("logger: sink write failed: " + e.message)   // diagnostic only
    finally:
        lock.release()
```

**Why a per-destination lock, not a global lock?** A global lock would mean writing to the file destination blocks all threads from writing to the console destination too. Per-destination locks allow concurrent writes across independent destinations.

**Why lock at all inside Destination?** Multiple threads call `Logger.log()` concurrently. Each thread calls `destination.write()` on the same Destination. Without a lock, two threads would write to the same FileSink concurrently — their output would interleave mid-line. The lock ensures one `sink.write()` completes before the next starts.

### `FileSink` — Open Once

```
class FileSink implements Sink:
    FileSink(filePath):
        fileWriter = openFile(filePath, mode=APPEND)

    write(formatted):
        fileWriter.append(formatted + "\n")
```

**Why open the file in the constructor, not in `write()`?** Opening a file is expensive (syscall, file descriptor allocation). If you open on every write, you pay that cost on every log line. Opening once in the constructor amortizes the cost. The file handle stays open for the lifetime of the FileSink.

---

## Extensibility

### Async Logging

Synchronous logging blocks the caller's thread during I/O. For high-throughput paths, log calls should return immediately.

**Design:** Give each Destination a bounded `BlockingQueue<LogRecord>` and a background worker thread that drains it.

```
class Destination:
    - queue: BlockingQueue<LogRecord>    // bounded, e.g. capacity 1000
    - worker: Thread

    Destination(formatter, minLevel, sink):
        queue = LinkedBlockingQueue(capacity=1000)
        worker = Thread(() → {
            while !Thread.interrupted():
                record = queue.take()   // blocks when empty
                formatted = formatter.format(record)
                sink.write(formatted)
        })
        worker.setDaemon(true)
        worker.start()

    write(record):
        if record.level < minLevel: return
        try:
            queue.put(record)           // blocks if queue is full (backpressure)
        catch InterruptedException:
            Thread.currentThread().interrupt()
```

**Bounded queue for backpressure:** If the sink can't keep up (slow disk, network), the queue fills. `queue.put()` blocks the caller instead of letting the queue grow without bound — preventing OOM.

**Alternative backpressure strategies:**
- `queue.offer()` instead of `queue.put()`: drops records silently when full (prefer for non-critical logging)
- Increase capacity to absorb bursts
- Drain queue in batches (faster for file writes)

### Hierarchical Loggers

Loggers in a hierarchy: a child logger named "com.myapp.service" inherits from "com.myapp", which inherits from the root logger.

```
class LoggerFactory:
    - loggers: Map<String, Logger>

    getLogger(name):
        if loggers.containsKey(name): return loggers[name]
        parent = findParent(name)   // strip last component: "com.myapp.service" → "com.myapp"
        logger = Logger(name, destinations=[], parent=parent)
        loggers[name] = logger
        return logger

class Logger:
    - parent: Logger?        // null for root logger

    log(level, message):
        record = LogRecord(...)
        current = this
        while current != null:
            for destination in current.destinations:
                destination.write(record)
            current = current.parent
```

Child loggers propagate records up the hierarchy unless `propagate = false` is set. This allows a "com.myapp.database" logger to log to both its own file destination *and* the root console, without duplicating configuration.

---

## Verification Trace

Setup:
```
consoleDest = Destination(PlainTextFormatter(), INFO, ConsoleSink())
fileDest    = Destination(JsonFormatter(), DEBUG, FileSink("/var/log/app.log"))
logger = Logger("app", [consoleDest, fileDest])
```

**DEBUG message from Thread "worker-1":**
```
logger.log(DEBUG, "connection pooled"):
  record = LogRecord(1000000, DEBUG, "connection pooled", "worker-1")

  consoleDest.write(record):
    DEBUG < INFO → return (filtered, nothing written)

  fileDest.write(record):
    DEBUG >= DEBUG → proceed
    formatted = jsonEncode({timestamp:1000000, level:DEBUG, thread:"worker-1", message:"connection pooled"})
    lock.acquire()
    fileSink.write(formatted) → written to /var/log/app.log
    lock.release()
```

**Concurrent INFO messages from two threads:**
```
Thread A: logger.log(INFO, "request started")  → consoleDest and fileDest
Thread B: logger.log(INFO, "request started")  → consoleDest and fileDest

// consoleDest: Thread A and B race for the lock
// Thread A gets lock → writes "1000001 [INFO] [Thread-A] request started"
// Thread B blocks on lock.acquire()
// Thread A releases lock → Thread B acquires → writes its own line
// Output: two clean lines, never interleaved ✓
```

---

## What Interviewers Look For

| Level | Bar |
|-------|-----|
| Junior | Logger, Destination, a way to format, and a way to write. Basic level filtering. |
| Mid-level | Separate Formatter from Sink (Bridge pattern / N×M avoidance). Timestamp captured at Logger.log(). Per-destination lock. Sink failures caught and swallowed. FileSink opens file once in constructor. |
| Senior | Justify format-outside-lock (pure function). Async design with bounded BlockingQueue + backpressure. Hierarchical loggers with propagation. Trade-offs between offer() (drop) vs. put() (backpressure) for queue overflow. |

---

## Practice Questions

1. If `formatter.format(record)` were called inside the `lock.acquire()/release()` block, what problem would that cause? When would this actually matter?
2. Two threads simultaneously call `destination.write()` with different records. Walk through what happens if there's no lock — show exactly how output could become corrupted.
3. Why does `Logger.log()` not need its own lock, even when multiple threads call it concurrently?
4. `FileSink` opens its file in the constructor. What would happen if the file couldn't be opened (path doesn't exist, permissions denied)? Where should this error be handled, and should it propagate to the caller?
5. In async mode, the worker thread calls `queue.take()`, which blocks when the queue is empty. What happens to the worker thread if the application shuts down without interrupting it? How do daemon threads help?

## One-Line Summary

Logging Service's key insight is separating Formatter from Sink to avoid N×M class explosion, capturing timestamp and thread name at Logger.log() (not inside Destination), formatting outside the lock (pure function), and catching sink failures inside Destination to ensure log calls never throw.
