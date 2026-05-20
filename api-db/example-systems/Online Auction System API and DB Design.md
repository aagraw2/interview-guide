# Online Auction System API and DB Design

## System Overview
An online auction platform (think eBay) where sellers list items with a starting price and end time, buyers place bids, and the highest bidder at auction close wins. The core challenge is handling concurrent bids correctly — two users bidding simultaneously must result in exactly one winner per bid increment.

## 1. Requirements

### Functional Requirements
- Sellers can create auction listings with starting price, reserve price, and end time
- Buyers can place bids on active auctions
- System enforces minimum bid increment rules
- Automatic bid extension if a bid is placed in the final minutes
- Bid history visible to all users
- Winner notification and payment initiation at auction close
- Buy-it-now option at a fixed price

### Non-Functional Requirements
- Consistency: Strong — concurrent bids must be serialized; no two users can win the same bid
- Availability: 99.9% — auctions must close on time even under load
- Latency: <500ms for bid placement; <200ms for current price reads
- Scalability: 10M active listings, 100K concurrent bidders during peak
- Auditability: Complete bid history must be immutable

## 2. Scale Estimation

```
Active listings         = 10M
Bids/day                = 50M
Bids/sec (avg)          = 50M / 86400 ≈ 580/sec
Peak bids/sec           ≈ 5,000/sec (auction closing rush)

Auction close events/hr = varies; spike at top-of-hour endings
Bid record size         = ~200B
Bid storage/year        = 50M × 365 × 200B ≈ 3.6TB

Listing storage         = 10M × 5KB = 50GB
```

## 3. API Design

### Key Endpoints

#### Create Auction Listing
```
POST /api/v1/auctions
Authorization: Bearer <token>

Request:
{
  "title": "Vintage Rolex Submariner 1965",
  "description": "Original dial, box and papers included",
  "category": "watches",
  "starting_price": 5000.00,
  "reserve_price": 8000.00,    // optional, hidden from buyers
  "buy_now_price": 15000.00,   // optional
  "min_bid_increment": 100.00,
  "ends_at": "2025-07-01T18:00:00Z",
  "images": ["https://cdn.example.com/img1.jpg"]
}

Response 201:
{
  "auction_id": "auc_abc123",
  "status": "ACTIVE",
  "current_price": 5000.00,
  "ends_at": "2025-07-01T18:00:00Z"
}
```

#### Place Bid
```
POST /api/v1/auctions/{auctionId}/bids
Authorization: Bearer <token>

Request:
{
  "amount": 5500.00,
  "max_auto_bid": 7000.00   // optional: auto-bid up to this amount
}

Response 201:
{
  "bid_id": "bid_xyz",
  "auction_id": "auc_abc123",
  "amount": 5500.00,
  "status": "WINNING",       // WINNING, OUTBID
  "current_price": 5500.00,
  "ends_at": "2025-07-01T18:00:00Z"
}

Errors:
- 409: Bid too low (must be >= current_price + min_increment)
- 409: Auction has ended
- 409: You are already the highest bidder
```

#### Get Auction Details
```
GET /api/v1/auctions/{auctionId}

Response 200:
{
  "auction_id": "auc_abc123",
  "title": "Vintage Rolex Submariner 1965",
  "seller": { "id": "user_seller", "username": "watchcollector" },
  "current_price": 6200.00,
  "bid_count": 14,
  "reserve_met": true,
  "buy_now_price": 15000.00,
  "ends_at": "2025-07-01T18:00:00Z",
  "status": "ACTIVE",
  "time_remaining_seconds": 3600
}
```

#### Get Bid History
```
GET /api/v1/auctions/{auctionId}/bids?page=1

Response 200:
{
  "bids": [
    { "bid_id": "bid_xyz", "bidder": "user_***", "amount": 6200.00, "placed_at": "..." },
    { "bid_id": "bid_abc", "bidder": "user_***", "amount": 5900.00, "placed_at": "..." }
  ],
  "total": 14
}
```

#### Buy Now
```
POST /api/v1/auctions/{auctionId}/buy-now
Authorization: Bearer <token>

Response 200:
{
  "auction_id": "auc_abc123",
  "status": "SOLD",
  "final_price": 15000.00,
  "winner_id": "user_buyer"
}
```

### Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| POST | /api/v1/auctions | Create auction listing |
| GET | /api/v1/auctions | Search/browse auctions |
| GET | /api/v1/auctions/{auctionId} | Get auction details |
| POST | /api/v1/auctions/{auctionId}/bids | Place a bid |
| GET | /api/v1/auctions/{auctionId}/bids | Get bid history |
| POST | /api/v1/auctions/{auctionId}/buy-now | Buy at fixed price |
| DELETE | /api/v1/auctions/{auctionId} | Cancel listing (seller, before first bid) |

## 4. Database Schema

```sql
CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    username        VARCHAR(50) NOT NULL UNIQUE,
    email           VARCHAR(255) NOT NULL UNIQUE,
    avg_seller_rating   DECIMAL(3, 2) DEFAULT 0,
    avg_buyer_rating    DECIMAL(3, 2) DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE auctions (
    id                  BIGSERIAL PRIMARY KEY,
    seller_id           BIGINT NOT NULL REFERENCES users(id),
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    category            VARCHAR(100),
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
                        -- ACTIVE, ENDED, SOLD, CANCELLED
    starting_price      DECIMAL(12, 2) NOT NULL,
    reserve_price       DECIMAL(12, 2),          -- NULL = no reserve
    buy_now_price       DECIMAL(12, 2),           -- NULL = no buy-now
    min_bid_increment   DECIMAL(10, 2) NOT NULL DEFAULT 1.00,
    current_price       DECIMAL(12, 2) NOT NULL,  -- denormalized for fast reads
    current_winner_id   BIGINT REFERENCES users(id),
    bid_count           INT NOT NULL DEFAULT 0,
    reserve_met         BOOLEAN NOT NULL DEFAULT FALSE,
    ends_at             TIMESTAMPTZ NOT NULL,
    auto_extend_minutes INT NOT NULL DEFAULT 5,   -- extend if bid in final N minutes
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_auctions_status ON auctions (status);
CREATE INDEX idx_auctions_ends_at ON auctions (ends_at) WHERE status = 'ACTIVE';
CREATE INDEX idx_auctions_seller_id ON auctions (seller_id);
CREATE INDEX idx_auctions_category ON auctions (category, status);
-- Full-text search on title
CREATE INDEX idx_auctions_title_fts ON auctions USING GIN (to_tsvector('english', title));

-- Immutable bid history — never updated, only inserted
CREATE TABLE bids (
    id              BIGSERIAL PRIMARY KEY,
    auction_id      BIGINT NOT NULL REFERENCES auctions(id),
    bidder_id       BIGINT NOT NULL REFERENCES users(id),
    amount          DECIMAL(12, 2) NOT NULL,
    bid_type        VARCHAR(20) NOT NULL DEFAULT 'MANUAL',  -- MANUAL, AUTO
    is_winning      BOOLEAN NOT NULL DEFAULT FALSE,
    placed_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bids_auction_id ON bids (auction_id, placed_at DESC);
CREATE INDEX idx_bids_bidder_id ON bids (bidder_id);

-- Auto-bid configuration (proxy bidding)
CREATE TABLE auto_bids (
    id              BIGSERIAL PRIMARY KEY,
    auction_id      BIGINT NOT NULL REFERENCES auctions(id),
    bidder_id       BIGINT NOT NULL REFERENCES users(id),
    max_amount      DECIMAL(12, 2) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (auction_id, bidder_id)
);

CREATE INDEX idx_auto_bids_auction_id ON auto_bids (auction_id) WHERE is_active = TRUE;

-- Auction results after close
CREATE TABLE auction_results (
    auction_id      BIGINT PRIMARY KEY REFERENCES auctions(id),
    winner_id       BIGINT REFERENCES users(id),
    winning_bid_id  BIGINT REFERENCES bids(id),
    final_price     DECIMAL(12, 2),
    reserve_met     BOOLEAN NOT NULL,
    closed_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE auction_images (
    id          BIGSERIAL PRIMARY KEY,
    auction_id  BIGINT NOT NULL REFERENCES auctions(id),
    url         VARCHAR(500) NOT NULL,
    position    INT NOT NULL DEFAULT 0
);
```

### Schema Summary

| Table | Purpose |
|-------|---------|
| auctions | Listings with current price (denormalized) |
| bids | Immutable bid history |
| auto_bids | Proxy bidding configuration per user per auction |
| auction_results | Final outcome after auction closes |
| auction_images | Listing photos |

## 5. Key Design Decisions

### 5.1 Concurrent Bid Handling with SELECT FOR UPDATE

The most critical challenge: two users submit bids simultaneously. Both read the current price as $5,000 and both try to place a $5,500 bid. Without locking, both succeed — but only one should win.

```sql
BEGIN;

-- Lock the auction row for the duration of this transaction
SELECT id, current_price, status, ends_at, min_bid_increment
FROM auctions
WHERE id = $1
FOR UPDATE;

-- Validate: auction active, not ended, bid high enough
-- If valid, insert bid and update auction
INSERT INTO bids (auction_id, bidder_id, amount, is_winning)
VALUES ($1, $2, $3, TRUE);

UPDATE auctions
SET current_price = $3,
    current_winner_id = $2,
    bid_count = bid_count + 1,
    updated_at = NOW()
WHERE id = $1;

COMMIT;
```

`SELECT FOR UPDATE` serializes concurrent bids at the database level. The second transaction waits for the first to commit, then re-reads the updated price and correctly rejects the duplicate bid.

### 5.2 Denormalized Current Price

`current_price` and `current_winner_id` are stored on the `auctions` table even though they could be derived from the `bids` table. This avoids an expensive `MAX(amount)` query on every auction page load. The `bids` table is the source of truth; `auctions` is the read-optimized cache.

### 5.3 Auction End-Time Extension

To prevent "sniping" (bidding in the last second), a bid placed within the final `auto_extend_minutes` extends the auction:
```sql
UPDATE auctions
SET ends_at = GREATEST(ends_at, NOW() + INTERVAL '5 minutes')
WHERE id = $1;
```
This runs inside the same `SELECT FOR UPDATE` transaction as the bid.

### 5.4 Proxy (Auto) Bidding

Users can set a maximum bid. When outbid, the system automatically places the minimum necessary counter-bid on their behalf. The `auto_bids` table stores the max amount. After each manual bid, the system checks for active auto-bids and triggers counter-bids in order of max amount.

### 5.5 Reserve Price Confidentiality

`reserve_price` is stored in the database but never returned in API responses to buyers. The API only returns `reserve_met: true/false`. This is enforced at the API layer — the column is never included in buyer-facing SELECT queries.

## 6. Failure Scenarios

### Concurrent Bid Race Condition
- **Impact**: Without locking, two users could both "win" the same bid increment
- **Recovery**: `SELECT FOR UPDATE` serializes bids; the second transaction sees the updated price and is rejected if too low
- **Prevention**: Always use `SELECT FOR UPDATE` within a transaction for bid placement; never use optimistic locking for auctions (too many conflicts at peak)

### Auction Close Timing Failure
- **Impact**: Auction doesn't close on time; bidders continue placing bids after intended end
- **Recovery**: Scheduled job (cron every minute) closes expired auctions; idempotent close operation
- **Prevention**: Use a reliable job scheduler (not cron); lock the auction row during close to prevent concurrent close attempts

### Seller Cancels Auction with Active Bids
- **Impact**: Bidders lose their bids; trust damage
- **Recovery**: Only allow cancellation before the first bid; after first bid, require admin approval
- **Prevention**: Enforce `bid_count = 0` check before allowing cancellation

### Payment Failure After Auction Win
- **Impact**: Winner can't pay; item unsold
- **Recovery**: Notify winner; allow 48-hour payment window; offer to second-highest bidder if unpaid
- **Prevention**: Pre-authorize payment method before allowing bids above a threshold
