## 1. What Is the CAP Theorem?

The CAP theorem states that a distributed system can guarantee at most **two** of the following three properties simultaneously:

- **Consistency (C):** Every read receives the most recent write or an error
- **Availability (A):** Every request receives a response (without guarantee it's the most recent write)
- **Partition Tolerance (P):** The system continues to operate despite network partitions

```
        Consistency
           /  \
          /    \
         /  CA  \
        /________\
       /\        /\
      /  \  CP  /  \
     / AP \    / CA \
    /______\  /______\
Availability    Partition
                Tolerance
```

**The reality:** Network partitions are inevitable in distributed systems. You can't avoid them. So in practice, you're choosing between **CP** (consistency + partition tolerance) or **AP** (availability + partition tolerance).

---

## 2. Understanding Each Property

### Consistency

All nodes see the same data at the same time. After a write completes, all subsequent reads return that value.

```
Client writes X=5 to Node A
→ All nodes (A, B, C) must show X=5 before acknowledging
→ Any read from any node returns X=5
```

This is **strong consistency** or **linearizability** — the strongest consistency model.

### Availability

Every request to a non-failing node must receive a response, even if some nodes are down or partitioned.

```
Client requests data from Node A
→ Node A must respond (even if it can't reach other nodes)
→ Response might be stale, but it responds
```

### Partition Tolerance

The system continues to function when network partitions occur — when some nodes can't communicate with others.

```
Network split:
  [Node A, Node B]  |  [Node C]
       ↑                  ↑
    Can't communicate
```

The system must handle this gracefully rather than failing entirely.

---

## 3. The Tradeoff: CP vs AP

When a network partition occurs, you must choose:

### CP — Consistency + Partition Tolerance

**Decision:** Reject requests to maintain consistency.

```
Network partition occurs:
  [Node A, Node B]  |  [Node C]

Node C can't reach majority → refuses writes
Node C returns error: "Service unavailable"
```

**Behavior:**
- Writes to minority partition fail
- Reads from minority partition may fail or return stale data
- System sacrifices availability to maintain consistency

**Use when:** Correctness is more important than availability. Financial transactions, inventory management, booking systems.

**Examples:** HBase, MongoDB (with majority read/write concern), etcd, ZooKeeper, Google Spanner

---

### AP — Availability + Partition Tolerance

**Decision:** Accept requests even during partition, allowing temporary inconsistency.

```
Network partition occurs:
  [Node A, Node B]  |  [Node C]

Both sides accept writes independently
→ Data diverges temporarily
→ Reconciled later when partition heals
```

**Behavior:**
- All nodes accept reads and writes
- Data may be inconsistent across nodes
- Eventually consistent after partition heals

**Use when:** Availability is more important than immediate consistency. Social media feeds, shopping carts, analytics, caching.

**Examples:** Cassandra, DynamoDB, Riak, CouchDB

---

## 4. Real-World Systems and CAP

### MongoDB

**Default:** CP

```
Replica set with 3 nodes:
  Primary + 2 secondaries

Network partition:
  Primary can't reach majority → steps down
  No primary → writes fail (unavailable)
  Maintains consistency
```

**Can be tuned to AP:**

```
Write concern: w=1 (write to primary only, don't wait for replicas)
Read preference: secondary (read from any replica)
→ More available, less consistent
```

### Cassandra

**Default:** AP

```
3 replicas, write with consistency level ONE:
  Write succeeds if 1 replica acknowledges
  Other replicas updated asynchronously
  Always available, eventually consistent
```

**Can be tuned to CP:**

```
Write: QUORUM (2 of 3 replicas)
Read: QUORUM (2 of 3 replicas)
→ W + R > N ensures consistency
→ Less available during partitions
```

### DynamoDB

**Default:** AP (eventually consistent reads)

```
Write to 3 replicas asynchronously
Default read: eventually consistent (may be stale)
```

**Can request CP:**

```
Strongly consistent read: reads from leader
→ Higher latency, less available during partition
```

### PostgreSQL (single node)

**CA** — Consistent and Available, but not partition tolerant

```
Single node: no partitions possible
All reads/writes go to one node
→ Consistent and available
→ But if node fails, entire system down
```

**With replication:** Becomes CP (with synchronous replication) or AP (with asynchronous replication)

---

## 5. The "CA" Myth

You'll sometimes see systems labeled "CA" — consistent and available but not partition tolerant.

**The truth:** CA systems don't exist in distributed systems. Network partitions are a physical reality, not a choice.

```
"CA" really means:
  - Single-node system (no distribution)
  - Or: distributed system that hasn't decided what to do during partition
```

Traditional single-node databases (PostgreSQL, MySQL) are CA only because they're not distributed. Once you add replication, you're forced to choose CP or AP.

---

## 6. Consistency Models Beyond CAP

CAP's "consistency" is actually **linearizability** — the strongest model. Real systems offer a spectrum:

```
Strongest:
  Linearizable       Every operation appears instantaneous and in real-time order
                     (CAP's "C")

  Sequential         Operations appear in some total order, but not necessarily real-time

  Causal             Causally related operations seen in order
                     "I saw your message before I replied"

  Read-your-writes   You see your own writes immediately

  Monotonic reads    Once you see a value, you never see an older one

  Eventual           All replicas converge eventually
                     (CAP's "A" often implies this)
Weakest
```

Most AP systems provide eventual consistency, not zero consistency.

---

## 7. PACELC — The Extended CAP

CAP only describes behavior during partitions. **PACELC** extends it:

```
If Partition:
  Choose Availability or Consistency (CAP)
Else (no partition):
  Choose Latency or Consistency
```

**Examples:**

```
DynamoDB:
  Partition: AP (available, eventually consistent)
  Else: EL (low latency, eventually consistent by default)
  → PA/EL

MongoDB:
  Partition: CP (consistent, unavailable if no majority)
  Else: EC (consistent, higher latency for majority writes)
  → PC/EC

Cassandra (tunable):
  With QUORUM: PC/EC
  With ONE: PA/EL
```

This explains why even without partitions, some systems are faster (eventual consistency) and others slower (strong consistency).

---

## 8. Practical Implications

### For system design interviews

When choosing a database, explicitly state the CAP tradeoff:

```
"For the payment service, I'll use PostgreSQL with synchronous replication
to a standby. This is CP — if the primary fails and the standby isn't
reachable, writes fail rather than risk inconsistency. Payments require
strong consistency."

"For the activity feed, I'll use Cassandra with consistency level ONE.
This is AP — the feed stays available during network issues, and eventual
consistency is acceptable. Users don't need to see every post instantly."
```

### For architecture decisions

**CP systems:**
- Banking transactions
- Inventory management (prevent overselling)
- Seat reservations
- Distributed locks
- Configuration management

**AP systems:**
- Social media feeds
- Shopping carts
- User profiles
- Analytics events
- Caching layers
- DNS

---

## 9. Common Misconceptions

### Misconception 1: "NoSQL is AP, SQL is CP"

**False.** Cassandra (NoSQL) can be CP with QUORUM. PostgreSQL with async replication is AP.

The data model (SQL vs NoSQL) is independent of CAP positioning.

### Misconception 2: "AP means no consistency"

**False.** AP means eventual consistency, not no consistency. After the partition heals, replicas converge. Most AP systems also offer tunable consistency (Cassandra's QUORUM, DynamoDB's strongly consistent reads).

### Misconception 3: "You can only pick two"

**More accurate:** You must tolerate partitions (P is mandatory). During a partition, choose C or A. When there's no partition, you can have both C and A, but there's a latency tradeoff (PACELC).

---

## 10. Interview Questions + Answers

### Q: Explain the CAP theorem and give examples of CP and AP systems.

> "CAP says a distributed system can't simultaneously guarantee consistency, availability, and partition tolerance. Since network partitions are inevitable, you're really choosing between CP and AP. CP systems like MongoDB and HBase prioritize consistency — during a partition, minority nodes refuse requests to avoid serving stale data. AP systems like Cassandra and DynamoDB prioritize availability — all nodes accept requests during a partition, accepting temporary inconsistency that's resolved later."

### Q: Is PostgreSQL CP or AP?

> "A single PostgreSQL node is neither — it's not distributed, so partitions don't apply. With replication: synchronous replication makes it CP (writes fail if the replica is unreachable), while asynchronous replication makes it AP (writes succeed even if replicas are behind, accepting eventual consistency)."

### Q: How does Cassandra achieve high availability?

> "Cassandra is AP by default. It uses leaderless replication with tunable consistency. With consistency level ONE, a write succeeds when any single replica acknowledges it, and reads come from any replica. During a network partition, all nodes continue accepting reads and writes independently. When the partition heals, anti-entropy repair and read repair reconcile differences. For stronger consistency, you can use QUORUM, which makes it CP."

### Q: When would you choose a CP system over an AP system?

> "Choose CP when correctness is non-negotiable and temporary unavailability is acceptable. Examples: financial transactions (can't have duplicate charges or lost money), inventory systems (can't oversell), booking systems (can't double-book a seat). Choose AP when availability matters more than immediate consistency: social feeds, shopping carts, user profiles, analytics. The key question is: what's worse — being temporarily unavailable or temporarily inconsistent?"

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Connect CAP to specific design decisions

Don't just recite the theorem. Apply it:

> "For the order service, I need CP because we can't risk double-charging a customer. I'll use PostgreSQL with synchronous replication. For the recommendation service, I need AP because stale recommendations are fine but the service must always respond. I'll use Cassandra with consistency level ONE."

### ✅ Trick 2: Mention tunability

Modern systems aren't purely CP or AP:

> "Cassandra is AP by default, but you can configure QUORUM reads and writes to get CP behavior. This flexibility lets you choose per-operation: use QUORUM for critical writes, ONE for high-throughput logging."

### ✅ Trick 3: Bring up PACELC

Shows deeper understanding:

> "Even without partitions, there's a latency-consistency tradeoff. DynamoDB's eventually consistent reads are faster than strongly consistent reads because they don't require coordination. That's the PACELC 'else' case."

### ❌ Pitfall 1: Saying "I'll use a CA system"

CA doesn't exist in distributed systems. If you say this, the interviewer knows you don't understand CAP.

### ❌ Pitfall 2: Treating CAP as absolute

Real systems are tunable. Don't say "Cassandra is AP" without acknowledging you can configure it for CP behavior.

### ❌ Pitfall 3: Forgetting that P is mandatory

Don't present it as "pick any two." The choice is always "CP or AP" because partitions will happen.

---

## 12. Quick Reference

```
CAP Theorem:
  C = Consistency (linearizability)
  A = Availability (every request gets a response)
  P = Partition tolerance (works despite network splits)

Reality: P is mandatory → choose C or A during partition

CP (Consistency + Partition Tolerance):
  Behavior: Reject requests during partition to stay consistent
  Use for: Financial transactions, inventory, bookings
  Examples: MongoDB (default), HBase, etcd, ZooKeeper, Spanner

AP (Availability + Partition Tolerance):
  Behavior: Accept requests during partition, eventual consistency
  Use for: Social feeds, carts, profiles, analytics
  Examples: Cassandra (default), DynamoDB, Riak, CouchDB

Tunable systems:
  Cassandra: AP with ONE, CP with QUORUM
  DynamoDB: AP with eventual reads, CP with consistent reads
  MongoDB: CP with majority, AP with w=1

PACELC extension:
  If Partition: choose A or C
  Else: choose Latency or Consistency

Key insight: Most systems are tunable, not fixed CP or AP
```
