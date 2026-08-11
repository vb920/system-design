# Real-Time Updates: System Design Notes

## Core Thesis

**Real-time updates require solving two separate communication problems:**

1. **Client hop:** How does a server deliver an update to a connected client?
2. **Server hop:** How does the server learn that an update occurred elsewhere in the system?

```text
Update source
    ↓
Server-side propagation
    ↓
Connection or endpoint server
    ↓
Client-side protocol
    ↓
Client
```

Examples include:

- Chat messages
- Collaborative document changes
- Driver-location updates
- Stock-price updates
- Live comments
- Auction bids
- Game events
- AI response streaming
- Live operational dashboards

The main design principle is:

> **Start with the simplest mechanism that satisfies the latency and communication requirements. Escalate to more complex infrastructure only when necessary.**

---

# 1. The Problem

Traditional HTTP primarily uses a request-response model:

```text
Client sends request
    ↓
Server processes request
    ↓
Server returns response
    ↓
Request completes
```

This works well when the client knows when it needs data. It becomes less convenient when the server must notify the client immediately after an event.

Consider a collaborative editor. When one user types a character, other viewers should see it almost immediately. Having every client ask the server for updates every few milliseconds would create excessive traffic, CPU load, connection overhead, and database reads.

A real-time system therefore needs an efficient way to:

- Keep communication channels available.
- Find the server holding a user's connection.
- Propagate events from producers to connection servers.
- Recover missed events after disconnection.
- Maintain acceptable ordering and delivery guarantees.
- Scale to many concurrent connections and high fan-out.

---

# 2. Two Hops for Real-Time Updates

## Hop 1: Server to Client

How does the server deliver an update to the user's device?

Common choices:

1. Simple polling
2. Long polling
3. Server-Sent Events, or SSE
4. WebSockets
5. WebRTC

## Hop 2: Update Source to Connection Server

How does the server connected to the client discover that an update happened?

Common choices:

1. Pulling from storage through polling
2. Routing through hashing or consistent hashing
3. Broadcasting through pub/sub

These two hops solve different problems and should be discussed separately in a system design interview.

---

# 3. Networking Foundations

## 3.1 Network Layer, Layer 3

The network layer is primarily associated with IP.

Its responsibilities include:

- Addressing hosts
- Routing packets between networks
- Forwarding packets toward a destination
- Best-effort delivery

IP does not guarantee that packets will arrive, arrive once, or arrive in order.

## 3.2 Transport Layer, Layer 4

### TCP

TCP is connection-oriented.

Before application data is exchanged, the endpoints establish a connection. TCP provides:

- Reliable delivery
- Ordered byte delivery
- Retransmission of lost data
- Congestion control
- Flow control

The trade-off is that a connection takes time and consumes state at both endpoints.

### UDP

UDP is connectionless.

It provides datagrams without guaranteeing:

- Delivery
- Ordering
- Deduplication
- Retransmission

It has lower protocol overhead, but reliability must be handled elsewhere when required.

## 3.3 Application Layer, Layer 7

Application-layer protocols build on transport protocols and define application communication semantics.

Examples include:

- HTTP
- WebSocket
- DNS
- WebRTC signaling and media protocols

---

# 4. Simplified HTTP Request Lifecycle

When a user opens a website, the process may include:

## Step 1: DNS Resolution

```text
example.com → IP address
```

DNS resolves a human-readable hostname into a network address.

## Step 2: TCP Connection Establishment

TCP commonly uses a three-way handshake:

```text
Client                         Server
  | -------- SYN ------------> |
  | <------ SYN-ACK ---------- |
  | -------- ACK ------------> |
```

## Step 3: HTTP Request

```http
GET /resource HTTP/1.1
Host: example.com
```

## Step 4: Server Processing

The server authenticates the request, executes application logic, reads required data, and prepares a response.

## Step 5: HTTP Response

The server sends status, headers, and content to the client.

## Step 6: Connection Reuse or Teardown

The connection may be reused through keep-alive. Otherwise, TCP teardown takes place.

### Interview-Relevant Consequences

1. Every additional network round trip adds latency.
2. A persistent connection consumes state and resources.
3. Reusing a connection avoids repeated setup overhead.
4. Long-lived connections affect deployment, scaling, and load balancing.

---

# 5. Load Balancers

## 5.1 Layer 4 Load Balancer

A Layer 4 load balancer routes using transport-level information such as:

- Source and destination IP addresses
- Source and destination ports
- TCP or UDP protocol information

It does not need to understand HTTP paths, headers, or cookies.

### Characteristics

- Fast and efficient
- Minimal application inspection
- Naturally preserves a selected TCP flow
- Useful for persistent, connection-oriented traffic
- Cannot make rich decisions based on HTTP content

### Mental Model

```text
Client
    ↓ persistent TCP flow
L4 load balancer
    ↓ same selected backend for that flow
Connection server
```

An L4 load balancer is often a natural fit for long-lived WebSocket connections.

## 5.2 Layer 7 Load Balancer

A Layer 7 load balancer understands application protocols such as HTTP.

It can route based on:

- URL path
- Hostname
- Headers
- Cookies
- Authentication information
- Request method

### Characteristics

- Terminates a client-side connection
- Usually establishes or reuses a separate backend connection
- Supports content-aware routing
- Provides greater flexibility
- Performs more processing than an L4 balancer

### Example Rules

```text
/api/*     → API servers
/static/*  → static-content servers
/socket/*  → WebSocket-capable connection servers
```

Some L7 balancers explicitly support WebSocket upgrades and streaming responses. Support, idle timeouts, buffering behavior, and connection draining must still be verified.

---

# 6. Simple Polling

## Definition

The client sends a request at a fixed interval to check for new information.

```text
Client  ---- request ----> Server
Client  <--- response ---- Server
        wait 2 seconds
Client  ---- request ----> Server
```

## Example

```javascript
async function poll() {
  const response = await fetch('/api/updates');
  const data = await response.json();
  processData(data);
}

setInterval(poll, 2000);
```

A better API normally asks only for changes after a cursor or sequence number:

```http
GET /api/messages?after=9182
```

## Advantages

- Very simple
- Stateless request handling
- Works with ordinary HTTP infrastructure
- Easy to cache, monitor, and debug
- Easy to explain in an interview
- No persistent connection registry required

## Disadvantages

- Update latency can approach the polling interval
- Many requests return no new data
- High bandwidth and request overhead at scale
- Frequent database reads
- Aggressive polling can overload infrastructure

## Capacity Example

If one million clients poll every ten seconds:

```text
1,000,000 / 10 = 100,000 requests per second
```

This load exists even when there are no updates.

## When to Use

Use simple polling when:

- A delay of a few seconds is acceptable.
- Updates are not central to the product experience.
- Simplicity is more important than immediate delivery.
- The real-time window is short.
- You want a reliable baseline before optimizing.

## Optimization

Use HTTP keep-alive so that repeated polls can reuse an existing TCP connection rather than establishing a new connection for every request.

---

# 7. Long Polling

## Definition

The client sends an HTTP request, and the server keeps it open until:

- New data becomes available, or
- A timeout occurs.

After receiving the response, the client immediately opens another request.

```text
1. Client sends request
2. Server waits
3. Update occurs
4. Server responds
5. Client immediately reconnects
6. Process repeats
```

## Client Example

```javascript
async function longPoll() {
  while (true) {
    try {
      const response = await fetch('/api/updates');
      const data = await response.json();
      processData(data);
    } catch (error) {
      console.error(error);
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}
```

## Important Latency Limitation

Suppose two updates occur almost together. The first is returned through the current request. The second may need to wait for:

1. The first response to arrive.
2. The client to create a new request.
3. The new request to reach the server.
4. The second response to return.

This reconnection gap makes long polling less suitable for frequent updates.

## Advantages

- Uses standard HTTP
- Easy to implement
- Near-real-time delivery for infrequent events
- No specialized browser protocol required
- Simpler infrastructure than WebSockets

## Disadvantages

- Repeated HTTP overhead
- A request exists per waiting client
- Reconnection gaps add latency
- Less efficient for frequent updates
- Proxy and load-balancer timeouts must align
- Long-running requests complicate monitoring
- Browser connection limits may matter

## When to Use

Long polling is useful for:

- Infrequent notifications
- Payment-status completion
- Background-job completion
- Near-real-time features where simplicity matters

## Operational Detail

All layers must use compatible timeouts:

```text
Client timeout
    ≥ load-balancer timeout
    ≥ application waiting interval
```

In practice, the server commonly returns an empty response after a bounded period so the client can reconnect cleanly.

---

# 8. Server-Sent Events

## Definition

Server-Sent Events provide a persistent, one-way event stream from server to client over HTTP.

```text
Client establishes stream
    ↓
Server sends event 1
    ↓
Server keeps connection open
    ↓
Server sends event 2
    ↓
Server keeps connection open
```

The browser exposes SSE through `EventSource`.

## Client Example

```javascript
const eventSource = new EventSource('/api/updates');

eventSource.onmessage = event => {
  const data = JSON.parse(event.data);
  updateUI(data);
};
```

## Server Example

```javascript
app.get('/api/updates', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  const sendUpdate = data => {
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };

  dataSource.on('update', sendUpdate);

  req.on('close', () => {
    dataSource.off('update', sendUpdate);
  });
});
```

## SSE Event Format

An event can include:

```text
id: 9183
event: message
data: {"text":"hello"}

```

The blank line terminates the event.

## Reconnection

SSE clients can reconnect automatically. An event ID allows the server to resume after the last acknowledged event.

```text
Last received event ID: 9183
Reconnect
Request events after 9183
```

This requires the server to retain replayable events or fetch them from durable storage.

## Advantages

- Simple one-way streaming
- Browser-native API
- Automatic reconnection
- Less per-message overhead than long polling
- Works over HTTP
- Suitable for high-frequency server-to-client streams

## Disadvantages

- Server-to-client only
- Client writes require separate HTTP requests
- Proxies may buffer streaming responses
- Long-lived streams affect observability and timeouts
- Browser connection limits can matter, especially without multiplexing
- Infrastructure must support streaming correctly

## When to Use

SSE is a strong choice for:

- AI token streaming
- Live dashboards
- Build and deployment logs
- Notification feeds
- News or score updates
- Progress streams
- Any primarily one-way real-time flow

## Important Design Insight

One-way streaming does not prevent the client from writing. A common architecture is:

```text
Server → client updates: SSE
Client → server commands: HTTP POST or PUT
```

This is often simpler than adopting WebSockets.

---

# 9. WebSockets

## Definition

WebSockets provide a persistent, full-duplex channel between client and server.

Both sides can send messages at any time:

```text
Client <================> Server
       bidirectional
```

## Connection Establishment

A WebSocket connection begins with an HTTP request that asks to upgrade the protocol.

```text
1. Client sends HTTP upgrade request
2. Server accepts the upgrade
3. The TCP connection becomes a WebSocket channel
4. Either side can send framed messages
```

## Client Example

```javascript
function connectWebSocket() {
  const ws = new WebSocket('wss://api.example.com/socket');

  ws.onmessage = event => {
    const data = JSON.parse(event.data);
    handleUpdate(data);
  };

  ws.onclose = () => {
    setTimeout(connectWebSocket, 1000);
  };
}

connectWebSocket();
```

## Server Example

```javascript
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', ws => {
  ws.on('message', message => {
    const data = JSON.parse(message);
    processMessage(data);
  });

  dataSource.on('update', data => {
    ws.send(JSON.stringify(data));
  });
});
```

## Advantages

- Full-duplex communication
- Low per-message overhead
- Efficient for frequent messages
- Good browser support
- Suitable for interactive applications

## Disadvantages

- Stateful long-running connections
- More complex scaling and deployment
- Requires reconnection and replay logic
- Infrastructure must support upgrades and long idle periods
- Load may become uneven across servers
- Connection servers must manage backpressure and slow clients

## When to Use

Use WebSockets when the application requires frequent communication in both directions:

- Chat
- Multiplayer games
- Collaborative editing
- Interactive trading screens
- Live auction bids
- Presence and typing indicators

Avoid choosing WebSockets only because the word “real-time” appears. If the client primarily receives data, SSE plus normal HTTP writes may be simpler.

---

# 10. WebSocket Operational Challenges

## 10.1 Reconnection

Connections fail because of:

- Mobile-network changes
- Wi-Fi interruptions
- Proxy timeouts
- Server crashes
- Deployments
- Load-balancer changes

Clients should reconnect using bounded exponential backoff with jitter.

```text
1 second
2 seconds
4 seconds
8 seconds
randomized delay
```

The server must support replay from a sequence number or cursor.

## 10.2 Heartbeats

A broken connection is not always detected immediately. The system can send periodic ping/pong or application heartbeat messages.

```text
Server ---- ping ----> Client
Server <--- pong ----- Client
```

If a heartbeat is missed repeatedly, the server closes and cleans up the connection.

## 10.3 Deployments

A practical deployment strategy is connection draining:

1. Stop assigning new connections to the old server.
2. Allow existing connections to continue briefly.
3. Ask clients to reconnect or close connections gradually.
4. Let clients reconnect to new instances.
5. Replay missed events.

Trying to transfer a live socket between application processes is usually much more complex.

## 10.4 Load Balancing

Because connections are long-lived, round-robin assignment can become uneven over time.

A **least-connections** strategy often performs better:

```text
New connection → server with the fewest active connections
```

Connection count is not always equal to resource usage. A busy chat connection may consume more bandwidth than an idle connection, so advanced systems may also consider CPU, outbound queue size, and message rate.

## 10.5 Dedicated Connection Service

A common architecture terminates WebSockets in a specialized connection service:

```text
Clients
    ↓
L4 or WebSocket-aware load balancer
    ↓
WebSocket gateway or connection service
    ↓
Stateless application services
    ↓
Databases and event systems
```

The gateway manages:

- Socket lifecycles
- Authentication
- Heartbeats
- Connection registries
- Rate limits
- Backpressure
- Reconnection metadata
- Event forwarding

Application services remain stateless and can scale independently.

---

# 11. WebRTC

## Definition

WebRTC enables direct peer-to-peer media and data communication, commonly between browsers or native clients.

It is particularly useful for:

- Audio calls
- Video calls
- Screen sharing
- Peer-to-peer data channels
- Some collaborative or gaming scenarios

## Main Components

### Signaling Server

Peers first exchange connection information through a signaling server.

Signaling is not fully prescribed by WebRTC. It can use:

- WebSockets
- SSE plus HTTP writes
- Long polling
- Another application-specific channel

### ICE Candidates

Peers exchange possible network paths, called ICE candidates, and attempt to find a working route.

### STUN

STUN helps a client discover its publicly visible address and port and assists with NAT traversal.

### TURN

When a direct peer-to-peer path cannot be established, TURN relays traffic through a server.

```text
Direct path succeeds:
Client A <===========> Client B

Direct path fails:
Client A → TURN relay → Client B
```

TURN increases server bandwidth cost because media or data passes through the relay.

## Simplified Setup Flow

```text
1. Peers contact signaling service
2. Peers exchange offers, answers, and ICE candidates
3. STUN helps discover reachable addresses
4. Peers try to establish a direct path
5. TURN is used as a fallback
6. Media or data flows
```

## Example

```javascript
async function startCall() {
  const pc = new RTCPeerConnection();

  const stream = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true
  });

  stream.getTracks().forEach(track => {
    pc.addTrack(track, stream);
  });

  const offer = await pc.createOffer();
  await pc.setLocalDescription(offer);
  signalingServer.send(offer);
}
```

## Advantages

- Direct peer communication when possible
- Low latency
- Native audio and video support
- Can reduce central-server bandwidth

## Disadvantages

- More complex than WebSockets
- Requires signaling
- NAT and firewall complications
- TURN may still centralize traffic
- Connection establishment takes time
- Peer-to-peer meshes scale poorly for large groups

## When to Use

Use WebRTC when peer-to-peer media or direct data transfer is central to the product. It is usually excessive for ordinary notifications or chat delivery.

---

# 12. Client Protocol Selection

## Decision Flow

```text
Are updates latency-sensitive?
│
├── No → Simple polling
│
└── Yes
    │
    ├── Mostly server-to-client?
    │   ├── Infrequent → Long polling may be sufficient
    │   └── Frequent → SSE
    │
    └── Frequent communication in both directions?
        ├── Client-server → WebSocket
        └── Peer-to-peer media/data → WebRTC
```

## Comparison Summary

### Simple Polling

- Direction: client requests, server responds
- Connection: short or reused HTTP requests
- Best for: low urgency and simplicity
- Main cost: wasteful empty requests and bounded delay

### Long Polling

- Direction: client request, delayed server response
- Connection: one waiting HTTP request at a time
- Best for: infrequent near-real-time events
- Main cost: HTTP churn and reconnection gaps

### SSE

- Direction: server to client
- Connection: persistent HTTP stream
- Best for: one-way event streams
- Main cost: infrastructure buffering and one-way semantics

### WebSocket

- Direction: full duplex
- Connection: persistent upgraded connection
- Best for: frequent interactive communication
- Main cost: connection state and operational complexity

### WebRTC

- Direction: peer to peer
- Connection: direct when possible, TURN fallback otherwise
- Best for: media and peer data
- Main cost: signaling and NAT traversal complexity

---

# 13. Server-Side Push and Pull

After selecting the client protocol, the next question is:

> **How does the connection server learn that an event occurred?**

Example:

```text
User A sends a chat message
    ↓
Message service persists it
    ↓
Which server is connected to User B?
    ↓
Deliver message to User B
```

Three common approaches are:

1. Pulling through polling
2. Pushing through hashing or consistent hashing
3. Pushing through pub/sub

---

# 14. Pulling Through Polling

## Architecture

```text
Update source
    ↓ writes update
Database
    ↑ repeated query
Server or client
```

For a chat application, a query may ask:

```sql
SELECT *
FROM messages
WHERE room_id = ?
  AND sequence_number > ?
ORDER BY sequence_number;
```

The poll acts as the trigger. The update may have happened earlier, but the system discovers it during the next query.

## Advantages

- Very simple
- Durable state remains in the database
- Producer and consumer are decoupled
- No connection-routing infrastructure

## Disadvantages

- Higher latency
- Repeated database load
- Wasteful when updates are rare
- Read volume grows with the number of clients

## When to Use

Use pull-based polling when responsiveness is useful but not the main product requirement.

---

# 15. Routing With Simple Hashing

Persistent connections create a routing problem:

```text
User C is connected somewhere.
Which server owns User C's connection?
```

One solution assigns each user to a predictable server.

For `N` servers:

```text
serverIndex = hash(userId) mod N
```

## Connection Flow

```text
1. Client contacts the service
2. Service computes the responsible server
3. Client is redirected or routed to that server
4. Server stores userId → connection mapping
```

## Delivery Flow

```text
1. Update service computes hash(userId) mod N
2. It sends the event to the responsible connection server
3. Connection server finds the user's socket
4. Connection server forwards the event
```

A coordination service such as ZooKeeper or etcd may maintain the active server list and assignment metadata.

## Limitation

If the number of servers changes from `N` to `N + 1`, many hash assignments change. This can force a large fraction of clients to reconnect.

---

# 16. Consistent Hashing

## Core Idea

Consistent hashing maps both users and servers onto a logical ring.

```text
hash ring:
0 -------------------------------- maximum hash
```

Each user is assigned to the next server encountered clockwise on the ring.

When a server is added or removed, only users in the affected portion of the ring need to move.

## Why It Helps

With ordinary modulo hashing:

```text
Change N → many assignments change
```

With consistent hashing:

```text
Add or remove one server → limited range of assignments changes
```

Virtual nodes are commonly used to improve distribution by placing each physical server at multiple positions on the ring.

## Advantages

- Predictable ownership
- Limited connection movement during scaling
- Useful for stateful connection servers
- Associated document or session state can remain localized

## Disadvantages

- More complex coordination
- Server failure still drops its live connections
- Routing metadata must remain consistent
- Hot users or documents can create hotspots
- Rebalancing needs careful orchestration

## Scaling Procedure

A safe transition may include:

1. Record old and new assignments.
2. Start the new server.
3. Route new connections according to the new ring.
4. Gradually disconnect affected old connections.
5. Let clients reconnect to their new owner.
6. Temporarily send events to old and new owners if required.
7. Complete the transition and remove old metadata.

## When to Use

Use consistent hashing when:

- Connections carry substantial server-side state.
- A document, room, game, or session should have stable ownership.
- Reconstructing state is expensive.
- Dynamic scaling should minimize movement.

If endpoint servers only forward small messages and hold little state, pub/sub is often simpler.

---

# 17. Pub/Sub

## Core Idea

A publisher sends an event to a topic. Subscribers interested in that topic receive the event.

```text
Publisher
    ↓ publish
Topic in pub/sub system
    ↓ fan-out
Subscribed endpoint servers
    ↓
Connected clients
```

Possible technologies include Redis pub/sub, Redis Streams, Kafka, managed messaging services, and dedicated event brokers. Their durability and delivery semantics differ and should not be treated as identical.

## Connection Flow

```text
1. Client connects to any endpoint server
2. Endpoint server authenticates the client
3. Endpoint server maintains topic → local connections mapping
4. Endpoint server subscribes to relevant topics
```

## Delivery Flow

```text
1. Update producer publishes to topic
2. Broker forwards event to subscribed endpoint servers
3. Each endpoint server finds matching local connections
4. Endpoint server sends event to clients
```

For chat, topics may be organized by:

- User
- Conversation
- Room
- Partition of rooms

## Advantages

- Decouples producers from connection servers
- Endpoint servers can remain lightweight
- Efficient fan-out
- Clients can connect to any endpoint server
- Easy horizontal scaling with least-connections balancing

## Disadvantages

- Broker can become a bottleneck or failure domain
- Additional network hop adds latency
- Subscription cardinality may become large
- Many-to-many broker-to-endpoint connections can be expensive
- Ephemeral pub/sub may lose events while a subscriber is offline

## Durability Warning

Pure pub/sub often delivers only to currently connected subscribers. Reconnection recovery requires durable storage, such as:

- A database message table
- A durable log
- Kafka-like retention
- Redis Streams
- A per-user inbox

Therefore, a robust design often uses:

```text
Durable write first
    ↓
Publish notification
    ↓
Push to connected clients
    ↓
Replay from durable store after reconnect
```

## Scaling the Broker

Subscriptions and topics can be partitioned across broker nodes:

```text
partition = hash(topicId) mod partitionCount
```

Endpoint servers may connect to multiple broker partitions, or a routing layer may direct subscriptions to the correct partition.

## When to Use

Use pub/sub when:

- Many events must be broadcast.
- Endpoint servers should remain interchangeable.
- Connection-specific state is small.
- Producers should not know which server owns a user connection.

---

# 18. Consistent Hashing Versus Pub/Sub

## Choose Consistent Hashing When

- A connection is associated with expensive in-memory state.
- One server should own a document, room, or session.
- Locality simplifies ordering and conflict resolution.
- Rebuilding state on another node is costly.

Example:

```text
Document ID → collaboration server
```

The server may hold document state, pending operations, participant state, and conflict-resolution metadata.

## Choose Pub/Sub When

- Endpoint servers are mostly connection forwarders.
- Producers should be decoupled from connection placement.
- Clients may connect to any endpoint server.
- Broadcasting is common.

Example:

```text
Conversation service
    ↓ publish room event
Pub/sub
    ↓
All endpoint servers with participants
```

---

# 19. Reference Architecture

A common scalable architecture is:

```text
Clients
    ↓
Connection-aware load balancer
    ↓
Endpoint or WebSocket gateway fleet
    ↓                         ↑
Pub/sub or event broker ← Update producers
    ↓
Durable event or message store
```

## Gateway Responsibilities

- Authenticate connections
- Track `userId → connection` locally
- Subscribe to relevant topics
- Send heartbeats
- Apply rate limits
- Buffer small bursts
- Disconnect slow consumers
- Accept client commands when using WebSockets

## Application-Service Responsibilities

- Validate commands
- Apply business rules
- Persist durable state
- Assign sequence numbers
- Publish resulting events

## Durable Store Responsibilities

- Preserve messages or operations
- Support replay after reconnection
- Provide idempotency and deduplication data
- Serve historical queries

---

# 20. Failure Handling and Reconnection

## The Problem

Networks and servers fail. A client can appear connected even when one side has already lost the connection. These are sometimes called half-open or zombie connections.

## Detection

Use:

- TCP keepalive where appropriate
- WebSocket ping/pong
- Application heartbeat events
- Idle timeouts
- Last-activity timestamps

## Recovery State

Every event should have a stable identifier or sequence number:

```json
{
  "conversationId": "c42",
  "sequence": 9183,
  "eventId": "evt-abc",
  "type": "message.created",
  "payload": {}
}
```

The client tracks the last applied sequence:

```text
lastAppliedSequence = 9183
```

After reconnection:

```http
GET /conversations/c42/events?after=9183
```

or:

```json
{
  "type": "resume",
  "afterSequence": 9183
}
```

## Delivery Semantics

Exactly-once network delivery is difficult. A practical design often uses:

- At-least-once delivery
- Stable event IDs
- Client-side or server-side deduplication
- Idempotent event application

### Invariant

> **A reconnecting client must be able to determine what it missed without applying the same logical event twice.**

---

# 21. Backpressure and Slow Consumers

A connection may receive events faster than it can transmit or process them.

Without protection:

```text
Producer rate > client consumption rate
    ↓
Outbound queue grows
    ↓
Memory exhaustion
```

Possible policies:

- Bound the outbound queue.
- Drop replaceable events, such as stale cursor positions.
- Coalesce updates.
- Disconnect persistently slow clients.
- Ask the client to resynchronize from durable state.
- Prioritize important messages over transient events.

## Event Classes

### Durable Events

Must not be silently dropped:

- Chat messages
- Document operations
- Purchase-state changes
- Auction bids

### Ephemeral Events

Can often be dropped or replaced:

- Typing indicators
- Mouse positions
- Presence heartbeats
- Intermediate metrics

A useful strategy is:

```text
Latest cursor position replaces older cursor positions.
Chat messages remain durable and ordered.
```

---

# 22. The Celebrity or Massive Fan-Out Problem

Suppose one event must reach millions of clients.

A naive single-node fan-out creates a hotspot:

```text
One publisher → millions of direct sends
```

A hierarchical approach distributes the work:

```text
Root event processor
    ↓
Regional or partitioned broadcast nodes
    ↓
Endpoint servers
    ↓
Clients
```

Useful techniques include:

- Hierarchical fan-out
- Regional distribution
- Topic partitioning
- Batching
- Coalescing frequent updates
- Caching the shared payload once
- Sending references rather than copying large data
- Applying per-client relevance filtering downstream

The system may intentionally trade a small amount of latency for controlled fan-out.

---

# 23. Message Ordering

## Why Ordering Is Difficult

Events traversing different servers or partitions can arrive in a different order from the order in which users generated them.

```text
Event A created first
Event B created second

Network delivers B before A
```

## Practical Solution: Partitioned Ordering

Route all related events through one ordered partition:

```text
partition = hash(conversationId)
```

The partition assigns monotonically increasing sequence numbers:

```text
Conversation c42:
9181, 9182, 9183, 9184
```

This provides an order within the conversation without requiring a global order across all conversations.

## Alternatives

Depending on requirements, systems may use:

- Server timestamps
- Logical clocks
- Lamport clocks
- Vector clocks
- Operational transformation
- CRDTs

For many product system-design interviews, routing one entity's events through a single server or partition is simpler and more appropriate than introducing distributed-clock machinery.

### Ordering Principle

> **Guarantee ordering only within the smallest scope that actually needs it.**

Examples:

- Per conversation
- Per document
- Per auction
- Per game session
- Per user inbox

---

# 24. Common Product Scenarios

## Chat Applications

Typical choice:

```text
WebSocket + pub/sub + durable message store
```

Discuss:

- Message ordering
- Delivery acknowledgements
- Offline inbox
- Reconnection replay
- Typing indicators
- Presence
- Multi-device delivery

## Live Comments

Typical choice:

```text
SSE or WebSocket + partitioned pub/sub + hierarchical fan-out
```

Discuss:

- Large fan-out
- Ranking and filtering
- Batching
- Moderation
- Dropping stale transient events

## Collaborative Editing

Typical choice:

```text
WebSocket + document-based routing + ordered operations
```

Discuss:

- Operational transformation or CRDTs
- Cursor and presence updates
- Per-document ordering
- Reconnection and operation replay
- Expensive in-memory document state

Consistent hashing by document can be useful when a collaboration server holds substantial document state.

## Live Dashboards

Typical choice:

```text
SSE + aggregation service
```

Discuss:

- Refresh granularity
- Server-side aggregation
- Coalescing updates
- Snapshot plus incremental events
- Whether “real time” means milliseconds or several seconds

## AI Response Streaming

Typical choice:

```text
SSE or streamed HTTP response
```

The client primarily receives generated tokens. User prompts can be sent through ordinary HTTP requests.

## Multiplayer Games

Possible choices:

```text
WebSocket for server-authoritative interactions
WebRTC data channel for selected peer-to-peer scenarios
```

Discuss:

- Latency
- Server authority
- Cheating prevention
- State snapshots
- Delta updates
- Different frequencies for critical and cosmetic state

## Audio and Video Calls

Typical choice:

```text
WebRTC + signaling service + STUN/TURN
```

---

# 25. When Not to Use Real-Time Infrastructure

Avoid complex push infrastructure when polling satisfies the product requirement.

Polling may be preferable when:

- A delay of several seconds is acceptable.
- Updates are rare.
- The page is open only briefly.
- The update is not user-visible or urgent.
- Operational simplicity is a priority.

Polling can avoid both complex hops:

```text
No long-lived client connection management
No immediate event routing to a connection server
```

Senior-level design is not about always choosing the most advanced technology. It is about selecting the simplest architecture that satisfies the requirements.

---

# 26. Interview Design Process

## Step 1: Clarify Latency

Ask:

- Is a five-second delay acceptable?
- Is sub-second delivery required?
- Does every event need immediate delivery?

## Step 2: Clarify Direction

Ask:

- Is communication mostly server to client?
- Does the client send frequent messages too?
- Is peer-to-peer communication required?

## Step 3: Clarify Frequency

Ask:

- How many events per second per connection?
- Are updates bursty?
- Can updates be batched or coalesced?

## Step 4: Clarify Durability

Ask:

- Can an event be dropped?
- Must offline users receive it later?
- Is replay required after reconnection?

## Step 5: Clarify Ordering

Ask:

- Is total ordering required?
- Is per-room or per-document ordering enough?
- Can transient events arrive out of order?

## Step 6: Choose the Client Hop

```text
Low urgency → polling
Infrequent near-real-time → long polling
One-way stream → SSE
Frequent bidirectional → WebSocket
Peer media/data → WebRTC
```

## Step 7: Choose the Server Hop

```text
Simple and delay-tolerant → pull from database
Expensive connection-local state → consistent hashing
Lightweight endpoints and fan-out → pub/sub
```

## Step 8: Add Operational Details

Discuss:

- Load balancing
- Heartbeats
- Reconnection
- Replay
- Backpressure
- Deployment draining
- Broker and database failure
- Rate limiting
- Metrics and alerts

---

# 27. Interview Answer Template

> I would first clarify the required update latency, event frequency, communication direction, and durability. If a delay of a few seconds is acceptable, I would start with polling because it minimizes operational complexity. For frequent one-way server updates, I would use SSE. For frequent bidirectional communication, such as chat or collaborative editing, I would use WebSockets terminated by a dedicated connection service. On the backend, I would persist durable events and publish notifications through a pub/sub system so that producers remain decoupled from connection placement. Each event would carry an ID and per-entity sequence number, allowing clients to reconnect, replay missed events, and deduplicate delivery. I would use heartbeats, bounded outbound queues, least-connections load balancing, and connection draining during deployments.

---

# 28. Important Invariants

## Connection Invariant

> **Every active connection has exactly one current owning endpoint server.**

## Durability Invariant

> **An event that must survive disconnection is persisted before the system considers it successfully created.**

## Replay Invariant

> **A reconnecting client can request every event after its last applied sequence or recover from a fresh snapshot.**

## Deduplication Invariant

> **Applying the same event more than once does not corrupt client or server state.**

## Ordering Invariant

> **Events are ordered within the smallest required domain, such as a conversation or document.**

## Backpressure Invariant

> **A slow client cannot cause an unbounded server-side queue.**

## Routing Invariant

> **The system can identify the current connection owner or broadcast through a mechanism that reaches it.**

---

# 29. Common Mistakes

## Choosing WebSockets Too Early

Not every real-time feature requires full-duplex communication.

Better approach:

```text
SSE for updates + HTTP for occasional writes
```

## Ignoring the Second Hop

A WebSocket only solves delivery from a connection server to a client. It does not explain how another service finds that connection server.

## Treating Pub/Sub as Durable Storage

Many pub/sub mechanisms are ephemeral. Persist important events separately or choose a durable stream.

## Ignoring Reconnection

A design is incomplete if users permanently miss events during a brief network failure.

## Ignoring Backpressure

A persistent connection does not imply infinite client capacity.

## Requiring Global Ordering

Global ordering is expensive and often unnecessary. Prefer per-entity ordering.

## Ignoring Infrastructure Timeouts

Load balancers and proxies may close idle streams unless heartbeats and timeout settings are aligned.

## Storing Too Much State in Connection Servers

Large amounts of session state make failures, scaling, and deployments more disruptive. Keep gateways lightweight unless explicit ownership provides an important benefit.

---

# 30. Quick Selection Guide

```text
Simple polling
Use when: a few seconds of delay is acceptable.
Avoid when: request volume or latency requirements are high.

Long polling
Use when: events are infrequent but should arrive quickly.
Avoid when: events are frequent.

SSE
Use when: updates are primarily server to client.
Avoid when: frequent bidirectional messaging is central.

WebSocket
Use when: communication is frequent and bidirectional.
Avoid when: one-way streaming or polling is sufficient.

WebRTC
Use when: peer-to-peer media or data is required.
Avoid when: ordinary server-mediated updates are sufficient.

Consistent hashing
Use when: connection-local or entity-local state is expensive.
Avoid when: lightweight endpoint servers and pub/sub are simpler.

Pub/sub
Use when: producers must reach clients without knowing connection placement.
Avoid relying on ephemeral pub/sub alone for durable delivery.
```

---

# 31. Thirty-Second Revision

- **Two hops:** source to server, then server to client.
- **Polling:** simplest, but delayed and potentially wasteful.
- **Long polling:** server waits before responding; good for infrequent events.
- **SSE:** efficient one-way server-to-client HTTP stream.
- **WebSocket:** persistent full-duplex channel for frequent interaction.
- **WebRTC:** peer-to-peer media or data with signaling and STUN/TURN.
- **L4 load balancer:** routes transport flows and fits persistent connections naturally.
- **L7 load balancer:** understands HTTP and offers content-aware routing.
- **Consistent hashing:** stable ownership when endpoint state is expensive.
- **Pub/sub:** decouples event producers from connection servers.
- **Reconnection:** use event IDs, sequence numbers, and replay.
- **Delivery:** prefer idempotent processing and deduplication.
- **Ordering:** order within a conversation, document, or other entity.
- **Backpressure:** bound queues and treat durable and ephemeral events differently.
- **Default strategy:** start simple and add complexity only when requirements demand it.

## Final Mental Model

```text
Persist important event
    ↓
Assign entity-scoped sequence number
    ↓
Publish notification
    ↓
Route to endpoint server
    ↓
Push through SSE or WebSocket
    ↓
Client applies event idempotently
    ↓
On reconnect, replay after last sequence
```
