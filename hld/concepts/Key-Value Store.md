## 1. What Is a Key-Value Store?

A key-value store is the simplest NoSQL database model: a distributed hash map where each key maps to a value.

```
Key                    Value
─────────────────────────────────────────────────
user:123            → {"name": "Alice", "email": "alice@example.com"}
session:abc         → {"user_id": 123, "expires": 1700000000}
cart:456            → {"items": [{"id": 1, "qty": 2}]}
counter:page_views  → 42
```

**Operations:**

```
GET key           → retrieve value
PUT key value     → store value
DELETE key        → remove key
```

**That's it.** No query language, no joins, no schema.

---

## 2. Why Key-Value Stores?

### Simplicity

```
SQL:
  CREATE TABLE users (id INT, name VARCHAR, email VARCHAR);
  INSERT INTO users VALUES (1, 'Alice', 'alice@example.com');
  SELECT * FROM users WHERE id = 1;

Key-Value:
  PUT user:1 '{"name":"Alice","email":"alice@example.com"}'
  GET user:1
```

**Simpler model = easier to scale, easier to reason about.**

---

### Performance

```
O(1) operations:
  GET key  → hash(key) → find node → retrieve value
  PUT key  → hash(key) → find node → store value

No query parsing, no query planning, no joins
Just: hash → lookup → return
```

**Typical latency:** < 1ms for in-memory stores (Redis), < 10ms for disk-based (DynamoDB).

---

### Scalability

Easy to distribute across nodes using consistent hashing.

```
hash(key) % N → node

key "user:123" → hash → node 2
key "user:456" → hash → node 5
```

**Add more nodes → linear scaling.**

---

## 3. Data Model

### Keys

Usually strings, often with namespacing.

```
user:123
user:456:profile
session:abc123
cache:product:789
rate_limit:user:123:2024-01-15
```

**Best practices:**
- Use consistent naming: `entity:id` or `entity:id:attribute`
- Include version if schema changes: `user:v2:123`
- Keep keys short (stored with every value)

---

### Values

Opaque to the database. Can be:

```
String:     "Alice"
JSON:       {"name": "Alice", "age": 28}
Binary:     <image data>
Number:     42
Serialized: <protobuf/msgpack data>
```

**Database doesn't parse or index values.** It's just bytes.

---

## 4. Redis: The Most Popular Key-Value Store

Redis is an in-memory key-value store with rich data structures.

### Basic operations

```bash
# String
SET user:123 '{"name":"Alice"}'
GET user:123
DEL user:123

# With expiration (TTL)
SET session:abc '{"user_id":123}' EX 3600  # expires in 1 hour
TTL session:abc  # check remaining time

# Atomic increment
INCR page_views
INCRBY page_views 10

# Check existence
EXISTS user:123
```

---

### Data structures

**1. String (simple key-value):**

```bash
SET name "Alice"
GET name
```

**2. Hash (object with fields):**

```bash
HSET user:123 name "Alice" age 28 email "alice@example.com"
HGET user:123 name  # "Alice"
HGETALL user:123    # {name: "Alice", age: 28, email: "alice@example.com"}
HINCRBY user:123 age 1  # increment age
```

**3. List (ordered list):**

```bash
LPUSH queue:tasks "task1"  # push to left
LPUSH queue:tasks "task2"
RPOP queue:tasks           # pop from right → "task1" (FIFO queue)

LRANGE queue:tasks 0 -1    # get all items
```

**4. Set (unordered unique values):**

```bash
SADD tags:post:123 "database" "redis" "nosql"
SMEMBERS tags:post:123  # ["database", "redis", "nosql"]
SISMEMBER tags:post:123 "redis"  # 1 (true)

# Set operations
SINTER tags:post:123 tags:post:456  # intersection
SUNION tags:post:123 tags:post:456  # union
```

**5. Sorted Set (set with scores):**

```bash
ZADD leaderboard 1000 "player1"
ZADD leaderboard 1500 "player2"
ZADD leaderboard 1200 "player3"

ZREVRANGE leaderboard 0 9  # top 10 (highest scores)
ZRANK leaderboard "player1"  # rank of player1
ZINCRBY leaderboard 100 "player1"  # add 100 to score
```

---

## 5. Common Use Cases

### 1. Caching

Most common use case. Store expensive query results.

```python
def get_user(user_id):
    # Check cache
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)
    
    # Cache miss → query database
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # Store in cache with 1-hour TTL
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    
    return user
```

---

### 2. Session storage

Store user sessions.

```python
# Login
session_id = generate_session_id()
redis.setex(
    f"session:{session_id}",
    3600,  # 1 hour
    json.dumps({"user_id": 123, "role": "admin"})
)

# Validate session
def get_session(session_id):
    data = redis.get(f"session:{session_id}")
    if data:
        return json.loads(data)
    return None  # expired or invalid
```

---

### 3. Rate limiting

Track request counts per user.

```python
def is_rate_limited(user_id):
    key = f"rate_limit:{user_id}:{current_minute()}"
    count = redis.incr(key)
    
    if count == 1:
        redis.expire(key, 60)  # expire after 60 seconds
    
    return count > 100  # limit: 100 requests/minute
```

---

### 4. Leaderboards

Sorted sets are perfect for leaderboards.

```python
# Update score
redis.zincrby("leaderboard", 10, f"player:{player_id}")

# Get top 10
top_10 = redis.zrevrange("leaderboard", 0, 9, withscores=True)
# [(player:123, 1500), (player:456, 1200), ...]

# Get player rank
rank = redis.zrevrank("leaderboard", f"player:{player_id}")
```

---

### 5. Pub/Sub

Real-time messaging.

```python
# Publisher
redis.publish("notifications", json.dumps({
    "user_id": 123,
    "message": "New order received"
}))

# Subscriber
pubsub = redis.pubsub()
pubsub.subscribe("notifications")

for message in pubsub.listen():
    if message['type'] == 'message':
        data = json.loads(message['data'])
        handle_notification(data)
```

---

### 6. Distributed locks

Coordinate access across multiple servers.

```python
def acquire_lock(resource, timeout=10):
    lock_key = f"lock:{resource}"
    lock_value = generate_unique_id()
    
    # Try to acquire lock
    acquired = redis.set(lock_key, lock_value, nx=True, ex=timeout)
    
    if acquired:
        return lock_value  # lock acquired
    return None  # lock held by another process

def release_lock(resource, lock_value):
    lock_key = f"lock:{resource}"
    
    # Only release if we own the lock (check value)
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    redis.eval(script, 1, lock_key, lock_value)
```

---

## 6. Internals: How Key-Value Stores Work

### Hash table

Core data structure: hash map.

```
hash(key) → bucket → value

hash("user:123") = 42 → bucket[42] → value
```

**O(1) average case** for GET/PUT/DELETE.

---

### LSM Tree (for disk-based stores)

Log-Structured Merge Tree — optimized for writes.

```
Write path:
  1. Write to in-memory table (MemTable)
  2. When MemTable full → flush to disk (SSTable)
  3. Background compaction merges SSTables

Read path:
  1. Check MemTable
  2. Check SSTables (newest to oldest)
  3. Use bloom filters to skip SSTables that don't have key
```

**Used by:** RocksDB, LevelDB, Cassandra, DynamoDB.

**Benefit:** Sequential writes (very fast), good for write-heavy workloads.

---

### Bloom filters

Probabilistic data structure: "does this SSTable contain key X?"

```
Bloom filter says "no"  → definitely not in SSTable (skip it)
Bloom filter says "yes" → maybe in SSTable (check it)
```

**No false negatives, possible false positives.**

**Benefit:** Avoid expensive disk reads for keys that don't exist.

---

## 7. Persistence and Durability

### Redis persistence

**RDB (snapshot):**

```
Save entire dataset to disk periodically
  save 900 1    # save if 1 key changed in 900 seconds
  save 300 10   # save if 10 keys changed in 300 seconds
  save 60 10000 # save if 10000 keys changed in 60 seconds

Pros: Fast, compact
Cons: Can lose data between snapshots
```

**AOF (append-only file):**

```
Log every write operation
  appendonly yes
  appendfsync everysec  # fsync every second

Pros: Minimal data loss (at most 1 second)
Cons: Larger files, slower restarts
```

**Hybrid:** Use both RDB (for fast restarts) and AOF (for durability).

---

### DynamoDB durability

Writes replicated to 3 availability zones before acknowledging.

```
Client → Write → AZ1, AZ2, AZ3 → ACK
```

**Fully durable by default.**

---

## 8. Scaling Key-Value Stores

### Sharding (partitioning)

Distribute keys across multiple nodes.

```
hash(key) % N → node

N = 3 nodes:
  hash("user:123") % 3 = 0 → Node 0
  hash("user:456") % 3 = 1 → Node 1
  hash("user:789") % 3 = 2 → Node 2
```

**Problem:** Adding/removing nodes remaps most keys.

**Solution:** Consistent hashing (see Consistent Hashing doc).

---

### Replication

Each key replicated to multiple nodes for availability.

```
Replication factor = 3:
  key "user:123" → Node 0 (primary), Node 1 (replica), Node 2 (replica)
```

**Redis Cluster:** Automatic sharding + replication.

**DynamoDB:** Automatic replication across 3 AZs.

---

## 9. Redis vs DynamoDB vs Memcached

|Feature|Redis|DynamoDB|Memcached|
|---|---|---|---|
|Storage|In-memory|Disk-backed|In-memory|
|Persistence|RDB + AOF|Durable|None|
|Data structures|Rich (hash, list, set, sorted set)|Key-value only|Key-value only|
|Replication|Yes|Yes (3 AZs)|No (client-side)|
|Clustering|Redis Cluster|Automatic|Client-side|
|Max value size|512 MB|400 KB|1 MB|
|Latency|< 1ms|< 10ms|< 1ms|
|Use case|Caching, sessions, real-time|Serverless, managed, durable|Pure caching|

**Choose Redis when:** You need rich data structures, pub/sub, or persistence.

**Choose DynamoDB when:** You need fully managed, serverless, durable storage.

**Choose Memcached when:** You need pure caching with maximum simplicity.

---

## 10. Common Interview Questions + Answers

### Q: What's the difference between a key-value store and a document database?

> "Key-value stores are simpler — just a hash map with get/set/delete. You must know the exact key to retrieve data. The database doesn't parse or index values. Document databases like MongoDB store structured documents (JSON) and let you query by any field, not just the ID. You can do `db.users.find({age: {$gt: 25}})` in MongoDB, but you can't do that in a pure key-value store. Key-value is better for caching and simple lookups. Document databases are better when you need to query by multiple fields."

### Q: How does Redis achieve high performance?

> "Redis is in-memory, so all data is in RAM — no disk I/O for reads. It's single-threaded for command execution, which eliminates locking overhead and makes operations atomic. It uses efficient data structures (hash tables, skip lists for sorted sets). And it supports pipelining — send multiple commands at once without waiting for responses. For persistence, it uses background processes (fork) so writes don't block. Typical latency is sub-millisecond."

### Q: When would you use a key-value store vs a SQL database?

> "Use key-value for simple lookups by ID, caching, sessions, counters, and real-time data where you know the exact key. Use SQL when you have relational data with joins, need ACID transactions, or require ad-hoc queries. Many systems use both — SQL for transactional data (orders, users), Redis for caching and sessions. Key-value stores don't replace SQL; they complement it."

### Q: How do you handle cache invalidation in Redis?

> "Three approaches: TTL-based expiration (set expiry on every key, simple but data can be stale for up to TTL duration), event-driven invalidation (when data changes in the database, explicitly delete the cache key), or cache versioning (change the key when data changes, like `user:v2:123`). For most use cases, I'd use TTL with a reasonable duration (minutes to hours) as a safety net, plus explicit invalidation on writes for critical data."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Mention specific use cases

Don't just say "I'll use Redis." Say:

> "I'll use Redis for session storage with a 1-hour TTL, for caching user profiles with a 5-minute TTL, and for rate limiting with per-minute counters. I'll also use sorted sets for the leaderboard."

### ✅ Trick 2: Discuss persistence

Show you understand durability:

> "Redis is in-memory, so I'd enable AOF with appendfsync everysec for durability. This ensures we lose at most 1 second of data on crash. For pure caching where loss is acceptable, I'd use RDB snapshots only."

### ✅ Trick 3: Know the data structures

Redis isn't just strings:

> "For the shopping cart, I'd use a Redis hash — `HSET cart:123 item:1 2` to store item 1 with quantity 2. This is more efficient than storing the entire cart as a JSON string and allows atomic updates of individual items."

### ❌ Pitfall 1: Treating Redis as a primary database

Redis is usually a cache or session store, not the primary database. If you propose Redis, clarify:

> "Redis is the cache layer. The source of truth is PostgreSQL. If Redis fails, we read from the database."

### ❌ Pitfall 2: Not mentioning eviction policies

If Redis runs out of memory, what happens?

> "I'd set maxmemory-policy to allkeys-lru so Redis evicts the least recently used keys when memory is full. This prevents out-of-memory errors."

### ❌ Pitfall 3: Forgetting about single-threaded nature

Redis is single-threaded for commands. Expensive operations block:

> "I'd avoid expensive operations like `KEYS *` in production — it blocks all other commands. Instead, I'd use `SCAN` for iterating keys."

---

## 12. Quick Reference

```
Key-Value Store = distributed hash map

Operations:
  GET key       → retrieve value
  PUT key value → store value
  DELETE key    → remove key

Benefits:
  - O(1) operations (very fast)
  - Simple model (easy to scale)
  - Horizontal scalability (consistent hashing)

Use cases:
  - Caching (most common)
  - Session storage
  - Rate limiting
  - Leaderboards (sorted sets)
  - Distributed locks
  - Pub/Sub messaging

Redis data structures:
  String:      Simple key-value
  Hash:        Object with fields
  List:        Ordered list (queue)
  Set:         Unique values
  Sorted Set:  Set with scores (leaderboards)

Redis persistence:
  RDB:  Snapshot (fast, can lose data)
  AOF:  Append-only log (durable, slower)
  Hybrid: Both (recommended)

Internals:
  Hash table:    O(1) lookups
  LSM tree:      Write-optimized (RocksDB, DynamoDB)
  Bloom filter:  Skip SSTables without key

Scaling:
  Sharding:      Distribute keys across nodes
  Consistent hashing: Minimize remapping on node changes
  Replication:   Multiple copies for availability

Redis vs DynamoDB vs Memcached:
  Redis:      Rich data structures, persistence, pub/sub
  DynamoDB:   Managed, durable, serverless
  Memcached:  Pure caching, simple, multi-threaded

Common patterns:
  Cache-aside:   Check cache → miss → query DB → populate cache
  Write-through: Write to cache and DB synchronously
  TTL:           Auto-expire keys after duration
```
