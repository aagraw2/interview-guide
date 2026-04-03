# Hotel Booking System Design

## System Overview
A hotel booking platform (think Booking.com / Airbnb) where users search hotels by location and dates, check room availability, make reservations with payment, and leave reviews — with strong consistency on room availability to prevent double-booking.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- User registration and authentication
- Search hotels by location, dates, price, rating, room type
- View hotel details, room types, pricing, and availability
- Book a room for given dates with payment
- View and manage booking history
- Cancel bookings with refund
- Leave reviews and ratings for hotels
- Hotel management (add/update hotel, rooms, pricing)

### Non-Functional Requirements
- Availability: 99.99% — search and booking must always work
- Latency: <300ms for search; <500ms for booking
- Scalability: Read >> Write — millions search, thousands book simultaneously
- Consistency: Strong consistency for room availability — no double-booking
- Durability: Confirmed bookings must never be lost
- Security: PCI-DSS for payments, auth on all booking operations

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 50M DAU browsing hotels
- 500K bookings/day
- 1M hotels, avg 100 rooms each = 100M rooms
- Read:Write ratio = 1000:1 (search/browse >> book)
- Peak: holiday season, 10× normal booking traffic
- Average booking: 3 nights, $150/night

### Traffic
```
Search requests/sec   = 50M × 10 searches/day / 86400 ≈ 5.8K/sec
Peak search/sec       ≈ 58K/sec (holiday season)

Bookings/sec (avg)    = 500K / 86400 ≈ 6/sec
Bookings/sec (peak)   = 5M / 86400 ≈ 58/sec

Availability checks   = 5.8K × 5 rooms avg = 29K/sec
```

### Storage
```
Hotels          = 1M × 5KB avg = 5GB
Rooms           = 100M × 500B = 50GB
Availability    = 100M rooms × 365 days × 50B = 1.8TB/year
Bookings/day    = 500K × 2KB = 1GB/day → ~365GB/year
Reviews         = 1M hotels × 50 reviews × 500B = 25GB
Images          = 1M hotels × 20 images × 500KB = 10TB → S3 + CDN
```

### Memory (Cache)
```
Hot hotel metadata    = top 100K hotels × 5KB = 500MB
Availability cache    = 100M rooms × 50B = 5GB (hot: top 1M = 50MB)
Sessions              = 50M × 500B = 25GB
Redis locks           = active bookings × 100B ≈ negligible
```

## 3. Core Components

**LB + API Gateway** — Authentication, authorization, routing, rate limiting, round-robin load balancing

**User Service** — Registration, login, JWT issuance, profile management; reads/writes to User DB (MySQL)

**Search Service** — Hotel discovery by location (geo), dates, price, rating, amenities; queries Elasticsearch; read-heavy, stateless

**Review Service** — Submit and fetch hotel reviews and ratings; writes to Review DB; async rating recalculation

**Booking Info Service** — Read-only service for fetching booking details and history; reads from Booking DB

**Booking Service** — Core transactional service; handles room hold (Redis lock), booking creation, payment orchestration, cancellation

**Room Availability Service** — Manages room availability per date range; checks and updates Availability table in Hotel DB; called by Booking Service for reservation

**Payment Service** — Payment gateway integration, idempotent transactions, refunds; writes to Payment DB

**Notification Service** — Kafka consumer; sends booking confirmations, cancellation alerts, reminders via email/push

**Consumer** — Kafka consumer for general events (analytics, review aggregation, search index updates)

**Hotel DB (PostgreSQL)** — Hotels, rooms, pricing, and availability tables; ACID for availability updates

**User DB (MySQL)** — User profiles, credentials, metadata

**Review DB** — Hotel reviews, ratings, images

**Booking DB** — Confirmed bookings and booking history

**Payment DB** — Payment records, refunds

**Redis Cache** — Hot hotel metadata, sessions, room availability cache, Redis lock for booking (TTL 5min)

**Elasticsearch** — Full-text and geo search on hotels; synced from Hotel DB via CDC

**S3** — Hotel images and room photos

**Kafka** — Async event bus for booking events, notifications, CDC propagation

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL (Hotel DB) | Hotels, rooms, availability — ACID for availability updates, row-level locking to prevent double-booking, geo queries via PostGIS |
| MySQL (User DB) | Structured user data, ACID, relational |
| PostgreSQL (Booking DB) | ACID transactions for booking creation, relational queries for history |
| MySQL (Payment DB) | PCI-DSS compliance, ACID, audit trail |
| Redis | Room availability cache, session store, booking lock (TTL 5min) |
| Elasticsearch | Geo-distance search, full-text on hotel name/amenities, faceted filters |
| S3 | Hotel and room images |

### PostgreSQL (Hotel DB) — hotels

| Field | Type |
|---|---|
| hotel_id | UUID (PK) |
| name | VARCHAR |
| address | TEXT |
| geo_location | POINT (lat/lng, PostGIS indexed) |
| room_type | VARCHAR |
| capacity | INT |
| metadata | JSONB (amenities, policies) |
| avg_rating | DECIMAL |
| created_at | TIMESTAMP |

### PostgreSQL (Hotel DB) — rooms

| Field | Type |
|---|---|
| room_id | UUID (PK) |
| hotel_id | UUID (FK → hotels) |
| room_type | VARCHAR (single / double / suite) |
| capacity | INT |
| price | DECIMAL |
| currency | VARCHAR |
| metadata | JSONB |

### PostgreSQL (Hotel DB) — availability

| Field | Type |
|---|---|
| hotel_id | UUID |
| room_id | UUID |
| start_date | DATE |
| end_date | DATE |
| status | ENUM (AVL / BOOKED / MAINTENANCE) |

### MySQL (User DB) — users

| Field | Type |
|---|---|
| user_id | UUID (PK) |
| email | VARCHAR, unique |
| password | VARCHAR (hashed) |
| date | DATE (DOB) |
| metadata | JSONB |

### PostgreSQL (Booking DB) — bookings

| Field | Type |
|---|---|
| booking_id | UUID (PK) |
| user_id | UUID |
| hotel_id | UUID |
| room_id | UUID |
| check_in | DATE |
| check_out | DATE |
| status | ENUM (pending / confirmed / cancelled) |
| total_amount | DECIMAL |
| payment_id | UUID |
| idempotency_key | VARCHAR, unique |
| created_at | TIMESTAMP |

### MySQL (Payment DB) — payments

| Field | Type |
|---|---|
| payment_id | UUID (PK) |
| user_id | UUID |
| booking_id | UUID |
| amount | DECIMAL |
| currency | VARCHAR |
| status | ENUM (pending / success / failed / refunded) |
| date | TIMESTAMP |

### Review DB — reviews

| Field | Type |
|---|---|
| review_id | UUID (PK) |
| hotel_id | UUID |
| user_id | UUID |
| rating | INT (1–5) |
| comments | TEXT |
| images | ARRAY\<TEXT\> (S3 URLs) |
| created_at | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `session:{sessionId}` | String | `{userId}` | 86400s |
| `hotel:meta:{hotelId}` | String | hotel metadata JSON | 300s |
| `avail:{hotelId}:{date}` | String | available room count | 60s |
| `book:lock:{roomId}:{checkIn}:{checkOut}` | String | userId (NX EX) | 300s (5min checkout window) |

## 5. Key Flows

### 5.1 Auth

1. User registers → User Service → hash password → write to User DB (MySQL) → return JWT
2. Login → validate credentials → generate JWT (1hr) + refresh token → store session in Redis
3. Every request: API Gateway validates JWT + Redis session
4. Booking and payment endpoints require auth; search and hotel browse are public

### 5.2 Hotel Search

```
Client → API Gateway → Search Service → Elasticsearch
                                              ↓
                                  geo-filtered, ranked results
                                              ↓
                       Hotel metadata from Redis cache → PostgreSQL fallback
                                              ↓
                              Images served via CDN (S3)
```

1. User searches "hotels in Paris, Dec 20–23, 2 guests, under $200" → Search Service
2. Elasticsearch query:
   - Geo-distance filter: within X km of "Paris" coordinates
   - Date availability filter (from availability cache or pre-indexed)
   - Price range filter
   - Full-text on hotel name, amenities
   - Sort by relevance / rating / price
3. Returns ranked hotel list with basic metadata
4. Client selects hotel → fetch full details from Redis cache (TTL 5min) → fallback to PostgreSQL
5. Room availability for selected dates fetched from Redis cache → fallback to Hotel DB

**CDC Sync (Hotel DB → Elasticsearch):**
- Hotel/room updates written to PostgreSQL → CDC captures change → Kafka → Elasticsearch consumer updates index
- Eventual consistency — search index may lag a few seconds

### 5.3 Room Availability Check

1. Client requests availability for `{hotelId, checkIn, checkOut}`
2. Booking Info Service checks Redis cache: `avail:{hotelId}:{date}` for each date in range
3. Cache hit: return available room count and types
4. Cache miss: query Hotel DB `availability` table for rooms with `status = AVL` in date range
5. Populate Redis cache (TTL 60s)
6. Return available rooms with pricing to client

### 5.4 Booking Flow — Room Hold & Payment

This is the critical flow — two users must not book the same room for overlapping dates.

```
Client → Booking Service
              ↓
    Acquire Redis lock: SET book:lock:{roomId}:{checkIn}:{checkOut} {userId} NX EX 300
              ↓ (lock acquired)
    Verify availability in Hotel DB (SELECT FOR UPDATE)
              ↓
    Update availability: status = BOOKED for date range
              ↓
    Create booking record (status = pending)
              ↓
    Call Payment Service → Payment Gateway
              ↓
    On success: booking.status = confirmed, publish to Kafka
    On failure: release lock, revert availability, cancel booking
```

**Step by step:**
1. User selects room and dates → `POST /booking`
2. Booking Service attempts Redis lock: `SET book:lock:{roomId}:{checkIn}:{checkOut} {userId} NX EX 300`
3. If lock exists (another user booking same room): return "room being booked, try again"
4. Lock acquired → verify availability in PostgreSQL with `SELECT FOR UPDATE` on availability rows
5. If available: update `availability.status = BOOKED` for the date range
6. Create `booking` record with `status = pending` in Booking DB
7. Call Payment Service with `{bookingId, amount, idempotencyKey}`
8. Payment Service calls external gateway
9. On payment success:
   - Update `booking.status = confirmed`
   - Write payment record to Payment DB
   - Release Redis lock
   - Invalidate availability cache: `DEL avail:{hotelId}:{date}` for affected dates
   - Publish `BOOKING_CONFIRMED` to Kafka
10. On payment failure:
    - Revert `availability.status = AVL`
    - Update `booking.status = cancelled`
    - Release Redis lock
    - Return error to client

**Why Redis lock + DB transaction:**
Redis `SET NX EX` is the fast first gate — prevents two users from even reaching the DB transaction simultaneously. PostgreSQL `SELECT FOR UPDATE` is the durable correctness guarantee. Together they handle both speed and safety.

**Concurrency example:**
```
User1 and User2 both try to book Room 101, Dec 20–23:

Redis: SET book:lock:room101:dec20:dec23 user1 NX EX 300 → OK (user1 wins)
Redis: SET book:lock:room101:dec20:dec23 user2 NX EX 300 → NIL (user2 blocked)

User2 gets "room is being booked" — retries after a moment
If user1 completes: user2 sees "room unavailable"
If user1 abandons: lock expires after 5min, user2 can try
```

### 5.5 Booking Cancellation & Refund

**Before check-in:**
1. User cancels → Booking Service
2. Verify booking belongs to user and is `confirmed`
3. Check cancellation policy (e.g., free cancellation >48hr before check-in)
4. PostgreSQL transaction:
   - `booking.status = cancelled`
   - `availability.status = AVL` for the date range (room freed)
5. Payment Service initiates refund via gateway
6. Invalidate availability cache for affected dates
7. Publish `BOOKING_CANCELLED` to Kafka → Notification Service sends confirmation

### 5.6 Reviews & Ratings

1. After check-out date passes, user submits review → Review Service
2. Validate user had a confirmed booking for that hotel (check Booking DB)
3. Write review to Review DB
4. Async (Consumer via Kafka): recalculate `avg_rating` for hotel, update Hotel DB
5. CDC propagates updated `avg_rating` to Elasticsearch (affects search ranking)
6. Review images uploaded to S3, URLs stored in Review DB

### 5.7 Notification Flow

Kafka consumers handle all async notifications:
- `BOOKING_CONFIRMED` → send booking confirmation email with details
- `BOOKING_CANCELLED` → send cancellation + refund confirmation
- Reminder 24hr before check-in → scheduled job publishes event to Kafka
- Payment failure → alert user to update payment method

## 6. Key Interview Concepts

### Preventing Double Booking — The Core Problem
Same as ticket booking but more complex — a room booking spans multiple dates, not a single seat. Two users booking the same room for overlapping date ranges must be prevented.

Solution:
1. Redis `SET NX EX` lock on `{roomId}:{checkIn}:{checkOut}` — fast atomic gate
2. PostgreSQL `SELECT FOR UPDATE` on availability rows — durable correctness
3. Availability table stores per-room per-date-range status — overlap check is a range query

**Date range overlap check:**
```
Existing booking: Dec 20 – Dec 23
New request:      Dec 22 – Dec 25  ← overlaps

Overlap condition: newStart < existingEnd AND newEnd > existingStart
```

### Why PostgreSQL for Hotel DB
Hotels, rooms, and availability have relational structure with foreign keys. Availability requires ACID transactions (update multiple date rows atomically). PostGIS extension enables efficient geo-distance queries for location-based search. Cassandra would be wrong here — no transactions, no row-level locking.

### Availability Table Design
Storing availability as date ranges (start_date, end_date, status) is more efficient than one row per day. A 365-day availability for 100M rooms as daily rows = 36.5B rows. As date ranges, most rooms have just a few records (available, booked, maintenance periods).

**Trade-off:** range queries are slightly more complex but storage is orders of magnitude smaller.

### Redis Lock TTL (5 Minutes)
5-minute checkout window gives users time to complete payment without holding the room indefinitely. If payment takes >5min (unlikely), lock expires and room becomes available again. The booking record in DB is also marked `expired` by a cleanup job. TTL is a business decision — balance between user experience and inventory availability.

### CDC for Search Sync
Hotel DB (PostgreSQL) → CDC (Debezium) → Kafka → Elasticsearch. Hotel updates (new rooms, price changes, rating updates) propagate to search index asynchronously. Eventual consistency — search may show slightly stale prices for a few seconds. Acceptable for browse; actual price confirmed at booking time.

### Geo Search with Elasticsearch + PostGIS
Elasticsearch `geo_distance` query filters hotels within X km of a point — fast, scalable, purpose-built for this. PostGIS on PostgreSQL is an alternative for more complex geo queries (polygon search, routing). For simple radius search, Elasticsearch is preferred since it also handles full-text and facets in the same query.

### Read >> Write Architecture
1000:1 read:write ratio. Search path (Elasticsearch + Redis cache) is completely separate from booking path (PostgreSQL transactions). CDN serves all images. Redis absorbs hot hotel metadata reads. PostgreSQL only sees transactional writes (bookings, availability updates).

### Idempotency in Payments
Payment gateway timeout → client retries → risk of double charge. Solution: `idempotency_key = hash(bookingId + userId)`. Gateway returns same response for duplicate keys. Unique constraint in Payment DB prevents duplicate records.

### Saga Pattern for Booking + Availability + Payment
Three operations must succeed together. If payment fails, availability must be released.
```
ROOM_HELD → initiate payment
PAYMENT_SUCCESS → confirm booking, keep availability as BOOKED
PAYMENT_FAILED → release availability back to AVL, cancel booking
```
Compensating actions ensure no room is permanently blocked by a failed payment.

## 7. Failure Scenarios

### Double Booking Attempt (Race Condition)
- Scenario: two users simultaneously book the same room for same dates
- Prevention: Redis `SET NX EX` — only one acquires lock; PostgreSQL `SELECT FOR UPDATE` as safety net
- If Redis is down: fall back to PostgreSQL `SELECT FOR UPDATE` only — slower but correct

### Redis Lock Failure (Lock Lost Mid-Booking)
- Scenario: Redis crashes after lock acquired but before booking completes
- Recovery: Redis Sentinel failover (<30s); lock lost means another user could acquire it; PostgreSQL `SELECT FOR UPDATE` prevents actual double-booking at DB level; first user's transaction either commits or rolls back cleanly
- Prevention: Redis Cluster + AOF; PostgreSQL is the ultimate correctness guarantee

### Payment Gateway Timeout
- Detection: no response within 5s
- Recovery: retry with same `idempotency_key`; after 3 retries, release availability, cancel booking, notify user; async webhook callback as fallback
- Prevention: idempotency key prevents double charge on retry

### PostgreSQL Primary Failure (Hotel DB)
- Impact: availability checks and booking writes fail; search (Elasticsearch) continues working
- Recovery: promote replica (<30s); Redis cache continues serving availability reads during failover; in-flight bookings with Redis locks retry after recovery
- Prevention: synchronous replication for availability and bookings; automated failover

### Availability Cache Stale (Redis)
- Scenario: room booked but Redis cache not invalidated → search shows room as available
- Recovery: user reaches booking step, DB check shows unavailable → "room no longer available" error; user searches again with fresh data
- Prevention: always invalidate `avail:{hotelId}:{date}` cache on booking confirmation and cancellation; short TTL (60s) limits staleness window

### Elasticsearch Failure
- Impact: hotel search unavailable; booking for known hotels still works via direct hotel ID
- Recovery: graceful degradation — fall back to PostgreSQL geo queries (slower, less rich); CDC replays missed changes on recovery
- Prevention: Elasticsearch cluster with replicas; non-critical path — bookings unaffected

### Booking Lock Expiry During Payment
- Scenario: payment takes slightly longer than 5min TTL → lock expires → another user acquires lock
- Recovery: second user's `SELECT FOR UPDATE` sees availability already `BOOKED` (first user's DB update committed) → correctly blocked; first user's booking completes normally
- Prevention: payment timeout enforced client-side at 4min 30s; 5min TTL is generous for any real payment flow

### Kafka Consumer Lag (Notification Service)
- Impact: booking confirmation emails delayed
- Recovery: Kafka retains events; Notification Service catches up; booking is confirmed in DB regardless
- Prevention: monitor consumer lag; Notification Service is non-critical path — booking correctness unaffected
