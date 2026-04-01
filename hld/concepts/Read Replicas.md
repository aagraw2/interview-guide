## 1. What Are Read Replicas?

Read replicas are copies of a database that handle read queries, allowing the primary database to focus on writes.

```
Without read replicas:
  [Primary DB] ← All reads and writes
  Bottleneck at 10,000 QPS

With read replicas:
  [Primary DB] ← Writes only (1,000 QPS)
       ↓ replication
  [Replica 1] ← Reads (3,000 QPS)
  [Replica 2] ← Reads (3,000 QPS)
  [Replica 3] ← Reads (3,000 QPS)
  
  Total: 10,000 QPS handled
```

**Purpose:** Scale read throughput without scaling writes.

---

## 2. How Read Replicas Work

### Replication process

```
1. Write to primary:
   Client → INSERT INTO users VALUES (...)
   Primary → Writes to database
   Primary → Writes to replication log (WAL/binlog)

2. Replicas read log:
   Replica → Reads replication log from primary
   Replica → Applies changes to local database

3. Read from replica:
   Client → SELECT * FROM users WHERE id = 123
   Replica → Returns data (may be slightly stale)
```

---

### Asynchronous replication (most common)

```
Primary:
  1. Write to database
  2. Write to replication log
  3. Acknowledge to client immediately
  4. Replicas catch up asynchronously

Pros: Low write latency
Cons: Replication lag (replicas may be behind)
```

---

### Synchronous replication (rare for read replicas)

```
Primary:
  1. Write to database
  2. Write to replication log
  3. Wait for at least one replica to acknowledge
  4. Then acknowledge to client

Pros: No replication lag
Cons: Higher write latency, reduced availability
```

**Most read replicas use asynchronous replication.**

---

## 3. Replication Lag

### What is replication lag?

```
Time between write on primary and availability on replica

Example:
  10:00:00.000 - Write to primary
  10:00:00.100 - Available on replica
  
  Replication lag: 100ms
```

---

### Causes of replication lag

```
1. Network latency:
   Primary in US-East, replica in EU-West
   Cross-region network: 100-150ms

2. Replica overload:
   Replica handling too many read queries
   Can't keep up with replication

3. Large transactions:
   Primary writes 1GB of data
   Replica takes time to apply

4. Slow disk on replica:
   Replica has slower storage than primary
```

---

### Typical replication lag

```
Same datacenter:     1-10ms
Cross-region:        100-500ms
Overloaded replica:  Seconds to minutes
```

---

## 4. Problems Caused by Replication Lag

### Problem 1: Read-your-writes inconsistency

```
User updates profile:
  Client → Write to primary → "Email updated"
  Client → Redirect to profile page
  Client → Read from replica → Old email (replica lagging)
  
User sees old data immediately after update
```

**Solution:**

```
1. Read from primary for user's own data (short window after write)
2. Track write timestamp, only read from replica if caught up
3. Use session stickiness (same user → same replica)
```

---

### Problem 2: Monotonic reads violation

```
User reads from Replica 1 (lag: 0ms):
  Sees latest data

User reads from Replica 2 (lag: 5s):
  Sees old data
  
User appears to "go back in time"
```

**Solution:**

```
Route user to same replica consistently (session stickiness)
User may see stale data, but never goes backward
```

---

### Problem 3: Causality violation

```
User A posts: "What's the weather?"
User B replies: "It's sunny!"

User C reads from replica:
  Sees reply: "It's sunny!"
  Doesn't see original post (not replicated yet)
  
Conversation appears out of order
```

**Solution:**

```
Ensure causally related writes are read in order
Use version vectors or logical timestamps
```

---

## 5. Routing Strategies

### Strategy 1: Application-level routing

```python
def get_user(user_id, read_own_writes=False):
    if read_own_writes:
        # Read from primary
        return primary_db.query("SELECT * FROM users WHERE id = ?", user_id)
    else:
        # Read from replica
        return replica_db.query("SELECT * FROM users WHERE id = ?", user_id)

# After user updates profile
update_user(user_id, new_email)
user = get_user(user_id, read_own_writes=True)  # Read from primary
```

---

### Strategy 2: Load balancer routing

```
Read queries → Load Balancer → Round-robin to replicas
Write queries → Load Balancer → Primary

Load balancer configuration:
  /api/users (GET)  → Replicas
  /api/users (POST) → Primary
```

---

### Strategy 3: Database proxy

```
Client → ProxySQL → Routes reads to replicas, writes to primary

ProxySQL configuration:
  - Read queries: distribute across replicas
  - Write queries: route to primary
  - Read-after-write: route to primary for 1 second
```

---

### Strategy 4: CQRS (Command Query Responsibility Segregation)

```
Write model:
  Commands → Primary DB

Read model:
  Queries → Replicas (or separate read-optimized store)

Separate read and write paths entirely
```

---

## 6. Read Replica Topologies

### Single primary, multiple replicas

```
[Primary] ← Writes
    ↓
    ├→ [Replica 1] ← Reads
    ├→ [Replica 2] ← Reads
    └→ [Replica 3] ← Reads

Most common topology
```

---

### Cascading replication

```
[Primary] ← Writes
    ↓
[Replica 1] ← Some reads
    ↓
    ├→ [Replica 2] ← Reads
    └→ [Replica 3] ← Reads

Reduces load on primary
Increases replication lag for Replica 2 and 3
```

---

### Multi-region replication

```
US-East:
  [Primary] ← Writes
      ↓
  [Replica 1] ← Reads (US users)

EU-West:
  [Replica 2] ← Reads (EU users)

APAC:
  [Replica 3] ← Reads (APAC users)

Reduces latency for global users
Higher replication lag for distant replicas
```

---

## 7. Failover

### Promoting a replica to primary

```
Scenario: Primary fails

1. Detect failure (health checks, heartbeat)
2. Choose replica to promote (most up-to-date)
3. Promote replica to primary
4. Reconfigure other replicas to replicate from new primary
5. Update application to write to new primary

Downtime: 30 seconds to 5 minutes (depending on automation)
```

---

### Automatic failover

```
Tools:
  - PostgreSQL: Patroni, repmgr
  - MySQL: MySQL Router, Orchestrator
  - AWS RDS: Automatic failover to standby

Process:
  1. Monitor primary health
  2. On failure, automatically promote replica
  3. Update DNS or connection string
  4. Application reconnects to new primary
```

---

## 8. Monitoring Read Replicas

### Key metrics

```
Replication lag:
  - Time behind primary
  - Alert if > 1 second (same DC) or > 5 seconds (cross-region)

Replica health:
  - Is replication running?
  - Any replication errors?

Query load:
  - QPS per replica
  - CPU, memory, disk I/O

Connection count:
  - Number of active connections
  - Alert if approaching max_connections
```

---

### PostgreSQL monitoring

```sql
-- Check replication lag
SELECT 
  client_addr,
  state,
  pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
  EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag_seconds
FROM pg_stat_replication;
```

---

### MySQL monitoring

```sql
-- Check replication lag
SHOW SLAVE STATUS\G

-- Key fields:
-- Seconds_Behind_Master: replication lag in seconds
-- Slave_IO_Running: Is replication I/O thread running?
-- Slave_SQL_Running: Is replication SQL thread running?
```

---

## 9. Best Practices

### 1. Use read replicas for read-heavy workloads

```
Good fit:
  - 90%+ reads, 10% writes
  - Analytics queries
  - Reporting dashboards
  - Search functionality

Not a good fit:
  - Write-heavy workloads (replicas don't help)
  - Strong consistency required (replication lag)
```

---

### 2. Monitor replication lag

```
Set alerts:
  - Warning: lag > 1 second
  - Critical: lag > 5 seconds

Investigate causes:
  - Replica overloaded? Add more replicas
  - Network issues? Check connectivity
  - Large transactions? Optimize queries
```

---

### 3. Handle read-your-writes

```
Options:
  1. Read from primary for user's own data (1 second after write)
  2. Check replica lag before reading
  3. Use session stickiness (same user → same replica)
  4. Include write timestamp in response, check replica caught up
```

---

### 4. Use connection pooling

```
Without pooling:
  Each request → new connection to replica
  Connection overhead, max_connections limit

With pooling (PgBouncer, ProxySQL):
  Pool of connections shared across requests
  Reduces overhead, handles more concurrent users
```

---

### 5. Don't overload replicas

```
Each replica has capacity limit:
  - CPU, memory, disk I/O
  - Connection limit

If replica overloaded:
  - Replication lag increases
  - Query performance degrades
  - Add more replicas
```

---

## 10. Common Interview Questions + Answers

### Q: What are read replicas and when would you use them?

> "Read replicas are copies of a database that handle read queries, allowing the primary to focus on writes. They're created through asynchronous replication from the primary. I'd use them for read-heavy workloads where reads outnumber writes significantly, like 90% reads and 10% writes. Examples include analytics dashboards, reporting systems, and search functionality. They don't help with write-heavy workloads since all writes still go to the primary. The tradeoff is replication lag — replicas may be slightly behind the primary, so you need to handle eventual consistency."

### Q: What is replication lag and how do you handle it?

> "Replication lag is the delay between a write on the primary and when it's available on a replica. It's typically 1-10ms in the same datacenter but can be 100-500ms cross-region. This causes problems like read-your-writes inconsistency — a user updates their profile but immediately sees old data when reading from a replica. To handle this, I'd read from the primary for a user's own data for a short window after writes, use session stickiness to route users to the same replica consistently, or check the replica's lag before reading and wait if it's too far behind."

### Q: How do read replicas differ from a standby database?

> "Read replicas actively serve read traffic and there can be multiple replicas. A standby database is for high availability — it's a hot backup that takes over if the primary fails, but typically doesn't serve traffic. Read replicas use asynchronous replication for low latency, while standbys often use synchronous replication for zero data loss. When the primary fails, a standby is promoted automatically. With read replicas, you can promote one to primary, but it's usually a manual process. You can have both — a synchronous standby for HA and asynchronous read replicas for scaling reads."

### Q: How would you scale a database that's hitting read capacity limits?

> "First, I'd add read replicas to distribute read traffic. If the workload is 90% reads, adding 3 read replicas could handle 4x the traffic. I'd also add caching with Redis in front of the database to reduce database load for frequently accessed data. For the application, I'd implement connection pooling to reduce connection overhead. If specific queries are slow, I'd optimize them with indexes. Only if these approaches are exhausted would I consider sharding, which is much more complex. For most applications, read replicas plus caching can scale reads to very high levels."

---

## 11. Quick Reference

```
Read Replicas = copies of database that handle read queries

How it works:
  1. Writes go to primary
  2. Primary writes to replication log (WAL/binlog)
  3. Replicas read log and apply changes
  4. Reads go to replicas

Replication:
  Asynchronous (common): Low latency, replication lag
  Synchronous (rare): No lag, higher latency

Replication lag:
  Same DC: 1-10ms
  Cross-region: 100-500ms
  Causes: Network latency, replica overload, large transactions

Problems from lag:
  1. Read-your-writes: User sees old data after update
  2. Monotonic reads: User goes "back in time"
  3. Causality: Related events out of order

Solutions:
  - Read from primary for user's own data (short window)
  - Session stickiness (same user → same replica)
  - Check replica lag before reading

Routing strategies:
  1. Application-level (code decides)
  2. Load balancer (route by endpoint)
  3. Database proxy (ProxySQL, PgBouncer)
  4. CQRS (separate read/write models)

Topologies:
  Single primary + multiple replicas (most common)
  Cascading (replica → replica)
  Multi-region (replicas in different regions)

Failover:
  1. Detect primary failure
  2. Promote replica to primary
  3. Reconfigure other replicas
  4. Update application
  Downtime: 30s - 5 minutes

Monitoring:
  - Replication lag (alert if > 1s same DC, > 5s cross-region)
  - Replica health (replication running?)
  - Query load (QPS, CPU, memory)
  - Connection count

Best practices:
  ✅ Use for read-heavy workloads (90%+ reads)
  ✅ Monitor replication lag
  ✅ Handle read-your-writes
  ✅ Use connection pooling
  ✅ Don't overload replicas

When to use:
  ✅ Read-heavy workloads
  ✅ Analytics, reporting
  ✅ Global users (multi-region replicas)
  ❌ Write-heavy workloads (doesn't help)
  ❌ Strong consistency required (lag is issue)

Key insight: Read replicas scale reads, not writes
```
