# Database Design Concepts

## Entity-Relationship Modeling

### Entities and Attributes
- **Entity**: A real-world object with independent existence (User, Order, Product)
- **Attribute**: A property of an entity (User.email, Order.total)
- **Primary Key**: Uniquely identifies each row — prefer surrogate keys (UUID or auto-increment integer)

### Relationships
| Type | Example | Implementation |
|------|---------|---------------|
| One-to-One | User → Profile | Foreign key in either table |
| One-to-Many | User → Orders | Foreign key in the "many" table |
| Many-to-Many | Orders ↔ Products | Junction table (order_items) |

## Normalization

### First Normal Form (1NF)
- Each column contains atomic (indivisible) values
- No repeating groups or arrays in columns

### Second Normal Form (2NF)
- Satisfies 1NF
- Every non-key attribute is fully dependent on the entire primary key (no partial dependencies)

### Third Normal Form (3NF)
- Satisfies 2NF
- No transitive dependencies (non-key attributes depend only on the primary key, not on other non-key attributes)

### Boyce-Codd Normal Form (BCNF)
- Stronger version of 3NF — every determinant is a candidate key

### When to Denormalize
- Read-heavy workloads where joins are expensive
- Pre-computed aggregates (total_orders on User table)
- Caching frequently accessed data inline
- Trade-off: faster reads, slower writes, data duplication risk

## Indexing

### B-Tree Index (default)
- Supports equality and range queries: `WHERE id = 5`, `WHERE created_at > '2024-01-01'`
- Ordered — supports ORDER BY efficiently
- Good for most use cases

### Hash Index
- Supports equality only: `WHERE id = 5`
- Faster than B-tree for exact lookups
- Cannot support range queries or sorting

### Composite Index
- Index on multiple columns: `(user_id, created_at)`
- Column order matters — leftmost prefix rule
- `WHERE user_id = 1 AND created_at > '2024-01-01'` uses the index
- `WHERE created_at > '2024-01-01'` alone does NOT use the index

### Covering Index
- Index includes all columns needed by a query — no table lookup required
- Example: `CREATE INDEX idx ON orders (user_id) INCLUDE (status, total)`

### When to Add Indexes
- Columns used in WHERE clauses frequently
- Foreign key columns (for join performance)
- Columns used in ORDER BY or GROUP BY
- Avoid over-indexing — each index slows down writes

## Query Optimization

- **EXPLAIN / EXPLAIN ANALYZE**: Shows query execution plan — look for sequential scans on large tables
- **N+1 Problem**: Loading a list then querying each item individually — use JOINs or batch loading
- **Pagination**: Use cursor-based pagination for large tables; avoid `OFFSET` on large values
- **Connection Pooling**: Reuse DB connections — avoid opening a new connection per request

## SQL vs NoSQL Selection

### Use SQL (Relational) When:
- Data has clear relationships and structure
- ACID transactions are required (financial data, inventory)
- Complex queries with JOINs are needed
- Schema is relatively stable

### Use NoSQL When:
- Schema is flexible or evolving rapidly
- Horizontal scaling is a primary concern
- Data access patterns are simple (key-value, document lookup)
- High write throughput with eventual consistency is acceptable

### NoSQL Types
| Type | Examples | Best For |
|------|---------|---------|
| Document | MongoDB, DynamoDB | Flexible schemas, nested data |
| Key-Value | Redis, DynamoDB | Caching, sessions, simple lookups |
| Column-Family | Cassandra, HBase | Time-series, write-heavy workloads |
| Graph | Neo4j, Neptune | Relationship-heavy queries |

## Sharding and Partitioning

### Horizontal Partitioning (Sharding)
- Split rows across multiple DB instances by a shard key
- Shard key choice is critical — avoid hot spots
- Good shard keys: user_id (even distribution), geographic region
- Bad shard keys: timestamp (all writes go to latest shard), status (low cardinality)

### Vertical Partitioning
- Split columns across tables — move rarely-accessed columns to a separate table
- Example: Move `user.profile_picture_blob` to a separate `user_profiles` table

### Table Partitioning (within one DB)
- Partition a large table by range (date), list (region), or hash
- Improves query performance by pruning irrelevant partitions
- Useful for time-series data with retention policies

## ACID vs BASE

### ACID (Relational DBs)
- **Atomicity**: All operations in a transaction succeed or all fail
- **Consistency**: DB moves from one valid state to another
- **Isolation**: Concurrent transactions don't interfere
- **Durability**: Committed transactions survive failures

### BASE (NoSQL DBs)
- **Basically Available**: System remains available even during failures
- **Soft State**: State may change over time without input (eventual consistency)
- **Eventually Consistent**: System will become consistent given enough time

## Schema Evolution

- **Additive changes** (new nullable columns, new tables) are safe — no migration needed for existing data
- **Breaking changes** (rename column, change type, drop column) require careful migration:
  1. Add new column
  2. Dual-write to old and new columns
  3. Backfill old data
  4. Switch reads to new column
  5. Drop old column
- Use migration tools (Flyway, Liquibase) to version schema changes
