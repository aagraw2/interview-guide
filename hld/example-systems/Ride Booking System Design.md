# Ride Booking System Design

## System Overview
A ride-hailing platform (think Uber / Ola) connecting riders with nearby drivers — handling real-time location tracking, driver matching, fare calculation, trip management, and payments with strong consistency on driver assignment.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- User and driver registration and authentication
- Rider requests a ride (vehicle type, pickup, destination)
- Match rider with nearest available driver
- Driver accepts or denies ride request
- Real-time location tracking of driver during trip
- Fare calculation (distance + wait time + surge pricing)
- Payment processing post-trip
- Ratings for rider and driver after trip
- Trip history

### Non-Functional Requirements
- Availability: 99.99% — ride requests must always be processed
- Latency: <500ms for driver matching; <5s for driver to receive request
- Scalability: 10M+ DAU, 1M+ concurrent active drivers
- Consistency: Strong consistency on driver assignment — one driver per ride
- Durability: Trip records and payments must never be lost
- Location freshness: Driver location updated every 5–10s

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 10M DAU (riders), 1M active drivers at peak
- 5M rides/day
- Average trip: 20 min, 10 km
- Driver location update every 5s while active
- Read:Write ratio = 10:1 for ride requests vs location updates

### Traffic
```
Rides/day               = 5M
Rides/sec               = 5M / 86400 ≈ 58/sec
Peak rides/sec          ≈ 580/sec (10× during rush hour)

Driver location pings   = 1M drivers × (1/5s) = 200K pings/sec
Location data/sec       = 200K × 100B = 20MB/sec
```

### Storage
```
Rides/day               = 5M × 2KB = 10GB/day → ~3.6TB/year
Driver location history = 200K pings/sec × 100B = 20MB/sec
                        → retain 30 days = ~52TB rolling
Driver records          = 1M × 1KB = 1GB
Rider records           = 10M × 1KB = 10GB
Ratings                 = 5M/day × 200B = 1GB/day
```

### Memory (Redis)
```
Active drivers (all with TTL) = 1M × 200B = 200MB
Active ride locks             = 580 × 100B ≈ negligible
Sessions                      = 10M × 500B = 5GB
```

## 3. Core Components

**LB + API Gateway** — Authentication, authorization, rate limiting, routing, round-robin traffic distribution

**WebSocket Gateway** — Persistent WebSocket connections for drivers; authentication, authorization, session stickiness; receives location updates every 5–10s

**Ride Service** — Core ride lifecycle management; fare calculation; creates and manages ride records in Ride DB; integrates with Location Map Service for distance/ETA

**Driver Matching Service** — Geoproximity search to find nearest available drivers; sends ride request to driver via Notification Service; uses Zookeeper lock to ensure one driver per ride

**Location Update Service (WebSocket Server)** — Receives driver location pings via WebSocket; updates Redis (all active drivers with TTL); publishes to Kafka for async persistence and trip tracking

**Trip Update Consumer** — Kafka consumer; periodically persists driver location, status, and rating to Drivers DB; routes driver navigation to drop passenger

**Location Map Service (Google Maps / Apple Maps)** — External map API; provides distance calculation, ETA, route planning

**Surge Calculator Service** — Calculates surge multiplier based on demand/supply ratio in a geographic area; feeds into fare calculation

**Rating Service** — Post-trip ratings for drivers and riders; writes to Rating DB; Aggregator Service computes rolling averages

**Aggregator Service** — Computes and updates average ratings for drivers and riders asynchronously

**Payment Service** — Post-trip payment processing via payment gateway; writes to Payment DB

**Notification Service** — Sends ride requests to drivers via FCM (Firebase Cloud Messaging) / APNs (Apple Push Notification); sends status updates to riders

**Zookeeper** — Distributed lock per `driverId`; ensures only one ride request is sent to a driver at a time; lock deleted after 10s if no response

**Redis** — All active drivers with status and TTL; geo-index for proximity search; session store; active ride state

**Kafka** — Location update stream; trip events for async processing by Trip Update Consumer

**Ride Request DB** — Pending and active ride requests

**Ride DB (PostgreSQL)** — Full ride/trip records with fare, route, status

**Drivers DB (PostgreSQL)** — Driver profiles, location, status, vehicle info, ratings

**Rating DB (PostgreSQL)** — Individual ratings per trip for drivers and riders

**Payment DB (MySQL)** — Payment records, fare breakdown, refunds

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL (Ride DB) | ACID for trip records, fare, payment linkage; relational queries for history |
| PostgreSQL (Drivers DB) | Driver profiles + location; PostGIS for geo queries as fallback to Redis |
| PostgreSQL (Rating DB) | Structured ratings per trip, relational to rides and drivers |
| MySQL (Payment DB) | PCI-DSS compliance, ACID, audit trail |
| Redis | Active driver geo-index (GEOADD/GEORADIUS), TTL-based presence, session store |
| Zookeeper | Distributed lock per driverId — ensures single ride request per driver |
| Kafka | Location update stream, trip event log for async consumers |

### PostgreSQL (Ride DB) — rides

| Field | Type |
|---|---|
| ride_id | UUID (PK) |
| rider_id | UUID |
| driver_id | UUID, nullable |
| pickup | POINT (lat/lng) |
| destination | POINT (lat/lng) |
| status | ENUM (requested / accepted / in_progress / completed / cancelled) |
| vehicle_type | VARCHAR (bike / car / auto) |
| fare | DECIMAL, nullable |
| currency | VARCHAR |
| timestamp | TIMESTAMP |
| metadata | JSONB |

### PostgreSQL (Ride DB) — estimated_fare

| Field | Type |
|---|---|
| ride_id | UUID (FK → rides) |
| requests | INT |
| pickup_lng | DOUBLE |
| pickup_lat | DOUBLE |
| drop_lng | DOUBLE |
| drop_lat | DOUBLE |
| fare | DECIMAL |
| currency | VARCHAR |
| timestamp | TIMESTAMP |
| metadata | JSONB |

### PostgreSQL (Drivers DB) — drivers

| Field | Type |
|---|---|
| driver_id | UUID (PK) |
| user_id | UUID |
| name | VARCHAR |
| vehicle_info | JSONB |
| rating | DECIMAL |
| mobile_num | VARCHAR |
| status | ENUM (IDLE / DRIVING / ...) |
| metadata | JSONB |

### PostgreSQL (Drivers DB) — location

| Field | Type |
|---|---|
| driver_id | UUID (PK) |
| latitude | DOUBLE |
| longitude | DOUBLE |
| timestamp | TIMESTAMP |
| modified_ts | TIMESTAMP |

### PostgreSQL (Rating DB) — ratings

| Field | Type |
|---|---|
| sender_id | UUID |
| driver_id | UUID |
| rating | DECIMAL (1–5) |
| name | VARCHAR |
| timestamp | TIMESTAMP |
| ride_id | UUID |

### MySQL (Payment DB) — payments

| Field | Type |
|---|---|
| payment_id | UUID (PK) |
| user_id | UUID |
| booking_id | UUID |
| amount | DECIMAL |
| drop_lng | DOUBLE |
| drop_lat | DOUBLE |
| currency | VARCHAR |
| status | ENUM (pending / success / failed / refunded) |
| timestamp | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `drivers:active` | GEO | driverId → lat/lng | — |
| `driver:status:{driverId}` | String | IDLE / DRIVING | 30s (refreshed by location ping) |
| `driver:lock:{driverId}` | String | rideId (Zookeeper-managed) | 10s |
| `session:{sessionId}` | String | `{userId, role}` | 86400s |
| `ride:active:{rideId}` | String | ride state JSON | until completed |

## 5. Key Flows

### 5.1 Auth

1. Rider/Driver registers → API Gateway → User Service → write to respective DB → return JWT
2. Login → validate credentials → JWT (1hr) + refresh token → session in Redis
3. Every HTTP request: API Gateway validates JWT + session
4. Driver WebSocket connection: WebSocket Gateway validates JWT + session on connect
5. Drivers and riders have separate roles — driver endpoints require `role=driver`

### 5.2 Driver Location Updates

```
Driver app → WebSocket Gateway → Location Update Service
                                          ↓
                              Redis: GEOADD drivers:active {lng} {lat} {driverId}
                              Redis: SET driver:status:{driverId} IDLE EX 30
                                          ↓
                              Kafka: publish location event
                                          ↓
                              Trip Update Consumer → Drivers DB (async persist)
```

1. Driver app sends GPS ping every 5–10s via WebSocket
2. WebSocket Gateway (session stickiness — same driver always hits same server) forwards to Location Update Service
3. Location Update Service updates Redis:
   - `GEOADD drivers:active {lng} {lat} {driverId}` — geo-indexed for proximity search
   - `SET driver:status:{driverId} IDLE EX 30` — TTL refreshed on each ping; expires if driver goes offline
4. Publishes location event to Kafka
5. Trip Update Consumer persists to Drivers DB `location` table (async — not in critical path)

**Driver goes offline:** TTL on `driver:status` expires → driver removed from active pool automatically

### 5.3 Ride Request & Fare Estimation

1. Rider opens app, enters destination → Ride Service
2. Ride Service calls Location Map Service (Google Maps / Apple Maps) for distance and ETA
3. Calls Surge Calculator Service for current surge multiplier in rider's area
4. Calculates estimated fare:
   ```
   Bike: Rs 20/km + Rs 1/min waiting
   Car:  Rs 60/km + Rs 1/min waiting
   × surge multiplier
   ```
5. Writes estimated fare to `estimated_fare` table
6. Returns fare estimate + ETA to rider
7. Rider confirms → Ride Service creates ride record with `status = requested`

### 5.4 Driver Matching — The Core Flow

```
Ride Service → Driver Matching Service
                      ↓
        GEORADIUS drivers:active {pickup_lat} {pickup_lng} 5km
        Filter: status = IDLE
                      ↓
        Rank by proximity → candidate list [d1, d2, d3...]
                      ↓
        For top driver d1:
          Zookeeper: acquire lock on driverId (SETNX)
          If lock acquired:
            Send ride request to d1 via Notification Service (FCM/APNs)
            Set lock TTL = 10s
          If lock not acquired (driver already has a request):
            Try next driver d2
                      ↓
        Wait for driver response (accept/deny) via WebSocket
                      ↓
        Accept → assign driver, update ride status
        Deny / No response in 10s → release lock, try next driver
```

1. Driver Matching Service queries Redis: `GEORADIUS drivers:active {lat} {lng} 5km ASC COUNT 10`
2. Filters for `status = IDLE` drivers
3. Ranks by proximity
4. For the top candidate: Zookeeper acquires lock on `driverId`
   - Lock ensures only one ride request goes to a driver at a time
   - Lock TTL = 10s — auto-released if driver doesn't respond
5. Notification Service sends ride request to driver via FCM/APNs
6. Driver sees request on app, accepts or denies via WebSocket
7. On accept:
   - Zookeeper releases lock
   - Ride Service updates `rides.status = accepted`, sets `driver_id`
   - Updates `driver:status:{driverId} = DRIVING` in Redis
   - Notifies rider via Notification Service
8. On deny or timeout (10s):
   - Zookeeper releases lock
   - Try next driver in candidate list
9. If no driver accepts after all candidates: notify rider "no drivers available", cancel ride

### 5.5 Active Trip Tracking

1. Driver starts trip → `rides.status = in_progress`
2. Driver continues sending location pings every 5s via WebSocket
3. Location Update Service updates Redis + publishes to Kafka
4. Trip Update Consumer consumes Kafka events:
   - Persists driver location to Drivers DB
   - Updates driver status
   - Computes route progress (how close to destination)
5. Rider app polls or receives WebSocket push for driver location every 5s
6. Map rendered on rider's app with live driver position

### 5.6 Trip Completion & Payment

1. Driver marks trip complete → Ride Service
2. Ride Service calls Location Map Service for actual distance travelled
3. Calculates final fare (actual distance + wait time + surge)
4. Updates `rides.status = completed`, sets final `fare`
5. Calls Payment Service: charge rider's saved payment method
6. Payment Service calls payment gateway
7. On success: write to Payment DB, notify rider of receipt
8. On failure: retry; if persistent failure, flag for manual resolution
9. Trigger rating prompt for both rider and driver (delayed 2 min)

### 5.7 Rating Flow

1. Rider rates driver (and vice versa) → Rating Service
2. Write to Rating DB
3. Aggregator Service (async) recalculates rolling average rating for driver/rider
4. Updates `drivers.rating` in Drivers DB
5. Rating affects driver's priority in future matching (higher-rated drivers ranked higher)

### 5.8 Ride Cancellation

**Before driver assigned:**
- Rider cancels → Ride Service updates `status = cancelled`; no charge

**After driver assigned (before pickup):**
- Rider cancels → cancellation fee may apply (business rule)
- Driver Matching Service releases driver: `driver:status = IDLE`
- Zookeeper lock released
- Driver notified via Notification Service

**Driver cancels:**
- Driver denies or cancels after accepting
- Ride Service re-triggers Driver Matching for next available driver
- Cancellation logged against driver (affects future matching priority)

## 6. Key Interview Concepts

### Zookeeper for Driver Lock
The core problem: two ride requests must not go to the same driver simultaneously. Zookeeper provides a distributed lock per `driverId`:
- `SETNX lock:{driverId} {rideId}` — only one request acquires the lock
- TTL = 10s — auto-released if driver doesn't respond (no deadlock)
- On accept/deny: lock explicitly released

Why Zookeeper over Redis for this lock? Zookeeper provides stronger consistency guarantees (ZAB consensus protocol) and is purpose-built for distributed coordination. Redis `SETNX` is simpler but can have split-brain issues under network partition. For a critical lock (driver assignment), Zookeeper is the safer choice.

### Redis GEO for Driver Proximity
Redis `GEOADD` stores driver locations in a geo-indexed sorted set. `GEORADIUS` returns drivers within X km sorted by distance — O(N+log M) where N is results and M is total drivers. With 1M active drivers, this is fast enough for real-time matching.

Example:
```
GEOADD drivers:active 72.8777 19.0760 "driver:abc"
GEOADD drivers:active 72.8800 19.0800 "driver:xyz"

GEORADIUS drivers:active 72.8790 19.0770 5 km ASC COUNT 10
→ ["driver:abc", "driver:xyz"]  (sorted by distance)
```

### Session Stickiness on WebSocket Gateway
Drivers maintain persistent WebSocket connections. Session stickiness (consistent hashing on `driverId`) ensures the same driver always connects to the same WebSocket Gateway instance. This means:
- Location updates don't need cross-instance routing
- Driver's accept/deny response reaches the same instance that sent the request
- On instance failure: driver reconnects to another instance (brief interruption)

### Surge Pricing
Surge Calculator Service monitors demand/supply ratio per geographic cell (H3 hexagonal grid or geohash):
```
surge = f(active_ride_requests / available_drivers in area)
surge = 1.0 (normal) → 2.5× (high demand)
```
Surge multiplier applied to base fare at estimation time. Shown to rider before confirmation. Recalculated every 30–60s.

### Driver Matching — Ranking Beyond Proximity
Proximity is the primary factor but not the only one:
- Distance to pickup (minimize ETA)
- Driver rating (higher-rated drivers preferred)
- Acceptance rate (avoid drivers who frequently reject)
- Current heading (driver moving toward pickup is better than one moving away)

### Location Update at Scale
200K pings/sec is significant. Redis handles this easily (single-threaded, in-memory). Kafka absorbs the stream for async DB persistence. Drivers DB is updated asynchronously — slight staleness in DB is fine since Redis is the live source. TTL on Redis driver status (30s) auto-cleans offline drivers without explicit cleanup.

### Fare Calculation
```
Base fare = distance × rate_per_km + wait_time × rate_per_min
Final fare = base_fare × surge_multiplier

Bike: Rs 20/km + Rs 1/min
Car:  Rs 60/km + Rs 1/min
```
Estimated fare shown before trip (based on predicted distance/time). Final fare calculated after trip completion using actual distance from Location Map Service.

### CAP Trade-off
- Driver assignment: CP — consistency critical, Zookeeper lock ensures one driver per ride
- Location tracking: AP — slight staleness acceptable, Redis TTL handles offline detection
- Fare/payment: CP — ACID transactions in PostgreSQL/MySQL
- Driver matching: AP — Redis geo-index may be slightly stale (30s TTL), acceptable

## 7. Failure Scenarios

### No Driver Accepts (All Reject or Timeout)
- Detection: all candidates in 5km radius tried, none accepted
- Recovery: expand search radius (5km → 10km → 15km); if still none, notify rider "no drivers available"; cancel ride
- Prevention: surge pricing incentivizes more drivers to come online in high-demand areas

### Zookeeper Lock Not Released (Driver Crash)
- Detection: lock TTL (10s) expires automatically
- Recovery: Driver Matching Service tries next candidate; no manual intervention needed
- Prevention: TTL on all Zookeeper locks; never hold lock indefinitely

### Location Update Service Crash
- Detection: WebSocket connections drop; health check fails
- Recovery: drivers reconnect (exponential backoff); session stickiness routes to new instance; Redis geo-index rebuilt from reconnecting drivers; brief gap in location data acceptable
- Prevention: multiple Location Update Service instances; WebSocket Gateway routes to healthy instances

### Redis Failure (Active Driver Index Lost)
- Impact: driver proximity search unavailable; new ride matching fails
- Recovery: Redis Sentinel failover (<30s); active drivers rebuild index as they send next location ping (within 5–10s); brief matching unavailability
- Prevention: Redis Cluster + AOF persistence; driver index rebuilds quickly from live pings

### Driver Goes Offline Mid-Trip
- Detection: no location ping for 30s (TTL expires on `driver:status`)
- Recovery: Trip Update Consumer detects missing heartbeat; alert ops; rider notified of delay; if driver reconnects, trip continues; if not, ops manually resolves
- Prevention: driver app retries WebSocket connection aggressively; offline detection threshold tunable

### Payment Failure Post-Trip
- Detection: payment gateway returns failure
- Recovery: retry 3 times with backoff; if persistent failure, mark payment `pending`, notify rider to update payment method; trip record preserved regardless
- Prevention: idempotency key on payment; store payment method securely; fallback to cash option

### Kafka Consumer Lag (Trip Update Consumer)
- Impact: Drivers DB location data stale; analytics delayed; real-time tracking unaffected (Redis is live source)
- Recovery: scale up consumers; Kafka retains events; DB catches up
- Prevention: monitor consumer lag; Trip Update Consumer is non-critical path — Redis is the live source of truth for active trips

### Surge Calculator Failure
- Impact: surge multiplier unavailable; fall back to 1.0× (no surge)
- Recovery: Ride Service uses default multiplier (1.0) on Surge Calculator timeout; graceful degradation
- Prevention: Surge Calculator is stateless, restarts quickly; cache last known surge values in Redis with short TTL
