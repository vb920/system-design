# Database Indexes: Quick Revision Notes

## Core Thesis

**Indexes trade storage and write performance for faster reads.**

## Key Insights

- **Indexes reduce scanned disk pages**
- **Index selection follows query patterns**
- **B-tree is the general-purpose default**
- **LSM tree favors write-heavy workloads**
- **Specialized data needs specialized indexes**
- **Composite-index column order matters**
- **Unused indexes create unnecessary write cost**

## Why Indexes Are Needed

### Without an Index

- Table stored as unordered rows
- Full-table scan
- Every page examined
- High disk I/O
- Query time grows with table size
- Approximate complexity: **O(N)**

#### Example

```sql
SELECT *
FROM users
WHERE email = 'alice@example.com';
```

Without email index:

- Scan every user
- Compare every email
- Millions of rows possible
- High latency

### With an Index

- Separate search structure
- Indexed values organized
- Direct path to matching rows
- Fewer disk pages loaded
- Faster lookup
- Extra table lookup sometimes required

#### Analogy

- Full scan: read every book
- Index: use library catalogue

## Physical Storage Basics

### **Heap File**

- Main table storage
- Rows in no guaranteed order
- New rows placed in available pages
- Efficient simple insertion
- Poor direct lookup without index

### **Disk Page**

- Basic database I/O unit
- Multiple rows per page
- Entire page loaded into memory
- Common size: approximately **8 KB**
- Exact size depends on database

### Query Cost

**Database performance often depends on pages read, not only rows examined.**

## Access Patterns

### **Sequential Scan**

- Read pages in order
- Efficient for large result sets
- Useful for small tables
- Useful when most rows match
- Avoids index traversal overhead

### **Random Access**

- Jump between unrelated pages
- Higher I/O cost
- Required for scattered table rows
- Still slower than sequential access on SSDs

### **Index Scan**

- Traverse index pages
- Find matching row pointers
- Fetch target table pages
- Efficient for selective queries

### **Index-Only Scan**

- Required columns available in index
- No main-table lookup
- Fewer disk reads
- Enabled by covering index
- Database visibility checks may still apply

## Index Costs

### **Storage Cost**

- Additional disk usage
- Key values stored again
- Row pointers stored
- Large indexes may approach table size
- Covering indexes consume more space

### **Write Cost**

Every insert, update, or delete may require:

- Main-table modification
- Index-entry insertion
- Index-entry deletion
- Page split
- WAL entry
- Additional disk writes

### **Memory Cost**

- Hot index pages use buffer pool
- More indexes compete for memory
- Cold pages eventually evicted
- Modern buffer managers reduce impact

### **Operational Cost**

- Index creation time
- Index rebuilding
- Statistics maintenance
- Fragmentation or bloat
- Backup size
- Replication traffic

## When Indexes May Hurt

### **Write-Heavy Tables**

- Frequent inserts
- Few queries
- Index maintenance dominates
- Example: append-only logs

### **Small Tables**

- Few hundred rows
- Full scan already cheap
- Index traversal may cost more
- Optimizer may ignore index

### **Low-Selectivity Columns**

- Few distinct values
- Query matches large table fraction
- Many table pages still fetched

Examples:

- Boolean flags
- Gender category
- Common status value

### **Unused Indexes**

- No read benefit
- Continued write overhead
- Additional storage
- Additional maintenance

## Selectivity

### Definition

**How narrowly an index condition filters rows.**

### High Selectivity

- Few rows match
- Index highly useful
- Example: unique email

### Low Selectivity

- Many rows match
- Full scan may be cheaper
- Example: `is_active = true` when 95% are active

### Interview Rule

**Index columns frequently used for selective filtering, joining, or sorting.**

## Major Index Types

1. **B-tree**
2. **LSM tree**
3. **Hash index**
4. **Geospatial index**
5. **Inverted index**
6. **Composite index**
7. **Covering index**

## B-Tree Index

### Definition

- Balanced multiway search tree
- Keys maintained in sorted order
- Multiple keys per node
- Multiple children per node
- Nodes sized around disk pages

### Core Properties

- All leaves at similar depth
- Predictable lookup depth
- Automatic balancing
- Efficient inserts and deletes
- Ordered traversal
- Supports equality and ranges

### Typical Complexity

- Search: **O(log N)**
- Insert: **O(log N)**
- Delete: **O(log N)**

## Why B-Trees Fit Databases

### High Fan-Out

- Hundreds of keys per node
- Few tree levels
- Very large dataset with shallow tree
- Usually only a few page reads

### Page-Aligned Nodes

- One node fits one disk page
- One I/O retrieves many keys
- Reduced tree traversal cost

### Example Lookup

```sql
SELECT *
FROM users
WHERE id = 350;
```

Possible reads:

1. Root page
2. Internal page
3. Leaf page
4. Table page if needed

## B-Tree Query Support

### **Equality Query**

```sql
WHERE email = 'alice@example.com'
```

- Direct key navigation
- Fast lookup

### **Range Query**

```sql
WHERE age BETWEEN 25 AND 35
```

- Locate range start
- Scan ordered leaf entries
- Efficient sequential traversal

### **Sorting**

```sql
ORDER BY created_at
```

- Index order already sorted
- Explicit sort may be avoided

### **Prefix Search**

```sql
WHERE name LIKE 'data%'
```

- Prefix defines ordered range
- B-tree may be used

### Unsupported Substring Pattern

```sql
WHERE content LIKE '%database%'
```

- Unknown starting prefix
- Normal B-tree usually unusable
- Inverted or specialized text index needed

## Why B-Tree Is the Default

- Exact-match support
- Range-query support
- Sorting support
- Prefix-query support
- Predictable performance
- Balanced under updates
- Disk-friendly layout
- Broad database support

### Interview Default

**When uncertain, propose a B-tree and justify it from the query pattern.**

## B-Tree Examples

### PostgreSQL

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE
);
```

Automatically created:

- B-tree on `id`
- B-tree on `email`

### MongoDB

```javascript
db.users.createIndex({ email: 1 });
```

- B-tree-family index
- Document value → document location

## LSM Tree

### Full Name

**Log-Structured Merge Tree**

### Main Goal

**Convert random writes into sequential writes.**

### Best Fit

- Write-heavy systems
- High ingestion rate
- Time-series data
- Metrics
- Logs
- Event streams
- IoT telemetry

## B-Tree Write Path

1. Find target leaf page
2. Load page
3. Modify page
4. Write page
5. Split page if full
6. Update parent nodes if required

### Main Cost

- Random page updates
- Write amplification
- Contention on hot pages

## LSM Write Path

1. Append write to WAL
2. Insert into memtable
3. Acknowledge write
4. Freeze full memtable
5. Flush as SSTable
6. Compact SSTables later

## LSM Components

### **Write-Ahead Log**

- Sequential disk append
- Durability before acknowledgement
- Recovery after crash
- Rebuilds lost memtable state

### **Memtable**

- In-memory sorted structure
- Fast inserts
- Often skip list or balanced tree
- Contains recent writes

### **Immutable Memtable**

- Frozen full memtable
- Waiting for disk flush
- Still available for reads

### **SSTable**

**Sorted String Table**

- Immutable disk file
- Keys stored in sorted order
- Created by sequential flush
- Efficient range scanning
- Never updated in place

### **Compaction**

- Merge SSTables
- Remove obsolete versions
- Remove deletion markers
- Reduce file count
- Improve read performance
- Consume CPU and disk bandwidth

## Why LSM Writes Are Fast

- Memory-first writes
- Sequential WAL append
- Batched disk flush
- No immediate in-place update
- Fewer random disk writes
- High sustained throughput

## LSM Read Path

For a point lookup:

1. Check active memtable
2. Check immutable memtables
3. Check newest SSTables
4. Continue through older SSTables
5. Select latest value

### Main Read Cost

- Data may exist in multiple places
- Multiple files may be checked
- Higher read amplification
- Compaction state affects latency

## LSM Read Optimizations

### **Bloom Filter**

- Probabilistic membership test
- Answers:
  - Definitely absent
  - Possibly present
- Skips unnecessary SSTable reads
- False positives possible
- False negatives not expected

**Memory trick:** No means no; maybe means check.

### **Sparse Index**

- Stores selected keys and offsets
- Narrows search to relevant block
- Exploits SSTable ordering
- Smaller than full in-memory index

### **Key-Range Metadata**

- Minimum key
- Maximum key
- Skip files outside target range

### **Block Cache**

- Frequently read blocks in memory
- Reduces repeated disk access
- Helps hot keys

### **Compaction**

- Fewer overlapping files
- Fewer reads per query
- Background rewrite overhead

## Compaction Strategies

### **Size-Tiered Compaction**

- Merge similarly sized SSTables
- Lower write amplification
- More overlapping files
- Higher read amplification
- More temporary disk use

### **Leveled Compaction**

- SSTables organized into levels
- Limited overlap per level
- Better read performance
- More rewriting
- Higher write amplification

### Core Trade-Off

- Fewer rewrites → more files to read
- More rewrites → faster reads

## B-Tree vs LSM Tree

### **B-Tree**

- Updates data in place
- Balanced read/write performance
- Fast point reads
- Strong range-query support
- Lower read amplification
- Random write cost
- Common in relational databases

### **LSM Tree**

- Append and merge model
- Excellent write throughput
- Sequential disk writes
- Higher read amplification
- Compaction overhead
- Bloom filters important
- Common in write-heavy distributed stores

### Selection Rule

- Read-heavy or balanced workload → B-tree
- Write-heavy ingestion workload → LSM tree

## Write Amplification

### Definition

**Physical bytes written compared with logical bytes written.**

### B-Tree Sources

- Page updates
- Page splits
- WAL
- Multiple secondary indexes

### LSM Sources

- WAL
- Initial flush
- Repeated compaction
- Data rewritten across levels

### Important Nuance

**LSM trees reduce random writes, but may still rewrite data many times.**

## Read Amplification

### Definition

**Extra reads required to locate one logical result.**

### B-Tree

- Tree traversal
- Optional table-page lookup
- Usually low and predictable

### LSM Tree

- Memtables
- Multiple SSTables
- Bloom-filter checks
- Version reconciliation
- Potentially higher

## Space Amplification

### Definition

**Extra storage beyond the current logical dataset.**

### Causes

- Old record versions
- Tombstones
- Duplicate values during compaction
- Secondary indexes
- Temporary merge files

## LSM Tree Examples

### **Cassandra**

- Write-heavy distributed storage
- Memtables
- SSTables
- Compaction
- Bloom filters

### **RocksDB**

- Embedded storage engine
- LSM architecture
- Configurable compaction
- High write throughput

### **DynamoDB**

- Write-optimized internal architecture
- Exact implementation not fully public
- Avoid claiming automatic engine switching
- Partition and access-pattern design still essential

## Hash Index

### Definition

- Persistent hash table
- Indexed key → bucket
- Bucket → row pointer
- Optimized for equality lookup

### Lookup Flow

1. Hash indexed value
2. Locate bucket
3. Search bucket entries
4. Follow row pointer

### Average Lookup

**O(1)** under good distribution assumptions

## Hash Collisions

### Problem

- Different keys produce same bucket
- Bucket contains multiple entries

### Handling

- Chaining
- Overflow pages
- Open addressing
- Bucket splitting

## Hash Index Strengths

- Very fast exact matches
- Simple key lookup
- Good in-memory behavior
- Useful for key-value access

### Suitable Query

```sql
WHERE email = 'alice@example.com'
```

## Hash Index Limitations

- No range queries
- No sorted traversal
- No `ORDER BY` support
- No prefix matching
- Collision handling required
- May consume significant memory

### Unsupported Queries

```sql
WHERE age > 25
```

```sql
ORDER BY email
```

```sql
WHERE name LIKE 'data%'
```

## Hash Index vs B-Tree

### **Hash Index**

- Equality only
- Average O(1)
- No ordering
- Specialized usage

### **B-Tree**

- Equality
- Ranges
- Prefixes
- Sorting
- Nearly as effective for equality
- Better general-purpose choice

### Interview Guidance

**Mention hash indexes only for strict exact-match workloads with no range or sorting need.**

## Geospatial Indexes

### Problem

- Latitude and longitude form 2D coordinates
- Nearby search requires spatial relationship
- Separate scalar indexes lose locality
- Circular radius ≠ simple 1D range

### Example Query

**Find restaurants within five miles.**

## Why Separate Latitude and Longitude Indexes Fail

### Latitude Index First

- Finds horizontal geographic band
- Band may span entire world
- Longitude filtering still required
- Many false candidates

### Longitude Index First

- Finds vertical geographic band
- Large candidate set
- Latitude filtering still required

### Index Intersection

- Two large result sets
- Expensive merge
- Produces rectangular candidate region
- Exact-distance filtering still needed

## Composite `(latitude, longitude)` B-Tree

### Better Than Separate Indexes

- Latitude and longitude in one structure
- Efficient latitude-range scan
- Longitude checked within candidates

### Still Weak for Proximity Search

- Sorted by latitude first
- Longitude useful only within latitude grouping
- 2D locality reduced to lexicographic order
- Nearby points may occupy distant index positions
- Large band scan possible
- Radius remains circular, not rectangular

### Core Issue

**Lexicographic ordering does not preserve 2D spatial locality well.**

## Main Geospatial Approaches

1. **Geohash**
2. **Quadtree**
3. **R-tree**

## Geohash

### Core Idea

**Convert 2D coordinates into a locality-preserving 1D string.**

### Process

1. Divide world into regions
2. Choose region containing point
3. Subdivide selected region
4. Repeat recursively
5. Encode choices as string

### Precision

- Short prefix → large area
- Long prefix → small area
- More characters → greater precision

### Example

```text
9q8y
9q8yy
9q8yyk
```

Possible interpretation:

- `9q8y` → city area
- `9q8yy` → neighborhood
- `9q8yyk` → city block

## Geohash Advantage

- Nearby points often share prefixes
- Prefixes stored in normal B-tree
- Existing database indexing reused
- Simple implementation
- Good for approximate proximity search

```sql
CREATE INDEX idx_geohash
ON restaurants(geohash);
```

## Geohash Radius Query

### Example Goal

**Find points within five miles.**

### Process

1. Geohash center point
2. Select suitable precision
3. Identify center cell
4. Identify adjacent cells
5. Run prefix queries
6. Collect candidates
7. Calculate exact distance
8. Remove false positives

### Typical Cell Coverage

- Center cell
- Eight neighboring cells
- Approximately 9 prefix searches
- Exact count depends on radius and precision

### Why Neighbors Are Required

- User may be near cell edge
- Nearby point may have different prefix
- Prefix equality alone may miss valid results

### Final Filter

Use actual geographic distance:

- Haversine formula
- Database spatial function
- Native geospatial operator

### Important Principle

**Geohash generates candidates; exact distance determines final results.**

## Geohash Boundary Problem

### Problem

- Physically close points
- Different grid cells
- Different prefixes
- Especially near major boundaries

### Mitigation

- Query adjacent cells
- Use multiple precision levels
- Apply exact-distance filtering

### Trade-Off

- Simple and scalable
- Approximate spatial locality
- Boundary handling required

## Quadtree

### Definition

- Recursive 2D space partition
- Each region split into four quadrants
- Dense regions subdivided further
- Sparse regions remain large

### Construction

1. Begin with one large square
2. Insert points
3. Split after threshold
4. Repeat recursively
5. Stop at capacity or depth limit

### Advantages

- Adaptive spatial resolution
- Fine partitioning in dense areas
- Coarse partitioning in sparse areas
- Natural map-tile hierarchy

### Disadvantages

- Specialized implementation
- Fixed rectangular subdivisions
- Boundary traversal
- Large objects may span several cells
- Less convenient than B-tree geohash storage

### Suitable For

- 2D points
- Map tiles
- Game worlds
- Spatial partitioning
- Density-aware regions

## R-Tree

### Definition

- Hierarchical spatial index
- Uses minimum bounding rectangles
- Rectangles may overlap
- Adapts to actual object distribution

### Supported Geometry

- Points
- Lines
- Rectangles
- Polygons
- Delivery zones
- Road networks

### Search Flow

1. Start at root bounding rectangle
2. Select intersecting child rectangles
3. Traverse matching branches
4. Reach candidate objects
5. Apply exact geometric test

## R-Tree Advantages

- Flexible object shapes
- Adaptive bounding regions
- Good spatial range queries
- Good intersection queries
- Production database support
- Disk-oriented design

### R-Tree Limitation

- Bounding rectangles overlap
- Query may traverse multiple branches
- Performance depends on overlap
- Insertion strategy affects tree quality

### Common Usage

- PostgreSQL with PostGIS
- MySQL spatial indexes
- Geographic information systems
- Mapping platforms

## Geohash vs Quadtree vs R-Tree

### **Geohash**

- 2D → 1D string
- Prefix-based lookup
- Reuses B-tree
- Simple
- Boundary false candidates
- Good interview default

### **Quadtree**

- Recursive fixed quadrants
- Adaptive depth
- Best for point density and map tiles
- Specialized tree required

### **R-Tree**

- Flexible bounding rectangles
- Handles points and shapes
- Rectangles may overlap
- Production spatial-index standard

### Interview Rule

- Explain why B-tree is insufficient
- Present geohash clearly
- Mention adjacent cells and exact filtering
- Contrast with R-tree for richer geometry

## Geospatial Interview Answer

> A normal B-tree treats latitude and longitude as independent scalar values, so it does not preserve two-dimensional proximity. I would use a geospatial index. A geohash converts coordinates into a one-dimensional prefix, allowing nearby points to be retrieved using B-tree prefix ranges. I would query the center and neighboring cells, then apply exact-distance filtering. For complex shapes or intersection queries, I would use an R-tree-based spatial index.

## Inverted Index

### Purpose

**Efficient full-text search.**

### Forward Representation

```text
document → words
```

### Inverted Representation

```text
word → documents
```

## Example

Documents:

```text
doc1: B-trees are fast and reliable
doc2: Hash tables are fast but limited
doc3: B-trees handle range queries well
```

Inverted index:

```text
b-trees  → [doc1, doc3]
fast     → [doc1, doc2]
reliable → [doc1]
hash     → [doc2]
range    → [doc3]
queries  → [doc3]
```

## Why B-Tree Fails for Full Text

### Query

```sql
SELECT *
FROM posts
WHERE content LIKE '%database%';
```

### Problem

- Leading wildcard
- Unknown starting position
- Every text value examined
- High CPU and I/O
- Index cannot seek to substring

### B-Tree Can Help With

```sql
WHERE content LIKE 'database%'
```

- Known prefix
- Ordered range possible

### B-Tree Usually Cannot Help With

```sql
WHERE content LIKE '%database%'
```

- Arbitrary substring
- Full-text index needed

## Text Analysis Pipeline

### **Tokenization**

- Split text into terms
- Words or subwords

### **Normalization**

- Convert case
- Normalize characters
- Standardize forms

### **Stop-Word Removal**

- Remove common low-value terms
- Examples: “the”, “and”, “of”

### **Stemming**

- Reduce word to root-like form
- Query variations match related forms

### **Lemmatization**

- Convert word to dictionary form
- More linguistically accurate than stemming

## Inverted-Index Features

### **Term Frequency**

- Count occurrences per document
- Helps relevance scoring

### **Document Frequency**

- Count documents containing term
- Rare terms often more informative

### **Relevance Scoring**

- Rank documents
- Example: TF-IDF or BM25

### **Phrase Search**

- Store term positions
- Match ordered sequences

### **Fuzzy Search**

- Match similar spellings
- Handle typographical errors

### **Boolean Search**

- AND
- OR
- NOT

## Inverted-Index Costs

- Significant storage overhead
- Token processing
- Complex updates
- Many postings per document
- Eventual indexing delay possible
- Separate search infrastructure often required

### Common Systems

- Elasticsearch
- Apache Lucene
- OpenSearch
- Database-native full-text indexes

## Composite Index

### Definition

**One index containing multiple ordered columns.**

### Example Query

```sql
SELECT *
FROM posts
WHERE user_id = 123
  AND created_at > '2024-01-01'
ORDER BY created_at DESC;
```

### Composite Index

```sql
CREATE INDEX idx_user_time
ON posts(user_id, created_at);
```

## Composite Key Ordering

Conceptual entries:

```text
(1, 2024-01-01)
(1, 2024-01-02)
(1, 2024-01-03)
(2, 2024-01-01)
(2, 2024-01-02)
```

### Ordering Rule

- Sort by `user_id` first
- Within each user, sort by `created_at`
- Filter and ordering handled together

### Benefit

- One index traversal
- No index intersection
- Smaller candidate range
- Sort may be avoided
- Sequential leaf scan

## Leftmost-Prefix Rule

For index:

```text
(user_id, created_at)
```

Efficient for:

```sql
WHERE user_id = 123
```

```sql
WHERE user_id = 123
  AND created_at > '2024-01-01'
```

Usually not efficient for:

```sql
WHERE created_at > '2024-01-01'
```

### Reason

- Index globally ordered by `user_id`
- Dates separated across user groups
- Cannot directly isolate all dates

### Memory Trick

**A composite B-tree is searchable from the left.**

## Composite Index Column Order

### Common Guideline

1. Equality columns
2. Range columns
3. Sorting columns
4. Included output columns

### Example

```text
(user_id, status, created_at)
```

Supports:

- Equality on user
- Equality on status
- Range or ordering on time

### Selectivity Guidance

- High-selectivity columns often useful early
- Query pattern more important than generic rule
- Equality before range usually critical
- Sorting requirement may influence order

### Interview Warning

**“Most selective first” is not a universal rule.**

## Composite Index Examples

### Order History

```text
(customer_id, order_date)
```

### Activity Feed

```text
(user_id, timestamp)
```

### Event Processing

```text
(status, priority, created_at)
```

### Messaging

```text
(conversation_id, created_at)
```

### Multi-Tenant Data

```text
(tenant_id, resource_id)
```

## Separate vs Composite Indexes

### Separate Indexes

```text
INDEX(user_id)
INDEX(created_at)
```

Possible work:

- Scan user index
- Scan time index
- Intersect results
- Sort output

### Composite Index

```text
INDEX(user_id, created_at)
```

Possible work:

- Seek directly to user
- Scan matching timestamps
- Return in index order

### Selection Rule

**Use a composite index when columns frequently appear together in the same query pattern.**

## Covering Index

### Definition

**Index contains every column required by a query.**

### Example

```sql
CREATE INDEX idx_user_time_likes
ON posts(user_id, created_at)
INCLUDE (likes);
```

### Query

```sql
SELECT created_at, likes
FROM posts
WHERE user_id = 123
ORDER BY created_at DESC;
```

### Benefit

- Index-only scan
- No main-table page lookup
- Lower random I/O
- Faster read-heavy query

## Key Columns vs Included Columns

### **Key Columns**

```text
(user_id, created_at)
```

- Control index ordering
- Support search conditions
- Support range scans
- Support sorting

### **Included Columns**

```text
INCLUDE (likes)
```

- Stored in leaf entries
- Do not control ordering
- Used only to satisfy output
- Increase index size

## Covering-Index Trade-Offs

### Advantages

- Fewer table reads
- Lower latency
- Good for narrow projections
- Effective for frequent queries

### Costs

- Larger index
- More write overhead
- More memory pressure
- More maintenance
- Reduced schema flexibility

### Best Fit

- Read-heavy workload
- Stable query pattern
- Small output column set
- Large base rows
- Proven table-lookup bottleneck

### Interview Guidance

**Use only after identifying a specific high-value query.**

## Index Design Process

### Step 1: Identify Query Patterns

- Filter columns
- Join columns
- Sort columns
- Grouping columns
- Returned columns
- Query frequency

### Step 2: Measure Selectivity

- Number of matching rows
- Percentage of table
- Value distribution
- Skew and hot values

### Step 3: Choose Index Type

- General lookup → B-tree
- Write-heavy engine → LSM tree
- Exact match only → hash
- Location search → geospatial
- Full text → inverted

### Step 4: Choose Column Order

- Equality predicates first
- Range predicate afterward
- Align with sorting
- Follow leftmost-prefix rule

### Step 5: Evaluate Costs

- Insert latency
- Update latency
- Storage
- Memory
- Maintenance
- Replication overhead

### Step 6: Inspect Query Plan

```sql
EXPLAIN
SELECT ...
```

or:

```sql
EXPLAIN ANALYZE
SELECT ...
```

Look for:

- Sequential scan
- Index scan
- Index-only scan
- Rows estimated
- Rows actually returned
- Sort operation
- Join strategy
- Disk-page reads

### Step 7: Monitor Production Usage

- Index scan count
- Query latency
- Buffer hit rate
- Index size
- Write amplification
- Unused indexes

## When the Optimizer May Ignore an Index

- Table is small
- Query returns many rows
- Index selectivity is low
- Statistics are stale
- Function applied to indexed column
- Type conversion prevents matching
- Leading wildcard used
- Composite-index prefix missing
- Sequential scan estimated cheaper

## Sargability

### Definition

**Whether a predicate can use an index search efficiently.**

### Sargable

```sql
WHERE created_at >= '2026-01-01'
```

### Potentially Non-Sargable

```sql
WHERE YEAR(created_at) = 2026
```

### Better Rewrite

```sql
WHERE created_at >= '2026-01-01'
  AND created_at < '2027-01-01'
```

### Core Principle

**Avoid transforming the indexed column when a direct range predicate is possible.**

## Common Indexing Mistakes

### **Index Every Column**

- Excess storage
- Slow writes
- No query-driven justification

### **Ignore Column Order**

- Composite index unusable
- Sorting not supported
- Range placed too early

### **Use B-Tree for Full Text**

- Leading wildcard scan
- Poor text relevance
- No token understanding

### **Use Scalar Indexes for Spatial Search**

- Latitude bands
- Longitude bands
- Poor 2D locality
- Excess candidate filtering

### **Assume Index Always Helps**

- Low-selectivity query
- Small table
- Large result set
- Full scan may be cheaper

### **Ignore Query Plans**

- Index exists but unused
- Wrong cardinality estimate
- Hidden sort or table lookup

### **Add Covering Index Too Early**

- Large write cost
- Unproven read benefit
- Premature complexity

## Index Selection Flow

### Small Table?

- Yes → consider full-table scan
- No → evaluate query pattern

### Full-Text Search?

- Yes → inverted index

### Location or Geometry?

- Yes → geospatial index

### Exact Match Only?

- In-memory and strict equality → consider hash
- Otherwise → B-tree

### General Filtering or Sorting?

- B-tree

### Write-Dominant Storage Engine?

- Consider LSM-tree-based system

### Multiple Columns Queried Together?

- Composite index

### Frequent Narrow Read Query?

- Consider covering index

## Interview Strategy

### Basic Product Interview

Mention:

- Query pattern
- B-tree index
- Read improvement
- Write and storage cost
- Composite index when appropriate

### Infrastructure Interview

Be ready to discuss:

- Disk pages
- B-tree fan-out
- LSM-tree write path
- Compaction
- Bloom filters
- Read/write amplification
- Specialized indexes

### Geospatial Interview

Explain:

- Why `(lat, lng)` is insufficient
- Geohash candidate generation
- Neighboring cells
- Exact-distance filtering
- R-tree alternative

### Search Interview

Explain:

- B-tree substring limitation
- Token-to-document mapping
- Analysis pipeline
- Relevance ranking
- Update and storage costs

## Interview Answer Template

> I would start from the dominant query pattern rather than indexing every column. For a lookup by email, I would use a B-tree because it supports efficient equality lookup and preserves flexibility for ordered operations. For repeated queries filtering by user and ordering by creation time, I would use a composite B-tree on `(user_id, created_at)`. The trade-off is additional storage and write amplification because every insert or update must maintain the index. I would validate the design using query plans and production index-usage metrics.

## Specialized Interview Answers

### Write-Heavy Metrics System

> I would consider an LSM-tree-based store because writes can be buffered in memory and flushed as sequential SSTables. This improves ingestion throughput but introduces read amplification and compaction overhead. Bloom filters, sparse indexes, and compaction reduce the read cost.

### Nearby-Location Search

> A normal B-tree on latitude and longitude does not preserve two-dimensional locality. I would use a geohash or native spatial index. With geohash, I would query the center and adjacent cells, then filter candidates using exact geographic distance.

### Full-Text Search

> A B-tree cannot efficiently answer arbitrary substring or relevance-ranked queries. I would use an inverted index that maps normalized terms to document postings, with tokenization, stemming, and relevance scoring.

## Important Terms

- **Index:** Auxiliary structure for faster data access
- **Heap File:** Unordered storage for table rows
- **Disk Page:** Basic database I/O block
- **Full-Table Scan:** Examination of every table page
- **Index Scan:** Index traversal followed by row lookup
- **Index-Only Scan:** Query answered from index data
- **Selectivity:** Fraction of rows filtered by predicate
- **Cardinality:** Number of distinct values
- **B-Tree:** Balanced ordered multiway search tree
- **Fan-Out:** Number of child pointers per tree node
- **Page Split:** Division of a full B-tree page
- **LSM Tree:** Append-and-merge storage structure
- **WAL:** Durable sequential record of writes
- **Memtable:** In-memory sorted write buffer
- **SSTable:** Immutable sorted disk file
- **Compaction:** Background merging of SSTables
- **Bloom Filter:** Probabilistic definite-absence test
- **Sparse Index:** Partial key-to-block mapping
- **Read Amplification:** Extra reads per logical query
- **Write Amplification:** Extra physical writes per logical update
- **Space Amplification:** Extra storage beyond logical data
- **Hash Index:** Hash-table index for equality queries
- **Hash Collision:** Multiple keys mapped to one bucket
- **Geohash:** Locality-preserving coordinate string
- **Quadtree:** Recursive four-way spatial partition
- **R-Tree:** Bounding-rectangle spatial hierarchy
- **Inverted Index:** Term-to-document mapping
- **Posting List:** Documents containing a term
- **Tokenization:** Splitting text into searchable units
- **Composite Index:** Ordered index over multiple columns
- **Leftmost-Prefix Rule:** Composite index usable from its leading columns
- **Covering Index:** Index containing all query-required columns
- **Included Column:** Stored output column not used for ordering
- **Sargable Predicate:** Condition suitable for index seeking
- **Query Plan:** Database execution strategy

## 30-Second Revision Summary

- **Why indexes?** Reduce scanned pages
- **Main cost?** Storage and slower writes
- **Default index?** B-tree
- **B-tree supports?** Equality, range, prefix, sorting
- **Write-heavy option?** LSM-tree-based storage
- **LSM write path?** WAL → memtable → SSTable → compaction
- **LSM read optimization?** Bloom filters and sparse indexes
- **Exact-match-only index?** Hash index
- **Location search?** Geohash, quadtree, or R-tree
- **Geohash radius query?** Center cells + neighbors + exact filtering
- **Full-text search?** Inverted index
- **Multiple query columns?** Composite index
- **Composite-index rule?** Column order matters
- **Leftmost-prefix rule?** Leading columns required
- **Avoid table lookup?** Covering index
- **Small or low-selectivity table?** Full scan may be better
- **Interview default?** B-tree based on access pattern
- **Validation method?** `EXPLAIN ANALYZE`
- **Best principle?** Index queries, not columns
