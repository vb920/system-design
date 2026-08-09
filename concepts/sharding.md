# Database Sharding: Quick Revision Notes

## Core Thesis

**Single database limit reached → split data across machines using sharding.**

## Key Insights

- **Scale vertically before sharding**
- **Partitioning: one machine**
- **Sharding: multiple machines**
- **Shard key determines scalability**
- **Query patterns determine shard boundaries**
- **Cross-shard work creates complexity**
- **Default strategy: hash-based sharding**

## Partitioning vs Sharding

### **Partitioning**

- Large table → smaller logical pieces
- Same database instance
- Same physical machine
- Better query efficiency
- Smaller indexes
- Easier maintenance
- No additional machine capacity

#### Example

- Orders table: **500 million rows**
- Total size: **2 TB**
- Monthly query → scan one partition
- No full-table scan

### **Sharding**

- Horizontal partitioning across machines
- Each shard: independent database
- Separate CPU, memory, storage, and connection pool
- Data and traffic distributed across shards
- Horizontal storage scaling
- Horizontal read/write scaling

#### Main Costs

- Shard-key selection
- Query routing
- Hot spots
- Data rebalancing
- Cross-shard queries
- Distributed consistency

## Types of Partitioning

### **Horizontal Partitioning**

- Split by rows
- Same columns
- Fewer rows per partition
- Example: orders partitioned by year

### **Vertical Partitioning**

- Split by columns
- Same logical rows
- Fewer columns per partition
- Frequently accessed columns separated
- Large or rarely used columns separated

### Memory Trick

- **Horizontal:** Fewer rows
- **Vertical:** Fewer columns
- **Sharding:** Fewer rows per machine

## When to Shard

### **Storage Limit**

- Dataset approaching machine capacity
- Continuous data growth
- Example: 500 million users × 5 KB = 2.5 TB
- 10× growth → sharding required

### **Write Throughput Limit**

- Database write bottleneck
- Peak writes beyond one machine
- Example: **50,000 writes/second**

### **Read Throughput Limit**

- Read replicas insufficient
- Very large user base
- High aggregate query volume

### **Single-Machine Ceiling**

- CPU exhausted
- Memory exhausted
- Storage exhausted
- Connection limit reached
- Query latency increasing

**Interview rule:** Do not shard prematurely.

### Decision Flow

1. Identify bottleneck
2. Quantify capacity
3. Explain single-database limit
4. Propose sharding

## Two Sharding Decisions

### 1. **What to Shard By**

- Select shard key
- Defines data grouping
- Example: `user_id`

### 2. **How to Distribute**

- Select distribution strategy
- Defines shard assignment
- Range, hash, or directory

## Choosing a Shard Key

### **High Cardinality**

- Many unique values
- Supports many shards
- Good: `user_id`
- Bad: boolean field

### **Even Distribution**

- Similar data volume per shard
- Similar request load per shard
- Avoid dominant values
- Avoid write concentration

### **Query Alignment**

- Common queries hit one shard
- Related data remains together
- Minimal scatter-gather queries
- Minimal cross-shard transactions

**Shard-key rule:** Choose from access patterns, not schema convenience.

## Good Shard Keys

### **`user_id`**

- Millions of unique values
- Usually even distribution
- User-centric queries
- User data on one shard
- Suitable for social applications

### **`order_id`**

- High cardinality
- Even order distribution
- Point lookups by order
- Easy order-status updates
- Suitable for order tables

## Bad Shard Keys

### **`is_premium`**

- Only two values
- Maximum two groups
- Uneven user distribution
- Free-user shard overloaded

### **`created_at`**

- All new writes → newest shard
- Recent shard becomes hot
- Old shards mostly idle
- Severe write imbalance

### **`country`**

- Geographical imbalance
- Dominant country → oversized shard
- Example: 90% users in one country

## Sharding Strategies

### **Range-Based Sharding**

#### Distribution

```text
Shard 1: User IDs 1 to 1M
Shard 2: User IDs 1M to 2M
Shard 3: User IDs 2M to 3M
```

#### Advantages

- Simple routing
- Easy to understand
- Efficient range scans
- Related ranges colocated

#### Disadvantages

- Uneven traffic
- Uneven data growth
- Time-based write hot spots
- Old shards may become idle

#### Best Fit

- Range-heavy queries
- Multi-tenant systems
- Natural tenant boundaries
- Users querying separate ranges

### **Hash-Based Sharding**

#### Distribution

```text
shard = hash(shard_key) % number_of_shards
```

#### Advantages

- Even data distribution
- Even write distribution
- Avoids sequential-key hot spots
- Simple point-query routing
- Default interview strategy

#### Disadvantages

- Poor range-query support
- Resharding difficulty
- Modulo change remaps most records
- Data movement when shard count changes

#### Best Fit

- High-cardinality keys
- Point lookups
- User-centric systems
- Even workload distribution

**Interview default:** Hash-based sharding with consistent hashing.

### **Consistent Hashing**

- Alternative to simple modulo
- Reduced data movement
- Easier shard addition and removal
- Only part of dataset remapped
- Requires resharding plan

**Key purpose:** Scale shard count without moving nearly all data.

### **Directory-Based Sharding**

#### Distribution

```text
User 15  → Shard 1
User 87  → Shard 4
User 204 → Shard 2
```

#### Advantages

- Maximum placement flexibility
- Easy custom routing
- Move individual hot users
- Dedicated shards for special tenants
- Dynamic load balancing

#### Disadvantages

- Extra lookup per request
- Added request latency
- Directory as critical dependency
- Potential single point of failure
- Operational complexity

#### Best Fit

- Dynamic placement requirements
- Large enterprise tenants
- Celebrity or hot-user isolation
- Custom shard assignments

#### Interview Guidance

- Rarely the default answer
- Mention only for specific requirements
- Prepare for availability questions

## Strategy Comparison

- **Range:** Simple and range-scan friendly
- **Hash:** Even distribution and default choice
- **Directory:** Flexible but operationally expensive

### Memory Trick

- **Range:** Nearby keys together
- **Hash:** Keys spread evenly
- **Directory:** Placement stored explicitly

## Challenge 1: Hot Spots

### Problem

- One shard receives excessive traffic
- Other shards underused
- One shard becomes bottleneck
- Sharding benefit reduced

### Causes

- **Celebrity problem:** One active user overloads one shard
- **Time-based sharding:** All new writes target latest shard

### Detection

- Per-shard CPU
- Query latency
- Request volume
- Storage growth
- Connection usage

### Solutions

- **Dedicated shard:** Isolate hot user or tenant
- **Compound shard key:** Example: `hash(user_id + date)`
- **Dynamic shard splitting:** Split large or hot shards

### Database Examples

- MongoDB: automatic chunk balancing
- Vitess: operator-driven online resharding

## Challenge 2: Cross-Shard Queries

### Problem

- Query touches multiple shards
- Multiple network calls
- Wait for all responses
- Merge partial results
- Higher tail latency
- Greater failure probability

### Single-Shard Example

```text
Get user 12345's profile
```

- Route using `user_id`
- Query one shard
- Fast and simple

### Cross-Shard Example

```text
Top 10 posts globally
```

- Query every shard
- Collect local top results
- Merge globally
- Return final top 10

### Solutions

- **Cache results:** Good for leaderboards and trending data
- **Precompute results:** Periodic background aggregation
- **Denormalize data:** Duplicate data to avoid joins
- **Accept rare expensive queries:** Suitable for occasional admin reports

**Interview warning:** Frequent scatter-gather → reconsider shard key or data model.

## Challenge 3: Cross-Shard Transactions

### Problem

- Related records on different shards
- No single local transaction
- Partial-write possibility
- Coordination required

### **Two-Phase Commit: 2PC**

#### Flow

1. Coordinator asks shards to prepare
2. Shards confirm readiness
3. Coordinator sends commit
4. All shards commit

#### Advantages

- Strong consistency
- Atomic distributed transaction

#### Disadvantages

- Slow
- Blocking
- Coordinator dependency
- Failure complexity
- Stuck transactions
- Reduced availability

**Practical guidance:** Textbook solution, often avoided in production.

### **Single-Shard Transaction Design**

- Keep related data together
- Account balance on user shard
- Transaction history on user shard
- Profile information on user shard
- Fast local transactions
- Best general solution

**Design principle:** Shard boundaries should match transaction boundaries.

### **Saga Pattern**

#### Transfer Example

1. Deduct from User A on Shard 1
2. Credit User B on Shard 2
3. Step 2 fails
4. Refund User A

#### Characteristics

- Independent local steps
- Compensating actions
- Eventual consistency
- Retry and idempotency requirements

### **Eventual Consistency**

- Temporary data differences
- Values converge later
- Suitable for follower counts, likes, analytics, feeds, and counters

**Key rule:** Use strict consistency only where business correctness requires it.

## Modern Database Sharding

### **Cassandra**

- Partition-key based
- Murmur3 partitioner
- Virtual nodes
- Token ranges
- Consistent-hashing style distribution

### **DynamoDB**

- Partition-key hashing
- Internal partition routing
- Automatic partition splitting and merging
- Details hidden from users
- Not classic exposed hash-ring design

### **MongoDB**

- Shard-key based
- Range-based chunks
- Optional hashed shard key
- Background balancer
- Automatic chunk movement
- Not classic consistent hashing

### **Vitess**

- MySQL sharding layer
- Query routing
- Online resharding
- Operator-driven management

### **Citus**

- PostgreSQL distribution layer
- Distributed tables
- Query routing
- Cross-node execution

## Sharding Interview Framework

1. **Prove sharding is needed:** Estimate storage, reads, writes, and single-node limits
2. **Select shard key:** High cardinality, even distribution, and aligned access patterns
3. **Choose strategy:** Hash by default; range for range-heavy access; directory for custom placement
4. **Explain trade-offs:** Global queries, transactions, hot spots, and resharding
5. **Handle cross-shard queries:** Cache, precompute, denormalize, or background aggregation
6. **Handle growth:** Add shards, rebalance data, and monitor per-shard capacity
7. **Handle consistency:** Prefer local transactions; use sagas when unavoidable

## Key Numbers

- Example orders table: **500 million rows**
- Example orders size: **2 TB**
- Aurora maximum mentioned: **approximately 256 TiB**
- Example write load: **50,000 writes/second**
- Example shard count: **64 shards**
- Example cached global result: **5-minute TTL**

## Important Terms

- **Partitioning:** Splitting data within one database instance
- **Horizontal Partitioning:** Splitting rows into partitions
- **Vertical Partitioning:** Splitting columns into partitions
- **Sharding:** Splitting rows across multiple machines
- **Shard:** Independent database containing part of the dataset
- **Shard Key:** Field used to group and route data
- **Cardinality:** Number of unique key values
- **Range Sharding:** Distribution using continuous key ranges
- **Hash Sharding:** Distribution using a hash function
- **Consistent Hashing:** Hashing method that reduces data movement
- **Directory Sharding:** Distribution using an explicit lookup mapping
- **Hot Spot:** Shard receiving disproportionate load
- **Celebrity Problem:** One highly active key overloading its shard
- **Compound Shard Key:** Key formed from multiple attributes
- **Scatter-Gather:** Querying multiple shards and combining results
- **Resharding:** Redistributing data across a changed shard layout
- **Cross-Shard Query:** Query requiring data from multiple shards
- **Cross-Shard Transaction:** Transaction modifying multiple shards
- **Two-Phase Commit:** Coordinated protocol for atomic distributed commits
- **Saga:** Multi-step distributed operation with compensating actions
- **Denormalization:** Data duplication to simplify and accelerate reads
- **Eventual Consistency:** Temporary inconsistency followed by convergence
- **Partition Key:** Key used by a distributed database for data placement
- **Virtual Node:** Logical partition used to distribute data more evenly

## 30-Second Revision Summary

- **Partitioning?** Split data inside one database
- **Sharding?** Split data across databases
- **When?** Single machine exceeds storage or throughput
- **Main decision?** Choose the correct shard key
- **Good key?** High-cardinality, even, query-aligned
- **Default strategy?** Hash-based sharding
- **Growth strategy?** Consistent hashing or online resharding
- **Range advantage?** Efficient range scans
- **Range risk?** Uneven load and write hot spots
- **Directory advantage?** Flexible placement
- **Directory risk?** Extra latency and critical dependency
- **Main problems?** Hot spots, cross-shard queries, distributed transactions
- **Global query solution?** Cache or precompute
- **Transaction solution?** Keep related data on one shard
- **Unavoidable distributed operation?** Saga with compensation
- **Interview rule?** Prove sharding is necessary before proposing it
