# API Design: Quick Revision Notes

## Core Thesis

**Define a clear client-server contract based on resources, operations, data flow, and non-functional requirements.**

## Key Insights

- **REST is the client-facing default**
- **RPC fits internal service communication**
- **GraphQL fits flexible client data requirements**
- **Resources should be nouns, not actions**
- **HTTP methods communicate operation semantics**
- **Idempotency protects retries from duplicate effects**
- **Pagination is mandatory for unbounded collections**
- **Authentication proves identity; authorization controls access**
- **Keep the API section short in interviews**

## API Design in Interviews

### Main Goal

- Define client interaction
- Select communication protocol
- Identify resources
- Define critical endpoints
- Specify request inputs
- Specify response outputs
- Address pagination
- Mention security
- Move to architecture

### Expected Depth

- Five-minute discussion
- Main user-facing endpoints
- Reasonable request and response shapes
- Important error cases
- Critical idempotency requirements
- No exhaustive production specification

**Interview principle:** Design a reasonable API, explain key choices, then move on.

## API Design Flow

```text
Functional requirements
        ↓
Core entities
        ↓
API resources
        ↓
Client operations
        ↓
Endpoints or procedures
        ↓
Request and response shapes
        ↓
Pagination, idempotency, and security
```

## API Protocol Options

1. **REST**
2. **GraphQL**
3. **RPC**
4. **WebSockets or SSE for real-time updates**

## Protocol Selection

### Client-Facing API

- Standard CRUD → REST
- Flexible field selection → GraphQL
- Bidirectional real-time updates → WebSocket
- Server-to-client event stream → SSE

### Internal API

- Frequent service communication → RPC
- Low latency → gRPC
- Type-safe contracts → gRPC
- Streaming → gRPC

### Decision Flow

```text
Is the API client-facing?
    |
    +-- Yes
    |    |
    |    +-- Flexible field selection required?
    |           |
    |           +-- Yes → GraphQL
    |           +-- No  → REST
    |
    +-- No
         |
         +-- Internal service communication → RPC
```

## REST

### Definition

- Resource-oriented API
- Standard HTTP methods
- Resources identified by URLs
- Usually JSON payloads
- Stateless request model
- Natural CRUD mapping

### Best Fit

- Web applications
- Mobile applications
- Public APIs
- Standard CRUD
- Simple resource interactions
- Broad client compatibility

### Advantages

- Familiar conventions
- Excellent tooling
- Easy debugging
- HTTP caching support
- Broad ecosystem
- Simple client integration

### Trade-Offs

- Multiple requests for related resources
- Over-fetching
- Under-fetching
- Endpoint proliferation
- Versioning concerns

**Interview default:** Choose REST unless requirements clearly justify something else.

## REST Resource Modeling

### Core Principle

**Model things, not actions.**

### Good Resources

- Events
- Venues
- Tickets
- Bookings
- Users
- Orders
- Payments

### Avoid Action-Oriented URLs

Avoid:

```http
POST /bookTicket
POST /cancelBooking
GET /getEvent
```

Prefer:

```http
POST /events/{event_id}/bookings
DELETE /bookings/{booking_id}
GET /events/{event_id}
```

## Resource Naming

- Plural nouns
- Lowercase names
- Stable identifiers
- Consistent naming
- Clear hierarchy

Examples:

```http
/events
/events/{event_id}
/bookings
/bookings/{booking_id}
/users/{user_id}/orders
```

Avoid:

- Verbs in resource names
- Singular and plural mixtures
- Database table details
- Implementation-specific URLs
- Deep nesting

## Ticketing API Example

```http
GET    /events
GET    /events/{event_id}
GET    /venues/{venue_id}
GET    /events/{event_id}/tickets
POST   /events/{event_id}/bookings
GET    /bookings/{booking_id}
DELETE /bookings/{booking_id}
```

## Nested vs Flat Resources

### Nested Resource

```http
GET /events/{event_id}/tickets
```

Use when:

- Parent relationship required
- Child has clear parent
- Query meaningless without parent
- Scope improves readability

### Flat Resource with Filter

```http
GET /tickets?event_id=123&section=VIP
```

Use when:

- Parent filter optional
- Multiple filters supported
- Resource queried independently
- Several access paths exist

**Selection rule:** Required relationship → path; optional filter → query parameter.

## Nesting Depth

Good:

```http
/events/{event_id}/tickets
```

Too deep:

```http
/venues/{venue_id}/events/{event_id}/sections/{section_id}/tickets/{ticket_id}
```

Better:

```http
/tickets/{ticket_id}
```

**Rule:** Keep nesting shallow, usually one relationship level.

## HTTP Methods

### GET

- Retrieve resource
- List resources
- No intended state change
- Safe
- Idempotent
- Cacheable

```http
GET /events
GET /events/{event_id}
```

### POST

- Create child resource
- Submit command
- Trigger non-idempotent operation
- Server usually generates ID
- Repeated requests may create duplicates

```http
POST /events/{event_id}/bookings
```

Typical response:

```http
201 Created
Location: /bookings/booking-123
```

### PUT

- Replace complete resource
- Create at known URI when supported
- Idempotent
- Same request → same final state

```http
PUT /users/{user_id}
```

### PATCH

- Partially update resource
- Idempotency depends on operation

Idempotent:

```json
{ "email": "new@example.com" }
```

Non-idempotent:

```json
{ "operation": "increment", "amount": 1 }
```

### DELETE

- Remove or cancel resource
- Idempotent final state
- First call may return `204`
- Later call may return `404`

```http
DELETE /bookings/{booking_id}
```

## HTTP Method Summary

- **GET:** Read
- **POST:** Create or execute
- **PUT:** Replace
- **PATCH:** Partially update
- **DELETE:** Remove

## Safety vs Idempotency

### Safe Method

- Does not change server state
- Example: GET

### Idempotent Method

- Repeated execution has same intended final state

### Classification

- GET → safe and idempotent
- PUT → idempotent
- DELETE → idempotent
- POST → not inherently idempotent
- PATCH → implementation-dependent

## Idempotency

**Definition:** Retrying the same request does not create additional effects.

### Why It Matters

- Network timeout
- Lost response
- Client retry
- Load-balancer retry
- Message redelivery
- Duplicate submission

### Failure Example

1. Client creates booking
2. Server commits booking
3. Response lost
4. Client retries
5. Second booking created

### Protection

**Idempotency key**

## Idempotency-Key Pattern

```http
POST /events/123/bookings
Idempotency-Key: booking-request-abc123
```

```json
{
  "ticket_ids": ["ticket-1", "ticket-2"],
  "payment_method_id": "payment-method-10"
}
```

### Server Flow

1. Receive idempotency key
2. Check existing result
3. If found, return saved response
4. Otherwise process request
5. Store key and result
6. Return response

```text
IdempotencyRecords
- key
- user_id
- request_hash
- status
- response
- expires_at
```

Suitable for:

- Booking creation
- Payment initiation
- Order creation
- Refund request
- Message submission

## Passing Data to REST APIs

1. **Path parameters**
2. **Query parameters**
3. **Request body**
4. **Headers**

## Path Parameters

- Identify specific resource
- Express required hierarchy
- Select endpoint target
- Usually identifiers

```http
GET /events/123
GET /users/456/orders
GET /bookings/789
```

Use when:

- Value is required
- Resource identity
- Parent scope required
- Request meaningless without value

## Query Parameters

- Filter
- Sort
- Search
- Paginate
- Modify optional behavior

```http
GET /events?city=London&date=2026-08-09&limit=20
```

Common parameters:

```text
city
status
sort
order
limit
cursor
offset
page
include
```

Use when:

- Filter optional
- Multiple filters possible
- Collection endpoint
- Retrieval behavior changes

## Request Body

- Create resource
- Update resource
- Send complex payload
- Carry nested data

```http
POST /events/123/bookings
Content-Type: application/json
```

```json
{
  "tickets": [
    { "section": "VIP", "quantity": 2 }
  ],
  "payment_method_id": "pm-123"
}
```

## Request Headers

Common uses:

- Authentication
- Content type
- Idempotency key
- Correlation ID
- Conditional request
- Client version

```http
Authorization: Bearer <token>
Content-Type: application/json
Idempotency-Key: abc123
X-Correlation-ID: request-789
If-Match: "version-7"
```

## Input Placement Summary

- **Path:** Which resource?
- **Query:** How should results be filtered or returned?
- **Body:** What data should be created or updated?
- **Header:** Request metadata

## Complete Booking Request

```http
POST /events/123/bookings?notify=true
Authorization: Bearer <token>
Idempotency-Key: booking-abc123
Content-Type: application/json
```

```json
{
  "tickets": [
    { "section": "VIP", "quantity": 2 },
    { "section": "General", "quantity": 1 }
  ],
  "payment_method_id": "pm-456"
}
```

## API Responses

### Components

1. Status code
2. Headers
3. Response body

```http
201 Created
Location: /bookings/booking-123
Content-Type: application/json
```

```json
{
  "booking_id": "booking-123",
  "event_id": "123",
  "status": "confirmed",
  "created_at": "2026-08-09T10:30:00Z"
}
```

## Common HTTP Status Codes

### Success

- **200 OK:** Successful read or update
- **201 Created:** Resource created
- **202 Accepted:** Asynchronous processing started
- **204 No Content:** Success without response body

### Client Errors

- **400 Bad Request:** Invalid input
- **401 Unauthorized:** Authentication missing or invalid
- **403 Forbidden:** Authenticated but not permitted
- **404 Not Found:** Resource absent
- **409 Conflict:** State conflict
- **422 Unprocessable Content:** Semantically invalid input
- **429 Too Many Requests:** Rate limit exceeded

### Server Errors

- **500 Internal Server Error:** Unexpected server failure
- **502 Bad Gateway:** Invalid upstream response
- **503 Service Unavailable:** Temporarily unavailable
- **504 Gateway Timeout:** Upstream timeout

## 401 vs 403

- **401:** Who are you?
- **403:** I know you, but you cannot do this.

## Error Response Design

```json
{
  "error": {
    "code": "SEAT_UNAVAILABLE",
    "message": "The selected seat is no longer available.",
    "request_id": "req-123"
  }
}
```

Useful fields:

- Machine-readable code
- Human-readable message
- Request or correlation ID
- Field-level validation errors
- Retry guidance

Avoid:

- Stack traces
- SQL errors
- Internal hostnames
- Secret values
- Implementation details

## Asynchronous APIs

Use when:

- Operation takes a long time
- Video processing
- Bulk export
- Report generation
- Background import
- Machine-learning job

```http
POST /exports
```

```http
202 Accepted
```

```json
{
  "job_id": "job-123",
  "status": "queued",
  "status_url": "/jobs/job-123"
}
```

Status endpoint:

```http
GET /jobs/job-123
```

Possible states:

- Queued
- Running
- Completed
- Failed
- Cancelled

## GraphQL

### Definition

- Single API endpoint
- Client-defined response shape
- Strongly typed schema
- Queries and mutations
- Relationship traversal

```http
POST /graphql
```

**Main goal:** Avoid over-fetching and under-fetching across diverse clients.

## GraphQL Query Example

```graphql
query {
  event(id: "123") {
    name
    date
    venue {
      name
      address
    }
    tickets {
      section
      price
      available
    }
  }
}
```

## GraphQL Strengths

- Flexible field selection
- Different mobile and web responses
- Reduced over-fetching
- Reduced endpoint proliferation
- Schema introspection
- Strong client tooling
- Relationship traversal

## GraphQL Trade-Offs

- Server complexity
- Query parsing and validation
- Field-level authorization
- Expensive arbitrary queries
- Difficult HTTP caching
- N+1 query problem
- Query-depth abuse
- Cost estimation required

**Interview rule:** Use GraphQL for a demonstrated client-data flexibility problem, not by default.

## GraphQL Schema Example

```graphql
type Event {
  id: ID!
  name: String!
  date: DateTime!
  venue: Venue!
  tickets: [Ticket!]!
}

type Venue {
  id: ID!
  name: String!
  address: String!
}

type Query {
  event(id: ID!): Event
  events(first: Int, after: String): EventConnection!
}
```

## GraphQL N+1 Problem

1. Query 100 events
2. Resolve venue for each event
3. Execute 100 venue queries
4. Total: 101 database queries

Solution:

```text
Load 100 events
      ↓
Collect venue IDs
      ↓
Fetch all venues in one query
      ↓
Map venues to events
```

Mitigations:

- DataLoader-style batching
- Request-scoped cache
- Join optimization
- Preloading
- Query planning

## GraphQL Authorization and Protection

- Field-level authorization
- Type-level authorization
- Resolver-level checks
- Maximum depth
- Maximum complexity
- Query timeout
- Pagination limits
- Persisted queries
- Cost-based rate limiting

## RPC

**Definition:** Call a remote procedure as if invoking a function.

### Style

- Action-oriented
- Strong service contracts
- Generated client and server code
- Usually binary payload
- Common for internal APIs

Examples:

- gRPC
- Apache Thrift

```text
getEvent(eventId)
createBooking(eventId, userId, tickets)
getAvailableTickets(eventId, section)
checkPermission(userId, resource)
```

## gRPC

### Main Technologies

- Protocol Buffers
- HTTP/2
- Binary serialization
- Code generation
- Streaming support

### Advantages

- Low serialization overhead
- Compact payloads
- Strong type safety
- Generated clients
- Multi-language support
- Bidirectional streaming
- Efficient service communication

### Trade-Offs

- Less browser-friendly
- Harder manual debugging
- Schema compatibility management
- Generated code dependency
- Public API adoption may be harder

## Protocol Buffer Example

```protobuf
service TicketService {
  rpc GetEvent(GetEventRequest) returns (Event);
  rpc CreateBooking(CreateBookingRequest) returns (Booking);
}

message GetEventRequest {
  string event_id = 1;
}

message Event {
  string id = 1;
  string name = 2;
  int64 date = 3;
}
```

## When to Use RPC

- Internal microservices
- High request frequency
- Performance-sensitive communication
- Polyglot services
- Strong contracts
- Streaming
- Low payload overhead

```text
Web or Mobile Client
        |
      REST
        |
    API Gateway
        |
       gRPC
        |
Booking Service
   |          |
  gRPC       gRPC
   |          |
Inventory   Payment
Service     Service
```

## REST vs GraphQL vs RPC

### REST

- Client-facing APIs
- CRUD
- Public APIs
- Simple resource interactions
- Main strength: simplicity
- Main issue: fixed response shapes

### GraphQL

- Diverse clients
- Flexible field selection
- Connected data
- Rapid frontend changes
- Main strength: client-defined response
- Main issue: query complexity

### RPC

- Internal services
- Low-latency communication
- Strong typing
- Streaming
- Main strength: performance and type safety
- Main issue: contract coupling and browser support

## Real-Time Communication

### WebSocket

- Persistent connection
- Bidirectional communication
- Low-latency updates

Best fit:

- Chat
- Multiplayer games
- Collaborative editing
- Live bidding
- Interactive notifications

Costs:

- Stateful connections
- Heartbeats
- Reconnection
- Message ordering
- Horizontal scaling complexity

### Server-Sent Events

- Persistent HTTP connection
- Server-to-client only
- Text-based stream
- Automatic browser reconnection
- Simpler than WebSocket

Best fit:

- Notifications
- Live dashboards
- Progress updates
- News feeds
- Status streams

**Selection rule:** Two-way real time → WebSocket; server-to-client only → SSE.

## Pagination

### Purpose

- Bound response size
- Reduce latency
- Reduce memory usage
- Protect database
- Improve client experience

Use for:

- Events
- Posts
- Comments
- Orders
- Search results
- Notifications
- Audit records

## Offset-Based Pagination

```http
GET /events?offset=20&limit=10
```

Advantages:

- Simple
- Human-readable
- Page-number navigation
- Easy implementation

Disadvantages:

- Slow deep offsets
- Duplicates under concurrent inserts
- Missed records under deletes
- Unstable for rapidly changing data

Best fit:

- Small datasets
- Admin dashboards
- Stable data
- Page-number navigation

## Cursor-Based Pagination

First request:

```http
GET /events?limit=10
```

Response:

```json
{
  "events": [],
  "next_cursor": "encoded-cursor-value",
  "has_more": true
}
```

Next request:

```http
GET /events?cursor=encoded-cursor-value&limit=10
```

Cursor usually includes:

- Last record ID
- Timestamp
- Sort value
- Tie-breaker ID

Query:

```sql
SELECT *
FROM events
WHERE (created_at, event_id) < (?, ?)
ORDER BY created_at DESC, event_id DESC
LIMIT 20;
```

Supporting index:

```text
(created_at DESC, event_id DESC)
```

Advantages:

- Stable under inserts
- Efficient deep pagination
- No large offset scan
- Good for feeds and timelines

Disadvantages:

- Cannot easily jump to page 50
- Cursor encoding required
- Deterministic sorting required
- More complex implementation

**Selection rule:** Small stable dataset → offset; large changing dataset → cursor.

## Filtering and Sorting

```http
GET /events?city=London&status=active&sort=starts_at&order=asc
```

Requirements:

- Validate allowed filters
- Validate sort fields
- Limit page size
- Add supporting indexes
- Avoid arbitrary expensive queries

Possible index:

```text
(city, starts_at)
```

**Principle:** API query options must be supported by the data model and indexes.

## API Versioning

### URL Versioning

```http
GET /v1/events
GET /v2/events
```

Advantages:

- Explicit
- Easy routing
- Easy documentation
- Easy browser testing

**Interview default:** Use URL versioning if versioning must be discussed.

### Header Versioning

```http
API-Version: 2
```

Advantages:

- Cleaner URLs
- Content-negotiation alignment

Disadvantages:

- Less visible
- Harder manual testing
- Less intuitive

## Backward-Compatible Changes

Usually safe:

- Add optional response field
- Add optional request field
- Add new endpoint

Potentially breaking:

- Remove field
- Rename field
- Change field type
- Change required-field behavior
- Change status semantics
- Change pagination ordering

**Rule:** Prefer additive API evolution.

## Authentication vs Authorization

### Authentication

**Who is making the request?**

Examples:

- Session cookie
- JWT
- API key
- OAuth access token

### Authorization

**Can the caller perform this operation?**

Examples:

- Owns booking
- Has manager role
- Can edit venue
- Can issue refund

**Memory trick:** Authentication → identity; authorization → permission.

## API Keys

- Long random credential
- Identifies application or integration
- Included in request header
- Stored and validated by server

```http
Authorization: Bearer sk_live_abc123
```

Best fit:

- External developers
- Server-to-server integrations
- Application-level identity
- Usage tracking
- Client-specific rate limits

Limitations:

- Weak user context
- Rotation required
- Leakage risk
- Often long-lived
- Not ideal for user sessions

## JWT

**Full name:** JSON Web Token

```json
{
  "sub": "user-123",
  "role": "customer",
  "exp": 1786271400,
  "iss": "auth-service",
  "aud": "ticket-api"
}
```

Advantages:

- Self-contained claims
- Signature verification
- Distributed validation
- No lookup for every request
- Useful across services

Trade-Offs:

- Revocation difficulty
- Claim staleness
- Token leakage risk
- Payload visible unless encrypted
- Key rotation required

**Important:** JWTs are signed, not inherently encrypted.

## Session-Based Authentication

1. User logs in
2. Server creates session
3. Session stored centrally
4. Client receives session cookie
5. Server looks up session per request

Advantages:

- Easy revocation
- Central control
- Smaller client credential
- Immediate permission changes

Costs:

- Session-store lookup
- Shared infrastructure
- State management
- Cross-region replication

## RBAC

```text
User → Role → Permissions
```

Example:

```text
Customer
- book tickets
- view own bookings
- cancel own bookings

Venue Manager
- create events
- view venue sales
- manage inventory

Admin
- access all resources
```

## Resource-Level Authorization

```http
GET /bookings/{booking_id}
```

Rules:

- Customer owns booking
- Venue manager manages venue
- Admin has global access

**Critical rule:** Never trust a resource ID merely because the client supplied it.

Safe query:

```sql
SELECT *
FROM bookings
WHERE booking_id = ?
  AND user_id = ?;
```

## Multi-Tenant Security

Tenant identity may come from:

- Token claim
- Subdomain
- Gateway context
- Trusted request path

```sql
SELECT *
FROM events
WHERE tenant_id = ?
  AND event_id = ?;
```

**Rule:** Derive tenant identity from trusted authentication context, not only client input.

## Rate Limiting

### Purpose

- Prevent abuse
- Protect infrastructure
- Control cost
- Prevent scalping
- Limit accidental overload
- Enforce quotas

### Dimensions

- Per user
- Per IP
- Per endpoint
- Per API key
- Per tenant

```http
429 Too Many Requests
Retry-After: 30
```

Implementation locations:

- API gateway
- Load balancer
- Edge service
- Middleware
- Distributed rate-limit service

### Algorithms

- Fixed window
- Sliding window
- Token bucket
- Leaky bucket

## Request Validation

Validate:

- Required fields
- Data types
- Length limits
- Numeric ranges
- Enum values
- Date formats
- Resource limits
- Unsupported fields

Example rule:

```text
1 <= quantity <= 10
```

## API Gateway

Responsibilities:

- TLS termination
- Authentication
- Rate limiting
- Request routing
- Logging
- Metrics
- Version routing
- Request-size limits
- WAF rules

Avoid putting in gateway:

- Core business logic
- Large transformations
- Long-running workflows
- Complex database operations

## Timeouts and Retries

### Timeout

- Bound waiting time
- Prevent resource exhaustion
- Protect thread and connection pools
- Operation-specific values

### Retry

Use for:

- Temporary failure
- Timeout
- `503`
- Some `429` responses

Avoid blind retry for:

- Validation error
- Permission failure
- Permanent conflict
- Non-idempotent operation without key

Strategy:

- Exponential backoff
- Random jitter
- Retry limit
- Deadline propagation
- Idempotency protection

## Optimistic Concurrency Control

Problem:

- Two clients read same resource
- Both update
- Later write overwrites earlier one

```http
PATCH /bookings/booking-123
If-Match: "7"
```

Result:

- Version matches → update
- Version differs → `409` or `412`

Best fit:

- Profile edits
- Inventory updates
- Collaborative modification
- Booking state transitions

## API Caching

Cacheable:

- GET
- Public resources
- Stable metadata
- Event details
- Product descriptions

```http
Cache-Control: max-age=60
ETag: "event-version-12"
```

Conditional request:

```http
If-None-Match: "event-version-12"
```

Possible response:

```http
304 Not Modified
```

Avoid shared caching for:

- Sensitive private responses
- Rapidly changing inventory
- Payment state
- User-specific data without proper controls

## API Observability

### Metadata

- Request ID
- Correlation ID
- Endpoint
- Status code
- Latency
- User or tenant ID
- Error code
- Retry count

### Metrics

- Requests per second
- Error rate
- p50 latency
- p95 latency
- p99 latency
- Rate-limit rejections
- Payload size
- Dependency latency

## Common API Design Mistakes

### Modeling Actions Instead of Resources

Bad:

```http
POST /createBooking
```

Better:

```http
POST /bookings
```

### Returning Unbounded Collections

Bad:

```http
GET /events
```

Better:

```http
GET /events?limit=20&cursor=...
```

### Using POST Without Retry Protection

Risks:

- Duplicate booking
- Duplicate order
- Duplicate payment

Solution:

- Idempotency key
- Unique request identifier
- Database constraint

### Putting Sensitive Data in URL

Avoid:

```http
GET /users?password=secret
```

### Other Mistakes

- Confusing 401 and 403
- Using GraphQL without need
- Using RPC for browser APIs by default
- Ignoring ownership checks
- Exposing internal database schema
- Designing every possible endpoint

## API Design Interview Framework

1. **Select protocol**
2. **Identify resources**
3. **Define critical endpoints**
4. **Define inputs**
5. **Define outputs**
6. **Address scale**
7. **Address correctness**
8. **Connect APIs to the data model**

### 1. Select Protocol

- REST for client-facing APIs
- gRPC for internal services when justified
- GraphQL only for flexible client data needs

### 2. Identify Resources

```text
User
Event
Ticket
Booking
Payment
```

Map to:

```text
/users
/events
/tickets
/bookings
/payments
```

### 3. Define Critical Endpoints

Focus on:

- List or search
- Get details
- Create
- Update
- Delete or cancel

### 4. Define Inputs

- Path identifiers
- Query filters
- Request body
- Authentication header
- Idempotency key

### 5. Define Outputs

- Status code
- Core response fields
- Error shape
- Pagination cursor
- Job status if asynchronous

### 6. Address Scale

- Pagination
- Maximum page size
- Filtering limits
- Rate limiting
- Caching
- Asynchronous processing

### 7. Address Correctness

- Idempotency
- Validation
- Unique constraints
- Optimistic concurrency
- Resource ownership
- Authentication and authorization

### 8. Connect to Data Model

For each endpoint:

- Query pattern
- Required index
- Consistency requirement
- Cacheability
- Expected load

## Ticketing API: Interview Example

### Search Events

```http
GET /v1/events?city=London&starts_after=2026-08-09T00:00:00Z&limit=20&cursor=...
```

```json
{
  "events": [
    {
      "event_id": "event-123",
      "name": "Live Concert",
      "starts_at": "2026-08-20T19:30:00Z",
      "venue": {
        "venue_id": "venue-10",
        "name": "Central Arena"
      }
    }
  ],
  "next_cursor": "cursor-value",
  "has_more": true
}
```

Supporting index:

```text
(city, starts_at, event_id)
```

### Get Available Tickets

```http
GET /v1/events/{event_id}/tickets?section=VIP&limit=50&cursor=...
```

Note:

- Availability may change immediately
- Response is informational
- Final validation occurs during reservation

### Create Booking

```http
POST /v1/events/{event_id}/bookings
Authorization: Bearer <token>
Idempotency-Key: booking-request-123
```

```json
{
  "ticket_ids": ["ticket-10", "ticket-11"],
  "payment_method_id": "pm-123"
}
```

Critical requirements:

- Authenticated user
- Idempotency
- Seat availability validation
- Strong consistency
- No double-booking
- Payment coordination

### Cancel Booking

```http
DELETE /v1/bookings/{booking_id}
Authorization: Bearer <token>
```

Checks:

- Booking exists
- Caller owns booking
- Cancellation permitted
- Refund policy satisfied

## Interview Answer Template

> I’ll use REST for the client-facing API because the main operations are resource-oriented CRUD workflows. The core resources are events, tickets, and bookings. Clients can list events with a cursor-paginated `GET /events`, retrieve ticket availability with `GET /events/{event_id}/tickets`, and create a booking with `POST /events/{event_id}/bookings`. Booking creation will require authentication and an idempotency key because clients may retry after a timeout. I’ll use standard status codes and a consistent error response. Internal communication between booking, payment, and inventory services can use gRPC for lower overhead and strongly typed contracts.

## Quick Protocol Questions

### Choose REST If

- Standard CRUD?
- Public or client-facing?
- Resources clearly defined?
- Simple request-response?
- Broad compatibility needed?

### Choose GraphQL If

- Multiple clients need different fields?
- Over-fetching significant?
- Connected data fetched together?
- Frontend iteration speed important?

### Choose RPC If

- Internal services?
- Strong typing?
- High call volume?
- Low latency?
- Streaming required?

### Choose WebSocket If

- Bidirectional real-time communication?
- Chat or collaboration?
- Immediate client and server messages?

### Choose SSE If

- Server-to-client updates only?
- Notification stream?
- Live dashboard?
- Simpler HTTP streaming preferred?

## Important Terms

- **API:** Contract for software interaction
- **REST:** Resource-oriented HTTP API style
- **Resource:** Domain object exposed through API
- **Endpoint:** Network-accessible API operation
- **HTTP Method:** Verb describing request semantics
- **Safe Method:** Operation intended not to change state
- **Idempotency:** Repeated request produces same intended final state
- **Idempotency Key:** Client identifier preventing duplicate processing
- **Path Parameter:** Required resource identifier in URL
- **Query Parameter:** Optional filter or behavior modifier
- **Request Body:** Structured operation payload
- **Status Code:** Numeric request-result classification
- **GraphQL:** Client-defined query API with typed schema
- **Resolver:** Function supplying a GraphQL field
- **N+1 Problem:** One initial query followed by many related queries
- **DataLoader:** Request-scoped batching and caching pattern
- **RPC:** Remote procedure invocation model
- **gRPC:** HTTP/2 and Protocol Buffers RPC framework
- **Protocol Buffers:** Binary schema and serialization format
- **Pagination:** Splitting a collection into bounded pages
- **Offset Pagination:** Pagination using skipped-record count
- **Cursor Pagination:** Pagination using stable record position
- **Authentication:** Verification of caller identity
- **Authorization:** Verification of caller permission
- **API Key:** Credential identifying an application or integration
- **JWT:** Signed token carrying claims
- **Session:** Server-managed authentication state
- **RBAC:** Permission model based on roles
- **Rate Limiting:** Restriction on request frequency
- **Throttling:** Delaying or rejecting excess requests
- **Optimistic Concurrency:** Version-based conflict detection
- **ETag:** Identifier representing a resource version
- **Correlation ID:** Identifier tracing a request across services
- **Backward Compatibility:** Ability for old clients to continue working
- **WebSocket:** Persistent bidirectional connection
- **SSE:** Persistent server-to-client event stream

## 30-Second Revision Summary

- **Client-facing default?** REST
- **Internal API default?** gRPC when justified
- **Flexible client fetching?** GraphQL
- **Real-time bidirectional?** WebSocket
- **Server-to-client stream?** SSE
- **REST models?** Resources, not actions
- **Resource naming?** Plural nouns
- **Required identifier?** Path parameter
- **Optional filter?** Query parameter
- **Complex payload?** Request body
- **Read method?** GET
- **Create method?** POST
- **Full replacement?** PUT
- **Partial update?** PATCH
- **Removal?** DELETE
- **Retry-sensitive write?** Idempotency key
- **Large collection?** Pagination
- **High-volume changing collection?** Cursor pagination
- **Identity check?** Authentication
- **Permission check?** Authorization
- **Invalid token?** 401
- **Insufficient permission?** 403
- **Rate limit exceeded?** 429
- **GraphQL danger?** N+1 queries
- **Public API evolution?** Versioning and additive changes
- **Best interview principle?** Define critical APIs, explain trade-offs, and move on
