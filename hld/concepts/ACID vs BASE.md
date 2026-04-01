## 1. ACID vs BASE: The Fundamental Tradeoff

ACID and BASE represent two opposing philosophies for handling data in distributed systems.

```
ACID (SQL databases):
  Strong consistency, reliability, correctness
  "Do it right or don't do it at all"

BASE (NoSQL databases):
  Availability, performance, scalability
  "Do it fast, fix it later"
```

**The tradeoff:** Consistency vs Availability (CAP theorem).

---

## 2. ACID Properties

ACID guarantees that database transactions are processed reliably.

### Atomicity

All operations in a transaction succeed or all fail. No partial state.

```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- deduct
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- add
COMMIT;

If either UPDATE fails → both are rolled back
Never: Account 1 debited but Account 2 not credited
```

**Real-world example:** Money transfer. Either both accounts update or neither does.

**Implementation:** Write-Ahead Log (WAL). Changes logged before applied. On crash, replay or rollback.

---

### Consistency

Database moves from one valid state to another. Constraints always satisfied.

```sql
CREATE TABLE accounts (
  id INT PRIMARY KEY,
  balance DECIMAL CHECK (balance >= 0)  -- constraint: no negative balance
);

-- This transaction will fail:
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
  -- If balance becomes negative → constraint violated → rollback
COMMIT;
```

**Constraints enforced:**
- Primary keys (uniqueness)
- Foreign keys (referential integrity)
- Check constraints (business rules)
- NOT NULL constraints

**Database prevents invalid state.**

---

### Isolation

Concurrent transactions don't interfere with each other.

```
Transaction A: Transfer $100 from Account 1 to Account 2
Transaction B: Read balance of Account 1

Without isolation:
  A reads balance: $500
  B reads balance: $500  (same time)
  A writes: $400
  B writes: $400  (overwrites A's change — lost update!)

With isolation:
  A locks Account 1
  B waits for A to complete
  B then sees correct balance: $400
```

**Isolation levels** (weakest to strongest):

```
Read Uncommitted:
  Can see uncommitted changes from other transactions
  Problem: Dirty reads (reading data that might be rolled back)

Read Committed:
  Only see committed changes
  Most common default
  Problem: Non-repeatable reads (same query returns different results)

Repeatable Read:
  Same query returns same results within transaction
  Problem: Phantom reads (new rows appear)

Serializable:
  Transactions appear to run sequentially
  Strongest isolation, lowest concurrency
```

**Tradeoff:** Stronger isolation = more locking = lower concurrency = slower.

---

### Durability

Once committed, data survives crashes.

```
Transaction commits → data written to disk (via WAL)
Power failure 1 second later → data still there on restart
```

**Implementation:**
- Write-Ahead Log (WAL): Changes logged to disk before data files
- fsync: Force OS to flush to physical disk
- On crash: Replay WAL to recover committed transactions

---

## 3. BASE Properties

BASE is the opposite philosophy: prioritize availability over consistency.

### Basically Available

System responds to requests even during partial failures.

```
3-node cluster:
  Node 1: up
  Node 2: up
  Node 3: down

ACID system: Might reject requests (can't reach quorum)
BASE system: Accepts requests (2 nodes still available)
```

**"Basically" = not guaranteed, but usually available.**

---

### Soft State

Data may change over time without new input (due to eventual consistency).

```
Write to Node 1: user.email = "alice@new.com"
Read from Node 2: user.email = "alice@old.com"  (not yet replicated)

Wait 100ms...

Read from Node 2: user.email = "alice@new.com"  (now replicated)
```

**State is "soft" — not fixed, converging.**

---

### Eventually Consistent

Given no new updates, all replicas will eventually converge to the same value.

```
Time 0:  Write to Node 1
Time 1:  Node 1 has new value, Node 2 has old value (inconsistent)
Time 2:  Replication completes, both nodes have new value (consistent)
```

**"Eventually" = not immediate, but will happen.**

**Typical convergence time:** Milliseconds to seconds.

---

## 4. ACID vs BASE Comparison

|Aspect|ACID|BASE|
|---|---|---|
|Consistency|Strong (immediate)|Eventual (delayed)|
|Availability|May be unavailable during failures|Highly available|
|Partition tolerance|Sacrifices availability|Sacrifices consistency|
|Transactions|Multi-row, multi-table|Single document/row (usually)|
|Latency|Higher (coordination overhead)|Lower (no coordination)|
|Scalability|Vertical + read replicas|Horizontal (add nodes)|
|Use case|Financial, inventory, bookings|Social feeds, carts, analytics|
|Examples|PostgreSQL, MySQL|Cassandra, DynamoDB, MongoDB (default)|

---

## 5. Real-World Examples

### ACID: Bank transfer

```sql
-- PostgreSQL
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
  
  -- Check constraints
  IF (SELECT balance FROM accounts WHERE id = 1) < 0 THEN
    ROLLBACK;  -- insufficient funds
  ELSE
    COMMIT;    -- both updates succeed atomically
  END IF;
```

**Why ACID:** Money can't be lost or duplicated. Consistency is non-negotiable.

---

### BASE: Social media feed

```javascript
// Cassandra
// User posts a status update
INSERT INTO posts (user_id, post_id, content, timestamp)
VALUES (123, uuid(), 'Hello world!', now());

// Replicate to 3 nodes asynchronously
// Consistency level: ONE (write succeeds when 1 node acknowledges)

// Followers read from any replica
// Might see post immediately, or after a few milliseconds
// Eventually, all followers see it
```

**Why BASE:** Availability matters more than immediate consistency. A few milliseconds delay is acceptable.

---

## 6. When to Use ACID

**Use ACID when:**

```
✅ Correctness is critical
   - Financial transactions (payments, transfers)
   - Inventory management (prevent overselling)
   - Booking systems (prevent double-booking)
   - Order processing

✅ Multi-row transactions needed
   - Transfer between accounts
   - Order + inventory update + payment

✅ Strong consistency required
   - Read-your-writes guarantee
   - No stale data acceptable

✅ Complex queries with joins
   - Reporting across multiple tables
   - Ad-hoc analytics
```

**Examples:**
- Banking systems
- E-commerce order processing
- Reservation systems (flights, hotels, tickets)
- Healthcare records
- Accounting systems

---

## 7. When to Use BASE

**Use BASE when:**

```
✅ Availability is critical
   - Service must stay up during failures
   - Partial degradation acceptable

✅ High write throughput needed
   - Millions of writes/second
   - Single SQL node can't handle load

✅ Eventual consistency acceptable
   - Social media feeds (delay OK)
   - Analytics (approximate counts OK)
   - Caching (stale data OK)

✅ Simple access patterns
   - Lookup by ID
   - No complex joins
```

**Examples:**
- Social media feeds
- Shopping carts
- User activity logs
- Analytics and metrics
- Content delivery
- Session storage

---

## 8. Hybrid Approaches

Most real systems use both ACID and BASE.

### Polyglot persistence

Different databases for different needs.

```
E-commerce system:

ACID (PostgreSQL):
  - User accounts
  - Orders
  - Payments
  - Inventory

BASE (Cassandra):
  - User activity logs
  - Product view history
  - Search history

BASE (Redis):
  - Session storage
  - Shopping cart
  - Cache
```

**Each database chosen for its strengths.**

---

### Tunable consistency

Some NoSQL databases let you choose per-operation.

**Cassandra:**

```sql
-- Strong consistency (ACID-like)
SELECT * FROM users WHERE id = 123 
USING CONSISTENCY QUORUM;

-- Eventual consistency (BASE)
SELECT * FROM users WHERE id = 123 
USING CONSISTENCY ONE;
```

**DynamoDB:**

```python
# Eventual consistency (default, BASE)
response = table.get_item(Key={'id': 123})

# Strong consistency (ACID-like)
response = table.get_item(
    Key={'id': 123},
    ConsistentRead=True
)
```

---

### Saga pattern

Distributed transactions without ACID.

```
Order Service → Payment Service → Inventory Service

Each step is a separate transaction (BASE)
If any step fails → compensating transactions (rollback)

Example:
  1. Create order (commit)
  2. Charge payment (commit)
  3. Decrement inventory (fails)
  4. Refund payment (compensating transaction)
  5. Cancel order (compensating transaction)
```

**Eventual consistency with compensation.**

---

## 9. The Spectrum

ACID and BASE aren't binary. There's a spectrum.

```
Strongest consistency:
  ↓
Serializable (ACID)
  ↓
Repeatable Read
  ↓
Read Committed
  ↓
Read Uncommitted
  ↓
Causal Consistency
  ↓
Read-Your-Writes
  ↓
Monotonic Reads
  ↓
Eventual Consistency (BASE)
  ↓
Weakest consistency
```

**Most systems operate somewhere in the middle.**

---

## 10. Common Misconceptions

### Misconception 1: "NoSQL = no consistency"

**False.** NoSQL databases offer eventual consistency by default, but many support stronger consistency.

```
Cassandra: QUORUM reads/writes → strong consistency
MongoDB: Majority read/write concern → strong consistency
DynamoDB: ConsistentRead=True → strong consistency
```

---

### Misconception 2: "ACID = slow"

**False.** Well-tuned ACID databases are very fast. PostgreSQL handles 10K+ TPS easily.

**ACID adds overhead, but it's not inherently slow.**

---

### Misconception 3: "BASE = no transactions"

**False.** Some NoSQL databases support transactions.

```
MongoDB: Multi-document transactions (since 4.0)
DynamoDB: TransactWriteItems (up to 25 items)
```

**Limited compared to SQL, but they exist.**

---

## 11. Common Interview Questions + Answers

### Q: Explain ACID properties and why they matter.

> "ACID ensures reliable transactions. Atomicity means all-or-nothing — a money transfer either completes fully or not at all. Consistency enforces constraints — you can't have negative balances. Isolation prevents concurrent transactions from interfering — two users can't double-book the same seat. Durability guarantees committed data survives crashes. These properties are critical for financial systems, e-commerce, and any application where data correctness is non-negotiable."

### Q: What's the difference between ACID and BASE?

> "ACID prioritizes consistency and correctness — transactions are atomic, isolated, and durable. It's used in SQL databases for financial transactions and inventory management. BASE prioritizes availability and performance — it accepts eventual consistency in exchange for higher availability and scalability. It's used in NoSQL databases for social feeds, analytics, and caching. The tradeoff is consistency vs availability — you can't have both during network partitions (CAP theorem)."

### Q: When would you choose BASE over ACID?

> "Choose BASE when availability matters more than immediate consistency, and when you need horizontal write scaling beyond what a single ACID database can handle. Examples: social media feeds where a few milliseconds delay is acceptable, shopping carts where eventual consistency is fine, or analytics where approximate counts are OK. But for financial transactions, inventory management, or booking systems where correctness is critical, always choose ACID."

### Q: Can you have both ACID and BASE in the same system?

> "Yes, and it's common — this is called polyglot persistence. An e-commerce system might use PostgreSQL (ACID) for orders and payments where consistency is critical, Cassandra (BASE) for user activity logs where high write throughput matters, and Redis (BASE) for caching and sessions. Each database is chosen for its strengths. You can also use tunable consistency in databases like Cassandra — use QUORUM for critical reads (ACID-like) and ONE for high-throughput writes (BASE)."

---

## 12. Interview Tricks & Pitfalls

### ✅ Trick 1: Connect to CAP theorem

ACID and BASE map to CAP:

> "ACID systems are typically CP — they prioritize consistency over availability during partitions. BASE systems are typically AP — they prioritize availability over consistency. This is the CAP theorem tradeoff."

### ✅ Trick 2: Give concrete examples

Don't just define ACID and BASE. Give examples:

> "For the payment service, I need ACID — PostgreSQL with transactions. For the activity feed, I can use BASE — Cassandra with eventual consistency. For the shopping cart, BASE is fine — Redis with a short TTL."

### ✅ Trick 3: Mention tunable consistency

Shows you know modern NoSQL isn't just "eventually consistent":

> "Cassandra is tunable. With consistency level ONE, it's BASE — highly available but eventually consistent. With QUORUM, it's ACID-like — strongly consistent but less available during failures. I'd choose per-operation based on requirements."

### ❌ Pitfall 1: Saying "ACID is always better"

ACID has tradeoffs. Don't dismiss BASE:

> "ACID provides strong guarantees but limits scalability and availability. For use cases where eventual consistency is acceptable, BASE systems can handle much higher throughput and stay available during failures."

### ❌ Pitfall 2: Treating BASE as "no consistency"

BASE is eventual consistency, not no consistency:

> "BASE systems eventually converge. After replication completes (typically milliseconds), all replicas have the same data. It's not chaos — it's a deliberate tradeoff."

### ❌ Pitfall 3: Not knowing isolation levels

If you mention ACID, be ready to discuss isolation levels:

> "Most databases default to Read Committed isolation — you only see committed changes. For critical operations, I'd use Serializable isolation to prevent all concurrency anomalies, accepting the performance cost."

---

## 13. Quick Reference

```
ACID = Strong consistency, reliability, correctness

Atomicity:    All-or-nothing transactions
Consistency:  Constraints always satisfied
Isolation:    Concurrent transactions don't interfere
Durability:   Committed data survives crashes

Use ACID for:
  - Financial transactions
  - Inventory management
  - Booking systems
  - Multi-row transactions
  - Strong consistency required

Examples: PostgreSQL, MySQL, Oracle

---

BASE = Availability, performance, scalability

Basically Available:     Responds even during failures
Soft State:              Data may change without input
Eventually Consistent:   Replicas converge eventually

Use BASE for:
  - High availability required
  - High write throughput
  - Eventual consistency acceptable
  - Simple access patterns

Examples: Cassandra, DynamoDB, Riak

---

Comparison:
  ACID: CP (consistency + partition tolerance)
  BASE: AP (availability + partition tolerance)

Hybrid approaches:
  - Polyglot persistence (different DBs for different needs)
  - Tunable consistency (Cassandra QUORUM vs ONE)
  - Saga pattern (distributed transactions with compensation)

Spectrum:
  Serializable → Repeatable Read → Read Committed → 
  Causal → Read-Your-Writes → Eventual

Key insight: Not binary. Most systems use both or operate in the middle.
```
