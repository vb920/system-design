# Scaling Writes: System Design Study Guide

## Core Thesis

**Scaling writes means increasing the rate at which a system can durably accept and process state changes without exceeding the CPU, memory, disk, network, or coordination capacity of any one component.**

Write-heavy systems include:

- Metrics and telemetry ingestion
- Ad-click collection
- Social engagement counters
- Location tracking
- Notification generation
- Search indexing pipelines
- Financial ledgers
- Event and audit logs
- IoT ingestion
- Live comments and reactions

The central idea is:

> **Reduce the write work performed by each component by improving the write path, spreading independent writes, buffering temporary bursts, removing low-value work, and combining compatible updates.**

A practical progression is:

```text
1. Measure the write workload.
2. Remove unnecessary write amplification.
3. Scale the existing node vertically.
4. Choose a storage engine that matches the access pattern.
5. Partition independent writes across nodes.
6. Buffer short bursts with durable queues.
7. Apply backpressure or shed low-value load.
8. Batch compatible operations.
9. Aggregate before reaching the final store.
10. Detect and isolate hot keys.
```

---

# 1. Why Writes Are Harder to Scale Than Reads

Reads can often be copied and served from:

- Caches
- Read replicas
- CDN edges
- Materialized views

Writes must usually reach an authoritative owner and satisfy durability, ordering, uniqueness, and consistency requirements.

A write may require:

```text
Validate request
    ↓
Acquire ownership or lock
    ↓
Append WAL or commit log
    ↓
Modify table or memtable
    ↓
Update secondary indexes
    ↓
Replicate to other nodes
    ↓
Publish downstream events
    ↓
Acknowledge client
```

One logical write can therefore produce many physical writes.

## Write Amplification

```text
write amplification
= physical bytes written / logical bytes written
```

Sources include:

- Write-ahead logging
- B-tree page updates and splits
- Secondary indexes
- Replication
- Change-data capture
- LSM flushes and compaction
- Materialized views
- Audit records

The first scaling task is often to reduce this amplification rather than immediately add nodes.

---

# 2. Characterize the Workload First

Before proposing sharding or queues, determine what kind of writes exist.

Measure:

- Average and peak writes per second
- Bytes per write
- Burst duration
- Read-to-write ratio
- Number of indexes updated
- Required durability
- Required acknowledgement latency
- Ordering scope
- Contention per key
- Duplicate-delivery tolerance
- Permitted processing delay
- Whether any writes may be dropped or coalesced

## Capacity Estimate

Suppose the service receives:

```text
20 million devices
× 1 update every 10 seconds
= 2 million writes/second
```

At 300 bytes per logical event:

```text
2,000,000 × 300 bytes
= 600 MB/second logical ingress
```

Replication, indexes, and compaction can multiply the physical bandwidth substantially.

## Peak Versus Average

A service averaging 50,000 writes per second may peak at 250,000 writes per second.

Design for:

```text
steady-state capacity
burst absorption
recovery rate after the burst
```

A queue that absorbs a burst is useful only if consumers later drain the backlog faster than new writes arrive.

---

# 3. Classify Write Semantics

Different writes require different strategies.

## Independent Append

Examples:

- Telemetry event
- Audit log
- Click event

Characteristics:

- Naturally partitionable
- Often tolerant of asynchronous processing
- Good fit for logs, queues, and LSM storage

## Update to One Entity

Examples:

- Profile edit
- Order-state transition
- Document metadata update

Characteristics:

- Requires entity ownership and concurrency control
- Partition by entity ID when possible

## Hot Counter

Examples:

- Viral-post likes
- Video views
- Rate-limit usage

Characteristics:

- Commutative or aggregatable
- One key may become a bottleneck
- Good fit for striped counters and batching

## Cross-Entity Transaction

Examples:

- Account transfer
- Multi-item inventory reservation

Characteristics:

- Hard to shard cleanly
- May require co-location, local transactions, or a multi-step process

## Replaceable State

Examples:

- Latest GPS location
- Current heartbeat
- Current typing state

Characteristics:

- New writes supersede old writes
- Intermediate updates may be coalesced or dropped

### Semantic Principle

> **Optimize from the meaning of the write. An append, counter increment, ownership transition, and replaceable state update are not the same workload.**

---

# 4. Vertical Scaling

Vertical scaling increases the capacity of one write node.

Possible upgrades:

- More CPU cores
- More RAM
- Faster NVMe storage
- Higher provisioned IOPS
- Larger network interfaces
- More capable database instance class

## Why It Helps

### CPU

More cores can increase:

- Transaction processing
- Compression
- Serialization
- Index maintenance
- Replication processing

### RAM

More memory supports:

- Larger buffer pools
- Larger memtables
- Fewer random reads during updates
- More effective batching

### Storage

Faster durable storage reduces:

- WAL flush latency
- Commit latency
- Compaction time
- Checkpoint pressure

### Network

Higher bandwidth supports:

- Larger ingress volume
- Replication
- Cross-node coordination
- Change streams

## Advantages

- Simple operationally
- Preserves transaction model
- Avoids repartitioning
- Fastest way to create headroom

## Limits

- Hardware has a maximum size.
- Cost rises steeply.
- One node remains a throughput and failure boundary.
- Hot-key contention may not improve proportionally.

Vertical scaling should be considered before adding distributed complexity, but it is not an infinite strategy.

---

# 5. Optimize the Existing Write Path

## Reduce Secondary Indexes

Every index affected by a write must also be maintained.

```text
Insert one row
    ↓
Update primary index
Update email index
Update status index
Update created_at index
Update full-text index
```

Remove unused or redundant indexes after verifying query requirements.

## Avoid Expensive Synchronous Work

Move non-critical side effects out of the transaction:

- Email sending
- Search indexing
- Analytics updates
- Recommendation refresh
- Thumbnail generation

Use an outbox or change stream so asynchronous work is not lost.

## Keep Transactions Short

Long transactions:

- Hold locks longer
- Increase conflict probability
- Delay WAL cleanup
- Increase replication lag
- Consume connections

## Use Bulk APIs

Prefer one bulk insert over many round trips:

```sql
INSERT INTO events (event_id, user_id, event_type, created_at)
VALUES
  (...),
  (...),
  (...);
```

## Tune Durability Deliberately

Databases may support group commit, asynchronous commit, or configurable WAL flushing.

Relaxing durability can improve throughput, but it changes the failure guarantee.

State the trade-off explicitly:

```text
Acknowledge after memory append
→ lower latency, possible loss after crash

Acknowledge after durable WAL flush
→ stronger durability, higher latency

Acknowledge after quorum replication
→ stronger failover safety, more network latency
```

### Durability Invariant

> **The acknowledgement point must match the amount of data loss the product can tolerate.**

---

# 6. Choose a Storage Engine That Matches the Workload

## B-Tree-Oriented Relational Storage

Strengths:

- Transactions
- Constraints
- Efficient point reads
- Ordered range queries
- Rich indexing and joins

Write costs:

- In-place page updates
- Page splits
- Secondary-index maintenance
- Random I/O patterns

Best for:

- Balanced workloads
- Strong relational consistency
- Moderate write rates with complex queries

## LSM-Tree Storage

Write path:

```text
WAL
  ↓
Memtable
  ↓
Immutable memtable
  ↓
SSTable
  ↓
Compaction
```

Strengths:

- Sequential WAL append
- Memory-first writes
- Batched flushes
- High ingestion throughput

Costs:

- Read amplification
- Compaction CPU and I/O
- Tombstones
- Space amplification
- More complex latency behavior

Best for:

- Logs
- Metrics
- Time-series data
- Event ingestion
- High-volume key-value writes

## Time-Series Stores

Useful features:

- Time-based partitioning
- Compression and delta encoding
- Retention policies
- Downsampling
- High sequential ingestion

## Columnar Analytical Stores

Good for:

- Batched analytics ingestion
- Large scans
- Compression
- Aggregation

Usually poor for:

- High-frequency small transactional updates
- Row-level transactional semantics

### Selection Principle

> **Choose the datastore from the dominant access pattern, durability requirement, query model, and operational constraints, not from an isolated throughput benchmark.**

---

# 7. Separate Workloads by Access Pattern

One table or database should not necessarily serve every workload.

A social post may contain:

```text
Stable content
Rapidly changing counters
Append-only analytics events
Full-text search representation
```

A monolithic row causes unrelated writes to contend and makes every storage decision a compromise.

## Vertical Partitioning Example

```sql
CREATE TABLE post_content (
    post_id BIGINT PRIMARY KEY,
    author_id BIGINT NOT NULL,
    body TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL
);
```

```sql
CREATE TABLE post_metrics (
    post_id BIGINT PRIMARY KEY,
    like_count BIGINT NOT NULL DEFAULT 0,
    comment_count BIGINT NOT NULL DEFAULT 0,
    view_count BIGINT NOT NULL DEFAULT 0
);
```

```sql
CREATE TABLE post_events (
    post_id BIGINT NOT NULL,
    event_id TEXT NOT NULL,
    event_type TEXT NOT NULL,
    user_id BIGINT,
    event_time TIMESTAMP NOT NULL,
    metadata JSONB
);
```

Storage choices can then differ:

```text
Content   → relational database
Counters  → counter or key-value store
Events    → log or time-series store
Search    → search index
```

## Trade-Off

Reads may need to compose data from several systems. Avoid splitting without a clear workload boundary.

---

# 8. Horizontal Sharding

Sharding distributes records across independent write owners.

```text
Write key
    ↓ hash or range mapping
Shard 1, Shard 2, ... Shard N
```

If one shard handles 20,000 writes per second, ten evenly loaded shards may approach 200,000 writes per second, subject to coordination, network, and workload overhead.

## Hash Partitioning

```text
shard = hash(user_id) mod shard_count
```

Benefits:

- Often distributes random identifiers evenly
- Simple point routing

Costs:

- Range queries scatter
- Changing shard count may move substantial data

## Range Partitioning

```text
A–F → shard 1
G–M → shard 2
N–Z → shard 3
```

Benefits:

- Efficient range scans
- Natural time or geography grouping

Costs:

- Uneven ranges
- Monotonically increasing keys can create a hot latest range

## Directory-Based Partitioning

A metadata service maps entity to shard:

```text
customer 842 → shard 17
```

Benefits:

- Flexible placement
- Easier targeted movement

Costs:

- Directory availability and caching
- Extra lookup or routing state

---

# 9. Selecting a Partition Key

A good partition key should:

- Spread write volume evenly
- Keep commonly accessed data together
- Avoid unbounded partition growth
- Support important ordering requirements
- Minimize cross-shard transactions
- Minimize scatter-gather reads

## Good Candidate

```text
hash(user_id)
```

when each user's writes are roughly independent and user activity is reasonably distributed.

## Poor Candidate

```text
country
```

when one country generates far more writes than others.

## Another Poor Candidate

```text
current_date
```

This sends all current writes to one time partition.

## Composite Bucketing

For time-series data:

```text
partition_key = hash(device_id) + time_bucket
```

This can distribute devices while bounding partition size.

### Partition-Key Invariant

> **The partition key must distribute the actual write rate, not merely the record count.**

---

# 10. Data Locality and Cross-Shard Cost

Sharding distributes load but can make reads and transactions expensive.

Ask:

```text
How many shards does the dominant write touch?
How many shards does the dominant read touch?
What ordering scope is required?
Which entities transact together?
```

## Co-Locate Related Data

For a messaging system:

```text
partition by conversation_id
```

This keeps message ordering and writes for one conversation on one partition.

## Scatter-Gather

A global query may contact every shard:

```text
Coordinator
    ├── shard 1
    ├── shard 2
    ├── shard 3
    └── shard N
```

Tail latency, partial failure, and merge cost grow with shard count.

Use precomputed global indexes or analytical stores for frequent global queries rather than scatter-gathering transactional shards repeatedly.

---

# 11. Consistent Hashing, Slots, and Virtual Nodes

## Fixed Slots

Map keys to stable logical slots:

```text
slot = hash(key) mod 16384
```

Assign slots to physical nodes. Rebalancing moves slots rather than changing every key's formula.

## Consistent Hash Ring

Map nodes and keys to a ring. A key belongs to the next node clockwise.

When nodes change, only a portion of keys move.

## Virtual Nodes

Each physical node owns several virtual positions.

Benefits:

- Better distribution
- Easier weighted capacity
- More gradual rebalancing

These schemes solve routing and movement. They do not automatically solve a single hot key.

---

# 12. Resharding Without Downtime

Suppose the system expands from 8 to 16 shards.

A naive full stop and rehash causes unacceptable downtime.

## Gradual Migration

```text
1. Compute old and new ownership.
2. Start copying historical data.
3. Capture concurrent changes.
4. Route or duplicate writes during migration.
5. Verify destination completeness.
6. Switch reads to the new owner.
7. Stop old writes.
8. Remove old copy after a safety window.
```

## Dual Write Risks

Writing old and new shards independently can diverge if one write succeeds and the other fails.

Safer techniques include:

- Write authoritative source plus CDC to destination
- Durable migration log
- Idempotent copy operations
- Per-record versions
- Read repair
- Reconciliation scans

## Read Strategy During Migration

Possible policy:

```text
Read new shard first.
If missing or behind expected version, read old shard.
Repair new shard asynchronously.
```

## Verification

Compare:

- Record counts
- Checksums
- Version watermarks
- Missing-key samples
- Write-log offsets

### Migration Invariant

> **Every write acknowledged during migration must appear in the final owner exactly once logically, even if transfer attempts repeat.**

---

# 13. Replication Is Not Write Scaling by Itself

Replicas improve durability and read capacity, but every replica may need to process each write.

```text
1 logical write
    ↓
leader + 2 followers
    ↓
3 physical copies
```

Replication increases write work.

## Quorum Writes

With replication factor `N`, a write may require acknowledgements from `W` replicas.

Higher `W` can improve durability and consistency but increases latency and may reduce availability during failure.

## Multi-Leader Writes

Multiple regions accept writes.

Benefits:

- Lower regional write latency
- Regional write availability

Costs:

- Conflicting concurrent writes
- Ordering ambiguity
- Replication loops
- Conflict resolution

Use multi-leader architecture only when the domain has a clear conflict strategy.

---

# 14. Durable Queues for Burst Absorption

A queue separates write acceptance from downstream processing.

```text
Client
  ↓
API validates and appends command/event
  ↓
Durable queue or log
  ↓
Workers
  ↓
Database or downstream stores
```

The API may acknowledge after durable queue admission rather than final database completion.

## Benefits

- Absorbs temporary traffic spikes
- Smooths database load
- Decouples producer and consumer scaling
- Enables retries
- Preserves ordering within partitions
- Supports replay

## Costs

- Eventual consistency
- Delayed visibility
- Duplicate delivery
- Backlog management
- More operational infrastructure
- Client status tracking

## Client Contract

An asynchronous write may return:

```http
202 Accepted
Location: /operations/op-842
```

The client can poll or subscribe for completion.

---

# 15. A Queue Does Not Create Capacity

Let:

```text
arrival rate = λ
processing rate = μ
```

If:

```text
λ < μ
```

then the queue can recover after a finite burst.

If:

```text
λ > μ continuously
```

the backlog grows without bound.

Approximate backlog growth:

```text
backlog increase per second = λ - μ
```

A queue is appropriate for burst mismatch, not permanent under-capacity.

## Drain-Time Estimate

Suppose:

```text
burst creates 10 million queued items
normal arrival = 50,000/s
consumer capacity = 70,000/s
```

Spare drain capacity:

```text
70,000 - 50,000 = 20,000/s
```

Drain time:

```text
10,000,000 / 20,000 = 500 seconds
```

about 8.3 minutes.

---

# 16. Backpressure

Backpressure tells upstream producers that downstream capacity is constrained.

Mechanisms include:

- Bounded queues
- HTTP 429 responses
- Producer throttling
- Consumer lag feedback
- Lower concurrency limits
- Adaptive rate limits
- Pausing partition consumption
- Retry-After headers

Without backpressure:

```text
Downstream slows
    ↓
Queues grow
    ↓
Memory and storage fill
    ↓
Timeouts trigger retries
    ↓
Load increases further
```

This creates a positive feedback loop and can collapse the system.

### Backpressure Invariant

> **No layer may accept unbounded work that the next layer cannot eventually process within the product's latency budget.**

---

# 17. Load Shedding

Load shedding intentionally rejects, samples, coalesces, or drops work to preserve the health of higher-value operations.

This is better than allowing every request to time out unpredictably.

## Good Candidates for Dropping or Coalescing

- Intermediate GPS updates
- Repeated heartbeats
- Typing indicators
- Duplicate impressions
- Low-priority analytics
- Obsolete progress updates

## Poor Candidates

- Financial ledger entries
- Purchases
- Security audit records
- Unique user-generated content
- Inventory ownership transitions

## Latest-Value Coalescing

For replaceable location state:

```text
location at t1
location at t2
location at t3
```

If only the latest location matters, retain `t3` and discard older unprocessed positions.

## Priority Shedding

```text
Priority 1: purchases and payments
Priority 2: user content
Priority 3: engagement events
Priority 4: diagnostic analytics
```

During overload, reject lower priorities first.

## Sampling

For observability or analytics:

```text
store 1 percent of low-value impressions
store 100 percent of purchases
```

The sampling rate must be recorded so aggregates can be interpreted correctly.

---

# 18. Idempotency and Duplicate Processing

Queues, retries, and network failures produce duplicate attempts.

Every logical write should carry a stable identifier:

```text
operation_id = click-123
order_id = order-842
location_sequence = 9918
```

## Database Deduplication

```sql
CREATE UNIQUE INDEX unique_event_id
ON events(event_id);
```

or:

```sql
INSERT INTO processed_operations(operation_id)
VALUES (:operation_id)
ON CONFLICT DO NOTHING;
```

Apply the business mutation in the same transaction when possible.

## Sequence-Based Replacement

For latest state:

```sql
UPDATE device_state
SET location = :location,
    sequence_number = :sequence
WHERE device_id = :device_id
  AND sequence_number < :sequence;
```

Older duplicates cannot overwrite newer state.

### Idempotency Invariant

> **At-least-once delivery must not create more than one logical business effect.**

---

# 19. Ordering

Global ordering is expensive and often unnecessary.

Choose the smallest required ordering scope:

- Per user
- Per device
- Per conversation
- Per account
- Per post
- Per partition

## Partitioned Ordering

```text
partition = hash(conversation_id)
```

All messages for one conversation enter the same ordered partition.

Different conversations process concurrently.

## Reordering at Consumer

If events carry sequence numbers, the consumer can:

- Reject older state
- Buffer small gaps
- Request replay
- Mark missing data for reconciliation

### Ordering Principle

> **Preserve order only where the business invariant requires it; allow unrelated entities to proceed independently.**

---

# 20. Batching Writes

Batching combines several logical writes into fewer physical operations.

Instead of:

```text
1000 network calls
1000 transactions
1000 commit flushes
```

use:

```text
10 batches of 100 writes
```

Benefits:

- Fewer network round trips
- Fewer transaction setups
- Better sequential I/O
- Better compression
- More efficient index updates
- Group commit

## Batch Triggers

Flush when either condition is reached:

```text
batch_size >= 500
or
oldest_item_age >= 100 ms
```

This balances throughput and latency.

## Trade-Offs

- Adds waiting latency
- Increases memory use
- Creates partial-batch failure questions
- Requires retry and deduplication
- Very large batches can cause long transactions and lock pressure

### Batch Principle

> **Choose batch size from throughput gain, maximum acceptable delay, payload limits, and failure-recovery cost.**

---

# 21. Safe Application-Layer Batching

If the application acknowledges data before it is durable, a crash may lose the in-memory batch.

Unsafe flow:

```text
Client request acknowledged
    ↓
Write held only in process memory
    ↓
Process crashes
```

Safer designs:

- Append to a durable queue before acknowledging.
- Write to a local durable log.
- Let the client retry with an idempotency key.
- Use database-native bulk operations synchronously.

If a durable log is the source of truth, workers can replay uncommitted batches after failure.

---

# 22. Aggregating Commutative Updates

Some operations can be combined mathematically.

Example events:

```text
post 4: +1 like
post 5: +1 like
post 4: +1 like
post 4: +1 like
```

Aggregate to:

```text
post 4: +3 likes
post 5: +1 like
```

Three database updates for post 4 become one.

## Suitable Operations

- Sum
- Count
- Minimum
- Maximum
- Histogram bucket increments
- Set union under controlled semantics
- Latest value by timestamp or sequence

## Unsuitable Operations

- Non-commutative state transitions
- Unique ownership changes
- Operations requiring exact individual history unless raw events are retained

## Windowing

Aggregate over:

- Count window
- Time window
- Memory threshold
- Queue partition batch

The longer the window, the greater the compression and the higher the visibility delay.

---

# 23. Counter Sharding and Hot-Key Splitting

One viral item can overwhelm its shard even when the overall partitioning is balanced.

Instead of one counter:

```text
post:42:likes
```

use `K` stripes:

```text
post:42:likes:0
post:42:likes:1
...
post:42:likes:K-1
```

Writers select a stripe:

```text
stripe = hash(user_id or request_id) mod K
```

Read total:

```text
sum(all K stripes)
```

## Benefits

- Approximately divides write contention by `K`
- Preserves parallelism
- Simple for commutative counters

## Costs

- Read amplification by `K`
- More storage and metadata
- Reconciliation complexity
- Changing `K` requires coordination

The logical data volume is not necessarily multiplied by `K`; metadata and row overhead increase, while the count value is distributed. Replicating full data would multiply storage more substantially.

---

# 24. Dynamic Hot-Key Splitting

Splitting every key wastes read work when most keys are cold.

A dynamic approach marks only hot keys as striped.

Metadata:

```text
key = post:42:likes
stripe_count = 100
version = 3
```

Writers and readers must agree on the stripe count and version.

## Transition Problem

Changing from 1 to 100 stripes can lose or double-count updates if writers and readers switch inconsistently.

Safer transition:

```text
1. Publish new stripe configuration with version.
2. Writers begin writing to new version.
3. Readers temporarily read old and new versions.
4. Drain or merge old stripes.
5. Retire old version after a watermark.
```

For interviews, fixed striping is often sufficient unless dynamic splitting is a core challenge.

### Hot-Key Invariant

> **All readers and writers must share a consistent interpretation of the key's split configuration.**

---

# 25. Hierarchical Aggregation

At extreme scale, reduce data volume in stages.

```text
Clients
    ↓
Edge or regional aggregators
    ↓
Partition aggregators
    ↓
Global reducer
    ↓
Durable aggregate store
```

Example for live reactions:

```text
Viewer likes
    ↓
Regional processor counts likes per comment for 1 second
    ↓
Global processor merges regional deltas
    ↓
Broadcast nodes distribute updated counts
```

## Benefits

- Reduces fan-in at the root
- Reduces writes to final storage
- Allows regional processing
- Supports incremental aggregation
- Combines naturally with broadcast trees

## Costs

- Additional latency
- Partial aggregate failure
- Duplicate delta handling
- Window alignment
- Late data
- Reconciliation requirements

## Delta Identity

Every aggregate delta should have a stable identity or range:

```text
processor_id
window_start
window_end
sequence_number
```

This allows the reducer to deduplicate retries.

---

# 26. Fan-In and Fan-Out

Some systems face both:

```text
Millions of users produce events
and
Millions of users consume the merged view
```

A single root cannot directly receive and rebroadcast every event.

Use two trees:

```text
Write tree:
clients → local reducers → regional reducers → root

Broadcast tree:
root → regional broadcast nodes → connection servers → clients
```

The write path aggregates. The read or push path distributes.

This architecture is useful when users need a shared, eventually consistent view rather than every raw event.

---

# 27. Time Bucketing

Time-series writes can be partitioned by time windows:

```text
device_id + hour
metric_id + minute
customer_id + day
```

Benefits:

- Bounded partition size
- Efficient retention deletion
- Natural range scans
- Parallel compaction and archival

Risk:

```text
partition by current hour only
```

All writes may target one hot partition.

Combine time with a distribution key:

```text
hash(device_id) + hour_bucket
```

---

# 28. Storage Lifecycle and Tiering

High write rates create large datasets.

Not all history needs expensive primary storage.

Lifecycle:

```text
Hot recent data
    ↓
Warm compressed data
    ↓
Cold object storage
    ↓
Deletion by retention policy
```

Examples:

- Keep raw metrics for 7 days.
- Keep 1-minute aggregates for 90 days.
- Keep hourly aggregates for 2 years.

Tiering controls cost and reduces active dataset size, indirectly improving write and compaction performance.

---

# 29. Indexing and Search Pipelines

Search ingestion is often asynchronous.

```text
Primary database commit
    ↓
Transactional outbox or CDC
    ↓
Indexing queue
    ↓
Document transformation
    ↓
Search index bulk write
```

Benefits:

- Search indexing does not slow the primary transaction.
- Indexers can batch documents.
- Failed transformations can retry.
- The search index can be rebuilt from durable source data.

The search index is usually a derived read model, not the authoritative source.

### Projection Invariant

> **Every asynchronous write projection must be rebuildable or repairable from an authoritative log or store.**

---

# 30. Transactional Outbox for Reliable Downstream Writes

A service may need to update its database and publish an event.

Doing them independently creates a dual-write problem.

Use one local transaction:

```sql
BEGIN;

UPDATE orders
SET status = 'PAID'
WHERE order_id = :order_id;

INSERT INTO outbox(event_id, event_type, aggregate_id, payload)
VALUES (:event_id, 'OrderPaid', :order_id, :payload);

COMMIT;
```

A relay publishes outbox events to the queue.

Because publishing may repeat, consumers must deduplicate.

This pattern reliably feeds search, analytics, notifications, and read projections without extending the primary transaction across services.

---

# 31. Contention on a Hot Row

Sharding by entity does not solve concurrent writes to one entity.

Examples:

- One auction
- One celebrity counter
- One inventory item
- One rate-limit bucket

Possible strategies:

- Conditional atomic update
- Optimistic concurrency
- Pessimistic lock
- Queue serialization by key
- Counter striping
- Aggregation
- Product-level admission control

The correct choice depends on whether the operation is commutative and whether exact ordering is required.

## Serialization Point

Strongly consistent modification of one indivisible resource requires a serialization point somewhere.

More servers cannot remove that requirement. They can only move or manage it.

---

# 32. Queue Serialization for Ordered Hot Writes

Route all commands for one entity to the same ordered partition:

```text
partition = hash(auction_id)
```

One logical consumer processes that partition in order.

Benefits:

- Eliminates concurrent updates for that key
- Simplifies ordering
- Protects database from conflict storms

Costs:

- Per-key throughput is bounded by one serial stream
- Queue delay
- Consumer failover
- Idempotent processing still required

Use this for auctions or other ordered state machines. Use striping for commutative counters.

---

# 33. Autoscaling and Why It Is Not Enough

Autoscaling helps when stateless workers are the bottleneck.

It is less effective when:

- Scaling takes longer than the traffic spike.
- The database remains fixed.
- New workers increase pressure on a saturated store.
- Rebalancing partitions temporarily reduces capacity.
- Warm-up is expensive.

Autoscaling should be combined with:

- Queue buffering
- Backpressure
- Rate limits
- Pre-provisioned headroom
- Predictive scaling for known events
- Load shedding

More producers against the same full database can make an incident worse.

---

# 34. Retry Storms

When overloaded dependencies time out, clients and workers often retry.

```text
Service slows
    ↓
Requests time out
    ↓
Callers retry
    ↓
Traffic increases
    ↓
Service slows further
```

Mitigations:

- Exponential backoff
- Jitter
- Retry budgets
- Idempotency
- Circuit breakers
- Deadline propagation
- Load shedding
- Queue visibility timeouts sized to processing time

## Retry Budget

Limit retry traffic as a fraction of normal traffic.

Do not let every client retry indefinitely.

### Retry Invariant

> **A failure must not multiply offered load faster than the system can recover.**

---

# 35. Client-Side and Server-Side Rate Limiting

Rate limits protect write paths from abusive or accidental overload.

Possible dimensions:

- Per user
- Per API key
- Per tenant
- Per IP
- Per resource
- Global

Server response:

```http
429 Too Many Requests
Retry-After: 10
```

For replaceable writes, the client can retry only the latest state rather than every rejected update.

Rate limiting protects fairness and capacity. It does not replace internal backpressure or partitioning.

---

# 36. Large Payloads and Blob Separation

Writing large media files through the application and database wastes CPU, network, and transaction capacity.

Better flow:

```text
Client requests upload URL
    ↓
Client uploads directly to object storage
    ↓
Client or storage event reports completion
    ↓
Application writes small metadata row
```

Benefits:

- Removes large payloads from database writes
- Uses object storage bandwidth
- Enables multipart upload
- Reduces application-server memory

The metadata transition should verify that the referenced object exists and belongs to the caller.

---

# 37. Compression

Compression reduces disk and network bytes but consumes CPU.

Useful for:

- Repetitive telemetry
- Columnar batches
- Network replication
- Archived events

Poor fit when:

- Payloads are tiny
- CPU is already saturated
- Data is already compressed

Batching usually improves compression ratio because the codec sees more repeated structure.

---

# 38. Write Failure Semantics

Define what acknowledgement means.

Possible states:

```text
RECEIVED
QUEUED
PERSISTED
REPLICATED
PROJECTED
COMPLETED
```

A `202 Accepted` may mean only that the command is durably queued.

A `201 Created` may mean the authoritative entity exists.

Do not tell the client a write completed when only an ephemeral application buffer contains it.

## Operation Status

For asynchronous writes:

```http
POST /exports
→ 202 Accepted
→ operation_id = op-842
```

The client can query:

```http
GET /operations/op-842
```

or receive a push notification on completion.

---

# 39. Poison Messages and Dead-Letter Handling

Some queued writes fail repeatedly because of malformed data or permanent business errors.

If retried forever, they block partitions or waste capacity.

Use:

- Error classification
- Maximum attempts
- Dead-letter queue
- Quarantine store
- Alerting
- Repair and replay tooling

A dead-letter queue is not a disposal bin. It requires ownership, dashboards, retention, and replay procedures.

---

# 40. Reconciliation

Distributed write pipelines can diverge despite retries.

Reconciliation compares derived state against authoritative state.

Examples:

- Search document missing for a committed post
- Aggregate count differs from raw events
- Destination shard missing a migrated record
- Queue offset advanced but projection incomplete

Techniques:

- Periodic checksum comparison
- Count comparison
- Version watermarks
- Rebuild from event log
- Read repair
- Audit sampling

### Repair Invariant

> **Every asynchronously maintained state has a way to detect and repair divergence.**

---

# 41. Common Product Scenarios

## Metrics Ingestion

Use:

- Time-bucketed partitioning
- Durable log
- Batch writes
- Local or regional aggregation
- Downsampling
- Retention tiers
- Sampling under overload

## Ad Click Aggregation

Use:

- Stable event IDs
- Partition by campaign or ad with hot-key protection
- Durable event log
- Windowed aggregation
- Separate raw and aggregate storage
- Reconciliation for billing

Clicks that affect billing require stronger durability than low-value impressions.

## Location Tracking

Use:

- Partition by driver or device
- Latest sequence number
- Coalescing of obsolete updates
- Load shedding for over-frequent reports
- Time-series history at lower frequency

## Social Likes

Use:

- Raw idempotent like relation for correctness when needed
- Striped counters for display total
- Batch aggregation
- Eventual reconciliation

## Search Indexing

Use:

- Database as authority
- Outbox or CDC
- Partitioned indexing queue
- Bulk indexing
- Rebuild capability

## Notification Generation

Use:

- Durable queue
- Partition by recipient or campaign
- Per-channel worker pools
- Rate limits
- Idempotency
- Batch provider APIs
- Fan-out trees for massive broadcasts

## Live Comments

Use:

- Partition by stream or comment ID
- Regional write processors
- Windowed aggregation for reactions
- Root merge
- Broadcast hierarchy

---

# 42. When Not to Add Write-Scaling Infrastructure

Avoid premature sharding, queues, and aggregation when:

- One database comfortably handles peak throughput.
- Writes require immediate synchronous confirmation.
- The workload is primarily read-heavy.
- Delayed visibility is unacceptable.
- The data volume is small.
- Operational complexity would exceed the expected benefit.

A queue, shard map, or dual-write migration creates real failure modes. Add them because measurements and requirements justify them.

---

# 43. Interview Design Process

## Step 1: Estimate the Write Load

Calculate:

```text
writers
× writes per writer per second
× peak factor
× bytes per write
```

Include physical amplification from replication and indexes.

## Step 2: Define Semantics

For each write:

- Is it append, replace, increment, or transition?
- Must it be durable before acknowledgement?
- Must it be ordered?
- Can it be delayed?
- Can it be dropped or coalesced?
- Is it idempotent?

## Step 3: Optimize One Node

- Reduce indexes
- Shorten transactions
- Use bulk operations
- Move side effects asynchronous
- Tune WAL and group commit carefully
- Scale CPU, memory, disk, and network

## Step 4: Select Storage

Explain why the chosen engine matches:

- Write pattern
- Read pattern
- Transaction needs
- Retention
- Query shape

## Step 5: Partition

State:

- Partition key
- Routing mechanism
- Expected distribution
- Ordering scope
- Cross-shard operations
- Hot-key plan

## Step 6: Handle Bursts

Use:

- Durable queue
- Consumer autoscaling
- Backpressure
- Status API
- Retry and dead-letter policy

Prove that the backlog can drain.

## Step 7: Reduce Work

Consider:

- Batching
- Coalescing
- Sampling
- Hierarchical aggregation
- Direct object-store uploads

## Step 8: Plan Operations

Discuss:

- Resharding
- Reconciliation
- Queue lag
- Retry storms
- Load shedding
- Capacity alarms
- Disaster recovery

---

# 44. Interview Answer Template

> I would first quantify average and peak write QPS, payload size, durability, ordering scope, contention, and permitted processing delay. I would reduce write amplification by removing unnecessary indexes, shortening transactions, using bulk operations, and moving non-critical projections behind an outbox. I would scale the primary vertically and choose a storage engine that matches the workload, such as an LSM-based store for append-heavy telemetry. Once one node is insufficient, I would partition by a high-cardinality key that distributes actual write rate while co-locating operations that require ordering. Short bursts would enter a durable queue, with bounded backlog, idempotent consumers, backpressure, and a status API. Replaceable or low-value updates could be coalesced, sampled, or shed under overload. Commutative counters would be striped and aggregated in windows, while ordered hot entities would be serialized through one queue partition. I would also explain resharding, dual-write avoidance, reconciliation, retry budgets, and the acknowledgement point for durability.

---

# 45. Common Mistakes

## Mistake 1: Use a Queue for Permanent Under-Capacity

If producers remain faster than consumers, the queue only delays failure.

## Mistake 2: Choose a Low-Cardinality Partition Key

Country, status, or current date may create severe skew.

## Mistake 3: Ignore Read Patterns

Perfectly even writes can produce expensive scatter-gather reads.

## Mistake 4: Assume Replication Multiplies Write Throughput

Replication often multiplies physical write work.

## Mistake 5: Add Workers Against a Saturated Database

Autoscaling consumers can increase overload.

## Mistake 6: Batch in Volatile Memory After Acknowledgement

A process crash can lose accepted writes.

## Mistake 7: Retry Without Idempotency

Duplicates become double charges, counts, or notifications.

## Mistake 8: Striping Non-Commutative State

Unique ownership and ordered state transitions cannot be safely summed later.

## Mistake 9: Dual Write During Resharding Without Repair

Old and new shards can diverge.

## Mistake 10: Drop Writes Without Business Classification

Load shedding must distinguish replaceable telemetry from durable business events.

## Mistake 11: Ignore Hot Keys

Even a well-balanced shard map cannot distribute one indivisible key automatically.

## Mistake 12: Claim Exactly-Once Delivery

Use at-least-once delivery plus idempotent logical effects and reconciliation.

---

# 46. Operational Metrics

## Ingress

- Accepted writes per second
- Rejected writes
- Payload bytes per second
- Peak-to-average ratio
- Rate-limit events

## Database

- Commit latency
- WAL bytes per second
- WAL flush latency
- Disk IOPS and throughput
- CPU utilization
- Lock waits
- Deadlocks
- Index-write volume
- Checkpoint duration
- Replication lag
- Compaction backlog

## Queue

- Produce latency
- Consumer lag
- Queue depth
- Oldest-message age
- Retry count
- Dead-letter count
- Drain rate
- Partition skew

## Partitioning

- Writes per shard
- Storage per shard
- Hottest keys
- Cross-shard request rate
- Migration progress
- Rebalance duration

## Batching and Aggregation

- Average batch size
- Batch wait time
- Compression ratio
- Raw-to-aggregate reduction ratio
- Late-event count
- Duplicate-delta count

## Reliability

- Deduplication conflicts
- Reconciliation mismatches
- Lost-write incidents
- Load-shed volume
- Recovery time

---

# 47. Important Invariants

## Durability Invariant

> **An acknowledged write exists in storage that satisfies the promised failure tolerance.**

## Idempotency Invariant

> **Repeated attempts create one logical effect.**

## Partitioning Invariant

> **Every key has one well-defined write owner or conflict-resolution rule.**

## Ordering Invariant

> **Operations requiring order are routed through the same ordered scope.**

## Backpressure Invariant

> **No component accepts unbounded work from a faster upstream producer.**

## Hot-Key Invariant

> **One popular key cannot consume unbounded capacity or destabilize unrelated keys.**

## Migration Invariant

> **Resharding preserves every acknowledged logical write through retries and failures.**

## Projection Invariant

> **Every derived store can be rebuilt or reconciled from authoritative data.**

## Load-Shedding Invariant

> **Only writes whose loss or replacement is explicitly acceptable may be dropped or coalesced.**

## Queue Invariant

> **Backlog age remains within the product's maximum processing delay and can return to zero after a burst.**

---

# 48. Quick Comparison

## Vertical Scaling

```text
Best for:
Creating simple headroom on one node

Benefit:
Low architectural complexity

Cost:
Finite hardware ceiling
```

## Write-Optimized Storage

```text
Best for:
Append-heavy ingestion

Benefit:
Sequential and batched disk writes

Cost:
Read and compaction trade-offs
```

## Sharding

```text
Best for:
Independent writes beyond one node's capacity

Benefit:
Parallel write ownership

Cost:
Routing, rebalancing, and cross-shard complexity
```

## Durable Queue

```text
Best for:
Short bursts and asynchronous processing

Benefit:
Burst smoothing and decoupling

Cost:
Delay, duplicates, and backlog management
```

## Load Shedding

```text
Best for:
Replaceable or lower-value writes during overload

Benefit:
Protects critical capacity

Cost:
Intentional data loss or reduced fidelity
```

## Batching

```text
Best for:
Many small writes

Benefit:
Amortized network, transaction, and flush overhead

Cost:
Added latency and batch recovery
```

## Counter Striping

```text
Best for:
Hot commutative counters

Benefit:
Parallel writes to one logical metric

Cost:
Read amplification and configuration coordination
```

## Hierarchical Aggregation

```text
Best for:
Massive metrics, reactions, and shared aggregates

Benefit:
Reduces data volume at each stage

Cost:
Latency and reconciliation complexity
```

---

# 49. Thirty-Second Revision

- **Write scaling:** reduce write work per component while preserving required durability and order.
- **First step:** estimate QPS, bytes, peak factor, amplification, and acknowledgement semantics.
- **Vertical scaling:** more CPU, RAM, IOPS, and network before distributed complexity.
- **Write amplification:** one logical write may update WAL, indexes, replicas, and projections.
- **Storage choice:** B-tree for balanced transactional access; LSM for high append throughput.
- **Vertical partitioning:** separate stable content, hot counters, analytics, and search projections.
- **Sharding:** distribute independent records across write owners.
- **Partition key:** distribute actual write rate and preserve important locality.
- **Resharding:** copy gradually, capture concurrent changes, verify, and cut over safely.
- **Queue:** absorbs temporary bursts but does not create steady-state capacity.
- **Backpressure:** slow or reject producers before queues become unbounded.
- **Load shedding:** drop, sample, or coalesce only explicitly expendable writes.
- **Idempotency:** duplicate attempts must create one logical effect.
- **Batching:** amortize network, transaction, index, and flush overhead.
- **Aggregation:** combine commutative updates before final storage.
- **Striped counter:** split one hot counter across several subkeys and sum on read.
- **Queue serialization:** ordered hot entities use one logical processing stream.
- **Hierarchical aggregation:** local reducers feed regional and global reducers.
- **Outbox:** commit business state and event-publication intent atomically.
- **Reconciliation:** detect and repair divergence in asynchronous projections.
- **Retries:** use backoff, jitter, deadlines, retry budgets, and idempotency.
- **Best principle:** partition independent work, serialize only what must be ordered, and eliminate writes that do not need to reach the final store individually.

## Final Mental Model

```text
Incoming write
    ↓
What semantic type is it?
    append | replace | increment | state transition
    ↓
Can one optimized node handle peak load?
    ├── Yes → keep it simple
    └── No
        ↓
Can independent entities be partitioned?
    ├── Yes → shard by a balanced locality-aware key
    └── No → identify the serialization point
        ↓
Is the excess temporary?
    ├── Yes → durable queue and backpressure
    └── No → add sustained processing capacity
        ↓
Can writes be combined or replaced?
    ├── Batch
    ├── Aggregate
    ├── Coalesce latest state
    └── Shed low-value events
        ↓
Is one key still hot?
    ├── Commutative → stripe and aggregate
    └── Ordered → serialize by key
        ↓
At every stage ask:
    What is durable?
    What is ordered?
    What can repeat?
    What can wait?
    What can be dropped?
    How is divergence repaired?
```
