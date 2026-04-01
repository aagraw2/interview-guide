
## 1. Why Cache?

Latency comparison (approximate):

```
L1 cache access:          ~1 ns
L2 cache access:          ~4 ns
RAM access:               ~100 ns
Redis (same DC):          ~0.5 ms    (500,000 ns)
Database query (indexed): ~1–10 ms
Database query (scan):    ~100 ms+
Disk read:                ~10 ms
Cross-region network:     ~100–300 ms
```

A cache reduces latency from database range (1–100ms) to sub-millisecond. For a service handling 10,000 RPS, the difference between reading from cache vs DB is the difference between needing 5 DB servers and needing 500.

Caching also **reduces database load** — the most common reason production databases get overwhelmed is an absent or ineffective cache layer.

---

## 2. Where to Cache

Caching exists at every layer of the stack.

```
Client
  │
  ├── Browser cache (HTTP Cache-Control headers)
  │
CDN
  ├── Edge cache (static assets, HTML, API responses)
  │
API Gateway / LB
  ├── Response cache (short TTL for repeated identical queries)
  │
Application Server
  ├── In-process cache (local HashMap, Guava LoadingCache)
  │   Fast, but limited by single server memory, not shared
  │
Distributed Cache
  ├── Redis / Memcached
  │   Shared across all app servers, large, fast
  │
Database
  ├── Query cache (deprecated in MySQL 8+, exists in other DBs)
  ├── Buffer pool (InnoDB caches hot pages in memory)
  │
Storage
  └── OS page cache (kernel-level file caching)
```

In a typical system design, when someone says "add a cache," they usually mean a **distributed cache** (Redis) in front of the database.

---

## 3. Cache Write Policies

These define what happens when you write data. This is one of the most heavily tested caching topics.

### Write-Through

Write to the cache AND the database synchronously. Both are always in sync.

```
Client → Write to Cache → Write to DB → Confirm to Client
```

```
Pros:
  ✅ Cache always consistent with DB
  ✅ No stale reads after writes
  ✅ Good for read-heavy workloads

Cons:
  ❌ Write latency = cache write + DB write
  ❌ Writes infrequently-read data also go to cache (cache bloat)
```

**Use when:** Data is read frequently after being written (user profiles, product details).

---

### Write-Back (Write-Behind)

Write to the cache immediately, write to the database **asynchronously** later.

```
Client → Write to Cache → Confirm to Client
                │
             (async) → Write to DB (after delay or on eviction)
```

```
Pros:
  ✅ Very low write latency (just a cache write)
  ✅ Reduces DB write load (batches writes)
  ✅ Good for write-heavy workloads

Cons:
  ❌ Risk of data loss — if cache crashes before async write, data is lost
  ❌ Cache and DB temporarily out of sync
  ❌ Complex recovery on failure
```

**Use when:** Write throughput matters more than durability (analytics events, counters, logging). Not for financial transactions.

---

### Write-Around

Write directly to the database, bypassing the cache. Cache is only populated on subsequent reads.

```
Client → Write to DB directly (cache not updated)
         Next read → cache miss → read from DB → populate cache
```

```
Pros:
  ✅ Cache isn't polluted with write-once data
  ✅ Useful for large data that won't be re-read soon

Cons:
  ❌ First read after write is always a cache miss (higher latency)
  ❌ Cache can be stale until TTL expires or explicit invalidation
```

**Use when:** Data is written once and rarely re-read immediately (file uploads, batch processing results, logs).

---

## 4. Cache Read Strategies

### Cache-Aside (Lazy Loading)

The application manages the cache directly. Most common pattern.

```
Read request:
  1. Check cache
  2a. Cache HIT  → return cached value
  2b. Cache MISS → query DB → write to cache → return value

Write request:
  1. Write to DB
  2. Invalidate or update cache entry
```

```python
def get_user(user_id):
    user = cache.get(f"user:{user_id}")
    if user is None:                        # cache miss
        user = db.query("SELECT * FROM users WHERE id = ?", user_id)
        cache.set(f"user:{user_id}", user, ttl=3600)
    return user
```

**Pros:** Cache only contains data that's actually been requested. Fault tolerant — if cache is down, app reads from DB. **Cons:** First request always a miss. Potential for stale data if DB is updated without invalidating cache.

---

### Read-Through

The cache sits in front of the DB. Application only talks to the cache.

```
App → Cache → (on miss) → DB → (populate cache) → return
```

Difference from cache-aside: the **cache itself** fetches from DB on a miss, not the application. Abstracts the DB from the app.

**Used by:** Some cache libraries (Guava `LoadingCache`), AWS ElastiCache with DAX.

---

## 5. Eviction Policies

When the cache is full and a new item needs to be added, an existing item must be evicted.

### LRU — Least Recently Used

Evict the item that hasn't been accessed for the longest time.

```
Cache (capacity=3): [A, B, C]

Access D:
  Evict A (least recently used) → [B, C, D]
```

**Implementation:** Doubly linked list + hash map (O(1) get and put). This is a classic interview coding problem.

**Use when:** Temporal locality — recently accessed items are more likely to be accessed again. Good default choice.

---

### LFU — Least Frequently Used

Evict the item that has been accessed the fewest total times.

```
Cache:
  A → accessed 10 times
  B → accessed 3 times
  C → accessed 1 time ← evict this

Access D → evict C
```

**Pros:** Better than LRU for long-lived hot items (they accumulate high frequency and survive). **Cons:** New items have low frequency and get evicted even if they're about to become hot. More complex to implement.

---

### TTL — Time To Live

Evict items after a fixed duration regardless of access pattern.

```
cache.set("user:1", data, ttl=3600)  # expires in 1 hour
```

Usually combined with LRU or LFU. TTL handles data freshness; LRU handles capacity.

---

### Other policies

- **FIFO:** First in, first out. Simple but ignores access patterns.
- **Random:** Evict randomly. Surprisingly effective, used in CPU TLBs.
- **MFU (Most Frequently Used):** Evict the hottest items. Rarely used in practice.

---

## 6. Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

### TTL-based invalidation

Set a time-to-live on every cache entry. After expiry, the cache re-fetches from DB on the next read.

```
Simple, no coordination required
Problem: data can be stale for up to TTL duration
```

### Event-driven invalidation

When data changes in DB, explicitly delete or update the cache entry.

```
User updates profile:
  1. Write to DB
  2. DELETE cache key "user:123"
  Next read → cache miss → re-fetches fresh data
```

**Problem:** Race condition — if two processes update the same key simultaneously, they might write stale data back to cache.

### Write-through invalidation

On every write, update both DB and cache atomically (write-through policy). Always consistent, but adds write latency.

### Cache versioning

Instead of invalidating, change the cache key on updates.

```
v1: cache key = "product:456:v1"
Update → increment version
v2: cache key = "product:456:v2"

Old key becomes orphaned and eventually evicted by TTL
```

---

## 7. Cache Problems: Stampede, Avalanche, Penetration

These three are common interview questions. Know all three.

### Cache Stampede (Thundering Herd)

**Problem:** A popular cache key expires. Thousands of concurrent requests all get a cache miss at the same time and simultaneously query the database.

```
TTL expires for "trending:posts"
→ 10,000 requests/sec all miss cache simultaneously
→ 10,000 DB queries arrive at once
→ DB is overwhelmed
```

**Solutions:**

1. **Mutex/lock:** First request acquires a lock and populates cache; others wait.

```
if cache.get(key) is None:
    with lock(key):
        if cache.get(key) is None:  # double-check
            value = db.query(...)
            cache.set(key, value)
```

2. **Probabilistic early expiration:** Each request has a small chance of refreshing the cache before it actually expires. Prevents the synchronized miss.
    
3. **Background refresh:** A background job refreshes cache before TTL expires, so users never see a miss.
    

---

### Cache Avalanche

**Problem:** Many cache keys expire at the same time (e.g., after a cache restart), causing a flood of DB queries.

```
Cache server restarts → ALL keys expire at once
→ Every request is a cache miss
→ DB gets 100x normal load → DB crashes
```

**Solutions:**

1. **Jitter on TTL:** Add random offset so keys don't all expire together.

```python
ttl = 3600 + random.randint(0, 300)   # 60–65 minutes
cache.set(key, value, ttl=ttl)
```

2. **Staggered cache warm-up:** When restarting, gradually load the cache rather than all at once.
    
3. **Circuit breaker on DB:** If cache miss rate spikes, stop hitting DB and return degraded responses.
    

---

### Cache Penetration

**Problem:** Requests for keys that **don't exist in the cache OR the database** — cache always misses, so every request hits the DB. Common attack vector.

```
Attacker sends requests for user_id = -1, -2, -3, ...
  → cache miss (doesn't exist)
  → DB query (returns nothing)
  → cache miss again on next request (nothing to cache)
  → DB hammered with useless queries
```

**Solutions:**

1. **Cache null values:** If DB returns nothing, cache that fact with a short TTL.

```python
value = db.query(key)
if value is None:
    cache.set(key, "NULL_SENTINEL", ttl=60)  # cache the miss
```

2. **Bloom filter:** Before hitting the cache, check a bloom filter of all valid keys. If the key is definitely not in the DB, reject the request immediately.

```
Request → Bloom filter says "not in DB" → return 404 immediately
Request → Bloom filter says "maybe in DB" → check cache → check DB
```

---

## 8. Distributed Caching

A single cache server has limited memory and is a SPOF. Distributed caches shard data across multiple nodes.

### Sharding strategies

**Hash-based sharding:**

```
node = hash(key) % num_nodes
```

Use consistent hashing to minimize remapping when nodes change.

**Replication:** Redis Cluster replicates each primary shard to one or more replicas. Reads can be served by replicas; writes go to primary.

```
Shard 1: Primary (A) → Replica (A')
Shard 2: Primary (B) → Replica (B')
Shard 3: Primary (C) → Replica (C')
```

### Redis Cluster topology

```
                  ┌───────────────────────────────┐
                  │         Redis Cluster          │
                  │                                │
                  │  [Primary 0]  [Primary 1]  [Primary 2]
                  │    slots 0-5k   slots 5k-10k  slots 10k-16k
                  │       │             │              │
                  │  [Replica 0]  [Replica 1]  [Replica 2]
                  └───────────────────────────────┘
```

16,384 hash slots are divided among primaries. Keys map to slots via `CRC16(key) % 16384`.

---

## 9. Redis vs Memcached

|Feature|Redis|Memcached|
|---|---|---|
|Data structures|Strings, lists, sets, sorted sets, hashes, streams, HyperLogLog|Strings only|
|Persistence|RDB snapshots + AOF log|None|
|Replication|Yes (primary-replica, Cluster)|No|
|Clustering|Yes (Redis Cluster)|Yes (client-side sharding)|
|Pub/Sub|Yes|No|
|Lua scripting|Yes|No|
|Multi-threading|Single-threaded (I/O is async)|Multi-threaded|
|Max value size|512 MB|1 MB|
|Atomic operations|Yes (INCR, SETNX, etc.)|Limited|

**Choose Redis when:** You need persistence, rich data structures (sorted sets for leaderboards, lists for queues), pub/sub, or atomic operations.

**Choose Memcached when:** Pure caching, maximum throughput, horizontal scaling simplicity, and you don't need any Redis features.

In practice, Redis is chosen almost exclusively for new systems today.

---

## 10. Common Interview Questions + Answers

### Q: What cache write policy would you use for a user profile service?

> "Write-through. User profiles are read far more than they're written, so I want the cache to always be warm and consistent after a write. The extra latency of writing to both cache and DB on update is acceptable — profile updates are infrequent. I'd also set a TTL as a safety net."

### Q: How do you handle cache invalidation in a microservices architecture?

> "I'd use event-driven invalidation via a message queue. When Service A updates data, it publishes a `user.updated` event to Kafka. Any service with a cache of that user's data subscribes to the topic and deletes its cache entry. This decouples services and ensures eventual consistency without synchronous coordination."

### Q: What's cache penetration and how do you prevent it?

> "Cache penetration is when requests for non-existent keys bypass the cache on every request and hit the database — typically from buggy clients or malicious actors probing IDs. Prevention: cache null results with a short TTL so repeated lookups for the same missing key hit the cache. For large-scale protection, a bloom filter at the cache layer lets you reject requests for keys provably absent from the DB without a DB lookup at all."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Always specify TTL

Never say "store in cache" without mentioning TTL. Every cache entry should have an expiry. It shows you think about consistency and memory limits.

### ✅ Trick 2: Know the three failure modes

Cache stampede, avalanche, and penetration come up repeatedly. Knowing all three with solutions immediately signals depth.

### ✅ Trick 3: Distinguish what to cache

Not everything should be cached. Cache:

- Expensive queries with high read frequency
- Immutable or slowly changing data
- Computed results (leaderboards, recommendations)

Don't cache:

- Highly personalised data (unless per-user keys)
- Data that changes on every request
- Sensitive data without encryption

### ❌ Pitfall 1: Write-back for financial data

Never use write-back (write-behind) for anything where data loss is unacceptable. If the cache crashes before the async write, the data is gone.

### ❌ Pitfall 2: Forgetting distributed cache has network latency

Redis in the same datacenter is fast (~0.5ms) but still has latency. For extremely hot paths (millions of RPS), consider in-process (L1) cache in front of Redis.

### ❌ Pitfall 3: Assuming cache solves all scaling problems

Cache is a read optimisation. It doesn't help with write-heavy workloads. For write scaling, you need sharding, write queues, or CQRS.

---

## 12. Quick Reference

```
Write policies:
  Write-through    → cache + DB synchronously (consistent, slower writes)
  Write-back       → cache only, async DB (fast writes, data loss risk)
  Write-around     → DB only, cache on next read (avoids cache bloat)

Read strategy:
  Cache-aside      → app manages cache (most common)
  Read-through     → cache fetches from DB on miss

Eviction:
  LRU → evict least recently used (good default)
  LFU → evict least frequently used (better for hot/cold mix)
  TTL → time-based expiry (always combine with above)

Failure modes:
  Stampede    → mass miss on one key expiry → mutex or background refresh
  Avalanche   → mass miss on restart → jitter TTL
  Penetration → missing keys flood DB → cache nulls + bloom filter

Redis vs Memcached:
  Redis   → persistence, data structures, pub/sub, replication
  Memcached → pure cache, multi-threaded, simpler

Cache invalidation strategies:
  TTL        → simple, eventually consistent
  Event-driven → accurate, more complex
  Write-through → always consistent, adds write latency
```