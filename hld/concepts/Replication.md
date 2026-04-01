
## 1. What Is Replication?

Replication means keeping **copies of the same data on multiple nodes**.

Why:

- **Availability:** If one node fails, others can serve traffic
- **Read scalability:** Route reads to replicas to reduce primary load
- **Latency:** Place replicas geographically closer to users
- **Durability:** Data survives single-node hardware failure

Every replication strategy involves a fundamental tension: **how do you keep copies consistent while maintaining performance?**

---

## 2. Single-Leader Replication

The most common model. Also called **primary-replica**, **master-slave**, or **leader-follower**.

```
Writes ──▶ [Leader/Primary]
                  │
         ┌────────┼────────┐
         ▼        ▼        ▼
    [Replica 1] [Replica 2] [Replica 3]
         │        │        │
    Reads ◀──────────────────
```

**How it works:**

1. All writes go to the **leader**
2. Leader writes to its local storage and a **replication log** (WAL or binlog)
3. Replicas connect to leader and replay the replication log
4. Reads can go to leader (strong consistency) or replicas (may be stale)

**Used by:** PostgreSQL streaming replication, MySQL binlog replication, MongoDB (replica sets), Redis primary-replica

### Advantages

- Simple to reason about — one source of truth for writes
- Read scaling — add more replicas to scale reads
- Easy failover — promote a replica to leader on failure

### Disadvantages

- Leader is a write bottleneck — all writes serialized through one node
- If leader fails, failover takes time (seconds to minutes)
- Replicas can be stale (replication lag)

---

## 3. Multi-Leader Replication

Multiple nodes accept writes simultaneously. Each leader replicates to the others.

```
[Leader A] ◀──replication──▶ [Leader B]
     │                              │
     ▼                              ▼
[Replica A1]              [Replica B1]
     │                              │
  Writes ◀                       ▶ Writes
  (Region 1)                  (Region 2)
```

**Primary use case:** Multi-region active-active deployments. Users in US-East write to a leader in US-East; users in EU write to a leader in EU. Both leaders replicate asynchronously.

### Conflict resolution (the hard part)

When two leaders accept conflicting writes concurrently:

```
Leader A: User updates email → alice@gmail.com
Leader B: Same user updates email → alice@yahoo.com

Both replicate to each other — conflict!
```

**Conflict resolution strategies:**

1. **Last Write Wins (LWW):** Timestamp determines winner. Simple but loses data — the losing write is discarded.
    
    ```
    alice@gmail.com  (timestamp: 10:00:01)  ← wins
    alice@yahoo.com  (timestamp: 10:00:00)  ← discarded
    ```
    
    Risk: clock skew can cause the "wrong" write to win. NTP synchronisation is imperfect.
    
2. **Highest replica ID wins:** Deterministic but arbitrary — may discard valid writes.
    
3. **Merge:** Both values are kept and merged. Works for some data types (e.g., a shopping cart can be the union of both versions).
    
4. **Custom conflict resolution logic:** Application-level resolution on read (CouchDB) or on write. Most flexible but most complex.
    
5. **CRDTs (Conflict-free Replicated Data Types):** Data structures mathematically designed to merge without conflicts (counters, sets). Used in Riak, Redis Cluster.
    

### When to use multi-leader

- Multi-region active-active deployments
- Offline-capable clients (e.g., Google Docs — your device is a leader)
- High write availability requirements

### Examples

CouchDB, Amazon DynamoDB Global Tables, MySQL Group Replication, Cassandra (sort of)

---

## 4. Leaderless Replication

No designated leader. Any replica can accept writes. Clients (or a coordinator) send writes to multiple replicas.

```
Client ──▶ [Node A]  (write accepted)
       ──▶ [Node B]  (write accepted)
       ──▶ [Node C]  (write not reached — partitioned)
```

Based on Amazon Dynamo's design. Used by Cassandra, Riak, DynamoDB.

### Quorum reads and writes

The system is consistent if write quorum and read quorum overlap.

```
N = total replicas (typically 3)
W = nodes that must acknowledge a write
R = nodes queried for a read

Strong consistency:  W + R > N
  e.g. N=3, W=2, R=2  →  2+2 > 3 ✅ (read set and write set must overlap)

Eventual consistency: W + R ≤ N
  e.g. N=3, W=1, R=1  →  1+1 ≤ 3 (fast but potentially stale)
```

**Cassandra consistency levels:**

```
ONE:     W=1, R=1  (fastest, eventual consistency)
QUORUM:  W=2, R=2  (strong consistency for N=3)
ALL:     W=3, R=3  (slowest, maximum consistency)
LOCAL_QUORUM: Quorum within local DC only (lower cross-DC latency)
```

### Read repair

When a client reads from multiple replicas and finds stale data on one, it writes the fresh value back to the stale replica.

```
Read from Node A: {name: "Alice v2"}
Read from Node B: {name: "Alice v1"}  ← stale

Client detects discrepancy → writes v2 back to Node B (read repair)
```

### Anti-entropy

Background process continuously compares replicas and syncs differences. Uses Merkle trees to efficiently find divergences without comparing every value.

---

## 5. Synchronous vs Asynchronous Replication

### Synchronous replication

Leader waits for **at least one replica** to confirm before acknowledging the write to the client.

```
Client → Write → Leader → replicates → Replica 1 → ACK
                                          (waits for ACK)
                        → Leader ACKs client ← ACK received
```

**Pros:** At least one replica always has the latest data. If leader fails immediately after a write, replica has it. **Cons:** Write latency increases (must wait for replica). If replica is slow/unavailable, writes stall.

### Asynchronous replication

Leader acknowledges write immediately. Replication happens in the background.

```
Client → Write → Leader → ACKs client immediately
                        → replicates to replicas (async, later)
```

**Pros:** Low write latency. Leader doesn't block on replica availability. **Cons:** If leader fails before replication completes, acknowledged writes can be lost.

### Semi-synchronous (PostgreSQL default)

One replica is synchronous, others are asynchronous.

```
Leader → confirms write when sync replica ACKs
       → replicates to async replicas in background
```

Guarantees at least 2 copies of every committed write (leader + sync replica), while not waiting for all replicas.

**PostgreSQL config:**

```sql
synchronous_commit = on        -- wait for sync replica
synchronous_standby_names = 'replica1'
```

---

## 6. Replication Lag and Its Problems

Async replication means replicas are behind the leader. This lag causes subtle bugs.

### Problem 1: Reading your own writes (read-after-write consistency)

```
User updates profile photo → writes to leader
User refreshes page        → reads from replica (stale)
User sees OLD photo        → confusing!
```

**Solutions:**

- Read from leader for data the user just modified
- Track the replication position; read from replica only if it's caught up
- Route reads from a given user to the same replica consistently
- Read-after-write: for 1 minute after an update, always read from leader

### Problem 2: Monotonic reads

```
User A reads from Replica 1 (lag=0s):  sees update
User A reads from Replica 2 (lag=5s):  doesn't see update
User A appears to "go back in time"
```

**Solution:** Route each user to the same replica (using user_id hash). They may see stale data but never go backward.

### Problem 3: Consistent prefix reads

```
User A writes: "How are you?"    (seq=1)
User A writes: "I'm fine, thanks" (seq=2)

User B reads from Replica:
  Reads seq=2 first → "I'm fine, thanks"
  Reads seq=1 later → "How are you?"
  Conversation appears out of order
```

**Solution:** Causally related writes must be read in order. Ensure reads from the same partition see writes in order.

---

## 7. Consistency Models in Practice

```
Strongest:
  Linearizable     Every read returns the latest write globally.
                   Requires synchronous replication + strong quorum.

  Sequential       All nodes see operations in same order,
                   but not necessarily real-time.

  Causal           Causally related writes seen in order.
                   "I saw your message before I replied."

  Read-your-writes You always see your own writes.
                   Others may not see them yet.

  Monotonic reads  Once you see a value, you won't see an older value.

  Eventual         All replicas converge given no new writes.
                   Default for AP systems (Cassandra ONE).
Weakest
```

---

## 8. Failover and Leader Election

When a leader fails, a new leader must be elected. This is harder than it sounds.

### Automatic failover steps (single-leader)

```
1. Detect leader failure (heartbeat timeout, typically 30s)
2. Elect new leader (replica with most recent data, or voted by replicas)
3. Reconfigure clients/other replicas to use new leader
4. Old leader (if it comes back) must be demoted — "fencing"
```

### Problems

**Split-brain:** Old leader comes back and doesn't know it was replaced. Two nodes think they're leader simultaneously.

```
Leader A fails → network partition
Replica B elected as new leader
Network heals → Leader A comes back
Now: A thinks it's leader, B thinks it's leader → writes can diverge
```

**Solution:** Fencing — when the new leader is elected, it increments an **epoch number**. Writes are tagged with epoch. Storage layer rejects writes from a stale epoch. This ensures the old leader's writes are rejected even if it comes back.

**Raft consensus:** Modern systems use Raft to make leader election and log replication correct by design. Used in etcd, CockroachDB, TiDB.

```
Raft guarantees:
  - At most one leader per term
  - Leader has all committed entries
  - Log entries committed by majority = durable
```

---

## 9. Real-World Systems

|System|Replication Type|Notes|
|---|---|---|
|PostgreSQL|Single-leader|Streaming WAL replication, sync/async configurable|
|MySQL|Single-leader|Binlog replication; Group Replication for multi-primary|
|MongoDB|Single-leader|Replica sets, automatic failover via elections|
|Redis|Single-leader|Primary-replica; Redis Sentinel for automatic failover|
|Cassandra|Leaderless|Quorum-based, tunable consistency, no single leader|
|DynamoDB|Leaderless|Multi-AZ replication, eventual by default, consistent opt-in|
|Kafka|Single-leader per partition|ISR (in-sync replicas) determine quorum|
|etcd / Zookeeper|Raft/ZAB consensus|Strongly consistent, used for coordination|

---

## 10. Common Interview Questions + Answers

### Q: What is replication lag and how do you handle it?

> "Replication lag is the delay between a write being committed on the leader and being visible on replicas. For async replication, it's typically milliseconds under normal load but can spike to seconds under heavy load or network issues. Handling it: for read-after-write consistency, route reads for a user's own data to the leader for a short window after writes. For monotonic reads, pin each user to a replica. For critical reads (payments, inventory), always read from the leader."

### Q: How does Cassandra achieve high availability without a leader?

> "Cassandra uses leaderless replication with quorum-based reads and writes. For N=3, W=2, R=2: a write succeeds when 2 replicas ACK, and a read is satisfied by 2 replicas — their response sets must overlap, guaranteeing at least one replica has the latest write. For higher availability, you can drop to W=1 R=1, accepting eventual consistency. For multi-DC, LOCAL_QUORUM limits the quorum to one DC, avoiding cross-DC latency on every operation."

### Q: What is split-brain and how do you prevent it?

> "Split-brain is when a network partition causes two nodes to both believe they are the leader, leading to divergent writes. Prevention: use a quorum — a node can only become leader if it has acknowledgment from a majority of nodes. A minority partition can't reach quorum, so it won't elect a leader. Combined with fencing tokens (epoch numbers on writes), you ensure the old leader's writes are rejected by storage even if it comes back after partition heals."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Distinguish replication from sharding

These are orthogonal:

- **Replication:** Multiple copies of the same data for availability/read scale
- **Sharding:** Different data on different nodes for write scale

Most production systems use both: shard data across nodes, replicate each shard.

### ✅ Trick 2: Know the Kafka ISR model

In Kafka, the "in-sync replica set" (ISR) is the set of replicas that are caught up with the leader. A write is committed only when all ISR members acknowledge it. If a replica falls behind, it's removed from ISR. This is a specific replication design that comes up in Kafka-heavy interviews.

### ✅ Trick 3: Mention the CAP connection

Sync replication → CP (unavailable during partition rather than serving stale data). Async replication → AP (available but potentially stale).

### ❌ Pitfall 1: Assuming failover is instant

Automatic failover with Sentinel/Patroni/Raft typically takes 10-30 seconds. Design your client to handle this: connection retries with backoff, read-only mode during failover window.

### ❌ Pitfall 2: Forgetting about replication for Kafka

Kafka is often discussed in terms of producers/consumers, but it also has replication. Each partition has a leader replica and follower replicas. The `replication.factor` and `min.insync.replicas` settings are directly analogous to W and N in quorum-based systems.

---

## 12. Quick Reference

```
Types:
  Single-leader:   One write node, N read replicas
  Multi-leader:    Multiple write nodes (multi-region)
  Leaderless:      Any node accepts writes (Cassandra, Dynamo)

Sync vs Async:
  Synchronous:     Low RPO (no data loss), higher write latency
  Asynchronous:    Low write latency, possible data loss on failover
  Semi-sync:       One sync replica + rest async (PostgreSQL default)

Quorum (leaderless):
  N=replicas, W=write quorum, R=read quorum
  Strong: W + R > N     (e.g. W=2, R=2, N=3)
  Eventual: W + R ≤ N   (e.g. W=1, R=1, N=3)

Replication lag problems:
  Read-your-writes   → read from leader after your own writes
  Monotonic reads    → pin user to same replica
  Consistent prefix  → ensure ordered reads within partition

Failover:
  Automatic via: Sentinel (Redis), Patroni (Postgres), Raft (etcd)
  Split-brain prevention: quorum majority + fencing tokens

Key systems:
  PostgreSQL/MySQL → single-leader, WAL/binlog
  MongoDB          → single-leader, automatic elections
  Cassandra        → leaderless, tunable quorum
  Kafka            → single-leader per partition, ISR
```