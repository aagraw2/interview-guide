# Hotel Booking System API and DB Design

## System Overview
A hotel booking platform where users search for hotels by location and dates, check room availability, and make reservations with payment. Strong consistency on availability is critical — double-booking a room is a serious business failure that must be prevented at the database level.

## 1. Requirements

### Functional Requirements
- Search hotels by location, check-in/check-out dates, and guest count
- View hotel details, room types, and pricing
- Book a room for given dates with payment
- View and cancel existing bookings
- Hotel managers can update room inventory and pricing
- Support for multiple room types per hotel (single, double, suite)

### Non-Functional Requirements
- Availability: 99.99% — booking must always work
- Consistency: Strong — no double-booking under any concurrency scenario
- Latency: <300ms for search; <1s for booking confirmation
- Scalability: Read-heavy (1000:1 browse-to-book ratio)
- Durability: Confirmed bookings must never be lost
- Security: PCI-DSS compliance for payment data

## 2. Scale Estimation

```
DAU                   = 5M users browsing
Bookings/day          = 500K
Search requests/sec   = 5M × 10 searches / 86400 ≈ 580/sec
Peak search/sec       ≈ 5,800/sec (holiday season 10×)
Bookings/sec (avg)    = 500K / 86400 ≈ 6/sec
Bookings/sec (peak)   ≈ 60/sec

Storage:
  Hotels              = 500K × 5KB = 2.5GB
  Rooms               = 50M × 1KB = 50GB
  Availability rows   = 50M rooms × 365 days × 50B ≈ 900GB/year
  Bookings/year       = 500K × 365 × 2KB ≈ 365GB/year
```

## 3. API Design

### Key Endpoints

#### Search Hotels
```
GET /api/v1/hotels/search?location=Paris&check_in=2025-06-01&check_out=2025-06-05&guests=2&room_type=double

Response 200:
{
  "hotels": [
    {
      "id": "hotel_abc",
      "name": "Grand Paris Hotel",
      "address": "12 Rue de Rivoli, Paris",
      "avg_rating": 4.7,
      "price_per_night": 189.00,
      "currency": "EUR",
      "available_rooms": 3,
      "thumbnail_url": "https://cdn.example.com/hotel_abc.jpg"
    }
  ],
  "total": 42,
  "next_cursor": "eyJpZCI6MTAwfQ"
}
```

#### Get Room Availability
```
GET /api/v1/hotels/{hotelId}/rooms?check_in=2025-06-01&check_out=2025-06-05

Response 200:
{
  "rooms": [
    {
      "id": "room_xyz",
      "type": "DOUBLE",
      "name": "Deluxe Double",
      "price_per_night": 189.00,
      "max_guests": 2,
      "available": true,
      "amenities": ["wifi", "minibar", "city_view"]
    }
  ]
}
```

#### Create Booking
```
POST /api/v1/bookings
Authorization: Bearer <token>

Request:
{
  "room_id": "room_xyz",
  "check_in": "2025-06-01",
  "check_out": "2025-06-05",
  "guests": 2,
  "payment_method_id": "pm_abc123",
  "idempotency_key": "client-uuid-1234"
}

Response 201:
{
  "booking_id": "bkg_789",
  "status": "CONFIRMED",
  "hotel_name": "Grand Paris Hotel",
  "room_type": "DOUBLE",
  "check_in": "2025-06-01",
  "check_out": "2025-06-05",
  "total_amount": 756.00,
  "currency": "EUR",
  "confirmation_code": "GPH-2025-789"
}

Errors:
- 409: Room no longer available for selected dates
- 402: Payment failed
- 422: Invalid date range
```

#### Cancel Booking
```
DELETE /api/v1/bookings/{bookingId}
Authorization: Bearer <token>

Response 200:
{
  "booking_id": "bkg_789",
  "status": "CANCELLED",
  "refund_amount": 756.00,
  "refund_eta_days": 5
}
```

#### List User Bookings
```
GET /api/v1/bookings?status=upcoming&page=1&page_size=10
Authorization: Bearer <token>

Response 200:
{
  "bookings": [...],
  "total": 12,
  "page": 1
}
```

### Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/hotels/search | Search hotels by location and dates |
| GET | /api/v1/hotels/{hotelId} | Get hotel details |
| GET | /api/v1/hotels/{hotelId}/rooms | Get room availability for dates |
| POST | /api/v1/bookings | Create a booking |
| GET | /api/v1/bookings | List user's bookings |
| GET | /api/v1/bookings/{bookingId} | Get booking details |
| DELETE | /api/v1/bookings/{bookingId} | Cancel a booking |
| PUT | /api/v1/hotels/{hotelId}/rooms/{roomId} | Update room details (manager) |

## 4. Database Schema

```sql
CREATE TABLE hotels (
    id              BIGSERIAL PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    address         TEXT NOT NULL,
    city            VARCHAR(100) NOT NULL,
    country         CHAR(2) NOT NULL,
    latitude        DECIMAL(9, 6),
    longitude       DECIMAL(9, 6),
    avg_rating      DECIMAL(3, 2) DEFAULT 0,
    review_count    INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_hotels_city ON hotels (city);
CREATE INDEX idx_hotels_location ON hotels (latitude, longitude);

CREATE TABLE room_types (
    id              BIGSERIAL PRIMARY KEY,
    hotel_id        BIGINT NOT NULL REFERENCES hotels(id),
    name            VARCHAR(100) NOT NULL,       -- "Deluxe Double", "Junior Suite"
    type            VARCHAR(50) NOT NULL,         -- SINGLE, DOUBLE, SUITE, FAMILY
    max_guests      INT NOT NULL,
    price_per_night DECIMAL(10, 2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    total_rooms     INT NOT NULL,                -- total physical rooms of this type
    amenities       JSONB,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE INDEX idx_room_types_hotel_id ON room_types (hotel_id);

-- One row per physical room
CREATE TABLE rooms (
    id              BIGSERIAL PRIMARY KEY,
    hotel_id        BIGINT NOT NULL REFERENCES hotels(id),
    room_type_id    BIGINT NOT NULL REFERENCES room_types(id),
    room_number     VARCHAR(20) NOT NULL,
    floor           INT,
    UNIQUE (hotel_id, room_number)
);

CREATE INDEX idx_rooms_hotel_id ON rooms (hotel_id);
CREATE INDEX idx_rooms_room_type_id ON rooms (room_type_id);

-- Date-range based availability (not one row per day)
CREATE TABLE room_availability (
    id              BIGSERIAL PRIMARY KEY,
    room_id         BIGINT NOT NULL REFERENCES rooms(id),
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,               -- exclusive end (checkout day)
    status          VARCHAR(20) NOT NULL DEFAULT 'AVAILABLE',
                    -- AVAILABLE, BOOKED, MAINTENANCE, BLOCKED
    booking_id      BIGINT,                      -- set when BOOKED
    CHECK (end_date > start_date)
);

CREATE INDEX idx_room_availability_room_id ON room_availability (room_id);
CREATE INDEX idx_room_availability_dates ON room_availability (room_id, start_date, end_date);

CREATE TABLE bookings (
    id                  BIGSERIAL PRIMARY KEY,
    user_id             BIGINT NOT NULL REFERENCES users(id),
    hotel_id            BIGINT NOT NULL REFERENCES hotels(id),
    room_id             BIGINT NOT NULL REFERENCES rooms(id),
    room_type_id        BIGINT NOT NULL REFERENCES room_types(id),
    check_in            DATE NOT NULL,
    check_out           DATE NOT NULL,
    guests              INT NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'PENDING',
                        -- PENDING, CONFIRMED, CANCELLED, COMPLETED, NO_SHOW
    price_per_night     DECIMAL(10, 2) NOT NULL,  -- snapshot at booking time
    total_amount        DECIMAL(10, 2) NOT NULL,
    currency            CHAR(3) NOT NULL,
    payment_id          VARCHAR(100),
    idempotency_key     VARCHAR(100) UNIQUE,
    confirmation_code   VARCHAR(50) UNIQUE,
    cancelled_at        TIMESTAMPTZ,
    cancellation_reason TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bookings_user_id ON bookings (user_id);
CREATE INDEX idx_bookings_hotel_id ON bookings (hotel_id);
CREATE INDEX idx_bookings_room_id ON bookings (room_id);
CREATE INDEX idx_bookings_check_in ON bookings (check_in);
CREATE INDEX idx_bookings_status ON bookings (status);

CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    email           VARCHAR(255) NOT NULL UNIQUE,
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(30),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Schema Summary

| Table | Purpose |
|-------|---------|
| hotels | Hotel properties with location and rating |
| room_types | Room categories per hotel with pricing |
| rooms | Individual physical rooms |
| room_availability | Date-range availability status per room |
| bookings | Confirmed reservations with payment info |
| users | Guest accounts |

## 5. Key Design Decisions

### 5.1 Double-Booking Prevention: Two-Gate Approach

The core challenge is preventing two users from booking the same room for overlapping dates simultaneously.

**Gate 1 — Redis distributed lock (fast path):**
```
SET book:lock:{roomId}:{checkIn}:{checkOut} {userId} NX EX 300
```
Atomic, sub-millisecond. Blocks a second request from even reaching the database while the first is in-flight.

**Gate 2 — PostgreSQL SELECT FOR UPDATE (correctness guarantee):**
```sql
SELECT id FROM room_availability
WHERE room_id = $1
  AND status = 'AVAILABLE'
  AND start_date <= $2 AND end_date >= $3
FOR UPDATE;
```
Even if Redis fails or the lock expires, the row-level lock ensures only one transaction can modify the availability row. The database is the ultimate source of truth.

**Date overlap check:** `newCheckIn < existingCheckOut AND newCheckOut > existingCheckIn`

### 5.2 Date-Range Schema vs. One-Row-Per-Day

Storing availability as date ranges rather than one row per calendar day:
- 50M rooms × 365 days = 18.25B rows as daily records
- As date ranges, most rooms have 2–10 records (available, booked, available...)
- Storage reduction: ~1000×
- Overlap queries use simple date comparison rather than range scans

### 5.3 Price Snapshot in Bookings

`price_per_night` is stored in the `bookings` table at booking time, not referenced from `room_types`. Hotel managers can change prices; historical bookings must reflect what the guest actually paid. This is intentional denormalization.

### 5.4 Idempotency Key

The `idempotency_key` (unique constraint) prevents duplicate bookings if the client retries after a network timeout. The server checks for an existing booking with the same key before processing — if found, returns the existing result without charging again.

### 5.5 Saga Pattern for Payment

Booking flow uses a saga:
1. Acquire Redis lock
2. `SELECT FOR UPDATE` on availability
3. Insert booking with `status = PENDING`
4. Charge payment gateway
5. On success: update booking to `CONFIRMED`, set availability to `BOOKED`
6. On failure: update booking to `CANCELLED`, release availability, release Redis lock

Compensating actions ensure no room is permanently blocked by a failed payment.

## 6. Failure Scenarios

### Double-Booking Race Condition
- **Impact**: Two guests assigned the same room — severe customer experience failure
- **Recovery**: Redis `SET NX` prevents concurrent requests; PostgreSQL `SELECT FOR UPDATE` is the final guarantee even if Redis is unavailable
- **Prevention**: Never skip the DB-level lock; Redis is an optimization, not the sole guard

### Payment Gateway Timeout
- **Impact**: Booking stuck in `PENDING`, room held but not confirmed
- **Recovery**: Retry with same `idempotency_key`; after 3 retries with exponential backoff, release availability and cancel booking; async webhook callback as fallback
- **Prevention**: Set payment timeout to 10s; background job cleans up `PENDING` bookings older than 15 minutes

### Redis Lock Lost Mid-Booking
- **Impact**: A second request could reach the DB while the first is processing
- **Recovery**: PostgreSQL `SELECT FOR UPDATE` prevents actual double-booking at the DB level regardless
- **Prevention**: Redis Cluster with AOF persistence; PostgreSQL is the correctness guarantee

### Database Primary Failure
- **Impact**: New bookings fail; availability checks fail; search continues via replica
- **Recovery**: Promote read replica in <30s; Redis cache continues serving hotel browse during failover
- **Prevention**: Streaming replication with automatic failover (Patroni/pg_auto_failover); confirmed bookings are durable once written
