
## 1. What Is Sharding?

Sharding (also called horizontal partitioning) splits a large dataset across multiple database nodes, where each node holds a **subset** of the data.

```
Without sharding:
  All data on one node
  ┌────────────────────┐
  │ users: 1 billion   │  ← 10 TB, single DB can't handle it
  └────────────────────┘

With sharding (4 shards):
  ┌──────────────┐  ┌──────────────┐
  │ Shard 1      │  │ Shard 2      │
  │ users 0–250M │  │ users 250M–500M│
  └──────────────┘  └──────────────┘
  ┌──────────────┐  ┌──────────────┐
  │ Shard 3      │  │ Shard 4      │
  │ users 500M–750M│ │ users 750M–1B │
  └──────────────┘  └──────────────┘
```

**Sharding vs replication:**

- **Replication:** Same data on multiple nodes (read scale, availability)
- **Sharding:** Different data on different nodes (write scale, storage scale)

These are complementary — production systems shard data AND replicate each shard.

**When do you need sharding?**

- Dataset is too large for one server's disk
- Write throughput exceeds what one primary can handle
- Query latency is too high because one node has too much data

---

## 2. Partitioning Strategies

### Hash-Based Partitioning

Hash the partition key and use the hash to determine the shard.

```
shard = hash(user_id) % num_shards

hash("user_123") % 4 = 2  → Shard 2
hash("user_456") % 4 = 0  → Shard 0
hash("user_789") % 4 = 3  → Shard 3
```

**Pros:**

- Even distribution of data (assuming good hash function)
- Simple to compute — no lookup table needed

**Cons:**

- Range queries are inefficient — `user_id BETWEEN 100 AND 200` hits all shards
- Resharding (adding/removing shards) remaps almost all keys (use consistent hashing to solve this)

**Best for:** Workloads with high write volume and mostly point lookups (lookup by exact ID).

---

### Range-Based Partitioning

Divide the keyspace into continuous ranges. Each shard owns a range.

```
Shard 0:  user_id 1        → 1,000,000
Shard 1:  user_id 1,000,001 → 2,000,000
Shard 2:  user_id 2,000,001 → 3,000,000
Shard 3:  user_id 3,000,001 → 4,000,000
```

**Pros:**

- Range queries are efficient — `user_id BETWEEN 500K AND 700K` hits only Shard 0
- Natural for time-series data (partition by date range)

**Cons:**

- Can create hot spots — if IDs are sequential, all new writes go to the last shard
- Uneven distribution — some ranges may have much more data than others

**Best for:** Time-series data, data with natural ordering, workloads that need range scans.

**HBase and Google Bigtable use range partitioning** — they auto-split partitions when they get too large.

---

### Directory-Based Partitioning

A lookup service maps each key to its shard. No algorithm — the directory is the source of truth.

```
Lookup table:
  user_1   → Shard A
  user_2   → Shard B
  user_3   → Shard A
  user_100 → Shard C
```

**Pros:**

- Maximum flexibility — can rebalance individual keys
- Can handle heterogeneous data sizes

**Cons:**

- Lookup service is a bottleneck and SPOF (must be cached/replicated)
- Extra hop for every query

**Best for:** When data distribution is highly irregular and you need fine-grained control.

---

### Geo-Based Partitioning

Partition by user geography. All European users go to EU shards; US users to US shards.

```
US users    → US-East, US-West shards
EU users    → EU-West, EU-Central shards
APAC users  → APAC shards
```

**Pros:**

- Data residency compliance (GDPR — EU user data stays in EU)
- Lower latency — users' data is near them

**Cons:**

- Uneven distribution if regions have different user volumes
- Cross-region queries (e.g., "find all users who've bought from this seller in any region") are slow

---

## 3. Hot Spots and How to Avoid Them

A hot spot is when a disproportionate amount of traffic hits one shard — typically because a partition key has very unequal distribution.

### Hot spot cause 1: Sequential keys

```
Auto-increment IDs: 1, 2, 3, 4, ...
Range partition: all new records go to the last shard
→ Last shard is 100% of write traffic
```

**Fix:** Use random or hash-based keys (UUID, Snowflake IDs). Distribute new writes across all shards.

### Hot spot cause 2: Celebrity / skewed data

```
On Twitter, Elon Musk has 100M followers.
If user_id is the partition key, all writes about his tweets
go to the same shard.
```

**Fix for write hot spots:**

- Add a random suffix to hot keys: `tweet_123_0`, `tweet_123_1`, ..., `tweet_123_9`
- Writes spread across 10 keys → 10 shards
- Reads must query all 10 keys and aggregate

**Fix for read hot spots:**

- Cache the hot key in memory (Redis) so DB shards don't see the read load
- Read from replicas of the hot shard

### Hot spot cause 3: Time-based partitioning

If you partition by date (to support range queries), today's shard always gets all the writes.

```
Partition by month:
  Jan shard: reads only (no new Jan data being written)
  Feb shard: reads only
  Mar shard: ALL writes right now
```

**Fix:** Combine time-based partitioning with hash within the time partition, or use pre-splitting.

---

## 4. Cross-Shard Operations

Sharding breaks operations that span multiple shards. This is the main complexity of sharding.

### Cross-shard joins

```sql
-- Easy on a single DB:
SELECT u.name, o.amount
FROM users u JOIN orders o ON u.id = o.user_id
WHERE u.country = 'US'

-- Sharded: users and orders might be on different shards!
```

**Solutions:**

1. **Denormalization:** Store the user name inside the orders table. No join needed.
2. **Application-level join:** Fetch from both shards in the application, join in code.
3. **Co-location:** Always shard users and their orders on the same shard (using the same partition key: `user_id`).

Co-location is the preferred approach — design your sharding scheme so related data lives on the same shard.

### Cross-shard transactions

A transaction that updates data on two different shards requires **distributed transactions** (2-phase commit), which are complex and slow.

**Solutions:**

1. **Avoid them by design:** Structure data so a transaction only touches one shard.
2. **Saga pattern:** Break into a series of local transactions with compensating transactions for rollback.
3. **Two-phase commit (2PC):** Correct but adds latency and is prone to blocking on coordinator failure.

### Cross-shard aggregations

```sql
SELECT COUNT(*) FROM users WHERE age > 25  -- hits ALL shards
```

**Solution:** Use a separate analytics store (data warehouse, ClickHouse, Redshift) for aggregations. Don't run analytics queries on your OLTP shards.

---

## 5. Resharding

Adding or removing shards — the hardest part of operating a sharded system.

### Why it's hard

```
Old: 4 shards, 1M users each
Add shard 5: 1M users need to move

Moving 1M rows while the service is live:
  - Can't take downtime
  - Reads/writes to migrating data must work during migration
  - Double-writes during migration period
```

### Resharding approaches

**1. Consistent hashing (preferred):** With consistent hashing, adding a node moves ~1/N of keys. Much better than naive modulo hashing.

**2. Pre-sharding:** Create far more logical shards than physical nodes. E.g., 1024 logical shards on 4 physical nodes (256 logical per physical). When scaling, migrate logical shards rather than rehashing all data.

```
Initial: 4 nodes × 256 logical shards each
Scale to 8 nodes: move 128 logical shards from each old node to new nodes
Data movement = 50% of total data (still a lot, but no rehashing)
```

**3. Online migration:**

1. Set up new shard with replication from old shard
2. Double-write to old and new shards
3. Verify new shard is caught up
4. Switch reads to new shard
5. Drain old shard

---

## 6. Directory-Based Sharding

Worth expanding because it solves resharding elegantly.

```
Client → Shard Lookup Service → finds shard for key → routes request

Shard Lookup Service:
  Stores: key_range → shard_id mapping
  Cached aggressively (LRU cache on client side)
  Updated when resharding happens
```

**Resharding with directory:**

1. Move data from Shard A to new Shard E
2. Update the lookup table: those key ranges now point to Shard E
3. Clients with cached mappings will miss once, get the new mapping, and be updated

No application code changes. Just update the directory.

---

## 7. Application-Level vs Middleware vs DB-Level Sharding

### Application-level sharding

The application knows about shards and routes directly.

```python
def get_shard(user_id):
    return user_id % NUM_SHARDS

def get_user(user_id):
    shard = get_shard(user_id)
    db_connection = SHARD_CONNECTIONS[shard]
    return db_connection.query("SELECT * FROM users WHERE id = ?", user_id)
```

**Pros:** Full control, no middleware overhead. **Cons:** Sharding logic in every service, language, and team. Schema changes are harder.

### Middleware / proxy sharding

A proxy sits between application and database, routing queries transparently.

```
App → [ProxySQL / Vitess / PgBouncer] → correct shard
```

Application thinks it's talking to one database. Proxy handles shard routing.

**Vitess** (used by YouTube, Slack) does this for MySQL. The app speaks MySQL; Vitess handles sharding, resharding, and query routing.

### DB-level sharding

Some databases shard natively:

- **Cassandra:** Built-in consistent hashing and replication
- **MongoDB:** Built-in sharding with `mongos` router
- **CockroachDB/Spanner:** Transparent distributed SQL

---

## 8. Real-World Systems

|System|Sharding Approach|
|---|---|
|Cassandra|Consistent hashing, vnodes, automatic|
|DynamoDB|Automatic hash-based partitioning|
|MongoDB|Hash or range on shard key, mongos router|
|Vitess|Directory-based for MySQL|
|HBase / Bigtable|Range-based, auto-splits on size|
|Elasticsearch|Hash-based, fixed at index creation|
|Instagram|PostgreSQL with application-level sharding (Shard ID in ID)|

**Instagram's ID structure (interesting for interviews):**

```
64-bit ID = [41 bits timestamp] [13 bits shard ID] [10 bits sequence]

Shard ID is embedded in every ID → routing is O(1), no lookup service needed
```

---

## 9. Common Interview Questions + Answers

### Q: How would you shard a user table for a social network with 1 billion users?

> "I'd use hash-based sharding on `user_id`. A consistent hashing ring with 10–20 physical shards, each with a replica set. I'd pre-shard with 1024 logical shards to make future scaling easier. I'd embed the shard ID in the user ID (like Instagram) so routing is computed locally without a lookup service. For celebrity accounts that cause hot spots, I'd cache their data aggressively in Redis."

### Q: What problems does sharding introduce?

> "Three main ones: cross-shard joins (solved by denormalization and co-location), distributed transactions (avoided by design — structure data to keep transactions within one shard, use sagas for the rest), and resharding complexity (use consistent hashing or pre-sharding with logical shards to minimise data movement)."

### Q: How do you choose a sharding key?

> "The ideal sharding key: has high cardinality (many distinct values), distributes writes evenly, and is the primary lookup key for most queries. User_id works well for user-centric apps — most queries are scoped to a user. Bad choices: status field (only a few values → hot shards), timestamp (new data always hits one shard). The key must also support co-location of related data — user and their orders should share the shard key so they land on the same shard."

---

## 10. Interview Tricks & Pitfalls

### ✅ Trick 1: Sharding is a last resort

Start with:

1. Vertical scaling (bigger server)
2. Read replicas (scale reads)
3. Caching (reduce DB load)
4. Indexing optimisation

Only shard when those are exhausted. Sharding adds enormous operational complexity.

### ✅ Trick 2: Always address the hot spot problem

When proposing any sharding scheme, the interviewer will ask about hot spots. Have the answer ready:

- Hash sharding → watch for celebrity users or skewed access
- Range sharding → watch for sequential write patterns
- Fix: random suffixes for write spreading, caching for read hot spots

### ✅ Trick 3: Mention pre-sharding

Proposing 1024 logical shards on 4 physical nodes shows operational maturity. You've thought about the future pain of resharding and designed to minimise it.

### ❌ Pitfall 1: Using sharding for a small system

If your data fits on one server with read replicas and a cache, don't shard. The operational complexity isn't worth it until you've clearly exhausted simpler options.

### ❌ Pitfall 2: Choosing a low-cardinality shard key

`status` (active/inactive), `country` (200 values), `gender` — these are terrible shard keys. They create hot shards with no way to redistribute load within the shard.

### ❌ Pitfall 3: Forgetting that resharding is painful

Never present sharding as "and then we can easily add more shards." Always acknowledge that resharding is an operational challenge and explain your mitigation (consistent hashing, pre-sharding, online migration).

---

## 11. Quick Reference

```
Sharding = different data on different nodes (write scale)
Replication = same data on multiple nodes (read scale, availability)

Strategies:
  Hash-based    → even distribution, no range queries
  Range-based   → good for ranges/time-series, hot spot risk
  Directory     → flexible, lookup service needed
  Geo-based     → data residency, latency, uneven if regions differ

Choosing a shard key:
  ✅ High cardinality
  ✅ Even write distribution
  ✅ Matches primary access pattern
  ✅ Enables co-location of related data
  ❌ Sequential IDs (range hot spot)
  ❌ Low cardinality (status, gender, country)

Hot spot fixes:
  Write hot spot → random suffix on hot key (spread across shards)
  Read hot spot  → Redis cache in front

Cross-shard:
  Joins         → denormalize or co-locate
  Transactions  → design to avoid; use Sagas if unavoidable
  Aggregations  → use separate analytics store

Resharding:
  Use consistent hashing   → O(K/N) keys move
  Use pre-sharding         → logical shards > physical nodes
  Use online migration     → double-write + cut over

Tools: Vitess (MySQL), mongos (MongoDB), Cassandra (native)
```