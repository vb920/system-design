# Numbers to Know 2026: Quick Revision Notes

> These are approximate planning ranges from the supplied article, not hard limits. Actual capacity depends on workload, schema, hardware, durability, replication, and configuration.

## Core Thesis

**Modern hardware handles far more than older rules of thumb suggest, so calculate capacity before adding distributed-system complexity.**

## Key Insights

- **Measure before scaling**
- **Vertical scaling remains powerful**
- **Storage is rarely the first constraint**
- **CPU and throughput often fail before memory**
- **Replication for availability is not sharding**
- **Indexed database lookups are already fast**
- **Queues need delivery or decoupling justification**
- **Use order-of-magnitude estimates, not fake precision**

## Modern Server Capacity

### General-Purpose Server

- **CPU:** 8–128+ vCPUs
- **Memory:** 64–512 GiB common
- **High-memory instances:** Multiple TiB
- **Specialized maximum cited:** Up to 24 TiB RAM
- **Network:** 25 Gbps common
- **High-performance network:** 50–100+ Gbps

### Local Storage

- **Local SSD cited:** Around 60 TB
- **Local HDD cited:** Around 336 TB
- **Object storage:** Effectively elastic at petabyte scale

### Network Latency

- **Same availability zone:** Sub-1 ms
- **Cross-AZ, same region:** Roughly 1–2 ms
- **Cross-region:** Roughly 50–150 ms

### Main Lesson

**A single modern machine may replace a prematurely distributed design.**

---

## Caching

### Useful Planning Numbers

- **Latency:** Around 1 ms or lower
- **Throughput:** 100k–200k+ simple operations/second per instance
- **Memory:** Hundreds of GiB to around 1 TiB per large cache node
- **Same-AZ writes:** Often below 1 ms
- **Cross-AZ writes:** Roughly 1–2 ms in optimized systems

### Scale Triggers

- Dataset approaching node memory limit
- Memory usage above **80%**
- Sustained throughput above **100k ops/s**
- Latency consistently above target
- Hit rate below **80%**
- Eviction churn or cache thrashing
- Network bandwidth near instance limit

### Interview Implications

- Entire working set may fit in memory
- Selective caching may be unnecessary
- Memory may not be the first bottleneck
- Throughput and network often limit earlier
- Cache only when database or computation is measurably slow

### Avoid

- Adding Redis for every indexed lookup
- Complex partial-cache logic without need
- Ignoring invalidation and failure behavior
- Assuming cache nodes are limited to tens of GiB

---

## Databases

### Useful Planning Numbers

- **Storage:** Around 64 TiB for many managed engines
- **Aurora storage cited:** Up to 256 TiB
- **Cached reads:** Roughly 1–5 ms
- **Disk reads:** Roughly 5–30 ms
- **Write commit latency:** Roughly 5–15 ms
- **Read throughput:** Up to around 50k TPS for suitable workloads
- **Write throughput:** Around 10k–20k TPS for simple, well-tuned workloads
- **Connections:** Roughly 5k–20k, database and instance dependent

### Consider Scaling When

- Dataset approaches **50+ TiB**
- Sustained writes exceed roughly **10k TPS**
- Read latency misses requirements after optimization
- Backup or recovery windows become impractical
- Cross-region distribution is required
- Primary CPU, I/O, or connections remain saturated
- One machine no longer meets availability or operational needs

### Scale in This Order

1. Fix slow queries
2. Add correct indexes
3. Use connection pooling
4. Increase instance size
5. Add caching for expensive reads
6. Add read replicas
7. Partition or archive old data
8. Shard only after proving necessity

### Important Distinction

- **Replication:** Availability and read scaling
- **Sharding:** Storage and write scaling

A primary with replicas is still logically one unsharded dataset.

### Interview Implications

- A few terabytes do not automatically require sharding
- Millions of users may fit on one well-tuned database
- Operational limits may matter before raw storage limits
- Indexed point lookups may already meet latency targets

---

## Application Servers

### Useful Planning Numbers

- **Concurrent connections:** 100k+ in optimized event-driven servers
- **CPU:** 8–64 cores common
- **Memory:** 64–512 GiB common
- **High-memory instances:** Up to multiple TiB
- **Network:** 25 Gbps common; 50–100+ Gbps high end
- **Container startup:** Roughly 30–60 seconds

### Scale Triggers

- CPU consistently above **70–80%**
- Memory consistently above **70–80%**
- Response latency exceeds SLA
- Network approaches instance capacity
- Connection count approaches tested limit
- Queue depth grows continuously

### Main Bottleneck

**CPU often becomes limiting before memory.**

### Interview Implications

- Use local memory for hot configuration or reference data
- Stateless design still simplifies scaling
- Do not ignore substantial per-server capacity
- Autoscaling is useful, but startup delay matters
- Load testing matters more than generic connection limits

---

## Message Queues

### Useful Planning Numbers

- **Throughput:** Up to around 1 million messages/second per broker in optimized cases
- **Latency:** Roughly 1–5 ms within a region
- **Message size:** Approximately 1 KB–10 MB practical range
- **Storage:** Up to tens of TB per broker
- **Retention:** Weeks to months, capacity dependent

### Scale Triggers

- Throughput approaches tested broker limit
- Consumer lag grows continuously
- Disk usage approaches retention capacity
- Partition count becomes operationally excessive
- Cross-region replication is required
- Tail latency violates processing SLA

### Use a Queue For

- Producer-consumer decoupling
- Guaranteed or durable delivery
- Retryable background processing
- Traffic-spike buffering
- Event sourcing
- Fan-out to several consumers
- Downstream failure isolation

### Do Not Add a Queue Merely Because

- Writes reach 5k per second
- The architecture looks more scalable
- Database writes are assumed slow
- The operation can remain synchronous and simple

### Important Correction

**Low queue latency does not automatically make a durable queue ideal inside every synchronous request path.**

Consider:

- Broker acknowledgement mode
- Replication durability
- Consumer processing time
- End-to-end tail latency
- Failure semantics
- Whether the request needs the final result

---

## Capacity-Planning Formulas

### Storage

```text
Total storage = records × bytes per record × retention factor
```

Add allowance for:

- Indexes
- Replicas
- Metadata
- WAL or logs
- Temporary files
- Growth
- Backups

### Average Requests Per Second

```text
Average RPS = daily requests / 86,400
```

### Peak Requests Per Second

```text
Peak RPS = average RPS × peak factor
```

Typical interview assumption:

```text
Peak factor ≈ 3–10×
```

State the assumption explicitly.

### Network Bandwidth

```text
Bandwidth = requests/second × bytes/request
```

### Cache Memory

```text
Cache memory = hot objects × average object size × overhead factor
```

### Database Write Load

```text
Writes/second = active users × writes/user/second
```

### Queue Retention Storage

```text
Storage = messages/second × bytes/message × retention seconds × replication factor
```

---

## Worked Examples

### Yelp-Style Business Data

```text
10 million businesses × 1 KB = 10 GB
```

Allow 10× for reviews and overhead:

```text
≈ 100 GB
```

Conclusion:

- Easily fits in one modern database
- No immediate sharding requirement
- Indexing and query design matter more

### Leaderboard Cache

```text
100k competitions
× 100k users
× 40 bytes per entry
≈ 400 GB
```

Conclusion:

- Fits on a large cache node by raw size
- Check data-structure overhead and replication
- Throughput or hot keys may limit before memory
- Sharding is not automatically required

### Write-Heavy Service

```text
5k simple writes/second
```

Conclusion:

- May fit on tuned PostgreSQL
- First evaluate transaction complexity and indexes
- Do not add a queue solely for this rate

Potential real bottlenecks:

- Multi-table transactions
- Excessive secondary indexes
- Cascading updates
- Lock contention
- Synchronous replication
- Heavy competing reads

---

## Common Interview Mistakes

### **Premature Sharding**

- Dataset only hundreds of GB
- Write load within one primary’s capacity
- No calculation performed
- Complexity introduced too early

Better response:

> The estimated dataset is about 100 GB, so I’ll begin with one relational primary plus replicas and revisit sharding after observing sustained capacity pressure.

### **Outdated Memory Assumptions**

- Assuming Redis is limited to 32–64 GB
- Designing many shards for a few hundred GB
- Ignoring high-memory instances

Better response:

- Calculate working-set size
- Include object overhead
- Include replica memory
- Check throughput and network

### **Overestimating Indexed Read Latency**

- Assuming every database read takes tens or hundreds of milliseconds
- Adding cache before measuring
- Ignoring buffer pool and indexes

Reality:

- Cached indexed lookup: often low single-digit milliseconds
- Disk-based indexed lookup: often a few to tens of milliseconds
- Expensive joins or scans remain slow

### **Adding a Queue for Moderate Writes**

- 5k writes/second treated as extreme
- No delivery or decoupling requirement
- Extra operational complexity

Use a queue when:

- Delivery must survive downstream failure
- Spikes exceed database capacity
- Processing is asynchronous
- Several consumers need events
- Event sourcing is required

### **Treating Published Numbers as Guarantees**

- Ignoring payload size
- Ignoring durability settings
- Ignoring indexes and transactions
- Ignoring p99 latency
- Ignoring workload skew

Better response:

> I’ll use this as an order-of-magnitude starting point and validate it with a benchmark matching our schema and workload.

### **Confusing HA With Sharding**

- “One database” interpreted as one physical server
- Replication omitted
- Sharding proposed for availability alone

Correct model:

```text
Primary + replicas = high availability
Multiple data partitions = sharding
```

---

## Practical Scaling Triggers

### Cache

- Memory above 80%
- Sustained 100k+ ops/s
- Hit rate below target
- Eviction churn
- Hot keys
- Network saturation

### Database

- Sustained write saturation
- CPU or I/O above safe threshold
- Slow backups or recovery
- Dataset approaching storage boundary
- Geographic distribution requirement
- Connection or lock contention

### Application Server

- CPU above 70–80%
- Memory above 70–80%
- Latency above SLA
- Growing queue depth
- Network or connection pressure

### Message Queue

- Growing consumer lag
- Broker throughput near benchmarked limit
- Disk or retention pressure
- Excessive partitions
- Replication bottleneck

---

## Cost Reasoning

### Interview Expectation

- Compare broad architecture costs
- Avoid exact cloud pricing tables
- Prefer order-of-magnitude reasoning
- Identify obviously wasteful designs

### Discuss

- Number of machines
- Storage class
- Memory versus disk
- Cross-region transfer
- Operational complexity
- Engineering effort
- Managed versus self-hosted services

### Avoid

- Memorizing current provider prices
- Pretending rough estimates are exact
- Ignoring engineering and operational cost

### TCO

**Total Cost of Ownership includes:**

- Infrastructure
- Operations
- On-call burden
- Engineering maintenance
- Reliability work
- Migration complexity

---

## Interview Capacity Framework

### 1. Estimate Demand

- Daily active users
- Requests per user
- Read/write ratio
- Peak multiplier
- Payload size
- Retention period

### 2. Estimate Resource Needs

- Storage
- Read QPS
- Write QPS
- Bandwidth
- Active connections
- Cache working set

### 3. Compare With Single-Node Capacity

- Does it fit in storage?
- Does throughput fit?
- Does latency meet SLA?
- Are backups manageable?
- Is geographic distribution needed?

### 4. Apply Simple Optimizations

- Correct indexes
- Connection pooling
- Query optimization
- Batching
- Caching expensive results
- Read replicas
- Larger instance

### 5. Scale Horizontally Only When Justified

- Sharding
- Cache clustering
- More application servers
- More queue brokers
- Regional deployment

### 6. State Uncertainty

> These are order-of-magnitude estimates. I would validate them through representative load tests before committing to a topology.

---

## Concise Cheat Sheet

### Caching

- **Latency:** ~1 ms
- **Throughput:** 100k–200k+ ops/s
- **Memory:** Hundreds of GiB to ~1 TiB
- **Scale on:** Memory, throughput, churn, hot keys

### Databases

- **Cached read:** ~1–5 ms
- **Disk read:** ~5–30 ms
- **Writes:** ~10k–20k TPS for simple tuned workloads
- **Storage:** Tens of TiB; Aurora cited up to 256 TiB
- **Scale on:** Write saturation, operations, geography, recovery

### Application Servers

- **Connections:** 100k+ when optimized
- **CPU:** 8–64 cores common
- **Memory:** 64–512 GiB common
- **Network:** 25–100+ Gbps
- **Scale on:** CPU, latency, memory, network, connections

### Message Queues

- **Throughput:** Up to ~1M messages/s per broker in optimized cases
- **Latency:** ~1–5 ms
- **Storage:** Tens of TB
- **Scale on:** Throughput, lag, disk, partitions, replication

---

## Important Terms

- **Capacity Planning:** Estimating resources needed for expected demand
- **Order of Magnitude:** Approximate scale rather than exact value
- **Vertical Scaling:** Increasing resources on one machine
- **Horizontal Scaling:** Adding more machines
- **Working Set:** Data actively accessed and worth retaining in memory
- **TPS:** Transactions per second
- **RPS:** Requests per second
- **WPS:** Writes per second
- **Throughput:** Work completed per unit time
- **Latency:** Time required for one operation
- **Tail Latency:** Slow-end latency such as p95 or p99
- **Replication:** Maintaining copies for availability or read scale
- **Sharding:** Splitting the dataset across machines
- **Connection Pooling:** Reusing bounded database connections
- **Consumer Lag:** Queue data produced but not yet processed
- **Cache Thrashing:** Frequent eviction and repopulation with low benefit
- **Write Amplification:** Extra physical writes caused by one logical write
- **TCO:** Total Cost of Ownership
- **Scale Trigger:** Observed condition justifying more capacity or complexity

---

## 30-Second Revision Summary

- **Main lesson?** Modern machines are powerful; calculate before distributing
- **Cache latency?** Around 1 ms
- **Cache throughput?** 100k–200k+ simple ops/s
- **Database read latency?** Low single-digit ms when cached and indexed
- **Simple database writes?** Roughly 10k–20k TPS on suitable tuned systems
- **Database storage?** Tens of TiB before sharding becomes inevitable
- **App-server connections?** 100k+ in optimized configurations
- **Queue throughput?** Up to roughly 1M messages/s per optimized broker
- **Shard at 100 GB?** Usually no
- **Queue at 5k writes/s?** Not solely because of throughput
- **Caching every lookup?** No, only measured bottlenecks
- **HA same as sharding?** No
- **First database optimizations?** Indexes, pooling, tuning, replicas
- **Main app-server bottleneck?** Often CPU
- **Main queue warning?** Consumer lag and failure semantics
- **Cost approach?** Order-of-magnitude TCO, not memorized pricing
- **Best interview phrase?** Let me estimate whether one well-tuned node can handle this first
