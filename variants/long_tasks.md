# Managing Long-Running Tasks: System Design Study Guide

## Core Thesis

A long-running task should usually be separated from the HTTP request that creates it.

The API accepts and validates the request quickly, records a durable job, and returns a job identifier. Background workers process the job independently and persist its result.

```text
Client
    ↓ submit task
API service
    ↓ durable job creation
Queue
    ↓ claim work
Worker pool
    ↓
Result store and job database
    ↓
Polling, webhook, SSE, WebSocket, email, or push notification
```

Typical long-running tasks include:

- Video transcoding
- Image transformation
- PDF report generation
- Large data exports
- CSV imports
- Search indexing
- Bulk email delivery
- Machine-learning inference
- AI-agent tool execution
- Feed fan-out
- Virus scanning
- Third-party API calls with low rate limits

The central principle is:

> **Acknowledge accepted work quickly, execute it durably outside the request path, and make retries, progress, cancellation, and recovery explicit parts of the design.**

---

# 1. Why Synchronous Processing Breaks Down

A normal web request works well when the server can respond quickly:

```text
Client request
    ↓
Small database query
    ↓
Response in 100 ms
```

Now consider a report requiring 45 seconds:

```text
Client request
    ↓
Aggregate millions of rows
    ↓
Render charts
    ↓
Generate PDF
    ↓
Return response after 45 seconds
```

Problems include:

- Load balancer, gateway, or client timeout
- Long-held application connection
- Poor user feedback
- Expensive compute occupying web capacity
- Duplicate submissions when users retry
- Lost progress after server failure
- Difficult autoscaling
- Resource mismatch between API and task execution

Even if the request does not time out, keeping a user connected for minutes is usually a poor interaction model.

### Request-Path Invariant

> **The availability and latency of the interactive API must not depend on the completion time of expensive background work.**

---

# 2. Recognizing a Long-Running Task

Async processing becomes attractive when one or more conditions hold:

- The operation may exceed a few seconds.
- Runtime has high variance.
- Processing requires CPU, GPU, or large memory.
- Work can continue after the user navigates away.
- The task calls rate-limited external services.
- The task fans out to many recipients.
- Progress reporting is useful.
- The task should survive API-server restart.
- Completion can be eventually visible.

The “five-second rule” is only a heuristic. Some operations under five seconds should still be asynchronous because of resource cost or burstiness. Some slightly longer operations may remain synchronous if the product explicitly requires it and the infrastructure supports it.

---

# 3. Two-Phase Interaction

## Phase 1: Accept Work

The API:

1. Authenticates the caller.
2. Validates input.
3. Checks quota and authorization.
4. Deduplicates the logical request.
5. Creates a durable job record.
6. Ensures the job becomes dispatchable.
7. Returns immediately.

Example response:

```http
HTTP/1.1 202 Accepted
Content-Type: application/json
Location: /jobs/job-842
Retry-After: 2

{
  "jobId": "job-842",
  "status": "QUEUED",
  "statusUrl": "/jobs/job-842"
}
```

## Phase 2: Process Work

A worker:

1. Claims the job.
2. Loads job input.
3. Marks or leases it as running.
4. Performs the operation.
5. Stores artifacts and output metadata.
6. Marks success, failure, cancellation, or manual review.
7. Notifies interested clients.

---

# 4. Reference Architecture

```text
                         ┌──────────────┐
Client ──submit────────> │ API service  │
                         └──────┬───────┘
                                │ create job and enqueue intent
                                v
                         ┌──────────────┐
                         │ Job database │
                         └──────┬───────┘
                                │
                                v
                         ┌──────────────┐
                         │ Durable queue│
                         └──────┬───────┘
                                │ claim
                                v
                         ┌──────────────┐
                         │ Worker pools │
                         └──────┬───────┘
                                │
                 ┌──────────────┼──────────────┐
                 v              v              v
          Object storage    Databases     External APIs
                 │
                 v
         status and notification
```

## Component Responsibilities

### API Service

- Request validation
- Job admission
- Idempotency
- Job ownership
- Fast status responses

### Job Database

- Durable job metadata
- State transitions
- Progress
- Attempt history
- Result references
- Cancellation intent

### Queue

- Durable dispatch
- Load buffering
- Worker decoupling
- Redelivery after failure
- Delayed retries

### Workers

- Heavy computation
- External calls
- Checkpointing
- Heartbeats
- Result storage

### Result Store

- Large files in blob storage
- Structured output in database
- Logs and diagnostic artifacts

---

# 5. Job State Machine

A job should have a clearly defined lifecycle.

```text
PENDING
    ↓
QUEUED
    ↓
RUNNING
    ├── SUCCEEDED
    ├── RETRY_WAIT
    ├── FAILED
    ├── CANCELLATION_REQUESTED
    ├── CANCELLED
    └── NEEDS_MANUAL_REVIEW
```

Additional states may include:

```text
PAUSED
TIMED_OUT
PARTIALLY_SUCCEEDED
EXPIRED
DEAD_LETTERED
```

## Example Schema

```sql
CREATE TABLE jobs (
    job_id UUID PRIMARY KEY,
    owner_id UUID NOT NULL,
    job_type TEXT NOT NULL,
    status TEXT NOT NULL,
    idempotency_key TEXT,
    priority INTEGER NOT NULL DEFAULT 0,
    progress_percent INTEGER,
    current_stage TEXT,
    attempt_count INTEGER NOT NULL DEFAULT 0,
    max_attempts INTEGER NOT NULL,
    lease_owner TEXT,
    lease_expires_at TIMESTAMP,
    next_attempt_at TIMESTAMP,
    input_ref TEXT,
    result_ref TEXT,
    error_code TEXT,
    error_message TEXT,
    created_at TIMESTAMP NOT NULL,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    updated_at TIMESTAMP NOT NULL
);
```

## State Transition Rule

Use conditional updates so two workers cannot both finalize the same job incorrectly.

```sql
UPDATE jobs
SET status = 'RUNNING',
    lease_owner = :worker_id,
    lease_expires_at = NOW() + INTERVAL '30 seconds',
    attempt_count = attempt_count + 1,
    started_at = COALESCE(started_at, NOW()),
    updated_at = NOW()
WHERE job_id = :job_id
  AND status IN ('QUEUED', 'RETRY_WAIT');
```

### State Invariant

> **Every job has one authoritative state, and transitions occur only through valid conditional state changes.**

---

# 6. Durable Job Creation

A subtle failure exists between database insertion and queue publication.

## Failure A

```text
Job row committed
    ↓
API crashes before queue publish
```

The job remains pending forever.

## Failure B

```text
Message published
    ↓
Database transaction fails
```

A worker receives a job that does not exist.

## Transactional Outbox Solution

Write the job and enqueue intent in one database transaction:

```sql
BEGIN;

INSERT INTO jobs (
    job_id,
    owner_id,
    job_type,
    status,
    input_ref,
    created_at,
    updated_at
)
VALUES (
    :job_id,
    :owner_id,
    :job_type,
    'PENDING',
    :input_ref,
    NOW(),
    NOW()
);

INSERT INTO outbox (
    event_id,
    event_type,
    aggregate_id,
    payload,
    created_at
)
VALUES (
    :event_id,
    'JobRequested',
    :job_id,
    :payload,
    NOW()
);

COMMIT;
```

An outbox relay publishes the message. Publication may repeat, so workers must deduplicate by `job_id` or operation ID.

## Queue-First Alternative

If the queue is the durable source of truth, the API may append the complete command there and construct the status view asynchronously. The design must still expose a durable operation ID and handle duplicate submissions.

### Admission Invariant

> **A job acknowledged as accepted must exist in a durable source from which dispatch can be recovered.**

---

# 7. What Goes in the Queue Message?

Prefer a small message:

```json
{
  "jobId": "job-842",
  "jobType": "GENERATE_REPORT",
  "attempt": 1,
  "traceId": "trace-991"
}
```

Store large input elsewhere:

- Database
- Object storage
- Durable document store

Benefits:

- Avoid queue message-size limits
- Avoid duplicated large payloads
- Support input versioning
- Simplify retries
- Keep sensitive data out of queue logs

The worker must verify that it is loading the intended immutable or versioned input. If job input can change, record the expected input version.

---

# 8. Queue Choices

## Redis-Based Job Queues

Strengths:

- Simple developer experience
- Low latency
- Delayed jobs and priorities through frameworks
- Good for moderate workloads

Trade-offs:

- Durability depends on persistence and replication configuration.
- Memory-first operation can raise cost.
- Queue semantics may come from a library rather than Redis itself.

## Managed Queue Service

Strengths:

- Minimal operational burden
- Automatic scaling
- Built-in redelivery and dead-letter support

Trade-offs:

- Message-size and retention limits
- Delivery delay and pricing model
- Provider-specific semantics

## RabbitMQ-Like Broker

Strengths:

- Routing exchanges
- Acknowledgements
- Flexible delivery patterns
- Mature queue semantics

Trade-offs:

- Cluster and disk operations
- Scaling and upgrade burden
- Careful flow-control tuning

## Kafka-Like Durable Log

Strengths:

- High throughput
- Replay
- Long retention
- Multiple consumer groups
- Ordering within a partition

Trade-offs:

- Job claiming differs from a traditional work queue.
- A slow message can block a partition.
- Per-message delay and arbitrary priority are less natural.
- Consumer and partition design matter.

### Selection Principle

> **Choose from durability, ordering, delay, retry, fan-out, throughput, and operating model. Kafka is not automatically the best task queue merely because it is popular.**

---

# 9. Worker Deployment Models

## Long-Lived Servers or VMs

Best for:

- Very long tasks
- Specialized local state
- Predictable load
- Easy interactive debugging

Costs:

- Server management
- Idle capacity
- Manual or custom autoscaling

## Containers

Best for:

- Mixed worker types
- Custom dependencies
- CPU, memory, or GPU resource requests
- Tasks longer than serverless limits

Benefits:

- Reproducible environment
- Independent scaling
- Isolation by worker pool

Costs:

- Orchestrator complexity
- Image startup and scheduling latency

## Serverless Functions

Best for:

- Short, stateless, bursty tasks
- Event-driven execution
- Low operational overhead

Limitations:

- Execution-duration ceiling
- Cold starts
- Ephemeral disk and memory limits
- Provider concurrency limits
- Poor fit for multi-hour jobs

## Batch and HPC Systems

Best for:

- Large parallel compute jobs
- GPU fleets
- Scientific processing
- Schedulable bulk workloads

---

# 10. Independent Resource Scaling

Separate worker classes by resource profile:

```text
Web API workers       → small general-purpose instances
PDF workers           → CPU and memory optimized
Video transcoders     → GPU or media-optimized instances
ML inference workers  → accelerator instances
Email workers         → network-heavy, low CPU
```

The queue decouples request rate from execution hardware.

### Isolation Invariant

> **Expensive or unstable task execution must not exhaust the resources required to serve interactive APIs.**

---

# 11. Claiming Jobs Safely

Queue systems typically deliver a message to one consumer, but duplicate delivery remains possible.

The worker should establish ownership through a lease or queue acknowledgement protocol.

```text
Receive message
    ↓
Conditionally claim job
    ↓
Process
    ↓
Persist terminal state
    ↓
Acknowledge queue message
```

If the message is acknowledged before the result is durable, a crash can lose the job.

Safer order:

```text
Persist result and job state
    ↓
Acknowledge queue message
```

If the worker crashes after persistence but before acknowledgement, the message is redelivered. Idempotency makes the duplicate harmless.

---

# 12. Visibility Timeouts, Leases, and Heartbeats

A worker needs temporary ownership while processing.

## Visibility Timeout

After a worker receives a message, the queue hides it for a period.

```text
Message received
    ↓
Invisible for 60 seconds
    ↓
Worker acknowledges success
or
Timeout expires and message becomes visible again
```

## Lease Renewal

If a job may exceed the initial timeout, the worker extends visibility or renews a database lease.

```text
Every 10 seconds:
renew lease for another 30 seconds
report progress checkpoint
```

## Heartbeat Trade-Off

Too infrequent:

- Slow crash detection
- Delayed retry

Too frequent:

- More queue and database traffic
- False failure during temporary pauses if thresholds are too aggressive

Choose from expected task duration, tolerated failover delay, network conditions, and runtime pauses.

### Lease Invariant

> **A worker may mutate job progress only while it owns a valid lease, and stale workers must not overwrite newer attempts.**

## Fencing Attempts

Store an attempt number or fencing token. Every progress and completion update includes it:

```sql
UPDATE jobs
SET status = 'SUCCEEDED',
    result_ref = :result_ref,
    completed_at = NOW()
WHERE job_id = :job_id
  AND attempt_count = :attempt
  AND lease_owner = :worker_id;
```

A worker resuming after lease expiry cannot finalize over the current owner.

---

# 13. Failure Timeline

Consider:

```text
1. Worker receives job.
2. Worker generates report.
3. Worker stores PDF.
4. Worker crashes before updating job status.
5. Queue redelivers job.
```

The next worker must determine whether to:

- Reuse the existing artifact
- Regenerate safely
- Reconcile ambiguous output

A deterministic output key helps:

```text
reports/{job_id}/report-v1.pdf
```

A retry can verify the artifact and continue instead of producing a second unrelated object.

---

# 14. Idempotent Submission

Users may click repeatedly or retry after a timeout.

The client should provide an idempotency key or logical request ID:

```http
POST /reports
Idempotency-Key: user-7-annual-report-2025
```

Create a unique constraint:

```sql
CREATE UNIQUE INDEX jobs_owner_idempotency_key
ON jobs(owner_id, idempotency_key)
WHERE idempotency_key IS NOT NULL;
```

Submission flow:

```text
1. Begin transaction.
2. Insert job with idempotency key.
3. If conflict, return existing job.
4. Insert outbox event for new job only.
5. Commit.
```

Avoid generating idempotency keys only from rounded timestamps when distinct legitimate jobs may fall in the same window. Prefer caller-supplied request IDs or deterministic business identities.

---

# 15. Idempotent Job Effects

Submission deduplication prevents duplicate jobs, but worker retries can still repeat side effects.

Examples:

- Sending email twice
- Charging a payment twice
- Creating duplicate files
- Applying database changes twice

Strategies:

- Stable provider idempotency key
- Unique database constraint
- Conditional state transition
- Deterministic artifact key
- Processed-operation table
- Query external provider by operation reference

### Idempotency Invariant

> **Duplicate job delivery and duplicate worker execution must converge on one logical business effect.**

---

# 16. Retry Policies

Retry transient failures, not permanent ones.

## Retryable

- Network timeout
- Temporary dependency outage
- Rate limit
- Worker termination
- Transient database error

## Non-Retryable

- Invalid input
- Unsupported format
- Authorization failure
- Missing required source data
- Business rule rejection

## Exponential Backoff

```text
attempt 1 → 1 second
attempt 2 → 2 seconds
attempt 3 → 4 seconds
attempt 4 → 8 seconds
```

Add jitter so many failed jobs do not retry simultaneously.

## Retry Budget

Define:

- Maximum attempts
- Maximum elapsed time
- Maximum delay
- Per-error policy
- Terminal action

### Retry Invariant

> **Every job type classifies errors and has a bounded retry policy with a defined terminal outcome.**

---

# 17. Dead-Letter Queues

A poison job repeatedly fails because of bad data, a deterministic bug, or an unsupported condition.

After the retry budget:

```text
Main queue
    ↓ repeated failures
Dead-letter queue or quarantine state
```

Store:

- Job ID
- Error class
- Attempt count
- Worker version
- Last stack trace or diagnostic reference
- Input reference
- Failure timestamps

## Operational Process

1. Alert on DLQ growth.
2. Classify the failure.
3. Fix code or data.
4. Replay selected jobs idempotently.
5. Record resolution.

A DLQ without ownership and replay tooling is merely hidden failure storage.

---

# 18. Progress Reporting

Progress can be:

- Percent complete
- Current stage
- Items processed
- Total items
- Estimated completion time
- Last heartbeat

Example:

```json
{
  "jobId": "job-842",
  "status": "RUNNING",
  "stage": "RENDERING_CHARTS",
  "completedUnits": 72,
  "totalUnits": 100,
  "progressPercent": 72
}
```

## Progress Update Frequency

Updating the database for every processed item can create a write bottleneck.

Throttle progress writes:

```text
Update every 5 seconds
or
when progress changes by at least 1 percent
```

Progress is often approximate. It should be monotonic within one attempt unless the UI explains otherwise.

---

# 19. Client Completion Models

## Polling

```http
GET /jobs/job-842
```

Advantages:

- Simple
- Stateless API
- Works everywhere

Costs:

- Repeated requests
- Completion delay up to polling interval

Use backoff and `Retry-After`.

## Long Polling

The server waits until status changes or timeout occurs.

Useful when jobs complete infrequently and a simple HTTP model is preferred.

## SSE

Good for one-way progress and completion updates.

## WebSocket

Useful when the application already needs frequent bidirectional communication.

## Webhook

Good for server-to-server task completion.

Requirements:

- Signature verification
- Retries
- Idempotent receiver
- Delivery history

## Email or Push Notification

Useful for jobs lasting minutes or hours when the user need not keep the application open.

---

# 20. Result Storage

Do not place large result artifacts directly in queue messages or job rows.

Use:

```text
PDF, video, archive → object storage
Structured metadata → database
Small scalar result → job row
```

The job stores a result reference:

```text
result_ref = object://reports/job-842/report.pdf
```

For private results, issue a temporary signed download URL after authorization.

Define result retention:

- Expiration date
- Download count if needed
- Regeneration policy
- Deletion when owner account is removed

---

# 21. Cancellation

A user may cancel a queued or running task.

State flow:

```text
QUEUED → CANCELLED
RUNNING → CANCELLATION_REQUESTED → CANCELLED
```

## Cooperative Cancellation

Workers periodically check:

- Job status
- Cancellation token
- Heartbeat response

On cancellation:

1. Stop at a safe point.
2. Clean temporary artifacts.
3. Release external resources.
4. Persist checkpoint or terminal status.
5. Acknowledge queue message.

Not every side effect can be reversed. Cancellation policy must distinguish “stop future work” from “undo completed work.”

### Cancellation Invariant

> **A cancelled job stops producing new effects after a defined safe boundary and leaves resources in a documented state.**

---

# 22. Checkpointing

Multi-hour tasks should avoid restarting from zero after failure.

Checkpoint examples:

- Last processed row ID
- Completed video segments
- Uploaded output parts
- Model checkpoint
- Finished fan-out batch

Flow:

```text
Process chunk
    ↓
Persist output and checkpoint
    ↓
Renew lease
    ↓
Process next chunk
```

A checkpoint must be consistent with the durable output. If a worker records progress before output is durable, recovery may skip missing work.

### Checkpoint Invariant

> **A checkpoint may advance only after the work it represents is durably committed or can be reproduced idempotently.**

---

# 23. Splitting Large Jobs

A five-hour job can often be decomposed:

```text
Parent job
    ├── chunk 1
    ├── chunk 2
    ├── chunk 3
    └── chunk N
          ↓
       final merge
```

Benefits:

- Parallelism
- Smaller retry units
- Better progress reporting
- Reduced lease duration
- More balanced worker utilization

Costs:

- Coordination
- Duplicate child handling
- Final aggregation
- Partial completion
- More queue messages

The parent should track expected child IDs and deduplicate child completions.

---

# 24. Mixed Workloads and Head-of-Line Blocking

If one queue contains both 2-second and 5-hour tasks, long jobs can delay short ones.

Separate by:

- Job type
- Expected duration
- Resource class
- Priority
- Tenant tier
- Deadline

Example:

```text
fast-cpu queue
slow-cpu queue
gpu queue
external-api queue
high-priority queue
```

Each queue can have different:

- Concurrency
- Worker image
- Instance type
- Timeout
- Retry policy
- Autoscaling rule

## Misclassification

If runtime is difficult to predict:

- Estimate from input size.
- Use historical runtime.
- Split by workload type.
- Checkpoint and requeue oversized work.

Killing a slow job merely because it exceeded the fast threshold can waste completed work. Prefer classification before execution or safe checkpoint-based migration.

---

# 25. Priority and Fairness

A strict priority queue can starve low-priority work.

Use:

- Weighted fair queuing
- Per-tenant concurrency limits
- Aging, where waiting jobs gain priority
- Reserved capacity for critical jobs
- Separate queues with controlled worker allocation

Example:

```text
70 percent worker capacity → standard jobs
20 percent → premium or urgent jobs
10 percent → maintenance and retries
```

### Fairness Invariant

> **One tenant, job class, or retry storm cannot consume unbounded worker capacity and starve unrelated work.**

---

# 26. Backpressure

When arrival rate exceeds completion rate, backlog grows.

```text
arrival rate λ
processing rate μ

if λ > μ continuously:
    queue age grows without bound
```

Backpressure mechanisms:

- Admission limits
- HTTP 429 or 503
- Tenant quotas
- Maximum queue depth
- Maximum estimated wait time
- Producer throttling
- Deferred scheduling
- Lower-priority rejection

Do not accept a job with a promised ten-minute completion time when the oldest waiting job is already six hours old.

---

# 27. Autoscaling Workers

Scale workers using workload signals, not only CPU.

Useful signals:

- Queue depth
- Oldest-message age
- Arrival rate
- Completion rate
- Estimated queued work seconds
- Per-resource utilization
- Deadline miss rate

## Better Metric: Work Backlog

If jobs vary greatly in duration, 1,000 jobs is not an informative backlog measure.

Estimate:

```text
queued work seconds
= sum(predicted runtime of queued jobs)
```

Workers required approximately:

```text
queued work seconds / target drain time
```

subject to concurrency and resource constraints.

## Scaling Delay

Account for:

- VM launch time
- Container image pull
- GPU scheduling
- Model loading
- Cache warm-up

Maintain warm capacity for strict latency SLOs.

---

# 28. Queue Capacity Does Not Equal Processing Capacity

A durable queue can absorb a temporary burst, but it cannot fix permanent under-capacity.

Example:

```text
arrival = 1,000 jobs/minute
completion = 800 jobs/minute
backlog growth = 200 jobs/minute
```

After one hour:

```text
12,000 additional pending jobs
```

The response must be one or more of:

- Add sustainable worker capacity
- Reduce work per job
- Batch tasks
- Rate limit producers
- Shed low-value jobs
- Change product expectations

---

# 29. Retry Storms

A dependency outage can cause every job to retry together.

```text
Provider fails
    ↓
Thousands of jobs fail
    ↓
Immediate retries
    ↓
Provider remains overloaded
```

Mitigations:

- Exponential backoff
- Jitter
- Circuit breaker
- Global concurrency cap per dependency
- Retry budget
- Delayed queue
- Provider-specific rate limiter

Retries should not dominate fresh work indefinitely.

---

# 30. External API Rate Limits

If a job calls a provider limited to 100 requests per second, adding 1,000 workers does not increase provider capacity.

Use a shared limiter:

```text
Worker pool
    ↓
Distributed token bucket
    ↓
External API
```

Additional considerations:

- Provider-specific backoff
- `Retry-After`
- Tenant fairness
- Credential-level limits
- Request batching
- Circuit breaking

Queue the work at the rate-limited boundary rather than allowing workers to block while holding scarce resources.

---

# 31. Timeouts

Define several timeouts:

- Queue wait deadline
- Attempt execution timeout
- Total job deadline
- Dependency timeout
- Heartbeat timeout
- Result-retention expiry

A timeout is not automatically a cancellation. The underlying external operation may still complete after the caller stops waiting.

Use operation IDs and reconciliation for ambiguous external effects.

---

# 32. Poison Jobs and Worker Safety

Malformed inputs may crash processes or exhaust memory.

Protect workers using:

- Input validation before enqueue
- Resource limits
- Process or container isolation
- Maximum file and batch size
- Sandboxing for untrusted code
- Per-job timeout
- DLQ
- Crash-loop detection

For online code execution:

- Isolate filesystem and network
- Limit CPU and memory
- Restrict syscalls
- Enforce wall-clock timeout
- Recycle sandbox after execution

One hostile task must not compromise the fleet.

---

# 33. Logging and Observability

Correlate every event with:

```text
job_id
attempt_id
owner_id or tenant_id
trace_id
worker_id
queue name
job type
```

Monitor:

## Queue

- Depth
- Oldest-message age
- Enqueue rate
- Dequeue rate
- Redelivery rate
- DLQ depth

## Workers

- Active workers
- Success rate
- Failure rate
- Runtime percentiles
- Heartbeat failures
- Resource utilization
- Crash rate

## Jobs

- End-to-end latency
- Queue wait time
- Execution time
- Retry count
- Cancellation count
- Stuck jobs
- Progress-update age

## Dependencies

- Provider latency
- Rate-limit responses
- Timeout rate
- Circuit-breaker state

Average runtime alone is insufficient. Track p95 and p99 by job type and input size.

---

# 34. Detecting Stuck Jobs

A watchdog can search for:

```sql
SELECT *
FROM jobs
WHERE status = 'RUNNING'
  AND lease_expires_at < NOW();
```

Possible actions:

- Requeue job
- Mark retry wait
- Increment attempt
- Move to manual review
- Alert if output is ambiguous

Also detect:

- Queued too long
- No progress update
- Cancellation requested but not observed
- Retry wait past due
- Completed job missing result

---

# 35. Graceful Worker Shutdown

During deployment:

```text
1. Stop claiming new jobs.
2. Continue current jobs until drain deadline.
3. Checkpoint long tasks.
4. Extend leases while draining.
5. Release or requeue unfinished jobs.
6. Exit.
```

Abruptly terminating every worker can cause mass redelivery and duplicate side effects.

Container orchestrators need termination grace periods appropriate for the worker type.

---

# 36. Job Versioning

Queued jobs may wait while worker code changes.

Include a schema or job version:

```json
{
  "jobId": "job-842",
  "jobType": "GENERATE_REPORT",
  "jobVersion": 3
}
```

Strategies:

- Keep old worker versions until old jobs drain.
- Migrate queued job payloads.
- Make new workers backward compatible.
- Reject unsupported versions into a repair queue.

Do not assume that a message created by version 1 remains interpretable by version 5.

### Version Invariant

> **Every queued job is executable by a known worker version or has a controlled migration path.**

---

# 37. Database Polling as a Simple Queue

For modest systems, the jobs table itself can be the queue.

Example claim pattern:

```sql
BEGIN;

SELECT job_id
FROM jobs
WHERE status = 'QUEUED'
  AND next_attempt_at <= NOW()
ORDER BY priority DESC, created_at
FOR UPDATE SKIP LOCKED
LIMIT 1;

UPDATE jobs
SET status = 'RUNNING',
    lease_owner = :worker_id,
    lease_expires_at = NOW() + INTERVAL '30 seconds'
WHERE job_id = :job_id;

COMMIT;
```

Benefits:

- Fewer components
- Atomic state and claim
- Easy operational inspection

Costs:

- Polling load
- Table and index contention
- Limited throughput
- Cleanup and vacuum pressure

This can be an excellent starting point before introducing a broker.

---

# 38. Queue Versus Workflow Engine

A queue is sufficient when:

- There is one main task.
- Retry behavior is straightforward.
- The job has little branching.
- No durable multi-day waits exist.
- Compensation is minimal.

A workflow engine is preferable when:

- Several dependent steps exist.
- Steps run in parallel.
- The process waits for callbacks or humans.
- Compensation is required.
- Workflow history must survive deployments.
- Operators need step-level visibility.

Example complex flow:

```text
Fetch data
    ↓
Generate charts ─┐
Generate tables ─┼─> Merge PDF → Upload → Email
Render summary ──┘
```

Do not recreate a workflow engine through ad hoc queue chaining once branching, durable timers, and compensation become central.

---

# 39. Simple Job Chaining

For a small sequence, one job may enqueue the next using a transactional outbox.

```text
FETCH_DATA succeeds
    ↓
Create GENERATE_PDF job
    ↓
GENERATE_PDF succeeds
    ↓
Create SEND_EMAIL job
```

Requirements:

- Stable workflow ID
- Idempotent child creation
- Durable handoff
- Step status
- Failure policy

Avoid placing large intermediate data in messages. Store it in object storage and pass references.

Once the chain needs loops, parallel joins, long waits, or compensation, use durable orchestration.

---

# 40. Result and Side-Effect Ambiguity

Suppose a worker sends an email and crashes before marking success.

The next attempt cannot know whether the provider accepted the email.

Correct strategies:

- Provider idempotency key
- Query provider by operation reference
- Deterministic message ID
- Outbox owned by the email service
- Reconciliation

A local `IN_PROGRESS` row identifies ambiguity but does not prove whether the external effect occurred.

### Effect Invariant

> **Every irreversible external effect has an idempotency or reconciliation strategy for the crash-after-effect-before-acknowledgement window.**

---

# 41. Security and Multi-Tenant Isolation

Job APIs must enforce:

- Owner authorization
- Tenant-scoped input and result references
- Per-tenant quotas
- Resource limits
- Signed result URLs
- Secret redaction
- Encrypted sensitive input

Workers must not trust queue payloads merely because they came from an internal queue. Validate identifiers and authorization-relevant ownership before accessing data.

Do not expose raw stack traces or sensitive job input through public status endpoints.

---

# 42. Common Product Scenarios

## Video Platform

Jobs:

- Virus scan
- Metadata extraction
- Transcode variants
- Generate thumbnails
- Produce streaming manifests
- Content moderation

Use GPU or media-optimized queues and deterministic output keys.

## Photo Sharing

Jobs:

- Resize variants
- Remove metadata
- Moderate content
- Index image tags
- Fan out post references

## Coding Platform

Jobs:

- Compile code
- Execute test cases
- Aggregate result

Strong sandboxing, resource limits, per-language pools, and deterministic test IDs are important.

## AI Generation

Jobs:

- Model inference
- Tool calls
- Safety checks
- Artifact creation

Use accelerator pools, token or cost budgets, cancellation, and streamed progress where useful.

## Data Export

Jobs:

- Query chunks
- Write partial files
- Merge archive
- Upload result
- Notify user

Use checkpoints to avoid restarting a multi-hour export.

## Bulk Email

Jobs:

- Expand recipients
- Apply suppression lists
- Render templates
- Send in provider-sized batches
- Track delivery events

Use rate limits, idempotency, and per-tenant fairness.

---

# 43. When Not to Use Async Workers

Keep processing synchronous when:

- The operation is fast and predictable.
- The user requires immediate authoritative success or failure.
- Adding a queue creates more complexity than value.
- The action fits safely in one local transaction.
- Result polling or notification would worsen the user experience.

Examples:

- Reading a profile
- Updating a small preference
- Validating credentials
- Creating a simple database row

Async processing is not automatically more scalable. It shifts complexity into durable state, retries, and eventual consistency.

---

# 44. Interview Design Process

## Step 1: Estimate Runtime and Workload

Ask:

- Typical and maximum runtime
- Arrival rate and peak factor
- CPU, memory, GPU, disk, and network needs
- Input and output size

## Step 2: Define the API Contract

Use:

- `202 Accepted`
- Job ID
- Status URL
- Idempotency key
- Cancellation endpoint

## Step 3: Define Durable Admission

Explain how job creation and dispatch survive a crash, such as an outbox.

## Step 4: Choose Queue Semantics

State:

- Delivery model
- Ordering requirement
- Visibility timeout or lease
- Delayed retry
- Retention
- DLQ

## Step 5: Design Workers

Define:

- Resource class
- Concurrency
- Timeout
- Heartbeat
- Checkpoint
- Idempotency
- Result storage

## Step 6: Design User Feedback

Choose:

- Polling
- SSE
- WebSocket
- Webhook
- Email or push

## Step 7: Address Scale

Discuss:

- Queue age
- Autoscaling
- Backpressure
- Fairness
- Queue separation
- Rate-limited dependencies

## Step 8: Address Failure

Discuss:

- Worker crash
- Duplicate delivery
- Poison jobs
- Ambiguous external effects
- Deployment
- Cancellation
- Reconciliation

## Step 9: Decide Whether It Is Really a Workflow

If dependencies, branching, waits, and compensation dominate, choose a workflow engine rather than a plain worker queue.

---

# 45. Interview Answer Template

> This operation takes too long and uses resources that should not be tied to the interactive API, so I would return `202 Accepted` with a durable job ID and process it asynchronously. The API validates the request, applies an idempotency key, creates the job and an outbox record in one transaction, and returns a status URL. A relay publishes the job ID to a durable queue. Workers claim jobs using a visibility timeout or lease, heartbeat while running, checkpoint long work, and persist the result before acknowledging the queue message. Since delivery is at least once, both the job and every external side effect use stable operation IDs and idempotent state transitions. Retries use bounded exponential backoff and jitter, while permanent failures move to a monitored DLQ. Clients receive progress through polling, SSE, WebSocket, webhook, or notification. Worker pools are separated by duration and resource class, autoscale from queue age and estimated work, and apply backpressure when completion capacity falls behind arrivals. If the task evolves into a branching multi-step process with durable waits or compensation, I would move it to a workflow engine.

---

# 46. Common Mistakes

## Mistake 1: Perform Minute-Long Work in the HTTP Request

This creates timeouts and couples user-facing capacity to batch work.

## Mistake 2: Insert Job Then Publish Without Atomic Handoff

A crash can strand a job. Use an outbox or one durable source of truth.

## Mistake 3: Acknowledge the Queue Before Persisting the Result

A worker crash can lose successful work.

## Mistake 4: Assume Queue Delivery Is Exactly Once

Design for duplicate delivery and ambiguous attempts.

## Mistake 5: Store Large Payloads in Queue Messages

Pass durable references instead.

## Mistake 6: Use One Queue for Every Workload

Mixed durations and resource classes create head-of-line blocking.

## Mistake 7: Scale Only From CPU

Queue age and pending work may be growing before CPU appears saturated.

## Mistake 8: Retry Permanent Errors Forever

Classify errors and use a DLQ.

## Mistake 9: Ignore Cancellation

Users and operators need a safe way to stop wasteful work.

## Mistake 10: Update Progress Too Frequently

Progress writes can become their own bottleneck.

## Mistake 11: Lose Accepted Jobs in an In-Memory Batch

Acknowledged work must be durable.

## Mistake 12: Chain Complex Steps Manually Forever

Use a workflow engine once orchestration becomes core complexity.

## Mistake 13: Reuse Timestamp-Derived Idempotency Keys Carelessly

Distinct valid jobs may collide. Use logical request IDs.

## Mistake 14: Let an Expired Worker Finalize

Use attempt fencing or conditional updates.

---

# 47. Operational Metrics

## Admission

- Job submission rate
- Deduplicated submissions
- Rejected jobs
- API latency
- Outbox publication lag

## Queue

- Queue depth
- Oldest-message age
- Enqueue and dequeue rate
- Redelivery rate
- DLQ depth
- Per-partition lag

## Workers

- Active and idle workers
- Job runtime percentiles
- Success and failure rate
- Heartbeat failures
- Lease expirations
- Worker crashes
- CPU, memory, GPU, and disk usage

## Jobs

- End-to-end completion latency
- Queue wait latency
- Attempt count
- Cancellation latency
- Progress age
- Stuck jobs
- Result-storage failures

## Scaling

- Estimated queued work seconds
- Worker startup latency
- Scale-up and scale-down events
- Deadline misses
- Tenant fairness

## Dependencies

- External API rate limits
- Timeout rate
- Circuit-breaker state
- Provider error classes

---

# 48. Important Invariants

## Admission Invariant

> **Every accepted job is recoverably durable and eventually dispatchable.**

## State Invariant

> **Every job has one authoritative state and valid conditional transitions.**

## Lease Invariant

> **Only the current unexpired attempt may report progress or terminal completion.**

## Idempotency Invariant

> **Duplicate submissions and executions create one logical effect.**

## Checkpoint Invariant

> **Recorded progress never advances beyond durable work.**

## Retry Invariant

> **Retries are bounded, classified, delayed, and observable.**

## Backpressure Invariant

> **The system does not accept unbounded work beyond its processing and latency capacity.**

## Fairness Invariant

> **One tenant or workload cannot starve unrelated jobs.**

## Cancellation Invariant

> **Cancellation stops future effects at a defined safe boundary.**

## Version Invariant

> **Every queued payload remains executable or migratable across deployments.**

## Effect Invariant

> **Every external side effect has a strategy for duplicate and ambiguous execution.**

---

# 49. Quick Comparison

## Synchronous Request

```text
Best for:
Fast, predictable operations needing immediate result

Benefit:
Simple and strongly synchronous UX

Cost:
Ties API resources to execution time
```

## Database-Backed Job Queue

```text
Best for:
Modest workloads and simple architecture

Benefit:
Atomic job state and claim

Cost:
Polling and database contention at scale
```

## Managed Work Queue

```text
Best for:
Independent background jobs with retries

Benefit:
Durability and low operational burden

Cost:
Provider limits and eventual consistency
```

## Durable Log

```text
Best for:
High-throughput replayable event processing

Benefit:
Retention, partitions, multiple consumers

Cost:
Less natural arbitrary priority and delayed-job semantics
```

## VM or Container Worker

```text
Best for:
Long jobs and custom resource profiles

Benefit:
Runtime flexibility

Cost:
Infrastructure management
```

## Serverless Worker

```text
Best for:
Short, spiky, stateless jobs

Benefit:
Automatic scaling and pay-per-use

Cost:
Runtime, storage, and concurrency constraints
```

## Workflow Engine

```text
Best for:
Multi-step jobs with branching, waits, and compensation

Benefit:
Durable orchestration and visibility

Cost:
Additional platform and programming model
```

---

# 50. Thirty-Second Revision

- **Long-running task:** work that should outlive one interactive HTTP request.
- **Core pattern:** accept quickly, enqueue durably, process in workers, persist result, notify user.
- **API response:** usually `202 Accepted` with job ID and status URL.
- **Job database:** authoritative state, progress, attempts, result, and ownership.
- **Queue:** durable dispatch, buffering, redelivery, and delayed retries.
- **Outbox:** atomically records job state and queue-publication intent.
- **Worker lease:** temporary ownership renewed by heartbeat.
- **Visibility timeout:** hides a claimed message until acknowledgement or expiry.
- **Fencing attempt:** prevents an expired worker from overwriting the current attempt.
- **Idempotency key:** deduplicates repeated submissions and side effects.
- **Retry:** transient failures only, with backoff, jitter, and limits.
- **DLQ:** isolates repeatedly failing poison jobs for investigation.
- **Progress:** update at bounded intervals, not on every tiny unit.
- **Checkpoint:** persist completed work so long jobs can resume.
- **Cancellation:** cooperative stop at a safe boundary.
- **Mixed workloads:** separate queues by duration, priority, and resource type.
- **Autoscaling:** use queue age and estimated queued work, not CPU alone.
- **Backpressure:** reject or defer work when completion capacity is insufficient.
- **Result storage:** large artifacts in object storage, references in job metadata.
- **Workflow boundary:** use durable orchestration when simple independent jobs become branching state machines.
- **Best principle:** durable admission plus at-least-once processing, idempotent effects, bounded retries, and observable state.

## Final Mental Model

```text
Request arrives
    ↓
Can it finish quickly and predictably?
    ├── Yes → synchronous request may be simplest
    └── No
        ↓
Validate, authorize, and deduplicate
        ↓
Create durable job and outbox intent
        ↓
Return 202 + job ID
        ↓
Queue dispatches to appropriate worker pool
        ↓
Worker claims lease and heartbeats
        ↓
Process with checkpoints and idempotent effects
        ↓
Transient failure?
    ├── Yes → bounded delayed retry
    └── No → DLQ, failed, or manual review
        ↓
Persist result before queue acknowledgement
        ↓
Notify client and apply retention

At every stage ask:
    Is accepted work durable?
    Can the message repeat?
    Who owns the current attempt?
    Can the job resume?
    Can it be cancelled?
    How deep and old is the queue?
    When should this become a workflow instead?
```
