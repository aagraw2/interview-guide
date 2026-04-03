# Ticket Booking System Design

## System Overview
A ticket booking platform (think BookMyShow / Ticketmaster) where users discover events and movies, select seats, and complete bookings with payment — with strong consistency on seat allocation to prevent double-booking.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Browse and search events/movies by name, venue, artist, location, date
- View event details — venue, seats, pricing
- Select seats and temporarily hold them during checkout
- Complete booking with payment
- View booking history and tickets

### Non-Functional Requirements
- Availability: 99.99% — especially during high-demand event launches
- Latency: <200ms for search; <500ms for seat selection
- Scalability: Read >> Write — millions browse, thousands book simultaneously
- Consistency: Strong consistency for seat booking — no double-booking ever
- Durability: Confirmed bookings must never be lost
- Security: PCI-DSS for payments, auth on all booking operations

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 50M DAU browsing events
- 500K concurrent users during peak event launch (e.g., popular concert goes on sale)
- 1M bookings/day on average, 100K bookings in first 10 min of a hot event
- Average event has 10K seats
- Read:Write ratio = 100:1 (browse >> book)

### Traffic
```
Search/browse reads/sec  = 50M × 20 searches/day / 86400 ≈ 11.5K/sec
Peak reads/sec           ≈ 100K/sec (event launch)

Booking writes/sec       = 1M / 86400 ≈ 12/sec average
Peak booking/sec         = 100K bookings / 600s ≈ 167/sec (hot event)

Seat hold operations     = 167/sec × 10 seats avg = 1670 Redis ops/sec (trivial)
```

### Storage
```
Events          = 1M events × 10KB avg = 10GB
Seats per event = 10K seats × 200B = 2MB per event → 2TB total
Bookings/day    = 1M × 1KB = 1GB/day → ~365GB/year
Media (posters) = 1M events × 500KB = 500GB → S3 + CDN
```

### Memory (Redis)
```
Active seat maps (hot events) = 1000 hot events × 2MB = 2GB
Seat holds                    = 500K concurrent users × 200B = 100MB
Sessions                      = 50M × 500B = 25GB
```

## 3. Core Components

**API Gateway** — Authentication, authorization, rate limiting, routing; critical during flash sales (aggressive rate limiting per user)

**Search Service** — Handles event/movie discovery; queries Elasticsearch for full-text and geo search; read-heavy, stateless, horizontally scalable

**Event Service** — Manages event and movie metadata (CRUD); writes to Cassandra; changes propagated to Elasticsearch via CDC (Change Data Capture)

**Booking Service** — Core transactional service; handles seat selection, temporary holds, booking confirmation, and cancellation; writes to MySQL/PostgreSQL

**Payment Gateway** — External payment processor integration; Booking Service calls it synchronously during checkout

**Elasticsearch** — Full-text search index for events and movies; kept in sync with Cassandra via CDC

**Cassandra** — Stores event and movie metadata; optimized for high read throughput (browse >> write)

**MySQL / PostgreSQL** — Stores bookings and seat allocation; ACID transactions prevent double-booking

**Redis** — Temporary seat holds during checkout (TTL-based); session cache; hot event seat map cache

**CDC (Change Data Capture)** — Captures row-level changes from Cassandra and syncs to Elasticsearch; keeps search index eventually consistent without coupling Event Service to Elasticsearch directly

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| Cassandra | Event/movie metadata — high read throughput, denormalized, scales horizontally; read >> write fits perfectly |
| MySQL / PostgreSQL | Bookings and seat allocation — ACID transactions are non-negotiable to prevent double-booking |
| Redis | Seat holds (TTL auto-release), session cache, hot seat map cache for fast availability checks |
| Elasticsearch | Full-text search on event name, artist, venue, description; geo-distance queries |

### Cassandra — events

| Field | Type |
|---|---|
| id | UUID (PK) |
| venue | VARCHAR |
| artist_id | UUID |
| name | VARCHAR |
| description | TEXT |
| seats | LIST\<UUID\> (seat IDs) |
| event_date | TIMESTAMP |
| created_at | TIMESTAMP |

### Cassandra — movies

| Field | Type |
|---|---|
| id | UUID (PK) |
| title | VARCHAR |
| actors | LIST\<VARCHAR\> |
| seats | LIST\<UUID\> (seat IDs per show) |
| description | TEXT |
| release_date | TIMESTAMP |

### MySQL / PostgreSQL — bookings

| Field | Type |
|---|---|
| booking_id | UUID (PK) |
| user_id | UUID (FK → users) |
| event_id | UUID |
| seats | JSON (array of seat IDs) |
| status | ENUM (pending / confirmed / cancelled) |
| total_amount | DECIMAL |
| payment_id | VARCHAR |
| idempotency_key | VARCHAR, unique |
| created_at | TIMESTAMP |
| expires_at | TIMESTAMP (for pending holds) |

### MySQL / PostgreSQL — seat_allocations

| Field | Type |
|---|---|
| seat_id | UUID (PK) |
| event_id | UUID |
| status | ENUM (available / held / booked) |
| held_by | UUID (user_id), nullable |
| booking_id | UUID, nullable |
| hold_expires_at | TIMESTAMP, nullable |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `hold:{eventId}:{seatId}` | String | userId | 600s (10 min checkout window) |
| `seat:map:{eventId}` | Hash | seatId → status | 300s (cache) |
| `session:{sessionId}` | String | `{userId}` | 86400s |
| `rate:{userId}` | Counter | request count | 60s window |

## 5. Key Flows

### 5.1 Auth

1. User registers/logs in → API Gateway → User Service → JWT issued
2. JWT stored client-side, sent in `Authorization` header on every request
3. API Gateway validates JWT on all requests
4. Booking and payment endpoints require valid auth — no anonymous bookings
5. Rate limiting at API Gateway: stricter limits during flash sales (e.g., 5 booking attempts/min per user)

### 5.2 Event Discovery & Search

```
Client → API Gateway → Search Service → Elasticsearch
                                              ↓
                                     ranked results
                                              ↓
                              enrich with metadata from Cassandra cache
```

1. User searches "SpiderMan" or "Coldplay Mumbai" → Search Service
2. Elasticsearch query: full-text match on `name`, `artist`, `venue`, `description` + optional geo filter
3. Returns ranked list of events/movies with basic metadata
4. Client selects event → Event Service fetches full details from Cassandra
5. Seat availability overview fetched from Redis cache (`seat:map:{eventId}`) → fallback to MySQL

**CDC Sync:**
- Event Service writes event/movie to Cassandra
- CDC agent (Debezium) captures the change from Cassandra's commit log
- Publishes change event to Kafka
- Elasticsearch consumer updates the search index
- Eventual consistency — search index may lag by a few seconds

### 5.3 Seat Selection & Temporary Hold

This is the critical flow — two users must not be able to book the same seat.

```
User selects seats → Booking Service
                          ↓
              Check Redis: is seat held? (2ms)
                          ↓ (not held)
              Acquire hold in Redis: SET hold:{eventId}:{seatId} {userId} EX 600
                          ↓
              Write hold to MySQL seat_allocations (status=held)
                          ↓
              Return hold confirmation + 10min countdown to client
```

1. User selects seats (e.g., L12, L13 for SpiderMan)
2. Booking Service checks Redis for existing holds on each seat
3. If any seat already held by another user → return "seat unavailable"
4. If all seats free: atomically set Redis holds with 10-min TTL
5. Write `status = held, held_by = userId, hold_expires_at = now+10min` to MySQL `seat_allocations`
6. Create `booking` record with `status = pending` in MySQL
7. Return seat hold confirmation to client — checkout timer starts

**Why Redis for holds?**
Redis `SET NX EX` (set if not exists, with expiry) is atomic — two concurrent requests for the same seat, only one wins. Sub-millisecond. MySQL alone would require row-level locking which is slower and doesn't auto-expire.

**Concurrency on same seat (race condition):**
```
User1 and User2 both try to hold seat L12 simultaneously:

Redis: SET hold:event1:L12 user1 NX EX 600 → OK (user1 wins)
Redis: SET hold:event1:L12 user2 NX EX 600 → NIL (user2 loses, seat already held)

User2 gets "seat unavailable" immediately.
```

### 5.4 Payment & Booking Confirmation

```
Client submits payment → Booking Service → Payment Gateway
                                                  ↓
                                          payment success
                                                  ↓
                              MySQL transaction:
                              - booking.status = confirmed
                              - seat_allocations.status = booked
                              - clear Redis hold
```

1. User submits payment details within 10-min window
2. Booking Service verifies hold still valid (Redis key exists + not expired)
3. Calls Payment Gateway with `{amount, paymentDetails, idempotencyKey}`
4. On payment success:
   - Begin MySQL transaction
   - Update `bookings.status = confirmed`, set `payment_id`
   - Update `seat_allocations.status = booked` for all seats
   - Commit transaction
   - Delete Redis hold keys
5. On payment failure:
   - Update `bookings.status = cancelled`
   - Update `seat_allocations.status = available`
   - Delete Redis hold keys
   - Return error to client
6. Send booking confirmation (email/push) to user

**Idempotency:** `idempotency_key = hash(bookingId + userId)` — if payment gateway times out and client retries, same key prevents double charge.

### 5.5 Hold Expiry (Timeout During Checkout)

If user abandons checkout within the 10-min window:

1. Redis TTL expires → `hold:{eventId}:{seatId}` key deleted automatically
2. A background cleanup job (runs every minute) queries MySQL for `seat_allocations` where `status = held AND hold_expires_at < now`
3. Updates those rows to `status = available`
4. Updates corresponding `bookings.status = expired`
5. Seats become available for other users

Redis TTL handles the fast path (seat appears available to new users immediately). MySQL cleanup handles the durable state.

### 5.6 Booking Cancellation

1. User requests cancellation → Booking Service
2. Verify booking belongs to user and is in `confirmed` status
3. Check cancellation policy (e.g., >24hr before event → full refund)
4. MySQL transaction: `bookings.status = cancelled`, `seat_allocations.status = available`
5. Initiate refund via Payment Gateway
6. Notify user

### 5.7 High-Demand Event Launch (Flash Sale)

When a popular event goes on sale, hundreds of thousands of users hit simultaneously:

1. API Gateway enforces strict rate limiting per user (queue excess requests)
2. Virtual waiting room: users assigned a queue position, served in order
3. Seat map served from Redis cache — avoids hammering MySQL for availability checks
4. Redis `SET NX` ensures only one user holds each seat regardless of concurrent requests
5. MySQL handles only confirmed transactions — not every seat check

## 6. Key Interview Concepts

### Preventing Double Booking — The Core Problem
Two users selecting the same seat is the hardest problem. The solution is a two-phase approach:
1. **Optimistic hold in Redis** (`SET NX EX`) — atomic, fast, auto-expiring
2. **Durable confirmation in MySQL** — ACID transaction on payment success

Redis handles the race condition at the hold stage. MySQL handles durability at confirmation. Neither alone is sufficient — Redis alone loses data on crash; MySQL alone is too slow for concurrent hold checks.

### Why Cassandra for Events but MySQL for Bookings
Events are read-heavy (millions browse, few write) and don't need transactions — Cassandra's high read throughput and denormalized model fits perfectly. Bookings require ACID transactions (seat allocation must be atomic with booking creation) — MySQL/PostgreSQL is the right tool. Never use Cassandra for transactional data.

### CDC for Search Index Sync
Event Service writes to Cassandra. Instead of also writing to Elasticsearch directly (dual write — risky, can diverge), CDC captures changes from Cassandra's commit log and propagates to Elasticsearch asynchronously. Benefits: Event Service stays simple, Elasticsearch is eventually consistent, no data loss on Elasticsearch failure (CDC replays).

### Seat Hold TTL Design
10-minute TTL is a business + UX decision. Too short: users can't complete checkout. Too long: seats locked by abandoned users, reducing availability. Redis TTL auto-releases without any explicit cleanup — the key simply disappears. MySQL cleanup job handles the durable state sync.

### Optimistic vs Pessimistic Locking
- Pessimistic locking (SELECT FOR UPDATE in MySQL): locks the row, prevents concurrent access. Safe but slow — locks held for entire checkout duration (10 min) would block all other users.
- Optimistic locking (Redis NX + version check): assume no conflict, detect and reject if conflict occurs. Fast, scales well. Right choice here since conflicts are rare (most seats are available most of the time).

### Virtual Waiting Room for Flash Sales
Without a queue, 500K simultaneous requests overwhelm the booking service. Solution: API Gateway puts excess users in a virtual queue, issues a queue token, and admits users in batches. Users see "You are #4,521 in queue." Prevents thundering herd, gives fair access, protects backend.

### Read >> Write Architecture
100:1 read:write ratio means the browse/search path must be optimized independently of the booking path. Separate Search Service (Elasticsearch) and Event Service (Cassandra) handle reads. Booking Service (MySQL) handles writes. They scale independently — you can have 100 Search Service instances and 5 Booking Service instances.

### Idempotency in Payments
Payment gateway calls can time out. Client retries → risk of double charge. Solution: generate `idempotency_key = hash(bookingId + userId)` before calling gateway. Gateway returns same response for duplicate keys. Store key in MySQL with unique constraint — duplicate DB insert fails, preventing double processing.

## 7. Failure Scenarios

### Double Booking Attempt (Race Condition)
- Scenario: two users simultaneously select the same seat
- Prevention: Redis `SET NX EX` is atomic — only one succeeds; the other gets "seat unavailable" immediately
- If Redis is down: fall back to MySQL `SELECT FOR UPDATE` on `seat_allocations` row — slower but correct

### Redis Failure (Seat Holds Lost)
- Impact: active holds lost — users in checkout may lose their seats; seat map cache unavailable
- Recovery: Redis Sentinel failover (<30s); MySQL `seat_allocations` table is the durable record — holds still exist there; Booking Service falls back to MySQL for hold checks during Redis outage
- Prevention: Redis Cluster + AOF persistence; holds in MySQL are the source of truth

### Payment Gateway Timeout
- Detection: no response within 5s
- Recovery: retry with same `idempotency_key`; after 3 retries, mark booking `payment_failed`, release seat hold, notify user to retry
- Prevention: async payment with webhook callback as fallback — gateway calls back when payment completes

### MySQL Primary Failure
- Impact: booking writes fail; reads (booking history) can continue on replica
- Recovery: promote replica (auto-failover, <30s); Redis holds remain valid during failover; users may need to retry booking confirmation
- Prevention: synchronous replication for bookings table; automated failover with Patroni/RDS

### Hold Expiry Race (User Pays Just as Hold Expires)
- Scenario: user submits payment at second 599 of 600s hold; Redis TTL expires before MySQL transaction commits
- Recovery: Booking Service checks MySQL `hold_expires_at` (not just Redis) before processing payment; if within grace period (e.g., 30s), allow payment to proceed
- Prevention: payment submission deadline enforced client-side at 9min 30s (30s before actual expiry)

### Elasticsearch Failure
- Impact: search unavailable; event browsing degraded
- Recovery: graceful degradation — fall back to Cassandra queries (slower, less rich search); CDC replays missed changes to Elasticsearch on recovery
- Prevention: Elasticsearch cluster with replicas; non-critical path — bookings unaffected

### Flash Sale Overload
- Detection: API Gateway request queue depth spikes; error rates increase
- Recovery: virtual waiting room activates; rate limiting drops excess requests with 429; auto-scaling spins up Search and Booking Service instances
- Prevention: pre-scale before known high-demand launches; load test with expected peak traffic; CDN caches event detail pages to reduce origin load
