# E-Commerce API and DB Design

## Problem Statement

Design the API and database schema for an e-commerce platform (like Amazon). Users can browse products, add items to a cart, place orders, and track order status.

**Core requirements:**
- Product catalog (browse, search, view details)
- Shopping cart (add/remove/update items)
- Order placement and payment
- Order status tracking
- Inventory management

**Scale:** 10M products, 1M orders/day, 100M product views/day

---

## API Design

### Endpoints

#### Product Catalog
```
GET /api/v1/products?category=electronics&page=1&pageSize=20&sort=price_asc
GET /api/v1/products/{productId}

Response 200 (list):
{
  "items": [
    {
      "id": "prod_abc",
      "name": "Wireless Headphones",
      "price": 79.99,
      "currency": "USD",
      "stockCount": 150,
      "category": "electronics",
      "imageUrl": "https://cdn.example.com/prod_abc.jpg"
    }
  ],
  "total": 500,
  "page": 1,
  "pageSize": 20
}
```

#### Cart Management
```
GET /api/v1/cart
POST /api/v1/cart/items
PUT /api/v1/cart/items/{productId}
DELETE /api/v1/cart/items/{productId}
Authorization: Bearer <token>

POST /api/v1/cart/items Request:
{
  "productId": "prod_abc",
  "quantity": 2
}
```

#### Place Order
```
POST /api/v1/orders
Authorization: Bearer <token>

Request:
{
  "cartId": "cart_xyz",
  "shippingAddressId": "addr_123",
  "paymentMethodId": "pm_456"
}

Response 201:
{
  "orderId": "ord_789",
  "status": "PENDING_PAYMENT",
  "total": 159.98,
  "items": [...],
  "estimatedDelivery": "2024-01-20"
}
```

#### Order Management
```
GET /api/v1/orders                    // list user's orders
GET /api/v1/orders/{orderId}          // order details + status
POST /api/v1/orders/{orderId}/cancel  // cancel order
```

---

## Database Schema

### products table
```sql
CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(500) NOT NULL,
    description TEXT,
    price       DECIMAL(10, 2) NOT NULL,
    currency    CHAR(3) NOT NULL DEFAULT 'USD',
    category_id BIGINT REFERENCES categories(id),
    seller_id   BIGINT REFERENCES users(id),
    stock_count INT NOT NULL DEFAULT 0,
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_category_id ON products (category_id);
CREATE INDEX idx_products_seller_id ON products (seller_id);
CREATE INDEX idx_products_price ON products (price);
```

### categories table
```sql
CREATE TABLE categories (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    parent_id   BIGINT REFERENCES categories(id),  -- self-referential for hierarchy
    slug        VARCHAR(100) NOT NULL UNIQUE
);
```

### carts table
```sql
CREATE TABLE carts (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL REFERENCES users(id) UNIQUE,  -- one cart per user
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE cart_items (
    cart_id     BIGINT NOT NULL REFERENCES carts(id),
    product_id  BIGINT NOT NULL REFERENCES products(id),
    quantity    INT NOT NULL CHECK (quantity > 0),
    added_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (cart_id, product_id)
);
```

### orders table
```sql
CREATE TABLE orders (
    id                  BIGSERIAL PRIMARY KEY,
    user_id             BIGINT NOT NULL REFERENCES users(id),
    status              VARCHAR(50) NOT NULL DEFAULT 'PENDING_PAYMENT',
                        -- PENDING_PAYMENT, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED, REFUNDED
    subtotal            DECIMAL(10, 2) NOT NULL,
    tax                 DECIMAL(10, 2) NOT NULL DEFAULT 0,
    shipping_cost       DECIMAL(10, 2) NOT NULL DEFAULT 0,
    total               DECIMAL(10, 2) NOT NULL,
    shipping_address_id BIGINT REFERENCES addresses(id),
    payment_method_id   BIGINT REFERENCES payment_methods(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_orders_user_id ON orders (user_id);
CREATE INDEX idx_orders_status ON orders (status);
CREATE INDEX idx_orders_created_at ON orders (created_at DESC);
```

### order_items table
```sql
CREATE TABLE order_items (
    id          BIGSERIAL PRIMARY KEY,
    order_id    BIGINT NOT NULL REFERENCES orders(id),
    product_id  BIGINT NOT NULL REFERENCES products(id),
    quantity    INT NOT NULL,
    unit_price  DECIMAL(10, 2) NOT NULL,  -- snapshot of price at order time
    total_price DECIMAL(10, 2) NOT NULL
);

CREATE INDEX idx_order_items_order_id ON order_items (order_id);
```

### inventory_reservations table
```sql
CREATE TABLE inventory_reservations (
    id          BIGSERIAL PRIMARY KEY,
    product_id  BIGINT NOT NULL REFERENCES products(id),
    order_id    BIGINT NOT NULL REFERENCES orders(id),
    quantity    INT NOT NULL,
    expires_at  TIMESTAMPTZ NOT NULL,
    status      VARCHAR(20) NOT NULL DEFAULT 'ACTIVE'  -- ACTIVE, CONFIRMED, RELEASED
);
```

---

## Trade-offs and Considerations

### Inventory Management and Race Conditions
- Naive approach: `UPDATE products SET stock_count = stock_count - quantity WHERE id = ? AND stock_count >= quantity`
- This uses optimistic locking at the DB level — if `stock_count` drops to 0 between check and update, the update affects 0 rows
- Better: Use `inventory_reservations` table — reserve stock when item is added to cart, confirm on payment, release on expiry/cancellation
- Prevents overselling without pessimistic locking

### Price Snapshot in order_items
- `unit_price` in `order_items` stores the price at order time — not a foreign key to current price
- Product prices change; order history must reflect what the customer actually paid
- This is intentional denormalization

### Order Status Machine
```
PENDING_PAYMENT → PAID → PROCESSING → SHIPPED → DELIVERED
                ↓                              ↓
            CANCELLED                      REFUNDED
```
- Enforce valid transitions in application code
- Log all status changes to an `order_status_history` table for audit trail

### Cart Storage
- Option A: DB-backed cart (shown above) — durable, works across devices
- Option B: Redis-backed cart — faster, but lost on expiry
- Recommendation: DB-backed for logged-in users; Redis/cookie for anonymous users, merge on login

### Search
- Full-text search on product names/descriptions requires a search index (Elasticsearch, PostgreSQL full-text search)
- Do not use `LIKE '%keyword%'` — cannot use indexes, full table scan
