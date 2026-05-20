# Ride-Sharing App API and DB Design

## System Overview
A ride-sharing platform (think Uber or Lyft) where riders request trips, nearby drivers are matched and dispatched, and fares are calculated dynamically based on demand. The system manages a real-time geospatial state machine across millions of concurrent drivers and riders.

## 1. Requirements

### Functional Requirements
- Riders can request a ride from their current location to a destination
- System matches rider to the nearest available driver
- Drivers can accept or decline ride requests
- Real-time GPS tracking of driver location during a trip
- Fare calculation with surge pricing based on supply/demand
- Trip history and receipts for both riders and drivers
- Ratings after trip completion (rider rates driver, driver rates rider)

### Non-Functional Requirements
- Availability: 99.99% — matching must always work
- Latency: <2s for driver match; <500ms for location updates
- Scalability: 5M concurrent drivers, 10M ride requests/day
- Consistency: Eventual for location updates; strong for trip state transitions
- Geospatial: Sub-second nearest-driver queries within a city radius

## 2. Scale Estimation

```
DAU                     = 10M riders, 2M drivers
Ride requests/day       = 10M
Ride requests/sec       = 10M / 86400 ≈ 116/sec
Peak requests/sec       ≈ 1,160/sec

Driver location updates = 2M drivers × 1 update/4sec = 500K updates/sec
Location update storage = 500K × 100B = 50MB/sec (stream, not persisted per update)

Trip storage/day        = 10M × 2KB = 20GB/day
Annual trip storage     ≈ 7TB/year
```

## 3. API Design

### Key Endpoints

#### Request a Ride
```
POST /api/v1/rides
Authorization: Bearer <token>

Request:
{
  "pickup": { "lat": 40.7128, "lng": -74.0060, "address": "123 Main St, NYC" },
  "dropoff": { "lat": 40.7580, "lng": -73.9855, "address": "Times Square, NYC" },
  "ride_type": "STANDARD"   // STANDARD, XL, PREMIUM
}

Response 201:
{
  "ride_id": "ride_abc123",
  "status": "SEARCHING",
  "estimated_fare": { "min": 12.50, "max": 16.00, "currency": "USD" },
  "surge_multiplier": 1.2,
  "estimated_pickup_minutes": 4
}
```

#### Get Ride Status (polling or WebSocket)
```
GET /api/v1/rides/{rideId}

Response 200:
{
  "ride_id": "ride_abc123",
  "status": "DRIVER_EN_ROUTE",
  "driver": {
    "id": "driver_xyz",
    "name": "Carlos M.",
    "rating": 4.92,
    "vehicle": "Toyota Camry · ABC-1234",
    "location": { "lat": 40.7100, "lng": -74.0050 },
    "eta_minutes": 2
  }
}
```

#### Driver: Update Location
```
POST /api/v1/drivers/location
Authorization: Bearer <token>

Request:
{
  "lat": 40.7128,
  "lng": -74.0060,
  "heading": 270,
  "speed_kmh": 35
}

Response 204
```

#### Driver: Accept/Decline Ride
```
POST /api/v1/rides/{rideId}/accept
POST /api/v1/rides/{rideId}/decline
Authorization: Bearer <token>

Response 200: { "ride_id": "...", "status": "ACCEPTED" }
```

#### Complete Trip
```
POST /api/v1/rides/{rideId}/complete
Authorization: Bearer <token>  (driver only)

Response 200:
{
  "ride_id": "ride_abc123",
  "status": "COMPLETED",
  "fare": { "base": 8.00, "distance": 4.50, "surge": 2.40, "total": 14.90 },
  "distance_km": 6.2,
  "duration_minutes": 18
}
```

#### Submit Rating
```
POST /api/v1/rides/{rideId}/rating
Authorization: Bearer <token>

Request: { "rating": 5, "comment": "Great driver!" }
Response 201: { "ride_id": "...", "rating": 5 }
```

### Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| POST | /api/v1/rides | Request a ride |
| GET | /api/v1/rides/{rideId} | Get ride status and driver location |
| DELETE | /api/v1/rides/{rideId} | Cancel ride request |
| POST | /api/v1/rides/{rideId}/accept | Driver accepts ride |
| POST | /api/v1/rides/{rideId}/decline | Driver declines ride |
| POST | /api/v1/rides/{rideId}/complete | Driver marks trip complete |
| POST | /api/v1/rides/{rideId}/rating | Submit post-trip rating |
| POST | /api/v1/drivers/location | Driver sends GPS update |
| GET | /api/v1/drivers/nearby | Find nearby available drivers |

## 4. Database Schema

```sql
CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    email           VARCHAR(255) NOT NULL UNIQUE,
    phone           VARCHAR(30) NOT NULL UNIQUE,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(20) NOT NULL,   -- RIDER, DRIVER, BOTH
    avg_rating      DECIMAL(3, 2) DEFAULT 5.0,
    rating_count    INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE driver_profiles (
    user_id         BIGINT PRIMARY KEY REFERENCES users(id),
    license_number  VARCHAR(50) NOT NULL UNIQUE,
    vehicle_make    VARCHAR(100) NOT NULL,
    vehicle_model   VARCHAR(100) NOT NULL,
    vehicle_year    INT NOT NULL,
    license_plate   VARCHAR(20) NOT NULL UNIQUE,
    vehicle_type    VARCHAR(20) NOT NULL,   -- STANDARD, XL, PREMIUM
    is_verified     BOOLEAN NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE
);

-- Current driver state (updated frequently, kept in Redis primarily)
CREATE TABLE driver_status (
    driver_id       BIGINT PRIMARY KEY REFERENCES users(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'OFFLINE',
                    -- OFFLINE, AVAILABLE, ON_TRIP
    current_lat     DECIMAL(9, 6),
    current_lng     DECIMAL(9, 6),
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- PostGIS spatial index for nearest-driver queries
CREATE INDEX idx_driver_status_location
    ON driver_status USING GIST (
        ST_SetSRID(ST_MakePoint(current_lng, current_lat), 4326)
    );

CREATE TABLE rides (
    id                  BIGSERIAL PRIMARY KEY,
    rider_id            BIGINT NOT NULL REFERENCES users(id),
    driver_id           BIGINT REFERENCES users(id),
    status              VARCHAR(30) NOT NULL DEFAULT 'SEARCHING',
                        -- SEARCHING, DRIVER_ASSIGNED, DRIVER_EN_ROUTE,
                        -- RIDER_PICKED_UP, COMPLETED, CANCELLED, NO_DRIVER_FOUND
    ride_type           VARCHAR(20) NOT NULL DEFAULT 'STANDARD',
    pickup_lat          DECIMAL(9, 6) NOT NULL,
    pickup_lng          DECIMAL(9, 6) NOT NULL,
    pickup_address      TEXT NOT NULL,
    dropoff_lat         DECIMAL(9, 6) NOT NULL,
    dropoff_lng         DECIMAL(9, 6) NOT NULL,
    dropoff_address     TEXT NOT NULL,
    surge_multiplier    DECIMAL(4, 2) NOT NULL DEFAULT 1.0,
    estimated_fare_min  DECIMAL(8, 2),
    estimated_fare_max  DECIMAL(8, 2),
    final_fare          DECIMAL(8, 2),
    distance_km         DECIMAL(8, 3),
    duration_minutes    INT,
    driver_assigned_at  TIMESTAMPTZ,
    pickup_at           TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    cancelled_at        TIMESTAMPTZ,
    cancel_reason       VARCHAR(100),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rides_rider_id ON rides (rider_id);
CREATE INDEX idx_rides_driver_id ON rides (driver_id);
CREATE INDEX idx_rides_status ON rides (status);
CREATE INDEX idx_rides_created_at ON rides (created_at DESC);

-- GPS breadcrumb trail for completed trips
CREATE TABLE trip_locations (
    id          BIGSERIAL PRIMARY KEY,
    ride_id     BIGINT NOT NULL REFERENCES rides(id),
    lat         DECIMAL(9, 6) NOT NULL,
    lng         DECIMAL(9, 6) NOT NULL,
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_trip_locations_ride_id ON trip_locations (ride_id, recorded_at);

CREATE TABLE ratings (
    id          BIGSERIAL PRIMARY KEY,
    ride_id     BIGINT NOT NULL REFERENCES rides(id),
    rater_id    BIGINT NOT NULL REFERENCES users(id),
    ratee_id    BIGINT NOT NULL REFERENCES users(id),
    rating      SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment     TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (ride_id, rater_id)
);

-- Surge pricing zones
CREATE TABLE surge_zones (
    id              BIGSERIAL PRIMARY KEY,
    city            VARCHAR(100) NOT NULL,
    zone_name       VARCHAR(100) NOT NULL,
    multiplier      DECIMAL(4, 2) NOT NULL DEFAULT 1.0,
    active_until    TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Schema Summary

| Table | Purpose |
|-------|---------|
| users | Riders and drivers (unified) |
| driver_profiles | Driver vehicle and license info |
| driver_status | Current driver location and availability |
| rides | Trip lifecycle from request to completion |
| trip_locations | GPS breadcrumb trail per trip |
| ratings | Post-trip ratings (bidirectional) |
| surge_zones | Dynamic surge pricing by zone |

## 5. Key Design Decisions

### 5.1 Geospatial Driver Matching with PostGIS

Finding the nearest available driver is the core query. PostGIS `ST_DWithin` with a spatial index makes this sub-second:

```sql
SELECT driver_id,
       ST_Distance(
           ST_SetSRID(ST_MakePoint(current_lng, current_lat), 4326)::geography,
           ST_SetSRID(ST_MakePoint($1, $2), 4326)::geography
       ) AS distance_meters
FROM driver_status
WHERE status = 'AVAILABLE'
  AND ST_DWithin(
      ST_SetSRID(ST_MakePoint(current_lng, current_lat), 4326)::geography,
      ST_SetSRID(ST_MakePoint($1, $2), 4326)::geography,
      5000  -- 5km radius
  )
ORDER BY distance_meters
LIMIT 10;
```

In practice, driver locations are stored in Redis geospatial sets (`GEOADD`, `GEORADIUS`) for sub-millisecond lookups, with PostgreSQL as the durable store.

### 5.2 Trip State Machine

Valid transitions are enforced in application code:
```
SEARCHING → DRIVER_ASSIGNED → DRIVER_EN_ROUTE → RIDER_PICKED_UP → COMPLETED
SEARCHING → NO_DRIVER_FOUND
SEARCHING | DRIVER_ASSIGNED | DRIVER_EN_ROUTE → CANCELLED
```
Invalid transitions (e.g., COMPLETED → CANCELLED) are rejected. All transitions are logged with timestamps.

### 5.3 Driver Location Updates at Scale

500K location updates/second cannot all be written to PostgreSQL. The architecture:
1. Driver app sends GPS every 4 seconds to a location service
2. Location service writes to Redis geospatial set (sub-ms, in-memory)
3. Rider's app polls or receives WebSocket push from Redis
4. Only the final trip path is persisted to `trip_locations` at trip end

### 5.4 Surge Pricing Calculation

Surge multiplier is computed per zone based on the ratio of active ride requests to available drivers:
```
surge = max(1.0, requests_in_zone / (available_drivers_in_zone × target_utilization))
```
Surge zones are recalculated every 60 seconds and cached in Redis. The multiplier is locked in at ride request time and stored on the `rides` row.

### 5.5 Driver Assignment Race Condition

When a ride is dispatched to multiple nearby drivers simultaneously, two drivers could accept the same ride. Prevention:
```sql
UPDATE rides SET driver_id = $1, status = 'DRIVER_ASSIGNED'
WHERE id = $2 AND status = 'SEARCHING' AND driver_id IS NULL;
-- 0 rows affected = another driver already accepted
```
The atomic `WHERE status = 'SEARCHING'` check ensures only one driver wins.

## 6. Failure Scenarios

### No Available Drivers
- **Impact**: Rider waits indefinitely in `SEARCHING` state
- **Recovery**: After 2 minutes, transition to `NO_DRIVER_FOUND`; notify rider; suggest retry
- **Prevention**: Surge pricing increases driver supply; background job monitors stale `SEARCHING` rides

### Driver App Crash Mid-Trip
- **Impact**: Location updates stop; rider loses visibility; trip may not complete
- **Recovery**: Background job detects no location update for >60s on an active trip; contacts driver; auto-completes trip based on last known location after 10 minutes
- **Prevention**: Driver app heartbeat; server-side trip timeout

### Payment Processing Failure
- **Impact**: Trip completed but fare not charged
- **Recovery**: Retry payment with exponential backoff; fall back to charging saved payment method; flag account for manual review after 3 failures
- **Prevention**: Pre-authorize payment method before trip starts

### Redis Location Store Failure
- **Impact**: Driver matching falls back to PostgreSQL; latency increases from <1ms to ~50ms
- **Recovery**: Redis Cluster with automatic failover; fall back to PostGIS queries during outage
- **Prevention**: Redis Sentinel/Cluster; PostgreSQL spatial index as fallback
