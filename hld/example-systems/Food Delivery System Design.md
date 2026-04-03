# Food Delivery System Design

## System Overview
A food delivery platform (think Swiggy / Zomato / DoorDash) connecting customers, restaurants, and delivery partners — handling order placement, real-time order tracking, payments, and restaurant discovery.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- User registration and authentication (customers, restaurants, delivery partners)
- Restaurant and menu browsing with search and filters
- Cart management and order placement
- Payment processing
- Real-time order status tracking (placed → confirmed → preparing → picked up → delivered)
- Real-time delivery partner location tracking
- Ratings and reviews for restaurants and delivery partners
- Notifications (order updates, promotions)

### Non-Functional Requirements
- Availability: 99.99% uptime (especially during peak meal hours)
- Latency: <500ms for search/browse, <2s for order placement
- Scalability: 10M+ DAU, 5M+ orders/day
- Consistency: Strong consistency for payments and orders; eventual for reviews/ratings
- Durability: Orders and payments must never be lost
- Security: PCI-DSS compliance for payments, TLS everywhere

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 10M DAU (customers)
- 500K active restaurants, 1M delivery partners
- 5M orders/day, avg order value $15
- Peak hours: 12–2pm, 7–9pm (5× normal traffic)
- Each order has ~10 status updates
- Location ping from delivery partner every 5s while on delivery

### Traffic
```
Orders/day        = 5M
Orders/sec        = 5M / 86400 ≈ 58/sec
Peak orders/sec   = 58 × 5 ≈ 290/sec

Order events/sec  = 290 × 10 status updates ≈ 2900 events/sec

Active deliveries at peak = 290 orders/sec × 30 min avg = ~500K
Location pings/sec        = 500K partners × (1/5s) = 100K pings/sec

Search requests/sec = 10M DAU × 20 searches/day / 86400 ≈ 2300/sec
```

### Storage
```
Orders/day        = 5M × 2KB avg = 10GB/day → ~3.6TB/year
Location history  = 100K pings/sec × 100B = 10MB/sec → ~315TB/year
  (retain 30 days only → ~9TB rolling)
Menu/restaurant   = 500K restaurants × 50KB = 25GB (mostly static)
Media (food imgs) = 500K restaurants × 20 images × 200KB = 2TB
```

### Memory (Cache)
```
Active sessions       = 10M × 1KB  = 10GB
Restaurant cache      = 500K × 50KB = 25GB (top 10K hot = 500MB)
Active order state    = 500K orders × 2KB = 1GB
Driver location cache = 1M drivers × 100B = 100MB
```

## 3. Core Components

**API Gateway / LB** — SSL termination, JWT validation, rate limiting, routing to services

**User Service** — Registration, login, JWT issuance for customers, restaurants, and delivery partners

**Restaurant Service** — Restaurant CRUD, menu management, availability, operating hours

**Search & Discovery Service** — Restaurant search by location, cuisine, rating, ETA; backed by Elasticsearch

**Order Service** — Order lifecycle management (create → confirm → prepare → dispatch → deliver), the core orchestrator

**Cart Service** — Ephemeral cart state per user, stored in Redis

**Payment Service** — Payment initiation, gateway integration, refunds, idempotent transactions

**Delivery Service** — Delivery partner assignment, location tracking, ETA calculation

**Notification Service** — Push (FCM/APNs), SMS, email for order updates and promotions

**Rating & Review Service** — Post-delivery ratings for restaurants and delivery partners

**Analytics Service** — Async processing of events for reporting, recommendations, fraud detection

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL | Orders, payments, users — ACID critical, relational |
| MongoDB | Restaurant menus — flexible schema, nested items/variants |
| Redis | Cart, sessions, active order state, driver location cache, rate limiting |
| Elasticsearch | Restaurant/menu search by location, cuisine, text |
| Cassandra | Location history, order event log — time-series, high write throughput |
| S3 + CDN | Food images, restaurant banners |
| Kafka | Async event bus between services (order events, location updates) |

### PostgreSQL — users

| Field | Type |
|---|---|
| user_id | UUID (PK) |
| name | VARCHAR |
| email | VARCHAR, unique |
| phone | VARCHAR, unique |
| password_hash | VARCHAR |
| role | ENUM (customer / restaurant / driver) |
| created_at | TIMESTAMP |

### PostgreSQL — orders

| Field | Type |
|---|---|
| order_id | UUID (PK) |
| customer_id | UUID (FK → users) |
| restaurant_id | UUID (FK → restaurants) |
| driver_id | UUID (FK → users), nullable |
| status | ENUM (placed / confirmed / preparing / picked_up / delivered / cancelled) |
| total_amount | DECIMAL |
| delivery_address | JSONB |
| payment_id | UUID (FK → payments) |
| placed_at | TIMESTAMP |
| updated_at | TIMESTAMP |

### PostgreSQL — order_items

| Field | Type |
|---|---|
| item_id | UUID (PK) |
| order_id | UUID (FK → orders) |
| menu_item_id | UUID |
| name | VARCHAR (snapshot at order time) |
| quantity | INT |
| unit_price | DECIMAL |

### PostgreSQL — payments

| Field | Type |
|---|---|
| payment_id | UUID (PK) |
| order_id | UUID (FK → orders) |
| user_id | UUID (FK → users) |
| amount | DECIMAL |
| status | ENUM (pending / success / failed / refunded) |
| gateway_txn_id | VARCHAR, unique |
| idempotency_key | VARCHAR, unique |
| created_at | TIMESTAMP |

### PostgreSQL — restaurants

| Field | Type |
|---|---|
| restaurant_id | UUID (PK) |
| owner_id | UUID (FK → users) |
| name | VARCHAR |
| cuisine_type | VARCHAR[] |
| address | JSONB |
| location | POINT (lat/lng, indexed with PostGIS) |
| avg_rating | DECIMAL |
| is_open | BOOLEAN |
| created_at | TIMESTAMP |

### MongoDB — menus
One document per restaurant

| Field | Type |
|---|---|
| restaurant_id | UUID |
| categories | Array of { name, items[] } |
| items[].item_id | UUID |
| items[].name | STRING |
| items[].description | STRING |
| items[].price | DECIMAL |
| items[].is_available | BOOLEAN |
| items[].image_url | STRING |
| items[].customizations | Array of { name, options[] } |
| updated_at | TIMESTAMP |

### Cassandra — location_history
Partition key: `driver_id`, Clustering: `timestamp DESC`

| Field | Type |
|---|---|
| driver_id | UUID (partition key) |
| timestamp | TIMESTAMP (clustering) |
| lat | DOUBLE |
| lng | DOUBLE |
| order_id | UUID, nullable |

### Cassandra — order_events
Partition key: `order_id`, Clustering: `event_time DESC`

| Field | Type |
|---|---|
| order_id | UUID (partition key) |
| event_time | TIMESTAMP (clustering) |
| event_type | TEXT (placed / confirmed / preparing / picked_up / delivered) |
| actor_id | UUID |
| metadata | TEXT (JSON) |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `cart:{userId}` | Hash | `{itemId: {qty, price}}` | 3600s |
| `session:{sessionId}` | String | `{userId, role}` | 86400s |
| `order:active:{orderId}` | String | order state JSON | until delivered |
| `driver:loc:{driverId}` | String | `{lat, lng, updatedAt}` | 30s |
| `restaurant:cache:{id}` | String | restaurant + menu JSON | 300s |
| `rate:{userId}` | Counter | request count | 60s window |

## 5. Key Flows

### 5.1 Auth — Registration & Login

**Register:**
1. Client → API Gateway → User Service
2. Validate input, hash password (bcrypt)
3. Write to PostgreSQL with role (customer / restaurant / driver)
4. Return JWT token

**Login:**
1. Validate credentials against PostgreSQL
2. Generate JWT (1hr) + refresh token
3. Store session in Redis with TTL
4. Return tokens

**Every request:** API Gateway validates JWT signature + expiry + Redis session

### 5.2 Restaurant Discovery & Search

1. Customer opens app → sends location (lat/lng) + optional filters (cuisine, rating, ETA)
2. API Gateway → Search Service
3. Search Service queries Elasticsearch:
   - Geo-distance filter (within X km)
   - Filter by `is_open = true`
   - Text match on name/cuisine
   - Sort by relevance / rating / ETA
4. Returns paginated list of restaurants
5. Restaurant details (menu, images) served from Redis cache → fallback to MongoDB + PostgreSQL
6. Food images served via CDN

**ETA Calculation:**
- Estimated prep time from restaurant (stored in PostgreSQL)
- Estimated delivery time from Delivery Service (based on distance + active driver availability)
- Combined ETA shown to customer

### 5.3 Cart & Order Placement

**Cart:**
1. Customer adds/removes items → Cart Service updates Redis hash `cart:{userId}`
2. Cart is ephemeral — no DB write until order placed
3. On checkout, Cart Service validates item availability against MongoDB menu

**Order Placement:**
1. Customer submits order → Order Service
2. Order Service validates:
   - Cart items still available (MongoDB)
   - Restaurant is open (PostgreSQL)
3. Creates order record in PostgreSQL with status `placed`
4. Publishes `ORDER_PLACED` event to Kafka
5. Clears cart from Redis
6. Returns `orderId` to client

### 5.4 Payment Flow

1. Order Service triggers Payment Service with `orderId` + `amount` + `idempotency_key`
2. Payment Service calls payment gateway (Stripe / Razorpay)
3. On success:
   - Write payment record to PostgreSQL with `status = success`
   - Publish `PAYMENT_SUCCESS` event to Kafka
4. On failure:
   - Write `status = failed`
   - Publish `PAYMENT_FAILED` → Order Service cancels order
5. Order Service consumes `PAYMENT_SUCCESS` → updates order status to `confirmed`
6. Notification Service sends "Order confirmed" push to customer

**Idempotency:** `idempotency_key = hash(orderId + userId)` — if gateway times out and retries, same key prevents double charge.

### 5.5 Order Lifecycle & Restaurant Flow

```
ORDER_PLACED → payment → ORDER_CONFIRMED → restaurant accepts → PREPARING
→ driver assigned → driver arrives at restaurant → PICKED_UP
→ driver reaches customer → DELIVERED
```

1. `ORDER_CONFIRMED` event → Notification Service pushes to restaurant app
2. Restaurant accepts → Order Service updates status to `preparing`, sets estimated prep time
3. Delivery Service receives `ORDER_CONFIRMED` event → starts driver assignment
4. Restaurant marks ready → Order Service updates status, Delivery Service dispatches driver
5. Driver picks up → status `picked_up`, customer notified
6. Driver delivers → status `delivered`, payment settled, rating prompt triggered

### 5.6 Driver Assignment

1. Delivery Service receives `ORDER_CONFIRMED` event from Kafka
2. Queries Redis for nearby available drivers: geo-radius search on `driver:loc:{driverId}`
3. Ranks by proximity + acceptance rate + current load
4. Sends assignment request to top driver via push notification
5. Driver accepts → assigned; driver rejects or no response in 30s → try next driver
6. On assignment: write `driver_id` to order in PostgreSQL, publish `DRIVER_ASSIGNED` event
7. Customer notified with driver details and live ETA

**No driver available:** retry pool every 30s, expand radius, escalate alert after 5 min

### 5.7 Real-Time Location Tracking

1. Driver app sends GPS ping every 5s to Delivery Service via HTTP or WebSocket
2. Delivery Service writes to Redis: `SET driver:loc:{driverId} {lat, lng}` with 30s TTL
3. Appends to Cassandra `location_history` (async, for audit/analytics)
4. Customer app polls Delivery Service every 5s (or WebSocket push) for driver location
5. Delivery Service reads from Redis cache → returns lat/lng + recalculated ETA
6. Map rendered on client with smooth interpolation between pings

### 5.8 Order Cancellation

**Before restaurant confirms:**
1. Customer cancels → Order Service updates status to `cancelled`
2. Payment Service initiates full refund
3. Kafka event → Notification Service informs restaurant

**After restaurant starts preparing:**
- Cancellation may be rejected or partial refund applied (business rule)
- If driver already assigned: Delivery Service unassigns driver, driver gets cancellation fee

### 5.9 Ratings & Reviews

1. After `DELIVERED` event, Notification Service sends rating prompt (delayed 10 min)
2. Customer submits rating → Rating Service
3. Rating Service writes to PostgreSQL
4. Async job recalculates `avg_rating` for restaurant and driver
5. Updated rating pushed to Elasticsearch index for search ranking

## 6. Key Interview Concepts

### Geo-Search for Restaurants and Drivers
Elasticsearch supports geo-distance queries natively — filter restaurants within X km of customer. For driver lookup, Redis `GEOADD` / `GEORADIUS` commands give sub-ms geo queries on driver locations. PostGIS on PostgreSQL is an alternative for restaurant search if Elasticsearch isn't used.

### Idempotency in Payments
Network timeouts can cause duplicate payment requests. Solution: generate an `idempotency_key = hash(orderId + userId)` before calling the gateway. Gateway returns the same response for duplicate keys without charging twice. Store the key in PostgreSQL with a unique constraint.

### Order State Machine
Order status is a strict state machine: `placed → confirmed → preparing → picked_up → delivered`. Invalid transitions (e.g., `placed → delivered`) are rejected. This prevents race conditions where two events try to update status simultaneously. Use optimistic locking (version column) in PostgreSQL.

### Driver Assignment — Matching Problem
Similar to a ride-hailing matcher. Key considerations:
- Proximity (minimize pickup time)
- Driver acceptance rate (avoid assigning to drivers who reject often)
- Load balancing (don't overload one driver)
- Timeout + fallback: if no accept in 30s, try next candidate

### Real-Time Tracking Scalability
100K location pings/sec is high. Solutions:
- Write to Redis first (fast, in-memory), async flush to Cassandra
- Cassandra handles time-series writes well (partition by driverId)
- Customer polling at 5s intervals is acceptable; WebSocket push reduces polling load at scale

### Kafka as Event Bus
Decouples services — Order Service doesn't call Payment, Notification, Delivery directly. It publishes events; each service consumes independently. Benefits: resilience (consumer can retry), scalability (add consumers without changing producer), auditability (event log).

**Example:**
```
Order Service publishes: ORDER_PLACED
Consumers:
  - Payment Service → charge customer
  - Notification Service → "Order received" to restaurant
  - Analytics Service → log for reporting
```

### Caching Strategy
- Restaurant + menu: cache in Redis (TTL 5 min) — menus change infrequently
- Active order state: Redis for fast reads by customer/driver polling
- Driver locations: Redis with 30s TTL — stale after that, driver considered offline
- Search results: short TTL cache (30s) for popular location+cuisine combos

### Database Sharding for Orders
Orders table grows fast (5M/day). Shard by `customer_id` (range or hash). All orders for a customer on one shard — efficient customer history queries. Cross-shard queries (e.g., restaurant's orders) handled by Kafka event log or a separate read model.

### CAP Trade-off
- Orders and payments: CP — strong consistency, use PostgreSQL with transactions
- Location tracking: AP — availability over consistency, slight staleness in driver location is fine
- Restaurant search: AP — eventual consistency, cached results acceptable

### Peak Load Handling
Meal-time peaks (5× normal). Strategies:
- Auto-scaling for stateless services (Search, Order, Notification)
- Pre-warm caches before peak hours
- Queue-based load leveling: Order Service writes to Kafka, downstream services process at their own rate
- Circuit breakers prevent cascade if Payment gateway is slow

## 7. Failure Scenarios

### Payment Gateway Timeout
- Detection: no response within 5s
- Recovery: retry with same `idempotency_key` (safe, no double charge); after 3 retries, mark payment `failed`, cancel order, notify customer
- Prevention: async payment with webhook callback as fallback

### Order Service Crash Mid-Flow
- Detection: Kafka consumer group detects unacknowledged message
- Recovery: another Order Service instance picks up the Kafka message; idempotent processing (check if order already exists before creating)
- Prevention: write order to PostgreSQL before publishing Kafka event (outbox pattern)

### Outbox Pattern for Reliability
Risk: Order Service writes to DB and then crashes before publishing to Kafka — event lost.
Solution: write event to an `outbox` table in the same DB transaction. A separate poller reads the outbox and publishes to Kafka, then marks as published. Guarantees at-least-once delivery.

### Driver Goes Offline Mid-Delivery
- Detection: no location ping for 60s, TTL on `driver:loc:{driverId}` expires
- Recovery: Delivery Service detects missing location, marks driver unavailable, reassigns order to another driver if not yet picked up; if already picked up, alert ops team
- Customer notified of delay

### Restaurant Stops Responding
- Detection: no order confirmation within 5 min
- Recovery: auto-cancel order, full refund, notify customer; flag restaurant for ops review
- Prevention: restaurant SLA monitoring, auto-suspend restaurants with high cancellation rate

### PostgreSQL Primary Failure
- Detection: health check fails, replication lag spikes
- Recovery: promote read replica to primary (automated with Patroni/RDS); brief write unavailability (<30s)
- Prevention: synchronous replication for orders/payments, async for analytics

### Redis Failure
- Impact: cart data lost, active order cache miss, driver locations unavailable
- Recovery: Redis Sentinel failover (<30s); cart rebuilt from client state; order state re-read from PostgreSQL; driver locations rebuilt from next ping
- Prevention: Redis Cluster + AOF persistence

### Elasticsearch Failure
- Impact: restaurant search degraded
- Recovery: graceful degradation — fall back to PostgreSQL geo query (slower but functional); Elasticsearch rebuilt from PostgreSQL + MongoDB on recovery
- Search is non-critical path; orders in flight are unaffected

### Kafka Lag / Consumer Slowdown
- Detection: consumer group lag metric spikes
- Recovery: scale up consumer instances; if lag is critical (payment events), prioritize those topics
- Prevention: separate Kafka topics by priority (payments vs analytics); retention policy ensures no event loss

### DDoS / Traffic Spike
- Rate limiting at API Gateway (per IP + per userId)
- Separate rate limits for search (expensive) vs order status (cheap)
- Auto-scaling for stateless services
- CDN absorbs media/static traffic

### Data Consistency — Order + Payment
Risk: payment succeeds but order status update fails.
Solution: saga pattern — each step publishes a compensating event on failure.
```
Payment success → publish PAYMENT_SUCCESS
Order update fails → publish PAYMENT_REFUND_NEEDED
Payment Service consumes → initiates refund
```
Ensures no customer is charged for a failed order.
