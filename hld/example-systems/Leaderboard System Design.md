# Leaderboard System Design

## System Overview
A real-time leaderboard system (think game rankings, competitive coding, fantasy sports) that ingests score events, ranks players globally and within segments, and serves low-latency rank queries at scale.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Submit/update a player's score
- Fetch global leaderboard (top N players)
- Fetch a player's current rank and score
- Fetch players around a given player (±K neighbors)
- Support leaderboards scoped by time window (daily, weekly, all-time)

### Non-Functional Requirements
- Availability: 99.99% — leaderboard reads must always work
- Latency: <10ms for rank reads; <50ms for score updates to reflect in ranking
- Scalability: 100M+ players, millions of score updates/sec during peak events
- Consistency: Eventual consistency acceptable — slight ranking lag is fine
- Durability: Raw scores must never be lost

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 100M active players
- Peak score updates: 5M/sec (live game events, tournaments)
- Leaderboard reads: 50M/sec (much more read-heavy)
- Read:Write ratio = 10:1
- Score event size: ~100 bytes
- Leaderboard query: top 100 players

### Traffic
```
Score writes/sec  = 5M (peak)
Rank reads/sec    = 50M (peak)

Kafka throughput  = 5M × 100B = 500MB/sec ingestion
```

### Storage
```
Score DB record   = ~200B per player
100M players      = 100M × 200B = 20GB  (very small, fits in memory)

Score events/day  = 5M × 86400 = 432B events/day
                  = 432B × 100B = 43TB/day (raw event log, short retention)

Aggregated DB     = time-bucketed scores, ~1TB/year
```

### Memory (Redis ZSET)
```
Per player entry  = ~50B (score + member string)
100M players      = 100M × 50B = 5GB per leaderboard ZSET
Multiple windows  = 5GB × 3 (daily/weekly/all-time) = 15GB  (easily fits in Redis)
```

## 3. Core Components

**API Gateway & Load Balancer** — Authentication, authorization, routing, rate limiting for all client requests

**Score Service** — Receives score update events from clients; validates and publishes to Kafka; does not write to DB directly

**Kafka** — Durable event bus; decouples Score Service from persistence; two consumer groups read from the same score event stream

**DB Consumer Service** — Kafka consumer; persists raw/aggregated scores to Score DB (durable record of truth)

**Redis Consumer Service** — Kafka consumer; updates Redis Sorted Set (ZSET) with latest player scores for real-time ranking

**Ranking Service** — Serves all leaderboard read queries; reads from Redis ZSET for real-time rankings; reads from Aggregated DB for historical/time-windowed rankings

**Redis Sorted Set (ZSET)** — In-memory ranked data structure; O(log N) insert and rank lookup; the real-time leaderboard

**Score DB** — Persistent store for raw player scores; source of truth for recovery and historical queries

**Stream Processing Service** — Consumes score events from Kafka; computes time-windowed aggregations (per-minute, per-hour, daily rollups); writes to Aggregated DB

**Aggregated DB (Time-Series DB — InfluxDB)** — Stores pre-computed score aggregations per time window; used by Ranking Service for daily/weekly leaderboards

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| Redis ZSET | O(log N) rank operations, in-memory speed (<1ms), native sorted set with score |
| Score DB (PostgreSQL / DynamoDB) | Durable raw scores, player lookup by userId, ACID for score updates |
| InfluxDB (Time-Series DB) | Optimized for time-bucketed aggregations, efficient range queries by time window |
| Kafka | High-throughput event ingestion, durable log, fan-out to multiple consumers |

### Score DB — player_scores

| Field | Type |
|---|---|
| player_id | UUID (PK) |
| username | VARCHAR |
| score | BIGINT |
| last_updated | TIMESTAMP |

### Score DB — score_events (raw event log, short retention)

| Field | Type |
|---|---|
| event_id | UUID (PK) |
| player_id | UUID |
| delta | BIGINT (score change, can be negative) |
| game_id | UUID, nullable |
| event_time | TIMESTAMP |

### Aggregated DB (InfluxDB) — score_aggregations

| Field | Type |
|---|---|
| time | TIMESTAMP (index) |
| player_id | TAG (indexed) |
| window | TAG (daily / weekly / all-time) |
| aggregated_score | FLOAT |

### Redis Data Structures

| Key Pattern | Type | Value |
|---|---|---|
| `leaderboard:global` | ZSET | member=playerId, score=totalScore |
| `leaderboard:daily:{YYYY-MM-DD}` | ZSET | member=playerId, score=dailyScore |
| `leaderboard:weekly:{YYYY-WW}` | ZSET | member=playerId, score=weeklyScore |
| `player:score:{playerId}` | String | current score (for quick single-player lookup) |

## 5. Key Flows

### 5.1 Auth

- JWT token validated at API Gateway on every request
- Score submission requires authenticated player token
- Leaderboard reads can be public (no auth) or scoped to authenticated user for "my rank"
- Rate limiting at API Gateway: cap score submissions per player (e.g., 10 updates/sec) to prevent abuse

### 5.2 Score Update Flow

```
Client → API Gateway → Score Service → Kafka
                                          ↓              ↓
                                  DB Consumer     Redis Consumer
                                       ↓                ↓
                                   Score DB        Redis ZSET
                                                  (ZADD player score)
```

1. Player action triggers score event → client sends `POST /score` with `{playerId, delta, gameId}`
2. API Gateway validates JWT, rate limits
3. Score Service validates event (non-negative delta, valid gameId)
4. Publishes event to Kafka topic `score-events`
5. Returns 202 Accepted immediately (async — does not wait for DB write)

**DB Consumer Service:**
6. Consumes from Kafka, updates `player_scores` table: `score = score + delta`
7. Appends to `score_events` log
8. Commits Kafka offset after successful DB write

**Redis Consumer Service:**
9. Consumes from Kafka, executes `ZINCRBY leaderboard:global {delta} {playerId}`
10. Also updates time-windowed ZSETs: `ZINCRBY leaderboard:daily:{today} {delta} {playerId}`
11. Sets TTL on daily/weekly ZSETs to auto-expire after window closes

### 5.3 Stream Processing (Aggregation)

The Stream Processing Service is a stateful aggregation service (e.g., Kafka Streams, Spark Streaming, or a custom consumer with in-memory state):

1. Consumes score events from Kafka continuously
2. Maintains in-memory rolling windows (per minute, per hour, per day)
3. On window close: flushes aggregated scores to InfluxDB
4. InfluxDB stores `(playerId, window, aggregated_score, timestamp)`
5. Used by Ranking Service for historical leaderboards and analytics

### 5.4 Leaderboard Read — Top N Players

1. Client sends `GET /leaderboard?type=global&limit=100`
2. API Gateway routes to Ranking Service
3. Ranking Service executes: `ZREVRANGE leaderboard:global 0 99 WITHSCORES`
4. Redis returns top 100 players with scores in O(log N + 100) time
5. Ranking Service enriches with usernames (from cache or Score DB)
6. Returns ranked list to client

### 5.5 Player Rank Lookup

1. Client sends `GET /rank/{playerId}`
2. Ranking Service executes: `ZREVRANK leaderboard:global {playerId}`
3. Redis returns 0-indexed rank in O(log N)
4. Also fetch score: `ZSCORE leaderboard:global {playerId}`
5. Return `{rank: N+1, score: X}` to client

### 5.6 Players Around Me (±K Neighbors)

1. Client sends `GET /leaderboard/around/{playerId}?window=5`
2. Ranking Service gets player's rank: `ZREVRANK leaderboard:global {playerId}` → rank R
3. Fetch range: `ZREVRANGE leaderboard:global (R-5) (R+5) WITHSCORES`
4. Returns 11 players centered on the requesting player

### 5.7 Time-Windowed Leaderboard (Daily / Weekly)

**Real-time (today's leaderboard):**
- Ranking Service reads from `leaderboard:daily:{today}` ZSET — same O(log N) operations
- ZSET has TTL set to expire at end of day

**Historical (yesterday, last week):**
- Ranking Service queries InfluxDB: `SELECT player_id, aggregated_score WHERE window='daily' AND time='{date}' ORDER BY score DESC LIMIT 100`
- InfluxDB is optimized for exactly this time-range query pattern

### 5.8 Leaderboard Reset (Daily/Weekly)

- Daily ZSET key includes date: `leaderboard:daily:2024-01-15` — naturally expires via TTL
- No explicit reset needed — new day = new key
- Weekly ZSET: `leaderboard:weekly:2024-W03` — expires after week ends
- All-time ZSET: never expires, continuously updated

## 6. Key Interview Concepts

### Redis Sorted Set (ZSET) — The Core Data Structure
ZSET stores `(member, score)` pairs, always sorted by score. Key operations:
- `ZADD key score member` — insert or update, O(log N)
- `ZINCRBY key delta member` — atomic increment, O(log N)
- `ZREVRANK key member` — get rank (0-indexed from top), O(log N)
- `ZREVRANGE key start stop` — get range by rank, O(log N + M)
- `ZSCORE key member` — get score, O(1)

With 100M players, N = 100M. log₂(100M) ≈ 27 operations — effectively constant time.

Example:
```
ZADD leaderboard:global 5000 "player:alice"
ZADD leaderboard:global 7200 "player:bob"
ZADD leaderboard:global 6100 "player:carol"

ZREVRANGE leaderboard:global 0 2 WITHSCORES
→ 1. bob    7200
   2. carol  6100
   3. alice  5000

ZREVRANK leaderboard:global "player:alice" → 2  (0-indexed, so rank 3)
```

### Why Kafka Between Score Service and Consumers
Score Service publishes and returns 202 immediately — client doesn't wait for DB or Redis writes. Benefits:
- Score Service stays stateless and fast
- DB and Redis consumers can scale independently
- If Redis Consumer crashes, it replays from Kafka — no score lost
- Kafka acts as a buffer during traffic spikes (5M/sec bursts)

### Why Two Separate Consumers (DB + Redis)
DB Consumer and Redis Consumer read the same Kafka topic independently (different consumer groups). This means:
- DB write failure doesn't affect Redis update and vice versa
- Each can scale, retry, and fail independently
- Redis ZSET can be fully rebuilt from Score DB if needed (recovery path)

### Stream Processing Service for Aggregations
Raw score events need to be rolled up into time windows (daily, weekly). A stream processing service maintains in-memory state per window, accumulates deltas, and flushes to InfluxDB on window close. This is more efficient than querying raw events every time a historical leaderboard is requested.

### Time-Series DB for Historical Rankings
InfluxDB is purpose-built for time-stamped data with tag-based indexing. Querying "top 100 players for week of Jan 13" is a simple range + sort query. Doing this on PostgreSQL with 100M rows and billions of events would be slow without heavy pre-aggregation.

### Leaderboard Segmentation
Real systems need multiple leaderboards: global, by region, by game mode, by friend group. Each is a separate ZSET key. Adding a player to multiple leaderboards means multiple `ZINCRBY` calls — still O(log N) each, easily pipelined in Redis.

### Handling Score Corrections / Rollbacks
If a score was submitted fraudulently and needs to be reversed:
1. Publish a negative delta event to Kafka
2. DB Consumer subtracts from Score DB
3. Redis Consumer executes `ZINCRBY leaderboard:global -{delta} {playerId}`
4. ZSET automatically re-sorts

### CAP Trade-off
Leaderboard favors Availability + Partition tolerance (AP). A player seeing rank 1,342 instead of 1,341 for a few seconds is acceptable. Redis ZSET is the fast AP layer; Score DB is the CP source of truth. If Redis and DB diverge, DB wins — Redis can be rebuilt from DB.

### Rebuilding Redis from Score DB
If Redis is wiped (failure, migration):
1. Read all player scores from Score DB
2. Batch `ZADD` into Redis ZSET (pipeline for efficiency)
3. 100M records × 50B = 5GB — loads in minutes
4. During rebuild: serve reads from Score DB (slower but correct)

## 7. Failure Scenarios

### Redis Failure
- Impact: real-time rank reads unavailable; score updates still flow through Kafka → Score DB
- Recovery: Redis Sentinel failover (<30s); if data lost, rebuild ZSET from Score DB (batch ZADD); during rebuild, Ranking Service falls back to Score DB queries
- Prevention: Redis Cluster + AOF persistence; ZSET can be rebuilt so persistence is a performance optimization, not strictly required

### Kafka Consumer Lag (Redis Consumer)
- Impact: Redis ZSET falls behind real scores; rankings slightly stale
- Recovery: consumer catches up automatically; Kafka retains events (configurable retention, e.g., 24hr)
- Prevention: monitor consumer lag; scale up Redis Consumer instances; alert if lag > 5s

### DB Consumer Failure
- Impact: Score DB falls behind; raw scores not persisted yet
- Recovery: Kafka retains events; DB Consumer replays from last committed offset on restart; idempotent upsert prevents duplicates
- Prevention: multiple DB Consumer instances in consumer group; dead letter queue for failed writes

### Score DB Failure
- Impact: new score writes fail to persist; Redis ZSET continues updating (Kafka still flowing)
- Recovery: DB replica promoted; DB Consumer replays Kafka backlog after recovery
- Prevention: primary-replica with auto-failover; synchronous replication for score writes

### Duplicate Score Events
- Scenario: client retries a score submission → same event published twice to Kafka
- Recovery: DB Consumer uses `event_id` as idempotency key — upsert ignores duplicate event_ids; Redis Consumer: `ZINCRBY` is additive, so duplicate delta would inflate score
- Prevention: Score Service deduplicates on `event_id` before publishing to Kafka; client includes idempotency key in request

### Hot Player / Hot Key in Redis
- Scenario: a celebrity player's score is queried millions of times/sec → single Redis key becomes bottleneck
- Recovery: read replicas for Redis; route rank reads to replicas
- Prevention: Redis Cluster shards ZSETs across nodes; for extreme cases, cache top-N results at API Gateway level (TTL 1s)

### InfluxDB Failure
- Impact: historical/time-windowed leaderboard queries fail; real-time leaderboard unaffected
- Recovery: graceful degradation — return error for historical queries; Stream Processing Service buffers aggregations and replays to InfluxDB on recovery
- Prevention: InfluxDB clustering; non-critical path so brief unavailability is acceptable
