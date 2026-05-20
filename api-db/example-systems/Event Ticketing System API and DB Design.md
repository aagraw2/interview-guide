# Event Ticketing System API and DB Design

## System Overview
An event ticketing platform (think Ticketmaster or Eventbrite) where organizers create events with venue sections and seat maps, buyers select and purchase specific seats, and tickets can be transferred between users. The hardest problem is preventing two buyers from purchasing the same seat simultaneously during high-demand on-sales.

## 1. Requirements

### Functional Requirements
- Organizers create events with venue, sections, rows, and individual seats
- Buyers browse available seats and select specific ones
- Seat hold with TTL (10 minutes) while buyer completes checkout
- Purchase tickets with payment processing
- Transfer tickets to another user
- QR code generation for ticket validation at the door
- Waitlist for sold-out events

### Non-Functional Requirements
- Consistency: Strong — no two buyers can purchase the same seat
- Availability: 99.99% — on-sale events have massive traffic spikes
- Latency: <500ms for seat selection; <2s for purchase
- Scalability: Handle 100K concurrent users during popular on-sales
- Durability: Purchased tickets must never be lost

## 2. Scale Estimation

```
Events/day              = 10K
Avg seats/event         = 5,000
Total seats in system   = 10K × 5K = 50M active seats

Peak on-sale traffic    = 100K concurrent users for a single event
Seat hold requests/sec  = 50K/sec during peak on-sale
Purchase requests/sec   = 5K/sec during peak

Ticket storage          = 50M × 1KB = 50GB
```

## 3. API Design

### Key Endpoints

#### Get Seat Map
```
GET /api/v1/events/{eventId}/seats?section=floor

Response 200:
{
  "event_id": "evt_abc",
  "sections": [
    {
      "id": "sec_floor",
      "name": "Floor",
      "rows": [
        {
          "row": "A",
          "seats": [
            { "id": "seat_001", "number": "1", "status": "AVAILABLE", "price": 150.00 },
            { "id": "seat_002", "number": "2", "status": "HELD",      "price": 150.00 },
            { "id": "seat_003", "number": "3", "status": "SOLD",      "price": 150.00 }
          ]
        }
      ]
    }
  ]
}
```

#### Hold Seats
```
POST /api/v1/events/{eventId}/holds
Authorization: Bearer <token>

Request:
{
  "seat_ids": ["seat_001", "seat_004", "seat_005"]
}

Response 201:
{
  "hold_id": "hold_xyz",
  "seat_ids": ["seat_001", "seat_004", "seat_005"],
  "expires_at": "2025-06-01T10:10:00Z",
  "total_price": 450.00
}

Errors:
- 409: One or more seats no longer available (returns which seats)
```

#### Purchase Tickets
```
POST /api/v1/holds/{holdId}/purchase
Authorization: Bearer <token>

Request:
{
  "payment_method_id": "pm_abc123",
  "billing_name": "Jane Smith"
}

Response 201:
{
  "order_id": "ord_xyz",
  "tickets": [
    {
      "ticket_id": "tkt_001",
      "event_name": "Taylor Swift | The Eras Tour",
      "seat": "Floor A-1",
      "qr_code": "https://tickets.example.com/qr/tkt_001",
      "barcode": "TKT-2025-001-ABC"
    }
  ],
  "total_charged": 450.00
}
```

#### Transfer Ticket
```
POST /api/v1/tickets/{ticketId}/transfer
Authorization: Bearer <token>

Request: { "recipient_email": "friend@example.com" }

Response 200:
{
  "ticket_id": "tkt_001",
  "status": "TRANSFER_PENDING",
  "recipient_email": "friend@example.com",
  "expires_at": "2025-06-02T10:00:00Z"
}
```

### Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/events | Browse events |
| GET | /api/v1/events/{eventId} | Event details |
| GET | /api/v1/events/{eventId}/seats | Seat map with availability |
| POST | /api/v1/events/{eventId}/holds | Hold seats (TTL) |
| DELETE | /api/v1/holds/{holdId} | Release hold early |
| POST | /api/v1/holds/{holdId}/purchase | Purchase held seats |
| GET | /api/v1/tickets/{ticketId} | Get ticket details |
| POST | /api/v1/tickets/{ticketId}/transfer | Transfer ticket |
| POST | /api/v1/events/{eventId}/waitlist | Join waitlist |

## 4. Database Schema

```sql
CREATE TABLE venues (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    address     TEXT NOT NULL,
    city        VARCHAR(100) NOT NULL,
    capacity    INT NOT NULL
);

CREATE TABLE events (
    id              BIGSERIAL PRIMARY KEY,
    organizer_id    BIGINT NOT NULL REFERENCES users(id),
    venue_id        BIGINT NOT NULL REFERENCES venues(id),
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    event_date      TIMESTAMPTZ NOT NULL,
    doors_open_at   TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
                    -- DRAFT, ON_SALE, SOLD_OUT, CANCELLED, COMPLETED
    total_capacity  INT NOT NULL,
    tickets_sold    INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_events_event_date ON events (event_date);
CREATE INDEX idx_events_status ON events (status);

CREATE TABLE sections (
    id          BIGSERIAL PRIMARY KEY,
    event_id    BIGINT NOT NULL REFERENCES events(id),
    name        VARCHAR(100) NOT NULL,   -- "Floor", "Section 101", "VIP"
    description TEXT,
    UNIQUE (event_id, name)
);

CREATE TABLE rows (
    id          BIGSERIAL PRIMARY KEY,
    section_id  BIGINT NOT NULL REFERENCES sections(id),
    row_label   VARCHAR(10) NOT NULL,   -- "A", "B", "1", "2"
    UNIQUE (section_id, row_label)
);

-- Individual seats — one row per physical seat
CREATE TABLE seats (
    id          BIGSERIAL PRIMARY KEY,
    row_id      BIGINT NOT NULL REFERENCES rows(id),
    seat_number VARCHAR(10) NOT NULL,
    price       DECIMAL(10, 2) NOT NULL,
    status      VARCHAR(20) NOT NULL DEFAULT 'AVAILABLE',
                -- AVAILABLE, HELD, SOLD, BLOCKED
    UNIQUE (row_id, seat_number)
);

CREATE INDEX idx_seats_row_id ON seats (row_id);
CREATE INDEX idx_seats_status ON seats (status) WHERE status = 'AVAILABLE';

-- Temporary holds with TTL
CREATE TABLE seat_holds (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL REFERENCES users(id),
    event_id    BIGINT NOT NULL REFERENCES events(id),
    status      VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
                -- ACTIVE, PURCHASED, RELEASED, EXPIRED
    total_price DECIMAL(10, 2) NOT NULL,
    expires_at  TIMESTAMPTZ NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_seat_holds_user_id ON seat_holds (user_id);
CREATE INDEX idx_seat_holds_expires_at ON seat_holds (expires_at) WHERE status = 'ACTIVE';

CREATE TABLE seat_hold_items (
    hold_id     BIGINT NOT NULL REFERENCES seat_holds(id),
    seat_id     BIGINT NOT NULL REFERENCES seats(id),
    PRIMARY KEY (hold_id, seat_id)
);

CREATE TABLE orders (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES users(id),
    hold_id         BIGINT NOT NULL REFERENCES seat_holds(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'PENDING',
                    -- PENDING, COMPLETED, REFUNDED, CANCELLED
    total_amount    DECIMAL(10, 2) NOT NULL,
    payment_id      VARCHAR(100),
    idempotency_key VARCHAR(100) UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Individual tickets (one per seat per order)
CREATE TABLE tickets (
    id              BIGSERIAL PRIMARY KEY,
    order_id        BIGINT NOT NULL REFERENCES orders(id),
    seat_id         BIGINT NOT NULL REFERENCES seats(id) UNIQUE,  -- one ticket per seat
    owner_id        BIGINT NOT NULL REFERENCES users(id),
    barcode         VARCHAR(50) NOT NULL UNIQUE,
    status          VARCHAR(20) NOT NULL DEFAULT 'VALID',
                    -- VALID, USED, TRANSFERRED, REFUNDED, CANCELLED
    scanned_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tickets_owner_id ON tickets (owner_id);
CREATE INDEX idx_tickets_barcode ON tickets (barcode);

CREATE TABLE ticket_transfers (
    id              BIGSERIAL PRIMARY KEY,
    ticket_id       BIGINT NOT NULL REFERENCES tickets(id),
    from_user_id    BIGINT NOT NULL REFERENCES users(id),
    to_user_id      BIGINT REFERENCES users(id),
    to_email        VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'PENDING',
                    -- PENDING, ACCEPTED, DECLINED, EXPIRED
    expires_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE waitlist_entries (
    id          BIGSERIAL PRIMARY KEY,
    event_id    BIGINT NOT NULL REFERENCES events(id),
    user_id     BIGINT NOT NULL REFERENCES users(id),
    quantity    INT NOT NULL DEFAULT 1,
    joined_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (event_id, user_id)
);

CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    full_name   VARCHAR(255) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Schema Summary

| Table | Purpose |
|-------|---------|
| venues | Physical venue information |
| events | Event listings with status |
| sections / rows / seats | Hierarchical seat map |
| seat_holds | Temporary seat holds with TTL |
| orders | Purchase records |
| tickets | Individual tickets with barcodes |
| ticket_transfers | Transfer workflow |
| waitlist_entries | Waitlist for sold-out events |

## 5. Key Design Decisions

### 5.1 Seat Hold with TTL — Preventing Double-Booking

When a buyer selects seats, they're held for 10 minutes. The hold uses `SELECT FOR UPDATE` to atomically check and update seat status:

```sql
BEGIN;

-- Lock all requested seats in a consistent order (prevents deadlock)
SELECT id, status FROM seats
WHERE id = ANY($1)
ORDER BY id
FOR UPDATE;

-- Verify all are AVAILABLE
-- If any are HELD or SOLD → return 409

UPDATE seats SET status = 'HELD' WHERE id = ANY($1);

INSERT INTO seat_holds (user_id, event_id, total_price, expires_at)
VALUES ($2, $3, $4, NOW() + INTERVAL '10 minutes');

COMMIT;
```

Ordering the lock acquisition by `seat_id` prevents deadlocks when two users select overlapping seat sets.

### 5.2 Redis for High-Concurrency On-Sales

During a popular on-sale, thousands of users hit the seat map simultaneously. PostgreSQL `SELECT FOR UPDATE` serializes correctly but creates a bottleneck. The hybrid approach:
1. Redis `SET seat:{seatId}:held {userId} NX EX 600` — fast atomic hold
2. PostgreSQL `SELECT FOR UPDATE` — durable confirmation
3. Background job syncs Redis state to PostgreSQL

### 5.3 Barcode Generation

Each ticket gets a unique barcode generated at purchase time:
```
TKT-{eventId}-{seatId}-{randomBase62(8)}
```
Stored with a unique constraint. The QR code encodes the barcode URL. At the door, scanners call `POST /api/v1/tickets/{barcode}/scan` which atomically sets `scanned_at` and returns `VALID` or `ALREADY_USED`.

### 5.4 Seat Hierarchy Design

The four-level hierarchy (venue → section → row → seat) maps directly to physical reality. Each level is a separate table rather than a flat denormalized structure, enabling:
- Flexible pricing per section
- Row-level queries for seat map rendering
- Easy addition of new sections without schema changes

### 5.5 Hold Expiry and Waitlist

A background job runs every 30 seconds:
```sql
UPDATE seats SET status = 'AVAILABLE'
WHERE id IN (
    SELECT shi.seat_id FROM seat_hold_items shi
    JOIN seat_holds sh ON shi.hold_id = sh.id
    WHERE sh.status = 'ACTIVE' AND sh.expires_at < NOW()
);
UPDATE seat_holds SET status = 'EXPIRED' WHERE status = 'ACTIVE' AND expires_at < NOW();
```
When seats become available, the waitlist is notified in FIFO order.

## 6. Failure Scenarios

### Two Users Select the Same Seat Simultaneously
- **Impact**: Both users proceed to checkout believing they have the seat
- **Recovery**: `SELECT FOR UPDATE` with ordered lock acquisition ensures only one succeeds; the other gets a 409
- **Prevention**: Always acquire seat locks in consistent `ORDER BY id` to prevent deadlocks

### Hold Expiry Job Failure
- **Impact**: Expired holds keep seats in `HELD` status; event appears more sold-out than it is
- **Recovery**: Idempotent expiry job; re-run releases all expired holds
- **Prevention**: Monitor hold age; alert if holds older than 20 minutes exist; multiple job instances with distributed lock

### Payment Failure After Hold
- **Impact**: Seats held but not purchased; hold expires and seats return to available
- **Recovery**: Hold TTL handles this automatically; buyer can re-select if seats still available
- **Prevention**: Pre-authorize payment before hold; reduce hold TTL to 5 minutes for high-demand events

### Ticket Transfer to Wrong User
- **Impact**: Ticket transferred to unintended recipient
- **Recovery**: Transfer has a 24-hour acceptance window; sender can cancel before acceptance
- **Prevention**: Require recipient to accept transfer; send confirmation email to both parties; log all transfer events
