# CAP Theorem: Quick Revision Notes

## Core Thesis

**During a network partition, choose either consistency or availability.**

## Key Insights

- **CAP applies to distributed systems**
- **Network partitions are unavoidable**
- **Partition tolerance is mandatory in practice**
- **Real choice during partition: consistency vs availability**
- **Choice should be feature-specific**
- **Strong consistency increases coordination cost**
- **Availability often implies temporary staleness**

## CAP Properties

### **Consistency**

- All nodes expose the same logical value
- Read reflects the latest completed write
- No stale reads
- Coordination between replicas
- Higher latency
- Lower availability during partitions

**CAP meaning:** Every read observes the latest write or receives an error.

**Important distinction:**

- CAP consistency != ACID consistency
- CAP: replica visibility
- ACID: transaction correctness and invariants

### **Availability**

- Every request receives a response
- Non-failing node remains responsive
- Response may contain stale data
- No guarantee of latest write
- Better user experience during failures
- Temporary inconsistency possible

**CAP meaning:** Every request receives a non-error response, but possibly stale data.

### **Partition Tolerance**

- System operates despite network failure
- Nodes may lose communication
- Messages may be lost or delayed
- Cluster may split into isolated groups
- Distributed operation continues where possible

Examples:

- Region-to-region link failure
- Packet loss
- Network timeout
- Data-centre isolation
- Broken switch or router
- Partial cluster disconnection

## CAP Decision

### Normal Operation

- No partition
- Consistency and availability may coexist
- Replication working normally
- Nodes communicating successfully

### During Network Partition

Choose one:

#### **CP: Consistency + Partition Tolerance**

- Reject or delay some requests
- Prevent stale or conflicting operations
- Preserve latest-value guarantees
- Reduced availability

#### **AP: Availability + Partition Tolerance**

- Continue serving requests
- Allow stale or divergent data
- Preserve responsiveness
- Reconcile differences later

**Core interview question:** During a partition, should the system reject requests or return potentially stale data?

## Important CAP Nuance

CAP does not mean:

**A distributed system permanently chooses only two properties.**

CAP actually means:

**When a partition occurs, both consistency and availability cannot always be guaranteed simultaneously.**

Practical interpretation:

- Partition absent -> consistency and availability possible
- Partition present -> choose C or A
- Partition tolerance cannot be ignored
- Choice applies to partition behaviour

**Memory trick:** Partition happens -> block for correctness or respond with possible staleness.

## Practical Example: User Profile

### Architecture

- USA server
- Europe server
- Replicated profile data
- Users routed to nearest region

### Normal Flow

1. User updates name in USA
2. Update replicated to Europe
3. European user reads profile
4. New name displayed

### Partition

- USA-Europe connection fails
- Europe cannot confirm latest value
- User requests profile

### CP Choice

- Reject or delay read
- Avoid possibly stale name
- Preserve consistency
- Lower availability

### AP Choice

- Return locally stored name
- Name may be stale
- Preserve availability
- Eventual convergence

**Preferred choice:** Public profile read -> availability

Reason:

- Old name temporarily acceptable
- Error page worse than stale profile
- No critical correctness violation

## When to Choose Consistency

**Decision signal:** Stale or conflicting data causes serious business harm.

Common requirements:

- Prevent duplicate allocation
- Maintain accurate balances
- Enforce uniqueness
- Preserve inventory constraints
- Maintain transaction ordering
- Avoid invalid state transitions

### **Ticket Booking**

- One seat
- One valid buyer
- No double-booking
- Atomic reservation
- Strong consistency
- Availability sacrificed if ownership uncertain

**Failure risk:** Two users assigned the same seat.

### **E-Commerce Inventory**

- Limited stock
- Accurate remaining quantity
- Prevent overselling
- Atomic inventory decrement
- Strong consistency for checkout

**Failure risk:** Multiple orders consume the final item.

Nuance:

- Product browsing -> availability
- Checkout inventory -> consistency

### **Financial Systems**

- Accurate balances
- Correct transaction state
- Ordered trade execution
- No duplicate transfers
- Strict business invariants

Failure risks:

- Incorrect balance
- Double spending
- Invalid trade price
- Conflicting transactions

Nuance:

- Account balance -> consistency
- Historical statement view -> possibly stale but available
- Marketing content -> availability

## When to Choose Availability

**Decision signal:** Temporary staleness causes limited harm.

Common requirements:

- Fast reads
- Global reach
- Degraded-mode operation
- High uptime
- Eventual convergence
- Better user experience

### **Social Media**

- Profile pictures
- Bios
- Like counts
- Follower counts
- Feeds
- Temporary staleness acceptable

### **Content Platforms**

- Movie descriptions
- Thumbnails
- Recommendations
- Content metadata
- Temporary old values acceptable

### **Review Sites**

- Restaurant details
- Review counts
- Ratings
- Opening hours
- Slight delay acceptable

### **Analytics**

- Dashboards
- Metrics
- Aggregated counters
- Delayed updates acceptable

## CP vs AP

### **CP System**

- Consistency prioritized
- Some requests rejected during partition
- Stronger correctness guarantees
- More coordination
- Higher latency
- Lower partition-time availability

Suitable for:

- Seat reservations
- Inventory checkout
- Account balances
- Payment state
- Unique username allocation
- Distributed locks

### **AP System**

- Availability prioritized
- Requests continue during partition
- Stale or conflicting values possible
- Eventual reconciliation
- Lower coordination
- Better partition-time responsiveness

Suitable for:

- Social feeds
- User profiles
- Product descriptions
- Comments
- Recommendations
- Analytics

## CAP Classification Warning

Systems are not always purely CP or AP:

- Behaviour may be configurable
- Different operations may use different guarantees
- Reads and writes may behave differently
- Quorum settings may change trade-offs
- Features may require separate models

Avoid:

> The entire system is CP.

Prefer:

> Seat booking prioritizes consistency during a partition, while event browsing remains available and may return slightly stale data.

## Feature-Level CAP Decisions

**Core principle:** Choose consistency per operation, not necessarily per application.

Why?

- Features have different failure consequences
- One global consistency model may be wasteful
- Critical writes need stronger guarantees
- Non-critical reads benefit from availability

## Example 1: Ticketing System

### **Booking a Seat**

- Strong consistency
- Atomic reservation
- No double-booking
- Reject requests during uncertain ownership
- CP-oriented operation

### **Viewing Event Details**

- Availability preferred
- Cached data acceptable
- Slightly stale descriptions acceptable
- AP-oriented operation

Interview phrase:

> I'll prioritize consistency for seat reservations, but availability for event discovery and event-detail reads.

## Example 2: Matching Application

### **Creating a Match**

- Consistent match state
- Idempotent match creation
- No duplicate match records
- Reliable mutual-like detection
- Stronger coordination

### **Viewing Profiles**

- Availability preferred
- Stale picture acceptable
- Stale bio acceptable
- Low correctness risk

Interview phrase:

> I'll use stronger consistency for match creation and prioritize availability for profile browsing.

## Consistency Models

### **Strong Consistency**

- Reads observe latest completed write
- Single logical system state
- Highest coordination requirement
- Increased latency
- Reduced availability during partitions

Suitable for:

- Bank balances
- Seat ownership
- Inventory reservation
- Payment state
- Distributed locks

### **Causal Consistency**

- Cause appears before effect
- Related events preserve order
- Concurrent unrelated events may differ
- Weaker than strong consistency
- Stronger than eventual consistency

Example:

1. Post created
2. Comment added
3. Readers see post before comment

Suitable for:

- Social interactions
- Comments and replies
- Collaborative activity
- Message conversations

### **Read-Your-Own-Writes Consistency**

- User immediately sees own update
- Other users may temporarily see old value
- Session-level guarantee
- Good perceived consistency
- Lower global coordination cost

Example:

- User changes profile picture
- User immediately sees new picture
- Other users may see old picture briefly

Suitable for:

- Profile editing
- Settings updates
- User-generated posts
- Personal dashboards

### **Eventual Consistency**

- Replicas may temporarily differ
- Updates propagate asynchronously
- Values converge over time
- High availability
- Lower write latency
- Conflict handling may be required

Suitable for:

- DNS
- Social feeds
- Like counts
- Recommendations
- Analytics
- Content metadata

## Consistency Spectrum

```text
Strong
  |
Causal
  |
Read-your-own-writes
  |
Eventual
```

General trade-off:

```text
Stronger consistency
-> more coordination
-> higher latency
-> lower partition-time availability
```

```text
Weaker consistency
-> less coordination
-> lower latency
-> higher availability
-> possible stale reads
```

## Designing for Consistency

Possible techniques:

- Single authoritative writer
- Leader-based replication
- Synchronous replication
- Quorum reads and writes
- Consensus protocols
- Conditional writes
- Compare-and-set
- Version checks
- Distributed transactions
- Idempotency keys

### **Synchronous Replication**

- Wait for replicas before success
- Stronger durability and freshness
- Higher write latency
- Partition may block writes

### **Consensus Protocol**

- Nodes agree on operation order
- Maintains replicated state
- Majority required for progress
- Minority side becomes unavailable

Examples:

- Raft
- Paxos-family protocols

### **Conditional Write**

- Update only if expected version matches
- Prevent lost updates
- Detect concurrent modification
- Useful for inventory and reservations

### **Single-Node Solution**

- One source of truth
- Simple consistency model
- No cross-replica propagation
- Limited horizontal scalability
- Potential availability bottleneck
- Still requires backup and failover planning

### **Distributed Transaction**

- Coordinates multiple resources
- Stronger atomicity
- Higher latency
- Failure-handling complexity
- Reduced availability risk

Important nuance:

**Distributed transactions are not CAP consistency by themselves.**

- Transaction atomicity != replica freshness
- Two-phase commit != full partition tolerance
- Different distributed-system concerns

## Designing for Availability

Possible techniques:

- Multiple replicas
- Asynchronous replication
- Local-region reads
- Multi-region deployment
- Event-driven propagation
- Change Data Capture
- Conflict resolution
- Retry and reconciliation
- Stale cache serving

### **Asynchronous Replication**

- Primary acknowledges before replicas update
- Lower write latency
- Replica lag possible
- Better availability
- Potential stale reads

### **Change Data Capture**

- Capture committed database changes
- Publish change events
- Update replicas asynchronously
- Update caches and search indexes
- Eventual propagation

Challenges:

- Duplicate events
- Out-of-order delivery
- Consumer lag
- Retry handling
- Idempotency

### **Conflict Resolution**

- Reconcile divergent writes
- Application-specific rules
- Version vectors
- Last-write-wins
- Merge functions
- Manual resolution for critical conflicts

## Quorum-Based Trade-Offs

Variables:

- `N` = number of replicas
- `W` = write acknowledgements required
- `R` = read replicas queried

Common rule:

```text
R + W > N
```

- Read and write quorums overlap
- Better chance of observing latest write
- Higher coordination cost

Example:

```text
N = 3
W = 2
R = 2
```

- Write waits for two replicas
- Read checks two replicas
- Overlap expected
- Increased consistency
- Reduced partition-time availability

Availability-oriented configuration:

```text
N = 3
W = 1
R = 1
```

- Fast operations
- Fewer required nodes
- Better availability
- Greater stale-read risk

**Important nuance:** Quorum settings influence the trade-off but do not automatically guarantee linearizability.

## Technology Examples

### **PostgreSQL / MySQL**

- Strong local transactions
- ACID guarantees
- Primary-replica architectures
- Replica reads may be stale
- Distributed behaviour depends on configuration

### **Google Spanner**

- Distributed SQL
- Strong transaction support
- Synchronous coordination
- Higher coordination cost

### **DynamoDB Strongly Consistent Reads**

- Optional stronger read guarantee
- Higher resource cost
- Region and feature limitations may apply
- Behaviour depends on selected operation

### **Cassandra**

- Distributed architecture
- Tunable consistency
- Multiple replicas
- Quorum configuration
- Eventual consistency options
- Conflict reconciliation

### **Dynamo-Style Databases**

- Partitioned data
- Replica-based availability
- Tunable read/write guarantees
- Eventual consistency options

### **Redis Cluster**

- Partitioned in-memory storage
- Replica-based failover
- Fast operations
- Availability and data-loss behaviour depend on configuration
- Not automatically an AP database in every mode

## CAP and PACELC

### CAP Limitation

- Focuses on partition conditions
- Does not describe normal-operation trade-offs
- Says little about latency without partitions

### PACELC Extension

```text
If Partition:
    choose Availability or Consistency
Else:
    choose Latency or Consistency
```

Meaning:

- **P:** Partition occurs
- **A/C:** Availability vs consistency
- **E:** Else, normal operation
- **L/C:** Latency vs consistency

**Practical insight:** Even without a partition, stronger consistency usually costs latency.

## Common Misconceptions

### **Choose Any Two of Three**

Oversimplified.

- Partition tolerance is unavoidable in distributed systems
- Real choice occurs during a partition
- Choose consistency or availability

### **AP Means No Consistency**

Incorrect.

- AP systems may provide eventual consistency
- Session guarantees may still exist
- Causal ordering may still exist
- Data can converge after partition recovery

### **CP Means Entire System Stops**

Incorrect.

- Only affected operations may fail
- Majority partition may continue
- Minority partition may reject requests
- Unrelated features may remain available

### **Consistency Means ACID**

Incorrect.

- CAP consistency: latest-value visibility
- ACID consistency: valid transaction state
- Separate concepts

### **Replication Always Improves Availability**

Incomplete.

- Asynchronous replicas improve read availability
- Synchronous quorum may reduce write availability
- Failover requires detection and coordination
- Replication configuration determines behaviour

### **Eventual Consistency Means Random Data**

Incorrect.

- Temporary replica divergence
- Defined propagation process
- Eventual convergence expected
- Conflict resolution required

## CAP Interview Framework

1. **Identify critical operations:** Reads, writes, scarce resources, and invariants
2. **Ask the failure question:** What happens during a network partition?
3. **Estimate staleness impact:** Financial loss, duplicate allocation, or minor UX issue
4. **Choose per feature:** Consistency, availability, session guarantee, or eventual convergence
5. **Explain user behaviour:** Reject, delay, serve stale data, queue, or reconcile
6. **Select mechanisms:** Leader, quorum, consensus, replicas, CDC, or conflict resolution
7. **Discuss trade-offs:** Latency, staleness, complexity, recovery, and user experience

## Interview Answer Template

> Because this is a distributed system, I'll assume network partitions can occur. The relevant CAP decision is therefore how each operation behaves during a partition. For correctness-critical operations such as seat reservation, I'll prioritize consistency and reject or delay the request if the system cannot confirm ownership. For event browsing, I'll prioritize availability and serve potentially stale data from a local replica or cache. This gives strong guarantees where conflicting state would cause business harm while preserving availability for non-critical reads.

## Quick Decision Questions

### Choose Consistency If

- Must every read reflect latest committed write?
- Can stale data cause financial loss?
- Can two users claim the same resource?
- Must uniqueness be preserved?
- Is an incorrect response worse than an error?

### Choose Availability If

- Is temporary staleness acceptable?
- Is an old value better than an error?
- Can conflicts be reconciled later?
- Is the feature read-heavy?
- Is uptime more important than immediate freshness?

## Decision Examples

### Strong Consistency

- Seat reservation
- Inventory checkout
- Bank balance update
- Payment status transition
- Username uniqueness
- Distributed lock ownership

### Availability

- Product catalogue
- Social feed
- User profile
- Recommendation list
- Movie description
- Review count
- Analytics dashboard

### Mixed Model

- Product browsing -> availability
- Product purchase -> consistency
- Event browsing -> availability
- Seat booking -> consistency
- Profile viewing -> availability
- Match creation -> consistency

## Important Terms

- **CAP Theorem:** Partition-time trade-off between consistency and availability
- **Consistency:** Reads observe the latest completed write or fail
- **Availability:** Every request to a non-failing node receives a non-error response
- **Partition Tolerance:** Continued defined operation despite lost or delayed communication
- **Network Partition:** Nodes unable to communicate reliably
- **CP System:** Consistency prioritized during partition
- **AP System:** Availability prioritized during partition
- **Strong Consistency:** Reads observe the latest committed value
- **Causal Consistency:** Causes appear before dependent effects
- **Read-Your-Own-Writes:** User sees their own updates immediately
- **Eventual Consistency:** Replicas converge after temporary divergence
- **Replica:** Copy of data stored on another node
- **Replica Lag:** Delay before a replica receives an update
- **Quorum:** Minimum number of replicas required for an operation
- **Consensus:** Agreement among nodes on replicated state or operation order
- **Synchronous Replication:** Write waits for replica acknowledgement
- **Asynchronous Replication:** Write succeeds before all replicas update
- **CDC:** Change Data Capture for propagating committed changes
- **Conflict Resolution:** Reconciliation of divergent values
- **Stale Read:** Read returning an older value
- **Linearizability:** Operations appear atomic and ordered in real time
- **PACELC:** Partition trade-off plus normal-operation latency trade-off

## 30-Second Revision Summary

- **CAP properties?** Consistency, availability, partition tolerance
- **Partition tolerance optional?** No, not for realistic distributed systems
- **Real choice?** Consistency vs availability during partition
- **CP behaviour?** Reject or delay uncertain operations
- **AP behaviour?** Respond with potentially stale data
- **Consistency use case?** Seat booking or payment state
- **Availability use case?** Profiles, feeds, and content metadata
- **Can one application use both?** Yes, per feature
- **Strong consistency cost?** Coordination, latency, and lower availability
- **Eventual consistency benefit?** Availability and lower latency
- **CAP consistency same as ACID?** No
- **Does consistent hashing relate to CAP consistency?** No
- **Advanced extension?** PACELC
- **Best interview question?** Is stale data worse than no response?
- **Best design rule?** Apply the weakest consistency model that preserves correctness
