# Food Delivery Platform API and DB Design

## System Overview
A food delivery platform (think DoorDash or Uber Eats) connecting customers, restaurants, and delivery drivers. An order flows through multiple parties — customer places it, restaurant prepares it, driver picks it up and delivers it — with real-time status updates at each stage.

## 1. Requirements

### Functional Requirements
- Customers browse restaurants and menus, place orders, and track delivery in real time
- Restaurants receive orders, confirm, and mark items as ready for pickup
- Drivers receive pickup assignments, navigate to restaurant, then deliver to customer
- Order lifecycle: placed → confirmed → preparing → ready → picked up → delivered
- Delivery fee calculation based on distance and demand
- Ratings for restaurants and drivers after delivery
- Order history for all three parties

### Non-Functional Requirements
- Availability: 99.99% — order placement must always work
- Latency: <500ms for order placement; <2s for driver assignment
- Real-time: Driver location updates every 5 seconds; order status push notifications
- Scalability: 5M orders/day, 500K concurrent drivers
- Consistency: Strong for order state transitions; eventual for location tracking

## 2. Scale Estimation

```
DAU                     = 10M customers, 500K drivers, 200K restaurants
Orders/day              = 5M
Orders/sec (avg)        = 5M / 86400 ≈ 58/sec
Peak orders/sec         ≈ 580/sec (dinner rush)

Driver location updates = 500K × 1 update/5sec = 100K updates/sec
Order status events/day = 5M × 6 transitions = 30M events/day

Order storage           = 5M × 3KB = 15GB/day
Annual order storage    ≈ 5.5TB/year
```

## 3. API Design

### Key Endpoints

#### Browse Restaurants
```
GET /api/v1/restaurants?lat=40.71&lng=-74.00&radius_km=5&cuisine=italian

Response 200:
{
  "restaurants": [
    {
      "id": "rest_abc",
      "name": "Mario's Pizza",
      "cuisine": "italian",
      "avg_rating": 4.6,
      "delivery_fee": 2.99,
      "estimated_delivery_minutes": 35,
      "is_open": true,
      "thumbnail_url": "..."
    }
  ]
}
```

#### Place Order
```
POST /api/v1/orders
Authorization: Bearer <token>

Request:
{
  "restaurant_id": "rest_abc",
  "items": [
    { "menu_item_id": "item_001", "quantity": 2, "special_instructions": "extra cheese" },
    { "menu_item_id": "item_005", "quantity": 1 }
  ],
  "delivery_address_id": "addr_xyz",
  "payment_method_id": "pm_abc",
  "tip_amount": 3.00
}

Response 201:
{
  "order_id": "ord_xyz",
  "status": "PLACED",
  "subtotal": 28.50,
  "delivery_fee": 2.99,
  "tip": 3.00,
  "total": 34.49,
  "estimated_delivery_at": "2025-06-01T19:35:00Z"
}
```

#### Get Order Status
```
GET /api/v1/orders/{orderId}

Response 200:
{
  "order_id": "ord_xyz",
  "status": "DRIVER_EN_ROUTE",
  "driver": {
    "name": "Carlos M.",
    "rating": 4.9,
    "location": { "lat": 40.715, "lng": -74.002 },
    "eta_minutes": 8
  },
  "restaurant": { "name": "Mario's Pizza" },
  "items": [...],
  "total": 34.49
}
```

#### Restaurant: Update Order Status
```
PATCH /api/v1/orders/{orderId}/status
Authorization: Bearer <token>  (restaurant role)

Request: { "status": "PREPARING" }
Response 200: { "order_id": "ord_xyz", "status": "PREPARING" }
```

#### Driver: Accept Delivery Assignment
```
POST /api/v1/deliveries/{deliveryId}/accept
Authorization: Bearer <token>  (driver role)

Response 200:
{
  "delivery_id": "del_abc",
  "order_id": "ord_xyz",
  "pickup_address": "123 Pizza St",
  "dropoff_address": "456 Customer Ave",
  "estimated_earnings": 8.50
}
```

### Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/restaurants | Browse nearby restaurants |
| GET | /api/v1/restaurants/{id}/menu | Get restaurant menu |
| POST | /api/v1/orders | Place an order |
| GET | /api/v1/orders/{orderId} | Get order status + driver location |
| PATCH | /api/v1/orders/{orderId}/status | Update order status (restaurant/driver) |
| POST | /api/v1/deliveries/{deliveryId}/accept | Driver accepts delivery |
| POST | /api/v1/drivers/location | Driver sends GPS update |
| POST | /api/v1/orders/{orderId}/rating | Submit post-delivery rating |

## 4. Database Schema

```sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    phone       VARCHAR(30) NOT NULL UNIQUE,
    full_name   VARCHAR(255) NOT NULL,
    role        VARCHAR(20) NOT NULL,   -- CUSTOMER, DRIVER, RESTAURANT_OWNER
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE restaurants (
    id              BIGSERIAL PRIMARY KEY,
    owner_id        BIGINT NOT NULL REFERENCES users(id),
    name            VARCHAR(255) NOT NULL,
    cuisine         VARCHAR(100),
    address         TEXT NOT NULL,
    lat             DECIMAL(9, 6) NOT NULL,
    lng             DECIMAL(9, 6) NOT NULL,
    avg_rating      DECIMAL(3, 2) DEFAULT 0,
    is_open         BOOLEAN NOT NULL DEFAULT FALSE,
    prep_time_minutes INT NOT NULL DEFAULT 20,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_restaurants_location ON restaurants (lat, lng);

CREATE TABLE menu_items (
    id              BIGSERIAL PRIMARY KEY,
    restaurant_id   BIGINT NOT NULL REFERENCES restaurants(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    price           DECIMAL(8, 2) NOT NULL,
    category        VARCHAR(100),
    is_available    BOOLEAN NOT NULL DEFAULT TRUE,
    image_url       VARCHAR(500)
);

CREATE INDEX idx_menu_items_restaurant_id ON menu_items (restaurant_id);

CREATE TABLE delivery_addresses (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL REFERENCES users(id),
    label       VARCHAR(50),            -- "Home", "Work"
    address     TEXT NOT NULL,
    lat         DECIMAL(9, 6) NOT NULL,
    lng         DECIMAL(9, 6) NOT NULL,
    is_default  BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE TABLE orders (
    id                  BIGSERIAL PRIMARY KEY,
    customer_id         BIGINT NOT NULL REFERENCES users(id),
    restaurant_id       BIGINT NOT NULL REFERENCES restaurants(id),
    delivery_address_id BIGINT NOT NULL REFERENCES delivery_addresses(id),
    status              VARCHAR(30) NOT NULL DEFAULT 'PLACED',
                        -- PLACED, CONFIRMED, PREPARING, READY_FOR_PICKUP,
                        -- DRIVER_ASSIGNED, PICKED_UP, DELIVERED, CANCELLED
    subtotal            DECIMAL(8, 2) NOT NULL,
    delivery_fee        DECIMAL(6, 2) NOT NULL,
    tip_amount          DECIMAL(6, 2) NOT NULL DEFAULT 0,
    total_amount        DECIMAL(8, 2) NOT NULL,
    payment_id          VARCHAR(100),
    special_instructions TEXT,
    estimated_delivery_at TIMESTAMPTZ,
    placed_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    confirmed_at        TIMESTAMPTZ,
    picked_up_at        TIMESTAMPTZ,
    delivered_at        TIMESTAMPTZ,
    cancelled_at        TIMESTAMPTZ
);

CREATE INDEX idx_orders_customer_id ON orders (customer_id);
CREATE INDEX idx_orders_restaurant_id ON orders (restaurant_id);
CREATE INDEX idx_orders_status ON orders (status);
CREATE INDEX idx_orders_placed_at ON orders (placed_at DESC);

CREATE TABLE order_items (
    id              BIGSERIAL PRIMARY KEY,
    order_id        BIGINT NOT NULL REFERENCES orders(id),
    menu_item_id    BIGINT NOT NULL REFERENCES menu_items(id),
    quantity        INT NOT NULL CHECK (quantity > 0),
    unit_price      DECIMAL(8, 2) NOT NULL,   -- snapshot at order time
    special_instructions TEXT
);

CREATE INDEX idx_order_items_order_id ON order_items (order_id);

CREATE TABLE deliveries (
    id              BIGSERIAL PRIMARY KEY,
    order_id        BIGINT NOT NULL REFERENCES orders(id) UNIQUE,
    driver_id       BIGINT REFERENCES users(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'PENDING_ASSIGNMENT',
                    -- PENDING_ASSIGNMENT, ASSIGNED, PICKED_UP, DELIVERED
    pickup_lat      DECIMAL(9, 6),
    pickup_lng      DECIMAL(9, 6),
    dropoff_lat     DECIMAL(9, 6),
    dropoff_lng     DECIMAL(9, 6),
    distance_km     DECIMAL(6, 3),
    driver_earnings DECIMAL(6, 2),
    assigned_at     TIMESTAMPTZ,
    picked_up_at    TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ
);

CREATE INDEX idx_deliveries_driver_id ON deliveries (driver_id);
CREATE INDEX idx_deliveries_status ON deliveries (status);

-- Driver current location (primarily in Redis, synced here periodically)
CREATE TABLE driver_locations (
    driver_id   BIGINT PRIMARY KEY REFERENCES users(id),
    lat         DECIMAL(9, 6),
    lng         DECIMAL(9, 6),
    status      VARCHAR(20) NOT NULL DEFAULT 'OFFLINE',
                -- OFFLINE, AVAILABLE, ON_DELIVERY
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE ratings (
    id              BIGSERIAL PRIMARY KEY,
    order_id        BIGINT NOT NULL REFERENCES orders(id),
    rater_id        BIGINT NOT NULL REFERENCES users(id),
    ratee_id        BIGINT NOT NULL REFERENCES users(id),
    ratee_type      VARCHAR(20) NOT NULL,   -- DRIVER, RESTAURANT
    rating          SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (order_id, rater_id, ratee_type)
);
```

### Schema Summary

| Table | Purpose |
|-------|---------|
| restaurants | Restaurant profiles with location |
| menu_items | Restaurant menus |
| orders | Order lifecycle with timestamps |
| order_items | Line items with price snapshot |
| deliveries | Delivery assignment and tracking |
| driver_locations | Current driver GPS state |
| ratings | Post-delivery ratings |

## 5. Key Design Decisions

### 5.1 Multi-Party Order State Machine

The order status is owned by different parties at different stages:

```
PLACED          → customer places order
CONFIRMED       → restaurant accepts (or auto-confirm after 2 min)
PREPARING       → restaurant starts cooking
READY_FOR_PICKUP → restaurant marks food ready
DRIVER_ASSIGNED → system assigns nearest driver
PICKED_UP       → driver confirms pickup
DELIVERED       → driver confirms delivery
CANCELLED       → any party can cancel before PICKED_UP
```

State transitions are validated in application code. Invalid transitions (e.g., DELIVERED → PREPARING) are rejected with 422.

### 5.2 Delivery Fee Calculation

```
base_fee = 1.99
distance_fee = distance_km × 0.50
demand_multiplier = surge_factor(restaurant_zone)
delivery_fee = (base_fee + distance_fee) × demand_multiplier
```

Calculated at order placement time and stored on the order. The customer sees the fee before confirming.

### 5.3 Driver Assignment Race Condition

When a delivery becomes available, multiple drivers may accept simultaneously. The same pattern as ride-sharing applies:

```sql
UPDATE deliveries
SET driver_id = $1, status = 'ASSIGNED', assigned_at = NOW()
WHERE id = $2 AND status = 'PENDING_ASSIGNMENT' AND driver_id IS NULL;
-- 0 rows affected = another driver already accepted
```

### 5.4 Real-Time Driver Location

100K location updates/second cannot go directly to PostgreSQL. Architecture:
1. Driver app sends GPS every 5 seconds to location service
2. Location service writes to Redis geospatial set
3. Customer's app receives WebSocket push with driver location
4. PostgreSQL `driver_locations` updated every 30 seconds for persistence

### 5.5 Price Snapshot in Order Items

`unit_price` in `order_items` captures the menu price at order time. Restaurants can change prices; historical orders must reflect what the customer paid. This is intentional denormalization — same pattern as e-commerce.

## 6. Failure Scenarios

### Restaurant Doesn't Confirm Order
- **Impact**: Order stuck in `PLACED` state; customer waiting
- **Recovery**: Auto-confirm after 2 minutes if restaurant doesn't respond; notify restaurant; cancel and refund after 5 minutes of no response
- **Prevention**: Restaurant app sends heartbeat; alert if restaurant goes offline during active orders

### No Available Drivers
- **Impact**: Order ready at restaurant but no driver assigned; food gets cold
- **Recovery**: Expand search radius every 2 minutes; offer higher driver incentive; notify customer of delay
- **Prevention**: Predictive driver positioning based on historical order patterns

### Driver Marks Delivered Without Delivering
- **Impact**: Customer doesn't receive food; driver gets paid fraudulently
- **Recovery**: Customer reports non-delivery; refund issued; driver account flagged
- **Prevention**: Require GPS confirmation within delivery address radius; photo proof of delivery

### Payment Failure After Order Placed
- **Impact**: Order placed but payment not captured; restaurant may have started preparing
- **Recovery**: Retry payment 3 times; cancel order and notify restaurant if all retries fail
- **Prevention**: Pre-authorize payment at order placement; capture on restaurant confirmation
