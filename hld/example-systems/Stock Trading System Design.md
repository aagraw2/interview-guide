# Stock Trading System Design

## System Overview
A stock trading platform (think Zerodha / Robinhood / NSE) where users place buy/sell orders for stocks, orders are matched via an order book, trades are executed, portfolios updated, and real-time market data is streamed to clients.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- User registration, KYC, and account management
- View real-time stock prices and market data
- Place market and limit orders (buy/sell)
- Order matching via order book
- Trade execution and settlement
- Portfolio and holdings management
- Order history and trade confirmations
- Watchlist management

### Non-Functional Requirements
- Availability: 99.99% during market hours
- Latency: <10ms for order placement; <1ms for order matching
- Throughput: 1M+ orders/sec at peak
- Consistency: Strong — no duplicate orders, no incorrect trade execution
- Durability: Every order and trade must be permanently recorded
- Fairness: Orders matched in strict price-time priority

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 10M active traders, 1M DAU during market hours
- 1M orders/day average; 10M orders/day during volatile market
- 10K stocks listed
- Market data updates: 1M price ticks/sec across all stocks
- Market hours: 6.5hr/day (9:30am–4pm)

### Traffic
```
Orders/sec (avg)        = 1M / 23400s ≈ 43/sec
Orders/sec (peak)       = 10M / 23400s ≈ 427/sec

Market data ticks/sec   = 1M/sec → pushed to all subscribers
Price subscribers       = 1M users × 10 stocks watched = 10M subscriptions
```

### Storage
```
Orders/day          = 10M × 500B = 5GB/day → ~1.8TB/year
Trades/day          = 5M × 500B = 2.5GB/day
Market data ticks   = 1M/sec × 100B = 100MB/sec → retain 1 day = ~8.6TB
Portfolio records   = 10M users × 100 holdings × 200B = 200GB
```

## 3. Core Components

**LB + API Gateway** — Auth, rate limiting, routing, round-robin; strict rate limits per user (prevent order flooding)

**WebSocket Gateway** — Separate from API Gateway; handles persistent WebSocket connections with session stickiness; used by Price Tracker Service and order status updates

**User Service** — Registration, KYC verification, account management; writes to User DB (stores encrypted PAN, bank details, portfolio metadata)

**Order Service** — Receives and validates orders; writes to Order DB; publishes to Kafka `raw_orders` topic; returns `orderId` immediately

**Validator** — Kafka consumer on `raw_orders`; performs risk checks (sufficient funds, position limits, circuit breakers, KYC status); publishes to `verified_orders` or `rejected_orders` topics; separating validation from order placement keeps Order Service fast and stateless

**Order Tracker Service** — Consumes `verified_orders`; places orders on the exchange via Exchange Gateway; receives order status updates back from exchange (filled, partially filled, rejected); updates Order DB and publishes to Kafka `order_status` topic

**Exchange Gateway** — Dedicated adapter to NSE/BSE; maintains persistent WebSocket connections to the exchange (400–500 symbols/ws); translates internal order format to exchange protocol (FIX/proprietary); receives live price feeds and order acknowledgements; the only component that talks directly to the exchange

**Price Ingester Service** — Consumes live price updates from Exchange Gateway; writes historical ticks to InfluxDB; publishes to Kafka `stock_price` topic for downstream consumers

**Price Tracker Service (WebSocket Server)** — Subscribes to Redis Pub/Sub channels per stock symbol; pushes real-time price updates to connected clients via WebSocket; users subscribe to specific symbols they care about

**WatchList Service** — Manages user watchlists (add/remove symbols); reads/writes to Watch DB; used by Price Tracker to know which symbols a user is subscribed to

**Portfolio Service** — Manages user holdings and P&L calculation; reads from Trade DB; updated when trades are confirmed

**Payment Service** — Handles fund addition and withdrawal; integrates with Payment Gateway; writes to Payment DB

**Notification Service** — Kafka consumer; sends order confirmation, trade execution alerts, price alerts via push/email

**Order DB (PostgreSQL)** — Order records, status, history; `order_id, user_id, stock_id, order_type, price, quantity, trade_id, status`

**Trade DB (PostgreSQL)** — Executed trade records; read by Portfolio Service for P&L calculation

**User DB (PostgreSQL)** — User profiles, KYC data (encrypted PAN, bank details, portfolio metadata)

**Watch DB** — User watchlist mappings (userId → symbols)

**Payment DB (PostgreSQL)** — Payment records, fund transfers; `payment_id, user_id, transaction_id, amount, currency, timestamp, metadata`

**InfluxDB** — Historical price ticks per symbol; time-series optimized; used for charts and past price queries

**Redis Pub/Sub** — Price Ingester publishes price updates per symbol; Price Tracker Service instances subscribe; fan-out to all connected WebSocket clients watching that symbol

**Kafka** — Two logical streams:
- Order flow: `raw_orders` → Validator → `verified_orders` / `rejected_orders` → Order Tracker
- Market data: Exchange Gateway → `stock_price`, `order_status` → Price Ingester, Notification Service

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL (Order/Trade DB) | ACID for financial records; relational queries for history |
| PostgreSQL (User/Payment DB) | Structured user + KYC data, ACID, encrypted sensitive fields |
| InfluxDB | Historical price ticks — time-series, efficient range queries for charts |
| Redis Pub/Sub | Sub-ms price fan-out to WebSocket clients; publish once per symbol, deliver to all subscribers |
| Kafka | Durable order pipeline (raw → verified → exchange) and market data stream |

### PostgreSQL — orders

| Field | Type |
|---|---|
| order_id | UUID (PK) |
| user_id | UUID |
| stock_id | UUID |
| order_type | ENUM (market / limit) |
| side | ENUM (buy / sell) |
| quantity | INT |
| price | DECIMAL, nullable |
| trade_id | UUID, nullable |
| status | ENUM (pending / verified / rejected / open / partially_filled / filled / cancelled) |
| placed_at | TIMESTAMP |
| updated_at | TIMESTAMP |

### PostgreSQL — trades

| Field | Type |
|---|---|
| trade_id | UUID (PK) |
| buy_order_id | UUID |
| sell_order_id | UUID |
| stock_id | UUID |
| quantity | INT |
| price | DECIMAL |
| buyer_id | UUID |
| seller_id | UUID |
| executed_at | TIMESTAMP |

### PostgreSQL — users

| Field | Type |
|---|---|
| user_id | UUID (PK) |
| email | VARCHAR, unique |
| phone | VARCHAR |
| password_hash | VARCHAR |
| pan_card | VARCHAR (encrypted) |
| bank_details | JSONB (encrypted) |
| portfolio | JSONB (encrypted metadata) |
| kyc_status | ENUM (pending / verified / rejected) |
| created_at | TIMESTAMP |

### PostgreSQL — payments

| Field | Type |
|---|---|
| payment_id | UUID (PK) |
| user_id | UUID |
| transaction_id | VARCHAR |
| amount | DECIMAL |
| currency | VARCHAR |
| status | ENUM (pending / success / failed) |
| timestamp | TIMESTAMP |
| metadata | JSONB |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `price:{symbol}` | Pub/Sub channel | latest price tick JSON | — |
| `order:status:{orderId}` | String | status JSON | 3600s |
| `session:{sessionId}` | String | userId | 86400s |

## 5. Key Flows

### 5.1 Auth & Account Setup

1. User registers → User Service → KYC verification (PAN, bank account, identity)
2. On KYC approval: account activated, encrypted KYC data stored in User DB
3. Login → JWT (1hr) + session in Redis
4. API Gateway validates JWT on every HTTP request; WebSocket Gateway validates on connect

### 5.2 Order Placement & Validation

```
Client → API Gateway → Order Service
                            ↓
              Write to Order DB (status=pending)
              Publish to Kafka: raw_orders
                            ↓
                        Validator
              (funds check, position limits, KYC, circuit breaker)
                            ↓
              verified_orders → Order Tracker → Exchange Gateway → NSE/BSE
              rejected_orders → Order DB (status=rejected) → Notification
```

1. User places order: `{stockId, side: buy, type: limit, qty: 10, price: 150}`
2. Order Service validates format, writes to Order DB (`status = pending`), publishes to `raw_orders`
3. Validator consumes `raw_orders`:
   - KYC verified?
   - Sufficient funds: `cash_balance - reserved >= qty × price`
   - Position limit not exceeded
   - Stock not circuit-breaker halted
   - Time & session validity
4. Valid → publish to `verified_orders`; invalid → publish to `rejected_orders`, update Order DB, notify user
5. Order Tracker consumes `verified_orders` → calls Exchange Gateway → places order on NSE/BSE
6. Exchange acknowledges → Order Tracker updates `status = open` in Order DB
7. Reserve funds for buy orders on validation success

### 5.3 Order Matching (Exchange Side)

The actual matching happens at NSE/BSE, not in our system. The exchange runs its own order book with price-time priority. Our system:
- Sends orders to exchange via Exchange Gateway
- Receives fill notifications back (fully filled, partially filled, rejected)
- Order Tracker processes these status updates, writes to Order DB and Trade DB
- Publishes `order_status` events to Kafka → Notification Service

**For interview context — if asked about internal matching engine:**
If building a proprietary exchange (not a broker), the matching engine is in-memory, single-threaded per symbol, with Kafka partitioned by symbol. See order book structure in concepts section.

### 5.4 Real-Time Price Updates

```
NSE/BSE → Exchange Gateway (400–500 symbols/ws)
                ↓
         Price Ingester Service
                ↓              ↓
           InfluxDB        Kafka: stock_price
                                ↓
                         Redis Pub/Sub (publish per symbol)
                                ↓
                    Price Tracker Service (WebSocket Server)
                                ↓
                    Connected clients subscribed to that symbol
```

1. Exchange Gateway maintains WebSocket connections to NSE/BSE (400–500 symbols per connection)
2. Price Ingester consumes ticks: writes to InfluxDB (historical), publishes to Redis Pub/Sub channel `price:{symbol}`
3. Price Tracker Service instances subscribe to Redis Pub/Sub channels for symbols their connected users watch
4. On price update: push to all WebSocket clients subscribed to that symbol
5. User subscribes to symbol via WatchList Service → Price Tracker starts subscribing to that Redis channel

### 5.5 Portfolio & P&L

1. Trade confirmed → Trade DB updated
2. Portfolio Service reads Trade DB to compute holdings and average buy price
3. P&L = (current price from Redis/InfluxDB - avg buy price) × quantity
4. Portfolio page reads current prices from InfluxDB (past price) or Redis Pub/Sub (live)

### 5.6 Order Cancellation

1. User cancels order → Order Service
2. Verify order is cancellable (`status = open`)
3. Order Tracker sends cancel request to Exchange Gateway → exchange
4. Exchange confirms cancel → Order Tracker updates `status = cancelled`
5. Release reserved funds

## 6. Key Interview Concepts

### Exchange Gateway — The Critical Boundary
In a broker system (Zerodha, Robinhood), you don't run your own matching engine — the exchange (NSE/BSE) does. The Exchange Gateway is the adapter between your system and the exchange:
- Maintains persistent WebSocket/FIX connections to the exchange
- Handles 400–500 symbols per WebSocket connection
- Translates internal order format to exchange protocol
- Receives live price feeds and order fill notifications
- Single point of exchange connectivity — all other services go through it

### Validator as a Separate Service
Separating validation from Order Service keeps Order Service fast (just write + publish). Validator does the expensive checks (DB reads for balance, KYC status, position limits) asynchronously. Kafka topics make the separation clean:
- `raw_orders` → Validator → `verified_orders` / `rejected_orders`
- Rejected orders never reach the exchange

### Redis Pub/Sub for Price Fan-out
1M users watching 10 stocks = 10M subscriptions. Naive approach: update 10M Redis keys on each tick. Better: Redis Pub/Sub — Price Ingester publishes once to `price:{symbol}` channel; all Price Tracker Service instances subscribed to that channel deliver to their connected WebSocket clients. Publish once, fan-out to all.

### Order Book Data Structure (for proprietary exchange)
If asked about building the matching engine itself:
- Bids: sorted map (price DESC) → FIFO queue per price level
- Asks: sorted map (price ASC) → FIFO queue per price level
- Match: O(log N) price lookup + O(1) FIFO dequeue
- Single-threaded per symbol — eliminates locking, ensures price-time priority

### Price-Time Priority
Two orders at the same price: earlier order fills first. Enforced by FIFO queue within each price level. Standard fairness rule across all exchanges.

### Market vs Limit Orders
- Market order: fill immediately at best available price
- Limit order: fill only at specified price or better; sits in order book if no match

### Circuit Breakers
Stock price moves >10% in 5 minutes → trading halted. Validator checks circuit breaker flag before forwarding to exchange. Prevents flash crashes. Flag set by a monitoring service watching price movements.

### Fund Reservation
On order validation: reserve `qty × price` from user's balance. Prevents placing 10 orders with only enough funds for 1. On fill: reserved → actual deduction. On cancel/reject: reserved released.

### T+2 Settlement
Trade execution is immediate (T+0). Actual share and cash transfer happens T+2 (regulatory requirement). Settlement Service handles the deferred transfer asynchronously.

### InfluxDB for Historical Prices
Price ticks are time-series data — InfluxDB is purpose-built for this. Efficient range queries: "give me AAPL prices for the last 1 hour" is a simple time-range query. Used for charts on the portfolio and stock detail pages.

## 7. Failure Scenarios

### Exchange Gateway Disconnection
- Detection: WebSocket connection to NSE/BSE drops
- Recovery: reconnect immediately with exponential backoff; buffer outgoing orders locally; replay on reconnect
- Prevention: multiple WebSocket connections to exchange; heartbeat monitoring

### Validator Crash
- Detection: Kafka consumer lag on `raw_orders` grows; orders stuck in pending
- Recovery: another Validator instance picks up from Kafka; idempotent validation (same order → same result)
- Prevention: multiple Validator instances in consumer group

### Order Tracker Crash Mid-Placement
- Detection: order placed on exchange but status not updated in DB
- Recovery: on restart, query exchange for status of all open orders; reconcile with Order DB
- Prevention: exchange provides order status API for reconciliation; idempotent status updates

### Price Feed Failure
- Impact: stale prices shown to users; order placement unaffected (exchange handles pricing)
- Recovery: reconnect to exchange feed; Redis Pub/Sub retains last published price; users see "delayed data" warning
- Prevention: multiple feed connections; fallback to exchange REST API for last price

### Fund Reservation Race Condition
- Scenario: two concurrent orders both pass the balance check before either reserves
- Recovery: atomic DB update: `UPDATE accounts SET reserved = reserved + ? WHERE balance - reserved >= ?` — one fails
- Prevention: row-level lock on account during reservation check

### PostgreSQL Failure (Order DB)
- Impact: order writes fail; Kafka messages not acknowledged; orders not lost
- Recovery: promote replica; Order Service retries; idempotency key prevents duplicate orders
- Prevention: synchronous replication; automated failover
