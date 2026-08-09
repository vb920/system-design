# Data Modeling: Quick Revision Notes

## Core Thesis

**Structure data around entities, access patterns, consistency needs, and expected scale.**

## Key Insights

- **Start from requirements, not database technology**
- **Access patterns drive schema design**
- **PostgreSQL is the safe interview default**
- **Use stable system-generated primary keys**
- **Normalize first; denormalize only with justification**
- **Indexes should map directly to important queries**
- **Shard only after proving a single database is insufficient**
- **Keep related data and transactions together**

## What Is Data Modeling?

- Define application data
- Identify entities
- Choose attributes
- Define identifiers
- Define relationships
- Add constraints
- Support access patterns
- Plan storage and scaling

**Main goal:** A clear, functional schema aligned with system requirements.

## Interview Expectations

### Usually Expected

- Core entities
- Important fields
- Primary keys
- Foreign keys
- Main relationships
- Important constraints
- Query-supporting indexes
- Possible partition or shard key

### Usually Not Expected

- Complete production schema
- Every column
- Full normalization exercise
- Detailed entity-relationship diagram
- All migration strategies
- Database-internals discussion

**Interview principle:** Design enough schema to prove the system works, then continue to architecture.

## When Data Modeling Appears

### 1. Requirements Gathering

- Identify core entities
- Identify main operations
- Identify critical relationships
- Identify consistency boundaries

Example entities:

- User
- Post
- Comment
- Like
- Follow

### 2. High-Level Design

- Place schema near database component
- List important columns
- Mark primary and foreign keys
- Add indexes
- Mention partitioning or sharding
- Connect schema to APIs

## Basic Social Media Schema

```text
Users
- user_id          PK
- username         UNIQUE
- email            UNIQUE
- created_at

Posts
- post_id          PK
- user_id          FK -> Users.user_id
- content
- media_urls
- created_at
- INDEX(user_id, created_at)

Comments
- comment_id       PK
- post_id          FK -> Posts.post_id
- user_id          FK -> Users.user_id
- content
- created_at
- INDEX(post_id, created_at)

Likes
- user_id          FK -> Users.user_id
- post_id          FK -> Posts.post_id
- created_at
- PK(user_id, post_id)
- INDEX(post_id)
```

Supported queries:

- Fetch user by ID
- Fetch user by email
- Fetch recent posts by user
- Fetch comments for post
- Check whether user liked post
- Count or list likes for post

## Database Selection Rule

**Safe default:** Use PostgreSQL unless requirements clearly demand another model.

### Why PostgreSQL?

- Familiar relational model
- ACID transactions
- Constraints
- Foreign keys
- Flexible indexing
- Complex query support
- Read replicas
- Partitioning
- Sharding options
- JSON support when needed

### Avoid

- Exotic database without requirement
- Technology chosen for appearance
- Database-first reasoning
- Graph database for every relationship
- NoSQL without access-pattern justification

## Database Model Options

1. **Relational database**
2. **Document database**
3. **Key-value store**
4. **Wide-column database**
5. **Graph database**

## Relational Databases

### Structure

- Tables
- Rows as records
- Columns as attributes
- Fixed or controlled schema
- Foreign-key relationships
- Database constraints

### Main Strengths

- ACID transactions
- Referential integrity
- Complex queries
- Joins
- Unique constraints
- Mature indexing
- Strong tooling

### Best Fit

- Structured entities
- Clear relationships
- Transactional workflows
- Strong consistency
- Multi-table operations
- E-commerce
- Social applications
- Payments
- Inventory

### Examples

- PostgreSQL
- MySQL
- SQLite

### Trade-Offs

- Rigid schema
- Migration coordination
- Join cost at scale
- Cross-shard joins difficult
- Write scaling harder than read scaling
- Deeply nested structures less natural

**Interview warning:** Complex joins in a common high-traffic path may require a denormalized view, cache, materialized view, or precomputed result.

## Document Databases

### Structure

- JSON-like documents
- Flexible schema
- Nested objects
- Embedded collections
- Per-document organization

```json
{
  "_id": "user-123",
  "username": "john_doe",
  "email": "john@example.com",
  "posts": [
    {
      "post_id": "post-1",
      "content": "Hello, world!",
      "created_at": "2024-01-01T10:00:00Z"
    }
  ]
}
```

### Main Strengths

- Flexible fields
- Natural nested data
- Rapid schema evolution
- Single-document retrieval
- Fewer joins
- Aggregate-oriented storage

### Main Costs

- Data duplication
- Larger documents
- Complex partial updates
- Document-size limits
- Cross-document consistency
- Difficult many-to-many relationships
- Unbounded-array risk

### Best Fit

- Rapidly evolving schema
- Different record shapes
- Nested aggregate data
- Document-centric access
- Few cross-document joins

### Examples

- MongoDB
- Firestore
- CouchDB

## Embed vs Reference

### **Embed**

Use when:

- Data read together
- Child lifecycle follows parent
- Child collection remains bounded
- Atomic document update useful

Advantages:

- One read
- No join
- Natural aggregate

Disadvantages:

- Duplication
- Large documents
- Expensive parent updates
- Unbounded growth risk

### **Reference**

Use when:

- Relationship is many-to-many
- Child accessed independently
- Child collection is unbounded
- Child updated frequently
- Data shared across parents

Advantages:

- Less duplication
- Independent updates
- Smaller documents

Disadvantages:

- Additional queries
- Application-side joining
- More complex read path

**Core rule:** Embed bounded child data; reference unbounded child data.

## Key-Value Stores

### Structure

```text
key -> value
```

### Characteristics

- Exact-key lookup
- Flat access pattern
- Very low latency
- Limited secondary querying
- No joins
- Heavy denormalization

### Best Fit

- Caching
- Session storage
- Feature flags
- Rate limiting
- Shopping carts
- Idempotency records
- Precomputed responses

### Examples

- Redis
- Memcached
- DynamoDB for key-oriented access patterns

### Query-First Keys

```text
user:123:profile
post:456
session:abc123
feed:user:123
feature:checkout-v2
```

**Main principle:** Design keys from lookup patterns.

### Costs

- Duplicate representations
- Difficult ad hoc querying
- Manual relationship management
- Consistency across keys
- Key invalidation complexity

## SQL Plus Key-Value Cache

```text
Client
  |
Application
  |
Cache lookup
  +-- Hit  -> return cached value
  +-- Miss -> query PostgreSQL
              |
              populate cache
```

### PostgreSQL Role

- Source of truth
- Durable storage
- Transactions
- Complex queries

### Redis Role

- Fast lookup
- Flattened read model
- Hot data
- Sessions
- Counters

**Key principle:** Key-value stores often complement SQL rather than replace it.

## Wide-Column Databases

### Structure

- Partition keys
- Clustering or sort keys
- Column families
- Sparse columns
- Query-oriented tables
- Distributed storage

### Best Fit

- Massive write throughput
- Time-series data
- Event logging
- Telemetry
- IoT data
- Append-heavy workloads
- Predictable query patterns

### Examples

- Cassandra
- HBase

```text
Partition key: user_id
Clustering key: created_at

UserPosts
- user_id
- created_at
- post_id
- content
```

### Modeling Principle

**Create tables for queries, not normalized entities.**

### Costs

- Data duplication
- Application-managed consistency
- Limited ad hoc queries
- Partition-size planning
- Hot-partition risk

### Good Partition Key

- High cardinality
- Even distribution
- Matches query scope
- Bounded partition size
- Avoids hot keys

### Good Clustering Key

- Supports sorting
- Supports range queries
- Often time-based
- Unique within partition

```text
PRIMARY KEY ((user_id), created_at)
```

## Graph Databases

### Structure

- Nodes
- Edges
- Properties
- Relationship traversal

### Best Fit

- Variable-depth traversal
- Fraud rings
- Network topology
- Knowledge graphs
- Dependency graphs
- Multi-hop relationship queries

### Examples

- Neo4j
- Amazon Neptune

### Main Benefit

**Efficient relationship traversal without repeated relational joins.**

### Costs

- Specialized operations
- Additional infrastructure
- Different query language
- Harder horizontal scaling
- Smaller ecosystem
- Transactional data may still need SQL
- Unnecessary for simple one-hop relationships

**Interview guidance:** Use only when deep graph traversal is a primary requirement.

## Database Model Comparison

### **Relational**

- Strong schema
- Transactions
- Joins
- Constraints
- Default choice

### **Document**

- Flexible schema
- Nested aggregates
- Document-oriented reads
- Denormalized structure

### **Key-Value**

- Exact-key access
- Very fast
- Simple queries
- Cache or lookup layer

### **Wide-Column**

- Massive write scale
- Query-specific tables
- Partition-oriented
- Time-series friendly

### **Graph**

- Multi-hop traversal
- Relationship-centric
- Specialized use cases
- Higher operational complexity

## Three Main Schema Drivers

### 1. **Data Volume**

- Number of records
- Record size
- Growth rate
- Retention period
- Total storage
- Partition size
- Archival requirements

Questions:

- Fits on one machine?
- Needs partitioning?
- Needs sharding?
- Needs cold storage?
- Can old data expire?

### 2. **Access Patterns**

- Point reads
- Range reads
- Writes
- Updates
- Deletes
- Sorting
- Aggregation
- Joining
- Pagination

Questions:

- Which endpoints query this data?
- Which fields filter records?
- Which fields sort results?
- Which entities are read together?
- Which queries are most frequent?
- Which queries are latency-sensitive?

**Most important driver:** Schema and indexes should optimize dominant access patterns.

### 3. **Consistency Requirements**

- Strong consistency
- Eventual consistency
- Atomic updates
- Uniqueness
- Referential integrity
- Auditability

Strong-consistency examples:

- Payments
- Inventory reservation
- Seat booking
- Account balances
- Unique usernames

Eventual-consistency examples:

- Like counts
- Feeds
- Analytics
- Recommendations
- Search indexes

## Requirements-to-Schema Flow

```text
Functional requirements
        |
API endpoints
        |
Read and write queries
        |
Entities and relationships
        |
Database model
        |
Keys and constraints
        |
Indexes
        |
Partitioning or sharding
```

**Interview principle:** Every schema choice should trace back to a requirement or query.

## Entities

**Definition:** A distinct domain object requiring stored state.

Examples:

- User
- Post
- Comment
- Order
- Product
- Payment
- Event
- Reservation

Entity test:

- Does it have its own identity?
- Does it have an independent lifecycle?
- Is it queried independently?
- Is it referenced by other data?
- Does it require separate permissions?

## Primary Keys

### Purpose

- Uniquely identify record
- Support references
- Support joins
- Support updates and deletion
- Enable partitioning choices

### Preferred Keys

- `user_id`
- `post_id`
- `order_id`
- `payment_id`

### Avoid Mutable Business Keys

- Email
- Username
- Phone number
- Product name

**Core rule:** Use stable system-generated IDs for identity.

## ID Strategies

### **Auto-Incrementing Integer**

- Small
- Efficient index
- Naturally ordered
- Simple
- Central generation
- Guessable
- Possible distributed-write bottleneck

### **UUID**

- Decentralized generation
- Globally unique
- Good for distributed systems
- Larger indexes
- Random UUIDs can fragment B-trees
- Less human-friendly

### **Time-Ordered Distributed ID**

Examples:

- UUIDv7-style identifiers
- Snowflake-style IDs

Characteristics:

- Decentralized
- Roughly time ordered
- Better index locality
- Time dependence
- Generator coordination considerations

## Foreign Keys

### Purpose

- Express relationships
- Enforce valid references
- Prevent orphaned rows
- Improve data integrity

```text
posts.user_id -> users.user_id
comments.post_id -> posts.post_id
```

### Advantages

- Database-enforced integrity
- Clear schema semantics
- Safe deletion rules
- Fewer invalid records

### Costs

- Validation on writes
- Locking or coordination
- Bulk-loading complexity
- Difficult across shards
- Difficult across services

**Interview guidance:** Use foreign keys by default; remove only with a concrete scaling reason.

## Relationship Types

### **One-to-Many: 1:N**

Examples:

- User → posts
- Post → comments
- Order → order items

### **Many-to-Many: N:M**

Examples:

- Users ↔ posts through likes
- Users ↔ users through follows
- Products ↔ categories

```text
Likes
- user_id
- post_id
- created_at
- PK(user_id, post_id)
```

### **One-to-One: 1:1**

Possible reasons:

- Security isolation
- Different permissions
- Different lifecycle
- Large optional columns
- Separate service ownership

**Warning:** Without a clear reason, one-to-one tables may be mergeable.

## Constraints

### **NOT NULL**

- Required field
- Prevent missing values

### **UNIQUE**

- No duplicate value
- Usually backed by unique index

### **CHECK**

```sql
CHECK (price >= 0)
```

### **DEFAULT**

```sql
status DEFAULT 'pending'
```

### **Composite Uniqueness**

```sql
UNIQUE (user_id, post_id)
```

- One like per user per post
- One membership per user per group

**Best practice:** Validate in application; enforce critical invariants in database.

## Indexing for Access Patterns

**Main principle:** Index important queries, not every column.

### Social Media Examples

Fetch user by email:

```text
UNIQUE INDEX users(email)
```

Fetch recent posts by user:

```text
INDEX posts(user_id, created_at DESC)
```

Fetch comments for post:

```text
INDEX comments(post_id, created_at)
```

Fetch likes by post:

```text
INDEX likes(post_id)
```

Check whether user liked post:

```text
PRIMARY KEY likes(user_id, post_id)
```

## API-to-Index Mapping

Endpoint:

```http
GET /users/{user_id}/posts
```

Query:

```sql
SELECT *
FROM posts
WHERE user_id = ?
ORDER BY created_at DESC
LIMIT 20;
```

Index:

```text
(user_id, created_at DESC)
```

**Interview phrase:** The endpoint filters by user and sorts by creation time, so add a composite index on `(user_id, created_at)`.

## Normalization

**Definition:** Store each fact in one authoritative location.

```text
Users
- user_id
- username
- email

Posts
- post_id
- user_id
- content
```

### Benefits

- Less duplication
- Easier updates
- Stronger consistency
- Smaller storage
- Clear ownership
- Fewer anomalies

## Data Anomalies

### **Update Anomaly**

- Same value stored in many rows
- Some copies updated
- Other copies stale

### **Insert Anomaly**

- Cannot add one fact without unrelated data

### **Delete Anomaly**

- Deleting one record accidentally removes another fact

**Normalization goal:** Reduce duplication-driven anomalies.

## Denormalization

**Definition:** Intentionally duplicate or precompute data for faster access.

```text
Posts
- post_id
- user_id
- username_snapshot
- content
- like_count
- created_at
```

### Benefits

- Fewer joins
- Faster reads
- Simpler query path
- Precomputed values
- Better read scalability

### Costs

- Duplicate storage
- Complex updates
- Stale copies
- Eventual consistency
- Reconciliation logic
- Backfill complexity

### Good Candidates

- Read-heavy data
- Expensive joins
- Expensive aggregations
- Stable query pattern
- Staleness acceptable
- High fan-out reads
- Search documents
- Analytics tables

Examples:

- Like count on post
- Author name in feed item
- Product details in order snapshot
- Precomputed news feed
- Search-index document
- Daily analytics aggregate

### Avoid For

- Critical balance
- Inventory source of truth
- Frequently changed shared value
- Strictly consistent field
- Unproven performance issue
- Rare query

**Interview rule:** Normalize first; denormalize a measured read bottleneck.

## Snapshot vs Duplicate

### **Snapshot**

- Historical value captured intentionally
- Should not change with source

```text
OrderItem
- product_id
- product_name_at_purchase
- unit_price_at_purchase
```

### **Duplicate**

- Intended to mirror source
- Must stay synchronized
- Can become stale

**Critical difference:** Snapshot preserves history; duplicate mirrors current state.

## Cache as Denormalized Read Model

- Normalized database as source of truth
- Flattened object in cache
- Precomputed joins
- Fast query-specific read

```json
{
  "post_id": "post-123",
  "content": "Hello",
  "author": {
    "user_id": "user-10",
    "username": "john"
  },
  "like_count": 500
}
```

### Benefits

- Clean authoritative schema
- Fast user-facing reads
- Reduced database joins
- TTL-based refresh possible

### Costs

- Cache invalidation
- Stale values
- Rebuilding logic
- Additional infrastructure

## Source of Truth and Derived Data

### Source of Truth

**Authoritative location for a fact.**

Examples:

- User email → Users table
- Account balance → Ledger or Accounts table
- Inventory quantity → Inventory table

### Derived Data

Examples:

- Cache
- Search index
- Materialized view
- Analytics warehouse
- Recommendation model
- Denormalized feed

Properties:

- Rebuildable
- May be eventually consistent
- Generated from source of truth
- Should not silently become authoritative

**Design rule:** Every duplicated field should have one clear owner.

## Modeling Consistency Boundaries

### Strongly Consistent Data

Keep together when possible:

- Order and order items
- Payment state
- Inventory reservation
- Seat ownership
- Account balance and ledger entry

### Eventually Consistent Data

Can be separated:

- Search index
- Feed
- Like count
- Analytics
- Recommendation data
- Notification state

**Core principle:** Transaction boundaries should influence schema and shard boundaries.

## Auditability

Common fields:

```text
created_at
updated_at
created_by
updated_by
version
deleted_at
```

Audit log:

```text
AuditEvents
- event_id
- entity_type
- entity_id
- action
- actor_id
- previous_value
- new_value
- created_at
```

Use cases:

- Financial operations
- Administrative changes
- Compliance
- Security investigations
- Debugging

## Soft Delete

```text
deleted_at = timestamp
```

### Advantages

- Recovery
- Auditability
- Historical references
- Safer deletion workflow

### Costs

- Larger tables
- Queries must exclude deleted rows
- Unique constraints become harder
- Periodic cleanup needed

```sql
WHERE deleted_at IS NULL
```

## Versioning and Optimistic Locking

```text
version = 7
```

```sql
UPDATE posts
SET content = ?, version = version + 1
WHERE post_id = ?
  AND version = 7;
```

Result:

- One writer succeeds
- Conflicting writer detects mismatch
- Prevents silent lost update

## Pagination-Aware Modeling

### Offset Pagination

```sql
LIMIT 20 OFFSET 100000;
```

Problems:

- Large scan
- Slower deep pages
- Unstable under inserts

### Cursor Pagination

```sql
WHERE created_at < ?
ORDER BY created_at DESC
LIMIT 20;
```

Supporting index:

```text
(user_id, created_at DESC, post_id)
```

**Interview rule:** Design indexes and cursors together.

## Scaling a Relational Model

1. Optimize queries
2. Add indexes
3. Use connection pooling
4. Add caching
5. Add read replicas
6. Partition large tables
7. Archive old data
8. Shard only when necessary

**Avoid:** Jumping directly to sharding.

## Partitioning

**Definition:** Split a table into logical pieces within a database system.

Common strategies:

- Time range
- ID range
- Hash
- List or tenant

Good fit:

- Large tables
- Time-based retention
- Partition pruning
- Easier archival
- Smaller maintenance units

## Sharding

**Definition:** Split data across independent database machines.

Main goals:

- Scale storage
- Scale write throughput
- Scale read throughput
- Reduce per-node data

**Main decision:** Choose shard key from the dominant access pattern.

## Good Shard Key

- High cardinality
- Even data distribution
- Even traffic distribution
- Matches common queries
- Keeps related data together
- Supports bounded partitions
- Avoids cross-shard transactions

Example:

```text
Shard posts by user_id
```

Benefits:

- User posts colocated
- User-history query hits one shard
- User-level writes localized

## Shard-Key Trade-Offs

### Shard by `user_id`

Good for:

- Posts by user
- Profile data
- User settings
- User-owned content

Potential problems:

- Global feed query
- Celebrity hot user
- Cross-user aggregation

### Shard by `post_id`

Good for:

- Post lookup by ID
- Even distribution

Potential problems:

- Fetch all posts by user
- User data spread across shards

**Core rule:** Optimize the highest-value query, not every possible query.

## Time-Based Sharding

Benefits:

- Easy archival
- Efficient time-range queries
- Old data isolated
- Simple retention

Risks:

- All current writes hit latest shard
- Hot shard
- Uneven capacity usage
- Old shards mostly idle

Best fit:

- Event archives
- Historical analytics
- Retention management

## Cross-Shard Queries

Example: Fetch recent posts from 500 followed users.

Potential work:

- Identify relevant shards
- Query several shards
- Merge results
- Sort globally
- Return top page

Costs:

- Multiple network calls
- Tail-latency amplification
- Partial failures
- Complex sorting
- Higher fan-out

Solutions:

- Fan-out on write
- Precomputed feed
- Cache
- Background aggregation
- Materialized read model
- Limit shard fan-out

**Interview warning:** Frequent scatter-gather suggests a poor shard key or missing read model.

## Cross-Shard Transactions

Problems:

- Related writes on separate shards
- No local ACID transaction
- Partial failure possible
- Distributed coordination required

**Preferred solution:** Keep transactions on one shard.

Alternatives:

- Saga
- Outbox pattern
- Idempotent events
- Compensation
- Two-phase commit for rare strict cases

## Hot Partitions

Causes:

- Celebrity user
- Large enterprise tenant
- Time-based key
- Low-cardinality shard key
- Popular event
- Sequential key range

Mitigations:

- Compound shard key
- Key salting
- Dedicated shard
- Tenant isolation
- Replication
- Caching
- Dynamic repartitioning

## Data Modeling by API

### Create Post

```http
POST /users/{user_id}/posts
```

Schema impact:

```text
Posts(user_id, post_id, content, created_at)
```

### Get Recent User Posts

```http
GET /users/{user_id}/posts?cursor=...
```

Index:

```text
(user_id, created_at DESC, post_id)
```

### Like Post

```http
POST /posts/{post_id}/likes
```

Schema:

```text
Likes(user_id, post_id, created_at)
UNIQUE(user_id, post_id)
```

Derived data:

```text
Posts.like_count
```

- Eventually consistent counter
- Updated asynchronously

### Get Post Comments

```http
GET /posts/{post_id}/comments
```

Index:

```text
(post_id, created_at, comment_id)
```

## Example E-Commerce Schema

```text
Users
- user_id              PK
- email                UNIQUE
- created_at

Products
- product_id           PK
- name
- current_price
- stock_quantity
- updated_at

Orders
- order_id             PK
- user_id              FK
- status
- total_amount
- created_at
- INDEX(user_id, created_at)

OrderItems
- order_id             FK
- product_id           FK
- product_name_snapshot
- unit_price_snapshot
- quantity
- PK(order_id, product_id)

Payments
- payment_id           PK
- order_id             FK
- idempotency_key      UNIQUE
- status
- amount
- created_at
```

Key decisions:

- Product price stored in Products
- Purchase price snapshot stored in OrderItems
- Payment idempotency key prevents duplicate charge
- User-order index supports order history
- Order and items share transaction boundary

## Example Ticketing Schema

```text
Events
- event_id             PK
- venue_id
- name
- starts_at

Seats
- event_id             PK-part
- seat_id              PK-part
- section
- row
- number
- status
- version

Reservations
- reservation_id       PK
- event_id
- seat_id
- user_id
- status
- expires_at
- UNIQUE(event_id, seat_id)
```

Critical requirements:

- No double-booking
- Seat uniqueness
- Reservation expiration
- Strong consistency
- Conditional update or locking

## Example Feed Modeling

### Normalized Source

```text
Posts
- post_id
- author_id
- content
- created_at

Follows
- follower_id
- followee_id
```

### Direct Read Approach

- Fetch followed users
- Fetch their posts
- Merge and sort
- Expensive at scale

### Denormalized Feed

```text
UserFeed
- user_id
- post_id
- author_id
- created_at
- ranking_score
```

Benefits:

- Fast feed reads
- Cursor pagination
- User-specific ordering

Costs:

- Fan-out writes
- Duplicate feed entries
- Eventual consistency
- Celebrity fan-out problem

## Common Data Modeling Mistakes

### **Choosing Database Before Requirements**

- Technology-first design
- Poor requirement alignment
- Unnecessary specialization

### **Using NoSQL to Sound Scalable**

- Weak query justification
- Lost transaction guarantees
- Extra application complexity

### **Embedding Unbounded Collections**

- Huge documents
- Update contention
- Size limits
- Difficult pagination

### **Natural Key as Primary Key**

- Mutable identity
- Large foreign keys
- Business coupling

### **Indexing Every Field**

- Write amplification
- Extra storage
- Maintenance overhead

### **Ignoring Composite Index Order**

- Index cannot support query
- Sorting remains expensive

### **Premature Denormalization**

- Consistency burden
- Duplicate updates
- Unclear source of truth

### **Premature Sharding**

- Complex routing
- Cross-shard queries
- Distributed transactions
- Resharding burden

### **Ignoring Deletion and Retention**

- Infinite growth
- Compliance risks
- Large indexes
- Expensive backups

### **Ignoring Cardinality**

- Hot partitions
- Poor index usefulness
- Uneven distribution

## Data Modeling Interview Framework

1. **Identify core entities**
2. **Select database model**
3. **Define key fields**
4. **Define relationships**
5. **Add constraints**
6. **Map APIs to queries**
7. **Add indexes**
8. **Decide normalization level**
9. **Evaluate scale**
10. **Consider partitioning or sharding**
11. **State trade-offs**

## Interview Presentation Template

> I’ll start with PostgreSQL because the entities have clear relationships and the critical operations benefit from transactions and constraints. The core tables are Users, Posts, Comments, and Likes. Each table uses a stable system-generated primary key, while foreign keys represent ownership. The recent-posts endpoint filters by `user_id` and orders by `created_at`, so Posts gets a composite index on `(user_id, created_at)`. I’ll begin with a normalized source-of-truth schema. If feed generation becomes a read bottleneck, I’ll introduce a denormalized feed table or cache with eventual consistency. I would shard only after capacity planning proves one database is insufficient, likely by `user_id` to keep user-owned data together.

## Database Choice Questions

### Choose Relational If

- Clear entities?
- Strong transactions?
- Constraints important?
- Many query patterns?
- Joins manageable?
- Schema mostly stable?

### Choose Document If

- Records have different shapes?
- Nested aggregate read as one unit?
- Schema changes frequently?
- Cross-document joins uncommon?

### Choose Key-Value If

- Exact key lookup only?
- Very low latency?
- Cache or session data?
- Flat data acceptable?

### Choose Wide-Column If

- Massive append volume?
- Predictable partition queries?
- Time-series workload?
- Data duplicated per access pattern?

### Choose Graph If

- Deep variable-length traversal?
- Relationships are primary data?
- SQL joins truly insufficient?
- Specialized operational cost justified?

## Important Terms

- **Data Model:** Definition of stored entities, attributes, and relationships
- **Schema:** Formal structure of database data
- **Entity:** Domain object with independent identity
- **Attribute:** Stored property of an entity
- **Primary Key:** Unique stable record identifier
- **Natural Key:** Business value used as identity
- **Surrogate Key:** System-generated identifier
- **Foreign Key:** Reference to another table’s primary key
- **Referential Integrity:** Guarantee that references point to valid records
- **Constraint:** Database-enforced correctness rule
- **Cardinality:** Number of values or relationship multiplicity
- **One-to-Many:** One parent connected to many children
- **Many-to-Many:** Multiple records connected through junction table
- **Junction Table:** Table representing a many-to-many relationship
- **Normalization:** Storing each fact in one authoritative place
- **Denormalization:** Intentional duplication for faster access
- **Source of Truth:** Authoritative location for a fact
- **Derived Data:** Rebuildable data created from authoritative data
- **Snapshot:** Historical value preserved at a point in time
- **Access Pattern:** Way an application reads or writes data
- **Selectivity:** Fraction of records matched by a condition
- **Composite Index:** Ordered index over several columns
- **Partition Key:** Value deciding logical or physical data placement
- **Clustering Key:** Value controlling order within a partition
- **Shard Key:** Value deciding database-shard placement
- **Hot Partition:** Partition receiving disproportionate load
- **Scatter-Gather:** Querying many shards and combining results
- **Soft Delete:** Marking data deleted without physical removal
- **Optimistic Locking:** Version-based concurrent-update detection
- **Cursor Pagination:** Pagination based on a stable ordered position
- **Audit Log:** Immutable record of important changes
- **Read Model:** Query-optimized representation of data
- **Materialized View:** Precomputed persisted query result
- **Schema Evolution:** Controlled change to stored data structure

## 30-Second Revision Summary

- **Data modeling?** Entities, fields, relationships, and storage
- **Interview depth?** Clear and functional, not exhaustive
- **Default database?** PostgreSQL
- **Main drivers?** Volume, access patterns, consistency
- **Most important driver?** Access patterns
- **Primary key?** Stable system-generated ID
- **Relationships?** Foreign keys or explicit application references
- **Many-to-many?** Junction table
- **Correctness?** Constraints
- **Index selection?** Match filters and sorting
- **Starting model?** Normalized source of truth
- **Denormalize when?** Proven read bottleneck
- **Document embedding?** Only bounded child data
- **Key-value role?** Fast exact lookup or cache
- **Wide-column role?** Massive query-oriented writes
- **Graph role?** Deep relationship traversal
- **Shard when?** Single database no longer sufficient
- **Shard key?** Match primary access pattern
- **Avoid?** Cross-shard queries and transactions
- **Best interview principle?** Every schema choice should support a requirement
