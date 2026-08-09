# Caching: Quick Revision Notes

## Core Thesis

**Faster reads and lower database load, with added consistency and failure complexity.**

## Key Insights

- **Cache only for a measured bottleneck**
- **Cache location controls speed and consistency**
- **Cache pattern controls read/write behaviour**
- **High hit rate does not guarantee scalability**
- **Caching shifts complexity to invalidation and failures**

## Types & Classifications

### Cache Locations

#### **External Cache**

- Standalone shared cache
- Examples: Redis, Memcached
- Network-based access
- Shared across application servers
- Supports TTL and eviction
- Scalable and centrally managed
- Default interview choice
- Faster than databases, slower than local memory

#### **CDN Cache**

- Geographically distributed edge cache
- Content stored near users
- Best for images, videos, and static files
- Can also cache HTML and public APIs
- Cache miss → fetch from origin
- Reduces geographic latency
- Origin: **250–300 ms**
- CDN edge: **20–40 ms**
- Similar to read-through caching

#### **Client-Side Cache**

- Data stored near requester
- Browser cache
- `localStorage`
- Mobile-device storage
- Client-library metadata
- Fewer network calls
- Offline support
- Difficult backend invalidation
- Higher stale-data risk

#### **In-Process Cache**

- Cache inside application memory
- No network call
- Faster than Redis
- Separate cache per server instance
- No automatic cross-instance synchronization
- Best for configuration, feature flags, reference data, hot keys, and precomputed values
- Optimization layer, not Redis replacement

## Cache Architectures

### **Cache-Aside**

**Default interview pattern**

#### Flow

1. Check cache
2. Hit → return data
3. Miss → query database
4. Store in cache
5. Return data

#### Characteristics

- Lazy loading
- Only requested data cached
- Simple and widely used
- Higher latency on cache miss
- Application manages caching logic

### **Write-Through**

#### Flow

1. Application writes to cache
2. Cache writes to database
3. Return after both complete

#### Characteristics

- Fresh cached data
- Synchronous writes
- Slower write latency
- Possible cache pollution
- Requires application or framework support
- Still has dual-write risks
- Best for freshness-sensitive reads

### **Write-Behind / Write-Back**

#### Flow

1. Application writes to cache
2. Cache acknowledges quickly
3. Database updated asynchronously

#### Characteristics

- Very fast writes
- High write throughput
- Eventual consistency
- Possible data loss on cache failure
- Suitable for metrics, analytics, and non-critical event data

### **Read-Through**

#### Flow

1. Application queries cache
2. Cache miss occurs
3. Cache fetches from database
4. Cache stores and returns data

#### Characteristics

- Cache as smart proxy
- Centralized loading logic
- Specialized infrastructure required
- Less common than cache-aside
- CDN as common example

## Quick Pattern Comparison

- **Cache-aside:** Application manages cache misses
- **Read-through:** Cache manages cache misses
- **Write-through:** Database updated synchronously
- **Write-behind:** Database updated asynchronously

### Memory Trick

- **Aside:** App handles it
- **Through:** Cache passes it through
- **Behind:** Database update happens later

## Eviction Policies

### **LRU**

- Least Recently Used
- Removes longest-unused item
- Good general-purpose default
- Recent access predicts future access

### **LFU**

- Least Frequently Used
- Removes least popular item
- Uses access-frequency counters
- Good for consistently popular content

### **FIFO**

- First In, First Out
- Removes oldest inserted item
- Simple queue implementation
- Ignores access patterns
- May remove hot data

### **TTL**

- Time To Live
- Per-key expiration
- Controls staleness
- Limits cache lifetime
- Often combined with LRU or LFU
- Not a standalone eviction strategy

## Common Caching Problems

### **Cache Stampede**

#### Problem

- Popular key expires
- Many simultaneous cache misses
- Repeated database queries
- Sudden database overload
- Possible cascading failure

#### Solutions

- Request coalescing
- Single-flight requests
- Cache warming
- Probabilistic early expiration

**Best solution:** One request rebuilds; others wait.

### **Cache Consistency**

#### Problem

- Database contains new value
- Cache contains old value
- Users receive stale data

#### Solutions

- Invalidate cache after database write
- Use short TTL
- Accept eventual consistency
- Choose based on freshness needs

**Key principle:** No perfect solution, only freshness trade-offs.

### **Hot Keys**

#### Problem

- One key receives extreme traffic
- One Redis node or shard overloaded
- High overall hit rate still possible
- Viral content breaks traffic distribution

#### Solutions

- Replicate hot keys
- Load-balance replicas
- Add in-process caching
- Apply rate limiting

### **Cache Failure**

#### Problem

- Redis becomes unavailable
- Requests fall back to database
- Database receives sudden traffic spike
- Possible cascading failure

#### Solutions

- Circuit breakers
- Rate limiting
- Load shedding
- Request coalescing
- Small local fallback cache

## When to Use Caching

### **Read-Heavy Workload**

- Many repeated reads
- High database request volume
- Mostly unchanged data

### **Expensive Queries**

- Multiple joins
- Aggregations
- Personalized feeds
- Repeated computation

### **High Database CPU**

- Repeated identical queries
- Peak-hour overload
- Read processing dominates CPU

### **Strict Latency Requirement**

- Required response below 10 ms
- Database latency 30–50 ms
- Cache latency below 2 ms

## What to Cache

Cache data that is:

- Frequently read
- Rarely changed
- Expensive to query
- Expensive to compute
- Safe to serve slightly stale
- Small enough for memory

Avoid caching data that is:

- Rarely accessed
- Updated constantly
- Extremely consistency-sensitive
- Too large for available memory
- Cheap to retrieve directly

## Interview Answer Flow

1. **Identify bottleneck:** Database load, query latency, expensive computation, or geographic latency
2. **Quantify problem:** Requests per second, query latency, database CPU, and required response time
3. **Choose data:** Frequently read, rarely modified, and expensive to fetch
4. **Define cache keys:** `user:123:profile`, `trending:posts:global`
5. **Choose pattern:** Cache-aside by default; CDN for static media; in-process for extreme hot keys
6. **Manage memory:** LRU, TTL, and capacity limits
7. **Handle invalidation:** Database update, cache deletion, and repopulation on next read
8. **Handle failures:** Redis outage, stampede, hot keys, database overload, and stale data

## Key Numbers

- Database read: **~50 ms**
- Redis read: **~1 ms**
- Approximate improvement: **50×**
- Distant origin request: **250–300 ms**
- Nearby CDN response: **20–40 ms**
- Example API target: **below 10 ms**

## Important Terms

- **Cache:** Fast temporary data store
- **Cache Hit:** Data found in cache
- **Cache Miss:** Data absent from cache
- **Hit Rate:** Percentage of successful cache lookups
- **Origin Server:** Authoritative content server
- **Edge Server:** CDN server near users
- **Eviction:** Removal of cached entries
- **TTL:** Entry expiration time
- **Invalidation:** Removal of stale cached data
- **Eventual Consistency:** Temporary differences followed by convergence
- **Dual-Write Problem:** One update succeeds while another fails
- **Cache Stampede:** Concurrent misses overload the backend
- **Request Coalescing:** One request rebuilds; others wait
- **Cache Warming:** Proactive loading before demand
- **Hot Key:** One key receiving extreme traffic
- **Circuit Breaker:** Stops calls to an unhealthy dependency
- **Cache Pollution:** Unnecessary data consuming cache memory
- **Stale Data:** Outdated cached value

## 30-Second Revision Summary

- **Why cache?** Lower latency and database load
- **Default location?** External Redis cache
- **Default pattern?** Cache-aside
- **Default eviction?** LRU
- **Freshness control?** TTL plus invalidation
- **Main risks?** Stale data, stampede, hot keys, cache failure
- **Stampede fix?** Request coalescing
- **Hot-key fix?** Replication plus local caching
- **Cache failure fix?** Circuit breaker and database protection
- **Interview rule?** Quantify the bottleneck before proposing caching
