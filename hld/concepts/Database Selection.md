
## 1. The Decision Framework

When choosing a database in a system design interview, run through these questions in order:

```
1. What is the data structure?
   → Relational/tabular? → SQL
   → Nested/variable schema? → Document DB
   → Simple key→value? → Key-Value
   → Highly connected? → Graph DB
   → Timestamped metrics? → Time-Series

2. What are the access patterns?
   → Complex queries, joins, aggregations? → SQL
   → Lookup by ID, simple filters? → NoSQL
   → Full-text search? → Elasticsearch
   → Prefix/proximity search? → Search engine or specialized index

3. What are the consistency requirements?
   → Strong consistency, ACID transactions? → SQL or NewSQL
   → Eventual consistency acceptable? → NoSQL

4. What is the scale?
   → Moderate (< 10M rows, single region)? → SQL is fine
   → Massive, needs horizontal write scaling? → NoSQL
   → Multi-region writes? → DynamoDB, Cassandra, CockroachDB

5. What are the read/write ratios?
   → Read-heavy? → Add read replicas (SQL) or tune for reads (Cassandra)
   → Write-heavy? → LSM-tree based (Cassandra, RocksDB, DynamoDB)
```

---

## 2. Relational Databases (SQL)

### How it works

Data stored in tables with rows and columns. Relationships defined by foreign keys. Schema enforced at the DB level.

```
Users table:          Orders table:
┌────┬──────────┐    ┌────┬─────────┬──────────┐
│ id │ name     │    │ id │ user_id │ amount   │
├────┼──────────┤    ├────┼─────────┼──────────┤
│  1 │ Alice    │    │ 1  │    1    │  $49.99  │
│  2 │ Bob      │    │ 2  │    1    │  $12.00  │
└────┴──────────┘    │ 3  │    2    │  $99.00  │
                     └────┴─────────┴──────────┘

SELECT u.name, SUM(o.amount)
FROM users u JOIN orders o ON u.id = o.user_id
GROUP BY u.name
```

### ACID guarantees

- **Atomicity:** Transaction succeeds completely or rolls back entirely (no partial writes)
- **Consistency:** DB moves from one valid state to another (constraints enforced)
- **Isolation:** Concurrent transactions don't interfere (levels: read committed, repeatable read, serializable)
- **Durability:** Committed transactions survive crashes (WAL, fsync)

### Indexing

```
B-tree index:  fast for range queries (age > 25 AND age < 40)
Hash index:    fast for equality only (id = 123)
Composite:     index on (last_name, first_name) supports queries on last_name alone
Covering:      index includes all columns a query needs (no table lookup required)
```

### Scaling SQL

```
Vertical:   Bigger server (RAM, CPU, SSD). Simple but has a ceiling.
Read replicas: Route reads to replicas. Handles read-heavy loads.
Connection pooling: PgBouncer/ProxySQL reduces connection overhead.
Sharding:   Split data across multiple DB instances (hard, application-level).
```

**SQL works well until:**

- You need to shard (complex joins across shards don't work)
- Your schema changes too rapidly
- You need multi-region active-active writes

### Examples

PostgreSQL, MySQL, MariaDB, Oracle, SQL Server

**PostgreSQL is the default choice** for most applications — rich feature set, excellent JSON support, JSONB indexing, full-text search, PostGIS for geo.

---

## 3. NoSQL — Document Stores

### How it works

Stores data as JSON/BSON documents. Schema is flexible — different documents in the same collection can have different fields.

```json
{
  "_id": "user_123",
  "name": "Alice",
  "address": {
    "city": "New York",
    "zip": "10001"
  },
  "tags": ["premium", "early-adopter"],
  "preferences": {
    "theme": "dark",
    "notifications": true
  }
}
```

No joins — related data is **denormalized** and embedded in the document.

### When to use

- Variable or evolving schema (product catalog with different attributes per category)
- Hierarchical / nested data that would require many tables in SQL
- Content management, user profiles, catalogs
- Rapid iteration (schema changes don't require migrations)

### When NOT to use

- Complex multi-entity transactions (e.g., transfer money between two accounts)
- Complex joins across many entities
- When you need strong ACID guarantees across multiple documents

### Examples

MongoDB, CouchDB, Firestore, DynamoDB (also key-value)

---

## 4. NoSQL — Key-Value Stores

### How it works

The simplest data model: a hash map. Store and retrieve values by key. O(1) reads and writes.

```
SET user:123:session  →  {"token": "abc", "expires": 1700000000}
GET user:123:session  →  {"token": "abc", "expires": 1700000000}
DEL user:123:session
```

No query language. No schema. Just get, put, delete by key.

### When to use

- Session storage
- Caching
- Shopping carts
- Feature flags
- Rate limiting counters
- Leaderboards (Redis sorted sets)

### Examples

Redis, Memcached, DynamoDB, Riak, Amazon ElastiCache

---

## 5. NoSQL — Column-Family Stores

### How it works

Data is stored in rows, but each row can have a different set of columns. Columns are grouped into "column families" and stored together on disk. Optimised for write-heavy, time-series, and large-scale analytical workloads.

Think of it as a **distributed sorted map**:

```
Row key → {column_family: {column: value, column: value}, ...}
```

**Example (Cassandra — user activity log):**

```
Partition key: user_id = "user_123"
Clustering columns: timestamp (sorted)

user_123 | 2024-01-01 10:00 | action=login  | ip=1.2.3.4
user_123 | 2024-01-01 10:05 | action=view   | page=/home
user_123 | 2024-01-01 10:10 | action=click  | element=buy
```

Cassandra stores all rows for a partition on the same node — range queries within a partition are fast. Queries across partitions are expensive.

### Key concepts

**Partition key:** Determines which node stores the data. Design this for even distribution. **Clustering key:** Determines the sort order within a partition. **Wide rows:** A partition can hold millions of columns (e.g., all events for a user).

### When to use

- Time-series data (IoT sensors, logs, metrics)
- Write-heavy workloads (Cassandra uses LSM-tree → writes always sequential)
- High availability, multi-region active-active
- When you can define access patterns upfront (Cassandra needs query-driven design)

### When NOT to use

- Ad-hoc queries (you must know your queries before designing your schema)
- Transactions across multiple partitions
- Small data sets (overhead isn't worth it)

### Examples

Apache Cassandra, Apache HBase, Google Bigtable, Amazon Keyspaces

---

## 6. NoSQL — Graph Databases

### How it works

Data is stored as nodes (entities) and edges (relationships). Optimised for traversing relationships, not tabular lookups.

```
(Alice)──[FOLLOWS]──▶(Bob)
   │                   │
[LIKES]             [FOLLOWS]
   │                   │
   ▼                   ▼
(Post A)          (Charlie)
```

SQL can model this with foreign keys, but multi-hop traversals (friends-of-friends-of-friends) become exponentially expensive with joins. Graph DBs traverse these in constant time per hop.

### When to use

- Social graphs (followers, friends)
- Recommendation engines ("people who liked X also liked Y")
- Fraud detection (find circular transaction patterns)
- Knowledge graphs
- Network topology, access control with complex inheritance

### Examples

Neo4j, Amazon Neptune, ArangoDB, JanusGraph

---

## 7. Time-Series Databases

### How it works

Optimised for storing and querying data points indexed by time. Data is typically append-only, compressed by delta-encoding, and automatically downsampled/archived.

```
Metric:   cpu.usage
Tags:     host=server-1, region=us-east
Points:
  2024-01-01 10:00:00  →  72.3%
  2024-01-01 10:00:10  →  71.8%
  2024-01-01 10:00:20  →  73.1%
```

**Downsampling:** Old high-resolution data is automatically aggregated (e.g., 10-second resolution → 1-minute resolution after 7 days → 1-hour resolution after 30 days). Saves enormous storage.

### When to use

- Application metrics (CPU, memory, latency)
- IoT sensor data
- Financial tick data
- Infrastructure monitoring

### Examples

InfluxDB, TimescaleDB (PostgreSQL extension), Prometheus, Amazon Timestream, ClickHouse

---

## 8. Search Engines

### How it works

Builds an **inverted index**: maps each word to the list of documents containing it. Supports relevance scoring (TF-IDF, BM25), fuzzy matching, faceting, and aggregations.

```
Document 1: "The quick brown fox"
Document 2: "The quick red car"

Inverted index:
  "quick" → [doc1, doc2]
  "fox"   → [doc1]
  "car"   → [doc2]
  "brown" → [doc1]
```

### When to use

- Full-text search (search bar, product search)
- Log analysis and querying
- Faceted search (filter by category, price range, rating)
- Autocomplete and suggestions

### NOT a primary database

Elasticsearch is not durable by default — use it as a derived index on top of your primary database. Write to PostgreSQL/MongoDB, sync to Elasticsearch via CDC or streaming.

### Examples

Elasticsearch, OpenSearch, Solr, Typesense, Meilisearch, Algolia

---

## 9. NewSQL — Distributed SQL

### How it works

Combines the ACID guarantees and SQL interface of traditional RDBMS with the horizontal scalability of NoSQL. Uses distributed consensus (Raft/Paxos) for strongly consistent replication.

```
CockroachDB / Spanner:
  ┌─────────────────────────────────────┐
  │           SQL Interface             │
  ├─────────────────────────────────────┤
  │    Distributed Transaction Layer    │
  │    (2PC + Raft consensus)           │
  ├─────────────────────────────────────┤
  │  Node A  │  Node B  │  Node C      │
  └─────────────────────────────────────┘
```

### When to use

- Need SQL with horizontal write scaling
- Multi-region deployments with strong consistency
- Financial systems that need ACID but have outgrown a single SQL node

### Examples

Google Spanner, CockroachDB, TiDB, PlanetScale (MySQL-compatible), YugabyteDB

**Tradeoff:** Higher latency than single-node SQL (consensus adds ~2-10ms per write), more operational complexity.

---

## 10. ACID vs BASE

### ACID (SQL, NewSQL)

```
Atomicity:    All or nothing. Transfer $100 either completes fully or not at all.
Consistency:  DB constraints always satisfied. Balance can't go negative.
Isolation:    Concurrent transactions don't see each other's partial state.
Durability:   Committed data survives power loss.
```

### BASE (NoSQL)

```
Basically Available: System responds even during partial failures.
Soft state:          Data may change over time (due to async replication).
Eventually consistent: Given no new updates, all replicas will converge.
```

**The key question in interviews:** "What happens if two concurrent writes conflict?"

- ACID: One wins, one is retried. DB handles it.
- BASE: Both succeed; you get a conflict that must be resolved (last-write-wins, vector clocks, application logic).

---

## 11. Choosing in Practice — Real Scenarios

### E-commerce platform

```
Users, orders, inventory:     PostgreSQL  (ACID transactions, joins)
Product catalog:              MongoDB     (variable schema per category)
Session storage:              Redis       (fast key-value)
Product search:               Elasticsearch
Order activity feed:          Cassandra   (time-ordered, write-heavy)
```

### Social media platform

```
User profiles:                PostgreSQL or DynamoDB
Posts + comments:             Cassandra   (write-heavy, time-ordered)
Social graph (followers):     Graph DB (Neptune) or adjacency table in SQL
Notifications:                Cassandra
Search:                       Elasticsearch
Cache:                        Redis
```

### Ride-sharing app

```
User/driver accounts:         PostgreSQL
Ride history:                 Cassandra   (time-series, append-only)
Driver location (real-time):  Redis (geospatial, TTL)
Geospatial search:            PostGIS or dedicated geo DB
Payments:                     PostgreSQL  (must be ACID)
Analytics:                    ClickHouse / BigQuery
```

---

## 12. Common Interview Questions + Answers

### Q: When would you choose NoSQL over SQL?

> "When the data model is a poor fit for tables — like hierarchical documents, variable schemas per entity, or massive time-series. When you need horizontal write scaling beyond what a single SQL node can handle. When you're building with a team that needs to iterate schema quickly. But if you have complex relationships, need multi-row transactions, or value ad-hoc query flexibility, SQL is usually better."

### Q: Can you use multiple databases in one system?

> "Yes, and it's common — this is called polyglot persistence. An e-commerce system might use PostgreSQL for orders (ACID), Cassandra for activity logs (time-series write scale), Elasticsearch for product search (full-text), and Redis for sessions and caching. Each store is chosen for its strengths. The tradeoff is operational complexity — more moving parts to manage."

### Q: How do you handle a situation where your SQL database is becoming a bottleneck?

> "I'd diagnose first: is it a query problem (add indexes, optimize queries, use read replicas) or a capacity problem (connection pool exhaustion, disk I/O)? Common steps: add read replicas for read-heavy load, add a cache layer (Redis) in front, optimize slow queries, consider vertical scaling. If write throughput is the bottleneck and those don't help, consider sharding or migrating write-heavy tables to a NoSQL store."

---

## 13. Interview Tricks & Pitfalls

### ✅ Trick 1: Always justify your choice

Never just say "I'll use PostgreSQL." Say:

> "I'll use PostgreSQL because we have relational data with joins, need ACID transactions for the payment flow, and the scale (10M users) is well within what a well-tuned Postgres instance with read replicas can handle."

### ✅ Trick 2: Polyglot persistence is a valid design

Don't feel forced to pick one database. Real systems use multiple. The interviewer often expects you to call out that different parts of the system have different needs.

### ✅ Trick 3: Mention Cassandra's query-driven design

A very common follow-up for Cassandra: "How do you design the schema?" The answer is always:

> "In Cassandra, you design tables around your queries, not your data model. Define your access patterns first, then build a table that answers each pattern directly. Denormalization is expected."

### ❌ Pitfall 1: "NoSQL is better for scale"

This is oversimplified. PostgreSQL with read replicas handles enormous scale. Cassandra's write throughput is impressive, but it sacrifices flexibility and consistency. Make the case for each based on specific requirements.

### ❌ Pitfall 2: Using Elasticsearch as primary storage

Elasticsearch can lose data on cluster failures and is not designed for primary storage. It should always be a secondary index on top of a primary DB.

### ❌ Pitfall 3: Forgetting that NoSQL doesn't mean "no schema"

NoSQL means no rigid enforced schema, but you still need a logical schema. Cassandra schemas are very rigid in practice because of query-driven design. "NoSQL = flexible" is a naive characterisation.

---

## 14. Quick Reference

```
SQL (PostgreSQL, MySQL):
  ✅ Complex joins, ACID transactions, ad-hoc queries
  ❌ Hard to scale writes horizontally
  Use for: User data, payments, orders, inventory

Document (MongoDB):
  ✅ Flexible schema, nested data, fast iteration
  ❌ No multi-document transactions (well, limited)
  Use for: Catalogs, CMS, user profiles, event logs

Key-Value (Redis, DynamoDB):
  ✅ O(1) get/put, extremely fast
  ❌ No query language, no relationships
  Use for: Sessions, cache, counters, rate limiting

Column-Family (Cassandra, HBase):
  ✅ Massive write throughput, time-ordered data, multi-region
  ❌ Query-driven schema design required, no joins
  Use for: Logs, time-series, activity feeds, IoT

Graph (Neo4j, Neptune):
  ✅ Multi-hop relationship traversal
  ❌ Not for aggregate queries or tabular data
  Use for: Social graph, recommendations, fraud detection

Time-Series (InfluxDB, TimescaleDB):
  ✅ Compressed, auto-downsampling, time-range queries
  Use for: Metrics, monitoring, IoT, financial ticks

Search (Elasticsearch):
  ✅ Full-text, fuzzy, faceted search
  ❌ Not a primary store
  Use for: Search bars, log analysis, product search

NewSQL (CockroachDB, Spanner):
  ✅ SQL + horizontal scale + ACID
  ❌ Higher latency than single-node SQL
  Use for: Global ACID at scale
```