
This note covers core database topics commonly asked in interviews: schema design, indexing, transactions (ACID), and normalization.

---

## 1) Database design fundamentals

Goal:
- Model data so it is **correct**, **queryable**, **scalable**, and **maintainable**.

Design flow (interview-friendly):
1. Identify entities (tables) and relationships.
2. Define keys and constraints.
3. Choose data types carefully.
4. Design for read/write access patterns.
5. Add indexes for critical queries.
6. Validate with real queries and edge cases.

---

## 2) Core building blocks

### Tables and columns

- Table = entity (e.g., `users`, `orders`)
- Column = attribute (e.g., `email`, `status`, `created_at`)

### Keys

- **Primary key (PK)**: uniquely identifies each row.
- **Foreign key (FK)**: links related tables.
- **Candidate key**: any minimal unique key; one becomes PK.
- **Composite key**: key built from multiple columns.

### Constraints (data integrity)

- `NOT NULL`: required value
- `UNIQUE`: no duplicates
- `CHECK`: value rule (e.g., amount >= 0)
- `FOREIGN KEY`: relationship consistency

Example:

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP NOT NULL
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    amount DECIMAL(10,2) NOT NULL CHECK (amount >= 0),
    status VARCHAR(30) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    CONSTRAINT fk_orders_user FOREIGN KEY (user_id) REFERENCES users(id)
);
```

---

## 3) Relationships (schema modeling)

### One-to-one (1:1)

- Example: `users` and `user_profiles`
- Usually enforced by unique FK.

### One-to-many (1:N)

- Example: one user has many orders.
- FK lives on "many" side (`orders.user_id`).

### Many-to-many (M:N)

- Use join table.

```sql
CREATE TABLE students (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE courses (
    id BIGINT PRIMARY KEY,
    title VARCHAR(200) NOT NULL
);

CREATE TABLE student_courses (
    student_id BIGINT NOT NULL,
    course_id BIGINT NOT NULL,
    enrolled_at TIMESTAMP NOT NULL,
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (student_id) REFERENCES students(id),
    FOREIGN KEY (course_id) REFERENCES courses(id)
);
```

---

## 4) Indexing basics

Index is a data structure (commonly B-tree) that speeds up lookup/sort/filter, at the cost of extra storage and write overhead.

### Why indexes help

Without index:
- DB may scan full table (`O(n)` behavior).

With index:
- DB can find matching rows much faster (often logarithmic search + selective reads).

### Common index types (interview level)

- **Primary index**: on PK.
- **Unique index**: enforces uniqueness.
- **Single-column index**: one column.
- **Composite index**: multiple columns in defined order.
- **Covering index**: contains all columns needed by query (can avoid table lookup).

### Composite index rule (leftmost prefix)

For index `(user_id, status, created_at)`:
- Good: filters by `user_id`, or `user_id + status`
- Not good: only `status` without `user_id` (in most engines)

### Example

```sql
-- frequent query:
-- SELECT * FROM orders WHERE user_id = ? AND status = ? ORDER BY created_at DESC;

CREATE INDEX idx_orders_user_status_created
ON orders(user_id, status, created_at DESC);
```

### Indexing best practices

- Index high-value read paths, not every column.
- Avoid indexing very low-cardinality columns alone (e.g., boolean only).
- Keep indexes aligned with actual `WHERE`, `JOIN`, `ORDER BY`.
- Too many indexes slow down `INSERT/UPDATE/DELETE`.
- Use `EXPLAIN` to verify query plan.

---

## 5) Transactions and ACID

Transaction = logical unit of work that should succeed or fail as one unit.

### ACID properties

### Atomicity
- All operations in transaction happen, or none happen.

### Consistency
- Transaction moves DB from one valid state to another (constraints preserved).

### Isolation
- Concurrent transactions should not interfere incorrectly.

### Durability
- Once committed, data survives crashes (via logs/checkpoints, engine-specific).

### Example: money transfer

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;
```

If second update fails, rollback should undo first update.

### Common anomalies (isolation problems)

- **Dirty read**: read uncommitted data.
- **Non-repeatable read**: same row read twice gives different values.
- **Phantom read**: repeated query returns different row set.

### Isolation levels (high-level)

- **Read Uncommitted**: weakest, highest anomalies.
- **Read Committed**: prevents dirty reads.
- **Repeatable Read**: stable row reads in transaction.
- **Serializable**: strongest correctness, lowest concurrency.

Interview trade-off:
- Higher isolation -> safer but potentially lower throughput/greater lock contention.

---

## 6) Normalization

Normalization organizes schema to reduce redundancy and update anomalies.

### Why it matters

Without normalization:
- duplicate data
- inconsistent updates
- insert/delete anomalies

### 1NF (First Normal Form)

- Atomic column values (no repeating groups/arrays in a relational row).

### 2NF (Second Normal Form)

- In 1NF, and non-key columns depend on full primary key (important for composite PK tables).

### 3NF (Third Normal Form)

- In 2NF, and non-key columns depend only on key, not on other non-key columns (no transitive dependency).

### Example (denormalized -> normalized)

Denormalized order table:

```text
order_id, user_id, user_name, user_email, amount
```

Issue:
- user details repeated in every order row.

Normalized split:
- `users(id, name, email)`
- `orders(id, user_id, amount, ...)`

Benefits:
- less duplication
- easier updates
- better integrity

### Denormalization (when and why)

Sometimes you intentionally denormalize for read performance:
- precomputed aggregates
- duplicate display fields
- reporting tables/materialized views

Do this intentionally with clear ownership and update strategy.

---

## 7) Interview-ready checklist for DB design

- Define entities + cardinality (1:1, 1:N, M:N).
- Choose PK/FK and constraints.
- Show 2-3 key queries and index for them.
- Explain transaction boundaries and failure handling.
- Mention isolation level decision and trade-off.
- Normalize to 3NF by default, denormalize only with reason.
- Cover scale concerns briefly: partitioning/sharding/caching (if asked).

---

## 8) Quick interview Q&A

### Q: Should every foreign key be indexed?
- Usually yes for join/filter performance and lock efficiency, especially on large tables.

### Q: Why not create indexes on all columns?
- Reads may improve, but writes become slower and storage usage increases.

### Q: Is normalization always best?
- For correctness/maintainability, yes by default. For heavy reads, partial denormalization can help.

### Q: ACID in one line?
- Reliable transaction guarantees: all-or-nothing, valid state, controlled concurrency, durable commit.

---

## Summary

- Good schema design starts from entities, relationships, keys, and constraints.
- Indexes speed reads but add write/storage cost; design them around real queries.
- Transactions enforce correctness; ACID is the foundation.
- Normalize to reduce redundancy and anomalies; denormalize deliberately for performance.
