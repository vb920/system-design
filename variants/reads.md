# Scaling Reads: System Design Study Guide

## Core Thesis

**Scaling reads means serving growing query traffic without allowing latency, cost, or database load to become unacceptable.**

Many applications are naturally read-heavy:

- A post is written once and viewed millions of times.
- A product is created once and browsed repeatedly.
- A video is uploaded once and its metadata is fetched for every view.
- A short URL is created once and redirected many times.
- A news article is published once and delivered to many readers.

A common read-to-write ratio is:

```text
10 reads : 1 write
```

Content-heavy systems may reach:

```text
100 reads : 1 write
```

or much higher.

The natural progression is:

```text
1. Measure the read path.
2. Optimize queries and indexes.
3. Improve the data model and precompute expensive results.
4. Scale the database with replicas or partitions.
5. Add application caching.
6. Add CDN and edge caching.
7. Protect every layer from hot keys, stampedes, and stale data.
```

The primary invariant is:

> **Every read optimization must preserve the freshness, consistency, privacy, and availability guarantees required by that specific data.**

---

# 1. Why Read Traffic Becomes a Bottleneck

Consider a social-media feed containing 25 posts.

Rendering the page may require:

```text
1 feed query
25 post metadata reads
25 author-profile reads
25 like-count reads
25 comment-preview reads
25 media URL reads
```

A naive implementation may execute more than 100 reads for one page view.

Now suppose the service handles 20,000 feed requests per second:

```text
20,000 page requests/second
× 100 database reads/page
= 2,000,000 reads/second
```

The problem is not only the number of users. It is also **read amplification**, where one user action produces many internal reads.

## Physical Limits

A database eventually reaches limits in:

- CPU instructions
- Memory and buffer-pool capacity
- Disk IOPS
- Disk throughput
- Network bandwidth
- Connection count
- Lock and latch contention
- Query-planner and execution overhead

Once these resources are saturated, additional application code cannot create more physical capacity. The architecture must reduce work per read or distribute the work.

---

# 2. Read Scaling Versus Read Latency

These goals overlap but are not identical.

## Read Scaling

Goal:

```text
Handle more total read traffic.
```

Typical mechanisms:

- Better queries
- Read replicas
- Caches
- Partitioning
- Precomputation

## Read Latency

Goal:

```text
Return one read more quickly.
```

Typical mechanisms:

- Indexes
- In-memory caching
- CDN edges
- Geographic placement
- Smaller payloads
- Connection reuse

A database may handle current traffic safely while still providing poor latency to distant users. In that case, edge delivery or regional architecture may matter more than database throughput.

---

# 3. Begin With Measurement

Do not add replicas or caches before identifying the actual bottleneck.

Measure:

- Requests per second by endpoint
- Read-to-write ratio
- Queries per request
- Query latency percentiles
- Rows examined versus rows returned
- Database CPU and I/O
- Buffer-pool hit ratio
- Connection-pool saturation
- Slow-query frequency
- Cache hit ratio
- Replica lag
- Response payload size
- Dependency fan-out

## Percentiles

Average latency can hide a bad user experience.

Track:

```text
p50: typical request
p95: slower 5 percent
p99: slower 1 percent
p99.9: extreme tail
```

Tail latency often grows first when queues form or cache misses trigger expensive database work.

## Query Plans

Use database query-plan tools:

```sql
EXPLAIN
SELECT ...;
```

and, when safe:

```sql
EXPLAIN ANALYZE
SELECT ...;
```

Inspect:

- Sequential scans
- Index scans
- Index-only scans
- Join strategies
- Sort operations
- Estimated rows
- Actual rows
- Buffers or pages read
- Temporary disk usage

---

# 4. Reduce Reads Before Scaling Them

The cheapest read is the read the system does not perform.

## Common Sources of Unnecessary Reads

- N+1 query patterns
- Re-fetching the same entity several times in one request
- Returning columns the client does not need
- Loading complete collections without pagination
- Recomputing stable aggregates
- Repeated permission or configuration lookups
- Fetching large blobs through the transactional database
- Calling several services for data that could be composed earlier

## N+1 Example

Unsafe pattern:

```text
1 query to fetch 100 posts
100 queries to fetch each author
```

Total:

```text
101 queries
```

Better options:

- Join posts to authors when appropriate.
- Batch author IDs with an `IN` query.
- Use a request-scoped data loader.
- Store the required author snapshot with each post.
- Cache author profiles.

## Request-Scoped Deduplication

If several parts of one request need the same user profile, load it once and reuse the result inside that request.

This is different from a shared distributed cache. It has no cross-request invalidation problem and should often be used first.

---

# 5. Indexing

An index is an auxiliary structure that helps the database find relevant rows without scanning the entire table.

Without an appropriate index:

```text
Read every table page
    ↓
Test every row
    ↓
Return the match
```

With an index:

```text
Traverse index
    ↓
Locate candidate row or page
    ↓
Fetch only relevant data
```

## Example

```sql
SELECT *
FROM users
WHERE email = 'user@example.com';
```

Possible index:

```sql
CREATE UNIQUE INDEX idx_users_email
ON users(email);
```

## Index Candidates

Index columns frequently used for:

- Selective filters
- Joins
- Ordering
- Uniqueness
- Range predicates
- Composite access patterns

## Composite Index

For:

```sql
SELECT id, created_at, total
FROM orders
WHERE customer_id = :customer_id
  AND created_at >= :start_time
ORDER BY created_at DESC
LIMIT 50;
```

A useful index may be:

```sql
CREATE INDEX idx_orders_customer_time
ON orders(customer_id, created_at DESC);
```

Column order matters because a composite B-tree is primarily searchable from its leading columns.

## Covering Index

If a frequent query needs only a few columns, the index can include them so the database may answer without visiting the main table.

```sql
CREATE INDEX idx_orders_customer_time_cover
ON orders(customer_id, created_at DESC)
INCLUDE (total, status);
```

## Index Costs

Indexes consume:

- Storage
- Buffer-pool memory
- Write I/O
- WAL or replication bandwidth
- Maintenance time

The correct principle is not “index everything.” It is:

> **Create indexes for important query patterns and verify them through query plans and production metrics.**

---

# 6. Pagination

Returning an unbounded result set creates unnecessary database, network, and application work.

## Offset Pagination

```sql
SELECT *
FROM posts
ORDER BY created_at DESC
LIMIT 20 OFFSET 100000;
```

Large offsets can be expensive because the database may still locate and skip many earlier rows.

Concurrent inserts can also cause duplicates or omissions between pages.

## Cursor Pagination

Use the final ordered key from the previous page:

```sql
SELECT *
FROM posts
WHERE (created_at, post_id) < (:last_created_at, :last_post_id)
ORDER BY created_at DESC, post_id DESC
LIMIT 20;
```

The composite cursor provides a stable tie-breaker.

Suitable index:

```sql
CREATE INDEX idx_posts_cursor
ON posts(created_at DESC, post_id DESC);
```

## Benefits

- Predictable page cost
- Better behavior on large datasets
- More stable under concurrent inserts
- Natural fit for feeds and timelines

---

# 7. Vertical Scaling

Vertical scaling increases the capacity of one database machine.

Possible upgrades:

- More RAM
- More CPU cores
- Faster CPU
- Faster local or network storage
- Higher IOPS
- Greater network bandwidth

## Why RAM Helps

Databases keep frequently used pages in memory.

If the hot working set fits in the buffer pool:

```text
Memory read
instead of
Disk read
```

## Why SSDs Help

Indexes and table lookups often require random page access. SSDs provide much lower random-I/O latency than spinning disks.

## Advantages

- Operationally simple
- No application consistency changes
- Fast way to create headroom

## Limitations

- Hardware has an upper bound.
- Larger instances are expensive.
- A single machine remains one scaling unit.
- Maintenance and failure can still affect the entire workload.

Vertical scaling is not unsophisticated. It is often the lowest-risk step before distributing the system.

---

# 8. Query and Schema Optimization

Indexes cannot repair every inefficient query.

Review:

- Join order and join cardinality
- Sargability
- Unnecessary sorting
- Repeated functions on indexed columns
- Large object retrieval
- Data-type mismatches
- Excessive columns
- Query fan-out
- Missing limits
- Locking caused by long transactions

## Sargable Predicate

Prefer:

```sql
WHERE created_at >= '2026-01-01'
  AND created_at < '2027-01-01'
```

instead of:

```sql
WHERE EXTRACT(YEAR FROM created_at) = 2026
```

The direct range predicate can use an index more naturally.

## Avoid `SELECT *`

Fetch only the required columns:

```sql
SELECT user_id, display_name, avatar_url
FROM users
WHERE user_id = :user_id;
```

This reduces:

- Disk reads
- Network transfer
- Deserialization
- Cache size
- Memory usage

---

# 9. Normalization and Denormalization

## Normalization

Normalized schemas reduce duplication and update anomalies by separating entities.

Example:

```text
users
orders
order_items
products
```

An order summary may require joins across these tables.

Advantages:

- One authoritative copy of each mutable fact
- Simpler correctness rules
- Lower storage duplication
- Easier updates

## Denormalization

Denormalization stores data in the shape required by a common read.

Example order snapshot:

```text
order_id
customer_name_at_purchase
product_name_at_purchase
unit_price_at_purchase
quantity
order_total
```

Read:

```sql
SELECT customer_name_at_purchase,
       product_name_at_purchase,
       unit_price_at_purchase,
       quantity,
       order_total
FROM order_summary
WHERE order_id = :order_id;
```

## Snapshot Versus Duplicate Current State

Some duplicated values represent historical facts and should not change.

For example:

```text
product price at purchase time
shipping address used for the order
```

These are not merely cache copies. They are part of the order's history.

Other duplicated values represent the latest state and require propagation when the authoritative source changes.

## Trade-Off

```text
Faster and simpler reads
    ↕
More complex writes and synchronization
```

### Denormalization Invariant

> **For every duplicated field, define its authoritative owner, update mechanism, and tolerated staleness.**

---

# 10. Materialized Views and Precomputation

Expensive queries can be computed ahead of time.

## On-Demand Aggregate

```sql
SELECT product_id, AVG(rating)
FROM reviews
GROUP BY product_id;
```

Running this for every page request repeatedly scans or aggregates large data.

## Materialized Result

```text
product_id | average_rating | rating_count | refreshed_at
```

Reads become simple lookups.

## Refresh Strategies

### Synchronous Update

Update the aggregate in the same write path.

Advantages:

- Fresh result

Costs:

- More write latency
- More contention
- More complicated transactions

### Event-Driven Incremental Update

Publish review changes and update the aggregate asynchronously.

Advantages:

- Faster write path
- Scalable incremental computation

Costs:

- Eventual consistency
- Duplicate and ordering handling

### Periodic Batch Refresh

Recompute every few minutes or hours.

Advantages:

- Simple
- Good for analytics

Costs:

- Bounded staleness
- Expensive refresh jobs

## CQRS-Like Read Models

A system may keep a normalized write model and one or more denormalized read models optimized for APIs.

```text
Write model
    ↓ events or change stream
Read projections
    ├── profile view
    ├── feed view
    └── analytics view
```

This can scale reads well, but it requires replay, monitoring, and reconciliation of projections.

---

# 11. Read Replicas

Read replicas maintain copies of the primary database's data.

```text
Writes
  ↓
Primary or leader
  ↓ replication
Read replica 1
Read replica 2
Read replica 3
```

The application sends writes to the primary and eligible reads to replicas.

## Benefits

- Distributes read traffic
- Adds read capacity
- Isolates analytics or reporting workloads
- Provides additional copies for recovery
- May support geographic read placement

## Limitations

- Replication lag
- Replica connection and routing complexity
- More operational cost
- Failover complexity
- Long-running replica queries can affect replication
- Every replica still stores most or all of the data

Read replicas scale throughput better than they scale dataset size.

---

# 12. Synchronous and Asynchronous Replication

## Synchronous Replication

The primary waits for one or more replicas before considering a write committed.

```text
Primary receives write
    ↓
Replica confirms durable copy
    ↓
Primary acknowledges client
```

Advantages:

- Lower risk of losing acknowledged writes during failover
- Stronger freshness options

Costs:

- Higher write latency
- Availability may depend on replica reachability

## Asynchronous Replication

The primary acknowledges before replicas apply the change.

Advantages:

- Lower write latency
- Better write availability

Costs:

- Stale replica reads
- Potential loss of the newest acknowledged writes during failover, depending on setup

Replication choice is a consistency and availability decision, not only a performance decision.

---

# 13. Replication Lag

Replication lag is the delay between a write committing on the primary and becoming visible on a replica.

```text
12:00:00.000 write commits on primary
12:00:00.120 replica applies write
```

For 120 milliseconds, replica reads may show old data.

Lag can increase due to:

- Network delay
- Large write bursts
- Slow disks
- Long-running transactions
- Replica CPU saturation
- Schema changes
- Replication-worker limits

## User-Visible Failure

```text
User updates profile
    ↓
Application reads from replica
    ↓
Old profile appears
```

The user may think the write failed.

---

# 14. Read-After-Write Consistency

Several strategies can provide a user with a fresh view after their own write.

## Strategy 1: Read From Primary Temporarily

After a write, route that user's reads to the primary for a bounded period.

```text
write timestamp = now
if read occurs within freshness window:
    use primary
else:
    replica allowed
```

## Strategy 2: Session or Consistency Token

The write returns a replication position or logical version.

A replica can serve the read only after it has applied at least that position.

```text
minimum required version = 9183
```

## Strategy 3: Sticky Session

Pin related reads to a node that can satisfy the required freshness.

This is simple but can complicate load balancing and failover.

## Strategy 4: Read Critical Data From Primary

Use replicas for browse-heavy or stale-tolerant reads, while routing correctness-sensitive reads to the primary.

Examples:

```text
Product description → replica allowed
Final seat claim     → primary and transactional path
```

### Consistency Principle

> **Route each read according to its required freshness, not according to one global replica rule.**

---

# 15. Failover Is Not Automatic Correctness

A replica can improve availability, but promotion still requires:

- Detecting primary failure
- Choosing a new primary
- Preventing split brain
- Redirecting clients
- Verifying replication position
- Rebuilding failed replicas
- Handling in-flight writes

A replica used for reads is not automatically ready for safe promotion under every failure scenario.

Clients should also be prepared for:

- Connection resets
- Short write unavailability
- Transaction retries
- Stale DNS or topology information

---

# 16. Database Sharding

Sharding partitions data across independent database nodes.

```text
Shard 1: users A–F
Shard 2: users G–M
Shard 3: users N–Z
```

or:

```text
shard = hash(user_id) mod N
```

Each shard stores and serves only part of the dataset.

## Read Benefits

- Smaller indexes per shard
- Smaller working set per node
- Parallel read capacity
- Isolation between partitions
- Geographic locality when partitioned by region

## Costs

- Routing logic
- Rebalancing
- Cross-shard queries
- Distributed joins
- Global pagination
- Hot partitions
- Cross-shard transactions
- More difficult operations and backups

Sharding is often introduced for write throughput or dataset size. It is usually more complex than adding read replicas or caching.

---

# 17. Functional Sharding

Functional sharding separates data by domain:

```text
User database
Post database
Like database
Message database
```

Benefits:

- Independent scaling
- Clear ownership
- Smaller domain-specific datasets
- Fault isolation

Costs:

- Cross-service reads replace local joins.
- API composition can create network fan-out.
- Transactions across domains become difficult.
- Data ownership and duplication must be explicit.

Functional decomposition can improve scaling, but it should follow domain boundaries rather than arbitrary table splitting.

---

# 18. Geographic Partitioning

Data can be placed near the users or regulations associated with it.

```text
European users → European region
US users       → US region
Asian users    → Asian region
```

Benefits:

- Lower regional read latency
- Reduced cross-region traffic
- Data-residency support
- Regional failure isolation

Challenges:

- Users travel between regions.
- Global entities do not fit one region cleanly.
- Cross-region reads are slower.
- Replication and conflict rules become important.
- Data migrations are operationally difficult.

Do not confuse geographic replicas with geographic multi-writer systems. Multiple writable regions introduce conflict-resolution and consistency challenges beyond ordinary read scaling.

---

# 19. Application-Level Caching

A cache stores frequently requested data in a faster layer, typically memory.

```text
Client
    ↓
Application
    ↓
Cache lookup
    ├── hit  → return cached value
    └── miss → query database, populate cache, return value
```

This is commonly called **cache-aside** or **lazy loading**.

## Cache-Aside Pseudocode

```text
value = cache.get(key)

if value exists:
    return value

value = database.read(key)
cache.set(key, value, ttl)
return value
```

## Benefits

- Sub-millisecond or low-millisecond reads
- Reduced primary and replica load
- Natural concentration on hot data
- Independent cache scaling

## Costs

- Stale data
- Cache misses
- Invalidation complexity
- Additional failure mode
- Memory cost
- Serialization overhead
- Hot keys
- Stampedes

---

# 20. Cache Eligibility

Good cache candidates usually have:

- High read frequency
- Low or moderate update frequency
- Repeated access across users or requests
- Expensive underlying computation
- A clear staleness budget
- A bounded, serializable result

Examples:

- Public profiles
- Product details
- URL mappings
- Configuration
- Popular posts
- Search facets
- Recommendation summaries

Poor candidates include:

- One-time reads
- Large values with low reuse
- Security-sensitive data that is difficult to isolate
- Rapidly changing strongly consistent state
- Arbitrary per-user data with little cross-request reuse

Private data can still be cached in a correctly scoped application cache, but it is normally inappropriate for a shared public CDN.

---

# 21. Cache Hit Ratio

Cache effectiveness depends heavily on hit ratio.

```text
hit ratio = cache hits / total cache lookups
```

Suppose the application receives 100,000 reads per second.

At a 95 percent hit ratio:

```text
5,000 reads/second reach the database
```

At a 99 percent hit ratio:

```text
1,000 reads/second reach the database
```

A small percentage change can cause a large difference in backend load.

However, hit ratio alone is incomplete. Also measure:

- Byte hit ratio
- Miss penalty
- Hit latency
- Eviction rate
- Load latency
- Per-key distribution
- Backend load during cache failure

---

# 22. Cache-Aside Race Conditions

Consider this sequence:

```text
1. Reader misses cache.
2. Reader loads old value from database.
3. Writer updates database.
4. Writer invalidates cache.
5. Reader writes old value into cache.
```

The stale value has been resurrected after invalidation.

Possible mitigations include:

- Versioned cache keys
- Compare-and-set on cache population
- Short TTL safety net
- Write-through caching
- Delayed second invalidation in limited cases
- Reading and caching from a versioned snapshot
- Event-driven invalidation with monotonic versions

The exact solution depends on the consistency requirement and cache architecture.

---

# 23. TTL-Based Expiration

A TTL makes a cache entry expire automatically.

```text
cache.set(key, value, ttl = 300 seconds)
```

## Advantages

- Simple
- Bounds staleness in normal operation
- Cleans unused entries
- Recovers from missed invalidations eventually

## Disadvantages

- Data can remain stale until expiration.
- Simultaneous expirations can cause a stampede.
- A long TTL improves hit ratio but worsens freshness.
- A short TTL increases backend load.

## Choose TTL From Requirements

If the product says:

```text
Search results may be at most 30 seconds stale.
```

then the cache strategy should ensure that the effective maximum staleness is compatible with 30 seconds.

Do not choose TTL only by intuition. Tie it to:

- Freshness requirement
- Update frequency
- Miss cost
- Traffic level
- Invalidation reliability

---

# 24. Cache Write Strategies

## Cache Aside

Application reads cache, falls back to database, and populates cache.

Best for:

- General-purpose read caching
- Data not always requested

## Write Through

The write path updates the authoritative store and cache as part of the write flow.

Benefits:

- Cache is warm after writes
- Better immediate freshness

Risks:

- Dual-write failure handling
- Higher write latency
- Cache should not become an accidental authority

## Write Around

Writes go to the database and invalidate or bypass cache. Reads populate it later.

Benefits:

- Avoids filling cache with data that may never be read

Cost:

- First read after write misses

## Write Behind

Writes enter the cache and are persisted asynchronously.

Benefits:

- Low write latency
- Write batching

Risks:

- Data loss on cache failure
- Ordering and durability complexity
- Cache becomes part of the write authority

Write-behind is not merely an invalidation strategy. It changes durability semantics and must be designed carefully.

---

# 25. Active Invalidation

When authoritative data changes, the system can delete or update relevant cache entries.

```text
Write database
    ↓
Commit succeeds
    ↓
Publish invalidation event
    ↓
Caches remove or refresh affected keys
```

## Challenges

- Invalidation event may be lost.
- Cache deletion may fail.
- Dependencies may span multiple keys.
- CDN propagation takes time.
- Concurrent readers may repopulate stale values.

A TTL is often retained as a safety net even when active invalidation exists.

### Invalidation Invariant

> **A cache invalidation mechanism must be retryable, observable, and bounded by a fallback freshness policy.**

---

# 26. Versioned Cache Keys

Instead of overwriting one key, include a version in the key:

```text
product:123:v42
product:123:v43
```

When the entity changes, its authoritative version increments.

## Write Flow

```sql
BEGIN;

UPDATE products
SET name = :name,
    version = version + 1
WHERE product_id = :product_id
RETURNING version;

COMMIT;
```

The new result is written under:

```text
product:123:v43
```

Old versions become unreachable after readers learn the new version and later expire through TTL.

## Read Flow

```text
1. Obtain current entity version.
2. Construct versioned key.
3. Read cached representation.
4. On miss, read authoritative row.
5. Cache under that exact version.
```

## Benefits

- Late writers cannot overwrite the new version's key with old data.
- Old values do not need immediate deletion.
- Version can be included in CDN asset URLs.
- Concurrency boundaries are explicit.

## Costs and Nuances

- Readers still need a fresh-enough way to discover the current version.
- A stale version pointer can still route a reader to old data.
- Extra lookup may be required.
- Old keys consume memory until expiry.
- Computed queries and feeds do not map cleanly to one entity version.

Versioned keys simplify entity caching but do not eliminate all invalidation problems.

---

# 27. Dependency and Tag Invalidation

One source entity may appear in many cached results.

Example:

```text
post:42
user:7:feed
topic:databases:feed
home:trending
```

If post 42 is deleted, several cache entries may need invalidation.

## Tags

Cache entries can be associated with tags:

```text
post:42
user:7
feed:home
```

An invalidation process removes entries associated with a changed tag.

## Trade-Offs

- Dependency metadata consumes memory.
- Tag sets can become very large.
- Invalidation fan-out can be expensive.
- Failed partial invalidation needs repair.

For complex derived data, it may be better to asynchronously rebuild read models rather than maintain an exact dependency graph.

---

# 28. Deleted-Item or Suppression Cache

Some changes must become visible quickly even when large derived caches are stale.

Examples:

- Deleted post
- Hidden content
- Privacy removal
- Moderation block
- Revoked product

Instead of immediately rebuilding every feed containing the item, maintain a small fast set:

```text
suppressed_content_ids = {42, 91, 108}
```

When serving cached feeds:

```text
1. Load cached feed IDs.
2. Check suppression set.
3. Remove suppressed IDs.
4. Return remaining items.
5. Rebuild full feed asynchronously.
```

This pattern prioritizes fast removal while allowing expensive derived caches to converge later.

The suppression cache itself must be highly available and scoped to the required consistency guarantee.

---

# 29. Negative Caching

Repeated requests for missing data can overload the database.

Example:

```text
GET /products/nonexistent-id
```

Cache a bounded negative result:

```text
product:missing-id → NOT_FOUND, TTL 30 seconds
```

Benefits:

- Protects against repeated misses
- Helps with bot traffic or malformed IDs
- Reduces lookup amplification

Risks:

- Newly created data may remain hidden until negative TTL expires.
- Attackers can fill cache with random absent keys.

Use short TTLs and admission policies.

---

# 30. Cache Stampede

A cache stampede occurs when many requests simultaneously discover an absent or expired hot key and all rebuild it.

```text
Hot key expires
    ↓
100,000 requests miss
    ↓
100,000 database queries
    ↓
Database overload
```

The cache normally protects the database, but synchronized expiration removes that protection at the worst moment.

---

# 31. Request Coalescing

Request coalescing, also called single-flight, allows one in-flight load per key within a process or coordination scope.

```text
Request A misses and starts backend fetch.
Requests B, C, D see the same in-flight future.
All await A's result.
```

Pseudocode:

```python
class SingleFlight:
    def __init__(self):
        self.inflight = {}

    async def get(self, key):
        if key in self.inflight:
            return await self.inflight[key]

        future = create_future()
        self.inflight[key] = future

        try:
            value = await load_from_backend(key)
            future.set_result(value)
            return value
        except Exception as error:
            future.set_exception(error)
            raise
        finally:
            self.inflight.pop(key, None)
```

## Scope

If coalescing occurs only inside each application server and there are 100 servers, the backend may still receive up to approximately 100 rebuild requests.

Distributed coalescing can reduce this further but introduces coordination complexity.

## Failure Handling

- Bound waiting time.
- Remove failed futures.
- Do not let one hung rebuild block forever.
- Consider serving stale data during rebuild.

---

# 32. Stale-While-Revalidate

Keep two conceptual deadlines:

```text
fresh_until
serve_stale_until
```

Behavior:

```text
Before fresh_until:
    Return cached value.

After fresh_until but before serve_stale_until:
    Return stale value immediately.
    Trigger one background refresh.

After serve_stale_until:
    Require fresh load or use an explicit fallback.
```

Benefits:

- Low user latency
- Reduced stampede risk
- Availability during temporary backend failure

Trade-Off:

- Explicitly serves stale data

This is appropriate only when the staleness window is acceptable for that data.

---

# 33. Early Refresh and TTL Jitter

## Probabilistic Early Refresh

As an entry approaches expiry, a small subset of requests refreshes it early.

```text
Far from expiry  → tiny refresh probability
Near expiry      → larger refresh probability
```

This distributes refresh work over time rather than creating one cliff.

## TTL Jitter

If many keys are populated together with the same TTL, they may expire together.

Instead of exactly 600 seconds:

```text
TTL = 600 seconds ± random jitter
```

This spreads misses over time.

## Proactive Refresh

A background worker refreshes known hot entries before expiry.

Best for:

- Homepage content
- Global configuration
- Extremely hot product or event pages
- Expensive values with predictable demand

Costs:

- Refreshes may occur even when nobody reads the data.
- The refresher becomes another system to operate.

---

# 34. Cache Penetration, Breakdown, and Avalanche

These terms describe related failure modes.

## Cache Penetration

Requests repeatedly target values that do not exist, causing misses to reach the database.

Mitigations:

- Negative caching
- Input validation
- Bloom filters in suitable architectures
- Rate limiting

## Cache Breakdown

One very hot key expires and many requests rebuild it.

Mitigations:

- Single-flight
- Stale-while-revalidate
- Proactive refresh

## Cache Avalanche

Many keys expire or the cache fleet fails at the same time, causing broad database overload.

Mitigations:

- TTL jitter
- Multi-level caching
- Graceful degradation
- Rate limiting
- Load shedding
- Backend capacity reservation
- Controlled cache warm-up

---

# 35. Hot Keys

A hot key receives far more traffic than normal keys.

```text
Average key: 100 reads/second
Celebrity post: 500,000 reads/second
```

Sharding by key may place the entire popular key on one cache node, overwhelming its CPU or network.

## Mitigation 1: Local In-Process Cache

Store the hottest immutable or bounded-staleness values inside each application process.

```text
Client → app-local cache
              ↓ miss
         distributed cache
              ↓ miss
            database
```

This removes repeated network calls to the shared cache.

## Mitigation 2: Replicate the Hot Key

Store multiple copies:

```text
post:42:replica:0
post:42:replica:1
...
post:42:replica:9
```

Readers choose a replica.

Trade-offs:

- More memory
- More complicated updates and invalidation
- Inconsistent copies during propagation

## Mitigation 3: CDN

For public shared content, push delivery to many edge nodes.

## Mitigation 4: Pre-Serialized Payload

Cache the final response bytes to reduce repeated object construction and serialization.

### Hot-Key Principle

> **A distributed cache still has finite per-node CPU and network capacity; one popular key may require replication closer to readers.**

---

# 36. Multi-Level Caching

A read path may contain several cache layers:

```text
Browser cache
    ↓ miss
CDN edge
    ↓ miss
Reverse proxy
    ↓ miss
Application local cache
    ↓ miss
Distributed cache
    ↓ miss
Read replica or primary database
```

Each layer has different:

- Scope
- Latency
- Capacity
- Invalidation behavior
- Privacy boundary
- Staleness tolerance

## Example Policy

```text
Static thumbnail:
Browser 1 day, CDN 7 days, versioned URL

Public profile:
CDN 30 seconds, Redis 5 minutes

Private account settings:
No public CDN, scoped application cache 30 seconds

Seat purchase decision:
No stale cache in authoritative transaction
```

The complexity of invalidation grows with the number of layers.

---

# 37. CDN and Edge Caching

A content delivery network serves cacheable content from locations near users.

```text
User in Tokyo
    ↓
Tokyo edge cache
    ↓ miss only
Origin region
```

Benefits:

- Lower global latency
- Reduced origin bandwidth
- Reduced application and database traffic
- High fan-out capacity
- Protection from traffic bursts

Good candidates:

- Images
- Video segments
- JavaScript and CSS
- Public product pages
- Public profiles
- Public API responses
- Search pages with shared query patterns

Poor candidates:

- Private messages
- User-specific account data
- Responses varying on sensitive authorization state
- Immediate-consistency decisions

---

# 38. HTTP Cache Semantics

Important response headers include:

```http
Cache-Control: public, max-age=60, s-maxage=300
ETag: "product-123-v42"
Vary: Accept-Encoding
```

## `max-age`

Controls freshness in private caches such as a browser.

## `s-maxage`

Can control freshness in shared caches such as a CDN.

## `public` and `private`

Indicate whether shared caches may store the response.

## `no-store`

Instructs caches not to store the response.

## ETag and Conditional Requests

Client sends:

```http
If-None-Match: "product-123-v42"
```

If unchanged, server can return:

```http
304 Not Modified
```

This saves response bytes but may still require an origin validation unless the edge handles it.

## `Vary`

Defines which request headers change the cached representation.

Using high-cardinality headers in `Vary`, such as user-specific authorization, can destroy hit ratio or create privacy risk.

---

# 39. CDN Invalidation and Versioned URLs

CDN purge APIs remove cached content, but purge propagation is not instantaneous or infallible.

For immutable assets, prefer versioned URLs:

```text
/images/logo.v42.png
/video/segment/hash-abc123.ts
```

When content changes, produce a new URL.

Old content can remain cached safely because new references point to the new asset.

For dynamic data:

- Use short edge TTLs.
- Use surrogate keys or tags where supported.
- Purge critical content.
- Keep an origin-level correctness check for sensitive operations.

---

# 40. Cache Security and Privacy

Caching can leak data if keys or HTTP policies omit authorization context.

Dangerous example:

```text
cache key = /api/account
```

If the response differs by user, one user's response could be returned to another.

Safer approaches:

- Do not use shared cache for private data.
- Include tenant or user identity in scoped cache keys.
- Include authorization-relevant version or role where needed.
- Use `Cache-Control: private` or `no-store` appropriately.
- Encrypt sensitive values in shared infrastructure.
- Apply deletion and retention policies to caches.

### Privacy Invariant

> **Two requests may share a cached response only when their authorization and representation semantics are equivalent.**

---

# 41. Cache Failure and Graceful Degradation

A cache should reduce database load, but the system must survive cache impairment.

A complete cache outage can cause:

```text
100 percent cache miss rate
    ↓
Database traffic increases dramatically
    ↓
Database overload
```

This is sometimes called a cache-collapse or fail-open problem.

## Protection Mechanisms

- Rate limiting
- Circuit breakers
- Request coalescing
- Stale serving
- Load shedding
- Database concurrency limits
- Priority queues
- Cache replica or cluster failover
- Controlled warm-up
- Partial feature degradation

## Graceful Degradation Example

During cache failure:

```text
Return core product details
Skip recommendations
Reduce comment preview count
Serve older aggregate counts
```

The system should protect authoritative dependencies before preserving every optional feature.

---

# 42. Cache Warm-Up

A cold cache can overload the database after:

- Deployment
- Cache restart
- Regional failover
- Data migration
- Full invalidation

Warm-up strategies:

- Preload known hot keys.
- Replay recent access logs.
- Gradually shift traffic.
- Limit concurrent cache fills.
- Serve stale snapshots during recovery.
- Warm only the hot working set, not the entire database.

A distributed cache may contain far less data than the database. Its value comes from keeping the most useful working set, not duplicating everything.

---

# 43. Precomputed Feeds and Read Models

Feed generation can be performed at read time or write time.

## Fan-Out on Read

At read time:

```text
Fetch followed users
Fetch recent posts from each
Merge and rank
Return page
```

Benefits:

- Simple writes
- Fresh relationship data

Costs:

- Expensive reads
- High fan-out
- High latency for users following many accounts

## Fan-Out on Write

When a user posts:

```text
Insert post ID into followers' feed inboxes
```

Read becomes:

```text
Fetch precomputed feed IDs
Hydrate recent items
```

Benefits:

- Fast reads

Costs:

- Expensive writes
- Storage amplification
- Celebrity fan-out problem
- Complex delete and edit propagation

## Hybrid Strategy

- Fan out ordinary users' posts on write.
- Merge celebrity posts at read time.
- Cache only recent feed pages.

This balances write amplification and read latency.

---

# 44. Strongly Consistent Data

Some data cannot safely tolerate stale cached values.

Examples:

- Seat ownership
- Account balance used for authorization
- Inventory claim
- Permission revocation
- Security policy
- Payment state before settlement

A cache can still help with display or browsing, but the final decision must use the authoritative consistency path.

Example:

```text
Event page seat map:
May use a short-lived cache for approximate browsing.

Purchase transaction:
Must atomically verify current seat state in authoritative storage.
```

### Authority Principle

> **Caches may inform users, but correctness-sensitive state transitions must be validated by the authoritative system.**

---

# 45. Common Product Scenarios

## URL Shortener

Read pattern:

```text
One mapping creation
Potentially millions of redirects
```

Design:

- Unique index on short code
- Redis or in-process cache for mappings
- CDN or edge redirects when safe
- Negative caching for invalid codes
- Long TTL for immutable mappings

If links can be edited or disabled, define invalidation and abuse-control behavior.

## Ticketing

Cache:

- Event description
- Venue metadata
- Static seating map
- Public images

Do not trust stale cache for:

- Final seat availability
- Reservation ownership
- Purchase decision

Use the authoritative transactional path for claims.

## News Feed

Design:

- Cursor pagination
- Precomputed feed IDs
- Cache recent pages
- Batch hydration
- Hybrid fan-out strategy
- Suppression cache for deletes or moderation

## Video Platform

Cache:

- Metadata
- Channel summary
- Thumbnails
- Recommendation responses

Use CDN for:

- Video segments
- Thumbnails
- Static assets

Aggregate view counts asynchronously where exact immediate counts are unnecessary.

## Product Catalog

Use:

- Search index for product discovery
- CDN for public product pages and images
- Application cache for product details
- Materialized aggregates for rating summaries
- Authoritative inventory path for checkout

## Metrics Dashboard

Use:

- Pre-aggregated time buckets
- Downsampling
- Result caching
- Query limits
- Separate analytical store

Do not send every dashboard refresh to the raw event store.

---

# 46. When Not to Add Read-Scaling Infrastructure

## Small Systems

A well-indexed database may handle the required workload comfortably.

Do not add Redis, replicas, and CDN purging without a demonstrated need.

## Write-Dominated Systems

If writes are the bottleneck, read replicas or caches may not address the main problem.

## Real-Time Collaborative State

Aggressive caching can conflict with immediate propagation and conflict-resolution requirements.

## Strong Freshness Requirements

Caching may still be used selectively, but stale reads must not control authoritative decisions.

## Low-Reuse Private Data

A shared cache may offer little hit-rate benefit and can create privacy risk.

Good system design solves the stated workload, not an imagined future one.

---

# 47. Interview Design Process

## Step 1: Estimate Read Load

```text
Daily active users
× reads per user per day
÷ seconds per day
× peak factor
```

Also estimate internal reads per API request.

## Step 2: Identify Hot Endpoints

Examples:

- `GET /users/{id}`
- `GET /feed`
- `GET /products/{id}`
- `GET /short/{code}`

## Step 3: State Freshness Requirements

For each endpoint:

```text
Must be current
Read-your-writes
Up to 5 seconds stale
Up to 5 minutes stale
Immutable
```

## Step 4: Optimize the Database

- Add query-driven indexes.
- Fix N+1 queries.
- Add pagination.
- Select only required columns.
- Inspect plans.
- Consider vertical scaling.

## Step 5: Precompute Expensive Reads

- Denormalized read table
- Materialized view
- Search index
- Feed inbox
- Aggregate table

## Step 6: Add Replicas When Appropriate

Explain:

- Which reads may use replicas
- How lag is monitored
- How read-after-write is provided
- How failover works

## Step 7: Add Cache

Define:

- Key schema
- Value shape
- TTL
- Invalidation
- Miss behavior
- Stampede prevention
- Failure behavior
- Privacy boundary

## Step 8: Add CDN for Shared Global Data

Define:

- Cache headers
- Versioned assets
- Purge needs
- Origin fallback
- Vary dimensions

## Step 9: Discuss Operations

- Hit ratio
- Replica lag
- Hot keys
- Cache memory and eviction
- Database saturation
- Warm-up
- Load shedding

---

# 48. Interview Answer Template

> I would first quantify the read-to-write ratio, peak QPS, internal query fan-out, and freshness requirements. Before adding distributed infrastructure, I would inspect query plans, add composite indexes that match the dominant filters and ordering, eliminate N+1 queries, use cursor pagination, and fetch only required columns. For expensive repeated joins or aggregates, I would create a denormalized read model or materialized view with an explicit update and staleness policy. If the primary still reaches read capacity, I would route stale-tolerant reads to asynchronous replicas while preserving read-after-write through primary routing or consistency tokens. For high-reuse data, I would add cache-aside caching with TTL as a safety net, active or version-based invalidation, request coalescing, stale-while-revalidate, and TTL jitter. Public shared content would move to a CDN with correct HTTP cache headers and versioned URLs. Correctness-sensitive decisions such as inventory claims would always revalidate against the authoritative transactional store.

---

# 49. Common Mistakes

## Mistake 1: Add Cache Before Fixing Queries

A cache can hide inefficient queries until a miss storm exposes them.

## Mistake 2: Ignore Internal Read Amplification

One API call may generate dozens of queries or service calls.

## Mistake 3: Send Every Read to a Replica

Some reads require the primary or a minimum replication position.

## Mistake 4: Treat Cache as Authoritative Accidentally

Final correctness decisions must use the system of record unless the cache is deliberately designed as a durable authority.

## Mistake 5: Use One TTL for All Data

Freshness requirements differ by entity and endpoint.

## Mistake 6: Ignore Stampedes

A 99 percent cache hit ratio is irrelevant if one synchronized expiration overloads the database.

## Mistake 7: Ignore Hot Keys

Even an in-memory cache node has finite CPU and bandwidth.

## Mistake 8: Cache Private Responses Publicly

Incorrect cache keys or CDN headers can expose user data.

## Mistake 9: Assume Versioned Keys Give Instant Freshness

Readers still need an up-to-date version pointer or another mechanism to discover the new version.

## Mistake 10: Shard Too Early

Sharding adds routing, rebalancing, and cross-shard complexity. Replicas and caches often solve read load first.

## Mistake 11: Cache Unbounded Query Combinations

Search and filter caches can have enormous key cardinality and low hit ratios.

## Mistake 12: Ignore Cache Failure

The database must be protected when the cache is cold or unavailable.

---

# 50. Operational Metrics

## Database

- Read QPS
- Query latency percentiles
- CPU utilization
- Disk IOPS and throughput
- Buffer hit ratio
- Active and waiting connections
- Slow-query count
- Rows scanned per result
- Temporary disk usage

## Replication

- Apply lag
- Byte lag
- Replay position
- Replica CPU
- Replica query latency
- Failover readiness

## Cache

- Request rate
- Hit ratio
- Byte hit ratio
- Miss latency
- Eviction rate
- Memory fragmentation
- Hot-key distribution
- Connection count
- Timeout rate
- Rebuild rate
- Stale-serving rate

## CDN

- Edge hit ratio
- Origin fetch rate
- Origin egress
- Regional latency
- Purge latency
- Error rate

## Application

- Queries per request
- Dependency calls per request
- Response size
- Coalesced waiter count
- Rate-limited requests
- Degraded responses

---

# 51. Important Invariants

## Freshness Invariant

> **Every read path has a defined maximum staleness or consistency level.**

## Authority Invariant

> **Correctness-sensitive state transitions are validated by the authoritative store.**

## Cache-Key Invariant

> **A cache key contains every dimension that changes the representation or authorization scope.**

## Invalidation Invariant

> **Missed invalidation eventually heals through TTL, versioning, rebuild, or reconciliation.**

## Replica Invariant

> **A replica serves a read only when its freshness is sufficient for that request.**

## Stampede Invariant

> **One missing hot key cannot create unbounded concurrent backend work.**

## Privacy Invariant

> **Shared caches never mix responses that differ by user or authorization context.**

## Degradation Invariant

> **Cache or replica failure cannot create unbounded load on the primary database.**

## Read-Model Invariant

> **Every denormalized or materialized view has an authoritative source and a repair mechanism.**

---

# 52. Quick Comparison

## Indexes

```text
Best for:
Selective database queries

Benefit:
Fewer pages scanned

Cost:
Storage and write maintenance
```

## Denormalized Read Model

```text
Best for:
Repeated joins or API-specific projections

Benefit:
Simple fast reads

Cost:
Write complexity and consistency lag
```

## Read Replica

```text
Best for:
Fresh-enough reads that exceed primary capacity

Benefit:
Horizontal read throughput

Cost:
Replication lag and failover complexity
```

## Sharding

```text
Best for:
Datasets or throughput beyond one database node

Benefit:
Smaller per-node data and parallel capacity

Cost:
Routing and cross-shard complexity
```

## Application Cache

```text
Best for:
Frequently repeated expensive reads

Benefit:
Low latency and reduced database load

Cost:
Staleness, invalidation, and failure handling
```

## CDN

```text
Best for:
Public shared content with global readers

Benefit:
Edge latency and origin offload

Cost:
Distributed invalidation and cache-policy complexity
```

## Materialized View

```text
Best for:
Expensive stable aggregates

Benefit:
Precomputed query result

Cost:
Refresh and staleness management
```

---

# 53. Thirty-Second Revision

- **Read scaling:** serve more read traffic without overloading authoritative storage.
- **First step:** measure endpoint QPS, queries per request, plans, and freshness needs.
- **Indexes:** reduce scanned pages for selective filters, joins, ranges, and ordering.
- **Cursor pagination:** avoids increasingly expensive large offsets.
- **Vertical scaling:** more RAM, CPU, IOPS, and network can provide simple headroom.
- **Denormalization:** trade write complexity and storage for simpler reads.
- **Materialized view:** precompute expensive joins or aggregates.
- **Read replica:** distribute eligible reads, but account for replication lag.
- **Read-after-write:** temporarily use primary or require a minimum replication position.
- **Sharding:** split the dataset when one node cannot hold or serve it efficiently.
- **Cache aside:** read cache, fall back to database, then populate cache.
- **TTL:** bounds ordinary staleness and heals missed invalidation eventually.
- **Versioned key:** separates old and new entity representations, but readers need a fresh version pointer.
- **Stampede:** many requests rebuild one expired value simultaneously.
- **Single-flight:** one in-flight load per key per coordination scope.
- **Stale-while-revalidate:** serve stale data while one refresh runs.
- **TTL jitter:** prevents many keys from expiring together.
- **Hot key:** one cache key overwhelms a node; use local caching, replication, or CDN.
- **Negative caching:** temporarily cache not-found results.
- **CDN:** cache shared public content near users.
- **Suppression cache:** quickly filter deleted or hidden items from stale derived caches.
- **Correctness rule:** never trust stale cache for final inventory, payment, or ownership decisions.
- **Best progression:** optimize, precompute, replicate, cache, move shared content to the edge.

## Final Mental Model

```text
Read request
    ↓
Can we reduce or batch the read?
    ↓
Does an index and bounded query solve it?
    ↓
Can we precompute the result?
    ↓
Is replica staleness acceptable?
    ↓
Will repeated requests benefit from a cache?
    ↓
Is the content public and globally shared?
    └── Use CDN or edge cache

At every layer ask:
    What is the authority?
    How stale may this be?
    How is it invalidated?
    What happens on a miss?
    What happens if the layer fails?
    Can one hot key overload it?
```
