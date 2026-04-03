# TinyURL System Design

## System Overview
A URL shortening service that converts long URLs into short aliases (e.g., `tinyurl.com/abc123`) and redirects users to the original URL when the short link is accessed.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Given a long URL, generate a unique short URL
- Redirect short URL to original long URL
- Short URLs should be as short as possible (6–7 characters)
- Optional: custom aliases, URL expiry

### Non-Functional Requirements
- Availability: 99.99% — redirects must always work
- Latency: <5ms for redirect (cache hit); <10ms for DB lookup
- Scalability: 100M+ URLs created, billions of redirects/day
- Uniqueness: no two long URLs map to the same short code (unless intentional)
- Durability: mappings must never be lost

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 100M new URLs created/day
- Read:Write ratio = 100:1 (redirects vastly outnumber creations)
- Average long URL size: 200 bytes
- Short code: 7 characters
- URL retention: indefinite (or configurable TTL)

### Traffic
```
Writes/day  = 100M
Writes/sec  = 100M / 86400 ≈ 1160/sec

Reads/day   = 100M × 100 = 10B
Reads/sec   = 10B / 86400 ≈ 115K/sec
Peak (3×)   ≈ 345K redirects/sec
```

### Storage
```
Per record  = 200B (long URL) + 7B (short code) + metadata ≈ 500B
Records/day = 100M × 500B = 50GB/day
5-year      = 50GB × 365 × 5 ≈ 90TB
```

### Cache
```
80/20 rule: 20% of URLs get 80% of traffic
Hot URLs    = 100M × 0.2 = 20M records
Cache size  = 20M × 500B = 10GB  (very manageable)
```

## 3. Core Components

**Load Balancer** — Round-robin across Encryption/Decryption servers; handles 345K req/sec at peak

**Encryption Server** — Handles URL shortening (write path); generates unique short code using one of the approaches below

**Decryption Server** — Handles redirect (read path); looks up short code → long URL from Redis cache, falls back to DB

**Redis (Cache + Counter)** — Two roles:
- Read path: caches `shortCode → longURL` mappings (2ms lookup)
- Write path (Counter approach): atomic counter for unique ID generation (2–3ms)

**Database** — Persistent store for all `shortCode → longURL` mappings (10ms lookup)

**Zookeeper** — (Zookeeper approach only) Assigns unique worker ID ranges to each Encryption Server instance; ensures no two servers generate the same counter value

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| MySQL / PostgreSQL | Simple key-value-like lookups, ACID for writes, easy to shard by shortCode |
| Redis | Sub-5ms cache for hot URLs; atomic INCR for counter approach |
| Zookeeper | Distributed coordination — assigns ID ranges to servers (not a data store) |

> Cassandra is also a valid choice for the URL mapping table — high read throughput, easy horizontal scale. Trade-off: eventual consistency means a freshly created URL might not be immediately readable on all nodes.

### Database — url_mappings

| Field | Type |
|---|---|
| short_code | VARCHAR(7) (PK) |
| long_url | TEXT |
| created_by | UUID, nullable |
| created_at | TIMESTAMP |
| expires_at | TIMESTAMP, nullable |
| hit_count | BIGINT |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `url:{shortCode}` | String | long URL | 24hr (sliding) |
| `counter` | String | global atomic integer | — (Counter approach) |

## 5. Short Code Generation — The Core Problem

The central challenge: generate a **unique, short, non-guessable** code for every URL, across multiple servers, without collisions.

### How encoding works
Take a numeric ID → encode in Base62 (a–z, A–Z, 0–9):
```
Base62 characters = 62
7-character code  = 62^7 ≈ 3.5 trillion unique codes
                  → enough for billions of URLs
```

Example:
```
ID: 125  →  Base62: "cb"
ID: 2009215674938  →  Base62: "zn9edcu"
```

The problem reduces to: **how do we generate a unique ID per request, across N servers, without duplicates?**

---

## 6. Approach 1 — Redis Counter

### How it works
A single Redis instance holds a global atomic counter. Every Encryption Server calls `INCR counter` to get the next unique ID, then Base62-encodes it into the short code.

```
Server 1: INCR counter → 1001 → Base62 → "QZ"
Server 2: INCR counter → 1002 → Base62 → "QZa"
Server 3: INCR counter → 1003 → Base62 → "QZb"
```

Redis `INCR` is atomic — no two servers ever get the same number.

### Architecture
```
Client → Load Balancer → Encryption Server
                              ↓
                         Redis (INCR counter)  ← 2–3ms
                              ↓
                         Base62 encode → shortCode
                              ↓
                          Database (write)      ← 10ms
```

### Read (Redirect) Path
```
Client → Load Balancer → Decryption Server
                              ↓
                         Redis cache lookup     ← 2ms (hit)
                              ↓ (miss)
                          Database lookup       ← 10ms
                              ↓
                         Populate Redis cache
                              ↓
                         301/302 redirect
```

### Pros
- Simple to implement
- Redis INCR is atomic — guaranteed uniqueness
- 2–3ms to get an ID — very fast
- No coordination infrastructure needed

### Cons
- Redis is a single point of failure for writes — if Redis goes down, no new URLs can be created
- Redis must be highly available (Sentinel/Cluster adds complexity)
- Counter is sequential — short codes are predictable/guessable (`abc123`, `abc124`)
- Global counter becomes a bottleneck at extreme scale (though Redis handles ~100K ops/sec easily)

---

## 7. Approach 2 — Zookeeper + Worker ID (Snowflake-style)

### How it works
Zookeeper assigns each Encryption Server a unique **worker ID** (e.g., 123, 234, 345). Each server maintains its own **local counter** starting from 0. The short code is generated from `workerID + localCounter` — no coordination needed per request.

```
Zookeeper assigns:
  Server 1 → workerID: 123
  Server 2 → workerID: 234
  Server 3 → workerID: 345

Server 1 generates: hash(123, 0) → "aB3xYz"
Server 1 generates: hash(123, 1) → "mK9pQr"
Server 2 generates: hash(234, 0) → "zT2wNv"  ← never collides with Server 1
```

The ID is typically structured as a **Snowflake ID**:
```
| timestamp (41 bits) | workerID (10 bits) | sequence (12 bits) |
→ 63-bit integer → Base62 encode → 7-char short code
```

This gives:
- 2^41 ms of timestamps ≈ 69 years
- 2^10 = 1024 unique worker IDs
- 2^12 = 4096 IDs per ms per worker

### Architecture
```
Zookeeper assigns workerIDs to servers at startup (one-time)

Client → Load Balancer → Encryption Server
                              ↓
                    Generate ID locally:
                    timestamp + workerID + localCounter
                              ↓
                         Base62 encode → shortCode
                              ↓
                          Database (write)      ← 10ms
```

No per-request network call needed — ID generation is purely in-memory on the server.

### Read (Redirect) Path
Same as Redis Counter approach — Redis cache → DB fallback.

### Pros
- No per-request coordination — ID generation is in-memory, sub-millisecond
- Zookeeper only contacted at server startup (or on crash/restart)
- Horizontally scalable — add more servers, Zookeeper assigns new worker IDs
- No single point of failure for ID generation (each server is independent)
- IDs are time-ordered (Snowflake structure) — useful for analytics

### Cons
- Zookeeper adds operational complexity (another distributed system to manage)
- If a server crashes and restarts, it must re-register with Zookeeper to get its workerID back (or get a new one) — brief unavailability
- Clock skew between servers can cause ID collisions in Snowflake — requires NTP sync
- Worker ID space is finite (1024 workers max in standard Snowflake) — sufficient for most cases
- More complex to implement than Redis counter

---

## 8. Approach Comparison

| Dimension | Redis Counter | Zookeeper + Worker ID |
|---|---|---|
| ID generation latency | 2–3ms (Redis round trip) | <1ms (in-memory) |
| Coordination per request | Yes (Redis INCR) | No (only at startup) |
| Single point of failure | Redis (mitigated by Sentinel) | None for ID gen; Zookeeper for registration |
| Predictability of codes | Sequential — guessable | Time-based — harder to guess |
| Operational complexity | Low | Medium (Zookeeper cluster) |
| Scalability ceiling | ~100K writes/sec (Redis limit) | ~4M IDs/sec per worker |
| Clock dependency | None | Yes — NTP sync required |
| Best for | Simpler systems, moderate scale | High scale, low-latency critical |

**When to pick Redis Counter:** simpler ops, moderate traffic, team already uses Redis.

**When to pick Zookeeper:** very high write throughput, latency-sensitive ID generation, already running Zookeeper (e.g., Kafka cluster uses it).

---

## 9. Key Flows

### 9.1 Auth
- API key or JWT per user for creation endpoint
- Redirect endpoint is public — no auth required
- Rate limiting at Load Balancer: cap URL creations per API key (e.g., 1000/min)

### 9.2 URL Shortening (Write)

1. Client sends `POST /shorten` with `{longUrl, customAlias?, expiresAt?}`
2. Load Balancer routes to Encryption Server (round-robin)
3. Validate URL format
4. Check if `longUrl` already exists in DB (optional dedup) — return existing short code if found
5. Generate unique ID via Redis Counter or Zookeeper approach
6. Base62 encode → `shortCode`
7. Write `{shortCode, longUrl, createdAt, expiresAt}` to Database
8. Return `https://tinyurl.com/{shortCode}` to client

### 9.3 Redirect (Read)

1. Client hits `GET /{shortCode}`
2. Load Balancer routes to Decryption Server
3. Check Redis cache: `GET url:{shortCode}` → 2ms
4. Cache hit: return 301/302 redirect to `longUrl`
5. Cache miss: query Database → 10ms
6. If found: populate Redis cache with TTL, return redirect
7. If not found or expired: return 404

**301 vs 302:**
- 301 (Permanent): browser caches redirect — reduces server load but you lose click analytics
- 302 (Temporary): browser always hits server — enables analytics, higher load
- TinyURL uses 301 for performance; analytics-focused services use 302

### 9.4 Custom Alias

1. Client sends `POST /shorten` with `customAlias = "my-link"`
2. Encryption Server checks DB: does `my-link` already exist?
3. If taken: return 409 Conflict
4. If free: write to DB with `shortCode = "my-link"`

### 9.5 URL Expiry

- `expires_at` stored in DB and Redis TTL set to match
- Decryption Server checks `expires_at` on DB lookup
- Expired URLs return 410 Gone
- Background cleanup job deletes expired records from DB periodically

## 10. Key Interview Concepts

### Why Base62 over MD5/SHA256?
MD5/SHA256 of the URL produces a hash — take first 7 chars. Problem: collisions (two different URLs produce same 7-char prefix). You'd need collision detection and retry logic. Base62 encoding of a unique integer has zero collision risk by construction.

### 301 vs 302 Redirect
301 is permanent — browser caches it, never hits your server again. Great for performance, bad for analytics (you can't count clicks). 302 is temporary — browser always asks your server, enabling click tracking. Choose based on whether analytics matter.

### Read-Heavy Caching
100:1 read:write ratio means caching is critical. 20% of URLs drive 80% of traffic (Pareto). A 10GB Redis cache covers the hot set entirely. Cache hit rate >95% means DB sees only 5% of redirect traffic — keeps DB load manageable at 345K req/sec peak.

### Database Sharding
At 90TB over 5 years, a single DB node won't cut it. Shard by `shortCode` (hash-based). All reads and writes for a given short code go to the same shard — no cross-shard joins needed since the access pattern is pure key-value lookup.

### Avoiding Hotspot on Counter
Redis INCR is single-threaded and sequential. At extreme scale, one option is **range-based pre-allocation**: each Encryption Server fetches a range of 1000 IDs from Redis at once (`INCRBY counter 1000`), uses them locally, then fetches the next batch. Reduces Redis calls by 1000×.

### Snowflake ID Structure
```
0 | 41-bit timestamp | 10-bit workerID | 12-bit sequence
```
- Timestamp: ms since epoch → time-sortable
- WorkerID: assigned by Zookeeper → unique per server
- Sequence: local counter reset each ms → 4096 IDs/ms/server
- Total: ~4M IDs/sec across 1024 workers

## 11. Failure Scenarios

### Redis Counter Failure
- Impact: no new short URLs can be created (write path blocked); redirects unaffected (separate Redis cache)
- Recovery: Redis Sentinel promotes replica in <30s; Encryption Servers retry with backoff
- Prevention: Redis Sentinel/Cluster; separate Redis instances for counter vs cache so a cache failure doesn't block writes

### Zookeeper Failure
- Impact: new Encryption Server instances can't register and get a workerID; existing running servers are unaffected (they already have their workerID in memory)
- Recovery: Zookeeper ensemble (3 or 5 nodes) — tolerates minority node failures; servers cache their workerID locally so a brief Zookeeper outage doesn't affect running instances
- Prevention: Zookeeper ensemble with odd node count; servers persist workerID to local disk as fallback

### Encryption Server Crash (Zookeeper approach)
- On restart, server re-registers with Zookeeper
- Gets a new workerID (old one released)
- Local counter resets to 0 — but new workerID ensures no collision with previous IDs
- Brief unavailability during re-registration (<1s)

### Database Failure
- Impact: cache misses can't be resolved; new URL writes fail
- Recovery: DB replica promoted; Redis cache continues serving hot URLs during failover (<30s gap)
- Prevention: primary-replica setup with auto-failover; write-ahead log for durability

### Cache Stampede (Thundering Herd)
- Scenario: a popular URL's cache entry expires → thousands of requests simultaneously hit DB
- Recovery: probabilistic early expiration (refresh cache slightly before TTL expires); mutex lock on cache miss so only one request hits DB, others wait
- Prevention: sliding TTL (reset on each access); jitter on TTL to avoid synchronized expiry

### Hash Collision (if MD5 approach used)
- Not applicable to Base62 + unique counter — collisions are impossible by construction
- This is one of the key reasons to prefer counter + Base62 over hash-based approaches

### Clock Skew (Zookeeper / Snowflake approach)
- Scenario: server clock moves backward → generates ID with older timestamp → potential collision with previously issued ID
- Recovery: server detects backward clock, waits until clock catches up before issuing next ID
- Prevention: NTP sync on all servers; monitor clock drift; alert if drift > 1ms
