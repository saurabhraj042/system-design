# Managing Long-Running Tasks

## The Core Idea

Some operations take seconds or minutes — video encoding, bulk report generation, sending 50,000 emails. Holding an HTTP connection open while that work completes is a dead end: browsers time out, load balancers cut the connection, and users have no idea if anything is happening. The fix is to split every heavy operation into two phases: **accept immediately, process in the background, notify when done.**

## Mental Model / Analogy

Think of a dry cleaner. You drop off your shirts, they hand you a ticket number, and you leave. You don't stand at the counter for two hours watching them work. When your clothes are ready, they call you. Your HTTP server is the front desk — its job is to hand out ticket numbers, not to do the cleaning.

---

## How It Works

### The Architecture

Three components work together:

```
Client → Web Server → Job Queue → Worker Pool → Storage / DB
              ↑                           ↓
         returns job_id           updates job status
```

1. **Web server** validates the request, writes a `pending` job record to the database, pushes the job ID onto the queue, and immediately returns a `{ job_id: "..." }` response to the client.
2. **Job queue** holds work durably until a worker picks it up. If a worker crashes, the job stays in the queue.
3. **Workers** run in a loop: pull a job, do the actual heavy lifting, write results to storage, mark the job `completed` or `failed`.

The web server's response time drops from "however long the task takes" to milliseconds. Workers scale independently on hardware matched to their workload (GPU instances for video, high-memory machines for large data exports).

### The Job Lifecycle

```
pending → processing → completed
                     ↘ failed → (retry N times) → dead-letter queue
```

Track status in a jobs table:
```sql
-- Schema pattern (write this from memory, don't memorize column names)
CREATE TABLE jobs (
  id           UUID PRIMARY KEY,
  type         TEXT,           -- 'video_encode', 'pdf_export', etc.
  payload      JSONB,          -- parameters the worker needs
  status       TEXT DEFAULT 'pending',
  result_url   TEXT,           -- where to find the output when done
  attempt_count INT DEFAULT 0,
  created_at   TIMESTAMPTZ,
  updated_at   TIMESTAMPTZ
);
```

### Worker Loop (Concept)

```python
# Simplified — real implementations use queue client SDKs
while True:
    job = queue.dequeue(block=True)   # wait until something arrives
    mark_processing(job.id)
    try:
        result = perform_work(job)
        store_result(result)
        mark_completed(job.id, result.url)
    except Exception as e:
        mark_failed(job.id, str(e))
        if job.attempt_count < MAX_RETRIES:
            queue.requeue(job, delay=backoff(job.attempt_count))
        else:
            dead_letter_queue.push(job)
```

---

## Choosing a Queue

| Queue | Sweet spot | Watch out for |
|---|---|---|
| Redis + BullMQ | Startups, simple jobs, fast setup | Memory-first; hard crashes can lose jobs |
| AWS SQS | Managed, no ops burden | 1MB message size cap; cost at huge scale |
| RabbitMQ | Complex routing, enterprise workflows | Self-hosted; operational overhead |
| Kafka | High throughput, replay, fan-out, event streaming | Overkill for simple task queues |

In interviews, **default to Kafka** unless the system is clearly small-scale. It's the most commonly expected answer and covers the hard cases.

---

## Trade-offs

**What you gain:**
- Web servers respond in milliseconds regardless of task duration
- Workers and web servers scale independently on different hardware
- A worker crash doesn't affect the API — jobs stay in the queue
- Clear retry and failure isolation

**What you lose:**
- Eventual consistency — the result isn't ready when the API returns
- More moving parts to monitor (queue depth, worker health, job failure rate)
- You need status-check endpoints and notification plumbing
- New failure modes: queue fills up, poison messages, duplicate processing

---

## When to Spot This in an Interview

You should call it out **before** the interviewer asks. Triggers:

- Any operation mentioned that takes more than ~5 seconds (video transcode, PDF export, bulk email, ML inference)
- Math that doesn't add up synchronously ("10M images/day at 8 seconds each = way more processing time than web servers can absorb")
- Mixed hardware needs ("login requests and GPU video encoding shouldn't share the same boxes")
- Failure questions ("what if a server crashes mid-process?" → with async workers, another worker picks up the queued job)

Example response: *"Video encoding will take several minutes per file, so I won't block the HTTP response. The server validates the upload, pushes an encode job to Kafka, returns a job ID, and the client polls or receives a webhook when encoding finishes."*

---

## Common Gotchas & Deep Dives

### Worker Crashes Mid-Job
Workers send periodic **heartbeats** to the queue. If a heartbeat stops arriving within the visibility timeout (SQS), session timeout (Kafka), or heartbeat interval (RabbitMQ), the queue assumes the worker died and makes the job available again. Set the timeout long enough to avoid false alarms from GC pauses (~10–30 seconds is a good default) but short enough that crashes don't delay jobs for too long.

### Poison Messages (Repeated Failures)
A job that crashes every worker that touches it — bad input, a bug in the processing code — will loop forever and eventually starve healthy jobs. After N failures (typically 3–5), route the job to a **Dead Letter Queue (DLQ)** instead of retrying. Monitor DLQ depth — it's an early signal that something is systematically broken.

### Duplicate Submissions
A user clicks "Export" three times before the first response arrives. You now have 3 identical jobs. Use an **idempotency key** (hash of user ID + action + time window). Before queuing, check if a job with that key already exists and return the existing job ID if so. Also make the work itself idempotent — checking before sending an email rather than sending unconditionally.

### Queue Backpressure
During a traffic spike, jobs pile up faster than workers can drain them. Solutions:
- Set a maximum queue depth and return `503 Service Busy` when exceeded — better to reject early than to queue work that won't finish for hours
- Autoscale workers based on **queue depth**, not CPU (by the time CPU is high, the backlog is already large)

### Mixed Job Durations
Short jobs (5-second report) shouldn't wait behind long jobs (5-hour data export). Use **separate queues** for fast vs slow jobs, each with their own worker fleet tuned for that duration. If you can't predict duration upfront, start in the fast queue and move to the slow queue if a time limit is exceeded.

### Job Dependencies / Chains
When step B requires step A's output (fetch data → generate PDF → email it), have each worker enqueue the next step as the last thing it does before marking itself complete. Include a workflow ID and step name in the payload so any step can be retried independently. For complex branching workflows, use a purpose-built orchestrator like AWS Step Functions or Temporal.

---

## Key Trade-offs at a Glance

| Scenario | Recommendation |
|---|---|
| Task takes < 5 seconds, low volume | Keep it synchronous |
| Task takes > 5 seconds | Async workers |
| Different hardware needs than web tier | Async workers |
| Caller needs result immediately in the same request | Can't use async; reconsider the requirement |
| Jobs must be ordered per-user | Kafka partition by user ID |
| Jobs are idempotent and cheap | Optimistic retry without DLQ |

---

## Practice Questions

1. An image hosting site lets users bulk-upload 500 photos at once. Walk through the async architecture from upload to all photos being processed and available.
2. Your PDF export jobs take anywhere from 2 seconds to 3 hours. All jobs currently share one queue with 20 workers. What goes wrong and how do you fix it?
3. A worker picks up a video encoding job and crashes at 80% complete. What happens to the job, and what guarantees does your system make to the user?
4. How would you implement a "cancel job" feature? What challenges arise if the job is already being processed by a worker?
5. Explain the difference between a job failing because of a bug (deterministic failure) vs. a transient network error. How should your retry strategy differ between these two cases?

---

## One-Line Summary

Accept immediately, process in the background, notify when done — and use a job queue as the durable handoff between the two so neither side needs to know the other is running.
