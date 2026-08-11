# Contention and Concurrency Control: System Design Notes

## Core Thesis

**Contention occurs when multiple operations compete for the same logical resource at the same time.**

Examples:

- Two users attempt to purchase the same seat.
- Several bidders update the same auction.
- Two transfers modify the same account.
- Multiple requests claim the same driver.
- Thousands of buyers decrement the same inventory item.

Without coordination, concurrent operations can produce:

- Double-booking
- Overselling
- Lost updates
- Invalid balances
- Duplicate work
- Broken cross-row invariants
- Inconsistent application state

The central design principle is:

> **Put every contended resource behind one authoritative state transition, then use the simplest mechanism capable of making that transition safe.**

---

# 1. The Race Condition

Suppose a concert has one seat remaining.

Alice and Bob both execute this logical sequence:

```text
1. Read available seats.
2. Check whether the value is greater than zero.
3. Charge the customer.
4. Decrement the number of seats.
5. Create a ticket.
```

A possible interleaving is:

```text
Time        Alice                         Bob
----------------------------------------------------------------
t1          Reads available_seats = 1
t2                                        Reads available_seats = 1
t3          Decides purchase is valid
t4                                        Decides purchase is valid
t5          Changes count from 1 to 0
t6                                        Changes count from 0 to -1
```

Both requests made their decision from the same stale value.

The application incorrectly treated these operations as one unit:

```text
Read → Decide → Write
```

But the database observed separate operations with a gap between them.

That gap is the race window.

---

# 2. Lost Update and Read-Modify-Write

A **read-modify-write cycle** has three conceptual phases:

```text
Read current state
    ↓
Compute or decide using that state
    ↓
Write a new state
```

A **lost update** occurs when concurrent operations read the same state and later write conflicting results, causing one operation to overwrite or invalidate another.

Example:

```text
Initial value: 10

Transaction A reads 10 and computes 9.
Transaction B reads 10 and computes 9.
Transaction A writes 9.
Transaction B writes 9.

Expected after two decrements: 8
Actual value: 9
```

The important invariant is:

> **A decision based on state must not commit if the state that justified the decision is no longer valid.**

Every concurrency-control strategy protects this invariant in a different way.

---

# 3. Model the Actual Contended Resource

Before choosing a lock or transaction mechanism, identify what users are actually competing for.

## Incorrect Model

Suppose a concert has 20 available seats and the system stores only this counter:

```text
available_seats = 20
```

Alice and Bob both request seat `A15`.

Two counter decrements can safely change the number from 20 to 18, but that does not prevent both users from receiving `A15`.

The counter answers:

> Is any seat available?

It does not answer:

> Is seat A15 available?

## Better Model

Represent each seat as an addressable resource:

```text
seat_id | concert_id | seat_number | status    | owner
-----------------------------------------------------------
101     | c1         | A15         | available | NULL
102     | c1         | A16         | available | NULL
```

Now the database can guard the exact seat:

```sql
UPDATE seats
SET status = 'sold',
    user_id = :user_id
WHERE concert_id = :concert_id
  AND seat_number = :seat_number
  AND status = 'available';
```

If one row is affected, the claim succeeded.

If zero rows are affected, the seat was not available at commit time.

### Resource-Modeling Invariant

> **The thing being contested must exist as a row, key, item, partition, or other authoritative addressable unit.**

---

# 4. Strategy 1: Conditional Writes

## Definition

A conditional write places the validation rule directly in the write operation.

Instead of:

```text
Read availability
Check it in application code
Write later
```

use:

```text
Write only if the current database state still satisfies the rule
```

## Counter Example

```sql
UPDATE concerts
SET available_seats = available_seats - 1
WHERE concert_id = :concert_id
  AND available_seats > 0;
```

The database atomically evaluates the predicate and performs the update.

## Seat Example

```sql
UPDATE seats
SET status = 'sold',
    user_id = :user_id,
    sold_at = NOW()
WHERE seat_id = :seat_id
  AND status = 'available';
```

## Application Contract

```text
rows_affected == 1 → success
rows_affected == 0 → precondition failed
```

A zero-row update is usually not a SQL error. It is a valid statement that matched nothing. The application must interpret the affected-row count.

## Transaction With Dependent Insert

A purchase may require both:

1. Claiming the seat.
2. Creating a ticket record.

Both operations must succeed or fail together.

```sql
BEGIN;

UPDATE seats
SET status = 'sold',
    user_id = :user_id,
    sold_at = NOW()
WHERE seat_id = :seat_id
  AND status = 'available';

-- Application checks the affected-row count here.
-- Continue only when exactly one row changed.

INSERT INTO tickets (
    ticket_id,
    seat_id,
    user_id,
    purchase_time
)
VALUES (
    :ticket_id,
    :seat_id,
    :user_id,
    NOW()
);

COMMIT;
```

Application logic:

```text
BEGIN
rows = execute(conditional_update)

if rows == 0:
    ROLLBACK
    return "Seat already taken"

execute(insert_ticket)
COMMIT
```

## PostgreSQL Data-Modifying CTE Alternative

The insert can depend directly on the update result:

```sql
BEGIN;

WITH claimed_seat AS (
    UPDATE seats
    SET status = 'sold',
        user_id = :user_id,
        sold_at = NOW()
    WHERE seat_id = :seat_id
      AND status = 'available'
    RETURNING seat_id
)
INSERT INTO tickets (
    ticket_id,
    seat_id,
    user_id,
    purchase_time
)
SELECT
    :ticket_id,
    seat_id,
    :user_id,
    NOW()
FROM claimed_seat;

COMMIT;
```

If no seat is claimed, `claimed_seat` contains no rows and the insert writes nothing.

## When to Use

Conditional writes are ideal when the rule is a predicate on the same item being written:

- Decrement inventory only when inventory is positive.
- Mark a seat sold only when it is available.
- Claim a task only when it is unassigned.
- Ship an order only when it is paid and not cancelled.
- Create a lock key only when it does not exist.

## Advantages

- One atomic write
- Low latency
- Little application complexity
- No explicit lock lifecycle
- Works well under ordinary contention

## Limitations

- The business rule must fit into the write predicate.
- It does not protect application-side decisions made after a complex read.
- It protects only the resource represented by the guarded row or key.

## Equivalent Mechanisms

The same compare-and-set idea appears as:

- SQL `UPDATE ... WHERE ...`
- DynamoDB conditional expressions
- Redis `SET ... NX`
- Cassandra lightweight transactions
- HTTP `If-Match`
- etcd revision comparisons

---

# 5. Compare-and-Set as the Unifying Primitive

Most contention mechanisms reduce to a form of **compare-and-set**, or CAS:

```text
If current state equals expected state:
    install new state
else:
    report conflict
```

Examples:

```text
Conditional write:
status must still equal "available"

Optimistic concurrency:
version must still equal 42

Lease acquisition:
reserved_until must be absent or expired

HTTP update:
ETag must still match the client's If-Match value
```

### CAS Invariant

> **A write succeeds only when the state used to justify it is still current.**

---

# 6. Strategy 2: Pessimistic Locking

## Definition

Pessimistic locking assumes concurrent conflicts are likely or costly. It acquires a lock before the application makes its decision.

In SQL, row locks are commonly acquired with:

```sql
SELECT ... FOR UPDATE;
```

## When a Conditional Write Is Insufficient

Suppose a group needs four adjacent seats.

The application must:

1. Read the available seat map.
2. Search for a contiguous block.
3. Choose a block.
4. Claim the selected rows.

The choice of adjacent seats may be too complex to express as one simple row predicate. There is application logic between the read and write.

## Example

```sql
BEGIN;

SELECT seat_id, row_number, seat_number
FROM seats
WHERE concert_id = :concert_id
  AND section = :section
  AND status = 'available'
ORDER BY row_number, seat_number
FOR UPDATE;

-- Application chooses four adjacent seats from the locked rows.

UPDATE seats
SET status = 'sold',
    user_id = :user_id,
    sold_at = NOW()
WHERE seat_id IN (:seat_1, :seat_2, :seat_3, :seat_4);

COMMIT;
```

While the transaction holds the locks, conflicting transactions must wait or fail according to database behavior and options.

## Lock Lifecycle

```text
BEGIN
    ↓
Acquire row locks
    ↓
Read stable state
    ↓
Make decision
    ↓
Write
    ↓
COMMIT or ROLLBACK
    ↓
Locks released
```

## When to Use

Use pessimistic locking when:

- The operation must read, decide in application code, and write.
- Contention is high.
- Retrying expensive work would be costly.
- Multiple related rows must remain stable while a decision is made.
- A predictable queue is preferable to repeated optimistic failures.

## Advantages

- Straightforward correctness model
- Prevents conflicting modifications upfront
- Useful under high contention
- Avoids repeated application retries when conflicts are common

## Disadvantages

- Waiting increases latency.
- Long transactions reduce throughput.
- Broad locks serialize unrelated work.
- Deadlocks are possible.
- A slow lock holder can block many requests.

---

# 7. Lock Only What You Need, Only as Long as Needed

A lock reduces concurrency. Its scope and duration should be minimized.

## Bad Pattern

```text
BEGIN transaction
Lock seat
Call payment provider
Wait several seconds
Update database
COMMIT
```

The database connection and lock remain occupied while an external dependency responds.

## Better Separation

A safer flow may use a reservation lease:

```text
1. Atomically reserve seat for a short period.
2. Commit database transaction.
3. Call payment provider without holding a row lock.
4. Finalize purchase with another conditional transaction.
5. Release or expire the reservation on failure.
```

## Lock-Scope Rules

- Prefer row locks over table locks.
- Lock only rows involved in the decision.
- Avoid network calls while holding database locks.
- Avoid user interaction inside a transaction.
- Keep transactions measured in milliseconds when possible.
- Monitor lock wait time and transaction age.

---

# 8. Deadlocks

## Definition

A deadlock occurs when transactions form a cycle of waiting dependencies.

Example:

```text
Transaction A holds Account 1 and waits for Account 2.
Transaction B holds Account 2 and waits for Account 1.
```

Neither can continue.

## Transfer Example

Unsafe order:

```text
Transfer Alice → Bob:
lock Alice, then Bob

Transfer Bob → Alice:
lock Bob, then Alice
```

Concurrent execution can deadlock.

## Prevention: Ordered Locking

Acquire all locks in a globally consistent order.

```text
Sort account IDs ascending.
Lock the smaller ID first.
Lock the larger ID second.
```

SQL sketch:

```sql
BEGIN;

SELECT account_id, balance
FROM accounts
WHERE account_id IN (:account_a, :account_b)
ORDER BY account_id
FOR UPDATE;

-- Validate and apply the transfer.

COMMIT;
```

The order is based on a stable key, not business-role order such as sender first.

## Detection and Retry

Databases commonly detect deadlock cycles and abort one transaction.

The application should treat the deadlock error as retryable:

```text
attempt operation
if deadlock or serialization failure:
    wait with randomized backoff
    retry within a bounded limit
```

A lock-wait timeout is different from deadlock detection. A timeout can occur when a transaction is blocked behind a long-running lock without a dependency cycle.

### Deadlock Invariant

> **All code paths that lock the same resource types must use the same acquisition order.**

---

# 9. Strategy 3: Optimistic Concurrency Control

## Definition

Optimistic concurrency control, or OCC, assumes conflicts are uncommon.

It allows operations to proceed without holding a lock across the read-decision gap. At write time, the operation verifies that the row has not changed.

## Version-Column Pattern

Schema:

```sql
CREATE TABLE concerts (
    concert_id TEXT PRIMARY KEY,
    available_seats INTEGER NOT NULL,
    version BIGINT NOT NULL DEFAULT 0
);
```

Read:

```sql
SELECT available_seats, version
FROM concerts
WHERE concert_id = :concert_id;
```

Suppose Alice and Bob both read:

```text
available_seats = 1
version = 42
```

Alice writes first:

```sql
UPDATE concerts
SET available_seats = available_seats - 1,
    version = version + 1
WHERE concert_id = :concert_id
  AND available_seats > 0
  AND version = 42;
```

Alice changes the row to:

```text
available_seats = 0
version = 43
```

Bob then uses the stale expected version:

```sql
UPDATE concerts
SET available_seats = available_seats - 1,
    version = version + 1
WHERE concert_id = :concert_id
  AND available_seats > 0
  AND version = 42;
```

Bob's statement affects zero rows.

## Correct Transaction Pattern

The database does not automatically skip later statements merely because an update affected zero rows. The application must branch explicitly.

```text
BEGIN

rows_affected = UPDATE ... WHERE version = expected_version

if rows_affected == 0:
    ROLLBACK
    return conflict

INSERT ticket
COMMIT
```

## SQL and Application Pseudocode

```sql
BEGIN;

UPDATE concerts
SET available_seats = available_seats - 1,
    version = version + 1
WHERE concert_id = :concert_id
  AND available_seats > 0
  AND version = :expected_version;

-- Application checks rows_affected before continuing.

INSERT INTO tickets (
    ticket_id,
    user_id,
    concert_id,
    seat_number,
    purchase_time
)
VALUES (
    :ticket_id,
    :user_id,
    :concert_id,
    :seat_number,
    NOW()
);

COMMIT;
```

```text
rows = execute(update)

if rows != 1:
    rollback()
    return "Conflict: state changed"

execute(insert_ticket)
commit()
```

## Conflict Handling

After a conflict, the application may:

- Re-read and retry.
- Inform the user that the resource changed.
- Select another resource.
- Retry with exponential backoff and jitter.
- Stop immediately if the resource is no longer available.

## When to Use

Use OCC when:

- Conflicts are rare.
- Reads greatly outnumber writes.
- Blocking would unnecessarily reduce concurrency.
- Retrying the operation is safe and relatively cheap.
- Users may edit data for a while before submitting changes.

## Advantages

- No lock held during read or user think time
- Excellent performance when conflicts are rare
- High read concurrency
- Natural fit for stateless applications and APIs

## Disadvantages

- Conflicting work is discarded.
- Retries can amplify load.
- High contention can cause retry storms or starvation.
- Every write path must update the version consistently.

---

# 10. Choosing a Version Token

A version token must detect every meaningful intervening update.

## Dedicated Monotonic Version

Recommended default:

```text
42 → 43 → 44 → 45
```

It changes on every update and never moves backward.

## Timestamp

A timestamp can work if:

- Every write updates it.
- Precision is sufficient.
- The system does not depend on unsafe cross-node clock assumptions.

A dedicated integer is usually easier to reason about.

## Business Value

Sometimes a business value can be the expected token:

```text
Auction high bid: 500 → 550 → 600
```

This is safer when it moves monotonically.

It becomes risky when the value can return to an earlier state.

---

# 11. The ABA Problem

## Definition

The ABA problem occurs when a checked value changes from `A` to `B` and later returns to `A`.

An equality check sees `A` and incorrectly concludes that nothing changed.

Example:

```text
Application reads review_count = 100.
Another transaction deletes a review: 100 → 99.
Another transaction adds a review: 99 → 100.
Application checks review_count = 100 and succeeds.
```

The count matches, but meaningful state transitions occurred.

## Safe Solution

Use a dedicated monotonically increasing version:

```sql
UPDATE restaurants
SET avg_rating = :new_average,
    review_count = :new_count,
    version = version + 1
WHERE restaurant_id = :restaurant_id
  AND version = :expected_version;
```

The version changes even when business values return to previous values.

## Alternative

If a version cannot be added, compare all relevant fields that justified the decision. This is more cumbersome and can still be fragile if some meaningful field is omitted.

### Version Invariant

> **Every successful logical update must produce a version token that has never represented an earlier state of the same record.**

---

# 12. Pessimistic Versus Optimistic Concurrency

## Pessimistic Locking

Choose it when:

- Conflicts are frequent.
- The resource is hot.
- Work performed before commit is expensive.
- Retrying creates unacceptable cost.
- A stable multi-row view is required.

Behavior:

```text
Wait before doing conflicting work
```

## Optimistic Concurrency

Choose it when:

- Conflicts are rare.
- Reads dominate.
- Operations are easy to retry.
- Blocking would hurt throughput.

Behavior:

```text
Proceed concurrently, detect conflict at commit or write time
```

## Practical Rule

> **When waiting is cheaper than repeated failure, lock. When failure is rare and cheap, use OCC.**

---

# 13. Transactions

A transaction groups related database changes into one atomic unit.

```text
BEGIN
    operation A
    operation B
    operation C
COMMIT
```

If any required condition fails:

```text
ROLLBACK
```

## Why Transactions Matter

Suppose the application successfully decrements inventory but crashes before creating the order.

Without a transaction:

```text
Inventory reduced
Order missing
```

With a transaction:

```text
Both changes commit
or
Neither change commits
```

## Important Distinction

Atomic transaction execution does not automatically make an unsafe read-modify-write algorithm correct. The transaction must also use an appropriate conditional write, lock, version check, or isolation level.

---

# 14. Isolation Levels

Isolation controls how concurrent transactions interact and which anomalies are permitted.

Common names include:

1. Read Uncommitted
2. Read Committed
3. Repeatable Read
4. Serializable

Precise behavior varies by database implementation.

## Read Uncommitted

May permit reading data another transaction has not committed.

Potential anomaly:

- Dirty read

It is uncommon for correctness-sensitive application logic.

## Read Committed

A statement reads only committed data, but two statements in the same transaction may observe different committed versions.

Possible anomalies can include:

- Non-repeatable reads
- Lost-update patterns when application logic is unsafe
- Write skew, depending on the operation and engine behavior

## Repeatable Read

Repeated reads typically observe a stable snapshot or equivalent view during the transaction.

It prevents some anomalies but does not universally guarantee serial behavior. Write-skew behavior depends on the database's exact implementation.

## Serializable

Serializable isolation guarantees an outcome equivalent to some serial execution of the transactions.

The database may implement it using:

- Strict two-phase locking
- Serializable snapshot isolation
- Predicate or range locking
- Conflict detection and aborts

The application must be prepared to retry serialization failures.

---

# 15. Write Skew

## Definition

Write skew occurs when concurrent transactions:

1. Read overlapping data.
2. Make individually valid decisions.
3. Write different rows.
4. Jointly violate a cross-row invariant.

## On-Call Example

Invariant:

```text
At least one engineer must remain on call.
```

Initial state:

```text
Alice: active
Bob: active
```

Concurrent transactions:

```text
Alice sees Bob active and deactivates Alice.
Bob sees Alice active and deactivates Bob.
```

Each transaction writes a different row, so a row-level version check on each engineer may not detect a conflict.

Final invalid state:

```text
Alice: inactive
Bob: inactive
```

## Serializable Solution

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT COUNT(*)
FROM on_call
WHERE team_id = :team_id
  AND is_active = TRUE;

-- Application deactivates the engineer only when the invariant allows it.

UPDATE on_call
SET is_active = FALSE
WHERE team_id = :team_id
  AND engineer_id = :engineer_id;

COMMIT;
```

When concurrent transactions create a result that cannot correspond to a serial order, one may abort with a serialization failure.

## Materialized-Conflict Alternative

Move the invariant onto one row:

```text
team row:
active_on_call_count
```

or lock a shared team row before modifying membership:

```sql
SELECT team_id
FROM teams
WHERE team_id = :team_id
FOR UPDATE;
```

Now both transactions contend on the same addressable row.

### Cross-Row Invariant Principle

> **If practical, materialize a cross-row invariant onto one row or key. Otherwise, use an isolation mechanism capable of detecting predicate-level conflicts.**

---

# 16. Strategy 4: Serializable Isolation

## When to Use

Serializable isolation is appropriate when:

- The invariant spans several rows.
- Concurrent operations write different rows.
- Simpler row guards cannot expose the conflict.
- Correctness requires behavior equivalent to serial execution.

## Cost

Serializable execution may add:

- Conflict tracking
- Predicate or range locks
- More blocking
- Transaction aborts
- Retry work
- Lower throughput on hot workloads

## Retry Pattern

```text
for attempt in bounded_attempts:
    begin serializable transaction
    try:
        read state
        validate invariant
        write changes
        commit
        return success
    catch serialization_failure:
        rollback
        wait with jitter

return retryable_error
```

Use SERIALIZABLE because the invariant requires it, not merely because it sounds safer.

---

# 17. Strategy 5: Reservation Leases and Distributed Locks

## Why Transaction Locks Are Not Enough

A database row lock should not be held while a user spends ten minutes entering payment details.

Doing so would:

- Hold a database connection.
- Block other transactions.
- Increase deadlock and timeout risk.
- Couple correctness to one process and one transaction.

Instead, represent the hold as durable or shared state with an expiration time.

```text
seat A15
reserved_by = user123
reserved_until = 12:10:00
```

This is a **lease** or temporary reservation.

## Database-Backed Lease

```sql
UPDATE seats
SET reserved_by = :user_id,
    reserved_until = NOW() + INTERVAL '10 minutes'
WHERE seat_id = :seat_id
  AND status = 'available'
  AND (
      reserved_until IS NULL
      OR reserved_until < NOW()
  );
```

Interpretation:

```text
rows_affected == 1 → lease acquired
rows_affected == 0 → resource held or unavailable
```

## Finalization

The final purchase must verify lease ownership and expiry:

```sql
BEGIN;

UPDATE seats
SET status = 'sold',
    sold_to = :user_id,
    reserved_by = NULL,
    reserved_until = NULL
WHERE seat_id = :seat_id
  AND status = 'available'
  AND reserved_by = :user_id
  AND reserved_until >= NOW();

-- Continue only if one row changed.

INSERT INTO tickets (...)
VALUES (...);

COMMIT;
```

## Redis Lease

A common acquisition primitive is conceptually:

```text
SET reservation:seat:A15 unique_token NX EX 600
```

Properties:

- `NX`: create only if the key does not exist.
- `EX 600`: expire automatically after 600 seconds.
- `unique_token`: identifies the lease owner.

Release must verify ownership before deletion. A process must not delete a lock that expired and was subsequently acquired by someone else.

Conceptual atomic release:

```text
if current_value == my_unique_token:
    delete key
```

Redis Lua scripts are often used to make the comparison and deletion atomic.

## ZooKeeper or etcd

Coordination stores can provide:

- Strongly consistent metadata
- Sessions or leases
- Ephemeral ownership records
- Consensus-backed updates
- Watch mechanisms

They are appropriate when coordination correctness justifies operating specialized infrastructure.

## When to Use

Use a lease or distributed lock when exclusivity must span:

- User think time
- A call to another service
- Several application requests
- Multiple stateless servers
- Work that cannot fit inside one short database transaction

---

# 18. Lease Expiry and Fencing Tokens

## The Expired-Holder Problem

Suppose Worker A acquires a lease for 10 seconds.

```text
1. Worker A acquires lease.
2. Worker A pauses for 20 seconds.
3. Lease expires.
4. Worker B acquires a new lease.
5. Worker A resumes and still believes it owns the resource.
```

Now both may perform work.

A TTL alone does not prevent the old holder from acting after expiry.

## Fencing Token

Every successful lease acquisition receives a monotonically increasing token:

```text
Worker A receives token 41.
Worker B later receives token 42.
```

The protected storage system rejects operations with an older token:

```text
accept operation only if fencing_token >= latest_seen_token
```

Worker A's stale token 41 is rejected after token 42 has been observed.

### Fencing Invariant

> **A newer lease holder must be able to prevent an expired former holder from mutating the protected resource.**

For soft UX reservations, a simple TTL may be sufficient. For correctness-critical exclusive work, ownership tokens and fencing should be considered.

---

# 19. Distributed Lock Is Not the Source of Truth

A lock coordinates access. It should not replace authoritative business state.

For a ticket purchase:

```text
Lease service:
Who currently has a temporary opportunity to buy?

Database:
Who actually owns the sold seat?
```

The final database write should still use a conditional check:

```sql
UPDATE seats
SET status = 'sold', sold_to = :user_id
WHERE seat_id = :seat_id
  AND status = 'available'
  AND reserved_by = :user_id
  AND reserved_until >= NOW();
```

### Source-of-Truth Invariant

> **The authoritative datastore must enforce the final ownership transition, even when an external lock or lease reduces contention.**

---

# 20. Queue-Based Serialization

## The Hot-Resource Problem

If millions of requests target one resource, horizontal application scaling does not remove the contention.

```text
Many web servers
    ↓
One hot database row
```

All requests eventually serialize at the authoritative row anyway.

OCC may become particularly wasteful:

```text
Many requests read the same version
One succeeds
Thousands fail and retry
```

## Queue Strategy

Route all commands for the same resource to one ordered queue partition:

```text
Clients
    ↓
API servers
    ↓
Queue partition keyed by resource_id
    ↓
Single logical consumer for that partition
    ↓
Database
```

Example partition key:

```text
auction_id
concert_id
inventory_sku
account_id
```

Operations for the same key are processed sequentially. Different keys can be processed in parallel across partitions.

## Advantages

- Eliminates concurrent writers for one partition
- Absorbs bursts
- Creates a clear order
- Protects the database from retry storms
- Enables admission control

## Disadvantages

- Adds queue latency
- Caps throughput for one hot key
- Requires durable processing and recovery
- Consumer failure must be handled
- Clients may need asynchronous status APIs
- Exactly-once effects still require idempotency

## Idempotent Consumer Pattern

Every request carries an idempotency key:

```text
purchase_request_id = req-123
```

Before applying the purchase, the worker checks whether that request was already processed.

### Queue Principle

> **A queue turns uncontrolled concurrent contention into controlled sequential work, but it cannot make one resource process infinitely fast.**

---

# 21. Redis Lua for Atomic In-Memory Transitions

Redis executes a Lua script atomically with respect to other commands on the same Redis server execution path.

A script can:

1. Read inventory.
2. Validate availability.
3. Decrement inventory.
4. Record a reservation.
5. Return success or failure.

Conceptual script:

```lua
local stock = tonumber(redis.call('GET', KEYS[1]) or '0')

if stock <= 0 then
  return 0
end

redis.call('DECR', KEYS[1])
redis.call('SET', KEYS[2], ARGV[1], 'EX', ARGV[2])
return 1
```

## Advantages

- Very low latency
- No read-modify-write gap inside the script
- Useful for rate limits, inventory admission, and reservations

## Important Caveat

If Redis is not the authoritative durable store, the system must reconcile the in-memory decision with the database.

Potential problems include:

- Redis accepts a decrement but database persistence fails.
- Asynchronous consumers process an event twice.
- Redis state is lost or restored from an older snapshot.
- Cache and database disagree.

Possible supporting patterns:

- Durable event log
- Idempotency keys
- Transactional outbox
- Reservation records in the source-of-truth database
- Reconciliation jobs

Use Redis as the authority only when its durability, replication, and failure semantics satisfy the business requirement.

---

# 22. Partitioning the Contention

Sometimes one logical counter can be divided into several independent buckets.

Instead of:

```text
inventory_total = 10,000
```

use:

```text
bucket 0 = 1,000
bucket 1 = 1,000
...
bucket 9 = 1,000
```

Requests are distributed across buckets.

## Benefit

Ten buckets can reduce write contention on one row because unrelated claims update different rows.

## Trade-Offs

- One bucket may empty while others still have inventory.
- Routing and rebalancing become more complex.
- Computing exact global availability requires aggregation.
- A request may need to try another bucket.
- The business resource must be safely divisible.

## Good Fits

- Fungible inventory
- Rate-limit counters
- Like or view counters
- Allocatable capacity without per-unit identity

## Poor Fits

- One exact seat
- One auction item
- One specific username
- One unique driver assignment

### Partitioning Principle

> **Shard contention only when the resource is genuinely divisible without weakening the invariant.**

---

# 23. Unique Constraints as a Correctness Backstop

Database constraints can enforce invariants even if application code has a bug.

## Unique Seat Ownership

```sql
CREATE UNIQUE INDEX one_ticket_per_seat
ON tickets(concert_id, seat_number);
```

Two ticket inserts for the same seat cannot both commit.

## Idempotency Key

```sql
CREATE UNIQUE INDEX unique_purchase_request
ON purchases(idempotency_key);
```

Repeated processing of the same request does not create duplicate purchases.

## Constraint Role

A constraint may not provide ideal user experience by itself, but it is an excellent final correctness boundary.

```text
Application coordination prevents most conflicts.
Database constraint rejects any conflict that escapes.
```

---

# 24. Idempotency and Double Charging

Contention and retries often create duplicate requests.

A client may retry because it did not receive a response, even though the server completed the charge.

Use an idempotency key:

```http
POST /payments
Idempotency-Key: purchase-123
```

The server stores the result under that key:

```text
purchase-123 → payment succeeded, payment_id = p789
```

A repeated request returns the original result rather than charging again.

### Idempotency Invariant

> **Repeating the same logical command must not create an additional business effect.**

Concurrency control answers:

> Can two different commands claim the same resource?

Idempotency answers:

> Can the same command accidentally execute more than once?

Correct systems frequently need both.

---

# 25. Auctions

An auction is a good OCC example because the high bid normally moves in one direction.

```sql
UPDATE auctions
SET high_bid = :new_bid,
    high_bidder_id = :bidder_id,
    version = version + 1
WHERE auction_id = :auction_id
  AND version = :expected_version
  AND high_bid < :new_bid
  AND ends_at > NOW();
```

Interpretation:

```text
1 row affected → bid accepted
0 rows affected → stale version, bid too low, or auction ended
```

For very hot auctions, commands may be routed through one ordered partition by `auction_id`.

Important concerns:

- Per-auction ordering
- Server-authoritative timestamps
- Idempotent bid IDs
- Anti-sniping extension rules
- Durable bid history
- Reconnection and result notification

---

# 26. Event Booking

A robust seat-booking flow separates temporary reservation from final ownership.

## Selection

```text
Conditional lease acquisition
```

## Checkout

```text
User enters payment while the lease exists
```

## Payment

```text
Use an idempotency key with the payment provider
```

## Finalization

```text
Atomically verify lease ownership and mark seat sold
```

## Failure

```text
Payment fails → release lease or let it expire
Finalization conflict → do not issue ticket; compensate payment if necessary
```

A unique constraint on the seat identity prevents duplicate sold tickets.

---

# 27. Banking and Transfers

Within one relational database, a transfer can lock both accounts in a consistent order.

```sql
BEGIN;

SELECT account_id, balance
FROM accounts
WHERE account_id IN (:source_id, :destination_id)
ORDER BY account_id
FOR UPDATE;

-- Validate source balance.

UPDATE accounts
SET balance = balance - :amount
WHERE account_id = :source_id;

UPDATE accounts
SET balance = balance + :amount
WHERE account_id = :destination_id;

INSERT INTO transfers (...)
VALUES (...);

COMMIT;
```

If the operation spans independent databases or services, it is no longer only a local contention problem. It becomes a multi-step distributed consistency problem and may require sagas, transactional messaging, or a distributed transaction protocol.

---

# 28. Driver Dispatch

A driver can be temporarily claimed with a conditional state transition:

```sql
UPDATE drivers
SET dispatch_status = 'pending_request',
    pending_ride_id = :ride_id,
    reservation_expires_at = NOW() + INTERVAL '10 seconds'
WHERE driver_id = :driver_id
  AND online = TRUE
  AND (
      dispatch_status = 'available'
      OR reservation_expires_at < NOW()
  );
```

If one row changes, the request owns the temporary opportunity to contact that driver.

If zero rows change, choose another driver.

The final acceptance should verify the same ride and unexpired reservation.

---

# 29. Flash Sales

A flash sale may combine several mechanisms:

```text
Admission control
    ↓
Queue requests by SKU
    ↓
Atomic inventory claim
    ↓
Temporary checkout reservation
    ↓
Idempotent payment
    ↓
Final order persistence
```

Possible tools:

- Queue-based serialization for a very hot SKU
- Conditional database decrement
- Redis Lua for fast admission
- Reservation leases for checkout
- Unique idempotency key for orders
- Reconciliation against durable inventory

Do not assume one mechanism solves every stage.

---

# 30. Hot Partitions and Extreme Contention

Sharding does not help when every request targets the same key.

```text
Many application nodes
Many database shards
One popular item
    ↓
One shard and one row remain hot
```

Possible responses:

1. Change the product semantics.
2. Make the resource divisible.
3. Use eventual consistency where acceptable.
4. Introduce admission control.
5. Serialize operations by key.
6. Batch compatible updates.
7. Reject overload quickly.
8. Use a waiting room or lottery.

The physical maximum throughput of one strictly ordered resource remains bounded.

### Hot-Key Principle

> **Strong consistency for one indivisible resource creates a serialization point somewhere, whether visible or hidden.**

---

# 31. Selection Guide

## Use a Conditional Write When

- The condition is a predicate on the row or key being changed.
- One atomic statement can express the transition.

Examples:

```text
available → sold
stock > 0 → stock - 1
unassigned → assigned
```

## Use Pessimistic Locking When

- The application must read, decide, then write.
- The decision cannot be represented as one simple write predicate.
- Contention is frequent or retries are expensive.

## Use Optimistic Concurrency When

- The application must read, decide, then write.
- Conflicts are infrequent.
- Retrying is safe and inexpensive.

## Use Serializable Isolation When

- The invariant spans multiple rows.
- Transactions can violate it while modifying different rows.
- Materializing the conflict onto one row is impractical.

## Use a Lease or Distributed Lock When

- Exclusivity must outlive one short transaction.
- A user or process needs a temporary reservation.
- Coordination spans stateless servers or external work.

## Use Queue-Based Serialization When

- One resource is extremely hot.
- Retry storms or lock waits overwhelm the database.
- Asynchronous processing is acceptable.
- Per-resource ordering is valuable.

## Use Partitioned Counters When

- The resource is fungible and divisible.
- Exact per-request global count is unnecessary or can be aggregated.

---

# 32. Strategy Progression for Interviews

A strong answer can follow this progression:

## Step 1: Identify the Race

> A naive read-modify-write allows concurrent requests to decide from stale state, so it can oversell or double-book the resource.

## Step 2: Identify the Authoritative Resource

> I will model each seat as its own row because the exact seat, not only the total count, is the contended object.

## Step 3: Start With the Simplest Atomic Transition

> I will use a conditional update from `available` to `sold` and treat one affected row as success.

## Step 4: Add Transactional Side Effects

> Ticket creation occurs in the same transaction and only after the conditional update succeeds.

## Step 5: Address User Think Time

> During checkout, I will use a short reservation lease rather than holding a database row lock.

## Step 6: Address Extreme Scale

> If one event becomes a hot key, I will add admission control and route commands through an ordered queue partition by event or seat group.

## Step 7: Address Failures

> I will use idempotency keys, bounded retries, lease ownership checks, database constraints, and reconciliation.

---

# 33. Common Mistakes

## Mistake 1: Read Then Write Without a Guard

Unsafe:

```text
SELECT value
if value is valid:
    UPDATE later
```

Better:

```text
Conditional write, version check, or explicit lock
```

## Mistake 2: Ignore Affected-Row Count

An update that matches zero rows is often a successful SQL statement.

The application must not perform dependent work afterward.

## Mistake 3: Guard the Wrong Resource

A total seat counter does not prevent duplicate ownership of a specific seat.

## Mistake 4: Hold Locks Across External Calls

Do not keep a row lock while waiting for payment, email, or user input.

## Mistake 5: Use Distributed Locks by Default

A local database transaction is usually simpler and safer when all authoritative data is in one database.

## Mistake 6: Retry Without Idempotency

A retry can duplicate charges, orders, or messages.

## Mistake 7: Retry Immediately Under Heavy Contention

Immediate retries amplify load. Use bounded backoff, jitter, admission control, or serialization.

## Mistake 8: Ignore Deadlocks

Even carefully ordered code should handle database deadlock errors as retryable.

## Mistake 9: Treat a TTL as Perfect Exclusivity

A paused lease holder may resume after expiry. Use ownership tokens and fencing when stale holders would be dangerous.

## Mistake 10: Assume Sharding Fixes a Hot Key

One indivisible resource still has one serialization point.

---

# 34. Operational Metrics

Monitor:

- Conditional-update failure rate
- OCC conflict rate
- Retry count
- Retry success rate
- Lock wait duration
- Number of blocked transactions
- Deadlock count
- Transaction age
- Serialization-failure rate
- Lease acquisition failures
- Expired reservations
- Queue depth by partition
- Queue processing latency
- Hot-key frequency
- Duplicate idempotency-key requests
- Constraint violations
- Payment compensation rate

These metrics reveal whether the chosen strategy still matches the workload.

Example interpretation:

```text
OCC conflict rate rises sharply
    ↓
Retries increase load
    ↓
Move hot resource to pessimistic locking or ordered queue
```

---

# 35. Important Invariants

## Ownership Invariant

> **At most one committed owner exists for a unique resource.**

## Availability Invariant

> **Inventory never falls below zero.**

## Dependency Invariant

> **A ticket is created only if the corresponding seat claim succeeds.**

## Version Invariant

> **Every successful update advances the concurrency token.**

## Lease Invariant

> **Only the current unexpired lease owner may finalize the resource.**

## Idempotency Invariant

> **One logical request creates at most one business effect.**

## Ordering Invariant

> **Operations requiring order pass through one ordered authority for that resource.**

## Deadlock-Prevention Invariant

> **Shared resources are locked in one globally consistent order.**

## Backpressure Invariant

> **Overload cannot create unbounded retries, queues, or blocked transactions.**

---

# 36. Interview Answer Template

> The first risk is a lost update caused by a read-modify-write gap. I would model the exact contended resource as an authoritative row or key and begin with a conditional state transition, such as changing a seat from `available` to `sold`. The application treats one affected row as success and zero as a conflict, and it performs dependent writes in the same transaction only after the claim succeeds. If the decision requires application-side logic over several rows, I would use `SELECT ... FOR UPDATE` under high contention or optimistic concurrency with a version column when conflicts are rare. For cross-row invariants that do not share a writable row, I would either materialize the invariant onto one row or use serializable isolation with retry handling. If exclusivity must survive user think time or an external call, I would represent it as an expiring lease and verify lease ownership during finalization. Under extreme hot-key load, I would add admission control and serialize commands through a queue partition keyed by the resource. Unique constraints and idempotency keys remain final correctness backstops.

---

# 37. Quick Comparison

## Conditional Write

```text
Best for:
A simple state predicate on the row being written

Conflict behavior:
Zero rows affected or conditional-check failure

Main benefit:
One atomic statement

Main cost:
Limited to directly expressible conditions
```

## Pessimistic Locking

```text
Best for:
Read-decide-write with frequent conflicts

Conflict behavior:
Wait, timeout, or deadlock victim

Main benefit:
Predictable correctness under contention

Main cost:
Blocking and lower concurrency
```

## Optimistic Concurrency

```text
Best for:
Read-decide-write with rare conflicts

Conflict behavior:
Stale version updates zero rows

Main benefit:
No blocking during normal operation

Main cost:
Retries and discarded work
```

## Serializable Isolation

```text
Best for:
Cross-row invariants and write skew

Conflict behavior:
Serialization failure and retry

Main benefit:
Equivalent to serial execution

Main cost:
Conflict tracking, blocking, or aborts
```

## Lease or Distributed Lock

```text
Best for:
Exclusive access spanning requests, waits, or external calls

Conflict behavior:
Lease acquisition fails

Main benefit:
Shared temporary ownership

Main cost:
Expiry, stale holders, and coordination complexity
```

## Queue Serialization

```text
Best for:
Extremely hot resources and required per-key ordering

Conflict behavior:
Operations wait in queue rather than race

Main benefit:
Controlled sequential processing

Main cost:
Added latency and bounded per-key throughput
```

---

# 38. Thirty-Second Revision

- **Contention:** concurrent operations compete for one logical resource.
- **Lost update:** concurrent read-modify-write operations act on stale state.
- **First step:** model the actual contended resource as one authoritative row or key.
- **Conditional write:** put the validity rule in the write predicate.
- **Affected rows:** zero is usually not an exception; application logic must branch.
- **Pessimistic locking:** lock before a complex read-decide-write operation.
- **OCC:** compare a version at write time and retry on conflict.
- **ABA:** avoid reusable business values as versions; prefer a monotonic counter.
- **Write skew:** concurrent transactions modify different rows but break a shared invariant.
- **Serializable:** use when correctness requires serial behavior across rows.
- **Lease:** temporary ownership that outlives one transaction.
- **Fencing token:** prevents an expired holder from acting after a newer holder exists.
- **Queue serialization:** process one hot key sequentially when retries or lock waits collapse throughput.
- **Partitioning:** helps only when the resource is genuinely divisible.
- **Unique constraint:** final database backstop against duplicate ownership.
- **Idempotency key:** prevents retries from duplicating business effects.
- **Best default:** keep correctness in one authoritative database and choose the simplest mechanism that fits.

## Final Mental Model

```text
Identify the exact resource
    ↓
Put it in one authoritative location
    ↓
Can one conditional write enforce the rule?
    ├── Yes → use it
    └── No
        ↓
Does application logic need read-decide-write?
    ├── High contention → pessimistic lock
    └── Low contention → optimistic version check
        ↓
Does the invariant span non-conflicting rows?
    ├── Materialize one shared row, or
    └── use serializable isolation
        ↓
Must ownership survive a transaction boundary?
    └── use an expiring lease with ownership validation
        ↓
Is one key overwhelmed?
    └── admission control plus ordered queue processing
```
