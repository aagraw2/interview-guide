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

**API Gateway** — Auth, rate limiting, routing; strict rate limits per user (prevent order flooding)

**User Service** — Registration, KYC, account management; writes to User DB

**Order Service** — Receives and validates orders; writes to Order DB; publishes to Order Queue (Kafka); returns `orderId` immediately

**Order Matching Engine** — The core component; in-memory order book per stock; matches buy/sell orders by price-time priority; produces trade events; single-threaded per stock for consistency

**Trade Service** — Consumes trade events; updates portfolio, deducts funds, records trade; writes to Trade DB

**Market Data Service** — Receives price ticks from exchange feed; distributes to subscribers via WebSocket/SSE; updates Redis with latest prices

**Portfolio Service** — Manages user holdings and P&L; reads from Portfolio DB; updated by Trade Service

**Risk Service** — Pre-trade risk checks: sufficient funds, position limits, circuit breakers; called synchronously before order acceptance

**Settlement Service** — T+2 settlement; transfers actual shares and funds between accounts

**Notification Service** — Kafka consumer; sends order confirmation, trade execution, price alerts

**Order DB (PostgreSQL)** — Order records, status, history

**Trade DB (PostgreSQL)** — Executed trade records

**Portfolio DB (PostgreSQL)** — User holdings, cash balance

**Market Data Store (Redis + Time-Series DB)** — Latest prices in Redis; historical ticks in InfluxDB

**Order Queue (Kafka)** — Orders published here; Matching Engine consumes

**Redis** — Latest stock prices, user session, order status cache

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL (Orders/Trades) | ACID for financial records; relational queries for history |
| PostgreSQL (Portfolio) | ACID for balance/holdings updates; row-level locking |
| Redis | Latest stock prices (sub-ms reads), order status cache, sessions |
| InfluxDB | Historical price ticks — time-series, efficient range queries |
| Kafka | Order queue, trade events, market data distribution |

### PostgreSQL — orders

| Field | Type |
|---|---|
| order_id | UUID (PK) |
| user_id | UUID |
| stock_symbol | VARCHAR |
| order_type | ENUM (market / limit) |
| side | ENUM (buy / sell) |
| quantity | INT |
| price | DECIMAL, nullable (null for market orders) |
| status | ENUM (pending / open / partially_filled / filled / cancelled / rejected) |
| filled_quantity | INT |
| avg_fill_price | DECIMAL, nullable |
| idempotency_key | VARCHAR, unique |
| placed_at | TIMESTAMP |
| updated_at | TIMESTAMP |

### PostgreSQL — trades

| Field | Type |
|---|---|
| trade_id | UUID (PK) |
| buy_order_id | UUID |
| sell_order_id | UUID |
| stock_symbol | VARCHAR |
| quantity | INT |
| price | DECIMAL |
| buyer_id | UUID |
| seller_id | UUID |
| executed_at | TIMESTAMP |

### PostgreSQL — portfolio

| Field | Type |
|---|---|
| user_id | UUID |
| stock_symbol | VARCHAR |
| quantity | INT |
| avg_buy_price | DECIMAL |
| updated_at | TIMESTAMP |

### PostgreSQL — accounts

| Field | Type |
|---|---|
| user_id | UUID (PK) |
| cash_balance | DECIMAL |
| reserved_cash | DECIMAL (held for open buy orders) |
| updated_at | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `price:{symbol}` | String | latest price + timestamp | — (updated on each tick) |
| `orderbook:{symbol}:bids` | ZSET | price → quantity | — |
| `orderbook:{symbol}:asks` | ZSET | price → quantity | — |
| `order:status:{orderId}` | String | status JSON | 3600s |
| `session:{sessionId}` | String | userId | 86400s |

## 5. Key Flows

### 5.1 Auth & Account Setup

1. User registers → KYC verification (identity + bank account linking)
2. On KYC approval: account created with initial cash balance
3. Login → JWT (1hr) + session in Redis
4. API Gateway validates JWT on every request

### 5.2 Order Placement

```
Client → Order Service
              ↓
    Validate: market hours, valid symbol, valid quantity
              ↓
    Risk Service: sufficient funds? position limits? circuit breaker?
              ↓
    Reserve funds (for buy orders): accounts.reserved_cash += order_value
              ↓
    Write order to PostgreSQL (status=pending)
              ↓
    Publish to Kafka order queue (partitioned by stock_symbol)
              ↓
    Return orderId (202 Accepted)
```

1. User places order: `{symbol: AAPL, side: buy, type: limit, qty: 10, price: 150}`
2. Order Service validates: market is open, symbol exists, quantity > 0
3. Risk Service checks:
   - Sufficient cash: `cash_balance - reserved_cash >= qty × price`
   - Position limit: user doesn't exceed max position in one stock
   - Circuit breaker: stock not halted
4. Reserve funds: `reserved_cash += qty × price` (prevents double-spending on concurrent orders)
5. Write order to PostgreSQL with `status = pending`
6. Publish to Kafka, partition key = `stock_symbol` (all orders for same stock → same partition → same Matching Engine)
7. Return `orderId` immediately

### 5.3 Order Matching Engine

The most critical component. Single-threaded per stock for strict price-time priority.

**Order Book structure:**
```
Bids (buy orders) — sorted by price DESC, then time ASC:
  150.00 → [order1 (100 shares, 9:30:01), order2 (50 shares, 9:30:05)]
  149.50 → [order3 (200 shares, 9:30:02)]

Asks (sell orders) — sorted by price ASC, then time ASC:
  150.50 → [order4 (75 shares, 9:30:03)]
  151.00 → [order5 (100 shares, 9:30:04)]
```

**Matching algorithm:**
```
New buy order arrives at price 150.50:
  Best ask = 150.50 (order4, 75 shares)
  150.50 <= 150.50 → MATCH
  Trade: 75 shares at 150.50
  order4 fully filled, removed from book
  New buy order partially filled (75/100 shares)
  Remaining 25 shares added to bid side at 150.50
```

**Flow:**
1. Matching Engine consumes order from Kafka
2. For market order: match against best available price immediately
3. For limit order: check if crossing price exists in book
4. If match: create trade event, update order quantities
5. If no match (limit order): add to order book
6. Publish trade event to Kafka → Trade Service
7. Publish order book update → Market Data Service

**Why single-threaded per stock:**
Matching requires strict ordering — two concurrent matches for the same stock could create inconsistencies. Single-threaded eliminates locking overhead and ensures deterministic price-time priority.

### 5.4 Trade Execution

1. Trade Service consumes trade event from Kafka
2. PostgreSQL transaction:
   - Insert trade record
   - Update buyer's order: `filled_quantity += trade_qty`, `avg_fill_price = weighted avg`
   - Update seller's order similarly
   - Update buyer's portfolio: `holdings += trade_qty`
   - Update seller's portfolio: `holdings -= trade_qty`
   - Update buyer's account: `reserved_cash -= trade_value`, `cash_balance -= trade_value`
   - Update seller's account: `cash_balance += trade_value`
3. Publish `TRADE_EXECUTED` to Kafka → Notification Service

### 5.5 Market Data Distribution

1. Exchange feed sends price ticks to Market Data Service
2. Market Data Service updates Redis: `SET price:{symbol} {price, volume, timestamp}`
3. Broadcasts to all WebSocket subscribers for that symbol
4. Stores tick to InfluxDB for historical queries
5. Client receives real-time price updates via WebSocket

**Fan-out challenge:** 1M users watching 10 stocks each = 10M subscriptions. Solution: Redis Pub/Sub per symbol — Market Data Service publishes once per symbol, all WebSocket servers subscribed to that symbol deliver to their connected clients.

### 5.6 Order Cancellation

1. User cancels order → Order Service
2. Verify order belongs to user and is cancellable (`status = open or pending`)
3. Publish cancel request to Kafka (same partition as original order)
4. Matching Engine processes cancel: removes from order book
5. Order Service updates `status = cancelled`
6. Release reserved funds: `reserved_cash -= remaining_qty × price`

## 6. Key Interview Concepts

### Order Book Data Structure
Order book is an in-memory data structure:
- Bids: max-heap (or sorted map) by price — best bid at top
- Asks: min-heap (or sorted map) by price — best ask at top
- Within same price: FIFO queue (time priority)

For fast matching: use a sorted map (TreeMap/SortedDict) keyed by price, value = queue of orders at that price. Match in O(log N) for price lookup + O(1) for FIFO dequeue.

### Price-Time Priority
Two orders at the same price: the one placed earlier gets filled first. This is the standard fairness rule in all exchanges. Enforced by the FIFO queue within each price level.

### Market vs Limit Orders
- Market order: execute immediately at best available price; always fills (unless no liquidity)
- Limit order: execute only at specified price or better; may not fill immediately; sits in order book

### Why Single-Threaded Matching Engine
Concurrent matching for the same stock creates race conditions:
- Two buy orders both match against the same sell order → oversell
- Locking is expensive and complex
- Single-threaded eliminates all concurrency issues
- Kafka partitioning by symbol ensures all orders for a stock go to the same engine instance

### Circuit Breakers
If a stock price moves >10% in 5 minutes, trading is halted (circuit breaker). Matching Engine stops accepting orders for that stock. Prevents flash crashes. Implemented as a flag per symbol checked before matching.

### Fund Reservation
When a buy order is placed, funds are reserved immediately (`reserved_cash += order_value`). This prevents a user from placing 10 orders worth $10K each with only $10K in their account. On fill: reserved → actual deduction. On cancel: reserved released.

### T+2 Settlement
Trade execution is immediate (T+0). Actual transfer of shares and cash happens T+2 (2 business days later). This is a regulatory requirement. Settlement Service handles the actual transfer asynchronously.

### Idempotency
Network retry could place the same order twice. Solution: `idempotency_key = hash(userId + symbol + side + qty + price + timestamp_bucket)`. Unique constraint in PostgreSQL prevents duplicate orders.

## 7. Failure Scenarios

### Matching Engine Crash
- Detection: health check fails; Kafka consumer stops
- Recovery: restart Matching Engine; rebuild order book from PostgreSQL (open orders) + Kafka (unprocessed messages); resume matching
- Prevention: persist order book state to Redis periodically; warm restart from snapshot

### Duplicate Order (Kafka Redelivery)
- Scenario: Matching Engine processes order, crashes before committing Kafka offset → redelivery
- Recovery: idempotency key on order; Matching Engine checks if order already in book or filled before processing
- Prevention: idempotent order processing

### Fund Reservation Race Condition
- Scenario: two concurrent orders both pass the fund check before either reserves
- Recovery: PostgreSQL `UPDATE accounts SET reserved_cash = reserved_cash + ? WHERE cash_balance - reserved_cash >= ?` — atomic check-and-update; one fails if insufficient funds
- Prevention: row-level lock on account during reservation

### Market Data Feed Failure
- Impact: stale prices shown to users; order placement unaffected (uses order book prices)
- Recovery: reconnect to exchange feed; Redis retains last known prices; users see "delayed data" warning
- Prevention: multiple feed connections; failover to backup feed provider

### PostgreSQL Failure (Trade DB)
- Impact: trade records not persisted; Kafka trade events not acknowledged
- Recovery: promote replica; Trade Service retries from Kafka; idempotency prevents duplicate trade records
- Prevention: synchronous replication; automated failover
