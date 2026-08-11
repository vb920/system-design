# Reliable Multi-Step Processes and Durable Workflows

## Core Thesis

A multi-step process coordinates several actions that cannot be completed inside one short, local database transaction.

Examples include:

- Charging a payment, reserving inventory, and arranging shipment
- Matching a rider with a driver and waiting for acceptance
- Processing a loan application with human approval
- Running an AI-agent pipeline across tools and models
- Provisioning cloud infrastructure across several services
- Sending notifications through multiple channels
- Processing refunds, disputes, and account adjustments

The central challenge is not merely executing steps in order. A production system must also survive:

- Process crashes
- Machine restarts
- Deployments
- Network timeouts
- Duplicate callbacks
- Partial success
- External-service outages
- Human delays
- Retries
- Workflow changes
- Operations lasting hours, days, or months

The main design principle is:

> **Persist workflow progress outside any one process, make every retried effect safe, and model failure handling as part of the business flow rather than as scattered exception code.**

---

# 1. Why Multi-Step Processes Are Difficult

Consider an order-fulfillment process:

```text
Create order
    ↓
Authorize or charge payment
    ↓
Reserve inventory
    ↓
Create shipping label
    ↓
Wait for warehouse picking
    ↓
Hand package to carrier
    ↓
Send confirmation
```

Each step may involve a different system. Some steps are fast, while others wait on a person or external webhook.

Every boundary introduces uncertainty:

```text
Did the request reach the service?
Did the service complete the action?
Was the response lost?
Did the worker crash before recording success?
Can the action be repeated safely?
Should earlier work be undone?
```

## The Partial-Failure Problem

Suppose payment succeeds but inventory reservation fails.

The system has already changed one service:

```text
Payment service: customer charged
Inventory service: item unavailable
```

A normal local rollback cannot undo the payment because the payment was committed by another system.

The application now needs a business-level recovery action:

```text
Refund or void payment
```

This is the basic shape of a distributed saga.

---

# 2. Why a Single In-Memory Function Is Not Reliable

A first implementation may look like this:

```typescript
async function fulfillOrder(order: Order): Promise<void> {
  await chargePayment(order);
  await reserveInventory(order);
  await createShippingLabel(order);
  await sendConfirmation(order);
}
```

This code is easy to understand but has no durable memory.

Suppose the process crashes here:

```text
Payment completed
Inventory reserved
        ↓
Process crashes
        ↓
Shipping not started
```

After restart, application memory is gone. The new process cannot safely infer whether it should:

- Start from the beginning
- Continue with shipping
- Retry inventory reservation
- Refund the payment
- Do nothing

Starting from the beginning may duplicate payment or inventory reservations. Doing nothing leaves a permanently stuck order.

### Process-Memory Invariant

> **The correctness of a long-running business process must not depend on the memory of one application process.**

---

# 3. Callback Routing Is a Durable-Correlation Problem

External systems often complete work asynchronously.

For example:

```text
Application sends payment request
    ↓
Payment provider processes request
    ↓ several seconds or minutes
Payment provider sends webhook
```

The webhook may reach any API server behind the load balancer, not the server that initiated the operation.

Therefore, the system cannot rely on an in-memory callback or promise.

Every external operation needs a durable correlation identifier:

```text
workflow_id = order-842
payment_attempt_id = pay-attempt-17
provider_reference = provider-991
```

Webhook handling becomes:

```text
1. Authenticate webhook.
2. Deduplicate provider event.
3. Extract correlation identifier.
4. Persist callback result.
5. Notify or resume the correct workflow.
6. Return success to provider.
```

### Correlation Invariant

> **Every delayed callback must be mappable to one durable workflow instance and one logical operation.**

---

# 4. The Hand-Rolled State Machine

A system can persist its current step in a database:

```text
order_id | workflow_state       | payment_id | next_attempt_at
----------------------------------------------------------------
842      | INVENTORY_PENDING    | p-991      | 2026-08-11 12:00
```

After each successful step, the service updates this row.

Possible states include:

```text
CREATED
PAYMENT_PENDING
PAYMENT_COMPLETED
INVENTORY_PENDING
INVENTORY_RESERVED
SHIPPING_PENDING
COMPLETED
COMPENSATING
FAILED
```

This makes the process recoverable, but persistence alone is not enough.

The system also needs:

- A scanner that finds runnable or stale workflows
- A claim mechanism so only one worker continues a workflow
- Retry counters and schedules
- Timeout handling
- Callback correlation
- Compensation logic
- Idempotency for every step
- Dead-letter or manual-review handling
- Deployment-safe workflow versioning
- Operational tooling and audit history

At this point the application is implementing a workflow engine.

## When a Hand-Rolled State Machine Is Reasonable

It may still be appropriate when:

- The flow is very small.
- There are only a few states.
- All work is in one service and database.
- The process finishes quickly.
- Compensation is simple.
- The team can confidently operate the recovery loop.

Once waits, callbacks, branches, compensations, and repeated retries accumulate, a dedicated workflow abstraction usually becomes safer.

---

# 5. Local Transactions Do Not Span Independent Services

A local transaction can atomically update one transactional database:

```sql
BEGIN;

UPDATE orders
SET status = 'PAID'
WHERE order_id = :order_id;

INSERT INTO outbox (...)
VALUES (...);

COMMIT;
```

It cannot normally atomically include:

- A payment provider
- A warehouse service
- An email provider
- A different database
- A human approval
- A process that may take several days

## Why Not Use One Giant Distributed Transaction?

Distributed transaction protocols can coordinate compatible participants, but they are usually a poor fit for long-running business processes because:

- Participants may not implement the protocol.
- External vendors are outside your control.
- Locks cannot be held through human think time.
- Coordinator failure complicates availability.
- One slow participant delays the whole operation.
- Long-lived atomicity creates unacceptable coupling.

Multi-step workflows therefore usually use local commits plus compensating business actions rather than one global rollback.

---

# 6. The Saga Pattern

A **saga** is a sequence of local transactions coordinated as one larger business process.

Each forward action may have a corresponding compensation.

```text
Forward action                 Compensation
---------------------------------------------------------
Charge payment                 Refund or void payment
Reserve inventory              Release inventory
Create shipping label          Cancel label
Allocate driver                Release driver
Create cloud resource          Delete cloud resource
```

## Forward Path

```text
T1 → T2 → T3 → T4 → success
```

## Failure and Compensation

If `T3` fails:

```text
T1 → T2 → T3 fails
          ↓
        C2 → C1
```

Compensations usually run in reverse order because later steps often depend on earlier ones.

## Important Nuance

Compensation is not a database rollback.

A refund does not erase the historical fact that a charge occurred. It creates a new business event that counteracts the financial effect.

Likewise:

- A cancellation email cannot make a previous email unseen.
- A released inventory reservation may no longer be recoverable later.
- A shipped package may require a return rather than cancellation.
- Some actions are irreversible.

### Saga Invariant

> **Every committed step must have an explicitly defined success continuation, failure policy, and, where possible, compensating action.**

---

# 7. Design Compensations Carefully

Compensations are production operations and can fail like forward steps.

A refund may time out. Releasing inventory may return an error. Cancelling a label may be rejected after carrier pickup.

Therefore, compensations need:

- Stable idempotency keys
- Retry policies
- Timeouts
- Durable progress
- Alerting
- Manual-repair paths
- Clear terminal states

## Semantic Compensation

A compensation should restore an acceptable business state, not necessarily the exact previous physical state.

Example:

```text
Original action: capture payment
Compensation: issue refund
```

The final balance may be restored, but the payment history now contains both charge and refund.

## Pivot Transactions

Some saga steps are effectively irreversible or mark a business commitment point.

Before the pivot:

```text
Failure can usually trigger compensation.
```

After the pivot:

```text
The workflow usually moves forward through repair or alternative actions.
```

Example:

```text
Before package pickup → cancel label and refund
After package pickup  → continue shipment or create return workflow
```

This distinction helps avoid pretending every real-world action can be cleanly undone.

---

# 8. Saga Choreography

In choreography, services react to events without one central coordinator controlling the whole process.

Example:

```text
Order service emits OrderPlaced
    ↓
Payment worker emits PaymentAuthorized
    ↓
Inventory worker emits InventoryReserved
    ↓
Shipping worker emits ShipmentCreated
    ↓
Notification worker sends confirmation
```

Compensation can also be event-driven:

```text
Inventory worker emits InventoryReservationFailed
    ↓
Payment worker consumes failure event
    ↓
Payment worker issues refund
    ↓
Payment worker emits PaymentRefunded
```

## Advantages

- Services remain loosely coupled.
- Teams can own independent reactions.
- New consumers can subscribe without changing the producer.
- Durable logs provide replay and audit capabilities.
- Worker pools scale independently.

## Disadvantages

- The end-to-end flow is implicit.
- One workflow is spread across many event handlers.
- Event contracts become tightly important.
- Debugging a single business instance requires correlation across services.
- Cycles and accidental reactions are possible.
- Inserting or reordering steps may require coordinated changes.
- Compensation logic becomes distributed.

## Appropriate Use

Choreography is a good fit when:

- The flow has moderate complexity.
- Reactions are naturally independent.
- Services should remain organizationally decoupled.
- No single team needs central control of the entire sequence.
- Events are already first-class integration contracts.

### Choreography Principle

> **Choreography reduces direct service coupling, but it can increase cognitive coupling through an implicit event graph.**

---

# 9. Durable Event Logs

A durable log stores events in an ordered, replayable stream.

Conceptually:

```text
offset 101: OrderPlaced
102: PaymentRequested
103: PaymentAuthorized
104: InventoryReserved
105: ShipmentCreated
```

Workers process the log and commit their progress.

## Failure Recovery

```text
1. Worker receives event at offset 104.
2. Worker performs an action.
3. Worker crashes before committing offset.
4. Another worker receives offset 104 again.
```

The event may be processed more than once. Consumers must therefore be idempotent.

## Partitioning

Ordering is normally guaranteed only inside a partition.

Choose a key that matches the required ordering scope:

```text
order_id → all events for one order stay ordered
payment_id → all transitions for one payment stay ordered
```

## Retention

A durable log can retain events for a configured duration or, in some systems, indefinitely. Retention policy and storage strategy determine whether the log is merely a transport, a recovery buffer, or part of the permanent record.

---

# 10. Event-Driven Architecture Versus Event Sourcing

These concepts often use similar infrastructure but solve different problems.

## Event-Driven Architecture

Primary goal:

```text
Decouple producers and consumers through events
```

Events may be retained only long enough for delivery, replay, or operational recovery.

The authoritative current state may live in ordinary databases.

## Event Sourcing

Primary goal:

```text
Store state changes as the authoritative history
```

Current state is reconstructed by replaying events, often with snapshots for efficiency.

```text
Initial state
    + Event 1
    + Event 2
    + Event 3
    = Current state
```

## Relationship to Sagas

A saga can use events without event sourcing.

Event sourcing can record saga state, but it does not automatically provide:

- Retry scheduling
- Timers
- Compensation orchestration
- Human-task waiting
- Activity execution
- Workflow versioning

### Distinction

> **Event-driven architecture describes communication. Event sourcing describes how authoritative state is represented. A workflow or saga describes how a long-running process is coordinated.**

---

# 11. The Transactional Outbox

A service often needs to:

1. Commit local database state.
2. Publish an event.

Doing these separately creates a dual-write problem.

## Failure Window A

```text
Database commits
Process crashes before publishing
```

The state changes, but consumers never receive the event.

## Failure Window B

```text
Event publishes
Database transaction rolls back
```

Consumers react to a state change that is not real.

## Outbox Solution

Write business state and an outbox record in one local transaction:

```sql
BEGIN;

UPDATE orders
SET status = 'PLACED'
WHERE order_id = :order_id;

INSERT INTO outbox (
    event_id,
    aggregate_id,
    event_type,
    payload,
    published
)
VALUES (
    :event_id,
    :order_id,
    'OrderPlaced',
    :payload,
    FALSE
);

COMMIT;
```

A relay publishes pending outbox rows to the broker.

The relay may publish more than once, so downstream consumers still need deduplication.

## Inbox Pattern

A consumer can maintain a processed-message table:

```sql
BEGIN;

INSERT INTO processed_messages (consumer_name, event_id)
VALUES (:consumer_name, :event_id)
ON CONFLICT DO NOTHING;

-- Apply the business effect only if the insert succeeded.

COMMIT;
```

### Messaging Invariant

> **Persist the local state change and publication intent atomically, then tolerate at-least-once delivery with idempotent consumers.**

---

# 12. Saga Orchestration

In orchestration, one coordinator owns the workflow's control flow.

```text
Orchestrator
    ├── tells Payment service to authorize
    ├── tells Inventory service to reserve
    ├── waits for warehouse signal
    ├── tells Shipping service to create label
    └── initiates compensation on failure
```

The coordinator stores:

- Current workflow state
- Completed steps
- Pending commands
- Timer deadlines
- Retry attempts
- External signals
- Compensation progress
- Terminal outcome

## Advantages

- The complete sequence is visible in one place.
- Complex branching is easier to reason about.
- Compensation order is explicit.
- Operations teams can inspect one workflow instance.
- Central retry, timeout, and timer policies are possible.

## Disadvantages

- The orchestrator becomes an important dependency.
- Services may become coupled to workflow commands.
- Central workflow ownership may create an organizational bottleneck.
- A custom orchestrator is difficult to implement correctly.

## Appropriate Use

Orchestration is usually preferable when:

- The process has many steps and branches.
- Failure policy is complex.
- Human tasks or long timers are involved.
- Central visibility is important.
- The sequence changes frequently.
- The process has business-level compensation.

---

# 13. Workflow Engine Versus Ordinary Message Queue

A message queue transports work.

```text
Producer → queue → consumer
```

A workflow engine manages a durable state machine.

```text
State + history + timers + retries + signals + branching + compensation
```

## Use a Queue When

- There is one asynchronous step.
- A worker resizes an image.
- A worker sends an email.
- A job can be retried independently.
- There is no complex branching or durable waiting.

## Use a Workflow Engine When

- Several steps must be coordinated.
- Later behavior depends on earlier results.
- The process waits for callbacks or humans.
- Compensation is required.
- Progress must survive crashes and deployments.
- Operators need end-to-end visibility.

A workflow engine may internally use queues, but the abstractions are not equivalent.

---

# 14. Durable Execution

Durable execution lets developers write workflow logic in code while the runtime persists enough history to resume after failure.

Conceptually:

```typescript
async function orderWorkflow(order: Order): Promise<OrderResult> {
  const payment = await authorizePayment(order);

  if (!payment.approved) {
    return { status: 'PAYMENT_FAILED' };
  }

  const inventory = await reserveInventory(order);

  if (!inventory.reserved) {
    await refundPayment(order.id, payment.id);
    return { status: 'INVENTORY_FAILED' };
  }

  await createShipment(order);
  await sendConfirmation(order);
  return { status: 'COMPLETED' };
}
```

The code resembles ordinary orchestration, but the runtime does not treat it as one fragile process execution.

The runtime records workflow decisions and activity results durably.

---

# 15. Workflow and Activity Separation

Durable execution systems commonly distinguish between workflows and activities.

## Workflow

The workflow contains control logic:

- Ordering
- Branching
- Loops
- Timer decisions
- Compensation order
- Waiting for signals

Workflow code should generally be deterministic.

## Activity

An activity interacts with the outside world:

- Calling a payment provider
- Reading or writing a database
- Sending an email
- Invoking another service
- Generating a label
- Calling an AI model

Activities can fail, time out, and be retried.

### Separation Invariant

> **Deterministic decisions belong in workflow code; nondeterministic side effects belong in activities.**

---

# 16. Why Workflow Determinism Matters

Many durable execution systems recover workflow state through replay.

They execute workflow code again and supply previously recorded results whenever the code reaches completed operations.

Suppose history contains:

```text
WorkflowStarted
PaymentActivityScheduled
PaymentActivityCompleted(result = approved)
InventoryActivityScheduled
InventoryActivityCompleted(result = reserved)
```

After a worker crash, a new worker replays the workflow from the beginning.

During replay:

```text
authorizePayment() → recorded result returned
reserveInventory() → recorded result returned
createShipment()   → no recorded result, schedule it now
```

Completed activities are not intentionally executed again during ordinary replay. Their recorded outcomes are reused.

For this to work, workflow code must make the same decisions when given the same history.

## Nondeterministic Operations to Avoid Directly in Workflow Code

- Current wall-clock time from the operating system
- Unseeded random numbers
- Direct network calls
- Arbitrary database reads
- Thread scheduling dependencies
- Iteration over unstable unordered data

Workflow SDKs often provide deterministic alternatives for time, randomness, and timers.

### Replay Invariant

> **The same workflow code, inputs, and recorded history must produce the same sequence of workflow decisions.**

---

# 17. Activities Are Usually At-Least-Once Attempts

A workflow engine can encounter an ambiguous outcome:

```text
Activity calls payment provider
Payment provider completes charge
Worker crashes before reporting success
```

The engine cannot know whether the external effect happened. It may retry the activity.

Therefore, external activities should usually be designed for at-least-once attempts.

## Correct Framing

```text
Activity attempt count: one or more
Business effect count: ideally one
```

The business effect is made effectively once through idempotency at the system that owns the effect.

## Payment Example

```text
idempotency_key = order-842-payment
```

Every retry sends the same key to the payment provider.

If the provider already processed the charge, it returns the existing result rather than charging again.

## Local Database Effect

When the idempotency record and mutation share one transaction, the system can provide an exactly-once local database effect.

## External Effect Without Provider Idempotency

Exactly-once effect cannot generally be guaranteed. The workflow may need reconciliation or manual review for ambiguous outcomes.

### Activity Invariant

> **Every retryable activity must have a stable operation identity and a defined duplicate-handling strategy.**

---

# 18. Idempotency State Machine

A useful activity-execution record can contain:

```text
operation_id | status       | provider_reference | result
----------------------------------------------------------
op-17        | IN_PROGRESS  | NULL               | NULL
```

Possible states:

```text
NOT_STARTED → IN_PROGRESS → COMPLETED
                         ↘ FAILED_RETRYABLE
                         ↘ FAILED_FINAL
                         ↘ NEEDS_RECONCILIATION
```

## Important Limitation

Recording `IN_PROGRESS` before an external call does not close the dual-write gap.

If the worker crashes after the provider succeeds but before the local record becomes `COMPLETED`, the retry observes an ambiguous state.

Correct handling may include:

- Querying the provider by idempotency key
- Querying by merchant reference
- Waiting for a webhook
- Running reconciliation
- Escalating to human review

Do not blindly repeat irreversible actions from ambiguous states.

---

# 19. Workflow Engine Architecture

A simplified durable-execution deployment contains:

```text
Client
    ↓
API service starts workflow
    ↓
Workflow service or control plane
    ↓                     ↘
History store              Task queues
                               ↓
                      Workflow and activity workers
                               ↓
                        External services
```

## Control Plane or Workflow Service

Responsibilities:

- Persist workflow events
- Create workflow and activity tasks
- Track timers and deadlines
- Dispatch retries
- Accept external signals
- Coordinate worker failover
- Expose workflow visibility APIs

It normally does not execute application business code.

## History Store

Stores events such as:

```text
Workflow started
Timer created
Activity scheduled
Activity completed
Signal received
Compensation completed
Workflow finished
```

## Worker Pools

Workers host application code.

Workflow workers:

- Replay workflow history
- Compute the next deterministic decision

Activity workers:

- Execute side effects
- Report results or failures

Workers can be scaled independently by task type.

---

# 20. Crash Recovery Through Replay

Consider this execution:

```text
1. Payment activity completes.
2. Inventory activity completes.
3. Workflow worker crashes.
4. Shipping has not started.
```

Recovery:

```text
1. Another workflow worker receives a task.
2. It loads workflow history.
3. It re-executes workflow code from the beginning.
4. Recorded payment result is returned from history.
5. Recorded inventory result is returned from history.
6. Replay reaches shipping with no result in history.
7. The engine schedules shipping.
```

The workflow code reads as though it ran continuously, while execution may move across many machines.

The engine persists logical progress, not a live operating-system thread or stack for the entire duration.

---

# 21. Durable Timers and Long Waits

A workflow may need to wait for:

- A payment webhook
- A driver response
- Document signature
- Warehouse pickup
- Cooling-off period
- Subscription renewal date
- Retry after several hours

A durable timer does not require a thread to sleep for that duration.

```text
Workflow records timer deadline
    ↓
Worker is released
    ↓ days later
Timer becomes due
    ↓
Workflow task is scheduled
    ↓
Workflow resumes
```

The durable state lives in storage, not in worker memory.

### Waiting Invariant

> **A waiting workflow consumes durable metadata, not a continuously blocked application thread.**

---

# 22. External Signals

A signal is an external event delivered to a specific workflow instance.

Examples:

- Driver accepted ride
- Customer signed document
- Payment provider completed authorization
- Warehouse marked item picked
- Administrator approved request

## Signal Flow

```text
External system or webhook
    ↓
Signal API with workflow ID
    ↓
Workflow engine records signal
    ↓
Waiting workflow becomes runnable
    ↓
Worker replays and handles signal
```

## Conceptual Workflow

```typescript
async function documentWorkflow(documentId: string): Promise<void> {
  await sendForSignature(documentId);

  const signed = await waitForSignalOrTimeout(
    'documentSigned',
    '30 days'
  );

  if (!signed) {
    await sendReminder(documentId);

    const signedAfterReminder = await waitForSignalOrTimeout(
      'documentSigned',
      '7 days'
    );

    if (!signedAfterReminder) {
      await cancelDocument(documentId);
      return;
    }
  }

  await processSignature(documentId);
}
```

Signals should be durable and deduplicated where the source may retry delivery.

---

# 23. Polling Versus Signals

## Polling

```text
Workflow periodically asks external system for status
```

Advantages:

- Works when the external system has no callback mechanism.
- Simple integration model.

Disadvantages:

- Adds unnecessary requests.
- Increases latency up to the polling interval.
- Requires backoff and rate-limit handling.

## Signals or Webhooks

```text
External system notifies workflow when state changes
```

Advantages:

- Lower latency
- Less repeated traffic
- Natural for long waits

Disadvantages:

- Requires callback authentication
- Requires durable correlation
- Callbacks may be duplicated or reordered
- Lost callbacks may still require periodic reconciliation

A robust design often uses webhooks for fast progress and polling or reconciliation as a safety net.

---

# 24. Retry Policies

Not every failure should be retried in the same way.

## Retryable Failures

Examples:

- Network timeout
- Temporary service unavailability
- Rate limit
- Transient database error
- Worker crash

Possible policy:

```text
initial delay = 1 second
backoff coefficient = 2
maximum delay = 5 minutes
maximum attempts = 8
jitter = enabled
```

## Non-Retryable Failures

Examples:

- Invalid payment details
- Unsupported address
- Authorization denied
- Malformed request
- Business rule violation

These should move directly to an alternative branch, compensation, or terminal failure.

## Retry Exhaustion

After the retry budget is exhausted:

- Start compensation
- Move to manual review
- Open an operational ticket
- Notify the user
- Park in a dead-letter state

### Retry Invariant

> **Every activity must classify errors, bound retries, and define what happens after the retry budget is exhausted.**

---

# 25. Timeouts and Heartbeats

## Activity Timeout Types

Useful timeout concepts include:

- Schedule-to-start: maximum queue delay
- Start-to-close: maximum duration of one attempt
- Schedule-to-close: maximum total duration including retries
- Heartbeat timeout: maximum silence from a long-running activity

## Why Heartbeats Matter

A warehouse export or large AI job may run for an hour.

A heartbeat can report:

```text
Still alive
Current progress
Checkpoint identifier
Cancellation received
```

If heartbeats stop, the engine can consider the attempt lost and retry or reassign it.

Heartbeats do not make arbitrary external effects idempotent. They improve liveness detection and can preserve progress metadata.

---

# 26. Cancellation

Cancellation is a business operation, not merely a thread interruption.

A workflow may receive cancellation while:

- Payment is pending
- Inventory is reserved
- A package is being packed
- A driver is en route
- An AI task is generating output

The workflow should define cancellation behavior for each state.

Example:

```text
Before payment capture:
Cancel pending authorization.

After capture but before shipment:
Release inventory and refund.

After carrier pickup:
Create return or intercept process.
```

Some cleanup or compensation activities may need to run even when the main workflow is cancelled.

---

# 27. Managed State-Machine Workflows

Managed workflow services allow developers to describe a workflow as states and transitions.

Conceptual definition:

```json
{
  "StartAt": "AuthorizePayment",
  "States": {
    "AuthorizePayment": {
      "Type": "Task",
      "Next": "ReserveInventory",
      "Catch": [
        {
          "ErrorEquals": ["PaymentDeclined"],
          "Next": "OrderFailed"
        }
      ]
    },
    "ReserveInventory": {
      "Type": "Task",
      "Next": "CreateShipment",
      "Catch": [
        {
          "ErrorEquals": ["InventoryUnavailable"],
          "Next": "RefundPayment"
        }
      ]
    }
  }
}
```

The real definitions can be written directly or generated using infrastructure-as-code libraries.

## Strengths

- Managed operational control plane
- Built-in state visualization
- Native cloud-service integrations
- Explicit state transitions
- Built-in retries, catches, waits, and parallel branches

## Limitations

- Logic must fit the state-machine model.
- Large workflows can become verbose.
- Service-specific limits apply to duration, payload, and history.
- Portability may be reduced.
- Complex reusable programming abstractions may be less natural.

---

# 28. Code-Driven Durable Execution Versus Declarative Workflows

## Code-Driven Durable Execution

Strengths:

- Natural loops and branching
- Strong language tooling
- Easier expression of complex domain logic
- Workflow flow remains close to application code

Costs:

- Determinism rules must be understood.
- Replay-safe code changes require discipline.
- The platform may need to be operated or purchased.
- History growth must be managed.

## Declarative State Machine

Strengths:

- Clear visual representation
- Managed infrastructure
- Explicit state transitions
- Good integration with cloud services

Costs:

- Logic may become verbose.
- Highly dynamic algorithms can be awkward.
- Provider limits and pricing shape design.
- State-machine definitions may be less ergonomic for complicated code.

The correct interview choice depends more on fit and understanding than on brand preference.

---

# 29. Workflow Versioning

Workflows may run for weeks while code is deployed repeatedly.

Suppose workflow version 1 has:

```text
Payment → Inventory → Shipping
```

Version 2 adds:

```text
Payment → Compliance Check → Inventory → Shipping
```

An existing workflow may already have history created under version 1. Replaying it against incompatible version 2 logic can violate determinism or change historical decisions.

## Strategy 1: Keep Old Versions Running

```text
Existing executions → workflow v1 workers
New executions      → workflow v2 workers
```

Advantages:

- Easy to reason about
- Low migration risk

Disadvantages:

- Multiple versions remain deployed
- Old workflows may live for a long time
- Urgent policy changes may not apply to existing executions

## Strategy 2: Deterministic Patching

Workflow code records or derives a version marker so replay chooses the historical path consistently.

Conceptual example:

```typescript
if (workflowPatchEnabled('add-compliance-check')) {
  await runComplianceCheck(application);
}

await reserveInventory(application);
```

The patch decision becomes part of workflow history or is otherwise replay-safe.

## Strategy 3: Explicit Migration

For selected workflows:

1. Pause or inspect the existing execution.
2. Transform state to a new schema.
3. Start or continue under a new definition.
4. Preserve correlation and audit links.

This provides control but adds significant operational complexity.

### Versioning Invariant

> **A workflow must make the same historical decisions during replay, even after new code is deployed.**

---

# 30. Workflow History Growth

Durable execution records many events:

- Activity scheduling and completion
- Timers
- Signals
- Retries
- Child workflows
- Branch decisions

A workflow that loops for months can accumulate a large history.

Consequences include:

- Slower replay
- Larger storage cost
- Service history limits
- Increased worker startup latency

## Reduce Payload Size

Prefer identifiers over large objects:

```text
Pass order_id
instead of
passing the entire order, product catalog, and documents
```

Store large documents in a database or object store.

Be careful that external reads during replay belong in activities, not nondeterministic workflow code.

## Continue as New

A long-running workflow can periodically start a fresh run with compact current state.

```text
Old run with long history
    ↓ summarize current durable state
New run with empty history and summarized input
```

The workflow continues logically while history size resets.

## Child Workflows

Break large processes into independently managed subflows:

```text
Order workflow
    ├── Payment child workflow
    ├── Fulfillment child workflow
    └── Returns child workflow
```

This can improve ownership and history management, but it also adds boundaries and failure semantics that must be understood.

---

# 31. Workflow Data and Sensitive Information

Workflow history may be retained for a long time and exposed through operational tooling.

Avoid storing unnecessary:

- Payment-card data
- Authentication secrets
- Personal documents
- Large model prompts or outputs
- Raw medical or financial information

Prefer:

- Stable references
- Encrypted payloads
- Tokenized provider identifiers
- Redacted operational metadata
- Separate data stores with appropriate retention controls

Workflow histories are operational records and must be included in privacy, deletion, and compliance designs.

---

# 32. Observability

A production workflow system should answer:

- Which step is running?
- Which step failed?
- How many retries occurred?
- What is the next retry time?
- Which compensation is pending?
- Which external callback is missing?
- Which workflows are stuck?
- Which version of workflow code is executing?
- What business identifiers correlate with this workflow?

## Useful Identifiers

```text
workflow_id
workflow_run_id
order_id
customer_id
activity_id
activity_attempt
idempotency_key
provider_reference
trace_id
```

## Useful Metrics

- Workflow completion latency
- Workflow failure rate
- Workflows by state
- Activity retry rate
- Retry exhaustion count
- Compensation count and failure rate
- Queue depth
- Worker task latency
- Signal delivery latency
- Stuck workflow age
- History size
- Manual-review backlog

## Audit History Versus Operational View

A raw event history is useful but not always operator-friendly.

Build searchable views that summarize:

```text
Current state
Last successful step
Current blocker
Next action
Compensation status
Business owner
```

---

# 33. Manual Intervention

Some workflows cannot complete automatically.

Examples:

- Provider reports an ambiguous payment state.
- Refund repeatedly fails.
- Inventory and warehouse records disagree.
- Compliance requires human review.
- A shipment has entered an irreversible state.

A manual-review state should be explicit:

```text
NEEDS_MANUAL_REVIEW
```

Operations should be able to:

- Inspect complete workflow context
- Re-run a safe activity
- Provide missing data
- Record a business decision
- Trigger compensation
- Mark an exceptional terminal outcome

Manual actions should themselves be authenticated, authorized, audited, and idempotent.

---

# 34. Complete Order Workflow Example

## State Model

```text
ORDER_CREATED
PAYMENT_PENDING
PAYMENT_AUTHORIZED
INVENTORY_PENDING
INVENTORY_RESERVED
FULFILLMENT_PENDING
SHIPPED
COMPLETED
COMPENSATING
REFUNDED
FAILED
NEEDS_MANUAL_REVIEW
```

## Happy Path

```text
1. Create order with stable order ID.
2. Authorize payment using order-scoped idempotency key.
3. Reserve inventory using reservation ID.
4. Wait for warehouse pickup signal.
5. Create shipment using shipment idempotency key.
6. Capture payment at the chosen business point.
7. Send confirmation.
8. Mark workflow completed.
```

## Inventory Failure

```text
1. Payment authorization succeeds.
2. Inventory reservation fails permanently.
3. Void authorization or refund captured payment.
4. Mark order failed.
5. Notify customer.
```

## Shipping Failure

```text
1. Payment succeeds.
2. Inventory is reserved.
3. Label creation fails after retries.
4. Release inventory.
5. Refund or void payment.
6. Mark order failed or manual review.
```

## Crash After Payment

```text
1. Payment provider applies operation.
2. Activity worker crashes before reporting success.
3. Engine retries activity.
4. Same payment idempotency key is used.
5. Provider returns original operation result.
6. Workflow continues without double charge.
```

## Lost Callback

```text
1. Provider processes request.
2. Webhook is lost.
3. Workflow timer expires.
4. Reconciliation activity queries provider by reference.
5. Workflow records the authoritative result.
```

---

# 35. Human-in-the-Loop Example

Consider driver matching:

```text
1. Receive ride request.
2. Find candidate drivers.
3. Offer ride to first driver.
4. Wait ten seconds for response.
5. If rejected or timed out, try next driver.
6. If accepted, notify rider.
7. If all candidates fail, expand search or fail request.
```

This is naturally a workflow because it contains:

- A loop
- External signals
- Durable timers
- Timeouts
- Cancellation
- Potentially long waits
- State that must survive worker restarts

Conceptual flow:

```typescript
async function matchRide(ride: Ride): Promise<MatchResult> {
  for (const driver of await findCandidates(ride)) {
    await proposeRide(driver.id, ride.id);

    const response = await waitForDriverResponse(
      ride.id,
      driver.id,
      '10 seconds'
    );

    if (response === 'ACCEPTED') {
      await assignDriver(ride.id, driver.id);
      await notifyRider(ride.id, driver.id);
      return { matched: true, driverId: driver.id };
    }
  }

  return { matched: false };
}
```

The try-next-driver loop must remain deterministic, and external searches or calls belong in activities.

---

# 36. AI and Agent Workflows

Agentic systems often have multi-step, failure-prone pipelines:

```text
Receive user goal
    ↓
Plan tasks
    ↓
Call retrieval tool
    ↓
Call external API
    ↓
Wait for approval
    ↓
Execute action
    ↓
Validate result
    ↓
Retry, re-plan, or compensate
```

Workflow concerns include:

- Model calls are nondeterministic.
- Tool calls may have side effects.
- Human approval may take days.
- Provider rate limits require durable backoff.
- Replaying model output may produce a different answer.
- Large prompts and outputs can inflate workflow history.
- An agent may loop indefinitely without budgets.

## Recommended Pattern

- Keep model calls in activities.
- Record chosen outputs for replay.
- Assign stable IDs to every side-effecting tool call.
- Bound iterations, cost, tokens, and wall-clock time.
- Require approval before high-impact actions.
- Store large artifacts externally and pass references.
- Define compensation or repair actions for tools.

---

# 37. Exactly-Once Claims

Be precise when discussing exactly once.

## Message Delivery

Across a network, delivery may be repeated or uncertain.

## Activity Attempts

An activity may be attempted more than once after a crash or timeout.

## Local Database Effect

An idempotency row and business mutation can be committed in the same transaction, producing one committed local effect.

## External Effect

An external effect is effectively once only when the system owning that effect supports:

- Idempotency keys
- Conditional creation
- Unique operation identifiers
- Query by operation reference
- Deduplication

Otherwise, ambiguous outcomes require reconciliation.

### Correct Interview Wording

> **The workflow provides durable at-least-once attempts. Stable idempotency keys and transactional deduplication make the logical business effect occur once where the downstream system supports it.**

---

# 38. Failure Matrix

## Worker Crashes Before Activity Starts

```text
Effect: none
Recovery: task is assigned again
```

## Worker Crashes During External Call

```text
Effect: uncertain
Recovery: retry with same idempotency key or reconcile
```

## Worker Completes Activity but Cannot Report Result

```text
Effect: probably happened
Recovery: activity may be retried; idempotency required
```

## Workflow Worker Crashes

```text
Effect: workflow decisions are recovered from persisted history
Recovery: another worker replays workflow
```

## Control Plane Temporarily Unavailable

```text
Effect: workflow progress pauses
Recovery: workers and control plane resume when service returns
```

## Duplicate Webhook

```text
Effect: callback may be processed repeatedly
Recovery: deduplicate by provider event ID and operation ID
```

## Callback Arrives Before Workflow Waits

```text
Effect: race between signal and wait registration
Recovery: durable signal recording allows later replay to observe it
```

## Compensation Fails

```text
Effect: workflow remains partially compensated
Recovery: retry, reconcile, or move to manual review
```

## Deployment Changes Workflow Logic

```text
Effect: replay may diverge
Recovery: versioning, patch markers, or pinned definitions
```

---

# 39. Choosing the Right Approach

## Use a Synchronous Function When

- All work is fast.
- It fits within one request.
- One local transaction can protect the state.
- Failure recovery is straightforward.

## Use a Queue When

- There is one independent asynchronous job.
- The job can be retried without a larger state machine.

## Use Event Choreography When

- Several independent services react to domain events.
- The flow has moderate complexity.
- Loose organizational coupling is important.
- The implicit event graph remains understandable.

## Use Workflow Orchestration When

- The process has many dependent steps.
- It waits for callbacks, timers, or humans.
- Compensation is important.
- Central visibility and control are valuable.
- The process lasts longer than one server execution.

## Use Event Sourcing When

- The event history itself is the authoritative record.
- State reconstruction and historical queries justify the model.

Do not choose event sourcing merely because a saga uses events.

---

# 40. When Not to Use a Workflow Engine

Avoid workflow infrastructure for:

- Ordinary CRUD requests
- One database transaction
- A single fire-and-forget job
- Very high-frequency, low-value operations
- Latency-critical synchronous request paths that cannot tolerate workflow round trips
- Simple polling that already satisfies the requirement

Workflow engines add:

- Control-plane round trips
- History writes
- Operational dependencies
- New programming constraints
- Platform-specific concepts
- Cost per transition or execution

The extra machinery is justified only when it eliminates greater complexity in failure recovery, long waits, compensation, or observability.

---

# 41. Interview Recognition Signals

A workflow or saga is likely appropriate when the prompt contains phrases such as:

- “If step C fails, undo A and B.”
- “The process may take several days.”
- “Wait for a webhook.”
- “Wait for a person to approve.”
- “Retry without charging twice.”
- “Resume after a server crash.”
- “Track where each order is stuck.”
- “Coordinate several internal and external services.”
- “Some steps are irreversible.”
- “The business process has many states and branches.”

A strong candidate identifies these requirements before introducing a specific product.

---

# 42. Interview Design Process

## Step 1: Define the Business States

Write the state machine before discussing technology.

```text
PENDING_PAYMENT
PAYMENT_AUTHORIZED
INVENTORY_RESERVED
SHIPPING
COMPLETED
COMPENSATING
FAILED
```

## Step 2: Classify Every Step

For each step, specify:

- Owner service
- Input and output
- Timeout
- Retryable errors
- Non-retryable errors
- Idempotency key
- Compensation
- Irreversibility point

## Step 3: Identify Waits

Find:

- Webhooks
- Human approvals
- Scheduled dates
- External job completion
- Retry delays

Use durable timers and signals rather than blocked threads.

## Step 4: Choose Coordination Style

```text
Moderate independent reactions → choreography
Complex central business flow → orchestration
```

## Step 5: Define Durable Progress

Explain where workflow state or history is persisted and how another worker resumes after failure.

## Step 6: Define Delivery Semantics

Assume retries and duplicates. Explain idempotency, deduplication, and ambiguous-outcome reconciliation.

## Step 7: Define Compensation

Specify reverse actions, retry policy, and manual escape hatch.

## Step 8: Define Versioning

Explain how in-flight workflows survive code changes.

## Step 9: Define Operations

Discuss tracing, search, stuck-workflow detection, metrics, and manual repair.

---

# 43. Interview Answer Template

> This process spans several services and may outlive a single request, so I would model it as a durable saga rather than keep progress in one API server. Each step would be a local transaction or idempotent external activity, and the workflow would persist its progress after every durable decision. For a moderately complex process with independent reactions, event choreography over a durable log may be sufficient. For a long-running process with branching, timeouts, human input, and compensation, I would use a workflow orchestrator or durable-execution engine. Activities would use stable operation IDs because the engine may retry after ambiguous failures. External callbacks would be correlated to a workflow ID and delivered as durable signals. If a later step fails, the workflow would execute idempotent compensations in reverse order, with bounded retries and a manual-review state for unresolved cases. I would also address workflow versioning, history growth, transactional event publication, and operational visibility.

---

# 44. Common Mistakes

## Mistake 1: Store Progress Only in Memory

A crash loses the workflow state.

## Mistake 2: Retry External Effects Without Idempotency

This can duplicate charges, emails, shipments, or refunds.

## Mistake 3: Claim Exactly-Once Activity Execution

Activity attempts may be repeated after ambiguous failures.

## Mistake 4: Treat Compensation as Perfect Rollback

Compensation is a new business action and may be incomplete or irreversible.

## Mistake 5: Publish Events Separately From Database Commit

This creates a dual-write inconsistency. Use an outbox or equivalent atomic publication mechanism.

## Mistake 6: Use a Workflow Engine for One Async Task

A queue may be sufficient.

## Mistake 7: Ignore In-Flight Workflow Deployments

Replay can fail or diverge when code changes incompatibly.

## Mistake 8: Put Large Payloads in History

Store large data externally and pass references.

## Mistake 9: Retry All Errors

Business failures and malformed requests should not be retried indefinitely.

## Mistake 10: Omit Manual Repair

Some ambiguous or irreversible failures require human intervention.

## Mistake 11: Confuse Event Sourcing With Event-Driven Architecture

Events can coordinate work without being the authoritative state model.

## Mistake 12: Assume a Waiting Workflow Holds a Thread

Durable waits are persisted and workers are released.

---

# 45. Important Invariants

## Durability Invariant

> **Every workflow decision required for recovery survives worker and machine failure.**

## Determinism Invariant

> **Replaying workflow code against the same history produces the same decisions.**

## Activity Invariant

> **Every retryable side effect has a stable operation identity and duplicate-handling policy.**

## Correlation Invariant

> **Every callback or signal maps to one durable workflow and logical step.**

## Compensation Invariant

> **Every compensable step has a durable, idempotent compensation with its own failure policy.**

## Publication Invariant

> **A local state change and its publication intent are committed atomically.**

## Versioning Invariant

> **Existing workflow histories remain compatible with deployed workflow code.**

## Termination Invariant

> **Retries, loops, and waits have explicit budgets, timeouts, or escalation paths.**

## Observability Invariant

> **Operators can determine the current state, blocker, next action, and repair path for every workflow instance.**

---

# 46. Quick Comparison

## Synchronous Function

```text
Best for:
Short operations within one request and one local transaction

Main benefit:
Simple and low latency

Main risk:
No durable progress across process failure
```

## Message Queue

```text
Best for:
One independent asynchronous task

Main benefit:
Simple buffering and retry

Main risk:
No built-in multi-step state machine
```

## Saga Choreography

```text
Best for:
Moderately complex flows with independent event reactions

Main benefit:
Loose service coupling

Main risk:
Implicit flow and difficult end-to-end debugging
```

## Saga Orchestration

```text
Best for:
Complex, long-running business processes

Main benefit:
Central flow, compensation, and visibility

Main risk:
Coordinator and platform dependency
```

## Code-Driven Durable Execution

```text
Best for:
Complex workflows that benefit from ordinary programming constructs

Main benefit:
Durable code, replay, timers, and signals

Main risk:
Determinism and versioning requirements
```

## Managed State Machine

```text
Best for:
Cloud-integrated workflows with clear state transitions

Main benefit:
Managed operation and visualization

Main risk:
Provider limits and state-machine expressiveness
```

## Event Sourcing

```text
Best for:
Domains where event history is the authoritative state

Main benefit:
Complete historical reconstruction

Main risk:
More complex modeling, evolution, and projections
```

---

# 47. Thirty-Second Revision

- **Multi-step process:** work spanning several actions, services, waits, or local transactions.
- **Primary problem:** partial failure leaves the system between business states.
- **Saga:** sequence of local commits with compensating actions.
- **Choreography:** services react to events without one central coordinator.
- **Orchestration:** one durable coordinator controls the flow.
- **Workflow engine:** persists progress, timers, retries, signals, and history.
- **Workflow code:** deterministic control logic.
- **Activity:** nondeterministic side effect that must tolerate retries.
- **Replay:** reconstruct workflow state from recorded history.
- **Durable wait:** persisted timer or condition, not a blocked thread.
- **Signal:** durable external event delivered to one workflow.
- **Idempotency:** repeated attempts produce one logical effect.
- **Outbox:** atomically stores local state and publication intent.
- **Compensation:** business action that counteracts an earlier commit.
- **Write ambiguity:** external action may succeed even when acknowledgement is lost.
- **Versioning:** old histories must remain compatible with new workflow code.
- **Continue as new:** restart a logical workflow with compact state and fresh history.
- **Manual review:** explicit state for failures automation cannot safely resolve.
- **Event-driven architecture:** uses events for communication and decoupling.
- **Event sourcing:** uses events as the authoritative state history.
- **Best principle:** persist progress, assume retries, and design every side effect for ambiguity.

## Final Mental Model

```text
Can the operation fit in one local transaction?
    ├── Yes → use the transaction
    └── No
        ↓
Is it one independent asynchronous action?
    ├── Yes → use a queue and idempotent worker
    └── No
        ↓
Model the process as a saga
        ↓
Are reactions independent and moderately complex?
    ├── Yes → event choreography may fit
    └── No → durable orchestration
        ↓
For every step define:
    input, timeout, retry, idempotency, compensation
        ↓
Persist progress and external signals
        ↓
On ambiguous failure:
    retry with stable identity or reconcile
        ↓
On unrecoverable failure:
    compensate or move to manual review
```
