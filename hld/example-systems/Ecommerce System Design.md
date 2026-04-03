# Ecommerce System Design

## System Overview
A large-scale ecommerce platform (think Amazon / Flipkart) where users browse and search products, manage a cart, place orders, make payments, and track deliveries — with strong consistency on inventory and payments, and high read throughput for product discovery.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- User registration and authentication
- Product listing, search, and filtering (by category, price, rating, brand)
- Product detail pages with images, reviews, and stock availability
- Cart management (add, remove, update quantity)
- Order placement with inventory reservation
- Payment processing
- Order tracking and history
- Seller product and inventory management
- Ratings and reviews

### Non-Functional Requirements
- Availability: 99.99% — especially during flash sales
- Latency: <200ms for search/browse; <500ms for order placement
- Scalability: Read >> Write — 100M+ DAU, billions of product views/day
- Consistency: Strong consistency for inventory and payments; eventual for reviews, search index
- Durability: Orders and payments must never be lost
- Security: PCI-DSS for payments, auth on all order/payment operations

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 100M DAU
- Each user views 20 product pages/day, performs 2 searches
- 1M orders/day average; 10M orders/day during flash sale
- Average order: 2 items, $50 value
- 500M products in catalog
- Read:Write ratio = 1000:1 for product views vs inventory updates

### Traffic
```
Product views/sec   = 100M × 20 / 86400 ≈ 23K/sec
Search queries/sec  = 100M × 2 / 86400  ≈ 2.3K/sec
Peak (flash sale)   ≈ 10× = 230K views/sec

Orders/sec (avg)    = 1M / 86400 ≈ 12/sec
Orders/sec (peak)   = 10M / 86400 ≈ 116/sec

Inventory updates   ≈ 116 × 2 items = 232 writes/sec (peak)
```

### Storage
```
Product catalog     = 500M × 2KB avg = 1TB
Product images      = 500M × 5 images × 200KB = 500TB → S3 + CDN
Orders/day          = 1M × 2KB = 2GB/day → ~730GB/year
Inventory           = 500M products × 200B = 100GB (fits in memory partially)
Reviews             = 500M products × 10 reviews × 500B = 2.5TB
```

### Memory (Cache)
```
Hot products (top 1%)  = 5M × 2KB = 10GB
Cart data              = 10M active carts × 1KB = 10GB
Sessions               = 100M × 500B = 50GB
Inventory cache        = 500M × 50B = 25GB (hot items only: top 1M = 50MB)
```

## 3. Core Components

**API Gateway** — Authentication, authorization, rate limiting, routing; aggressive throttling during flash sales

**User Service** — Registration, login, JWT issuance, profile and address management

**Product Service** — Product catalog CRUD (seller-facing); writes to Product DB; changes propagated to Elasticsearch via CDC

**Search Service** — Product discovery via Elasticsearch; full-text search, filters, facets, geo; read-heavy, stateless

**Inventory Service** — Tracks stock levels per product per seller; handles reservation and deduction; strong consistency required

**Cart Service** — Manages user cart state; ephemeral, stored in Redis; validates against Inventory Service on checkout

**Order Service** — Orchestrates order lifecycle: cart → inventory reservation → payment → confirmation; writes to Order DB

**Payment Service** — Integrates with payment gateway; idempotent transactions; handles refunds

**Notification Service** — Order confirmation, shipping updates, promotional emails/push via async queue

**Review Service** — Product ratings and reviews; eventually consistent; writes to Review DB, async sync to Elasticsearch for search ranking

**CDN** — Serves product images and static assets; absorbs majority of read traffic

**CDC (Change Data Capture)** — Captures product catalog changes and syncs to Elasticsearch; decouples Product Service from search index

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL (Orders, Payments) | ACID transactions — inventory reservation + order creation must be atomic |
| Cassandra (Product Catalog) | 500M products, read-heavy, denormalized, high throughput; no joins needed |
| MySQL (Inventory) | Row-level locking for stock deduction; ACID critical; moderate data size |
| Redis | Cart (ephemeral), sessions, hot product cache, inventory cache, rate limiting |
| Elasticsearch | Full-text product search, faceted filtering, ranking by relevance + rating |
| S3 + CDN | Product images, static assets |
| Kafka | Async event bus — order events, inventory updates, notifications, CDC |

### Cassandra — products

| Field | Type |
|---|---|
| product_id | UUID (PK) |
| seller_id | UUID |
| title | VARCHAR |
| description | TEXT |
| category | VARCHAR |
| brand | VARCHAR |
| price | DECIMAL |
| image_urls | LIST\<TEXT\> |
| attributes | MAP\<TEXT, TEXT\> (color, size, etc.) |
| avg_rating | DECIMAL |
| review_count | INT |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

### MySQL — inventory

| Field | Type |
|---|---|
| inventory_id | UUID (PK) |
| product_id | UUID |
| seller_id | UUID |
| quantity_available | INT |
| quantity_reserved | INT |
| updated_at | TIMESTAMP |

### PostgreSQL — orders

| Field | Type |
|---|---|
| order_id | UUID (PK) |
| user_id | UUID |
| status | ENUM (pending / confirmed / shipped / delivered / cancelled / returned) |
| total_amount | DECIMAL |
| shipping_address | JSONB |
| payment_id | UUID |
| idempotency_key | VARCHAR, unique |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

### PostgreSQL — order_items

| Field | Type |
|---|---|
| item_id | UUID (PK) |
| order_id | UUID (FK → orders) |
| product_id | UUID |
| seller_id | UUID |
| quantity | INT |
| unit_price | DECIMAL (snapshot at order time) |
| title | VARCHAR (snapshot at order time) |

### PostgreSQL — payments

| Field | Type |
|---|---|
| payment_id | UUID (PK) |
| order_id | UUID |
| user_id | UUID |
| amount | DECIMAL |
| status | ENUM (pending / success / failed / refunded) |
| gateway_txn_id | VARCHAR, unique |
| idempotency_key | VARCHAR, unique |
| created_at | TIMESTAMP |

### Cassandra — reviews

| Field | Type |
|---|---|
| product_id | UUID (partition key) |
| review_id | TIMEUUID (clustering) |
| user_id | UUID |
| rating | INT (1–5) |
| title | VARCHAR |
| body | TEXT |
| created_at | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `cart:{userId}` | Hash | `{productId: {qty, price, title}}` | 7 days |
| `product:{productId}` | String | product JSON | 300s |
| `inventory:{productId}` | String | available quantity | 60s |
| `session:{sessionId}` | String | `{userId}` | 86400s |
| `rate:{userId}` | Counter | request count | 60s window |
| `flash:lock:{productId}` | String | userId (NX EX) | 30s (checkout hold) |

## 5. Key Flows

### 5.1 Auth

1. Register/login → User Service → JWT (1hr) + refresh token
2. Session stored in Redis
3. API Gateway validates JWT + session on every request
4. Order, cart, and payment endpoints require auth
5. Product browse and search are public (no auth required)

### 5.2 Product Search & Browse

```
Client → API Gateway → Search Service → Elasticsearch
                                              ↓
                                    ranked product list
                                              ↓
                       Product detail → Redis cache → Cassandra fallback
                                              ↓
                              Images served via CDN (S3)
```

1. User searches "wireless headphones under $100" → Search Service
2. Elasticsearch query: full-text on `title`, `description`, `brand` + price range filter + category facet
3. Returns ranked product list (by relevance × rating × sales rank)
4. User clicks product → Product Service fetches detail from Redis cache (TTL 5min) → fallback to Cassandra
5. Inventory availability fetched from Redis cache (TTL 60s) → fallback to MySQL
6. Images served from CDN

**CDC Sync (Product → Elasticsearch):**
- Seller updates product → Product Service writes to Cassandra
- CDC agent (Debezium) captures change from Cassandra commit log → Kafka
- Elasticsearch consumer updates search index
- Eventual consistency — index may lag a few seconds

### 5.3 Cart Management

Cart is ephemeral — stored only in Redis until order is placed.

1. User adds item → `HSET cart:{userId} {productId} {qty, price, title}`
2. Update quantity → `HSET` with new qty; remove → `HDEL`
3. Cart TTL: 7 days (refreshed on each update)
4. On cart view: fetch from Redis, enrich with current prices from Product cache
5. Price shown in cart is live — if price changed since add, show updated price

**Price snapshot on order:** at order creation, prices are snapped into `order_items` — cart price is display only.

### 5.4 Order Placement & Inventory Reservation

This is the critical flow — inventory must be reserved atomically to prevent overselling.

```
Client → Order Service
              ↓
    Validate cart items (Product Service)
              ↓
    Reserve inventory (Inventory Service) ← MySQL row-level lock
              ↓
    Create order record (PostgreSQL, status=pending)
              ↓
    Initiate payment (Payment Service)
              ↓
    On success: confirm order, deduct inventory
    On failure: release reservation, cancel order
```

**Inventory Reservation (preventing oversell):**
```sql
BEGIN TRANSACTION;
  SELECT quantity_available FROM inventory
  WHERE product_id = ? FOR UPDATE;  -- row-level lock

  IF quantity_available >= requested_qty THEN
    UPDATE inventory SET
      quantity_available = quantity_available - qty,
      quantity_reserved  = quantity_reserved  + qty
    WHERE product_id = ?;
    -- proceed
  ELSE
    ROLLBACK; -- out of stock
  END IF;
COMMIT;
```

1. Order Service validates all cart items are still available
2. For each item: Inventory Service acquires row lock, checks and reserves stock
3. If any item out of stock: release all reservations, return error
4. Create `order` record in PostgreSQL with `status = pending`
5. Publish `ORDER_CREATED` event to Kafka
6. Clear cart from Redis

### 5.5 Payment Flow

1. Order Service calls Payment Service with `{orderId, amount, idempotencyKey}`
2. Payment Service calls external gateway (Stripe / Razorpay)
3. On success:
   - Write payment record to PostgreSQL (`status = success`)
   - Publish `PAYMENT_SUCCESS` to Kafka
4. Order Service consumes `PAYMENT_SUCCESS`:
   - Update `orders.status = confirmed`
   - Inventory Service converts `quantity_reserved → quantity_deducted` (permanent deduction)
   - Invalidate inventory Redis cache for affected products
5. On failure:
   - Write `status = failed`
   - Publish `PAYMENT_FAILED` → Order Service releases inventory reservation, cancels order
6. Notification Service sends order confirmation email/push

**Idempotency:** `idempotency_key = hash(orderId + userId)` — safe to retry on gateway timeout.

### 5.6 Flash Sale / Limited Stock

During a flash sale (e.g., 1000 units, 100K users):

1. API Gateway enforces strict rate limiting per user
2. Virtual queue: excess users get a queue token, served in order
3. Inventory cache in Redis shows availability — users see "In Stock" / "Only 3 left"
4. On add-to-cart: optimistic — don't reserve yet
5. On checkout: Inventory Service does the actual reservation with MySQL row lock
6. If stock runs out mid-checkout: user gets "Sorry, item sold out" at payment step
7. Redis inventory cache updated after each reservation (invalidate or decrement)

**Flash sale inventory pre-load:**
- Before sale starts, load inventory counts into Redis
- Inventory Service checks Redis first: `DECR inventory:{productId}` (atomic)
- If result >= 0: proceed to MySQL reservation
- If result < 0: `INCR` back (rollback), return out-of-stock immediately
- Avoids hitting MySQL for every request — Redis absorbs the spike

### 5.7 Order Cancellation & Refund

**Before shipment:**
1. User cancels → Order Service updates `status = cancelled`
2. Inventory Service releases reservation: `quantity_reserved -= qty, quantity_available += qty`
3. Payment Service initiates refund via gateway
4. Invalidate inventory cache

**After shipment:**
- Return flow initiated; refund processed after item received

### 5.8 Reviews & Ratings

1. After order delivered, user submits review → Review Service
2. Write to Cassandra `reviews` table (partition by `product_id`)
3. Async: recalculate `avg_rating` for product, update Cassandra `products` table
4. CDC propagates updated `avg_rating` to Elasticsearch (affects search ranking)
5. Reviews are eventually consistent — slight lag acceptable

## 6. Key Interview Concepts

### Preventing Oversell — The Core Problem
Two users buying the last item simultaneously is the hardest problem. Solution:
- MySQL `SELECT FOR UPDATE` acquires a row-level lock on the inventory row
- Only one transaction proceeds; the other waits, then sees 0 stock and fails
- For flash sales: Redis `DECR` as a fast pre-check before hitting MySQL — absorbs the spike, MySQL only sees requests that passed the Redis check

### Why Cassandra for Products but MySQL for Inventory
Products are read-heavy (23K reads/sec), denormalized, no transactions needed — Cassandra is perfect. Inventory requires row-level locking and ACID transactions to prevent oversell — MySQL is the right tool. Same reasoning as ticket booking: never use Cassandra for transactional data.

### Cart in Redis (Not DB)
Cart is ephemeral — most carts are never converted to orders (browse abandonment). Writing every cart update to a DB wastes resources. Redis Hash per user is fast, cheap, and auto-expires. Only on order placement does cart data get snapshotted into the `order_items` table.

### Price Snapshot on Order
Product prices change. If you store only `product_id` in `order_items`, the order history shows wrong prices after a price change. Solution: snapshot `unit_price` and `title` at order creation time. Order history is always accurate regardless of future price changes.

### CDC for Search Sync
Product Service writes to Cassandra. CDC (Debezium) captures changes from Cassandra's commit log → Kafka → Elasticsearch consumer updates index. Avoids dual-write risk (writing to both Cassandra and Elasticsearch in the same request — if one fails, they diverge). CDC guarantees eventual consistency with no data loss.

### Kafka as Order Event Bus
Order Service publishes events (`ORDER_CREATED`, `PAYMENT_SUCCESS`, `ORDER_SHIPPED`). Consumers: Notification Service, Inventory Service, Analytics Service. Benefits: Order Service stays simple, consumers scale independently, event log enables replay for debugging and recovery.

### Read >> Write Architecture
1000:1 read:write ratio. Product browse path (CDN → Redis → Cassandra) is completely separate from the order path (Order Service → MySQL). They scale independently. CDN absorbs image traffic. Redis absorbs hot product reads. Cassandra handles catalog reads. MySQL only sees transactional writes.

### Inventory Cache Strategy
Showing live stock counts on product pages at 23K reads/sec would destroy MySQL. Solution: cache inventory in Redis with 60s TTL. Users may see slightly stale counts ("In Stock" when actually 0) — acceptable for browse. The actual reservation at checkout always hits MySQL for correctness. Stale cache = bad UX; stale reservation = oversell (unacceptable).

### Saga Pattern for Order + Inventory + Payment
Three distributed operations must succeed together. If payment fails, inventory must be released. Solution: choreography-based saga via Kafka events:
```
ORDER_CREATED → reserve inventory
INVENTORY_RESERVED → initiate payment
PAYMENT_SUCCESS → confirm order, deduct inventory
PAYMENT_FAILED → release inventory, cancel order
```
Each step has a compensating action on failure. No distributed transaction needed.

### Database Sharding for Orders
1M orders/day = 365M/year. Shard by `user_id` — all orders for a user on one shard (efficient order history queries). Cross-shard queries (e.g., seller's orders) handled via Kafka event log or a separate read model (CQRS).

## 7. Failure Scenarios

### Oversell on Inventory Service Crash
- Scenario: inventory reserved in MySQL, then Order Service crashes before creating order record
- Recovery: background reconciliation job checks `quantity_reserved` with no matching pending order → releases reservation after timeout (e.g., 15 min)
- Prevention: outbox pattern — write reservation and order creation in same MySQL transaction

### Payment Gateway Timeout
- Detection: no response within 5s
- Recovery: retry with same `idempotency_key`; after 3 retries, mark `payment_failed`, release inventory reservation, notify user
- Prevention: async payment with webhook callback — gateway calls back on completion; poll for status as fallback

### Redis Cache Failure (Cart + Inventory Cache Lost)
- Impact: cart data lost for active users; inventory cache miss → all reads hit MySQL
- Recovery: Redis Sentinel failover (<30s); cart loss is a UX issue (user must re-add items) — not a data loss issue; inventory reads fall back to MySQL (slower but correct)
- Prevention: Redis Cluster + AOF persistence for cart data; inventory cache is reconstructable from MySQL

### Cassandra Node Failure (Product Catalog)
- Impact: some product reads fail or return stale data
- Recovery: RF=3, QUORUM reads continue on remaining replicas; hinted handoff replays writes to recovered node
- Prevention: multi-datacenter replication; Redis cache absorbs most reads so Cassandra failure impact is limited

### MySQL Primary Failure (Inventory / Orders)
- Impact: inventory reservations and order writes fail
- Recovery: promote replica (<30s); Kafka retains events; Order Service retries after failover
- Prevention: synchronous replication for inventory and orders tables; automated failover

### Elasticsearch Failure
- Impact: search unavailable; product browse degraded
- Recovery: graceful degradation — fall back to Cassandra queries (no full-text, but category/brand filters work); CDC replays missed changes on recovery
- Prevention: Elasticsearch cluster with replicas; non-critical path — orders unaffected

### Flash Sale Overload
- Detection: API Gateway queue depth spikes; error rates increase
- Recovery: virtual queue activates; rate limiting drops excess with 429; Redis `DECR` pre-check absorbs spike before MySQL
- Prevention: pre-scale before known flash sales; load test at 10× expected peak; pre-warm Redis inventory cache before sale starts

### Duplicate Order on Retry
- Scenario: client retries order placement after timeout → two orders created
- Recovery: `idempotency_key = hash(cartId + userId + timestamp_bucket)` — second request finds existing order, returns it
- Prevention: unique constraint on `idempotency_key` in PostgreSQL; client generates key before first attempt and reuses on retry
