## 1. What Is a SQL Database?

A SQL (Structured Query Language) database stores data in tables with rows and columns. It enforces a schema and provides ACID guarantees for transactions.

```
Users table:
┌────┬──────────┬───────────────────┬─────────┐
│ id │ name     │ email             │ age     │
├────┼──────────┼───────────────────┼─────────┤
│  1 │ Alice    │ alice@example.com │  28     │
│  2 │ Bob      │ bob@example.com   │  35     │
│  3 │ Charlie  │ charlie@ex.com    │  42     │
└────┴──────────┴───────────────────┴─────────┘

Orders table:
┌────┬─────────┬──────────┬────────────┐
│ id │ user_id │ amount   │ created_at │
├────┼─────────┼──────────┼────────────┤
│  1 │    1    │  $49.99  │ 2024-01-15 │
│  2 │    1    │  $12.00  │ 2024-01-16 │
│  3 │    2    │  $99.00  │ 2024-01-16 │
└────┴─────────┴──────────┴────────────┘
```

**Key characteristics:**
- Fixed schema (columns defined upfront)
- Relationships via foreign keys
- ACID transactions
- Rich query language (SQL)
- Strong consistency

---

## 2. ACID Properties

### Atomicity

All operations in a transaction succeed or all fail. No partial state.

```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

If either UPDATE fails → both are rolled back
Account balances never in inconsistent state
```

**Real-world example:** Transferring money between accounts. Either both updates happen or neither does.

---

### Consistency

Database moves from one valid state to another. Constraints are always satisfied.

```sql
CREATE TABLE accounts (
  id INT PRIMARY KEY,
  balance DECIMAL CHECK (balance >= 0)  -- constraint
);

-- This will fail:
UPDATE accounts SET balance = -50 WHERE id = 1;
-- Error: CHECK constraint violated

Database prevents invalid state
```

**Constraints enforced:**
- Primary keys (uniqueness)
- Foreign keys (referential integrity)
- Check constraints (business rules)
- NOT NULL constraints

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

**Isolation levels** (from weakest to strongest):

```
Read Uncommitted:  Can see uncommitted changes (dirty reads)
Read Committed:    Only see committed changes (most common default)
Repeatable Read:   Same query returns same results within transaction
Serializable:      Transactions appear to run sequentially (strongest)
```

**Tradeoff:** Stronger isolation = more locking = lower concurrency.

---

### Durability

Once committed, data survives crashes.

```
Transaction commits → data written to disk (via WAL)
Power failure 1 second later → data still there on restart
```

**Implementation:** Write-Ahead Log (WAL). Changes written to log before data files. On crash, replay log to recover.

---

## 3. Schema and Data Modeling

### Normalization

Organize data to reduce redundancy.

**Unnormalized (bad):**

```
Orders table:
┌────┬──────────┬───────────────────┬──────────┐
│ id │ user_name│ user_email        │ amount   │
├────┼──────────┼───────────────────┼──────────┤
│  1 │ Alice    │ alice@example.com │  $49.99  │
│  2 │ Alice    │ alice@example.com │  $12.00  │
│  3 │ Bob      │ bob@example.com   │  $99.00  │
└────┴──────────┴───────────────────┴──────────┘

Problem: User data duplicated. If Alice changes email, must update all rows.
```

**Normalized (good):**

```
Users table:
┌────┬──────────┬───────────────────┐
│ id │ name     │ email             │
├────┼──────────┼───────────────────┤
│  1 │ Alice    │ alice@example.com │
│  2 │ Bob      │ bob@example.com   │
└────┴──────────┴───────────────────┘

Orders table:
┌────┬─────────┬──────────┐
│ id │ user_id │ amount   │
├────┼─────────┼──────────┤
│  1 │    1    │  $49.99  │
│  2 │    1    │  $12.00  │
│  3 │    2    │  $99.00  │
└────┴─────────┴──────────┘

User data stored once. Orders reference via foreign key.
```

**Normal forms:**
- 1NF: Atomic values (no arrays in cells)
- 2NF: No partial dependencies
- 3NF: No transitive dependencies (most common target)

**When to denormalize:** For read-heavy workloads where joins are expensive. Trade storage for query speed.

---

### Relationships

**One-to-Many:**

```sql
-- One user has many orders
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(100)
);

CREATE TABLE orders (
  id INT PRIMARY KEY,
  user_id INT REFERENCES users(id),
  amount DECIMAL
);
```

**Many-to-Many:**

```sql
-- Students enroll in many courses, courses have many students
CREATE TABLE students (id INT PRIMARY KEY, name VARCHAR(100));
CREATE TABLE courses (id INT PRIMARY KEY, name VARCHAR(100));

-- Junction table
CREATE TABLE enrollments (
  student_id INT REFERENCES students(id),
  course_id INT REFERENCES courses(id),
  PRIMARY KEY (student_id, course_id)
);
```

**One-to-One:**

```sql
-- One user has one profile
CREATE TABLE users (id INT PRIMARY KEY, email VARCHAR(100));
CREATE TABLE profiles (
  user_id INT PRIMARY KEY REFERENCES users(id),
  bio TEXT,
  avatar_url VARCHAR(255)
);
```

---

## 4. Indexing

Indexes speed up queries by creating a sorted data structure for fast lookups.

### B-Tree Index (default)

```sql
CREATE INDEX idx_users_email ON users(email);

-- Without index: Full table scan O(n)
SELECT * FROM users WHERE email = 'alice@example.com';

-- With index: B-tree lookup O(log n)
```

**B-tree structure:**

```
                [M]
              /     \
          [D, H]    [P, T]
         /  |  \    /  |  \
      [A] [E] [J] [N] [Q] [U]
```

**Good for:**
- Equality: `WHERE id = 123`
- Range: `WHERE age BETWEEN 25 AND 35`
- Sorting: `ORDER BY created_at`
- Prefix: `WHERE name LIKE 'Ali%'`

**Not good for:**
- Suffix: `WHERE name LIKE '%son'` (can't use index)
- Functions: `WHERE UPPER(name) = 'ALICE'` (can't use index on name)

---

### Hash Index

```sql
CREATE INDEX idx_users_id_hash ON users USING HASH (id);

-- O(1) lookup for equality
SELECT * FROM users WHERE id = 123;
```

**Good for:** Exact equality only. **Not good for:** Range queries, sorting.

**Rarely used** — B-tree handles equality well and also supports ranges.

---

### Composite Index

Index on multiple columns.

```sql
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- Can use index:
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 1 AND created_at > '2024-01-01';

-- Cannot use index (doesn't start with user_id):
SELECT * FROM orders WHERE created_at > '2024-01-01';
```

**Rule:** Index can be used if query filters on a prefix of the index columns (left-to-right).

---

### Covering Index

Index includes all columns needed by the query.

```sql
CREATE INDEX idx_orders_covering ON orders(user_id, created_at, amount);

-- Query only needs columns in index → no table lookup needed
SELECT user_id, created_at, amount 
FROM orders 
WHERE user_id = 1;
```

**Benefit:** Faster — no need to access table rows, just read from index.

---

### Index Tradeoffs

```
Pros:
  ✅ Faster reads (especially for large tables)
  ✅ Enforce uniqueness (UNIQUE index)

Cons:
  ❌ Slower writes (index must be updated on INSERT/UPDATE/DELETE)
  ❌ Storage overhead (index takes disk space)
  ❌ Too many indexes → query planner confusion
```

**Rule of thumb:** Index columns used in WHERE, JOIN, ORDER BY. Don't over-index.

---

## 5. Query Optimization

### EXPLAIN

Shows query execution plan.

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 1;

Output:
Seq Scan on orders  (cost=0.00..35.50 rows=10 width=32)
  Filter: (user_id = 1)

"Seq Scan" = full table scan (slow)
```

**After adding index:**

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);

EXPLAIN SELECT * FROM orders WHERE user_id = 1;

Output:
Index Scan using idx_orders_user_id on orders  (cost=0.29..8.31 rows=10 width=32)
  Index Cond: (user_id = 1)

"Index Scan" = using index (fast)
```

---

### Common slow query patterns

**1. Missing index:**

```sql
-- Slow: full table scan
SELECT * FROM users WHERE email = 'alice@example.com';

-- Fix: add index
CREATE INDEX idx_users_email ON users(email);
```

**2. Function on indexed column:**

```sql
-- Slow: can't use index on email
SELECT * FROM users WHERE UPPER(email) = 'ALICE@EXAMPLE.COM';

-- Fix: use functional index or store lowercase
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
```

**3. OR conditions:**

```sql
-- Slow: can't efficiently use indexes
SELECT * FROM users WHERE name = 'Alice' OR email = 'alice@example.com';

-- Fix: use UNION
SELECT * FROM users WHERE name = 'Alice'
UNION
SELECT * FROM users WHERE email = 'alice@example.com';
```

**4. SELECT * :**

```sql
-- Slow: fetches all columns
SELECT * FROM users WHERE id = 1;

-- Fast: only fetch needed columns
SELECT id, name, email FROM users WHERE id = 1;
```

---

## 6. Transactions and Locking

### Transaction example

```sql
BEGIN TRANSACTION;

-- Deduct from sender
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

-- Add to receiver
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- Check if sender has sufficient balance
IF (SELECT balance FROM accounts WHERE id = 1) < 0 THEN
  ROLLBACK;  -- undo both updates
ELSE
  COMMIT;    -- make both updates permanent
END IF;
```

---

### Locking

**Pessimistic locking:** Lock rows before reading/writing.

```sql
BEGIN TRANSACTION;

-- Lock row for update
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

-- Other transactions trying to read this row will wait
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

COMMIT;  -- releases lock
```

**Optimistic locking:** Check version before writing.

```sql
-- Add version column
ALTER TABLE accounts ADD COLUMN version INT DEFAULT 0;

-- Read with version
SELECT id, balance, version FROM accounts WHERE id = 1;
-- Returns: id=1, balance=500, version=5

-- Update only if version matches
UPDATE accounts 
SET balance = 400, version = 6 
WHERE id = 1 AND version = 5;

-- If version changed (another transaction updated), UPDATE affects 0 rows
-- Application detects conflict and retries
```

**Use pessimistic when:** High contention, conflicts are common. **Use optimistic when:** Low contention, conflicts are rare.

---

### Deadlocks

Two transactions wait for each other's locks.

```
Transaction A:
  1. Lock row 1
  2. Wait for lock on row 2 (held by B)

Transaction B:
  1. Lock row 2
  2. Wait for lock on row 1 (held by A)

→ Deadlock! Both wait forever.
```

**Database detects deadlock and aborts one transaction.**

**Prevention:**
- Always acquire locks in the same order
- Keep transactions short
- Use appropriate isolation levels

---

## 7. Scaling SQL Databases

### Vertical scaling

Bigger server (more CPU, RAM, faster SSD).

```
Pros: Simple, no code changes
Cons: Expensive, has a ceiling (largest server)
```

**Good first step.** Modern servers can handle 10M+ rows easily.

---

### Read replicas

Route reads to replicas, writes to primary.

```
         ┌→ Replica 1 (reads)
Primary ─┼→ Replica 2 (reads)
(writes) └→ Replica 3 (reads)
```

**Pros:** Scales read throughput. **Cons:** Replication lag (replicas may be stale), doesn't scale writes.

**Use for:** Read-heavy workloads (90%+ reads).

---

### Connection pooling

Reuse database connections instead of creating new ones.

```
Without pooling:
  Each request → new connection → expensive

With pooling (PgBouncer, ProxySQL):
  Pool of 100 connections shared by 1000 requests
  Requests wait for available connection
```

**Benefit:** Reduces connection overhead, prevents connection exhaustion.

---

### Sharding

Split data across multiple databases.

```
Users 0-1M    → Shard 1
Users 1M-2M   → Shard 2
Users 2M-3M   → Shard 3
```

**Pros:** Scales writes and storage. **Cons:** Complex, cross-shard joins don't work, application must route queries.

**Last resort** — only when other options exhausted.

---

## 8. PostgreSQL vs MySQL

|Feature|PostgreSQL|MySQL|
|---|---|---|
|ACID compliance|Full|Full (InnoDB)|
|Concurrency|MVCC (better)|MVCC (InnoDB)|
|JSON support|Excellent (JSONB)|Good (JSON)|
|Full-text search|Built-in|Limited|
|Geospatial|PostGIS (excellent)|Spatial extensions|
|Replication|Streaming, logical|Binlog, Group Replication|
|Window functions|Yes|Yes (8.0+)|
|CTEs (WITH)|Yes|Yes (8.0+)|
|Licensing|PostgreSQL (permissive)|GPL (or commercial)|
|Ecosystem|Rich extensions|Large community|

**Choose PostgreSQL when:** You need advanced features (JSONB, full-text search, PostGIS), complex queries, or strong standards compliance.

**Choose MySQL when:** You need simplicity, have existing MySQL expertise, or use tools that require MySQL.

**In practice:** PostgreSQL is the default choice for new projects.

---

## 9. Common Interview Questions + Answers

### Q: What are ACID properties and why do they matter?

> "ACID ensures reliable transactions. Atomicity means all-or-nothing — a money transfer either completes fully or not at all, preventing partial state. Consistency enforces constraints — you can't have negative balances. Isolation prevents concurrent transactions from interfering — two users can't double-book the same seat. Durability guarantees committed data survives crashes. These properties are critical for financial systems, e-commerce, and any application where data correctness is non-negotiable."

### Q: How do indexes improve query performance?

> "Indexes create a sorted data structure (usually B-tree) that allows O(log n) lookups instead of O(n) table scans. For a table with 1 million rows, an index reduces lookups from 1 million comparisons to about 20. The tradeoff is slower writes — every INSERT/UPDATE/DELETE must also update the index — and storage overhead. You should index columns used in WHERE clauses, JOIN conditions, and ORDER BY, but avoid over-indexing."

### Q: When would you use a SQL database vs NoSQL?

> "Use SQL when you have relational data with complex joins, need ACID transactions, or require ad-hoc queries. Examples: user accounts, orders, inventory, financial transactions. Use NoSQL when you have flexible schemas, need horizontal write scaling beyond what a single SQL node can handle, or have simple access patterns. Examples: product catalogs with varying attributes, time-series logs, user activity feeds. Many systems use both — SQL for transactional data, NoSQL for high-volume logs or caching."

### Q: How do you scale a SQL database?

> "Start with vertical scaling — modern servers handle 10M+ rows easily. Add read replicas for read-heavy workloads — route reads to replicas, writes to primary. Implement connection pooling to reduce connection overhead. Add caching (Redis) in front to reduce database load. Optimize queries and add indexes. Only shard as a last resort — it adds enormous complexity. For write-heavy workloads that outgrow a single node, consider NewSQL (CockroachDB, Spanner) or migrating write-heavy tables to NoSQL."

---

## 10. Interview Tricks & Pitfalls

### ✅ Trick 1: Mention specific databases

Don't just say "SQL database." Say:

> "I'd use PostgreSQL because it has excellent JSONB support for flexible product attributes, built-in full-text search for the product catalog, and PostGIS for geospatial queries on store locations."

### ✅ Trick 2: Discuss indexing proactively

When designing a schema, immediately mention indexes:

> "I'd add an index on user_id in the orders table since we'll frequently query orders by user. I'd also add a composite index on (user_id, created_at) to support queries that filter by user and sort by date."

### ✅ Trick 3: Know the scaling path

Show you understand the progression:

> "We'd start with a single PostgreSQL instance with vertical scaling. At 10K QPS, add read replicas. At 50K QPS, add Redis caching. Only if we exceed 100K writes/sec would we consider sharding, and even then, I'd look at CockroachDB first."

### ❌ Pitfall 1: Saying "SQL doesn't scale"

SQL scales very well vertically and with read replicas. Don't dismiss it prematurely.

### ❌ Pitfall 2: Forgetting about transactions

If you propose SQL, be ready to discuss transactions:

> "For the payment flow, I need a transaction that atomically deducts from the user's balance and creates an order record. If either fails, both roll back."

### ❌ Pitfall 3: Not knowing isolation levels

If you mention transactions, the interviewer may ask about isolation levels. Know the four levels and their tradeoffs.

---

## 11. Quick Reference

```
SQL Database = relational, schema-enforced, ACID transactions

ACID:
  Atomicity:    All-or-nothing transactions
  Consistency:  Constraints always satisfied
  Isolation:    Concurrent transactions don't interfere
  Durability:   Committed data survives crashes

Schema:
  Normalization:  Reduce redundancy (3NF is common target)
  Relationships:  One-to-many, many-to-many, one-to-one
  Foreign keys:   Enforce referential integrity

Indexing:
  B-tree:      Default, good for equality and ranges
  Hash:        Equality only, rarely used
  Composite:   Multiple columns, use left-to-right
  Covering:    Includes all query columns, no table lookup

Query optimization:
  Use EXPLAIN to see execution plan
  Index WHERE, JOIN, ORDER BY columns
  Avoid functions on indexed columns
  SELECT only needed columns

Transactions:
  BEGIN, COMMIT, ROLLBACK
  Pessimistic locking: FOR UPDATE
  Optimistic locking: version column
  Deadlock prevention: lock in same order

Scaling:
  1. Vertical scaling (bigger server)
  2. Read replicas (scale reads)
  3. Connection pooling (reduce overhead)
  4. Caching (Redis in front)
  5. Sharding (last resort)

PostgreSQL vs MySQL:
  PostgreSQL: Advanced features, JSONB, full-text, PostGIS
  MySQL: Simplicity, large community
  Default choice: PostgreSQL for new projects
```
