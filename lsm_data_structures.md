# LSM-Tree Data Structures for Beginners

Let’s build these LSM-tree structures from scratch using one running example. The key idea is that an LSM-based database accepts writes in memory, periodically creates immutable sorted files on disk, and later merges those files.

## 1. The Overall Picture

Suppose an application performs these writes:

```text
PUT user:3 = Charlie
PUT user:1 = Alice
PUT user:2 = Bob
```

An LSM-based database typically moves the data through this pipeline:

```text
Client write
    ↓
Write-Ahead Log
    ↓
Active Memtable
    ↓
Immutable Memtable
    ↓
SSTable on disk
    ↓
Compaction into larger SSTables
```

To make reads efficient, it also uses:

```text
Bloom filters
Sparse indexes
Key-range metadata
Block cache
```

The central strategy is:

> **Accept writes quickly in memory, write sorted files sequentially, and reorganize those files later.**

---

## 2. Write-Ahead Log

### What is the WAL?

The **Write-Ahead Log**, or WAL, is an append-only file containing recent modifications.

Before the database confirms a write, it records the operation in the WAL:

```text
PUT user:3 = Charlie
PUT user:1 = Alice
PUT user:2 = Bob
```

The database appends new entries at the end:

```text
Old entries → New entry → End of file
```

It does not need to find and modify an unrelated location in the file.

### Why do we need it?

The active memtable is stored in memory.

Memory is fast, but volatile:

```text
Server crashes
    ↓
Memory contents disappear
```

Without the WAL, acknowledged writes in the memtable could be lost.

With a WAL:

```text
Server restarts
    ↓
Database reads WAL
    ↓
Replays recent operations
    ↓
Reconstructs memtable
```

### Example

Suppose the database receives:

```text
PUT user:1 = Alice
PUT user:2 = Bob
```

It first appends them to the WAL:

```text
WAL:
1. PUT user:1 = Alice
2. PUT user:2 = Bob
```

It then updates the memtable.

If the server crashes, the database can replay these entries during recovery.

### WAL Invariant

> **A write must become durable in the WAL before the database reports it as durably successful.**

Some databases batch multiple WAL entries before forcing them to stable storage. This improves throughput but affects durability and latency settings.

### What Happens After Flushing?

Once the corresponding memtable has safely become an SSTable, those WAL records are no longer required for recovery.

The old WAL segment can eventually be deleted or recycled.

---

## 3. Memtable

### What is a memtable?

A **memtable** is an in-memory, sorted data structure containing recent writes.

It may be implemented using:

- Skip list
- Balanced search tree
- Other ordered in-memory structures

Suppose writes arrive in this order:

```text
user:3 = Charlie
user:1 = Alice
user:2 = Bob
```

The memtable maintains key order:

```text
user:1 → Alice
user:2 → Bob
user:3 → Charlie
```

### Why must it be sorted?

Eventually, the memtable will be written to disk as an SSTable.

Because the memtable is already sorted, the database can write it sequentially:

```text
user:1 → Alice
user:2 → Bob
user:3 → Charlie
```

It does not need to sort a large collection during the flush or update many random locations on disk.

### Why are memtable writes fast?

A write generally requires:

1. A sequential append to the WAL.
2. An update to an in-memory data structure.

It does not immediately modify a disk page in its final long-term location.

```text
B-tree-style update:
Find disk page → modify page

LSM-style update:
Append to WAL → update memory
```

### Updating the Same Key

Suppose the memtable contains:

```text
user:1 → Alice
```

Then a new write arrives:

```text
PUT user:1 = Alicia
```

The active memtable keeps the latest value:

```text
user:1 → Alicia
```

The older WAL entry may still exist, but recovery replays operations in order, so the latest operation wins.

### Memtable Invariant

> **Within one memtable, keys are ordered, and the latest operation for a key represents its current state in that memtable.**

---

## 4. Immutable Memtable

### Why make a memtable immutable?

A memtable cannot grow forever. It has a configured size limit.

Suppose it reaches that limit:

```text
Active Memtable A: FULL
```

The database freezes it:

```text
Immutable Memtable A
```

It then immediately creates a new active memtable:

```text
Immutable Memtable A → waiting for flush
Active Memtable B    → accepting new writes
```

This separation is important.

The background thread can flush Memtable A to disk while application writes continue entering Memtable B.

### Why not keep modifying the old memtable during flush?

If the structure changed while it was being written to disk, the resulting file could be inconsistent.

Making it immutable gives the flush process a stable snapshot:

```text
Immutable memtable:
No inserts
No updates
No deletions
```

### Can there be multiple immutable memtables?

Yes.

If writes arrive faster than the database can flush them:

```text
Active memtable
Immutable memtable 1
Immutable memtable 2
Immutable memtable 3
```

Too many waiting memtables indicate that storage is not keeping up.

The database may:

- Slow incoming writes.
- Temporarily stop writes.
- Increase background flush activity.

This is called **write stalling** or **backpressure**.

### Important Distinction

```text
Active memtable:
Mutable and receiving writes

Immutable memtable:
Read-only and waiting to be flushed

SSTable:
Read-only file already stored on disk
```

---

## 5. SSTable

### What is an SSTable?

**SSTable** commonly means **Sorted String Table**.

It is an immutable disk file that stores key-value entries in sorted key order.

Example:

```text
user:1 → Alice
user:2 → Bob
user:3 → Charlie
user:7 → Grace
```

The term “string” is historical. Keys and values do not necessarily have to be human-readable strings.

They can contain serialized binary data.

### Why are SSTables immutable?

After an SSTable is created, the database does not update individual entries inside it.

If a key changes, the new version is written elsewhere:

```text
Older SSTable:
user:1 → Alice

Newer SSTable:
user:1 → Alicia
```

During a read, the newer version wins.

Later, compaction can merge the files and discard the obsolete version.

### Why is immutability useful?

It allows:

- Sequential file creation.
- Simple concurrent reads.
- No in-place file modifications.
- Easier caching.
- Background compaction.
- Efficient replication and file movement.
- Simpler recovery behavior.

The cost is that old versions temporarily remain on disk.

### Conceptual SSTable Structure

An SSTable is more than one flat sequence. Conceptually, it may contain:

```text
+----------------------------------+
| Data block 1                     |
| apple → value                    |
| banana → value                   |
+----------------------------------+
| Data block 2                     |
| carrot → value                   |
| dog → value                      |
+----------------------------------+
| Data block 3                     |
| elephant → value                 |
| fox → value                      |
+----------------------------------+
| Sparse/block index               |
+----------------------------------+
| Bloom filter                     |
+----------------------------------+
| Metadata and footer              |
+----------------------------------+
```

The exact format depends on the database.

### SSTable Invariant

> **Entries inside an SSTable are ordered, and the file is never modified after it is created.**

---

## 6. Why Multiple Versions Exist

Consider this sequence:

```text
Day 1:
user:1 → Alice

Day 2:
user:1 → Alicia

Day 3:
user:1 → Ally
```

These versions may be distributed across several structures:

```text
Active memtable:
user:1 → Ally

New SSTable:
user:1 → Alicia

Old SSTable:
user:1 → Alice
```

A read must find the newest visible version.

A simplified search order is:

```text
1. Active memtable
2. Immutable memtables, newest first
3. Newer SSTables
4. Older SSTables
```

If the database finds:

```text
user:1 → Ally
```

in the active memtable, that value is newer than the versions in the SSTables.

Real systems use sequence numbers, timestamps, or internal version identifiers rather than relying only on file names.

### Versioning Invariant

> **When the same key is present in multiple places, the logically newest visible version wins.**

---

## 7. Sparse Index

### The Problem

An SSTable may contain millions of entries.

Even though it is sorted, reading the entire file to find one key would be expensive.

A **sparse index** stores pointers for selected keys or data blocks.

Suppose an SSTable contains:

```text
apple
apricot
banana
blueberry
carrot
coconut
date
fig
grape
```

A sparse index might contain:

```text
apple  → Block 1
carrot → Block 2
date   → Block 3
```

To search for `coconut`, the database can determine:

```text
carrot ≤ coconut < date
```

Therefore, it searches Block 2.

### Why not index every key?

A full index would require more memory.

A sparse index is smaller because one index entry can represent an entire block:

```text
One index entry
    ↓
One data block
    ↓
Many key-value entries
```

Because data inside the block is sorted, the database can use binary search or another efficient local search.

### Sparse-Index Invariant

> **Because the SSTable is sorted, a small set of boundary keys can identify the block that may contain the target.**

A sparse index tells the database **where to look inside an SSTable**.

It does not tell the database whether the key definitely exists.

That is where the Bloom filter helps.

---

## 8. Bloom Filter

### What problem does it solve?

Imagine that the database has 20 SSTables.

You search for:

```text
user:999
```

Without additional metadata, the database may need to inspect each SSTable:

```text
SSTable 1: not found
SSTable 2: not found
SSTable 3: not found
...
SSTable 20: found
```

Most of these reads are wasted.

A Bloom filter provides a cheap membership test for each SSTable.

It answers:

```text
Definitely not present
Possibly present
```

### How does a Bloom filter work?

A Bloom filter contains a bit array:

```text
Initial bits:

0 0 0 0 0 0 0 0 0 0 0 0
```

When inserting a key, several hash functions select bit positions.

For example:

```text
hash1("alice") → position 2
hash2("alice") → position 6
hash3("alice") → position 9
```

Those bits are set to 1:

```text
0 0 1 0 0 0 1 0 0 1 0 0
```

Another key sets other positions.

To test a key, the Bloom filter calculates the same positions.

#### If any required bit is zero

```text
Key is definitely absent.
```

Why?

If the key had been inserted, all of its positions would have been set to one.

#### If all required bits are one

```text
Key is possibly present.
```

The bits may have been set by different keys.

Therefore, the database must check the actual SSTable.

### False Positive

The Bloom filter says:

```text
Possibly present
```

But the key is not actually there.

This is allowed.

It causes an unnecessary SSTable check, but it does not return an incorrect database result.

### False Negative

The Bloom filter says:

```text
Definitely absent
```

But the key actually exists.

A correctly implemented Bloom filter should not produce false negatives, assuming normal operation and that all relevant keys were inserted into it.

### Memory Trick

> **No means no. Maybe means check.**

### Bloom-Filter Limitation

A standard Bloom filter is mainly useful for membership tests such as:

```text
Does this SSTable possibly contain user:123?
```

It does not naturally answer:

```text
Find all users between user:100 and user:200
```

Range queries rely more heavily on sorted data, key-range metadata, and sparse indexes.

### Bloom-Filter Invariant

> **A negative result safely skips the SSTable; a positive result requires verification.**

---

## 9. Key-Range Metadata

Every SSTable can record its smallest and largest keys.

Example:

```text
SSTable A:
minimum = user:001
maximum = user:100

SSTable B:
minimum = user:200
maximum = user:300
```

For a search of:

```text
user:150
```

the database can skip both files immediately:

```text
user:150 is outside 001–100
user:150 is outside 200–300
```

This is cheaper than consulting the complete file.

### Range Metadata Versus Bloom Filter

They solve related but different problems.

```text
Key-range metadata:
Could this key lie inside the file's overall range?

Bloom filter:
Was this exact key probably added to this file?

Sparse index:
Which block inside the file might contain the key?
```

A point lookup may use all three:

```text
Check key range
    ↓
Check Bloom filter
    ↓
Consult sparse index
    ↓
Read candidate data block
    ↓
Verify actual key
```

---

## 10. Block Cache

### What is a block cache?

Reading a block from disk repeatedly is expensive.

A **block cache** keeps frequently accessed SSTable blocks in memory.

Suppose many requests ask for:

```text
user:1
user:2
user:3
```

and all three entries are in the same SSTable block.

After the first disk read, the block may remain in cache:

```text
First read:
Disk → Memory → Result

Later read:
Memory → Result
```

### What gets cached?

Depending on the system, the cache may hold:

- Data blocks
- Index blocks
- Bloom-filter data
- Metadata

### Memtable Versus Block Cache

Both use memory, but they have different jobs.

```text
Memtable:
Contains recent writes that have not yet been flushed.

Block cache:
Contains copies of disk blocks that are frequently read.
```

Losing a block cache is usually not a durability problem because the blocks still exist on disk.

Losing a memtable could lose recent writes, which is why the WAL is required.

---

## 11. Tombstones

### How is a key deleted from an immutable SSTable?

Suppose an old SSTable contains:

```text
user:1 → Alice
```

Because SSTables are immutable, the database cannot simply erase that entry.

Instead, it writes a special deletion marker called a **tombstone**:

```text
DELETE user:1
```

or conceptually:

```text
user:1 → TOMBSTONE
```

This newer record means:

> Treat older versions of `user:1` as deleted.

### Read Behavior

The structures contain:

```text
New memtable:
user:1 → TOMBSTONE

Old SSTable:
user:1 → Alice
```

The database finds the newer tombstone and returns “not found.” It must not continue to the old value and return Alice.

### Why can’t the tombstone be removed immediately?

The old value may still exist in another SSTable.

If the tombstone were deleted too early, the old value could become visible again:

```text
Remove tombstone
    ↓
Old SSTable still contains Alice
    ↓
Alice incorrectly reappears
```

This is sometimes called **data resurrection**.

The tombstone can be removed only when the database knows that no relevant older version can reappear, usually during a safe compaction process.

### Tombstone Invariant

> **A deletion must remain visible until all older versions it hides have been safely eliminated.**

Large numbers of tombstones can increase read and storage costs.

---

## 12. Compaction

### Why is compaction necessary?

Over time, the database creates many SSTables:

```text
SSTable 1
SSTable 2
SSTable 3
SSTable 4
SSTable 5
```

They may contain:

- Duplicate keys
- Obsolete values
- Tombstones
- Overlapping key ranges

Without compaction, reads may need to examine an increasing number of files.

Compaction merges SSTables into new SSTables.

### Merge Example

Two sorted SSTables:

```text
Older SSTable:

A → 10
B → 20
D → 40
```

```text
Newer SSTable:

B → 25
C → 30
D → TOMBSTONE
```

During compaction, the database performs a sorted merge:

```text
A → 10
B → 25
C → 30
```

Here:

- The newer value of `B` replaces the older one.
- `C` is retained.
- The old value of `D` is suppressed by the tombstone.
- The tombstone may be removed if it is safe to do so.

### Why is merging efficient?

Both input files are already sorted.

The database can walk through them from beginning to end:

```text
File A pointer →
File B pointer →
Output pointer →
```

This is similar to the merge phase of merge sort.

The work is approximately linear in the amount of data being merged.

### Compaction Benefits

Compaction:

- Reduces the number of SSTables.
- Removes obsolete versions.
- Eventually removes safe tombstones.
- Improves read performance.
- Reorganizes key ranges.
- Reclaims some disk space.

### Compaction Costs

Compaction:

- Reads existing files.
- Writes new files.
- Uses CPU.
- Uses disk bandwidth.
- Temporarily requires additional storage.
- Can interfere with foreground reads and writes.

This creates write amplification.

For example:

```text
Application writes 1 MB
Database flushes 1 MB
Compactions later rewrite that data several times
```

The total physical writes may be much greater than 1 MB.

---

## 13. Size-Tiered and Leveled Compaction

### Size-Tiered Compaction

Similarly sized SSTables are grouped and merged.

```text
Four 10 MB files
    ↓
One approximately 40 MB file
```

#### Benefit

Files are generally not rewritten as aggressively, which can help write throughput.

#### Cost

Several SSTables may have overlapping key ranges:

```text
SSTable A: A–Z
SSTable B: A–Z
SSTable C: A–Z
```

A read may need to check multiple files.

### Leveled Compaction

SSTables are organized into levels:

```text
Level 0: New small files
Level 1: Larger organized files
Level 2: Still larger files
Level 3: Much larger files
```

Within many levels, key ranges are arranged to reduce overlap.

```text
Level 1:

SSTable A: A–F
SSTable B: G–M
SSTable C: N–Z
```

A point lookup may need to consult only one candidate file at that level.

#### Benefit

- Better read performance.
- More predictable file selection.
- Lower space amplification in many configurations.

#### Cost

Moving data through levels can rewrite it multiple times.

```text
Flush → Level 0
Level 0 → Level 1
Level 1 → Level 2
Level 2 → Level 3
```

This can create greater write amplification.

### Fundamental Trade-Off

> **Less compaction means cheaper writes but more work during reads. More compaction means better-organized reads but more rewriting.**

---

## 14. Complete Write Example

Suppose we write:

```text
PUT A = 10
PUT C = 30
PUT B = 20
```

### Step 1: WAL

```text
WAL:
PUT A = 10
PUT C = 30
PUT B = 20
```

### Step 2: Active Memtable

The memtable keeps keys sorted:

```text
A → 10
B → 20
C → 30
```

### Step 3: Memtable Becomes Full

```text
Active Memtable 1
    ↓ freeze
Immutable Memtable 1
```

A new active memtable starts accepting writes.

### Step 4: Flush

The immutable memtable becomes an SSTable:

```text
SSTable 1:

A → 10
B → 20
C → 30
```

The file also receives structures such as:

```text
Minimum key: A
Maximum key: C
Sparse index: key boundaries → data blocks
Bloom filter: approximate membership information
```

### Step 5: Retire Old WAL Data

After the SSTable is safely persisted, the corresponding obsolete WAL segment can eventually be removed.

---

## 15. Complete Read Example

Suppose the database needs:

```text
GET B
```

It may perform:

### Step 1: Check the Active Memtable

```text
B not found
```

### Step 2: Check Immutable Memtables

```text
B not found
```

### Step 3: Select Candidate SSTables

Check key ranges:

```text
SSTable 1 range: A–C
```

`B` is inside the range, so continue.

### Step 4: Check the Bloom Filter

```text
B may be present
```

Continue.

If it said “definitely absent,” the database would skip the file.

### Step 5: Use the Sparse Index

```text
B should be in Data Block 1
```

### Step 6: Get the Block

First check the block cache.

```text
Cache hit  → read from memory
Cache miss → read block from disk
```

### Step 7: Verify the Key

The database searches the actual block:

```text
A → 10
B → 20
C → 30
```

Result:

```text
B → 20
```

Notice that the Bloom filter never supplies the value. It only helps avoid useless file reads.

---

## 16. How the Structures Work Together

```text
WAL
Purpose: durability and crash recovery
Question answered: Can recent writes be reconstructed?

Memtable
Purpose: fast in-memory writes
Question answered: Is the newest value still in active memory?

Immutable memtable
Purpose: stable snapshot waiting for disk flush
Question answered: Is the value in memory that is currently being flushed?

SSTable
Purpose: immutable sorted disk storage
Question answered: Is a persisted version stored in this file?

Key-range metadata
Purpose: skip files outside the target range
Question answered: Could the target key lie within this file's bounds?

Bloom filter
Purpose: avoid unnecessary point-lookup reads
Question answered: Is the key definitely absent or possibly present?

Sparse index
Purpose: locate a candidate block in an SSTable
Question answered: Which part of this file should be read?

Block cache
Purpose: avoid repeated disk reads
Question answered: Is the required disk block already in memory?

Tombstone
Purpose: represent deletion without modifying old files
Question answered: Has a newer operation logically deleted this key?

Compaction
Purpose: merge files and eliminate obsolete data
Question answered: How can accumulated files and versions be reorganized?
```

---

## 17. Data-Structure Relationships

A useful way to remember the entire system is:

```text
Write safety:
WAL

Fast recent writes:
Memtable

Safe background flushing:
Immutable memtable

Permanent sorted storage:
SSTable

Skip impossible files:
Key-range metadata

Skip files that probably do not contain the exact key:
Bloom filter

Find the correct block:
Sparse index

Avoid repeated disk reads:
Block cache

Represent deletion:
Tombstone

Clean and reorganize everything:
Compaction
```

---

## 18. The Three Most Important Invariants

### Invariant 1: Newer Versions Win

```text
Memtable: user:1 → Alicia
Old SSTable: user:1 → Alice
```

Return `Alicia`.

### Invariant 2: Immutability Moves Updates Elsewhere

An SSTable is never edited in place.

Updates and deletions create newer records. Compaction later removes obsolete records.

### Invariant 3: Auxiliary Structures May Skip Work, but the Actual Data Determines Correctness

A Bloom filter, sparse index, or key range helps locate or skip data.

The database must still verify the actual entry before returning a result.

---

## 19. Thirty-Second Revision

```text
WAL:
Durable append-only record used for crash recovery.

Memtable:
Sorted in-memory structure receiving current writes.

Immutable memtable:
Frozen memtable waiting to become an SSTable.

SSTable:
Immutable sorted key-value file on disk.

Bloom filter:
Says “definitely absent” or “possibly present.”

Sparse index:
Maps selected keys to SSTable data blocks.

Key-range metadata:
Stores minimum and maximum keys for a file.

Block cache:
Keeps frequently accessed disk blocks in memory.

Tombstone:
A newer record indicating that a key was deleted.

Compaction:
Merges SSTables and removes obsolete versions when safe.
```

The simplest complete mental model is:

```text
WAL protects the write
    ↓
Memtable absorbs the write
    ↓
SSTable stores the write
    ↓
Bloom filter avoids useless searches
    ↓
Sparse index locates the disk block
    ↓
Compaction cleans accumulated files
```
