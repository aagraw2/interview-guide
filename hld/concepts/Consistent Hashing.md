
## 1. The Problem It Solves

When you have a distributed cache or database with multiple nodes, you need a way to decide which node stores a given key. This is called **key-to-node mapping**.

Requirements of a good mapping strategy:

- Keys are spread evenly across nodes
- When a node is added or removed, minimal data needs to move
- The mapping is deterministic (same key always goes to the same node)

---

## 2. Naive Approach and Why It Fails

The simplest approach is **modulo hashing**:

```
node = hash(key) % N       (N = number of nodes)
```

**Example with 3 nodes:**

```
hash("user_1") % 3 = 0  → Node 0
hash("user_2") % 3 = 1  → Node 1
hash("user_3") % 3 = 2  → Node 2
```

This works fine until you add or remove a node.

**Add a 4th node (N changes from 3 to 4):**

```
hash("user_1") % 4 = 3  → Node 3  (was Node 0)
hash("user_2") % 4 = 2  → Node 2  (was Node 1)
hash("user_3") % 4 = 3  → Node 3  (was Node 2)
```

Almost every key maps to a different node. In a cache, this means a **near-100% cache miss** — all data is in the "wrong" place. In a database, it means a massive migration.

With N nodes, adding 1 node remaps `N/(N+1)` of all keys — for 10 nodes, that's ~90% of data moving.

**Consistent hashing solves this.** When a node is added/removed, only `K/N` keys move on average (K = total keys, N = nodes). For 1 million keys on 10 nodes, only ~100K keys need to move when adding one node.

---

## 3. How Consistent Hashing Works

### The Ring

Imagine a circular number line (ring) from 0 to 2³²-1 (or 0 to 360 degrees conceptually).

```
                    0 / 2^32
                   /
              315 /  45
               \ /
         270 ---+--- 90
               / \
              /   \
            225   135
                180
```

**Step 1:** Hash each node to a position on the ring.

```
hash("NodeA") = 45   → Node A sits at position 45
hash("NodeB") = 135  → Node B sits at position 135
hash("NodeC") = 225  → Node C sits at position 225
hash("NodeD") = 315  → Node D sits at position 315
```

**Step 2:** Hash each key to a position on the ring.

```
hash("user_1") = 90   → position 90
hash("user_2") = 180  → position 180
hash("user_3") = 270  → position 270
```

**Step 3:** Each key is assigned to the **first node clockwise** from its position.

```
                    0 / 2^32
                   /
         NodeD  315     45  NodeA
               \ /
         270 ---+--- 90
         |      / \      |
    user_3     /   \   user_1
             225   135
           NodeC   NodeB
             |       |
           user_3  user_2
```

```
user_1 at 90  → first node clockwise → NodeB (135)
user_2 at 180 → first node clockwise → NodeC (225)
user_3 at 270 → first node clockwise → NodeD (315)
```

### Adding a Node

Add NodeE at position 75:

```
hash("NodeE") = 75   → NodeE sits at position 75
```

Now `user_1` (at 90) maps to NodeE (75 → 90, clockwise next is NodeE).

**Only the keys between NodeA (45) and NodeE (75) need to move.** Everything else is unaffected.

```
Before:  user_1 → NodeB
After:   user_1 → NodeE   ← only this changed
```

### Removing a Node

If NodeB (135) is removed, only the keys that were assigned to NodeB need to move — they now go to NodeC (225). Nothing else changes.

---

## 4. Virtual Nodes

Basic consistent hashing has two problems:

1. **Uneven distribution** — nodes may end up clustered on the ring, leaving large arcs with many keys
2. **Heterogeneous capacity** — all nodes are treated equally even if some are more powerful

### Solution: Virtual Nodes (vnodes)

Instead of each physical node having one position on the ring, it gets **many positions** (virtual nodes).

```
Physical Node A → 3 virtual nodes: vA1, vA2, vA3
Physical Node B → 3 virtual nodes: vB1, vB2, vB3
Physical Node C → 3 virtual nodes: vC1, vC2, vC3
```

```
Ring:  vB2  vA1  vC3  vB1  vA3  vC1  vA2  vB3  vC2
        |    |    |    |    |    |    |    |    |
       B    A    C    B    A    C    A    B    C
```

Each physical node's virtual nodes are spread across the ring, ensuring roughly even distribution regardless of where physical nodes happen to hash to.

**For heterogeneous capacity:** Assign more vnodes to more powerful servers.

```
Node A (32 GB RAM) → 150 virtual nodes
Node B (16 GB RAM) → 75 virtual nodes
Node C (8 GB RAM)  → 37 virtual nodes
```

Node A will handle ~2x the load of Node B, which handles ~2x of Node C.

**Cassandra uses 256 vnodes per node by default.**

### Tradeoff of virtual nodes

- More vnodes = better distribution, but more metadata to track
- On node failure, data redistributes to more nodes (both pro and con — spreading load is good, but more nodes need to respond)

---

## 5. Data Replication on the Ring

In most systems, you don't store data on just one node. Cassandra and Dynamo replicate to the next N clockwise nodes.

**Replication factor = 3:**

```
Ring positions: NodeA(45) → NodeB(135) → NodeC(225) → NodeD(315)

key "user_1" hashes to position 90 → primary is NodeB
Replicas: next 2 clockwise → NodeC, NodeD
```

So "user_1" is stored on NodeB, NodeC, NodeD.

**If NodeB fails:** Reads and writes automatically route to NodeC (next clockwise node with the data).

---

## 6. Real-World Usage

|System|How It Uses Consistent Hashing|
|---|---|
|Amazon DynamoDB|Partitions data across storage nodes|
|Apache Cassandra|Ring-based partitioning with vnodes (256 per node)|
|Amazon S3|Key-to-partition mapping|
|Memcached (client-side)|Consistent hashing client routes keys to cache nodes|
|Nginx|Upstream consistent hashing for sticky sessions|
|Chord DHT|Foundational academic algorithm|

---

## 7. Common Interview Questions + Answers

### Q: Why not just use modulo hashing?

> "Modulo hashing remaps nearly all keys when N changes, making scaling events catastrophic — essentially a cache stampede or mass data migration. Consistent hashing limits remapping to K/N keys on average, so adding one node to a 10-node cluster only moves ~10% of data."

### Q: What's the time complexity of a lookup?

> "Lookup is O(log N) where N is the number of nodes (or virtual nodes) — implemented as a binary search on the sorted ring positions. In practice, it's negligible overhead."

### Q: How does consistent hashing handle hotspots?

> "If a key is accessed far more than others (a 'hot key'), consistent hashing doesn't solve that on its own — it just routes that key to one node. Solutions include: caching the hot key in multiple nodes and randomly picking one for reads, adding a random suffix to the key to spread writes, or handling it at the application layer with read replicas."

### Q: Consistent hashing vs rendezvous hashing?

> "Both solve the same problem. Rendezvous (highest random weight) hashing picks the node with the highest score of hash(key + node) — no ring needed, simpler conceptually. But it's O(N) per lookup vs O(log N) for consistent hashing. Consistent hashing wins at scale."

---

## 8. Interview Tricks & Pitfalls

### ✅ Trick 1: Lead with the problem, not the solution

Start by explaining why modulo hashing fails before explaining the ring. Interviewers want to see problem-solution thinking, not just a definition recitation.

### ✅ Trick 2: Mention virtual nodes immediately after the basic ring

Basic consistent hashing without vnodes has well-known weaknesses (uneven distribution). Bringing up vnodes immediately shows you know the practical version, not just the textbook concept.

### ✅ Trick 3: Connect to a real system you know

> "Cassandra uses consistent hashing with 256 virtual nodes per node by default. This is why adding a node to a Cassandra cluster causes roughly 1/N of data to stream to the new node, and you typically run `nodetool cleanup` afterward."

### ❌ Pitfall 1: Forgetting the "clockwise" direction

The rule is always the **first node clockwise** (or a specific direction). If you draw the ring, be consistent about direction.

### ❌ Pitfall 2: Confusing consistent hashing with consistent replication

Consistent hashing is about key-to-node assignment. Replication factor is a separate concern layered on top. They work together but are independent concepts.

### ❌ Pitfall 3: Not knowing when NOT to use it

Consistent hashing is for distributing data/load across homogeneous nodes. You wouldn't use it for:

- Routing to specialised services (use a service registry instead)
- Very small node counts (3 nodes — just use modulo, overhead isn't worth it)

---

## 9. Quick Reference

```
Problem:  modulo hashing remaps ~N/(N+1) keys on every scale event

Solution: hash ring — only K/N keys move when a node changes

Steps:
  1. Hash nodes to ring positions
  2. Hash keys to ring positions
  3. Assign key to first clockwise node

Virtual nodes:
  Each physical node = multiple ring positions
  Cassandra default = 256 vnodes/node
  More powerful nodes = more vnodes

Replication:
  key → primary (first clockwise node)
  replicas → next N-1 clockwise nodes

Complexity:
  Lookup: O(log N) binary search on sorted positions
  Remapping: O(K/N) keys on node add/remove

Real systems: DynamoDB, Cassandra, Memcached, S3
```