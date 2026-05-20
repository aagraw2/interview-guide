# Inventory Management API and DB Design

## System Overview
An inventory management system for a multi-warehouse retail operation where products are tracked across locations, stock movements are audited, and low-stock alerts trigger replenishment. The core challenge is preventing overselling through atomic stock reservation while maintaining a complete audit trail.

## 1. Requirements

### Functional Requirements
- Manage product catalog with SKUs across multiple warehouses
- Reserve stock when an order is placed; confirm or release on payment outcome
- Record all stock movements (receipts, shipments, adjustments, returns)
- Query current stock levels per product per warehouse
- Trigger low-stock alerts when quantity falls below threshold
- Support stock transfers between warehouses
- Generate stock movement reports and audit trails

### Non-Functional Requirements
- Consistency: Strong — stock levels must be accurate; no overselling
- Availability: 99.9%
- Latency: <200ms for stock checks; <500ms for reservations
- Auditability: Every stock change must be traceable to a user and reason
- Scalability: 1M SKUs, 500 warehouses, 10M stock movements/day

## 2. Scale Estimation

```
SKUs                    = 1M products
Warehouses              = 500
Stock movements/day     = 10M
Movements/sec (avg)     = 10M / 86400 ≈ 116/sec
Peak movements/sec      ≈ 1,000/sec (flash sales)

Stock level rows        = 1M SKUs × 500 warehouses = 500M rows (sparse in practice)
Movement record size    = ~300B
Movement storage/year   = 10M × 365 × 300B ≈ 1TB/year
```

## 3. API Design

### Key Endpoints

#### Check Stock Level
```
GET /api/v1/inventory/{sku}/stock?warehouse_id=wh_nyc

Response 200:
{
  "sku": "WIDGET-RED-L",
  "warehouse_id": "wh_nyc",
  "quantity_on_hand": 150,
  "quantity_reserved": 12,
  "quantity_available": 138,
  "reorder_point": 50,
  "reorder_quantity": 200
}
```

#### Reserve Stock (for order placement)
```
POST /api/v1/inventory/reserve
Authorization: Bearer <token>

Request:
{
  "order_id": "ord_abc123",
  "items": [
    { "sku": "WIDGET-RED-L", "warehouse_id": "wh_nyc", "quantity": 3 },
    { "sku": "GADGET-BLUE",  "warehouse_id": "wh_nyc", "quantity": 1 }
  ],
  "expires_at": "2025-06-01T11:00:00Z"
}

Response 201:
{
  "reservation_id": "res_xyz",
  "status": "RESERVED",
  "items": [
    { "sku": "WIDGET-RED-L", "quantity": 3, "status": "RESERVED" },
    { "sku": "GADGET-BLUE",  "quantity": 1, "status": "RESERVED" }
  ]
}

Errors:
- 409: Insufficient stock for one or more items (returns which items failed)
```

#### Confirm or Release Reservation
```
POST /api/v1/inventory/reservations/{reservationId}/confirm
POST /api/v1/inventory/reservations/{reservationId}/release

Response 200: { "reservation_id": "res_xyz", "status": "CONFIRMED" | "RELEASED" }
```

#### Record Stock Movement
```
POST /api/v1/inventory/movements
Authorization: Bearer <token>

Request:
{
  "sku": "WIDGET-RED-L",
  "warehouse_id": "wh_nyc",
  "movement_type": "RECEIPT",
  "quantity": 500,
  "reference_id": "po_purchase_order_123",
  "notes": "Supplier delivery"
}

Response 201:
{
  "movement_id": "mov_abc",
  "sku": "WIDGET-RED-L",
  "quantity_before": 150,
  "quantity_after": 650,
  "movement_type": "RECEIPT",
  "created_at": "2025-06-01T09:00:00Z"
}
```

#### Transfer Stock Between Warehouses
```
POST /api/v1/inventory/transfers
Authorization: Bearer <token>

Request:
{
  "sku": "WIDGET-RED-L",
  "from_warehouse_id": "wh_nyc",
  "to_warehouse_id": "wh_la",
  "quantity": 50
}

Response 201: { "transfer_id": "txf_abc", "status": "COMPLETED" }
```

### Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/inventory/{sku}/stock | Check stock levels |
| GET | /api/v1/inventory/low-stock | List items below reorder point |
| POST | /api/v1/inventory/reserve | Reserve stock for an order |
| POST | /api/v1/inventory/reservations/{id}/confirm | Confirm reservation |
| POST | /api/v1/inventory/reservations/{id}/release | Release reservation |
| POST | /api/v1/inventory/movements | Record stock movement |
| GET | /api/v1/inventory/movements | Query movement history |
| POST | /api/v1/inventory/transfers | Transfer between warehouses |

## 4. Database Schema

```sql
CREATE TABLE warehouses (
    id          BIGSERIAL PRIMARY KEY,
    code        VARCHAR(20) NOT NULL UNIQUE,
    name        VARCHAR(255) NOT NULL,
    address     TEXT,
    is_active   BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE products (
    id              BIGSERIAL PRIMARY KEY,
    sku             VARCHAR(100) NOT NULL UNIQUE,
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    category        VARCHAR(100),
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EACH',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_sku ON products (sku);
CREATE INDEX idx_products_category ON products (category);

-- Current stock levels per product per warehouse
CREATE TABLE stock_levels (
    id                  BIGSERIAL PRIMARY KEY,
    product_id          BIGINT NOT NULL REFERENCES products(id),
    warehouse_id        BIGINT NOT NULL REFERENCES warehouses(id),
    quantity_on_hand    INT NOT NULL DEFAULT 0 CHECK (quantity_on_hand >= 0),
    quantity_reserved   INT NOT NULL DEFAULT 0 CHECK (quantity_reserved >= 0),
    reorder_point       INT NOT NULL DEFAULT 0,
    reorder_quantity    INT NOT NULL DEFAULT 0,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (product_id, warehouse_id)
);

CREATE INDEX idx_stock_levels_product_id ON stock_levels (product_id);
CREATE INDEX idx_stock_levels_warehouse_id ON stock_levels (warehouse_id);
-- Partial index for low-stock monitoring
CREATE INDEX idx_stock_levels_low_stock ON stock_levels (product_id, warehouse_id)
    WHERE quantity_on_hand - quantity_reserved <= reorder_point;

-- Stock reservations (holds for pending orders)
CREATE TABLE stock_reservations (
    id              BIGSERIAL PRIMARY KEY,
    order_id        VARCHAR(100) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'RESERVED',
                    -- RESERVED, CONFIRMED, RELEASED, EXPIRED
    expires_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_stock_reservations_order_id ON stock_reservations (order_id);
CREATE INDEX idx_stock_reservations_expires_at ON stock_reservations (expires_at)
    WHERE status = 'RESERVED';

CREATE TABLE stock_reservation_items (
    id              BIGSERIAL PRIMARY KEY,
    reservation_id  BIGINT NOT NULL REFERENCES stock_reservations(id),
    product_id      BIGINT NOT NULL REFERENCES products(id),
    warehouse_id    BIGINT NOT NULL REFERENCES warehouses(id),
    quantity        INT NOT NULL CHECK (quantity > 0)
);

CREATE INDEX idx_reservation_items_reservation_id ON stock_reservation_items (reservation_id);

-- Immutable audit trail of all stock changes
CREATE TABLE stock_movements (
    id              BIGSERIAL PRIMARY KEY,
    product_id      BIGINT NOT NULL REFERENCES products(id),
    warehouse_id    BIGINT NOT NULL REFERENCES warehouses(id),
    movement_type   VARCHAR(30) NOT NULL,
                    -- RECEIPT, SHIPMENT, ADJUSTMENT, RETURN, TRANSFER_IN, TRANSFER_OUT, RESERVATION, RELEASE
    quantity        INT NOT NULL,           -- positive = in, negative = out
    quantity_before INT NOT NULL,
    quantity_after  INT NOT NULL,
    reference_id    VARCHAR(100),           -- order_id, po_id, transfer_id
    reference_type  VARCHAR(50),            -- ORDER, PURCHASE_ORDER, TRANSFER, MANUAL
    performed_by    BIGINT REFERENCES users(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_stock_movements_product_id ON stock_movements (product_id, created_at DESC);
CREATE INDEX idx_stock_movements_warehouse_id ON stock_movements (warehouse_id, created_at DESC);
CREATE INDEX idx_stock_movements_reference ON stock_movements (reference_id, reference_type);

CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    full_name   VARCHAR(255) NOT NULL,
    role        VARCHAR(30) NOT NULL DEFAULT 'STAFF',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Schema Summary

| Table | Purpose |
|-------|---------|
| warehouses | Physical warehouse locations |
| products | Product catalog with SKUs |
| stock_levels | Current on-hand and reserved quantities |
| stock_reservations | Holds for pending orders with TTL |
| stock_reservation_items | Line items per reservation |
| stock_movements | Immutable audit trail of all changes |

## 5. Key Design Decisions

### 5.1 Stock Reservation Pattern (Prevent Overselling)

The core challenge: two orders placed simultaneously for the last 3 units of a product. Naive approach reads stock, checks availability, then decrements — a classic TOCTOU race.

The reservation pattern uses `SELECT FOR UPDATE` to atomically check and reserve:

```sql
BEGIN;

-- Lock the stock level row
SELECT quantity_on_hand, quantity_reserved
FROM stock_levels
WHERE product_id = $1 AND warehouse_id = $2
FOR UPDATE;

-- Check available = on_hand - reserved >= requested
-- If sufficient:
UPDATE stock_levels
SET quantity_reserved = quantity_reserved + $3,
    updated_at = NOW()
WHERE product_id = $1 AND warehouse_id = $2;

INSERT INTO stock_movements (product_id, warehouse_id, movement_type, quantity, ...)
VALUES ($1, $2, 'RESERVATION', $3, ...);

COMMIT;
```

`quantity_available = quantity_on_hand - quantity_reserved` is computed, never stored.

### 5.2 Two-Phase Stock Commitment

Stock flows through two phases:
1. **Reserve**: Increment `quantity_reserved` when order is placed (stock held, not yet shipped)
2. **Confirm**: Decrement both `quantity_on_hand` and `quantity_reserved` when order ships

This means available stock = `on_hand - reserved` is always accurate. If payment fails, the reservation is released (decrement `quantity_reserved` only).

### 5.3 Immutable Audit Trail

`stock_movements` is append-only — never updated or deleted. Every change to `stock_levels` produces a corresponding movement record with `quantity_before` and `quantity_after`. This enables:
- Full reconstruction of stock history at any point in time
- Discrepancy investigation
- Compliance reporting

### 5.4 Reservation Expiry

Reservations have a TTL (`expires_at`). A background job runs every minute:
```sql
UPDATE stock_levels sl
SET quantity_reserved = quantity_reserved - sri.quantity
FROM stock_reservation_items sri
JOIN stock_reservations sr ON sri.reservation_id = sr.id
WHERE sr.status = 'RESERVED' AND sr.expires_at < NOW()
  AND sl.product_id = sri.product_id AND sl.warehouse_id = sri.warehouse_id;

UPDATE stock_reservations SET status = 'EXPIRED' WHERE status = 'RESERVED' AND expires_at < NOW();
```

### 5.5 Low-Stock Alerts

The partial index on `stock_levels` makes low-stock queries efficient:
```sql
SELECT p.sku, p.name, sl.quantity_on_hand, sl.reorder_point, sl.reorder_quantity
FROM stock_levels sl
JOIN products p ON sl.product_id = p.id
WHERE sl.quantity_on_hand - sl.quantity_reserved <= sl.reorder_point;
```
Alerts are sent via a notification service when this query returns results.

## 6. Failure Scenarios

### Overselling Race Condition
- **Impact**: More units sold than available; fulfillment failures
- **Recovery**: `SELECT FOR UPDATE` prevents this at the DB level; if it occurs (bug), cancel excess orders and notify customers
- **Prevention**: Always use the reservation pattern; never check-then-update without a lock

### Reservation Expiry Job Failure
- **Impact**: Expired reservations hold stock indefinitely; available stock appears lower than actual
- **Recovery**: Idempotent expiry job; re-run catches all expired reservations
- **Prevention**: Monitor reservation age; alert if reservations older than 2× TTL exist

### Stock Level Drift (Audit Mismatch)
- **Impact**: `stock_levels.quantity_on_hand` doesn't match sum of movements
- **Recovery**: Reconciliation query sums movements and compares to current level; manual adjustment with audit record
- **Prevention**: Always update `stock_levels` and insert `stock_movements` in the same transaction

### Warehouse Transfer Partial Failure
- **Impact**: Stock decremented from source warehouse but not added to destination
- **Recovery**: Transfer runs in a single transaction — both updates succeed or both roll back
- **Prevention**: Wrap both `UPDATE stock_levels` statements in one transaction; never do them separately
