## 1. What is a Column-Family Store?

A column-family store (also called wide-column store) is a NoSQL database that stores data in column families rather than rows. Data is organized by columns, not rows, optimizing for write-heavy workloads and time-series data.

```
Key examples:
  - Apache Cassandra
  - Apache HBase
  - Google Bigtable
  - ScyllaDB
```

**Core concept:** Instead of storing a row with all columns together, store each column family separately, allowing efficient writes and reads of specific columns without loading entire rows.

---

## 2. Data Model

### Traditional row-oriented database (SQL)

```
users table:
┌────────┬──────────┬───────────────────┬─────────┐
│ user_id│   name   │      email        │  age    │
├────────┼──────────┼───────────────────┼─────────┤
│   1    │  Alice   │ alice@example.com │   30    │
│   2    │  Bob     │ bob@example.com   │   25    │
└────────┴──────────┴───────────────────┴─────────┘

Stored on disk row-by-row:
  [1, Alice, alice@example.com, 30]
  [2, Bob, bob@example.com, 25]
```

To read just the `name` column, you must read the entire row.

### Column-family store

```
users column family:
  Row key: user_id
  Columns: name, email, age

Stored on disk column-by-column:
  user_id: [1, 2]
  name:    [Alice, Bob]
  email:   [alice@example.com, bob@example.com]
  age:     [30, 25]
```

To read just the `name` column, you only read the `name` column data.

### Cassandra data model

```
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name TEXT,
    email TEXT,
    age INT
);

Logical view:
┌──────────────────────────────────────┬───────┬───────────────────┬─────┐
│              user_id                 │ name  │      email        │ age │
├──────────────────────────────────────┼───────┼───────────────────┼─────┤
│ 550e8400-e29b-41d4-a716-446655440000 │ Alice │ alice@example.com │ 30  │
└──────────────────────────────────────┴───────┴───────────────────┴─────┘

Physical storage (simplified):
  Partition key: user_id
  Columns: {name: Alice, email: alice@example.com, age: 30}
```

---

## 3. Column Families

A column family is a container for rows, similar to a table in SQL. But unlike SQL tables, each row can have different columns.

### Static columns (like SQL)

```
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name TEXT,
    email TEXT
);

Every row has the same columns: user_id, name, email
```

### Dynamic columns (wide rows)

```
CREATE TABLE user_events (
    user_id UUID,
    event_time TIMESTAMP,
    event_type TEXT,
    event_data TEXT,
    PRIMARY KEY (user_id, event_time)
);

Row for user_id = 123:
  event_time: 2024-01-01 10:00:00 → {event_type: login, event_data: ...}
  event_time: 2024-01-01 10:05:00 → {event_type: click, event_data: ...}
  event_time: 2024-01-01 10:10:00 → {event_type: logout, event_data: ...}

Each row can have millions of columns (time-series data)
```

This is called a "wide row" — a single partition key with many clustering columns.

---

## 4. Primary Key Structure

Cassandra's primary key has two parts: partition key and clustering columns.

```
PRIMARY KEY (partition_key, clustering_column1, clustering_column2, ...)
```

### Partition key

Determines which node stores the data (via consistent hashing).

```
PRIMARY KEY (user_id)

user_id = 123 → hash(123) → node 2
user_id = 456 → hash(456) → node 5
```

All data for a partition key is stored on the same node.

### Clustering columns

Determine the sort order within a partition.

```
PRIMARY KEY (user_id, event_time)

Partition: user_id = 123
  Sorted by event_time:
    2024-01-01 10:00:00 → {event_type: login}
    2024-01-01 10:05:00 → {event_type: click}
    2024-01-01 10:10:00 → {event_type: logout}
```

Queries can efficiently read a range of clustering columns:

```sql
SELECT * FROM user_events
WHERE user_id = 123
  AND event_time >= '2024-01-01 10:00:00'
  AND event_time < '2024-01-01 11:00:00';
```

This reads a contiguous block of data on disk (very fast).

---

## 5. Write Path

Column-family stores are optimized for high write throughput.

### Cassandra write path

```
1. Write to commit log (append-only, sequential write)
   → Durability guarantee

2. Write to memtable (in-memory sorted structure)
   → Fast write acknowledgment

3. Return success to client

4. (Background) When memtable is full:
   → Flush to SSTable (sorted string table) on disk
   → Immutable, sorted by partition key + clustering columns

5. (Background) Compaction:
   → Merge multiple SSTables into one
   → Remove deleted/overwritten data
```

```
Write latency: ~1-2ms (just commit log + memtable)
No read-before-write (unlike SQL UPDATE)
```

### Why writes are fast

- **Append-only commit log:** Sequential writes (no disk seeks)
- **No read-before-write:** Unlike SQL UPDATE, no need to read existing data first
- **No locks:** Writes don't block reads or other writes
- **LSM tree structure:** Optimized for write throughput

---

## 6. Read Path

Reads are slower than writes because data may be spread across multiple SSTables.

### Cassandra read path

```
1. Check memtable (in-memory)
   → If found, return immediately

2. Check bloom filters for each SSTable
   → Skip SSTables that definitely don't contain the key

3. Read from SSTables (disk I/O)
   → May need to read multiple SSTables
   → Merge results (latest timestamp wins)

4. Return merged result to client
```

```
Read latency: ~5-10ms (disk I/O + merge)
Slower than writes, but still fast for time-series queries
```

### Read optimization: Compaction

Compaction merges multiple SSTables into one, reducing the number of files to read.

```
Before compaction:
  SSTable 1: {user:123 → name: Alice, age: 30}
  SSTable 2: {user:123 → age: 31}  (update)
  SSTable 3: {user:123 → email: alice@example.com}

After compaction:
  SSTable 4: {user:123 → name: Alice, age: 31, email: alice@example.com}

Read now touches 1 file instead of 3
```

---

## 7. Time-Series Data

Column-family stores excel at time-series data (logs, metrics, events).

### Example: IoT sensor data

```
CREATE TABLE sensor_readings (
    sensor_id UUID,
    timestamp TIMESTAMP,
    temperature FLOAT,
    humidity FLOAT,
    PRIMARY KEY (sensor_id, timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);

Query last 24 hours for sensor 123:
  SELECT * FROM sensor_readings
  WHERE sensor_id = 123
    AND timestamp >= now() - 24h;

Efficient:
  - All data for sensor 123 is on the same node (partition key)
  - Data is sorted by timestamp (clustering column)
  - Read a contiguous block of disk (no random seeks)
```

### TTL (Time To Live)

Automatically delete old data after a specified duration.

```
INSERT INTO sensor_readings (sensor_id, timestamp, temperature)
VALUES (123, now(), 25.5)
USING TTL 86400;  -- delete after 24 hours

No manual cleanup needed — data expires automatically
```

---

## 8. Consistency and Replication

### Replication

Data is replicated across multiple nodes for fault tolerance.

```
Replication factor = 3:
  Partition key = 123 → stored on nodes 2, 3, 4

Write:
  Client → Coordinator node → replicate to 3 nodes → ACK

Read:
  Client → Coordinator node → read from 1 or more replicas
```

### Tunable consistency

Cassandra allows you to choose consistency level per query.

```
Consistency levels:
  ONE   → read/write from 1 replica (fast, low consistency)
  QUORUM → read/write from majority (balanced)
  ALL   → read/write from all replicas (slow, high consistency)

Write with QUORUM:
  Write to 3 replicas, wait for 2 ACKs → return success

Read with QUORUM:
  Read from 2 replicas, return latest value
```

### Eventual consistency

With `ONE` or `QUORUM`, replicas may temporarily be out of sync.

```
Write to node 1: user:123 → age: 31
  (Replicas 2 and 3 haven't received the update yet)

Read from node 2: user:123 → age: 30 (stale)

After a few milliseconds:
  Replicas 2 and 3 receive the update → eventually consistent
```

---

## 9. Cassandra vs HBase

|Feature|Cassandra|HBase|
|---|---|---|
|Architecture|Peer-to-peer (no master)|Master-slave (HMaster + RegionServers)|
|Consistency|Tunable (eventual or strong)|Strong consistency|
|CAP theorem|AP (availability + partition tolerance)|CP (consistency + partition tolerance)|
|Write performance|Very high (no master bottleneck)|High (but master can be bottleneck)|
|Read performance|Good (with compaction)|Good (with block cache)|
|Query language|CQL (SQL-like)|HBase API (no SQL)|
|Use case|High availability, write-heavy|Strong consistency, Hadoop integration|

**Choose Cassandra when:** High availability and write throughput are critical, eventual consistency is acceptable.

**Choose HBase when:** Strong consistency is required, you're already using Hadoop ecosystem.

---

## 10. Use Cases

### 1. Time-series data

Logs, metrics, IoT sensor data, financial tick data.

```
Why column-family stores?
  - High write throughput (millions of writes/sec)
  - Efficient range queries by time
  - Automatic TTL for data expiration
```

### 2. Event logging

User activity logs, audit logs, clickstream data.

```
CREATE TABLE user_activity (
    user_id UUID,
    event_time TIMESTAMP,
    event_type TEXT,
    event_data TEXT,
    PRIMARY KEY (user_id, event_time)
);

Query: Get all events for user 123 in the last hour
  → Single partition read, sorted by time
```

### 3. Messaging and chat

Store messages with high write throughput and efficient retrieval by conversation.

```
CREATE TABLE messages (
    conversation_id UUID,
    message_time TIMESTAMP,
    sender_id UUID,
    message_text TEXT,
    PRIMARY KEY (conversation_id, message_time)
);

Query: Get last 50 messages in conversation 456
  → Single partition read, sorted by time descending
```

### 4. Product catalog

E-commerce product data with varying attributes.

```
CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    name TEXT,
    price DECIMAL,
    category TEXT,
    attributes MAP<TEXT, TEXT>  -- dynamic attributes
);

Product 1: {color: red, size: large}
Product 2: {weight: 5kg, material: steel}

Each product can have different attributes (schema flexibility)
```

---

## 11. Common Interview Questions + Answers

### Q: What's the difference between a column-family store and a traditional SQL database?

> "A column-family store organizes data by columns rather than rows, optimizing for write-heavy workloads and time-series data. It uses an LSM tree structure with append-only writes, making writes very fast. SQL databases organize data by rows and use B-trees, optimizing for transactional workloads with strong consistency. Column-family stores trade strong consistency for high availability and write throughput."

### Q: How does Cassandra achieve high write throughput?

> "Cassandra writes are append-only: first to a commit log (sequential write), then to an in-memory memtable. There's no read-before-write, no locks, and no master node bottleneck. When the memtable fills up, it's flushed to an immutable SSTable on disk. This LSM tree structure is optimized for write throughput, achieving millions of writes per second."

### Q: What's the difference between partition key and clustering columns?

> "The partition key determines which node stores the data via consistent hashing. All data for a partition key is stored together on the same node. Clustering columns determine the sort order within a partition, enabling efficient range queries. For example, PRIMARY KEY (user_id, event_time) means all events for a user are on the same node, sorted by time."

### Q: When would you use Cassandra over a SQL database?

> "Use Cassandra for write-heavy workloads like time-series data, event logging, or IoT sensor data where high availability and write throughput are critical. It's also good for data that doesn't require complex joins or transactions. Use SQL when you need strong consistency, complex queries with joins, or ACID transactions."

---

## 12. Interview Tricks & Pitfalls

### ✅ Trick 1: Emphasize write optimization

Column-family stores are designed for write-heavy workloads. Always mention the LSM tree structure, append-only writes, and no read-before-write.

### ✅ Trick 2: Know the partition key vs clustering column distinction

This is a common interview question. Partition key = which node, clustering columns = sort order within partition.

### ✅ Trick 3: Mention time-series use cases

Time-series data is the sweet spot for column-family stores. Mentioning IoT, logs, or metrics shows you understand the practical applications.

### ❌ Pitfall 1: Confusing column-family stores with columnar databases

Column-family stores (Cassandra) are for OLTP workloads with high write throughput. Columnar databases (Redshift, BigQuery) are for OLAP analytics with complex aggregations. Don't mix them up.

### ❌ Pitfall 2: Thinking Cassandra is always eventually consistent

Cassandra has tunable consistency. With `QUORUM` or `ALL`, you can achieve strong consistency. Don't assume it's always eventually consistent.

### ❌ Pitfall 3: Forgetting about compaction

Reads can be slow if there are many SSTables. Compaction is critical for read performance. Mentioning compaction shows you understand the operational aspects.

---

## 13. Quick Reference

```
What is a column-family store?
  NoSQL database that stores data by columns, not rows
  Optimized for write-heavy workloads and time-series data
  Examples: Cassandra, HBase, Bigtable, ScyllaDB

Data model:
  Column family = container for rows (like SQL table)
  Row = partition key + clustering columns + columns
  Wide rows = millions of columns per partition key

Primary key:
  PRIMARY KEY (partition_key, clustering_column1, ...)
  Partition key → which node (consistent hashing)
  Clustering columns → sort order within partition

Write path:
  1. Commit log (append-only, durable)
  2. Memtable (in-memory, sorted)
  3. ACK to client
  4. Flush to SSTable (background)
  5. Compaction (background)

Read path:
  1. Check memtable
  2. Check bloom filters
  3. Read from SSTables (merge results)

Consistency:
  Tunable: ONE, QUORUM, ALL
  Trade-off: consistency vs latency vs availability

Cassandra vs HBase:
  Cassandra → AP (high availability, eventual consistency)
  HBase     → CP (strong consistency, master-slave)

Use cases:
  Time-series data (IoT, metrics, logs)
  Event logging (user activity, audit logs)
  Messaging (chat, notifications)
  Product catalog (varying attributes)

Why fast writes?
  Append-only commit log (sequential writes)
  No read-before-write
  No locks
  LSM tree structure

Why slower reads?
  Data spread across multiple SSTables
  Need to merge results
  Compaction helps (fewer SSTables to read)
```
