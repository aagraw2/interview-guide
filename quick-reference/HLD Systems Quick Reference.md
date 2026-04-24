# HLD Systems Quick Reference

One-page mental model per system. Core problem, key design decisions, and the concepts the interviewer is actually testing.

---

## Ticket Booking System (BookMyShow)

**Core problem:** Prevent double booking under high concurrency during flash sales.

**The trick:** Two-phase seat locking. Redis `SET NX EX` for the temporary hold (atomic, auto-expiring). MySQL ACID transaction for the durable confirmation. Redis handles the race condition, MySQL handles durability.

**Key concepts:**
- Seat hold: `SET hold:{eventId}:{seatId} userId NX EX 600` — only one user wins
- Payment confirmation: `BEGIN TXN → seat=booked + booking=confirmed → COMMIT`
- Hold expiry: Redis TTL auto-releases + background job syncs MySQL
- Flash sale: API Gateway virtual queue + Redis seat map cache to avoid DB hammering
- Idempotency key on payment: `hash(bookingId + userId)` prevents double charge on retry

**Storage split:** Cassandra for event metadata (read-heavy, no transactions). MySQL for bookings and seat allocations (ACID required). Redis for holds and seat map cache. Elasticsearch for search.

**Interviewer is testing:** Double booking prevention, Redis NX pattern, TTL-based holds, idempotency.

---

## Ecommerce

**Core problem:** Prevent inventory oversell when thousands of users buy the same item simultaneously.

**The trick:** Redis DECR gate at flash sale entry. `DECR inventory:{productId}` — if result < 0, reject. This is the fast path. MySQL is the source of truth for actual inventory.

**Key concepts:**
- Cart: stored in Redis (TTL-based), not DB
- Price snapshot: capture price at checkout time, not at cart-add time
- Inventory oversell: Redis DECR gate + DB-level check with `UPDATE inventory SET qty = qty - 1 WHERE qty > 0`
- Saga pattern: distributed transaction across inventory, payment, order services
- CDC: product catalog changes → Kafka → Elasticsearch for search sync

**Interviewer is testing:** Oversell prevention, saga pattern, cart design, CDC.

---

## Netflix (Video Streaming)

**Core problem:** Deliver high-quality video to millions of concurrent viewers with minimal buffering.

**The trick:** CDN is the primary delivery path, not the origin. Video is pre-encoded into multiple bitrates (ABR). Client picks the right bitrate based on network conditions.

**Key concepts:**
- ABR streaming: HLS/DASH — video split into 2-10s segments at multiple bitrates
- CDN as primary: 95%+ of traffic served from edge, origin only for cache miss
- Video encoding pipeline: upload → S3 → encoding workers (multiple resolutions) → CDN
- Kafka for async: encoding jobs, analytics events, recommendation updates
- Recommendation: offline ML pipeline, not real-time

**Interviewer is testing:** ABR streaming, CDN architecture, async encoding pipeline.

---

## Hotel Reservation

**Core problem:** Prevent double booking for date ranges (not just single slots).

**The trick:** Date range availability check + Redis distributed lock + DB transaction. The lock prevents two users from booking the same room for overlapping dates simultaneously.

**Key concepts:**
- Availability check: `SELECT count(*) FROM bookings WHERE room_id = ? AND dates OVERLAP ?`
- Double booking prevention: Redis lock on `room:{roomId}:{checkIn}:{checkOut}` + DB transaction
- Geo search: Redis GEOADD/GEORADIUS for "hotels near me"
- CDC: hotel metadata → Kafka → Elasticsearch for search

**Interviewer is testing:** Date range overlap queries, distributed locking, geo search.

---

## Food Delivery (Swiggy/Zomato)

**Core problem:** Match drivers to orders in real-time with location tracking.

**The trick:** Redis GEOADD for driver locations. On order placed, query GEORADIUS for nearby available drivers. Assign via distributed lock to prevent two orders going to same driver.

**Key concepts:**
- Driver location: WebSocket updates every 5s → Redis GEOADD
- Driver matching: GEORADIUS query → rank by distance + rating → lock + assign
- Order lifecycle: Kafka event bus (placed → accepted → picked up → delivered)
- Saga pattern: payment + order + driver assignment as distributed transaction
- ETA: pre-computed routes + real-time traffic adjustment

**Interviewer is testing:** Geo-based matching, real-time location tracking, Kafka event bus, saga.

---

## Chat System (WhatsApp)

**Core problem:** Deliver messages in real-time to online users and reliably to offline users.

**The trick:** WebSocket for online delivery. Cassandra for durable storage. On reconnect, client sends last received messageId and server delivers the gap.

**Key concepts:**
- WebSocket registry: `SET ws:user:{userId} {connId, serverId}` in Redis — routes messages to right server instance
- Consistent hashing: same userId always routes to same Chat Service instance
- Message ordering: TIMEUUID as message ID in Cassandra — embeds timestamp + uniqueness
- Offline delivery: message persisted first, then push notification via FCM/APNs
- Group fan-out: Redis Pub/Sub channel per group — publish once, all subscribed instances deliver
- Idempotency: client-generated messageId, server deduplicates on insert

**Interviewer is testing:** WebSocket architecture, consistent hashing, Cassandra partitioning, offline delivery, fan-out.

---

## Job Scheduler

**Core problem:** Execute jobs exactly once at the right time, even with multiple scheduler instances running.

**The trick:** `SELECT FOR UPDATE SKIP LOCKED` — each scheduler instance picks jobs without blocking others. The DB lock prevents two instances from picking the same job.

**Key concepts:**
- Job polling: `SELECT * FROM jobs WHERE status=pending AND scheduled_at <= now FOR UPDATE SKIP LOCKED LIMIT 10`
- Duplicate execution prevention: DB row lock + status transition (pending → running) in same transaction
- Stuck job detection: background job finds `status=running AND updated_at < now - 5min`, resets to pending
- Kafka for async: job results, notifications, retries
- Retry with backoff: `next_retry_at = now + 2^attempt * base_delay`

**Interviewer is testing:** SELECT FOR UPDATE SKIP LOCKED, idempotent job execution, stuck job handling.

---

## Google Docs (Collaborative Editor)

**Core problem:** Multiple users editing the same document simultaneously without conflicts.

**The trick:** Operational Transformation (OT) or CRDT to merge concurrent edits. Redis as the canonical in-memory copy for low-latency reads. Op log as event source for replay and recovery.

**Key concepts:**
- OT: transform operations against each other so they commute — complex but battle-tested
- CRDT: data structure that merges automatically without coordination — simpler, eventually consistent
- Redis as canonical copy: all reads from Redis, writes go to Redis + op log
- Snapshot + op replay: periodic snapshot + replay ops since snapshot for recovery
- Distributed lock: prevent concurrent saves to DB

**Interviewer is testing:** OT vs CRDT trade-offs, event sourcing, Redis as working copy.

---

## Ride Matching (Uber/Ola)

**Core problem:** Match riders to nearby available drivers in real-time.

**The trick:** Redis GEOADD for driver locations. GEORADIUS for nearby drivers. Zookeeper/Redis lock for driver assignment to prevent two riders getting the same driver.

**Key concepts:**
- Driver location: WebSocket updates every 3-5s → `GEOADD drivers:{city} lng lat driverId`
- Matching: `GEORADIUS drivers:{city} lng lat 5 km` → filter available → rank → lock + assign
- Surge pricing: supply/demand ratio per geohash cell, updated every minute
- Trip lifecycle: Kafka events (requested → matched → started → completed)
- ETA: Dijkstra on road graph + real-time traffic

**Interviewer is testing:** Geo-based matching, Redis GEOADD/GEORADIUS, distributed lock for assignment.

---

## Social Media (Twitter/Instagram)

**Core problem:** Fan-out — when a user with 10M followers posts, how do you deliver to all of them?

**The trick:** Hybrid fan-out. Push for regular users (write to followers' feeds on post). Pull for celebrities (too expensive to push to 10M feeds). Merge on read.

**Key concepts:**
- Fan-out on write (push): post → write to each follower's feed in Redis. Fast reads, expensive writes.
- Fan-out on read (pull): post stored once, each user's feed assembled on read. Cheap writes, expensive reads.
- Hybrid: push for users with < 10K followers, pull for celebrities
- Counter accuracy: approximate counters in Redis (INCR), periodic sync to DB
- Content moderation: async pipeline — post stored first, moderation runs in background

**Interviewer is testing:** Fan-out problem, push vs pull vs hybrid, counter design.

---

## Leaderboard

**Core problem:** Maintain a real-time ranked list of millions of users by score.

**The trick:** Redis Sorted Set (ZSET). `ZADD leaderboard score userId`. `ZRANK` for rank. `ZRANGE` for top N. O(log n) for all operations.

**Key concepts:**
- Redis ZSET: `ZADD`, `ZINCRBY`, `ZRANK`, `ZREVRANGE` — all O(log n)
- Two-consumer pattern: one consumer updates Redis ZSET in real-time, another writes to DB for durability
- Hot key problem: global leaderboard is a single Redis key — shard by region or time window
- Stream processing: Kafka → Flink for aggregating scores from events

**Interviewer is testing:** Redis ZSET operations, hot key problem, two-consumer pattern.

---

## Notification System

**Core problem:** Deliver notifications reliably across multiple channels (push, SMS, email) without losing any.

**The trick:** Outbox pattern — write notification to outbox table in same DB transaction as the triggering event. Separate worker reads outbox and delivers. Guarantees at-least-once delivery.

**Key concepts:**
- Outbox pattern: `BEGIN TXN → update business table + insert into outbox → COMMIT` — no dual-write risk
- Three-tier Kafka topics: critical (OTP, payment) → standard (order updates) → promotional (marketing)
- Idempotency: `notificationId` prevents duplicate sends on retry
- Channel routing: user preferences determine push vs SMS vs email
- Rate limiting: per-user per-channel limits to prevent spam

**Interviewer is testing:** Outbox pattern, Kafka topic design, idempotency, at-least-once delivery.

---

## Log Aggregation (Logstash/ELK)

**Core problem:** Ingest billions of log events per day, store efficiently, and make them searchable.

**The trick:** Kafka as the ingestion buffer — absorbs spikes, decouples producers from consumers. Flink for stream processing. Cassandra for primary storage with hot-warm-cold tiering.

**Key concepts:**
- Kafka as buffer: log agents → Kafka topics partitioned by service/host
- Flink stream processing: parse, enrich, aggregate, route
- Cassandra: partition by `(service, date)`, cluster by timestamp — efficient time-range queries
- Hot-warm-cold tiering: recent logs in Cassandra (hot), older in S3 (cold), Elasticsearch for search
- Backpressure: Kafka consumer lag monitoring, auto-scaling consumers

**Interviewer is testing:** Kafka as buffer, stream processing, tiered storage, time-series data modeling.

---

## Google Drive (File Storage)

**Core problem:** Upload large files reliably, sync changes across devices, deduplicate storage.

**The trick:** Chunked upload — split file into 4MB chunks, upload each independently. Content-addressed storage — chunk ID = SHA-256 of content. Same chunk never stored twice.

**Key concepts:**
- 6-step chunked upload: split → hash chunks → check which chunks server already has → upload missing chunks → commit → notify
- Content-addressed chunks: `chunk_id = SHA256(content)` — automatic deduplication
- Delta sync: only upload changed chunks on file update
- `file_version_chunks` table: maps file version to list of chunk IDs
- Client Watcher: file system watcher detects local changes, triggers sync

**Interviewer is testing:** Chunked upload, content-addressed storage, delta sync, deduplication.

---

## Payment System

**Core problem:** Process payments exactly once, never double-charge, handle gateway failures gracefully.

**The trick:** PaymentIntent pattern — create intent first, charge later. Idempotency key on every gateway call. Outbox pattern for reliable event publishing.

**Key concepts:**
- PaymentIntent: create intent (pending) → authorize → capture — separates authorization from charge
- Idempotency key: `hash(orderId + userId + amount)` — gateway deduplicates retries
- PCI Zone: card data never touches your servers — tokenized via payment processor
- Connector pattern: abstract over multiple payment gateways (Stripe, Razorpay, etc.)
- Double-entry ledger: every transaction has debit + credit entries — always balanced
- Outbox: payment events published reliably without dual-write

**Interviewer is testing:** Idempotency, PaymentIntent pattern, double-entry accounting, PCI compliance.

---

## URL Shortener (TinyURL)

**Core problem:** Generate short unique IDs, redirect fast, scale to billions of URLs.

**The trick:** Base62 encoding of a unique ID. Redis cache for hot URLs (most redirects hit cache). DB for persistence.

**Key concepts:**
- ID generation: auto-increment DB ID → Base62 encode (a-z, A-Z, 0-9) → 7 chars = 62^7 = 3.5T URLs
- Or: Snowflake ID / Redis counter for distributed ID generation
- Redirect: `GET /{shortCode}` → Redis lookup → 301/302 redirect
- Cache: 80/20 rule — 20% of URLs get 80% of traffic, cache those
- DB sharding: shard by shortCode hash for write scale

**Interviewer is testing:** ID generation strategies, Base62 encoding, caching, sharding.

---

## Stock Trading System

**Core problem:** Match buy and sell orders fairly and in order, process thousands per second.

**The trick:** Order book — sorted data structure (price-time priority). Price-time priority: best price first, then earliest order at that price.

**Key concepts:**
- Order book: two sorted maps — bids (descending price), asks (ascending price)
- Matching engine: when bid price >= ask price, match occurs
- Exchange Gateway: validates orders, rate limits, routes to matching engine
- Kafka topics: orders → matching engine → trade events → settlement
- Fund reservation: reserve funds on order placement, release on cancel/fill

**Interviewer is testing:** Order book design, price-time priority, Kafka pipeline, fund reservation.

---

## Digital Wallet

**Core problem:** Transfer money between wallets without double-spend or lost funds.

**The trick:** Double-entry bookkeeping — every transfer is two entries (debit source, credit destination) in one transaction. Balance is always sum of all entries.

**Key concepts:**
- Double-entry: `INSERT INTO ledger (account, amount, type)` — debit + credit in same transaction
- Double-spend prevention: `SELECT FOR UPDATE` on source account before debit
- Idempotency: `transferId` prevents duplicate transfers on retry
- Fraud detection: async rule engine on transaction events
- Balance: `SELECT SUM(amount) FROM ledger WHERE account_id = ?` — always accurate

**Interviewer is testing:** Double-entry bookkeeping, SELECT FOR UPDATE, idempotency.

---

## Ad Click Aggregation

**Core problem:** Count billions of ad clicks accurately, handle late-arriving events, detect fraud.

**The trick:** Lambda architecture — speed layer (Kafka + Flink for real-time approximate counts) + batch layer (Spark for accurate historical counts). Query layer merges both.

**Key concepts:**
- Kafka partitioned by adId: all clicks for same ad go to same partition — ordered processing
- Flink windowed aggregation: tumbling windows (1min, 5min, 1hr) for real-time counts
- Late events: watermarks define how late an event can arrive and still be counted
- Deduplication: bloom filter or Redis SET for click dedup within window
- Fraud detection: rule engine on click stream — same IP, same user, bot patterns

**Interviewer is testing:** Lambda architecture, Kafka partitioning, windowed aggregation, late events.

---

## S3-like Object Storage

**Core problem:** Store petabytes of objects durably with 11 nines durability.

**The trick:** Content-addressed chunks + erasure coding. Split object into chunks, apply erasure coding (e.g., 6+3 — can reconstruct from any 6 of 9 chunks). Store chunks across different racks/AZs.

**Key concepts:**
- Metadata vs data separation: metadata service (object name → chunk locations) separate from data nodes
- Erasure coding: more storage-efficient than 3x replication, same durability
- Content-addressed: `objectId = SHA256(content)` — automatic deduplication
- Consistent hashing: distribute chunks across data nodes, minimal reshuffling on scale-out
- Multipart upload: large objects split into parts, uploaded in parallel

**Interviewer is testing:** Erasure coding vs replication, metadata/data separation, consistent hashing.

---

## Recommendation System

**Core problem:** Suggest relevant items to users from a catalog of millions.

**The trick:** Two-stage pipeline. Recall stage: fast approximate nearest neighbor (ANN) search to get candidate set (thousands). Ranking stage: ML model scores and ranks candidates. Only top N shown.

**Key concepts:**
- Collaborative filtering: users who liked X also liked Y
- Content-based: item features similarity
- ANN search: FAISS or similar — approximate but fast for large catalogs
- Cold start: new user → popular items; new item → content-based similarity
- Offline pipeline: train model on batch data, serve via feature store
- A/B testing: different recommendation strategies for different user segments

**Interviewer is testing:** Two-stage pipeline, ANN search, cold start problem, offline vs online.

---

## Search Engine

**Core problem:** Index billions of documents and return relevant results in milliseconds.

**The trick:** Inverted index — maps each word to the list of documents containing it. TF-IDF or BM25 for relevance scoring. PageRank for authority.

**Key concepts:**
- Inverted index: `word → [(docId, positions)]` — fast lookup by term
- TF-IDF: term frequency × inverse document frequency — balances common vs rare words
- PageRank: link graph analysis — pages linked by many authoritative pages rank higher
- Crawl freshness: priority queue of URLs by freshness score, re-crawl based on change frequency
- SimHash: near-duplicate detection — similar documents have similar hashes
- Index sharding: shard by document hash, each shard has full inverted index for its docs

**Interviewer is testing:** Inverted index, TF-IDF, crawling, index sharding.

---

## HLD Interview Cheat Sheet

| System | Core problem | Key trick |
|--------|-------------|-----------|
| Ticket Booking | Double booking | Redis SET NX + MySQL ACID |
| Ecommerce | Inventory oversell | Redis DECR gate + saga |
| Netflix | Video delivery at scale | ABR + CDN as primary |
| Hotel Reservation | Date range double booking | Redis lock + overlap query |
| Food Delivery | Real-time driver matching | Redis GEOADD/GEORADIUS |
| Chat | Real-time + offline delivery | WebSocket + Cassandra + push |
| Job Scheduler | Exactly-once execution | SELECT FOR UPDATE SKIP LOCKED |
| Google Docs | Concurrent edits | OT/CRDT + Redis canonical copy |
| Ride Matching | Driver assignment | Redis geo + distributed lock |
| Social Media | Fan-out at scale | Hybrid push/pull |
| Leaderboard | Real-time ranking | Redis ZSET |
| Notification | Reliable delivery | Outbox pattern + Kafka tiers |
| Log Aggregation | Ingest at scale | Kafka buffer + tiered storage |
| Google Drive | Large file sync | Chunked + content-addressed |
| Payment | No double charge | Idempotency key + outbox |
| URL Shortener | Short unique IDs | Base62 + Redis cache |
| Stock Trading | Fair order matching | Order book price-time priority |
| Digital Wallet | No double spend | Double-entry + SELECT FOR UPDATE |
| Ad Click | Aggregate at scale | Lambda architecture + Flink |
| S3 Storage | 11 nines durability | Erasure coding + content-addressed |
| Recommendation | Relevant at scale | Two-stage recall + ranking |
| Search Engine | Fast full-text search | Inverted index + TF-IDF |

---

## Recurring Patterns Across Systems

**Idempotency** shows up everywhere. Any operation that can be retried needs an idempotency key. Payment, booking, notification, job execution.

**Outbox pattern** whenever you need to publish an event reliably after a DB write. Write to outbox in same transaction, separate worker publishes. No dual-write risk.

**Redis for fast path, DB for durability.** Redis handles the race condition or the hot read. DB is the source of truth. Never use Redis as your only store for critical data.

**Kafka as the backbone.** Decouples producers from consumers. Absorbs spikes. Enables replay. Almost every P1 system uses it.

**CDC for search sync.** Write to primary DB, Debezium captures changes, Kafka delivers to Elasticsearch. Avoids dual-write, eventual consistency is fine for search.

**Saga for distributed transactions.** When a business operation spans multiple services, use saga with compensating transactions instead of 2PC.
