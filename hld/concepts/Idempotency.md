## 1. What Is Idempotency?

An operation is idempotent if performing it multiple times has the same effect as performing it once.

```
Idempotent:
  SET x = 5
  Execute once:  x = 5
  Execute twice: x = 5  (same result)

Not idempotent:
  INCREMENT x
  Execute once:  x = 1
  Execute twice: x = 2  (different result)
```

**In distributed systems:** Idempotency ensures safe retries. If a request fails or times out, you can retry without causing duplicate side effects.

---

## 2. Why Idempotency Matters

### The retry problem

```
Client → Server: "Charge $100"
Server: Processes payment, charges $100
Server → Client: Response (but network fails, client doesn't receive it)

Client thinks: "Request failed, let me retry"
Client → Server: "Charge $100" (again)
Server: Charges $100 again

Result: Customer charged $200 instead of $100
```

**Without idempotency:** Retries cause duplicate operations.

**With idempotency:** Retries are safe.

```
Client → Server: "Charge $100 with idempotency key: abc123"
Server: Processes payment, charges $100, stores key abc123
Server → Client: Response (network fails)

Client → Server: "Charge $100 with idempotency key: abc123" (retry)
Server: Sees key abc123 already processed → returns original result, no duplicate charge

Result: Customer charged $100 (correct)
```

---

## 3. Idempotent vs Non-Idempotent Operations

### HTTP methods

```
Idempotent:
  GET    /users/123        → Read user (safe to retry)
  PUT    /users/123        → Update user (same result if repeated)
  DELETE /users/123        → Delete user (deleting twice = same as once)

Not idempotent:
  POST   /users            → Create user (creates duplicate if repeated)
  POST   /orders           → Create order (creates duplicate if repeated)
```

**GET, PUT, DELETE are naturally idempotent. POST is not.**

---

### Database operations

```
Idempotent:
  UPDATE users SET email = 'alice@example.com' WHERE id = 123
  (Running twice sets email to same value)

  DELETE FROM users WHERE id = 123
  (Deleting twice has same effect as once)

Not idempotent:
  INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com')
  (Running twice creates duplicate rows)

  UPDATE users SET login_count = login_count + 1 WHERE id = 123
  (Running twice increments twice)
```

---

## 4. Implementing Idempotency

### Idempotency keys

Client generates a unique key for each operation. Server tracks processed keys.

```
Client:
  idempotency_key = generate_uuid()  # "abc123"
  POST /payments
  Headers: Idempotency-Key: abc123
  Body: {"amount": 100, "user_id": 456}

Server:
  1. Check if key "abc123" already processed
  2. If yes → return cached result (no duplicate charge)
  3. If no → process payment, store key + result, return result
```

**Implementation:**

```python
def process_payment(amount, user_id, idempotency_key):
    # Check if already processed
    cached = redis.get(f"idempotency:{idempotency_key}")
    if cached:
        return json.loads(cached)  # return cached result
    
    # Process payment
    result = charge_customer(amount, user_id)
    
    # Store result with key (24-hour TTL)
    redis.setex(
        f"idempotency:{idempotency_key}",
        86400,  # 24 hours
        json.dumps(result)
    )
    
    return result
```

---

### Database-backed idempotency

Store idempotency keys in database for durability.

```sql
CREATE TABLE idempotency_keys (
  key VARCHAR(255) PRIMARY KEY,
  result JSON,
  created_at TIMESTAMP,
  INDEX idx_created_at (created_at)
);

-- Process request
BEGIN TRANSACTION;
  -- Try to insert key
  INSERT INTO idempotency_keys (key, result, created_at)
  VALUES ('abc123', NULL, NOW())
  ON CONFLICT (key) DO NOTHING;
  
  -- If key already exists, return cached result
  IF NOT INSERTED THEN
    SELECT result FROM idempotency_keys WHERE key = 'abc123';
    COMMIT;
    RETURN result;
  END IF;
  
  -- Process payment
  result = charge_customer(amount, user_id);
  
  -- Update result
  UPDATE idempotency_keys SET result = result WHERE key = 'abc123';
COMMIT;
```

---

### Natural idempotency (PUT)

Use PUT with resource ID instead of POST.

```
Not idempotent:
  POST /orders
  Body: {"product_id": 123, "quantity": 2}
  → Creates new order each time

Idempotent:
  PUT /orders/abc123
  Body: {"product_id": 123, "quantity": 2}
  → Creates or updates order abc123
  → Running twice has same effect
```

**Client generates the ID (UUID), not the server.**

---

## 5. Idempotency Key Generation

### Client-generated (recommended)

```javascript
// Client generates UUID
const idempotencyKey = uuidv4();  // "550e8400-e29b-41d4-a716-446655440000"

fetch('/api/payments', {
  method: 'POST',
  headers: {
    'Idempotency-Key': idempotencyKey
  },
  body: JSON.stringify({amount: 100})
});
```

**Benefits:**
- Client controls retries
- Works across network failures
- Client can retry with same key

---

### Request-based (hash of request)

```python
def generate_idempotency_key(user_id, amount, timestamp):
    # Hash of request parameters
    data = f"{user_id}:{amount}:{timestamp}"
    return hashlib.sha256(data.encode()).hexdigest()

# Same request → same key
key1 = generate_idempotency_key(123, 100, "2024-01-15")
key2 = generate_idempotency_key(123, 100, "2024-01-15")
# key1 == key2
```

**Problem:** If user legitimately wants to make same request twice (e.g., two $100 payments), they can't.

**Better:** Client-generated UUID.

---

## 6. Idempotency Window

How long should you store idempotency keys?

```
Too short (1 hour):
  Client retries after 2 hours → treated as new request → duplicate

Too long (forever):
  Storage grows unbounded
  User can never make same request again

Reasonable (24 hours):
  Covers most retry scenarios
  Bounded storage
  After 24 hours, key expires
```

**Stripe uses 24 hours. Most APIs use 24-72 hours.**

---

## 7. Idempotency in Distributed Systems

### At-least-once delivery

Message queues guarantee at-least-once delivery. Consumer must be idempotent.

```
Producer → Queue → Consumer

Message: "Charge user 123 $100"

Scenario:
  1. Consumer receives message
  2. Consumer processes (charges $100)
  3. Consumer crashes before acknowledging
  4. Queue redelivers message
  5. Consumer receives same message again

Without idempotency: User charged $200
With idempotency: Consumer detects duplicate, skips processing
```

**Implementation:**

```python
def process_message(message):
    message_id = message['id']
    
    # Check if already processed
    if redis.exists(f"processed:{message_id}"):
        return  # already processed, skip
    
    # Process message
    charge_customer(message['user_id'], message['amount'])
    
    # Mark as processed
    redis.set(f"processed:{message_id}", "1", ex=86400)
```

---

### Distributed transactions

Saga pattern with idempotent steps.

```
Order creation saga:
  1. Create order (idempotent: PUT /orders/{order_id})
  2. Reserve inventory (idempotent: PUT /inventory/reservations/{order_id})
  3. Charge payment (idempotent: POST /payments with idempotency key)

If step 3 fails and retries:
  - Step 1 already done → no duplicate order
  - Step 2 already done → no duplicate reservation
  - Step 3 retries safely with idempotency key
```

---

## 8. Idempotency Patterns

### Pattern 1: Unique constraint

Use database unique constraint to prevent duplicates.

```sql
CREATE TABLE payments (
  id SERIAL PRIMARY KEY,
  idempotency_key VARCHAR(255) UNIQUE,
  user_id INT,
  amount DECIMAL
);

-- Insert with idempotency key
INSERT INTO payments (idempotency_key, user_id, amount)
VALUES ('abc123', 456, 100.00)
ON CONFLICT (idempotency_key) DO NOTHING;

-- If key already exists, INSERT is skipped (no duplicate)
```

---

### Pattern 2: Check-then-act

Check if operation already done before doing it.

```python
def delete_user(user_id):
    # Check if user exists
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    if not user:
        return {"status": "already_deleted"}  # idempotent
    
    # Delete user
    db.execute("DELETE FROM users WHERE id = ?", user_id)
    
    return {"status": "deleted"}
```

**Both responses are success. Calling twice has same effect.**

---

### Pattern 3: Version numbers

Use version numbers to detect concurrent updates.

```sql
-- User table with version
CREATE TABLE users (
  id INT PRIMARY KEY,
  email VARCHAR(255),
  version INT DEFAULT 0
);

-- Update with version check
UPDATE users
SET email = 'alice@new.com', version = version + 1
WHERE id = 123 AND version = 5;

-- If version changed (another update happened), UPDATE affects 0 rows
-- Client detects conflict and retries with new version
```

**Optimistic locking ensures idempotency.**

---

## 9. Real-World Examples

### Stripe payments

```
POST /v1/charges
Headers: Idempotency-Key: abc123
Body: {"amount": 1000, "currency": "usd", "source": "tok_visa"}

First request: Charge created
Retry with same key: Returns original charge (no duplicate)

Idempotency key valid for 24 hours
```

---

### AWS S3 PUT

```
PUT /bucket/object.txt
Body: "Hello World"

First request: Object created
Second request: Object updated (same content)
Third request: Object updated (same content)

Result: Object contains "Hello World" (idempotent)
```

---

### Kafka exactly-once semantics

```
Producer:
  - Idempotent producer (enabled by default in Kafka 3.0+)
  - Assigns sequence number to each message
  - Broker deduplicates based on sequence number

Consumer:
  - Transactional consumer
  - Commits offset only after processing
  - If crash before commit, message redelivered but not reprocessed
```

---

## 10. Common Interview Questions + Answers

### Q: What is idempotency and why is it important?

> "Idempotency means performing an operation multiple times has the same effect as performing it once. It's critical in distributed systems because network failures and timeouts are common. If a client sends a payment request and doesn't receive a response, it doesn't know if the payment succeeded or failed. Without idempotency, retrying could charge the customer twice. With idempotency, the client can safely retry — the server detects the duplicate request and returns the original result without processing it again."

### Q: How do you implement idempotency for a payment API?

> "I'd use idempotency keys. The client generates a unique UUID for each payment and sends it in an Idempotency-Key header. The server checks if this key has been processed before. If yes, it returns the cached result without charging again. If no, it processes the payment, stores the key and result in Redis or a database with a 24-hour TTL, and returns the result. This ensures that retries with the same key don't create duplicate charges, while still allowing the user to make multiple legitimate payments."

### Q: What's the difference between idempotent and safe operations?

> "Safe operations don't modify state — like GET requests. They can be called repeatedly without side effects. Idempotent operations may modify state, but calling them multiple times has the same effect as calling once — like PUT or DELETE. GET is both safe and idempotent. PUT and DELETE are idempotent but not safe. POST is neither safe nor idempotent. For example, GET /users/123 is safe (just reads). PUT /users/123 is idempotent (updates to same value). POST /users creates a new user each time (not idempotent)."

### Q: How do you handle idempotency in a message queue?

> "Message queues typically guarantee at-least-once delivery, meaning messages can be delivered multiple times. To handle this, I'd make the consumer idempotent. Each message has a unique ID. Before processing, the consumer checks if this ID has been processed (stored in Redis or database). If yes, it skips processing and acknowledges the message. If no, it processes the message, stores the ID as processed, and acknowledges. This ensures duplicate messages don't cause duplicate side effects, like charging a customer twice."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Connect to retry scenarios

Don't just define idempotency. Explain why it matters:

> "In distributed systems, network failures and timeouts are common. A client might send a request, the server processes it, but the response is lost. The client doesn't know if it succeeded, so it retries. Without idempotency, this creates duplicates. With idempotency, retries are safe."

### ✅ Trick 2: Mention the idempotency window

Show you understand practical considerations:

> "I'd store idempotency keys for 24 hours. This covers most retry scenarios while keeping storage bounded. After 24 hours, the key expires and the user can make the same request again if needed."

### ✅ Trick 3: Discuss natural idempotency

Not everything needs idempotency keys:

> "PUT and DELETE are naturally idempotent. For updates, I'd use PUT /users/123 instead of POST /users. For deletions, DELETE /users/123 can be called multiple times safely. Only POST operations that create resources need explicit idempotency keys."

### ❌ Pitfall 1: Confusing idempotency with deduplication

Idempotency is about safe retries, not preventing all duplicates:

> "Idempotency allows the same request to be retried safely. It doesn't prevent a user from making two separate $100 payments — those are different requests with different idempotency keys. It only prevents accidental duplicates from retries."

### ❌ Pitfall 2: Not mentioning storage

If you propose idempotency keys, address where they're stored:

> "I'd store idempotency keys in Redis with a 24-hour TTL for fast lookups. For critical operations like payments, I'd also persist them to the database for durability in case Redis fails."

### ❌ Pitfall 3: Forgetting about message queues

Idempotency is critical for at-least-once delivery:

> "Since Kafka guarantees at-least-once delivery, the consumer must be idempotent. I'd track processed message IDs in Redis to detect and skip duplicates."

---

## 12. Quick Reference

```
Idempotency = performing operation multiple times has same effect as once

Why it matters:
  - Network failures → retries
  - Message queues → at-least-once delivery
  - Distributed transactions → saga retries
  - Without idempotency → duplicates (double charges, duplicate orders)

HTTP methods:
  Idempotent:     GET, PUT, DELETE
  Not idempotent: POST

Implementation:
  1. Idempotency keys (client-generated UUID)
  2. Natural idempotency (PUT instead of POST)
  3. Unique constraints (database prevents duplicates)
  4. Check-then-act (check if already done)
  5. Version numbers (optimistic locking)

Idempotency key flow:
  1. Client generates UUID
  2. Client sends request with Idempotency-Key header
  3. Server checks if key already processed
  4. If yes → return cached result (no duplicate)
  5. If no → process, store key + result, return result

Storage:
  - Redis (fast, TTL support)
  - Database (durable, for critical operations)
  - TTL: 24-72 hours (covers retries, bounds storage)

Message queues:
  - At-least-once delivery → consumer must be idempotent
  - Track processed message IDs
  - Skip duplicates

Patterns:
  Unique constraint:  Database prevents duplicate inserts
  Check-then-act:     Check if already done before doing
  Version numbers:    Optimistic locking

Real-world:
  Stripe:  24-hour idempotency keys for payments
  AWS S3:  PUT is naturally idempotent
  Kafka:   Idempotent producer + transactional consumer

Key insight: Idempotency enables safe retries in distributed systems
```
