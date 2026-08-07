# Multi-Step Processes

## The Core Idea

Some operations cannot fit inside a single database transaction — they span multiple services, involve third-party APIs, take days or weeks to complete, or require a human to take action before proceeding. Coordinating these long-running, multi-step flows across distributed systems is surprisingly hard. Machines crash mid-flight. Network calls fail and must be retried. Webhooks from external services arrive unpredictably. Steps that succeeded must sometimes be undone when a later step fails.

The challenge is maintaining progress and consistency across all of this unreliability without forcing everything to wait on a single server or a single lock.

## Mental Model / Analogy

Think of coordinating a catered dinner party. You need to: confirm the guest count, order food, rent tables and chairs, hire servers, and send invitations — in roughly that order, and with dependencies between steps. If the rental company cancels after you've already ordered food, you need to cancel the food order (compensate). If you write down each step in a shared planner rather than keeping it all in your head, anyone can pick up where you left off if you're unavailable. A wedding coordinator (orchestrator) tracks all steps and knows exactly what's outstanding; alternatively, each vendor proactively emails the next vendor in the chain when their part is done (choreography). The coordinator model is easier to reason about; the email chain is more flexible but harder to trace.

---

## The Problem in Detail

Consider an e-commerce order fulfillment flow:
1. Charge the customer's payment method
2. Reserve the items in inventory
3. Generate a shipping label
4. Alert the warehouse for pick and pack
5. Send a confirmation email
6. Wait for the carrier to scan the package at pickup

Each step calls a different service. Any step can fail. The order may sit at step 6 for three days. Meanwhile, the server handling this flow might restart. A webhook confirming carrier pickup arrives on a completely different server than the one that started the flow. How do you maintain progress reliably?

---

## Solutions (Ascending in Complexity)

### Option 1: Single Server (No Persistence)

Run all steps in sequence on one server. Simple, fast to implement.

**Why it breaks:** Server crashes lose all in-flight state. You have no record of which steps completed. On restart, you don't know whether to retry the payment charge (idempotent?) or risk double-charging. Webhook callbacks from external services arrive on a random server with no context about which workflow instance they belong to.

**When it's acceptable:** Internal tools, throwaway scripts, flows short enough to complete in milliseconds where a restart simply means the user retries.

---

### Option 2: Single Server with State Persisted to Database

Before each step, write the current progress to a database. On failure or restart, load the state from the database and continue from the last checkpoint.

```
DB row: { order_id, step: "awaiting_inventory", charged_at: ..., status: "in_progress" }
```

A background poller scans for rows stuck in intermediate states and resumes them.

**What this adds:** Crash recovery. Webhook routing (look up the order_id from the webhook payload and load its state). Visibility into in-progress workflows.

**What this still requires you to build by hand:**
- A poller with correct timing (how often? what if it runs twice simultaneously?)
- Distributed locking so two pollers don't advance the same order concurrently
- Compensation logic (if inventory reservation fails after payment was charged, refund the charge)
- Idempotency on each step (is it safe to call this API twice? does it return the same result?)

This approach works for simple flows. It gets painful fast as the number of steps, failure modes, and edge cases grows.

---

### Option 3: Saga Pattern

A saga is a sequence of local transactions, each paired with a compensating transaction that undoes its effect. If step K fails, the system executes compensating transactions for steps K-1 down to 1.

**Example compensation chain:**
```
Step 1: Charge payment         → Compensation: Issue refund
Step 2: Reserve inventory      → Compensation: Release reservation
Step 3: Generate shipping label → Compensation: Void label (if possible)
Step 4: [fails here]
→ Execute compensation 3, then 2, then 1
```

**Key properties:**
- Sagas accept a brief inconsistency window. While the saga is executing (between step 1 completing and step 4 failing), the system is in a partially committed state.
- Compensating transactions need their own retries and idempotency guarantees — they can also fail and must be retried reliably.
- Compensations are sometimes impossible (a refund takes days to process; the item has already shipped; the email was already sent).

**When sagas are the right tool:** Financial transactions and distributed workflows where occasional rollbacks are acceptable and compensations are well-defined. Payment systems are the classic case.

---

### Option 4: Choreography via Event Store

Instead of explicit coordination, each service reacts to events and emits new events when it completes its part.

**Architecture:**
- An event log (Kafka) holds all events in order
- Each service subscribes to the events it cares about
- When a service processes an event and succeeds, it emits a new event
- Kafka consumer groups track each service's read position (offset); if a service crashes and restarts, it resumes from where it left off

```
PaymentService subscribes to: OrderPlaced
  → emits: PaymentCharged

InventoryService subscribes to: PaymentCharged
  → emits: InventoryReserved

ShippingService subscribes to: InventoryReserved
  → emits: ShipmentCreated
```

**Benefits:** Fault-tolerant by default (Kafka replays events for any consumer that falls behind). Flexible (add a new subscriber to any event without changing existing services). No central coordinator.

**Drawbacks:** The flow is implicit — it lives in the relationships between event producers and consumers, not in one place. Adding new steps or changing the order of steps requires careful coordination across teams. Debugging a stuck workflow means tracing events across multiple topics and service logs. At high complexity, understanding "what is the current state of order 12345?" requires replaying all events for that order.

---

### Option 5: Workflow Orchestration (Temporal)

Temporal is a durable workflow system that makes long-running, multi-step processes look like ordinary code, while handling crashes, retries, and external events automatically.

**Two building blocks:**

**Workflow:** A regular function that describes the entire process step by step. Workflows must be deterministic — the same inputs always produce the same sequence of decisions. They do not call external services directly. Instead, they call Activities.

**Activity:** A function that performs exactly one side-effecting operation (charge the payment, call the inventory API, send an email). Activities are retried automatically on failure. They must be idempotent — calling them twice should produce the same result as calling them once.

**How durability works:** Temporal records every event (activity start, activity completion, signal received) to a persistent history log. If a worker crashes mid-execution, Temporal replays the history log to reconstruct the workflow's state at the crash point, then continues from there. The workflow function runs again from the beginning, but completed activities return their cached results from history instead of re-executing.

```python
# This function can run for days or weeks and survive worker crashes
@workflow.defn
class OrderFulfillmentWorkflow:
    @workflow.run
    async def run(self, order_id: str):
        await workflow.execute_activity(charge_payment, order_id)
        await workflow.execute_activity(reserve_inventory, order_id)
        await workflow.execute_activity(generate_label, order_id)
        
        # Wait up to 30 days for carrier pickup — no CPU consumed while waiting
        await workflow.wait_condition(lambda: self.carrier_confirmed, timeout=timedelta(days=30))
        
        await workflow.execute_activity(send_confirmation_email, order_id)
    
    @workflow.signal
    def carrier_pickup_confirmed(self):
        self.carrier_confirmed = True
```

**Signals:** External events (like a carrier webhook) arrive as signals to a running workflow. The workflow can `await` a signal and resume when it arrives — holding no CPU resources while waiting, regardless of how long the wait is.

**Continue-as-New:** Workflow history grows with every event. For workflows that loop indefinitely (a subscription renewal that repeats every month forever), the history log would grow without bound. `Continue-as-New` snapshots current state, closes the old history, and starts a fresh workflow execution that begins reading from the snapshot. The workflow appears to continue seamlessly.

**Workflow versioning:** When you need to change an in-flight workflow (update the sequence of steps while existing executions are still running), use the `patched()` API. The function checks which "patch" version is active and branches accordingly, so old executions follow old logic and new executions follow new logic without conflict.

**Infrastructure components:**
- **Temporal Server:** Manages workflow state, schedules activity executions, persists history
- **History Database:** Stores workflow execution history (PostgreSQL, Cassandra, or MySQL)
- **Worker Pools:** Your code running Workflow and Activity functions; Workers poll Temporal Server for tasks

---

### Option 6: Managed Workflow Services

When you want the benefits of orchestration without running your own Temporal cluster:

**AWS Step Functions:** Define workflows as JSON state machines (DAG of states). Serverless — no infrastructure to manage. Each step calls a Lambda function or other AWS service. Limits: max 1-year execution duration; max 256KB of state. Best for AWS-native workflows where everything is already in Lambda.

**Azure Durable Functions:** Microsoft's equivalent for Azure Functions. Durable orchestrator functions coordinate durable activity functions. Similar concepts to Temporal but within the Azure/Functions ecosystem.

**Apache Airflow:** DAG-based scheduler designed for data engineering pipelines. Each DAG is a Python file defining tasks and dependencies. Primarily for batch ETL, scheduled jobs, and data pipelines — not well-suited for event-driven or interactive workflows.

---

## Exactly-Once Execution

True exactly-once delivery is impossible over a distributed network — you can guarantee at-least-once delivery (retrying until you get a response) but not exactly-once (the response confirming success might get lost, causing an unnecessary retry that actually re-executes). The practical solution is:

1. Use **at-least-once delivery** for your activity invocations (retry until you receive success)
2. Make every Activity **idempotent** so that receiving the same call twice produces the same result as receiving it once

**Implementation pattern for idempotent Activities:**
```
On receive (activity_id, payload):
  existing = DB.lookup(activity_id)
  if existing.status == COMPLETED:
    return existing.result   # already done, return cached result
  if existing.status == IN_PROGRESS:
    // another worker is handling it; wait or return error
  mark IN_PROGRESS
  result = do_the_work(payload)
  mark COMPLETED, store result
  return result
```

The `activity_id` is a stable identifier (the Temporal activity execution ID, or a client-generated idempotency key). Duplicate calls return the stored result without re-executing.

---

## When to Use

- **Simple sequential flow, no external wait:** Option 2 (state in DB with poller) is often enough; don't over-engineer
- **Financial transactions and rollback requirements:** Saga pattern (Option 3)
- **Decoupled services, high scale, implicit flow is acceptable:** Choreography via Kafka (Option 4)
- **Complex flows, human-in-the-loop, long wait times, need full visibility:** Temporal (Option 5)
- **AWS-native, short-lived workflows (<1 year):** Step Functions (Option 6)
- **Batch/ETL data pipelines with scheduling:** Airflow (Option 6)

**Human-in-the-loop example:** An Uber driver acceptance loop. The system dispatches a ride request to the nearest driver, waits up to 15 seconds for acceptance, and if declined, moves to the next driver — repeating for however many drivers it takes. This is a loop that could run for minutes, waits on external human action, and needs to track state across many iterations. Temporal's `await signal` with a timeout per iteration is the natural fit.

---

## When NOT to Use

- **Simple async operations:** A background job that processes a queue of emails doesn't need orchestration; a queue + worker is sufficient
- **Synchronous, fast operations:** If the entire flow completes in under 100ms, keep it synchronous and in one service
- **High-frequency, low-value operations:** Analytics event processing, impression logging — the overhead of tracking per-event workflow state costs more than the events are worth

---

## Deep Dives

### Crash Recovery

When a worker handling a workflow step crashes mid-activity, how does execution resume without re-running steps that already completed?

**With Temporal:** The Temporal Server detects that the worker heartbeat stopped. It reschedules the activity on a different worker. That worker requests the workflow history, which shows which activities already completed and their results. The workflow function replays from the beginning, using cached results for completed steps, and re-executes only from the crashed point forward.

**Without Temporal (manual):** Load workflow state from database. Check which steps have a completed status. Resume from the first step that's not yet complete. Requires you to handle locking (prevent two workers from advancing the same workflow concurrently) and idempotency (the resumed step must be safe to retry).

### Handling State Size Limits

Temporal workflow history grows with every event. In rare cases (workflows that loop thousands of times), the history can become very large.

Two mitigations:
1. **Minimize Activity payloads:** Pass IDs, not full data objects. Have Activities fetch data from the source rather than receiving it in the payload.
2. **Continue-as-New:** At a known checkpoint (e.g., after 1,000 iterations), snapshot the essential state and start a new workflow execution from that snapshot. The previous history is archived.

### External Events (Signals)

A workflow waiting for a third-party webhook (carrier scanned package, bank confirmed settlement, user clicked an approval email) needs to receive that event reliably.

With Temporal, the webhook handler calls `workflow.signal(workflow_id, "carrier_pickup_confirmed")`. Temporal delivers the signal to the running workflow, which resumes from its `await` point. If the workflow is currently in the middle of another activity, Temporal queues the signal and delivers it when the workflow is ready.

The important detail: the webhook endpoint must look up the workflow ID from the incoming request (stored on the order record in your database) and call the Temporal signal API — it doesn't need to know the current state of the workflow or which worker is handling it.

---

## Key Trade-offs

| Approach | Complexity | Handles crashes? | Long waits? | Visibility | Best for |
|----------|-----------|-----------------|-------------|-----------|---------|
| Single server | Very low | No | No | None | Throwaway scripts |
| State in DB + poller | Low-medium | Yes (manual) | Yes | Limited | Simple linear flows |
| Saga | Medium | Partial | No | Limited | Financial rollbacks |
| Choreography (Kafka) | Medium | Yes | Yes | Hard | High-scale decoupled services |
| Temporal | High | Yes (automatic) | Yes | Excellent | Complex orchestration |
| Step Functions | Medium | Yes | Up to 1 year | Good | AWS-native |

## Common Gotchas

- **Non-idempotent Activities in Temporal:** If an Activity charges a credit card without checking if it's already been charged, a retry after a transient failure causes a double charge; every Activity must be idempotent
- **Storing full objects in workflow state:** Passing 1MB payloads through Temporal history causes the history to balloon; pass IDs and fetch data at Activity execution time
- **Forgetting compensation is also fallible:** Saga compensations can themselves fail; they need retry logic and idempotency too
- **Using Temporal for simple async tasks:** The overhead of maintaining workflow state is only justified for flows that are complex, long-running, or need human-in-the-loop steps
- **Not handling the "webhook on a different server" problem:** In manual implementations, external webhooks arrive on arbitrary servers; you need to look up workflow state from a shared store, not from in-memory state on one server

## Practice Questions

1. An e-commerce order flow charges a card, reserves inventory, and creates a shipment. The inventory step fails. Walk through a Saga-based rollback, including what happens if the compensation for the charge step also fails.
2. You're using Temporal for a loan approval workflow. A user applies, a human reviewer approves, and the loan is disbursed. The review can take up to 30 days. Walk through how you'd model this with Temporal, specifically how the human approval gets delivered to the workflow.
3. Compare choreography via Kafka to Temporal orchestration for a payment processing pipeline with 8 steps. When would you choose each? What changes as the system scales to 100k payments/day?
4. A Temporal workflow for user onboarding loops through 20 welcome emails over 30 days. After 6 months in production, you need to add a step between email 5 and 6, but 10,000 workflows are already mid-execution. Walk through a versioning strategy that doesn't break existing workflows.
5. Design the exactly-once execution strategy for an Activity that creates a Stripe payment intent. The Activity may be retried up to 3 times if a timeout occurs. How do you ensure the user isn't charged twice?

## One-Line Summary

Coordinate multi-step distributed processes by starting with state persisted to a database for simple flows, using Sagas when financial rollbacks are required, applying Kafka choreography when services are decoupled and flow is implicit, and reaching for Temporal when workflows are complex, long-running, or require human-in-the-loop steps with automatic crash recovery.
