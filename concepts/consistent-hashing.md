# Consistent Hashing: Quick Revision Notes

## Core Thesis

**Distribute keys across changing server clusters with minimal data movement.**

## Key Insights

- **Modulo hashing breaks during cluster resizing**
- **Hash ring minimizes redistribution**
- **Virtual nodes improve structural balance**
- **Consistent hashing distributes keys, not traffic**
- **Replication provides fault tolerance**
- **Hot keys require separate mitigation**
- **Usually handled by distributed infrastructure**

## Problem Being Solved

### Initial Architecture

- One database
- All event data together
- Simple request routing
- Easy transactions
- Limited machine capacity

### Growth Problem

- More users
- More events
- Higher read traffic
- Higher write traffic
- Larger storage requirements
- Single database bottleneck

### Scaling Requirement

**Split data across multiple databases.**

### Main Question

**Which database should store each record?**

## First Attempt: Modulo Hashing

### Formula

```text
database_id = hash(event_id) % number_of_databases
```

### Example: Three Databases

```text
Event 1234 → hash(1234) % 3 → Database 1
Event 5678 → hash(5678) % 3 → Database 0
Event 9012 → hash(9012) % 3 → Database 2
```

### Advantages

- Simple
- Fast routing
- Deterministic placement
- Reasonably even distribution
- Easy implementation

### Main Problem

**Shard count is part of the placement formula.**

## Problem 1: Adding a Node

### Before

```text
hash(event_id) % 3
```

### After

```text
hash(event_id) % 4
```

### Result

- Modulo divisor changes
- Most key mappings change
- Data moves between existing nodes
- New node not the only affected node
- Massive redistribution
- Database load spikes
- Slow or unavailable requests

### Key Issue

**Adding one node can remap almost the entire dataset.**

## Problem 2: Removing a Node

### Before

```text
hash(event_id) % 3
```

### After

```text
hash(event_id) % 2
```

### Result

- Most key mappings change
- Unaffected nodes also redistribute data
- Large recovery workload
- Increased database pressure
- Slow failure recovery

### Key Issue

**Removing one node can remap data from every node.**

## Modulo vs Consistent Hashing

### **Modulo Hashing**

- Placement depends on node count
- Node-count change → widespread remapping
- Simple implementation
- Expensive resizing
- Poor cluster elasticity

### **Consistent Hashing**

- Placement based on hash-ring positions
- Node-count change → local remapping
- Limited data movement
- Better cluster elasticity
- Suitable for dynamic membership

## Consistent Hash Ring

### Core Model

- Fixed circular hash space
- Keys placed on ring
- Nodes placed on ring
- Clockwise ownership rule
- Ring wraps at maximum value

### Simplified Hash Space

```text
0 to 99
```

### Realistic Hash Space

```text
0 to 2^32 - 1
```

### Example Node Positions

```text
DB1 → 0
DB2 → 25
DB3 → 50
DB4 → 75
```

## Key Placement

### Routing Flow

1. Hash the key
2. Find its ring position
3. Move clockwise
4. Select first database node
5. Store or retrieve the key there

### Ownership Rule

**A key belongs to the first node clockwise from its hash position.**

### Wrap-Around

- Key after final node
- Continue from ring start
- Last range owned by first node

### Memory Trick

**Hash → ring position → walk clockwise → first node**

## Adding a Database

### Example

- Existing nodes: 0, 25, 50, 75
- New node: position 90

### Data Movement

- Only keys between 75 and 90 move
- Keys previously owned by DB1
- Keys transferred to new DB5
- All other mappings unchanged

### Example Impact

- Moved range: 15 ring units
- Total ring: 100 units
- Approximate moved data: **15%**
- Far less than modulo remapping

### Key Benefit

**New node takes only a neighbouring key range.**

## Removing a Database

### Example

- DB2 located at position 25
- DB2 removed from ring

### Data Movement

- Only DB2-owned keys move
- Range 0–25 affected
- Keys move clockwise to DB3
- Other mappings remain unchanged

### Key Benefit

**Only failed node’s ownership range is reassigned.**

## Expected Data Movement

### Modulo Hashing

- Shard-count change affects most keys
- Potentially near-total redistribution

### Consistent Hashing

For `N` evenly balanced nodes:

```text
Expected moved fraction ≈ 1 / N
```

### Example

- 4 existing nodes
- Add 1 new balanced node
- Approximately one-fifth of final keyspace moves
- Exact movement depends on ring placement and balance

## Remaining Problem: Uneven Load

### Single Position per Node

- Each physical node appears once
- Each node owns one continuous range
- Range sizes may differ
- Node removal transfers load to one neighbour
- One successor may become overloaded

### Example

- DB2 fails
- Entire DB2 range moves to DB3
- DB3 may receive roughly double load
- Other databases receive no transferred load

### Required Improvement

**Spread each physical node across multiple ring positions.**

## Virtual Nodes

### Definition

- Multiple logical positions per physical server
- Also called virtual nodes or vnodes
- Each physical node owns several small ranges
- Virtual positions intermixed around ring

### Example

```text
DB1-vn1
DB1-vn2
DB1-vn3
DB1-vn4
```

### Placement

- Hash each virtual-node identifier
- Place resulting positions on ring
- Repeat for every physical server

## Virtual-Node Benefits

### **Better Key Distribution**

- Smaller ownership ranges
- More even key placement
- Reduced accidental imbalance
- Improved storage utilization

### **Better Failure Redistribution**

When DB2 fails:

- `DB2-vn1` range → DB1
- `DB2-vn2` range → DB3
- `DB2-vn3` range → DB4
- Remaining ranges → other successors

#### Result

- Failed-node load spread across survivors
- No single successor takes everything
- Lower overload risk

### **Better Node Addition**

- New node receives many virtual positions
- Each position takes a small range
- Data collected from multiple existing nodes
- Balanced capacity from the beginning

### **Heterogeneous Capacity**

- Powerful server → more virtual nodes
- Smaller server → fewer virtual nodes
- Ownership weighted by capacity

### Trade-Off

- More virtual nodes → better balance
- More virtual nodes → more routing metadata
- More virtual nodes → more bookkeeping

## Structural vs Workload Balance

### **Structural Imbalance**

- Uneven number of keys
- Uneven storage ownership
- Uneven ring ranges
- Caused by node placement

#### Main Solution

**Virtual nodes**

### **Workload Imbalance**

- Keys evenly distributed
- Some keys far more popular
- Uneven request traffic
- Caused by access patterns

#### Main Solutions

- Replication
- Key-space salting
- Adaptive rebalancing
- Caching

### Interview Distinction

**Virtual nodes balance keys; replication and salting balance traffic.**

## Hot Spots

### Definition

- One key receives disproportionate traffic
- Owning node becomes overloaded
- Key distribution may still be perfectly even

### Example

- Popular concert event
- 100× normal read traffic
- All requests target same event key
- One owning node receives all traffic

### Important Limitation

**Consistent hashing distributes keys, not request popularity.**

## Hot-Spot Solutions

### **Read Replicas**

- Copy hot data to multiple nodes
- Load-balance read requests
- Increase read capacity
- Most common solution
- Replication-lag consideration

### **Key-Space Salting**

#### Original Key

```text
taylor-swift
```

#### Salted Keys

```text
taylor-swift-0
taylor-swift-1
...
taylor-swift-9
```

#### Characteristics

- Variants hash to different nodes
- Requests spread across cluster
- Reads may require selection or aggregation
- Writes update multiple variants
- Added application complexity

### **Adaptive Rebalancing**

- Monitor real-time traffic
- Detect overloaded partitions
- Move hot key ranges
- Dynamic load redistribution
- Operationally complex
- Sometimes automated by managed systems

### **Local or Distributed Caching**

- Cache popular values
- Reduce database reads
- Serve repeated traffic quickly
- Useful for read-heavy hot keys

## Consistent Hashing vs Replication

### **Consistent Hashing**

- Determines key ownership
- Chooses responsible nodes
- Minimizes reassignment
- Supports membership changes
- Does not create data copies
- Does not provide durability alone

### **Replication**

- Creates multiple data copies
- Supports failover
- Improves read availability
- Protects against node failure
- Requires consistency coordination

### Combined Role

- Consistent hashing → **where data belongs**
- Replication → **where copies exist**
- Consensus or quorum → **which copy is valid**

## Node Failure in Practice

### Textbook View

- Node fails
- Its ranges move to successor nodes
- Data copied to new owners

### Production View

- Data already replicated
- Replica serves traffic immediately
- Failed primary replaced or bypassed
- No immediate large-scale transfer required
- Re-replication restores redundancy later

### Key Insight

**Replication handles immediate failure; consistent hashing controls long-term placement.**

## Replication on the Ring

### Typical Pattern

- Key assigned to primary node
- Copies stored on next `N - 1` nodes
- Replication factor defines copy count

### Example

```text
Replication factor = 3
```

- One primary copy
- Two replica copies
- Three total placements

### Failure Behaviour

- Primary fails
- Surviving replica serves requests
- Membership metadata updated
- Missing replica recreated later

## Planned Data Movement

### Common Triggers

- Adding capacity
- Permanent node replacement
- Removing a node
- Restoring replication factor
- Rebalancing uneven ownership

### Consistent-Hashing Benefit

- Bounded affected key ranges
- No full-cluster redistribution
- Lower migration cost
- Reduced operational impact

## Real-World Uses

### **Distributed Databases**

- Partition ownership
- Data routing
- Cluster resizing
- Replica placement

### **Distributed Caches**

- Cache-key assignment
- Client-side routing
- Reduced cache invalidation during resizing
- Better cache-hit retention

### **Message Brokers**

- Topic or partition assignment
- Consumer placement
- Work distribution

### **CDNs**

- Content-to-edge assignment
- Cache ownership
- Stable placement during edge changes

### **Application Servers**

- Session affinity
- Request routing
- Stateful workload placement

## System Examples

### **Apache Cassandra**

- Ring-based partitioning
- Partition-key hashing
- Token ranges
- Virtual nodes
- Replication across nodes

### **Dynamo-Style Systems**

- Hash-based partition placement
- Replicated partitions
- Minimized movement during scaling
- Exact implementation may differ from textbook ring

### **CDNs**

- Stable content assignment
- Reduced cache churn
- Edge membership changes
- Efficient cache utilization

### Important Nuance

**Real systems may use consistent-hashing principles without a literal textbook ring.**

## Fixed Hash Slots

### Alternative Model

- Fixed number of logical slots
- Key hashes to one slot
- Slots assigned to physical nodes
- Rebalancing moves slot ownership
- Node count excluded from key formula

### Redis Cluster Example

```text
slot = CRC16(key) % 16384
```

### Characteristics

- **16,384 fixed slots**
- Slots mapped to Redis nodes
- Resharding moves selected slots
- Explicit and predictable ownership
- More coordination during rebalancing

## Consistent Hashing vs Fixed Slots

### **Consistent Hashing**

- Circular hash space
- Clockwise ownership
- Minimal local reassignment
- Virtual nodes for balance
- Flexible membership

### **Fixed Hash Slots**

- Predefined logical partitions
- Explicit slot-to-node mapping
- Easier ownership inspection
- Controlled slot migration
- Additional rebalancing coordination

### Shared Principle

**Separate key placement from physical node count.**

## Common Misconceptions

### **“Consistent” Means Strong Consistency**

Incorrect.

- “Consistent” refers to stable key placement
- Not transactional consistency
- Not linearizability
- Not replica agreement

### **Consistent Hashing Prevents Hot Keys**

Incorrect.

- Balances key ownership
- Does not balance key popularity
- Hot traffic needs replication or salting

### **No Data Moves During Resizing**

Incorrect.

- Some ranges still move
- Goal: minimize movement
- Not eliminate movement

### **Virtual Nodes Provide Replication**

Incorrect.

- Virtual nodes improve distribution
- Replication creates data copies
- Different responsibilities

### **Every Distributed Database Uses a Ring**

Incorrect.

- Some use fixed slots
- Some use range partitions
- Some use directory metadata
- Implementations vary

## Interview Use Cases

### Mention Briefly When Using Managed Systems

Examples:

- DynamoDB
- Cassandra
- Distributed caches
- Managed distributed databases

#### Suggested Phrase

> The platform handles partition placement and rebalancing internally using hash-based distribution, so application code only needs an appropriate partition key.

### Explain Deeply For Infrastructure Designs

- Distributed database
- Distributed cache
- Distributed message broker
- Custom partition router
- Storage cluster
- CDN routing layer

## Interview Explanation Flow

1. **State the problem:** Deterministic placement under changing membership
2. **Explain modulo failure:** Changing `N` changes most mappings
3. **Introduce hash ring:** Hash nodes and keys; walk clockwise
4. **Explain membership change:** Only nearby or owned ranges move
5. **Introduce virtual nodes:** Better balance and failure redistribution
6. **Address hot keys:** Replicas, salting, caching, or adaptive movement
7. **Address fault tolerance:** Replication plus failover and re-replication
8. **Mention alternatives:** Fixed slots, ranges, or directory placement

## Interview Answer Template

> Simple modulo hashing uses `hash(key) % N`, but changing the number of nodes remaps most keys. Consistent hashing places nodes and keys on a hash ring, with each key assigned to the first node clockwise. Adding or removing a node therefore affects only nearby ranges. Virtual nodes place each physical server at multiple ring positions, improving balance and spreading failure load. Consistent hashing balances key ownership, not traffic, so hot keys still require replication, salting, or caching. In production, replication handles immediate failures, while consistent hashing controls placement and bounded rebalancing.

## Trade-Offs

### Advantages

- Minimal key remapping
- Easier horizontal scaling
- Stable request routing
- Reduced migration load
- Better handling of node changes
- Applicable across distributed systems
- Supports heterogeneous node capacity

### Disadvantages

- Ring metadata management
- More complex than modulo
- Uneven balance without virtual nodes
- Hot keys remain possible
- Replication still required
- Data migration still necessary
- Membership coordination required
- Harder range scans

## Important Terms

- **Hash Function:** Converts a key into a numeric value
- **Modulo Hashing:** Assigns a key using `hash(key) % N`
- **Consistent Hashing:** Hash-based placement with minimal remapping during membership changes
- **Hash Ring:** Circular representation of a fixed hash space
- **Clockwise Lookup:** Selecting the first node clockwise from a key
- **Physical Node:** Actual server or database instance
- **Virtual Node:** Logical ring position owned by a physical node
- **Token:** Position or boundary in a hash space
- **Token Range:** Portion of hash space owned by a node
- **Structural Imbalance:** Uneven key or storage distribution
- **Workload Imbalance:** Uneven request traffic
- **Hot Key:** Disproportionately popular key
- **Hot Spot:** Overloaded node or partition
- **Key Salting:** Creating key variants to spread load
- **Replication Factor:** Number of stored copies
- **Primary:** Main owner of a partition
- **Replica:** Additional copy used for availability
- **Failover:** Switching traffic to a surviving replica
- **Rebalancing:** Redistributing ownership after cluster changes
- **Re-replication:** Creating replacement copies after failure
- **Membership Change:** Node addition, removal, or replacement
- **Fixed Hash Slot:** Logical partition mapped separately to a node
- **Bounded Movement:** Only a limited fraction of data changes owners

## Key Numbers

- Simplified ring: **0–99**
- Common conceptual hash space: **0 to 2³² − 1**
- Example databases: **4**
- New database position: **90**
- Example moved range: **75–90**
- Approximate moved dataset: **15%**
- Redis Cluster slots: **16,384**
- Example replication factor: **3 copies**

## 30-Second Revision Summary

- **Problem?** Modulo hashing remaps most keys when `N` changes
- **Solution?** Consistent hashing
- **Core structure?** Hash ring
- **Key owner?** First node clockwise
- **Node added?** Only nearby ranges move
- **Node removed?** Only owned ranges move
- **Balance improvement?** Virtual nodes
- **VNode purpose?** Spread key ranges across physical nodes
- **Hot-key limitation?** Keys balanced, traffic not balanced
- **Hot-key solutions?** Replication, salting, caching
- **Failure handling?** Serve existing replicas
- **Long-term recovery?** Re-replicate affected ranges
- **Redis alternative?** Fixed 16,384 hash slots
- **Interview depth?** Deep for infrastructure designs, brief for managed databases
- **Memory trick?** Place on ring, walk clockwise
