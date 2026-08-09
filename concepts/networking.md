# Networking Essentials: Quick Revision Notes

## Core Thesis

**Distributed systems depend on layered protocols, efficient routing, locality, and explicit failure handling.**

## Key Insights

- **Networking layers hide lower-level complexity**
- **IP routes packets between machines**
- **TCP provides reliability and ordering**
- **UDP trades guarantees for lower overhead**
- **HTTP is the default application protocol**
- **Persistent connections require connection-aware scaling**
- **Load balancers distribute connections or requests**
- **Data locality reduces unavoidable network latency**
- **Every network call can fail, stall, or duplicate**
- **Timeouts, retries, idempotency, and circuit breakers work together**

## Interview Expectations

### Product or Full-Stack Roles

- Basic TCP vs UDP
- HTTP and HTTPS
- REST APIs
- Load balancers
- CDN usage
- Basic failure handling

### Infrastructure or Distributed-Systems Roles

- Connection lifecycle
- L4 vs L7 balancing
- Persistent connections
- DNS behavior
- Regional latency
- Retry amplification
- Circuit breakers
- Client-side routing
- WebSocket infrastructure
- QUIC and HTTP/3 trade-offs

**Interview principle:** Explain the networking choice only as deeply as the system requirements demand.

## Networking Layers

### OSI Model

```text
Layer 7: Application
Layer 6: Presentation
Layer 5: Session
Layer 4: Transport
Layer 3: Network
Layer 2: Data Link
Layer 1: Physical
```

### Most Relevant Layers

- **Layer 3:** IP addressing and routing
- **Layer 4:** TCP, UDP, and QUIC
- **Layer 7:** HTTP, DNS, WebSocket, SSE, and WebRTC

## Layered Abstraction

- Each layer uses the layer below
- Each layer exposes a simpler abstraction
- Application avoids electrical details
- Transport hides packet retransmission
- IP hides route selection
- HTTP hides message framing

**Benefit:** Application developers reason about requests, not electrical signals and router hops.

## Network Layer: Layer 3

### Internet Protocol, or IP

- Machine addressing
- Packet routing
- Packet forwarding
- Cross-network communication
- Best-effort delivery

### IP Guarantees

- Routes packets toward destination
- Does not guarantee delivery
- Does not guarantee ordering
- Does not prevent duplicates
- Does not establish a connection

## IP Addresses

### **Public IP**

- Globally routable
- Allocated address range
- Reachable through internet routing
- Used by public-facing infrastructure

### **Private IP**

- Internal-network address
- Not directly internet-routable
- Reusable across private networks
- Common inside data centers and VPCs

### Common Private Ranges

```text
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

### Dynamic Address Assignment

- Commonly assigned through DHCP
- Address provided when machine joins network
- Lease may expire or change

## Ports

**Purpose:** Identify the destination process or service on a machine.

### Connection Identity

```text
Source IP
Source port
Destination IP
Destination port
Transport protocol
```

### Common Examples

```text
HTTP  → 80
HTTPS → 443
DNS   → 53
```

## DNS

**Full name:** Domain Name System

**Purpose:** Convert human-readable names into network addresses.

```text
example.com → 203.0.113.10
```

### Simplified Resolution Flow

1. Browser checks local cache
2. Operating system checks cache
3. Resolver receives query
4. Resolver follows DNS hierarchy
5. Address record returned
6. Result cached using TTL
7. Client connects to returned address

### DNS Hierarchy

```text
Root server
    ↓
Top-level-domain server
    ↓
Authoritative name server
    ↓
IP address
```

### Common Record Types

- **A:** IPv4 address
- **AAAA:** IPv6 address
- **CNAME:** Alias to another name
- **MX:** Mail server
- **TXT:** Verification or policy text
- **NS:** Authoritative name server

## DNS Caching

### TTL

- Controls cache duration
- Reduces DNS traffic
- Improves resolution latency
- Delays configuration changes

### Trade-Off

- Long TTL → better caching, slower failover
- Short TTL → faster updates, more DNS traffic

**Important limitation:** DNS changes are not instant because clients and resolvers cache records.

## DNS-Based Load Distribution

- Domain maps to multiple addresses
- Clients receive different address orderings
- Traffic spreads across endpoints
- Useful for regional routing or failover

### Limitations

- Cached addresses remain in use
- Failed endpoint may remain cached
- Limited awareness of active connections
- Coarse-grained control
- Update speed bounded by TTL and client behavior

## Transport Layer: Layer 4

### Main Options

1. **TCP**
2. **UDP**
3. **QUIC**

**Main question:** Do we need delivery, ordering, flow control, and connection state?

## TCP

**Full name:** Transmission Control Protocol

### Core Properties

- Connection-oriented
- Reliable delivery
- Ordered byte stream
- Duplicate handling
- Retransmission
- Flow control
- Congestion control
- Error detection

### Best Fit

- Web APIs
- Database connections
- File transfer
- Payments
- Reliable messaging
- Most application traffic

**Interview default:** Assume TCP unless loss tolerance and latency requirements justify another protocol.

## TCP Three-Way Handshake

```text
Client                  Server

SYN       ────────────>
          <──────────── SYN-ACK
ACK       ────────────>

Connection established
```

### Purpose

- Establish connection
- Synchronize sequence numbers
- Confirm bidirectional reachability
- Initialize connection state

### Cost

- Additional network round trip
- Client and server state
- Connection resources
- Higher startup latency

## TCP Teardown

```text
Client                  Server

FIN       ────────────>
          <──────────── ACK
          <──────────── FIN
ACK       ────────────>
```

- Each direction closes independently
- TCP is full-duplex
- One side may finish before the other

## TCP Reliability

### Sequence Numbers

- Track byte position
- Restore ordering
- Detect missing segments
- Detect duplicates

### Acknowledgements

- Receiver confirms delivered data
- Missing acknowledgement triggers retransmission

### Retransmission

- Recovers lost data
- Increases latency
- Preserves correctness

**Important effect:** Packet loss appears as increased latency rather than missing application data.

## TCP Flow Control

**Purpose:** Prevent sender from overwhelming receiver.

- Receiver advertises available buffer
- Sender limits unacknowledged data
- Receive window changes dynamically

## TCP Congestion Control

**Purpose:** Prevent sender from overwhelming the network.

- Increase sending rate cautiously
- Detect congestion through loss or delay
- Reduce transmission rate
- Recover gradually

### Distinction

- Flow control → protects receiver
- Congestion control → protects network

## Head-of-Line Blocking

- Bytes delivered in order
- Missing earlier packet blocks later bytes
- Later packets may already have arrived
- Application waits for retransmission
- Increased latency under packet loss

## TCP Connection Reuse

### Without Reuse

- New handshake per request
- Repeated connection setup
- Socket churn
- Higher latency

### With Keep-Alive

- Multiple requests on one connection
- Fewer handshakes
- Lower latency
- Reduced CPU and socket churn

### HTTP/2 Improvement

- Multiple request streams
- Single TCP connection
- Application-level multiplexing

## UDP

**Full name:** User Datagram Protocol

### Core Properties

- Connectionless
- No handshake
- No delivery guarantee
- No ordering guarantee
- No duplicate protection
- No built-in flow control
- No built-in congestion control
- Small protocol overhead

**Mental model:** Send datagrams independently and let the application handle the consequences.

### Strengths

- Low startup latency
- Small header
- No connection state
- Supports multicast
- Good for time-sensitive data
- Application controls reliability

### Costs

- Packet loss
- Packet reordering
- Duplicate packets
- Application-level recovery
- Application-level congestion handling
- Limited browser access outside supported protocols

### Use Cases

- Voice communication
- Live video
- Online gaming
- DNS queries
- Real-time telemetry
- WebRTC media
- Loss-tolerant metrics

**Selection principle:** Use UDP when timely delivery matters more than complete delivery.

## Why Retransmission Can Be Harmful

### Voice Example

- Audio packet lost
- Retransmitted packet arrives late
- Playback time already passed
- Late packet no longer useful

### Better Behavior

- Drop missing audio frame
- Continue with newer audio
- Small audible interruption
- Maintain real-time conversation

## TCP vs UDP

### TCP

- Connection-oriented
- Reliable
- Ordered
- Flow control
- Congestion control
- More overhead
- Most application traffic

### UDP

- Connectionless
- Best effort
- Unordered
- No built-in flow control
- No built-in congestion control
- Lower overhead
- Real-time and loss-tolerant traffic

### Memory Trick

- **TCP:** Correct and ordered
- **UDP:** Fast and application-controlled

## QUIC

- Modern transport protocol
- Built over UDP
- Reliability implemented above UDP
- Encryption integrated through TLS
- Multiple independent streams
- Foundation of HTTP/3

### Advantages

- Faster connection establishment
- Stream-level multiplexing
- Avoids TCP-level head-of-line blocking between streams
- Connection migration
- Modern encryption
- Better behavior on changing networks

### Connection Migration

- Connection identified independently of IP tuple
- Mobile device changes network
- Wi-Fi → mobile data
- Existing logical connection can survive

### Trade-Offs

- More recent deployment
- User-space processing
- Operational complexity
- Middlebox compatibility concerns

**Interview guidance:** Mention QUIC or HTTP/3 only when latency, mobile connectivity, or modern transport is relevant.

## Simple Web Request

1. Resolve domain using DNS
2. Obtain server IP
3. Establish transport connection
4. Negotiate TLS for HTTPS
5. Send HTTP request
6. Server processes request
7. Receive HTTP response
8. Reuse or close connection

```text
HTTP request
    ↓
TCP or QUIC
    ↓
IP packets
    ↓
Network interfaces and links
```

## HTTP

**Full name:** Hypertext Transfer Protocol

### Characteristics

- Application-layer protocol
- Request-response model
- Stateless semantics
- Header-based metadata
- Extensible methods and status codes
- Usually runs over TCP
- HTTP/3 runs over QUIC

### Stateless Meaning

- Each request contains required context
- Server need not retain client session in local memory
- Easier horizontal scaling
- Session state may still exist externally

**Important nuance:** HTTP is stateless, but applications built on HTTP may still maintain sessions.

## HTTP Request and Response

```http
GET /posts/1 HTTP/1.1
Host: example.com
Accept: application/json
User-Agent: ExampleClient/1.0
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 42
```

```json
{
  "id": 1,
  "title": "A post"
}
```

## Common HTTP Methods

- **GET:** Retrieve data
- **POST:** Create or execute operation
- **PUT:** Replace resource
- **PATCH:** Partially update resource
- **DELETE:** Remove resource

### Idempotency

- GET → idempotent
- PUT → idempotent
- DELETE → idempotent final state
- POST → not inherently idempotent
- PATCH → operation-dependent

## Common HTTP Status Codes

### Success

- **200:** Successful request
- **201:** Resource created
- **202:** Accepted for asynchronous processing
- **204:** Success without body

### Redirection

- **301:** Permanent redirect
- **302:** Temporary redirect
- **304:** Cached representation still valid

### Client Errors

- **400:** Invalid request
- **401:** Authentication required
- **403:** Permission denied
- **404:** Resource not found
- **409:** State conflict
- **429:** Rate limit exceeded

### Server Errors

- **500:** Internal server failure
- **502:** Invalid upstream response
- **503:** Temporarily unavailable
- **504:** Upstream timeout

## HTTP Headers and Content Negotiation

```http
Authorization: Bearer <token>
Accept: application/json
Accept-Encoding: gzip, br
Content-Encoding: br
Cache-Control: max-age=60
X-Correlation-ID: request-123
```

### Benefits

- Backward compatibility
- Capability negotiation
- Efficient payload delivery
- Graceful degradation

## HTTPS and TLS

**HTTPS:** HTTP transmitted over an encrypted TLS connection.

### TLS Provides

- Encryption
- Server authentication
- Message integrity
- Optional client authentication

### Protects Against

- Passive eavesdropping
- Payload modification
- Many man-in-the-middle attacks

### Does Not Guarantee

- Request is honest
- User-supplied ID is authorized
- Client is uncompromised
- Payload is semantically valid
- Application has no vulnerabilities

**Core rule:** Encrypted input is still untrusted input.

## Server-Side Validation

- Authenticate caller
- Derive user ID from trusted token or session
- Validate resource ownership
- Validate request fields
- Authorize operation

## REST, GraphQL, and gRPC

### REST

- Resource-oriented
- JSON over HTTP common
- Broad compatibility
- Public API default
- Flexible and easy to debug

### GraphQL

- Client-selected fields
- Good for diverse frontend needs
- Avoids some over-fetching
- Can create resolver complexity
- Requires query-cost controls

### gRPC

- Procedure-oriented
- Protocol Buffers
- HTTP/2
- Binary serialization
- Internal service default when performance matters

## Under-Fetching

- Screen needs related data
- Client makes many requests
- More network round trips
- Higher latency
- Complicated frontend orchestration

Possible solutions:

- Aggregation endpoint
- GraphQL
- Backend-for-frontend
- Parallel requests
- Cached read model

## Over-Fetching

- Response contains unused fields
- Larger payload
- More serialization
- More network transfer
- Slower mobile experience

Possible solutions:

- Field selection
- GraphQL
- Purpose-specific endpoint
- Response projection
- Compression

## gRPC and Protocol Buffers

### Protocol Buffers

- Explicit schema
- Compact field tags
- Binary encoding
- Code generation
- Strong typing
- Backward-compatible evolution rules

### gRPC Features

- Unary requests
- Server streaming
- Client streaming
- Bidirectional streaming
- Deadlines
- Metadata
- Client-side load balancing

### Best Fit

- Internal microservices
- High-throughput communication
- Polyglot services
- Low-latency calls
- Streaming

### Avoid as Default For

- Public browser API
- Third-party integrations
- Human-debuggable API
- Simple low-volume service

## Server-Sent Events

**Full name:** Server-Sent Events, or SSE

### Core Model

- Persistent HTTP response
- Server-to-client messages
- One-way communication
- Text event stream
- Browser EventSource support

```text
data: {"id": 1, "status": "processing"}

data: {"id": 2, "status": "completed"}
```

### Best Fit

- Notifications
- Auction updates
- Dashboard updates
- Job progress
- Server-generated event streams

### Benefits

- Simple HTTP semantics
- Automatic browser reconnection
- Built-in last-event tracking
- Easier than WebSocket for one-way updates

### Limitations

- Server-to-client only
- Persistent connection resources
- Proxy and load-balancer timeouts
- Connection limits
- Buffering by intermediaries
- Replay storage may be needed

## SSE Reconnection

1. Stream connection closes
2. Client reconnects automatically
3. Client includes last received event ID
4. Server resumes from missed event
5. Events replayed if retained

Requirements:

- Stable event IDs
- Temporary event retention
- Replay or resume mechanism
- Duplicate handling

## WebSocket

- Persistent bidirectional connection
- Begins as HTTP upgrade
- Continues with WebSocket framing
- Usually runs over TCP
- Browser supported

### Best Fit

- Chat
- Multiplayer games
- Collaborative editing
- Live trading
- Interactive notifications
- High-frequency bidirectional messages

## WebSocket Connection Flow

1. Client opens TCP connection
2. Client sends HTTP upgrade request
3. Server accepts protocol upgrade
4. Connection switches to WebSocket
5. Both sides exchange frames
6. Connection remains open
7. Either side closes connection

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
```

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
```

## WebSocket Application Protocol

WebSocket provides:

- Connection
- Frames
- Ordering over underlying TCP
- Bidirectional communication

WebSocket does not define:

- Business message types
- Subscription semantics
- Authorization model
- Retry behavior
- Delivery acknowledgement
- Application-level ordering across reconnects

Client message:

```json
{
  "action": "subscribe",
  "ticker": "MSFT"
}
```

Server message:

```json
{
  "type": "ticker_update",
  "ticker": "MSFT",
  "value_in_cents": 42010
}
```

## WebSocket Scaling Challenges

- Long-lived connections
- Server memory per connection
- File descriptor limits
- Heartbeats
- Idle timeout
- Reconnection storms
- Connection draining
- Subscription state
- Message fan-out
- Backpressure
- Per-connection ordering
- Regional routing

## Correct WebSocket Load-Balancing Model

**Key principle:** Load balance the connection, not each message.

### Routing Moment

- Client sends HTTP upgrade request
- Load balancer chooses backend
- WebSocket connection established
- Connection remains pinned to backend
- Frames continue through same connection

### After Upgrade

- Load balancer may inspect frames
- Load balancer may enforce limits
- Load balancer may proxy traffic
- Backend normally remains unchanged
- Messages are not independently redistributed

### Why Not Rebalance Per Message?

- Breaks stream affinity
- Complicates frame ordering
- Requires duplicate backend connections
- Adds handshake and routing overhead
- Breaks connection-local state
- Conflicts with low-latency expectations

### Rebalancing Existing Connections

1. Close or drain connection
2. Client detects disconnect
3. Client reconnects
4. New upgrade request reaches load balancer
5. Load balancer selects another backend
6. Client restores subscriptions or session state

## L4 vs L7 for WebSockets

**Important correction:** WebSockets do not require an L4 load balancer.

Both are valid:

- L4 TCP load balancer
- WebSocket-aware L7 HTTP load balancer

The correct choice depends on routing and operational requirements.

## L4 WebSocket Load Balancing

### Behavior

- Routes TCP connection
- Uses IP and port information
- Does not require HTTP understanding
- Connection remains bound to selected backend
- Minimal application inspection

### Advantages

- High throughput
- Low overhead
- Protocol agnostic
- Natural connection affinity
- Suitable for raw TCP traffic

### Limitations

- No path-based routing
- No host-based routing
- Limited header awareness
- Authentication usually handled downstream
- Fewer HTTP-specific controls

### Choose L4 When

- Raw TCP performance important
- Simple WebSocket routing
- Very high connection volume
- No application-layer routing needed
- Protocol independence desired

## L7 WebSocket Load Balancing

### Behavior

1. Terminates or proxies HTTP connection
2. Inspects WebSocket upgrade request
3. Routes by host, path, cookie, or header
4. Establishes backend WebSocket connection
5. Proxies frames for connection lifetime

### Advantages

- Path-based routing
- Host-based routing
- Cookie-based affinity
- TLS termination
- Authentication integration
- HTTP observability
- WAF and gateway features
- Same listener for HTTP and WebSocket

### Limitations

- More processing
- Must explicitly support WebSocket upgrade
- Proxy timeout configuration required
- Connection state retained
- Backend fixed for connection lifetime

### Choose L7 When

- Route `/chat` and `/notifications` differently
- Route by hostname
- Use cookies or headers
- Central TLS termination
- HTTP gateway features required
- Shared HTTP and WebSocket infrastructure

## Actual L4 vs L7 Distinction

### L4

```text
TCP connection → selected backend
```

### L7

```text
HTTP upgrade request inspected
        ↓
Backend selected
        ↓
WebSocket connection proxied
        ↓
Same backend for connection lifetime
```

### Shared Behavior

- Selection usually occurs once
- Backend owns connection state
- Messages remain ordered within connection
- Midstream per-message rebalancing is not normal
- Failure requires client reconnection

### Interview Answer

> WebSockets can use either L4 or a WebSocket-aware L7 load balancer. L4 offers lower overhead and protocol-transparent TCP forwarding. L7 can inspect the HTTP upgrade request and route by host, path, headers, or cookies. In both cases, routing normally happens once when the connection is established. After the upgrade, the connection stays pinned to the chosen backend until it closes.

## WebSocket Backend Failure

- Backend connection breaks
- Load balancer cannot transparently restore application state
- Client observes disconnect
- Client reconnects
- New backend may be selected
- Client resubscribes or resumes session

### Application Requirements

- Reconnection backoff
- Jitter
- Subscription restoration
- Session resumption
- Last-seen message ID
- Duplicate handling
- Missed-message recovery

**Important principle:** A load balancer routes a new connection; it does not usually recreate a failed WebSocket session.

## WebSocket Connection Draining

- Stop accepting new connections
- Keep existing connections temporarily
- Wait for natural closure
- Notify clients before shutdown
- Force closure after deadline
- Clients reconnect to new servers

### Why Needed?

- Long-lived connections delay server removal
- Immediate termination causes reconnect storms
- Graceful draining reduces disruption

## SSE vs WebSocket

### SSE

- Server-to-client only
- HTTP-based
- Text events
- Automatic reconnection
- Simpler infrastructure
- Good for notifications and dashboards

### WebSocket

- Bidirectional
- Binary or text frames
- Application-defined protocol
- More stateful
- More connection management
- Good for chat, games, and collaboration

### Selection Rule

- One-way push → SSE
- Two-way persistent messaging → WebSocket
- Occasional updates → polling may be sufficient

## WebRTC

**Purpose:** Real-time peer-to-peer audio, video, and data communication.

### Main Components

- Signaling server
- STUN server
- TURN server
- Peer connection
- Media transport

### Main Use Cases

- Video calling
- Audio calling
- Conferencing
- Peer-to-peer media
- Some collaborative applications

**Interview guidance:** Use WebRTC primarily for audio and video communication.

## WebRTC Signaling

- Discover peers
- Exchange session descriptions
- Exchange network candidates
- Coordinate connection setup

**Important point:** WebRTC does not define the signaling protocol.

Possible signaling mechanisms:

- WebSocket
- HTTP
- SSE plus HTTP
- Custom service

## NAT

**Network Address Translation**

- Maps private addresses to public address
- Blocks unsolicited inbound connections
- Hides internal client topology
- Makes direct peer connection difficult

## STUN

**Session Traversal Utilities for NAT**

- Discovers public IP and port
- Identifies NAT mapping
- Helps peers attempt direct connection
- Candidate shared through signaling server

## TURN

**Traversal Using Relays around NAT**

- Relays traffic when direct connection fails
- Provides reachable intermediary
- Improves connectivity success

### Cost

- Server bandwidth
- Added latency
- Infrastructure expense
- Media passes through relay

### Memory Trick

- **STUN:** Discover how others see me
- **TURN:** Relay traffic when direct path fails

## WebRTC Connection Flow

1. Clients connect to signaling server
2. Clients exchange session metadata
3. STUN discovers public candidates
4. Peers exchange candidates
5. Direct connection attempted
6. TURN used if direct connection fails
7. Media or data begins flowing

## Load Balancing

### Purpose

- Distribute traffic
- Increase capacity
- Remove unhealthy servers
- Improve availability
- Simplify backend discovery

## Scaling Options

### Vertical Scaling

- Larger machine
- More CPU
- More memory
- Simpler architecture
- Hardware ceiling

### Horizontal Scaling

- More machines
- Greater aggregate capacity
- Higher availability
- Requires traffic distribution
- Adds distributed-system complexity

## Load-Balancing Models

1. **Client-side load balancing**
2. **Dedicated load balancer**
3. **DNS-based distribution**

## Client-Side Load Balancing

### Flow

1. Client discovers server list
2. Client stores routing metadata
3. Client chooses server
4. Client sends request directly
5. Client refreshes metadata periodically

### Advantages

- No proxy hop
- Lower routing latency
- Client-aware decisions
- Efficient internal communication

### Costs

- More complex client
- Stale server list
- Retry logic in client
- Discovery dependency
- Harder with uncontrolled clients

### Best Fit

- Internal services
- Controlled clients
- Redis Cluster
- gRPC clients
- Database drivers

## Service Discovery

### Components

- Service registry
- Health information
- Endpoint list
- Client cache
- Update mechanism

### Possible Systems

- Kubernetes service discovery
- Consul
- ZooKeeper
- etcd
- Cloud service registry

## Redis Cluster Routing Example

1. Client connects to cluster node
2. Client retrieves slot map
3. Client hashes key
4. Client selects owning node
5. Wrong node returns redirect
6. Client updates local metadata

**Benefit:** Direct requests without a proxy on every operation.

## Dedicated Load Balancer

```text
Client
   ↓
Load Balancer
   ↓
Backend Server
```

### Advantages

- Central routing
- Fast backend updates
- Health checks
- Traffic policies
- TLS termination
- Observability
- Clients unaware of backend topology

### Cost

- Additional network hop
- Load-balancer capacity
- Central infrastructure dependency
- Connection state in some modes

## Layer 4 Load Balancer

### Information Used

- Source IP
- Destination IP
- Source port
- Destination port
- Transport protocol

### Characteristics

- TCP or UDP aware
- Minimal payload inspection
- High performance
- Protocol agnostic
- Connection-level routing

### Best Fit

- Raw TCP
- UDP services
- High-throughput traffic
- WebSockets without L7 routing needs
- Database protocols
- Custom binary protocols

## Layer 7 Load Balancer

### Information Used

- Hostname
- URL path
- Headers
- Cookies
- HTTP method
- Query parameters

### Characteristics

- Understands application protocol
- Terminates or proxies client connection
- Opens or reuses backend connection
- Rich routing
- More processing overhead
- HTTP-specific features

### Best Fit

- REST APIs
- Web applications
- Path-based routing
- API gateways
- TLS termination
- WebSockets requiring L7 routing

## L4 vs L7

### L4 Strengths

- Faster
- Less inspection
- Protocol agnostic
- Good for raw transport
- Good for connection-level routing

### L7 Strengths

- Content-aware
- Path and host routing
- Header and cookie routing
- HTTP observability
- Authentication integration
- WAF integration

### Selection Rule

- Need application-aware routing → L7
- Need raw transport efficiency → L4
- Need WebSockets → either, based on routing requirements

## Health Checks

### Purpose

- Detect failed server
- Stop sending new traffic
- Restore routing after recovery
- Support automatic failover

### L4 Health Check

- Open TCP connection
- Verify port accepts connections
- Low overhead
- Limited application insight

### L7 Health Check

```http
GET /health
```

- Verify HTTP response
- Check application readiness
- More meaningful
- Potentially more expensive

## Liveness vs Readiness

### **Liveness**

**Is the process alive?**

Failure action:

- Restart process

### **Readiness**

**Can the process handle traffic?**

Failure action:

- Remove from load balancer
- Keep process running

## Health-Check Design

Avoid:

- Always returning `200`
- Checking every dependency deeply
- Expensive database queries
- Synchronized probe spikes

Prefer:

- Lightweight checks
- Separate liveness and readiness
- Failure threshold
- Recovery threshold
- Jittered intervals
- Graceful traffic removal

## Load-Balancing Algorithms

### **Round Robin**

- Sequential server selection
- Simple
- Good for similar stateless servers

### **Random**

- Random server selection
- Simple
- Good distribution at scale

### **Least Connections**

- Select server with fewest active connections
- Good for long-lived connections
- Useful for WebSocket and SSE

### **Least Response Time**

- Select fast and lightly loaded server
- Requires latency measurement
- Adapts to uneven performance

### **IP Hash**

- Hash client IP
- Creates approximate affinity
- Uneven behind shared NAT
- Remapping when backend set changes

### **Weighted Routing**

- More traffic to larger servers
- Supports heterogeneous capacity
- Useful for canary deployments

## Stateless Services

### Benefit

- Any server handles any request
- Round robin works well
- Easy autoscaling
- Easy failover
- Simple deployments

### Keep State Externally

- Database
- Redis
- Object storage
- Distributed session store

**Interview principle:** Prefer stateless application servers whenever possible.

## Session Affinity

**Definition:** Route the same client to the same backend.

### Mechanisms

- Cookie
- IP hash
- Connection affinity
- Load-balancer-generated token

### Costs

- Uneven load
- Difficult failover
- Reduced elasticity
- Local state loss

**Better long-term design:** Move session state to shared storage.

## Load-Balancer Availability

### Mitigations

- Multiple load balancers
- Active-active deployment
- Active-passive failover
- DNS distribution
- Anycast
- Managed cloud load balancer
- Health-monitored failover

## Global Latency

- Signals travel below vacuum light speed
- Fiber propagation approximately 200,000 km/s
- Long distance creates unavoidable delay
- Processing and routing add delay

**Core principle:** Distance creates latency that software cannot eliminate.

## Data Locality

Keep:

1. User near application server
2. Application server near database
3. Related data near each other

Bad architecture:

```text
User in Europe
      ↓
Application in North America
      ↓
Database in Asia
```

Result:

- Multiple long-distance round trips
- High latency
- Poor tail performance
- Higher inter-region cost

## Regional Architecture

```text
European Users
      ↓
European Services
      ↓
European Data

North American Users
      ↓
North American Services
      ↓
North American Data
```

### Benefits

- Lower latency
- Reduced cross-region traffic
- Failure isolation
- Data-residency support

### Challenges

- User movement
- Global queries
- Cross-region transactions
- Replication lag
- Conflict resolution
- Regional failover

## Availability Zones and Regions

### Availability Zone

- Isolated data-center location
- Low-latency within region
- Independent power and networking
- Protects against local failure

### Region

- Separate geographic area
- Higher inter-region latency
- Disaster-recovery boundary
- Data-residency boundary

### Common Pattern

- Multiple zones per region
- Multiple regions globally
- Local traffic routing
- Cross-region replication

## Regional Partitioning

**Core idea:** Store region-specific data near users who access it.

### Example: Ride Sharing

- Miami riders need Miami drivers
- Local matching only
- City or region becomes partition boundary
- Services and database colocated

### Benefits

- Smaller local dataset
- Low-latency matching
- Natural failure isolation
- Reduced global coordination

### Risks

- Users crossing regions
- Global accounts
- Regional imbalance
- Cross-region reporting
- Disaster recovery

## CDN

**Full name:** Content Delivery Network

### Purpose

- Cache content near users
- Reduce origin traffic
- Reduce geographic latency
- Absorb traffic spikes

### Best Fit

- Images
- Video
- JavaScript
- CSS
- Downloads
- Public cacheable APIs
- Static HTML

### Request Flow

1. Client requests content
2. DNS or anycast routes to edge
3. Edge checks cache
4. Hit → return content
5. Miss → fetch from origin
6. Store response
7. Return to client

### Trade-Offs

Benefits:

- Lower latency
- Reduced origin load
- Global distribution
- DDoS absorption
- Improved availability

Costs:

- Stale content
- Cache invalidation
- Provider expense
- Limited private-data caching
- Cache-key complexity

**Interview rule:** Use a CDN for globally distributed, cacheable content.

## Network Failure Assumption

**Fallacy:** The network is reliable.

Network calls may:

- Fail
- Time out
- Be delayed
- Duplicate
- Reorder
- Return partial data
- Succeed after caller gives up
- Reach dependency during overload

**Design consequence:** Every remote call requires an explicit failure policy.

## Timeouts

### Purpose

- Bound waiting time
- Release resources
- Prevent stuck requests
- Protect connection pools
- Limit tail latency

### Without Timeout

- Threads remain blocked
- Connections accumulate
- Queues grow
- Memory usage rises
- Failure spreads upstream

### Timeout Selection

Consider:

- Expected latency
- p99 latency
- User deadline
- Retry budget
- Downstream call depth
- Operation criticality

## Deadline Propagation

```text
Client deadline: 2 seconds
        ↓
Gateway uses 1.9 seconds
        ↓
Service A receives remaining time
        ↓
Service B receives smaller remainder
```

Benefits:

- Prevent downstream work after expiry
- Bound total latency
- Avoid resource waste
- Coordinate timeouts across call chain

## Retries

### Good For

- Transient network failure
- Temporary unavailability
- Connection reset
- Selected timeout cases
- Some `429` or `503` responses

### Poor For

- Validation errors
- Authorization failures
- Permanent conflicts
- Overloaded service without backoff
- Non-idempotent writes without protection

### Requirements

- Idempotent operation
- Attempt limit
- Retry budget
- Backoff
- Jitter
- Overall deadline

## Exponential Backoff

```text
Attempt 1 → wait 100 ms
Attempt 2 → wait 200 ms
Attempt 3 → wait 400 ms
Attempt 4 → wait 800 ms
```

Benefits:

- Gives dependency recovery time
- Reduces repeated pressure
- Avoids immediate retry loop

## Jitter

**Definition:** Random variation added to retry delay.

```text
Base delay: 400 ms
Actual delay: random value between 200 and 600 ms
```

Purpose:

- Prevent synchronized retries
- Reduce thundering herd
- Smooth recovery load
- Avoid retry waves

**Interview phrase:** Retry with exponential backoff and jitter.

## Retry Amplification

```text
Client retries 3 times
Service A retries 3 times
Service B retries 3 times
```

Potential downstream calls:

```text
3 × 3 × 3 = 27 attempts
```

Mitigations:

- Retry at one layer
- Retry budget
- Attempt limit
- Deadline propagation
- Circuit breaker
- Load shedding

## Idempotency and Retries

### Problem

- Write succeeds
- Response lost
- Client retries
- Duplicate side effect

Examples:

- Double charge
- Duplicate order
- Duplicate booking
- Duplicate message

Solutions:

- Idempotency key
- Unique database constraint
- Deduplication record
- Return stored result

**Core principle:** Retries are safe only when the operation is idempotent or deduplicated.

## Circuit Breaker

**Purpose:** Stop sending requests to a repeatedly failing dependency.

### States

1. **Closed**
2. **Open**
3. **Half-open**

### Closed

- Requests allowed
- Failures monitored
- Threshold not reached

### Open

- Requests rejected immediately
- Dependency not called
- Recovery time allowed
- Fallback may be returned

### Half-Open

- Limited test requests
- Success → close circuit
- Failure → reopen circuit

### Benefits

- Fail fast
- Reduce dependency load
- Prevent cascading failure
- Protect caller resources
- Allow recovery
- Enable fallback behavior

### Apply To

- Third-party APIs
- Database access
- Internal service calls
- Expensive operations
- Slow dependencies
- Remote storage

### Does Not Replace

- Timeout
- Retry policy
- Rate limiting
- Bulkhead isolation
- Health checks

## Cascading Failures

1. Database slows down
2. Requests time out
3. Clients retry
4. Database receives more traffic
5. Application queues grow
6. Threads and connections exhaust
7. Healthy services fail
8. Entire system degrades

Mitigations:

- Timeouts
- Retry limits
- Exponential backoff
- Jitter
- Circuit breakers
- Load shedding
- Queue bounds
- Bulkheads
- Graceful degradation

## Load Shedding

**Definition:** Reject excess work to preserve system health.

Possible responses:

- `429 Too Many Requests`
- `503 Service Unavailable`
- Reduced-quality response
- Cached response
- Drop low-priority work

**Principle:** Serving fewer requests successfully is better than failing every request.

## Bulkhead Isolation

- Separate thread pools
- Separate connection pools
- Separate queues
- Separate resource limits
- Separate tenant quotas

Benefits:

- Failure containment
- Critical traffic protected
- Noisy-neighbor isolation

## Graceful Degradation

Examples:

- Serve stale cache
- Disable recommendations
- Hide live counts
- Use default image
- Queue non-critical write
- Return read-only mode

**Goal:** Preserve essential functionality while dependencies are degraded.

## Networking Decision Framework

1. **Identify communication pattern**
2. **Choose application protocol**
3. **Choose transport behavior**
4. **Choose load-balancing model**
5. **Address connection state**
6. **Address geographic latency**
7. **Address failures**

### Communication Patterns

- Request-response
- One-way stream
- Bidirectional stream
- Peer-to-peer
- Internal service call

### Application Protocols

- HTTP or REST
- GraphQL
- gRPC
- SSE
- WebSocket
- WebRTC

### Transport Choices

- Reliable ordered stream → TCP
- Low-latency loss-tolerant data → UDP
- Modern multiplexed web transport → QUIC

### Load-Balancing Choices

- Client-side
- L4
- L7
- DNS or global traffic manager

### Connection State

- Short-lived request
- Keep-alive
- Persistent connection
- Session affinity
- Reconnection
- Graceful draining

### Geographic Latency

- CDN
- Regional services
- Regional data
- Global routing
- Replication

### Failure Handling

- Timeout
- Retry budget
- Backoff
- Jitter
- Idempotency
- Circuit breaker
- Load shedding

## Protocol Selection Examples

### Standard Web API

```text
HTTPS + REST + TCP
L7 load balancer
```

### Internal High-Performance Service

```text
gRPC + HTTP/2 + TCP
Client-side or L7 load balancing
```

### Chat Application

```text
REST for history and authentication
WebSocket for live messages
L4 or WebSocket-aware L7 load balancer
```

### Notification Stream

```text
REST for normal operations
SSE for server push
L7 load balancer with streaming support
```

### Video Call

```text
HTTPS or WebSocket signaling
WebRTC media transport
STUN for direct connectivity
TURN as relay fallback
```

### Live Telemetry

```text
UDP when loss acceptable
TCP or QUIC when delivery required
Regional collectors
```

## Networking Interview Template

> I’ll use HTTPS for client-facing request-response traffic, with an L7 load balancer routing requests to stateless application servers. Internal high-volume calls can use gRPC if serialization or network overhead becomes significant. I’ll use explicit timeouts and limited retries with exponential backoff and jitter. Retry-sensitive writes require idempotency keys. Static content will be served through a CDN, while application and database components will be colocated by region to reduce latency. For persistent bidirectional updates, I’ll use WebSockets and route each connection once through an L4 or WebSocket-aware L7 load balancer. If a backend fails, the client reconnects and restores its subscriptions.

## Common Networking Mistakes

### Assuming the Network Is Reliable

- No timeout
- Infinite waiting
- No failure path

### Retrying Every Failure

- Retry amplification
- Cascading overload
- Duplicate writes

### Using UDP Without Application Recovery

- Lost messages
- Reordered state
- No congestion handling

### Opening New TCP Connection Per Request

- Handshake overhead
- Socket churn
- Higher latency

### Using WebSocket for Occasional Updates

- Unnecessary state
- Expensive connections
- Operational complexity

### Assuming WebSockets Require L4

- Ignores WebSocket-aware L7 proxies
- Loses path and header routing
- Oversimplifies connection handling

### Balancing Individual WebSocket Messages

- Breaks connection affinity
- Threatens ordering
- Multiplies backend connections
- Complicates state

### Using Client-Side Balancing for Uncontrolled Clients

- Stale routing metadata
- Complex client behavior
- Slow membership updates

### Ignoring Physical Distance

- Cross-region database calls
- Poor p99 latency
- Unnecessary network cost

### Deep Health Checks

- Dependency cascade
- Expensive probes
- False unavailability

## Important Terms

- **OSI Model:** Layered abstraction for network communication
- **IP:** Network-layer addressing and routing protocol
- **Packet:** Network-layer unit of transmitted data
- **Port:** Identifier for an application endpoint
- **DNS:** Name-to-address resolution system
- **TTL:** Duration a cached record remains valid
- **TCP:** Reliable ordered transport protocol
- **UDP:** Connectionless best-effort datagram protocol
- **QUIC:** UDP-based reliable multiplexed transport
- **Flow Control:** Protection of receiver capacity
- **Congestion Control:** Protection of network capacity
- **Handshake:** Message exchange establishing connection state
- **Retransmission:** Resending missing data
- **Head-of-Line Blocking:** Later data blocked by missing earlier data
- **HTTP:** Stateless application-layer request-response protocol
- **HTTPS:** HTTP protected by TLS
- **TLS:** Encryption, integrity, and endpoint-authentication protocol
- **REST:** Resource-oriented HTTP API style
- **GraphQL:** Client-defined query protocol
- **gRPC:** Typed RPC framework using HTTP/2 and Protocol Buffers
- **SSE:** One-way server-to-client HTTP event stream
- **WebSocket:** Persistent bidirectional framed connection
- **WebRTC:** Real-time peer-to-peer media and data framework
- **NAT:** Translation between private and public addresses
- **STUN:** Public-address discovery for NAT traversal
- **TURN:** Relay service for failed peer-to-peer connectivity
- **Load Balancer:** Component distributing traffic across servers
- **L4 Load Balancer:** Transport-aware connection router
- **L7 Load Balancer:** Application-aware request or connection proxy
- **Service Discovery:** Mechanism for locating service instances
- **Health Check:** Probe determining backend availability
- **Liveness:** Whether a process should be restarted
- **Readiness:** Whether a process should receive traffic
- **Session Affinity:** Repeated routing to the same backend
- **CDN:** Globally distributed edge cache
- **Data Locality:** Keeping data near computation and users
- **Timeout:** Maximum wait duration
- **Retry:** Reattempt after failure
- **Backoff:** Increasing delay between retries
- **Jitter:** Random variation in retry timing
- **Idempotency:** Duplicate execution without duplicate effect
- **Circuit Breaker:** Pattern stopping calls to failing dependencies
- **Load Shedding:** Rejecting excess traffic to preserve health
- **Bulkhead:** Resource isolation preventing failure spread
- **Cascading Failure:** One failure causing failures elsewhere
- **Connection Draining:** Removing a server without immediately terminating active connections
- **Reconnection Storm:** Many clients reconnecting simultaneously

## 30-Second Revision Summary

- **Main layers?** IP, transport, application
- **IP purpose?** Addressing and routing
- **Default transport?** TCP
- **TCP provides?** Reliability, ordering, and flow control
- **UDP provides?** Low overhead without delivery guarantees
- **QUIC provides?** Reliable multiplexed transport over UDP
- **Default web protocol?** HTTPS
- **Client-facing API?** REST
- **Internal high-performance API?** gRPC
- **One-way real-time push?** SSE
- **Bidirectional real time?** WebSocket
- **Peer-to-peer media?** WebRTC
- **STUN purpose?** Discover public connectivity
- **TURN purpose?** Relay traffic
- **L4 balances?** Transport connections
- **L7 balances?** Application requests or upgraded connections
- **Can L7 support WebSockets?** Yes, if WebSocket-aware
- **WebSocket routing happens when?** During connection establishment
- **Per-message WebSocket balancing?** Normally no
- **Backend WebSocket failure?** Client reconnects
- **Global static content?** CDN
- **Global dynamic system?** Regional services and data
- **Transient failure handling?** Timeout plus limited retry
- **Retry strategy?** Exponential backoff with jitter
- **Retry-sensitive writes?** Idempotency key
- **Failing dependency protection?** Circuit breaker
- **Overload protection?** Load shedding
- **Best principle?** Assume every network call can fail
