## 1. What Is an Index?

An index is a data structure that improves the speed of data retrieval operations on a database table.

```
Without index:
  SELECT * FROM users WHERE email = 'alice@example.com';
  → Full table scan: check every row (O(n))
  → 1 million rows = 1 million comparisons

With index on email:
  → B-tree lookup: O(log n)
  → 1 million rows = ~20 comparisons
```

**Analogy:** Like a book index. Instead of reading every page to find "databases," you look in the index, which tells you "page 42."

---

## 2. B-Tree Index (Most Common)

The default index type in most databases. A balanced tree structure.

### Structure

```
                    [M, T]
                   /   |   \
              [D, H]  [P]  [W, Z]
             /  |  \   |   /  |  \
          [A] [E] [J][N] [U] [X] [...]
           ↓   ↓   ↓  ↓   ↓   ↓
        [rows][rows][rows]...
```

Each node contains:
- Keys (sorted)
- Pointers to child nodes or data rows

**Properties:**
- Balanced (all leaf nodes at same depth)
- Sorted (enables range queries)
- Self-balancing (maintains O(log n) on inserts/deletes)

---

### What B-Tree indexes are good for

**1. Equality lookups:**

```sql
SELECT * FROM users WHERE id = 123;
-- O(log n) lookup
```

**2. Range queries:**

```sql
SELECT * FROM orders WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';
-- Scan sorted range in index
```

**3. Sorting:**

```sql
SELECT * FROM users ORDER BY created_at DESC LIMIT 10;
-- Index already sorted, just read first 10
```

**4. Prefix matching:**

```sql
SELECT * FROM users WHERE name LIKE 'Ali%';
-- Can use index (starts with known prefix)
```

---

### What B-Tree indexes are NOT good for

**1. Suffix or infix matching:**

```sql
SELECT * FROM users WHERE name LIKE '%son';
-- Can't use index (unknown prefix)
```

**2. Functions on indexed column:**

```sql
SELECT * FROM users WHERE UPPER(email) = 'ALICE@EXAMPLE.COM';
-- Can't use index on email (function applied)
```

**3. OR conditions across different columns:**

```sql
SELECT * FROM users WHERE name = 'Alice' OR age = 25;
-- Can't efficiently use indexes on both columns
```

---

## 3. Hash Index

Uses a hash table for O(1) lookups.

```
hash("alice@example.com") = 42 → row pointer
hash("bob@example.com")   = 17 → row pointer
```

**Good for:**
- Exact equality: `WHERE email = 'alice@example.com'`

**Not good for:**
- Range queries: `WHERE age > 25` (hash destroys ordering)
- Sorting: `ORDER BY email` (no order in hash table)
- Prefix matching: `WHERE email LIKE 'alice%'`

**Rarely used in practice** — B-tree handles equality well and also supports ranges.

**Exception:** PostgreSQL hash indexes for very large tables with only equality queries.

---

## 4. Composite (Multi-Column) Index

Index on multiple columns.

```sql
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);
```

**Index structure:**

```
(user_1, 2024-01-01)
(user_1, 2024-01-02)
(user_1, 2024-01-03)
(user_2, 2024-01-01)
(user_2, 2024-01-02)
```

Sorted first by user_id, then by created_at within each user.

---

### Leftmost prefix rule

Index can be used if query filters on a **left prefix** of the index columns.

```sql
-- Index: (user_id, created_at)

-- ✅ Can use index (starts with user_id)
SELECT * FROM orders WHERE user_id = 1;

-- ✅ Can use index (both columns)
SELECT * FROM orders WHERE user_id = 1 AND created_at > '2024-01-01';

-- ❌ Cannot use index (doesn't start with user_id)
SELECT * FROM orders WHERE created_at > '2024-01-01';
```

**Rule:** Index columns must be used left-to-right, no gaps.

---

### Column order matters

```sql
-- Index A: (user_id, created_at)
-- Index B: (created_at, user_id)

-- Query 1: WHERE user_id = 1 AND created_at > '2024-01-01'
-- Index A: ✅ Perfect
-- Index B: ❌ Can't use efficiently (created_at is first, but we filter user_id)

-- Query 2: WHERE created_at > '2024-01-01' AND user_id = 1
-- Index A: ✅ Perfect (order in WHERE doesn't matter, index order does)
-- Index B: ❌ Can't use efficiently
```

**Design rule:** Put the most selective (filters to fewest rows) or most frequently queried column first.

---

## 5. Covering Index

An index that includes all columns needed by a query.

```sql
-- Query
SELECT user_id, created_at, amount 
FROM orders 
WHERE user_id = 1;

-- Regular index: (user_id)
-- Process: Index lookup → find row IDs → fetch rows from table

-- Covering index: (user_id, created_at, amount)
-- Process: Index lookup → all data in index, no table access needed
```

**Benefit:** Faster — no need to access table rows ("index-only scan").

**Tradeoff:** Larger index (stores more data), slower writes.

---

## 6. Partial Index

Index only a subset of rows.

```sql
-- Only index active users
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';

-- Query
SELECT * FROM users WHERE email = 'alice@example.com' AND status = 'active';
-- Uses partial index (smaller, faster)
```

**Benefits:**
- Smaller index (less storage, faster)
- Faster writes (fewer rows to index)

**Use when:** You frequently query a subset of data (active users, recent orders, published posts).

---

## 7. Functional Index

Index on an expression or function result.

```sql
-- Index on lowercase email
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Query can now use index
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
```

**Use when:** You frequently query with a function applied to a column.

---

## 8. Full-Text Index

Specialized index for text search.

```sql
-- PostgreSQL
CREATE INDEX idx_posts_content_fts ON posts USING GIN(to_tsvector('english', content));

-- Search
SELECT * FROM posts 
WHERE to_tsvector('english', content) @@ to_tsquery('database & performance');
```

**Features:**
- Tokenization (split text into words)
- Stemming (run → running → ran)
- Stop words (ignore "the", "a", "is")
- Relevance ranking

**Use for:** Search bars, content search, log analysis.

**Better alternatives for complex search:** Elasticsearch, Algolia.

---

## 9. Index Tradeoffs

### Pros

```
✅ Faster reads (O(log n) vs O(n))
✅ Enforce uniqueness (UNIQUE index)
✅ Speed up joins (index on foreign keys)
✅ Speed up sorting (ORDER BY)
```

### Cons

```
❌ Slower writes (index must be updated on INSERT/UPDATE/DELETE)
❌ Storage overhead (index takes disk space)
❌ Maintenance overhead (indexes need rebuilding, vacuuming)
❌ Too many indexes → query planner confusion
```

---

### Write penalty

```
Without index:
  INSERT → write 1 row

With 3 indexes:
  INSERT → write 1 row + update 3 indexes
  → 4x write operations
```

**Rule of thumb:** Index columns used in WHERE, JOIN, ORDER BY. Don't over-index.

---

## 10. When to Add an Index

### Symptoms of missing index

```
1. Slow queries (check with EXPLAIN)
2. Full table scans in EXPLAIN output
3. High CPU usage on database
4. Queries timing out
```

### Decision framework

```
Add index if:
  ✅ Column used in WHERE clause frequently
  ✅ Column used in JOIN condition
  ✅ Column used in ORDER BY
  ✅ Table is large (> 10K rows)
  ✅ Query is slow (> 100ms)

Don't add index if:
  ❌ Table is small (< 1K rows) — full scan is fast
  ❌ Column has low cardinality (few distinct values)
  ❌ Column is rarely queried
  ❌ Write-heavy workload (index slows writes)
```

---

## 11. Using EXPLAIN to Analyze Queries

### PostgreSQL EXPLAIN

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 1;

Output:
Seq Scan on orders  (cost=0.00..35.50 rows=10 width=32)
  Filter: (user_id = 1)
```

**"Seq Scan" = sequential scan = full table scan = slow.**

---

### After adding index

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);

EXPLAIN SELECT * FROM orders WHERE user_id = 1;

Output:
Index Scan using idx_orders_user_id on orders  (cost=0.29..8.31 rows=10 width=32)
  Index Cond: (user_id = 1)
```

**"Index Scan" = using index = fast.**

---

### EXPLAIN ANALYZE

Shows actual execution time, not just plan.

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1;

Output:
Index Scan using idx_orders_user_id on orders  
  (cost=0.29..8.31 rows=10 width=32) 
  (actual time=0.015..0.018 rows=10 loops=1)
Planning Time: 0.123 ms
Execution Time: 0.045 ms
```

**Use EXPLAIN ANALYZE to find slow queries.**

---

## 12. Index Maintenance

### Fragmentation

Over time, indexes become fragmented (not physically contiguous on disk).

```
Fresh index:  [A][B][C][D][E][F]  (contiguous)
After updates: [A]...[B].....[C].[D]...[E]  (fragmented)
```

**Impact:** Slower reads (more disk seeks).

**Fix:** Rebuild index.

```sql
-- PostgreSQL
REINDEX INDEX idx_orders_user_id;

-- MySQL
ALTER TABLE orders DROP INDEX idx_orders_user_id, 
                   ADD INDEX idx_orders_user_id(user_id);
```

---

### Statistics

Database maintains statistics about data distribution to help query planner.

```
Table: users
Column: age
Statistics:
  Min: 18
  Max: 90
  Distinct values: 72
  Most common values: 25 (5%), 30 (4%), 35 (3%)
```

**Stale statistics → bad query plans.**

**Fix:** Update statistics.

```sql
-- PostgreSQL
ANALYZE users;

-- MySQL
ANALYZE TABLE users;
```

**Run after bulk inserts/updates.**

---

## 13. Common Indexing Mistakes

### Mistake 1: Indexing low-cardinality columns

```sql
-- Bad: gender has only 2-3 values
CREATE INDEX idx_users_gender ON users(gender);

-- Query
SELECT * FROM users WHERE gender = 'female';
-- Returns 50% of rows → index not helpful, full scan is faster
```

**Rule:** Don't index columns with few distinct values (< 5% selectivity).

---

### Mistake 2: Too many indexes

```sql
-- Overkill: 10 indexes on one table
CREATE INDEX idx1 ON users(email);
CREATE INDEX idx2 ON users(name);
CREATE INDEX idx3 ON users(age);
CREATE INDEX idx4 ON users(created_at);
-- ... 6 more

-- Problem: Every INSERT updates 10 indexes → very slow writes
```

**Rule:** Start with 2-3 indexes, add more only if needed.

---

### Mistake 3: Redundant indexes

```sql
-- Redundant
CREATE INDEX idx1 ON orders(user_id);
CREATE INDEX idx2 ON orders(user_id, created_at);

-- idx1 is redundant — idx2 can handle queries on just user_id
-- (leftmost prefix rule)
```

**Rule:** Remove redundant indexes.

---

### Mistake 4: Not indexing foreign keys

```sql
-- orders.user_id references users.id

-- Without index on user_id:
SELECT * FROM orders WHERE user_id = 1;  -- slow

-- Add index
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

**Rule:** Always index foreign key columns.

---

## 14. Advanced Indexing Techniques

### Index-only scans

Query satisfied entirely from index, no table access.

```sql
-- Index: (user_id, created_at, amount)

-- Query
SELECT user_id, created_at, amount 
FROM orders 
WHERE user_id = 1;

-- All columns in index → index-only scan → very fast
```

---

### Index intersection

Database combines multiple indexes.

```sql
-- Indexes: idx_age, idx_city

-- Query
SELECT * FROM users WHERE age = 25 AND city = 'New York';

-- Database can:
-- 1. Use idx_age to find users with age=25
-- 2. Use idx_city to find users in New York
-- 3. Intersect the two result sets

-- Or: create composite index (age, city) for better performance
```

---

### Clustered vs non-clustered index

**Clustered index (InnoDB primary key):**
- Table data physically sorted by index
- One per table (usually primary key)
- Fastest for range scans on primary key

**Non-clustered index (secondary indexes):**
- Separate structure pointing to table rows
- Multiple per table
- Requires extra lookup to fetch row data

---

## 15. Common Interview Questions + Answers

### Q: How do indexes improve query performance?

> "Indexes create a sorted data structure, usually a B-tree, that allows O(log n) lookups instead of O(n) table scans. For a table with 1 million rows, an index reduces lookups from 1 million comparisons to about 20. The index stores pointers to the actual rows, so once you find the key in the index, you can jump directly to the row. The tradeoff is slower writes — every INSERT/UPDATE/DELETE must also update the index — and storage overhead."

### Q: What's a composite index and when would you use it?

> "A composite index is an index on multiple columns, like (user_id, created_at). It's useful when you frequently query on multiple columns together. The leftmost prefix rule applies — the index can be used if your query filters on a left prefix of the columns. So an index on (user_id, created_at) can handle queries on just user_id, or on both user_id and created_at, but not on just created_at. Column order matters — put the most selective or most frequently queried column first."

### Q: When should you NOT add an index?

> "Don't index small tables (< 1K rows) where a full scan is already fast. Don't index low-cardinality columns like gender or status with only 2-3 values — the index won't be selective enough. Don't over-index write-heavy tables — each index slows down inserts and updates. And don't add indexes speculatively — add them based on actual slow queries identified with EXPLAIN."

### Q: What's the difference between a covering index and a regular index?

> "A covering index includes all columns needed by a query, not just the columns in the WHERE clause. For example, if your query is `SELECT user_id, amount FROM orders WHERE user_id = 1`, a covering index on (user_id, amount) means the database can satisfy the entire query from the index without accessing the table rows. This is called an index-only scan and is much faster. The tradeoff is a larger index and slower writes."

---

## 16. Interview Tricks & Pitfalls

### ✅ Trick 1: Always mention EXPLAIN

When discussing slow queries:

> "I'd use EXPLAIN to see the query plan. If it shows a sequential scan, that means no index is being used and we need to add one. After adding the index, I'd run EXPLAIN again to verify it's using an index scan."

### ✅ Trick 2: Discuss the write tradeoff

Don't just say "indexes make queries faster":

> "Indexes speed up reads but slow down writes. Each index must be updated on every INSERT/UPDATE/DELETE. For a write-heavy table, I'd be conservative with indexes — only add them for the most critical queries."

### ✅ Trick 3: Mention composite indexes for common query patterns

Shows you think about real-world usage:

> "For the orders table, I'd create a composite index on (user_id, created_at) since we frequently query orders by user and sort by date. This single index handles both the filter and the sort."

### ❌ Pitfall 1: Saying "index everything"

Over-indexing is a real problem. Show you understand the tradeoffs.

### ❌ Pitfall 2: Not knowing the leftmost prefix rule

If you propose a composite index, be ready to explain which queries can use it.

### ❌ Pitfall 3: Forgetting about index maintenance

Indexes need rebuilding and statistics updates. Mention this for operational maturity.

---

## 17. Quick Reference

```
Index = data structure for fast lookups

B-Tree (default):
  - Balanced tree, sorted
  - O(log n) lookups
  - Good for: equality, ranges, sorting, prefix matching
  - Not good for: suffix matching, functions on column

Hash:
  - O(1) equality lookups
  - Not good for: ranges, sorting
  - Rarely used (B-tree handles equality well)

Composite index:
  - Multiple columns: (user_id, created_at)
  - Leftmost prefix rule: can use if query filters left-to-right
  - Column order matters: most selective first

Covering index:
  - Includes all query columns
  - Index-only scan (no table access)
  - Faster but larger

Partial index:
  - Index subset of rows: WHERE status = 'active'
  - Smaller, faster for common queries

Functional index:
  - Index on expression: LOWER(email)
  - Use when querying with function

Tradeoffs:
  ✅ Faster reads (O(log n) vs O(n))
  ❌ Slower writes (must update index)
  ❌ Storage overhead

When to index:
  ✅ WHERE, JOIN, ORDER BY columns
  ✅ Large tables (> 10K rows)
  ✅ High-cardinality columns
  ❌ Small tables (< 1K rows)
  ❌ Low-cardinality columns (< 5% selectivity)
  ❌ Write-heavy workloads

Tools:
  EXPLAIN: show query plan
  EXPLAIN ANALYZE: show actual execution time
  REINDEX: rebuild fragmented index
  ANALYZE: update statistics

Common mistakes:
  - Indexing low-cardinality columns
  - Too many indexes
  - Redundant indexes
  - Not indexing foreign keys
```
