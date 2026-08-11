# Handling Large Files and Blob Storage: System Design Study Guide

## Core Thesis

**Large binary objects should usually travel directly between clients and specialized object storage, while application servers authorize, coordinate, validate, and track the transfer.**

Typical large objects include:

- Videos
- High-resolution images
- Documents
- Backups
- Archives
- Audio recordings
- Machine-learning datasets
- Game assets

The preferred architecture is:

```text
Control plane:
Client ↔ Application API ↔ Metadata database

Data plane:
Client ↔ Object storage
Client ↔ CDN ↔ Object storage
```

The application server remains responsible for:

- Authentication
- Authorization
- Quotas
- Object-key allocation
- Temporary credential generation
- Metadata
- Processing state
- Security policy
- Lifecycle and deletion

The storage and CDN layers handle the bytes.

> **Separate the control plane from the data plane. Let applications decide who may transfer which object, but let storage infrastructure move the object itself.**

---

# 1. Why Large Files Need Special Handling

A traditional upload path may proxy every byte through the application tier:

```text
Client
  ↓ 2 GB upload
API gateway
  ↓ 2 GB stream
Application server
  ↓ 2 GB stream
Object storage
```

The server adds little value while consuming:

- Inbound bandwidth
- Outbound bandwidth
- Memory buffers
- Connections
- CPU for copying and encryption
- Load-balancer capacity
- Autoscaling capacity
- Timeout budgets

Downloads through the application tier create the same waste in reverse.

## Amplification

For a 2 GB upload proxied by the application:

```text
2 GB enters the application tier
2 GB leaves the application tier
```

Ignoring protocol overhead, the application network processes approximately 4 GB for one stored object.

Direct upload reduces this to control requests plus one client-to-storage transfer.

---

# 2. Why Object Storage Instead of a Database

Relational databases are optimized for:

- Structured rows
- Transactions
- Indexes
- Joins
- Constraints
- Selective queries

Large blobs can make databases more difficult to operate by increasing:

- Backup size
- Restore time
- Replication bandwidth
- Buffer-pool pressure
- WAL volume
- Table bloat
- Query latency

Object storage is optimized for:

- Very large capacity
- Durable object replication
- Multipart transfers
- Lifecycle policies
- Tiered storage
- Range reads
- CDN integration
- Per-object access controls

## Metadata Still Belongs in a Database

Store queryable metadata separately:

```text
file_id
owner_id
filename
storage_key
content_type
size_bytes
status
checksum
created_at
```

Store only the immutable bytes in object storage.

## Practical Heuristic

Large-file architecture becomes increasingly valuable as file size, transfer duration, and concurrency grow. A threshold such as 10 MB can be a useful interview heuristic, but it is not a protocol law. Decide from:

- File-size distribution
- Upload frequency
- Network cost
- Server limits
- Need for resumability
- Validation and compliance requirements

---

# 3. Control Plane and Data Plane

## Control Plane

The application API handles small requests:

```text
Create upload session
Authorize download
Report status
Abort upload
Delete file
Request processing
```

## Data Plane

The data plane handles large byte streams:

```text
Upload chunks
Download ranges
Serve video segments
Deliver images from edge cache
```

## Benefits of Separation

- Application servers scale with requests, not object bytes.
- Transfer failures do not occupy application workers.
- Storage SDKs provide retries and multipart support.
- CDN edges serve geographically distributed users.
- Application deployments do not interrupt active transfers.

### Architecture Invariant

> **Application services control access and state transitions, but are not bandwidth proxies unless inspection or transformation requires it.**

---

# 4. Basic Direct Upload Flow

```text
1. Client sends intended filename, size, and media type to API.
2. API authenticates and authorizes user.
3. API checks quota and policy.
4. API allocates file ID and storage key.
5. API inserts metadata row with PENDING status.
6. API returns a temporary upload capability.
7. Client uploads directly to object storage.
8. Storage event or verification confirms object.
9. Backend marks upload complete or starts processing.
```

## Metadata Record

```sql
CREATE TABLE files (
    file_id UUID PRIMARY KEY,
    owner_id UUID NOT NULL,
    original_filename TEXT NOT NULL,
    storage_key TEXT NOT NULL UNIQUE,
    declared_size_bytes BIGINT NOT NULL,
    actual_size_bytes BIGINT,
    declared_content_type TEXT,
    detected_content_type TEXT,
    checksum TEXT,
    status TEXT NOT NULL,
    upload_session_id TEXT,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

Possible statuses:

```text
PENDING
UPLOADING
UPLOADED
VERIFYING
QUARANTINED
PROCESSING
AVAILABLE
REJECTED
FAILED
DELETED
```

---

# 5. Presigned Upload URLs

A presigned URL is a temporary capability granting permission for a specific storage operation.

It usually binds some combination of:

- HTTP method
- Bucket or container
- Object key
- Expiration time
- Required headers
- Content length or range policy
- Content type
- Checksums

Conceptual URL:

```text
https://storage.example.com/private/uploads/owner-7/file-842
    ?expires=...
    &credential=...
    &signedHeaders=...
    &signature=...
```

The storage service validates the signature and policy without asking the application server for every byte.

## Capability Security Model

Anyone possessing a bearer-style presigned URL may be able to use it until expiry, subject to its restrictions.

Therefore:

- Use short expirations.
- Restrict the exact object key.
- Restrict the HTTP method.
- Restrict size when supported.
- Require expected headers.
- Never log URLs carelessly.
- Use TLS.
- Do not place secrets in predictable public locations.

## Presigned PUT Versus Form POST

A simple upload may use an HTTP PUT to one key.

Policy-based form uploads can express conditions such as:

```text
maximum content length
required key prefix
required media type
required metadata headers
```

The best choice depends on client and provider capabilities.

---

# 6. Object-Key Design

The server should allocate storage keys rather than trust a client-provided path.

Good key:

```text
quarantine/tenant-17/2026/08/uuid-842
```

Avoid:

```text
../../other-user/private-file
```

or user-controlled names that can overwrite existing objects.

## Key Properties

- Globally unique
- Not dependent on display filename
- Does not expose sensitive data unnecessarily
- Encodes useful lifecycle or tenant prefixes when appropriate
- Supports authorization and cleanup
- Immutable after allocation where possible

Store the original filename only as metadata.

### Key Invariant

> **A client may choose file content and display name, but the server chooses the authoritative storage location.**

---

# 7. Upload Restrictions and Their Limits

Presigned policies can constrain:

- Maximum size
- Minimum size
- Declared content type
- Required checksum
- Object key
- Expiry

These controls reduce abuse but do not prove that the uploaded bytes are safe.

A declared header such as:

```http
Content-Type: image/jpeg
```

is an assertion from the uploader, not trusted content analysis.

The backend should independently validate:

- Magic bytes
- Actual format
- File size
- Checksum
- Archive expansion ratio
- Malware
- Media decodability
- Business-specific structure

---

# 8. Direct Downloads

For private objects:

```text
1. Client asks API for download permission.
2. API authenticates and authorizes access.
3. API returns short-lived signed storage or CDN URL.
4. Client downloads directly.
```

## Direct Object-Storage Download

Best for:

- Infrequent access
- Internal tools
- Region-local clients
- Objects with low reuse

## CDN Download

Best for:

- Public or broadly shared data
- Globally distributed users
- Frequently accessed objects
- Images, video, and software assets

The CDN fetches from object storage on a cache miss and serves later requests from edge locations.

---

# 9. Storage Signatures Versus CDN Signatures

These grant access at different layers.

## Object-Storage Signed URL

Validated by the storage service.

Purpose:

```text
Allow one client to read or write one storage object temporarily.
```

## CDN Signed URL or Cookie

Validated by CDN edge servers.

Purpose:

```text
Allow edge delivery of protected content under a policy.
```

CDN policies may cover:

- One URL
- A path prefix
- An expiration
- An IP restriction
- Several files through a signed cookie

## Origin Protection

If private content is delivered through a CDN, prevent clients from bypassing the CDN and fetching the origin object directly.

Use:

- Private bucket or container
- CDN origin identity or origin access mechanism
- Storage policy allowing only the CDN
- No public object ACLs

### Origin Invariant

> **Protected CDN content must not remain anonymously reachable through the storage origin.**

---

# 10. Why Single-Request Uploads Fail for Large Files

A long upload is exposed to:

- Wi-Fi loss
- Mobile network changes
- Browser restart
- Device sleep
- Proxy timeout
- DNS failure
- Temporary storage error

If one request represents the whole file, a failure near completion can require retransmitting everything.

Chunked or multipart uploads limit retry cost to failed pieces.

---

# 11. Multipart and Resumable Uploads

Generic multipart flow:

```text
Initialize session
    ↓
Upload numbered parts
    ↓
Record provider receipts
    ↓
List or verify uploaded parts
    ↓
Complete session
    ↓
Provider assembles final object
```

Provider terminology differs, but the concepts are stable.

## Chunk Size Trade-Off

Small chunks:

- Fine-grained retry
- Better parallelism
- More requests and metadata

Large chunks:

- Fewer requests
- Less metadata
- More retransmission after failure

Choose from:

- File size
- Provider limits
- Network stability
- Client memory
- Desired parallelism
- Per-request cost

---

# 12. Upload ID, Part Number, ETag, and Fingerprint

These identifiers belong to different layers.

## Upload ID

The provider's identifier for one multipart session.

```text
upload_id = provider-session-xyz
```

Every part request refers to that session.

It answers:

> Which unfinished object assembly does this part belong to?

## Part Number

An integer indicating a part's position:

```text
1, 2, 3, ... N
```

It answers:

> Where does this part belong in the final object?

## ETag or Part Receipt

A provider-returned identifier for an uploaded part.

```text
part 7 → ETag "abc123"
```

The completion request may need the ordered list of part numbers and receipts.

### Important Checksum Nuance

Do not universally equate an ETag with a plain MD5 checksum. Its meaning can vary with provider, encryption mode, multipart behavior, and implementation. Treat it as a provider receipt unless the provider contract explicitly defines checksum semantics.

Use explicit checksum features when end-to-end integrity matters.

## Client Fingerprint

A client-generated identifier for recognizing the same local file later.

Possible inputs:

```text
filename
size
last-modified timestamp
relative path
optional sample hash
```

It answers:

> Is this local file probably the file associated with a previous resumable session?

A fingerprint is not a provider session ID and may collide. It maps a local file candidate to stored upload-session metadata.

---

# 13. Complete Multipart Execution Flow

## Phase 1: Initialization

```text
1. Client computes local fingerprint.
2. Client requests an upload session.
3. API validates ownership, quota, filename, and declared size.
4. API creates metadata row.
5. Backend initializes provider multipart session.
6. Backend stores upload ID.
7. Backend returns part size and upload authorization strategy.
```

## Phase 2: Part Upload

For each part:

```text
1. Client slices byte range.
2. Client obtains or reuses authorization for that part.
3. Client uploads part number + upload ID + bytes.
4. Storage validates and stores part.
5. Storage returns ETag or checksum receipt.
6. Client records completion locally.
7. Backend may optionally record part progress.
```

## Phase 3: Resume

```text
1. Client restarts and recomputes fingerprint.
2. Client retrieves active session metadata.
3. Backend validates that the session still belongs to user.
4. Backend or client lists parts from provider.
5. Local progress is reconciled with provider truth.
6. Missing parts are uploaded.
```

## Phase 4: Completion

```text
1. Client or backend sends ordered part-number and receipt list.
2. Provider validates parts.
3. Provider assembles the final object.
4. Provider returns object metadata.
5. Storage event announces final-object creation.
6. Backend verifies size and checksum.
7. File moves to processing or available state.
```

## Phase 5: Cleanup

```text
1. Abort abandoned sessions.
2. Delete stale parts through lifecycle rules.
3. Mark expired metadata rows.
4. Release reserved quota.
```

---

# 14. Who Tracks Multipart Progress?

## Client-Only Progress

The browser or application stores:

```text
upload ID
fingerprint
part size
completed part numbers
ETags
```

Advantages:

- Less backend traffic
- Natural progress UI

Risks:

- Local state may be lost.
- Another device cannot resume easily.
- Client state may disagree with provider.

## Backend Progress

The backend records part status.

Advantages:

- Cross-device resume
- Operational visibility
- Central policy enforcement

Costs:

- Many metadata writes
- More backend complexity
- State can still lag provider

## Provider as Final Authority

Whatever local state exists, list uploaded parts from the provider before final completion when correctness requires it.

### Progress Invariant

> **Client and database progress are hints; the storage provider's accepted parts determine what is actually resumable.**

---

# 15. Parallel Uploads

Clients may upload several parts concurrently.

Benefits:

- Better utilization of high-bandwidth connections
- Reduced effect of one slow request
- Independent retries

Costs:

- More memory
- More battery use
- More provider requests
- More radio and network contention
- Possible rate limits

Use adaptive concurrency:

```text
Stable fast network → increase parallelism
Mobile or error-prone network → reduce parallelism
```

Do not assume maximum parallelism gives maximum throughput.

---

# 16. Upload Completion Must Be Idempotent

The completion call may be retried because the client does not know whether the first request succeeded.

Use one logical operation ID and a state machine:

```text
UPLOADING → COMPLETING → UPLOADED
```

If the provider already assembled the object, a retry should return or reconstruct the existing result rather than create conflicting metadata.

The backend should handle:

- Duplicate completion calls
- Duplicate storage events
- Completion response lost after success
- Client completion and event arriving in either order

### Completion Invariant

> **Repeating completion must converge on one final object and one logical metadata result.**

---

# 17. Metadata and Object State Are Separate

Object storage and the metadata database do not share one transaction.

Possible inconsistent states:

```text
Metadata exists, object absent
Object exists, metadata absent
Metadata says complete, object incomplete
Object complete, metadata still pending
Object deleted, metadata still available
```

The design must treat these as normal recoverable states rather than impossible bugs.

---

# 18. Do Not Trust Client Completion Alone

A client may call:

```http
POST /files/{file_id}/complete
```

But the client may be:

- Incorrect
- Malicious
- Retrying
- Disconnected before upload completion
- Reporting the wrong object

Use the client completion call as a hint to begin verification, not unquestioned proof.

Verify with storage:

- Object exists
- Key is expected
- Size is within bounds
- Checksum matches
- Upload session belongs to file ID
- Required metadata is present

---

# 19. Storage Event Notifications

Object storage can emit events when an object is created or multipart completion succeeds.

```text
Object storage
    ↓ event
Queue or event bus
    ↓
Verification worker
    ↓
Metadata database
```

## Event Handler

```text
1. Deduplicate event ID.
2. Parse bucket and object key.
3. Find metadata row.
4. Inspect object metadata.
5. Validate size and checksum.
6. Mark UPLOADED or VERIFYING.
7. Trigger processing workflow.
```

## Delivery Semantics

Storage events may be:

- Delayed
- Duplicated
- Delivered out of order
- Retried

Design the handler to be idempotent.

The exact delivery guarantees depend on the provider and event path, so do not assume a notification alone proves globally exactly-once execution.

---

# 20. Reconciliation

Events are the fast path. Reconciliation is the correctness safety net.

A periodic job finds suspicious metadata:

```sql
SELECT *
FROM files
WHERE status IN ('PENDING', 'UPLOADING', 'COMPLETING')
  AND updated_at < NOW() - INTERVAL '1 hour';
```

For each record:

```text
1. Query object or multipart session.
2. If final object exists, verify and advance state.
3. If session is active, keep or expire according to policy.
4. If nothing exists, mark failed and release quota.
```

A reverse reconciliation job can inspect storage prefixes and find objects with no metadata row.

### Reconciliation Invariant

> **Every metadata-object mismatch has a deterministic repair or cleanup path.**

---

# 21. Orphan Objects and Missing Objects

## Orphan Object

```text
Object exists in storage
Metadata row missing or permanently pending
```

Causes:

- Client uploaded but never notified API.
- Metadata transaction failed.
- Event processing failed.
- Manual copy bypassed application.

Repair:

- Recover from embedded file ID or key mapping.
- Create quarantine metadata.
- Delete after retention period.

## Missing Object

```text
Metadata says available
Object does not exist
```

Causes:

- Accidental deletion
- Incorrect lifecycle policy
- Failed copy
- Cross-region replication delay or error

Repair:

- Restore from replica or version history.
- Mark unavailable.
- Alert and notify owner.

---

# 22. Quarantine Pipeline

Do not make untrusted uploads immediately downloadable.

```text
Client upload
    ↓
Private quarantine storage
    ↓
Verification and scanning
    ↓
Transcoding or transformation
    ↓
Approved output storage
    ↓
CDN
```

Checks may include:

- Malware scanning
- MIME detection
- Image or media decoding
- Content moderation
- Archive-bomb detection
- Document sanitization
- Metadata stripping
- Policy and copyright checks

## Status Flow

```text
UPLOADED
    ↓
QUARANTINED
    ↓
PROCESSING
    ├── AVAILABLE
    └── REJECTED
```

### Publication Invariant

> **Untrusted bytes are never exposed through the public delivery path before required validation succeeds.**

---

# 23. Abuse Prevention

## Quotas

Enforce:

- Maximum file size
- Total storage per user or tenant
- Upload requests per minute
- Concurrent multipart sessions
- Daily byte quota

Reserve quota when creating the upload session and reconcile it after completion or expiry.

## Storage-Cost Abuse

Prevent:

- Oversized uploads
- Unlimited abandoned parts
- High request-count attacks
- Repeated failed transcoding
- Public hotlinking

## Content Abuse

Use:

- Quarantine
- Validation
- Moderation
- Malware scanning
- Access revocation
- Audit logging

## Signed URL Leakage

Mitigate using:

- Short TTL
- Least privilege
- HTTPS
- Redacted logs
- Single-object scope
- CDN cookies for multi-object sessions

---

# 24. File Integrity

Integrity checks answer whether the final object contains the intended bytes.

Possible checksums:

- MD5 where appropriate
- SHA-256
- CRC variants
- Provider-specific multipart checksum

## End-to-End Flow

```text
1. Client computes checksum, possibly incrementally.
2. Upload request includes checksum when provider supports it.
3. Storage validates each part or final object.
4. Backend records algorithm and digest.
5. Processing workers verify before use when needed.
```

For untrusted clients, a client checksum detects transfer corruption but does not prove benign content.

## Multipart Nuance

The checksum of the assembled object may not equal a simple hash of concatenated part hashes. Follow the chosen provider and algorithm contract.

---

# 25. Content-Addressed Storage and Deduplication

A system may derive an object identity from a cryptographic content hash:

```text
sha256:ab12...
```

Benefits:

- Deduplication
- Integrity verification
- Immutable references
- Efficient synchronization

Challenges:

- Client hash cannot be trusted without server verification.
- Hashing large files consumes time and battery.
- Identical encrypted files may not have identical ciphertext.
- Existence checks can leak whether another user owns specific content.
- Ownership and deletion must be reference-counted carefully.

Deduplication is most valuable for file-sync or backup products, but it adds security and lifecycle complexity.

---

# 26. Processing Workflows

An upload often triggers asynchronous processing:

- Thumbnail creation
- Video transcoding
- Metadata extraction
- OCR
- Document conversion
- Virus scanning
- Audio waveform generation
- Search indexing

Architecture:

```text
Storage completion event
    ↓
Durable queue or workflow
    ↓
Processing workers
    ↓
Derived objects
    ↓
Metadata status update
```

## Idempotency

Derived output keys should be deterministic:

```text
processed/{file_id}/thumbnail-v3-320x320.jpg
```

Retries overwrite or reuse the same logical output rather than creating duplicates.

## Workflow State

Track:

```text
processing_version
attempt_count
last_error
output keys
started_at
completed_at
```

---

# 27. Video Processing and Adaptive Streaming

A video platform rarely serves the original upload directly.

Pipeline:

```text
Original upload
    ↓
Validate container and codecs
    ↓
Transcode multiple resolutions and bitrates
    ↓
Split into media segments
    ↓
Generate manifest
    ↓
Serve manifest and segments through CDN
```

Adaptive streaming lets the player switch quality based on bandwidth.

Store:

- Original object
- Transcoded variants
- Segment manifests
- Thumbnail images
- Media metadata

Processing may take minutes. The client should see explicit status and receive real-time or polling updates.

---

# 28. Fast Downloads With a CDN

A CDN reduces geographic latency and origin load.

```text
First regional request:
Edge miss → origin fetch → edge cache

Later regional requests:
Edge hit → client
```

Use cache-friendly immutable object names for processed public content:

```text
/media/file-842/variant-1080p/hash-abc/segment-17
```

When content changes, publish a new versioned key instead of relying only on purge.

## CDN Cache Key

Ensure the cache key contains every representation dimension:

- Object path
- Version
- Transformation
- Language if relevant
- Authorization policy where safely cacheable

Do not include per-user signed query parameters in a way that destroys shared cache hit rate unless the CDN can validate authorization separately from the cache key.

---

# 29. HTTP Range Requests

Clients can request part of an object:

```http
GET /downloads/archive.zip HTTP/1.1
Range: bytes=0-10485759
```

Successful partial response:

```http
206 Partial Content
Content-Range: bytes 0-10485759/5000000000
Accept-Ranges: bytes
```

Benefits:

- Resume interrupted downloads
- Seek within audio or video
- Fetch only required bytes
- Download in parallel when justified

The storage service and CDN must preserve range behavior.

## Entity Stability

A resumed range request must refer to the same object version. Use immutable keys, ETags, or conditional range requests so the client does not combine bytes from different versions.

---

# 30. Parallel Downloads

A client may request several byte ranges concurrently.

Potential benefits:

- Better utilization when one connection is throttled
- Independent retries

Costs:

- More requests
- More client memory and file assembly logic
- Greater CDN and origin load
- Limited benefit when total client bandwidth is the bottleneck

Use only after measuring ordinary CDN and range-download performance.

---

# 31. Progress Tracking

## Client Progress

For upload:

```text
bytes successfully uploaded / total bytes
```

For multipart:

```text
sum(completed part sizes + current in-flight bytes)
```

## Backend Progress

The backend may know only:

- Session created
- Provider events received
- Recorded completed parts
- Processing stages

Because direct bytes bypass the server, real-time transfer progress usually comes from the client or provider API rather than API-server observation.

## Processing Progress

After upload, track separately:

```text
upload progress
verification progress
transcoding progress
publication status
```

Do not present a completed transfer as a completed business workflow.

---

# 32. Cancellation and Abort

When a user cancels:

```text
1. Stop in-flight client requests.
2. Call multipart abort when possible.
3. Mark metadata ABORTED.
4. Release reserved quota.
5. Let lifecycle cleanup remove any leftovers.
```

Cancellation requests should be idempotent.

Race conditions are possible:

```text
Completion succeeds while cancellation is requested.
```

Use versioned state transitions or conditional updates to establish one terminal outcome.

---

# 33. Lifecycle Policies

Object storage lifecycle rules can:

- Abort incomplete multipart uploads
- Delete temporary quarantine objects
- Transition old objects to cheaper tiers
- Expire deleted-object versions
- Retain legal-hold objects
- Remove failed processing outputs

Example policy:

```text
Incomplete multipart parts → delete after 2 days
Rejected quarantine object → delete after 7 days
Available originals → archive after 90 days
Soft-deleted files → permanently delete after 30 days
```

Lifecycle rules are a safety net, not a substitute for correct application state transitions.

---

# 34. Deletion

Deleting a file may involve:

- Metadata
- Original object
- Derived variants
- CDN cache
- Search index
- Shared links
- Replicas
- Backups
- Audit and retention policy

## Soft Delete

```text
AVAILABLE → DELETION_PENDING → DELETED
```

Immediately revoke application access, then delete physical objects asynchronously.

## CDN Purge

Critical privacy deletion may require CDN invalidation. Purges take time, so short authorization lifetimes and private origin controls remain important.

## Versioning

Object versioning can support recovery but means a delete marker may not physically remove older bytes. Compliance deletion must address retained versions and backups according to policy.

---

# 35. Encryption

## In Transit

Use TLS for:

- API control requests
- Upload requests
- Download requests
- Storage events

## At Rest

Options include:

- Provider-managed keys
- Customer-managed keys
- Per-tenant keys
- Application-side encryption

## Client-Side Encryption

Benefits:

- Storage operator cannot read plaintext
- Strong privacy boundary

Costs:

- Server-side scanning and transcoding become difficult
- Deduplication changes
- Key recovery is critical
- Range and streaming behavior may be more complex

Key choice must match compliance, processing, and recovery requirements.

---

# 36. Authorization and Revocation

A presigned URL remains usable until expiry unless additional controls intervene.

For rapidly revocable access:

- Use very short-lived URLs.
- Authorize through a CDN signed cookie or token.
- Check entitlement before issuing each URL.
- Rotate object keys after serious leakage.
- Use an authorization-aware edge function where required.

Do not issue long-lived bearer links for sensitive data merely for convenience.

### Authorization Invariant

> **A temporary transfer capability grants only the minimum operation, object scope, and lifetime required.**

---

# 37. CORS and Browser Uploads

Browser clients uploading directly to storage require storage CORS configuration.

Allow only necessary:

- Origins
- Methods
- Request headers
- Exposed response headers such as ETag

Example concept:

```text
Allowed origin: https://app.example.com
Allowed methods: PUT, POST
Allowed headers: Content-Type, checksum headers
Exposed headers: ETag
```

Overly broad CORS does not by itself make a private bucket public, but it can expand browser-based abuse and should follow least privilege.

---

# 38. Mobile and Offline Resume

Mobile uploads face:

- Network switching
- Background execution limits
- Battery constraints
- App termination
- Limited local disk

Persist locally:

```text
file fingerprint
file ID
upload ID
part size
completed part receipts
session expiry
```

On restart:

```text
1. Reopen local file.
2. Verify fingerprint and permissions.
3. Refresh expired part authorizations.
4. List accepted provider parts.
5. Resume missing ranges.
```

If the local file changed, start a new upload rather than combining unrelated bytes.

---

# 39. Global Upload Architecture

For global users, consider:

- Nearest-region upload endpoint
- Provider transfer acceleration
- Multi-region object replication
- Data-residency rules
- Processing location
- Cross-region egress cost

Possible flow:

```text
User uploads to nearest allowed region
    ↓
Regional verification
    ↓
Asynchronous replication
    ↓
Global CDN delivery
```

Metadata must record object region and replication status.

Do not send regulated data to an arbitrary region merely for lower latency.

---

# 40. Multi-Tenant Isolation

Use:

- Tenant-scoped storage prefixes or buckets
- Tenant ID in metadata
- Server-generated keys
- Per-tenant quotas
- Encryption-key policies
- Authorization checks before every URL issuance

Never depend only on an unguessable object key as authorization.

A leaked key should not grant permanent public access.

---

# 41. Disaster Recovery

Plan for:

- Accidental deletion
- Regional storage outage
- Corrupt processing output
- Metadata database loss
- Key-management failure

Possible protections:

- Object versioning
- Cross-region replication
- Separate metadata backups
- Inventory reports
- Checksums
- Rebuildable derived outputs
- Documented restore procedure

The metadata database and object store must be recoverable together. Restoring one to a different point in time can create many mismatches requiring reconciliation.

---

# 42. Common Product Scenarios

## Video Platform

```text
Presigned multipart upload
    ↓
Quarantine
    ↓
Transcoding workflow
    ↓
Adaptive streaming segments
    ↓
CDN-signed delivery
```

Discuss:

- Multi-gigabyte resume
- Processing status
- Copyright or moderation scanning
- Immutable segment keys
- Range and streaming support

## Photo Sharing

```text
Direct original upload
    ↓
Virus and format validation
    ↓
Thumbnail and variant generation
    ↓
Public or private CDN delivery
```

Discuss:

- EXIF stripping
- Orientation
- Content moderation
- Hotlink protection
- Cache invalidation

## File Sync

Discuss:

- Content hashing
- Deduplication
- Chunk resume
- Version history
- Conflict handling
- Device synchronization
- Shared-link authorization

## Messaging Media

```text
Sender uploads encrypted or private media
    ↓
Chat stores media reference
    ↓
Recipients receive scoped download authorization
```

Discuss:

- Expiry
- Encryption
- Thumbnail privacy
- Retention
- Forwarding policy

## Document Processing

Discuss:

- Direct upload to quarantine
- OCR and preview generation
- Malware scanning
- Rich queryable metadata
- Long-running processing workflow

---

# 43. When Not to Use Direct Object Upload

## Small Payloads

Normal API endpoints may be simpler for small JSON, forms, and small media.

## Streaming Validation Before Acceptance

If the system must inspect bytes before they reach any storage, a controlled proxy or specialized scanning gateway may be required.

A common alternative is direct upload to isolated quarantine storage, followed by mandatory validation before publication.

## Transform During Ingress

Some systems need real-time transformation while receiving the stream. A dedicated media-ingest service may be appropriate.

## Compliance Boundary

If regulations require transfer through certified infrastructure, direct client-to-general storage may not satisfy the requirement.

## End-to-End Encrypted Content

Direct upload remains possible, but server-side scanning and processing may not be.

Choose architecture from actual security and product constraints rather than a fixed size threshold.

---

# 44. Interview Design Process

## Step 1: Characterize Files

Ask:

- Maximum and typical size
- Upload and download rate
- Public or private
- Required validation
- Retention
- Geographic distribution
- Processing requirements
- Resume requirements

## Step 2: Separate Metadata and Bytes

Define database metadata and object-key scheme.

## Step 3: Design Upload Authorization

Explain:

- Authentication
- Quota check
- Server-generated key
- URL scope
- Expiry
- Size and header restrictions

## Step 4: Add Multipart Resume

Define:

- Upload ID
- Chunk size
- Part numbers
- Provider receipts
- Client fingerprint
- Resume verification
- Completion

## Step 5: Synchronize State

Use:

- Storage events
- Idempotent handlers
- Storage verification
- Reconciliation

## Step 6: Add Security Pipeline

- Quarantine
- Malware scan
- Type validation
- Moderation
- Publication transition

## Step 7: Design Downloads

- CDN or direct storage
- Signed access
- Range requests
- Origin protection
- Cache policy

## Step 8: Plan Lifecycle

- Abort stale multipart sessions
- Delete rejected files
- Tier old files
- Handle account deletion

## Step 9: Discuss Failures

- Client crash
- Lost completion response
- Duplicate event
- Missing metadata
- Missing object
- Processing failure
- CDN stale content

---

# 45. Interview Answer Template

> I would separate the control plane from the data plane. The application API authenticates the user, checks quota, allocates a server-controlled object key, creates a pending metadata row, and returns a short-lived scoped upload capability. The client uploads directly to private object storage, using multipart or resumable upload for large files. The client tracks the provider upload ID, numbered parts, returned part receipts, and a local file fingerprint so it can list accepted parts and resume after failure. Completion is idempotent and does not trust the client alone. A storage event enters a durable queue, where a verification worker checks object existence, size, checksum, and ownership before advancing metadata. Untrusted files remain in quarantine until malware, format, and content checks complete. A reconciliation job repairs missing events, orphan objects, and stuck metadata. Downloads use origin-protected CDN delivery with short-lived signed authorization, immutable versioned keys, and HTTP range support. Lifecycle rules abort abandoned multipart sessions and tier or delete old data.

---

# 46. Common Mistakes

## Mistake 1: Proxy Every Byte Through API Servers

This wastes bandwidth and makes application capacity depend on file size.

## Mistake 2: Put Large Objects in the Relational Database

This increases backups, replication, and buffer pressure.

## Mistake 3: Let Clients Choose Arbitrary Storage Keys

This enables collisions, overwrites, and isolation failures.

## Mistake 4: Trust Declared Content Type

Validate actual format after upload.

## Mistake 5: Mark Complete Only From Client Notification

Verify with storage events or object inspection.

## Mistake 6: Assume ETag Always Means MD5

Use the provider's documented checksum mechanism.

## Mistake 7: Forget Incomplete Multipart Cleanup

Abandoned parts create ongoing storage cost.

## Mistake 8: Expose Quarantine Objects

Do not publish before security checks succeed.

## Mistake 9: Use Long-Lived Bearer URLs

Keep capabilities short-lived and narrowly scoped.

## Mistake 10: Make Origin Public Behind a Signed CDN

Users could bypass edge authorization.

## Mistake 11: Ignore Duplicate Events and Completion Calls

All handlers and transitions must be idempotent.

## Mistake 12: Store Rich Metadata Only as Object Tags

Use a queryable metadata database for application state.

## Mistake 13: Treat Upload Completion as Processing Completion

Transfer, verification, transformation, and publication are distinct states.

## Mistake 14: Omit Reconciliation

Separate systems inevitably diverge temporarily.

---

# 47. Operational Metrics

## Upload Control Plane

- Session creation rate
- Authorization failures
- Quota rejections
- Presigned URL issuance latency
- Active sessions

## Transfer

- Bytes uploaded per second
- Upload completion rate
- Part retry rate
- Average part size
- Upload duration percentiles
- Resumed-session rate
- Aborted-session rate

## Storage

- Object count
- Stored bytes
- Incomplete multipart bytes
- Lifecycle deletion volume
- Cross-region replication lag
- Storage error rate

## Synchronization

- Storage-event lag
- Duplicate-event rate
- Metadata rows stuck by state
- Orphan-object count
- Missing-object count
- Reconciliation repair count

## Processing

- Queue lag
- Scan duration
- Rejection rate
- Transcoding duration
- Processing retry rate
- Manual-review backlog

## Delivery

- CDN hit ratio
- Origin bytes
- Download throughput
- Range-request rate
- Signed URL issuance failures
- Regional latency

## Cost

- Storage by tier
- Request charges
- CDN egress
- Origin egress
- Duplicate or abandoned storage
- Processing compute

---

# 48. Important Invariants

## Ownership Invariant

> **Every object key maps to one authorized owner or tenant in metadata.**

## Transfer Invariant

> **A client may upload only to the server-allocated key and within the authorized policy.**

## Completion Invariant

> **A file becomes uploaded only after storage confirms a valid final object.**

## Publication Invariant

> **Untrusted bytes are never publicly downloadable before required validation.**

## Integrity Invariant

> **The final object satisfies the expected size and checksum policy.**

## Resume Invariant

> **Resumption uses the same local file, provider upload session, and authoritative accepted-part list.**

## Idempotency Invariant

> **Duplicate completion calls, events, and processing attempts converge on one logical file state.**

## Reconciliation Invariant

> **Every metadata-object mismatch can be detected and repaired or cleaned up.**

## Authorization Invariant

> **Temporary URLs grant minimum scope and lifetime, and private origins cannot be bypassed.**

## Lifecycle Invariant

> **Abandoned parts, rejected files, derivatives, and deleted content have explicit retention and cleanup rules.**

---

# 49. Quick Comparison

## API-Proxy Upload

```text
Best for:
Small files or mandatory inline inspection

Benefit:
Server sees every byte

Cost:
Bandwidth and scaling bottleneck
```

## Single Presigned Upload

```text
Best for:
Moderately sized files on reliable networks

Benefit:
Simple direct transfer

Cost:
Whole-file retry after failure
```

## Multipart or Resumable Upload

```text
Best for:
Large files and unreliable networks

Benefit:
Retry only missing chunks

Cost:
Session, part, and completion complexity
```

## Direct Storage Download

```text
Best for:
Low-reuse or region-local private objects

Benefit:
Simple delivery

Cost:
Higher latency for distant users
```

## CDN Delivery

```text
Best for:
Shared or global downloads

Benefit:
Edge latency and origin offload

Cost:
Authorization and invalidation complexity
```

## Storage Events

```text
Best for:
Fast metadata synchronization and processing triggers

Benefit:
Storage confirms actual object creation

Cost:
Duplicate, delayed, or missed event handling
```

## Reconciliation

```text
Best for:
Correcting drift between metadata and objects

Benefit:
Repairs missed-event edge cases

Cost:
Periodic scanning and operational logic
```

---

# 50. Thirty-Second Revision

- **Object storage:** stores large immutable bytes independently from application databases.
- **Metadata database:** stores ownership, filename, status, size, key, and processing state.
- **Control plane:** authorization, session creation, metadata, and lifecycle.
- **Data plane:** direct upload and CDN or storage download.
- **Presigned URL:** temporary scoped capability for a storage request.
- **Object key:** server-generated authoritative storage location.
- **Multipart upload:** upload numbered chunks under one provider session.
- **Upload ID:** provider identifier for the multipart session.
- **Part number:** ordering position of a chunk.
- **ETag or receipt:** provider identifier returned for a stored part; not universally a plain MD5.
- **Fingerprint:** client-side way to recognize the same local file after restart.
- **Resume:** list provider-accepted parts and upload only missing parts.
- **Completion:** submit ordered part receipts so storage assembles one final object.
- **Parallel transfer:** improves throughput carefully, but increases requests and memory use.
- **Storage event:** fast signal that a final object exists.
- **Reconciliation:** safety net for missed events, orphans, and stuck metadata.
- **Quarantine:** isolated location for untrusted uploads before scanning.
- **CDN signed access:** edge-validated authorization for protected downloads.
- **Origin protection:** prevent direct public access around the CDN.
- **Range request:** download selected bytes for resume and seeking.
- **Lifecycle rules:** abort incomplete sessions, tier old objects, and clean temporary data.
- **Integrity:** verify explicit checksums and expected size.
- **Best principle:** application servers orchestrate access and state; storage infrastructure transports bytes.

## Final Mental Model

```text
Client wants to upload
    ↓
API authenticates, checks quota, allocates file ID and key
    ↓
Metadata row = PENDING
    ↓
API returns short-lived scoped upload capability
    ↓
Small file?
    ├── Single direct upload
    └── Large file → multipart session
            ↓
        Upload ID + numbered parts + part receipts
            ↓
        Resume by listing accepted parts
            ↓
        Complete and assemble final object
    ↓
Storage event enters durable processing path
    ↓
Verify size, checksum, type, ownership
    ↓
Quarantine scan and transformations
    ↓
Metadata = AVAILABLE
    ↓
Authorized downloads through CDN or storage
    ↓
Range requests, versioned keys, lifecycle cleanup

At every stage ask:
    Who owns the object?
    What proves the bytes exist?
    Can this operation repeat safely?
    How does resume work?
    When is the object trusted?
    How is stale or orphaned state repaired?
```
