## 1. What Is NoSQL?

NoSQL (Not Only SQL) databases are non-relational databases designed for flexible schemas, horizontal scalability, and specific data models.

**Four main types:**

```
Document:      JSON/BSON documents (MongoDB, CouchDB)
Key-Value:     Simple key→value pairs (Redis, DynamoDB)
Column-Family: Wide-column stores (Cassandra, HBase)
Graph:         Nodes and edges (Neo4j, Neptune)
```

**Key characteristics:**
- Schema flexibility (no fixed schema)
- Horizontal scalability (add more nodes)
- Eventually consistent (by default)
- Optimized for specific access patterns

---

## 2. Document Databases

Store data as JSON/BSON documents. Each document can have different fields.

### MongoDB example

```json
// User document
{
  "_id": "user_123",
  "name": "Alice",
  "email": "alice@example.com",
  "address": {
    "street": "123 Main St",
    "city": "New York",
    "zip": "10001"
  },
  "tags": ["premium", "early-adopter"],
  "preferences": {
    "theme": "dark",
    "notifications": true
  },
  "created_at": "2024-01-15T10:30:00Z"
}

// Product document (different structure)
{
  "_id": "product_456",
  "name": "Laptop",
  "price": 999.99,
  "specs": {
    "cpu": "Intel i7",
    "ram": "16GB",
    "storage": "512GB SSD"
  },
  "reviews": [
    {"user": "user_789", "rating": 5, "comment": "Great!"},
    {"user": "user_012", "rating": 4, "comment": "Good value"}
  ]
}
```

**No joins** — related data is embedded or denormalized.

---

### When to use document databases

**Good for:**
- Variable schema (product catalogs with different attributes per category)
- Nested/hierarchical data (user profiles, blog posts with comments)
- Rapid iteration (schema changes don't require migrations)
- Content management systems
- User profiles and preferences

**Not good for:**
- Complex multi-document transactions
- Complex joins across many entities
- When you need strong ACID guarantees across documents

---

### Querying

```javascript
// Find users in New York
db.users.find({ "address.city": "New York" })

// Find premium users
db.users.find({ tags: "premium" })

// Update nested field
db.users.updateOne(
  { _id: "user_123" },
  { $set: { "preferences.theme": "light" } }
)

// Aggregation pipeline
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$user_id", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } },
  { $limit: 10 }
])
```

---

### Indexing

```javascript
// Single field index
db.users.createIndex({ email: 1 })

// Compound index
db.orders.createIndex({ user_id: 1, created_at: -1 })

// Text index for full-text search
db.products.createIndex({ name: "text", description: "text" })

// Geospatial index
db.stores.createIndex({ location: "2dsphere" })
```

**Without indexes:** Full collection scan (slow). **With indexes:** Fast lookups, just like SQL.

---

## 3. Key-Value Databases

Simplest data model: a distributed hash map.

```
Key                    Value
─────────────────────────────────────────────────
user:123:session    → {"token": "abc", "expires": 1700000000}
cart:456            → {"items": [{"id": 1, "qty": 2}]}
rate_limit:789      → 42
feature:dark_mode   → true
```

**Operations:**

```
GET key           → retrieve value
SET key value     → store value
DEL key           → delete key
EXISTS key        → check if key exists
INCR key          → increment counter
EXPIRE key ttl    → set expiration
```

---

### Redis example

```bash
# Session storage
SET user:123:session '{"token":"abc","expires":1700000000}' EX 3600

# Shopping cart
HSET cart:456 item:1 2  # item 1, quantity 2
HSET cart:456 item:2 1  # item 2, quantity 1
HGETALL cart:456

# Rate limiting
INCR rate_limit:user:789
EXPIRE rate_limit:user:789 60  # reset after 60 seconds

# Leaderboard (sorted set)
ZADD leaderboard 1000 "player1"
ZADD leaderboard 1500 "player2"
ZREVRANGE leaderboard 0 9  # top 10
```

---

### Redis data structures

```
String:      Simple key-value
Hash:        Object with fields (like a document)
List:        Ordered list (queue, stack)
Set:         Unordered unique values
Sorted Set:  Set with scores (leaderboards, priority queues)
Stream:      Append-only log (like Kafka)
```

---

### When to use key-value databases

**Good for:**
- Caching (most common use)
- Session storage
- Shopping carts
- Rate limiting counters
- Feature flags
- Leaderboards (Redis sorted sets)
- Real-time analytics

**Not good for:**
- Complex queries
- Relationships between entities
- Ad-hoc queries (you must know the key)

---

## 4. Column-Family Databases

Data stored in rows, but each row can have different columns. Columns grouped into families.

### Cassandra example

```
Table: user_activity

Partition key: user_id
Clustering key: timestamp (sorted)

user_123 | 2024-01-01 10:00 | action=login  | ip=1.2.3.4
user_123 | 2024-01-01 10:05 | action=view   | page=/home
user_123 | 2024-01-01 10:10 | action=click  | element=buy
user_456 | 2024-01-01 10:15 | action=login  | ip=5.6.7.8
```

**Key concepts:**

**Partition key:** Determines which node stores the data. All rows with same partition key stored together.

**Clustering key:** Determines sort order within a partition.

**Wide rows:** A partition can have millions of columns.

---

### Query patterns

```sql
-- Good: Query within a partition
SELECT * FROM user_activity 
WHERE user_id = 'user_123' 
AND timestamp > '2024-01-01';

-- Bad: Query across partitions (requires full table scan)
SELECT * FROM user_activity 
WHERE timestamp > '2024-01-01';  -- no partition key!
```

**Design rule:** Know your queries before designing your schema. Cassandra is query-driven.

---

### When to use column-family databases

**Good for:**
- Time-series data (IoT sensors, logs, metrics)
- Write-heavy workloads (Cassandra uses LSM-tree → sequential writes)
- High availability, multi-region active-active
- When you can define access patterns upfront

**Not good for:**
- Ad-hoc queries (must know partition key)
- Transactions across partitions
- Small datasets (overhead not worth it)

---

## 5. Graph Databases

Data stored as nodes (entities) and edges (relationships).

### Neo4j example

```
Nodes:
  (alice:User {name: "Alice", age: 28})
  (bob:User {name: "Bob", age: 35})
  (post:Post {title: "Hello World", content: "..."})

Edges:
  (alice)-[:FOLLOWS]->(bob)
  (alice)-[:LIKES]->(post)
  (bob)-[:FOLLOWS]->(alice)
```

**Visualization:**

```
(Alice)──[FOLLOWS]──▶(Bob)
   │                   │
[LIKES]             [FOLLOWS]
   │                   │
   ▼                   ▼
(Post)            (Charlie)
```

---

### Querying (Cypher)

```cypher
// Find Alice's followers
MATCH (follower:User)-[:FOLLOWS]->(alice:User {name: "Alice"})
RETURN follower.name

// Find friends-of-friends
MATCH (alice:User {name: "Alice"})-[:FOLLOWS]->()-[:FOLLOWS]->(fof)
WHERE fof <> alice
RETURN DISTINCT fof.name

// Shortest path between two users
MATCH path = shortestPath(
  (alice:User {name: "Alice"})-[:FOLLOWS*]-(bob:User {name: "Bob"})
)
RETURN path

// Recommendation: people who liked the same posts
MATCH (alice:User {name: "Alice"})-[:LIKES]->(post)<-[:LIKES]-(other)
WHERE other <> alice
RETURN other.name, COUNT(post) AS common_likes
ORDER BY common_likes DESC
```

---

### When to use graph databases

**Good for:**
- Social networks (followers, friends)
- Recommendation engines ("people who liked X also liked Y")
- Fraud detection (find circular transaction patterns)
- Knowledge graphs
- Network topology
- Access control with complex inheritance

**Not good for:**
- Tabular data
- Aggregate queries (SUM, AVG across millions of nodes)
- Simple key-value lookups

---

## 6. NoSQL vs SQL Comparison

|Aspect|SQL|NoSQL|
|---|---|---|
|Schema|Fixed, enforced|Flexible, optional|
|Relationships|Foreign keys, joins|Embedded or denormalized|
|Transactions|ACID across tables|Limited (single document/row)|
|Scaling|Vertical + read replicas|Horizontal (add nodes)|
|Consistency|Strong (default)|Eventual (default)|
|Query flexibility|Ad-hoc SQL queries|Query-driven design|
|Use case|Relational data, ACID|Flexible schema, high scale|

---

## 7. CAP Theorem and NoSQL

Most NoSQL databases are **AP** (Available + Partition Tolerant) by default.

```
Cassandra (AP):
  During partition: all nodes accept writes
  Eventually consistent
  Can be tuned to CP with QUORUM

MongoDB (CP):
  During partition: minority nodes refuse writes
  Strongly consistent
  Can be tuned to AP with w=1

DynamoDB (AP):
  Eventually consistent by default
  Strongly consistent reads available (opt-in)
```

---

## 8. Common NoSQL Patterns

### Denormalization

Embed related data instead of referencing.

**SQL approach:**

```sql
-- Two tables with join
SELECT u.name, o.amount 
FROM users u 
JOIN orders o ON u.id = o.user_id
```

**NoSQL approach:**

```json
// Embed user data in order
{
  "order_id": "order_123",
  "amount": 99.99,
  "user": {
    "id": "user_456",
    "name": "Alice",
    "email": "alice@example.com"
  }
}
```

**Tradeoff:** Faster reads (no join), but data duplication. If user changes email, must update all orders.

---

### Materialized views

Pre-compute and store query results.

```javascript
// Real-time: expensive aggregation
db.orders.aggregate([
  { $group: { _id: "$user_id", total: { $sum: "$amount" } } }
])

// Materialized view: pre-computed
db.user_totals.find({ user_id: "user_123" })
// Returns: { user_id: "user_123", total: 1234.56 }

// Update view on every order insert (via trigger or application)
```

---

### Time-series bucketing

Group time-series data into buckets.

```javascript
// Bad: One document per event (millions of documents)
{ sensor_id: "sensor_1", timestamp: "2024-01-01 10:00:00", temp: 72.5 }
{ sensor_id: "sensor_1", timestamp: "2024-01-01 10:00:01", temp: 72.6 }

// Good: One document per hour with array of readings
{
  sensor_id: "sensor_1",
  hour: "2024-01-01 10:00",
  readings: [
    { timestamp: "10:00:00", temp: 72.5 },
    { timestamp: "10:00:01", temp: 72.6 },
    // ... 3600 readings
  ]
}
```

**Benefit:** Fewer documents, faster queries, better compression.

---

## 9. Common Interview Questions + Answers

### Q: When would you choose NoSQL over SQL?

> "Choose NoSQL when the data model is a poor fit for tables — like hierarchical documents, variable schemas per entity, or massive time-series data. When you need horizontal write scaling beyond what a single SQL node can handle. When you're building with a team that needs to iterate schema quickly. But if you have complex relationships, need multi-row transactions, or value ad-hoc query flexibility, SQL is usually better. Many systems use both — SQL for transactional data, NoSQL for high-volume logs or flexible catalogs."

### Q: What's the difference between document and key-value databases?

> "Key-value stores are simpler — just a hash map with get/set/delete operations. You must know the exact key to retrieve data. Document databases store structured documents (JSON) and let you query by any field, not just the ID. MongoDB can query `db.users.find({age: {$gt: 25}})` — you can't do that with a pure key-value store. Key-value is better for caching and simple lookups. Document databases are better when you need to query by multiple fields or have nested data."

### Q: How does Cassandra achieve high availability?

> "Cassandra uses leaderless replication with tunable consistency. For replication factor 3, each write goes to 3 nodes. With consistency level ONE, a write succeeds when any single node acknowledges — very fast and available. With QUORUM, 2 of 3 must acknowledge — slower but more consistent. During a network partition, all nodes continue accepting writes independently. When the partition heals, anti-entropy repair reconciles differences. This AP design means Cassandra stays available even when nodes or entire datacenters fail."

### Q: What are the tradeoffs of denormalization in NoSQL?

> "Denormalization makes reads faster by avoiding joins — all data is in one document. But it creates data duplication. If a user changes their email, you must update it in every order document, not just the users collection. This makes writes more complex and risks inconsistency if updates fail partway through. The tradeoff is worth it for read-heavy workloads where the embedded data rarely changes. For frequently changing data, keep it normalized and accept the cost of multiple queries."

---

## 10. Interview Tricks & Pitfalls

### ✅ Trick 1: Know the four types

When asked about NoSQL, immediately categorize:

> "There are four main types: document stores like MongoDB for flexible schemas, key-value stores like Redis for caching, column-family stores like Cassandra for time-series, and graph databases like Neo4j for relationships. The choice depends on the data model and access patterns."

### ✅ Trick 2: Mention query-driven design

For Cassandra specifically:

> "Cassandra requires query-driven design — you define your queries first, then design tables to answer each query directly. You can't do ad-hoc queries like SQL. This is a major operational difference."

### ✅ Trick 3: Discuss consistency tuning

Show you understand NoSQL isn't just "eventually consistent":

> "Cassandra is tunable. With consistency level ONE, it's AP — highly available but eventually consistent. With QUORUM, it's CP — strongly consistent but less available during partitions. You choose per-operation based on requirements."

### ❌ Pitfall 1: Saying "NoSQL is faster"

NoSQL isn't inherently faster. It's optimized for specific patterns. A well-indexed SQL query can be faster than a poorly designed NoSQL query.

### ❌ Pitfall 2: Forgetting about transactions

If you propose NoSQL, be ready to explain how you handle multi-document updates:

> "MongoDB has limited multi-document transactions. For operations that must be atomic across documents, I'd either use transactions (with the performance cost) or design the schema to keep related data in one document."

### ❌ Pitfall 3: Not knowing when to use SQL

Don't default to NoSQL. SQL is often the better choice. Be ready to justify NoSQL over SQL.

---

## 11. Quick Reference

```
NoSQL = non-relational, flexible schema, horizontally scalable

Four types:
  Document:      JSON documents, flexible schema (MongoDB)
  Key-Value:     Simple hash map (Redis, DynamoDB)
  Column-Family: Wide-column, time-series (Cassandra, HBase)
  Graph:         Nodes and edges (Neo4j, Neptune)

Document (MongoDB):
  ✅ Variable schema, nested data, rapid iteration
  ❌ Limited multi-document transactions
  Use for: Catalogs, CMS, user profiles

Key-Value (Redis):
  ✅ O(1) operations, extremely fast
  ❌ No complex queries, must know key
  Use for: Caching, sessions, counters

Column-Family (Cassandra):
  ✅ Massive write throughput, multi-region, time-series
  ❌ Query-driven design, no ad-hoc queries
  Use for: Logs, metrics, IoT, activity feeds

Graph (Neo4j):
  ✅ Multi-hop traversals, relationship queries
  ❌ Not for tabular data or aggregations
  Use for: Social graphs, recommendations, fraud detection

NoSQL vs SQL:
  NoSQL: Flexible schema, horizontal scale, eventual consistency
  SQL: Fixed schema, ACID, ad-hoc queries
  Many systems use both (polyglot persistence)

Common patterns:
  Denormalization:     Embed related data (faster reads, data duplication)
  Materialized views:  Pre-compute aggregations
  Time-series bucketing: Group events into documents

CAP positioning:
  Cassandra: AP (tunable to CP with QUORUM)
  MongoDB: CP (tunable to AP with w=1)
  DynamoDB: AP (strongly consistent reads opt-in)
```
