## 1. What is Quorum?

A quorum is the minimum number of nodes that must agree on an operation for it to succeed in a distributed system, ensuring consistency despite failures.

```
Without Quorum:
  Write to 1 node → Success
  Node fails → Data lost

With Quorum:
  Write to 3 nodes → Need 2 to acknowledge → Success
  1 node fails → Still have 2 copies → Data safe
```

**Key principle:** Majority agreement ensures consistency and availability despite failures.

---

## 2. Core Concepts

### Quorum Size

Minimum nodes needed for agreement.

```
N = Total nodes
W = Write quorum (nodes that must acknowledge write)
R = Read quorum (nodes that must respond to read)

Common configuration:
  N = 3, W = 2, R = 2
  
  Majority quorum: W + R > N
  Ensures read sees latest write
```

### Read Quorum

Minimum nodes that must respond to a read.

```
N = 3 nodes
R = 2 (read quorum)

Read process:
  1. Send read to all 3 nodes
  2. Wait for 2 responses
  3. Return latest value (by timestamp/version)

If 1 node is down:
  Still get 2 responses → Read succeeds
```

### Write Quorum

Minimum nodes that must acknowledge a write.

```
N = 3 nodes
W = 2 (write quorum)

Write process:
  1. Send write to all 3 nodes
  2. Wait for 2 acknowledgments
  3. Return success

If 1 node is down:
  Still get 2 acks → Write succeeds
```

---

## 3. Quorum Configurations

### Majority Quorum (W + R > N)

```
N = 3, W = 2, R = 2

Guarantees:
  ✅ Read sees latest write (strong consistency)
  ✅ Tolerates 1 node failure

Example:
  Write to nodes A, B (W=2)
  Read from nodes B, C (R=2)
  Node B has latest value → Read succeeds
```

### Read-Heavy Quorum (W = N, R = 1)

```
N = 3, W = 3, R = 1

Guarantees:
  ✅ Fast reads (only 1 node)
  ❌ Slow writes (all nodes)
  ❌ No fault tolerance for writes

Use case: Read-heavy workloads, rare writes
```

### Write-Heavy Quorum (W = 1, R = N)

```
N = 3, W = 1, R = 3

Guarantees:
  ✅ Fast writes (only 1 node)
  ❌ Slow reads (all nodes)
  ❌ No fault tolerance for reads

Use case: Write-heavy workloads, rare reads
```

### Eventual Consistency (W = 1, R = 1)

```
N = 3, W = 1, R = 1

Guarantees:
  ✅ Fast reads and writes
  ❌ No consistency guarantee
  ❌ May read stale data

Use case: High availability, eventual consistency acceptable
```

---

## 4. Quorum in Distributed Systems

### Dynamo-Style (Cassandra, DynamoDB)

```
Tunable consistency:
  - Choose N, W, R per request
  - Trade consistency for availability

Example (Cassandra):
  CONSISTENCY QUORUM;
  INSERT INTO users (id, name) VALUES (1, 'Alice');
  
  N = 3 (replication factor)
  W = 2 (quorum)
  
  Write to 2 of 3 nodes → Success
```

### Raft Consensus

```
Leader-based consensus:
  - Leader receives writes
  - Replicates to followers
  - Commits when majority acknowledges

Example:
  N = 5 nodes
  Quorum = 3 (majority)
  
  Leader writes to 2 followers
  3 nodes have data (including leader)
  Commit → Success
```

### Paxos Consensus

```
Multi-phase consensus:
  - Prepare phase (propose value)
  - Accept phase (majority accepts)
  - Learn phase (broadcast decision)

Quorum in each phase:
  Majority must accept for consensus
```

---

## 5. Implementation Example

### Simple Quorum (Python)

```python
import requests
from concurrent.futures import ThreadPoolExecutor, as_completed

class QuorumClient:
    def __init__(self, nodes, write_quorum, read_quorum):
        self.nodes = nodes  # ['node1:8000', 'node2:8000', 'node3:8000']
        self.write_quorum = write_quorum
        self.read_quorum = read_quorum
    
    def write(self, key, value):
        """Write with quorum"""
        def write_to_node(node):
            try:
                response = requests.post(
                    f"http://{node}/write",
                    json={"key": key, "value": value},
                    timeout=1.0
                )
                return response.status_code == 200
            except:
                return False
        
        # Send write to all nodes
        with ThreadPoolExecutor(max_workers=len(self.nodes)) as executor:
            futures = [executor.submit(write_to_node, node) for node in self.nodes]
            
            # Wait for quorum
            acks = 0
            for future in as_completed(futures):
                if future.result():
                    acks += 1
                    if acks >= self.write_quorum:
                        return True  # Quorum reached
        
        return False  # Quorum not reached
    
    def read(self, key):
        """Read with quorum"""
        def read_from_node(node):
            try:
                response = requests.get(
                    f"http://{node}/read",
                    params={"key": key},
                    timeout=1.0
                )
                if response.status_code == 200:
                    data = response.json()
                    return (data['value'], data['timestamp'])
            except:
                pass
            return None
        
        # Send read to all nodes
        with ThreadPoolExecutor(max_workers=len(self.nodes)) as executor:
            futures = [executor.submit(read_from_node, node) for node in self.nodes]
            
            # Collect responses
            responses = []
            for future in as_completed(futures):
                result = future.result()
                if result:
                    responses.append(result)
                    if len(responses) >= self.read_quorum:
                        break  # Quorum reached
        
        if len(responses) < self.read_quorum:
            return None  # Quorum not reached
        
        # Return latest value (by timestamp)
        return max(responses, key=lambda x: x[1])[0]

# Usage
client = QuorumClient(
    nodes=['node1:8000', 'node2:8000', 'node3:8000'],
    write_quorum=2,
    read_quorum=2
)

# Write
success = client.write('user:1', 'Alice')
print(f"Write: {success}")

# Read
value = client.read('user:1')
print(f"Read: {value}")
```

---

## 6. Quorum and CAP Theorem

### Consistency vs Availability

```
Strong consistency (W + R > N):
  - Quorum ensures read sees latest write
  - Sacrifice availability (need majority)
  - CP system (Consistency + Partition tolerance)

High availability (W = 1, R = 1):
  - Fast, always available
  - Sacrifice consistency (may read stale)
  - AP system (Availability + Partition tolerance)
```

### Tunable Consistency

```
Cassandra allows per-request tuning:

Strong consistency:
  CONSISTENCY QUORUM;
  SELECT * FROM users WHERE id = 1;

Eventual consistency:
  CONSISTENCY ONE;
  SELECT * FROM users WHERE id = 1;

Trade-off per use case:
  - Critical reads: Use QUORUM
  - Non-critical reads: Use ONE
```

---

## 7. Sloppy Quorum and Hinted Handoff

### Sloppy Quorum

```
Problem:
  N = 3 nodes (A, B, C)
  W = 2
  Nodes A and B are down
  Write fails (can't reach quorum)

Solution (Sloppy Quorum):
  Write to nodes C and D (temporary node)
  D is not in preference list but accepts write
  Write succeeds (quorum reached)

Trade-off:
  ✅ Higher availability
  ❌ Weaker consistency
```

### Hinted Handoff

```
After sloppy quorum:
  Node D stores hint: "This write belongs to node A"
  When node A recovers:
    Node D transfers write to node A
    Node D deletes hint

Ensures eventual consistency
```

---

## 8. Challenges

### Split brain

```
Problem:
  Network partition splits cluster
  Both sides think they have quorum
  Conflicting writes

Solution:
  Odd number of nodes (3, 5, 7)
  Only one side can have majority
  
Example:
  N = 5 nodes
  Partition: 3 nodes | 2 nodes
  Side with 3 nodes has quorum (majority)
  Side with 2 nodes cannot write
```

### Latency

```
Problem:
  Quorum requires waiting for multiple nodes
  Slower than single-node write

Solutions:
  - Async replication (eventual consistency)
  - Lower quorum (W = 1, R = 1)
  - Co-locate nodes (reduce network latency)
```

### Conflict resolution

```
Problem:
  Concurrent writes to different nodes
  Which value is correct?

Solutions:
  - Last-write-wins (timestamp)
  - Vector clocks (causal ordering)
  - Application-level resolution
```

---

## 9. When to Use Quorum

### Good fit

```
✅ Distributed databases (Cassandra, DynamoDB)
✅ Consensus systems (Raft, Paxos)
✅ Need consistency despite failures
✅ Can tolerate some latency

Examples:
  - User data (need consistency)
  - Financial transactions
  - Configuration data
```

### Poor fit

```
❌ Single-node systems
❌ Need lowest latency
❌ Eventual consistency acceptable

Examples:
  - Caching (eventual consistency OK)
  - Logging (order doesn't matter)
  - Metrics (approximate OK)
```

---

## 10. Common Interview Questions + Answers

### Q: What is a quorum and why is it important?

> "A quorum is the minimum number of nodes that must agree on an operation for it to succeed in a distributed system. For example, with 3 nodes and a write quorum of 2, you need 2 nodes to acknowledge a write before returning success. This ensures that even if one node fails, you still have the data on multiple nodes. Quorums are critical for achieving consistency in distributed systems while tolerating failures. The key insight is that if your read and write quorums overlap, you're guaranteed to read the latest write."

### Q: How do you choose quorum sizes?

> "The most common configuration is majority quorum where W + R > N. For example, with N=3 nodes, use W=2 and R=2. This guarantees strong consistency because any read quorum will overlap with the previous write quorum. For read-heavy workloads, you might use W=N and R=1 for fast reads. For write-heavy workloads, W=1 and R=N for fast writes. For highest availability with eventual consistency, use W=1 and R=1. The choice depends on your consistency requirements and read/write ratio."

### Q: What happens during a network partition with quorum?

> "With an odd number of nodes, only one side of the partition can have a majority quorum. For example, with 5 nodes split 3-2, the side with 3 nodes can continue operating since it has quorum. The side with 2 nodes cannot accept writes since it lacks quorum. This prevents split-brain where both sides accept conflicting writes. When the partition heals, the minority side syncs from the majority. This is why distributed systems use odd numbers of nodes — to ensure only one side can have quorum during partitions."

### Q: How does quorum relate to the CAP theorem?

> "Quorum lets you tune the trade-off between consistency and availability. With strong quorum (W + R > N), you get consistency but sacrifice availability — if you can't reach quorum, operations fail. This is a CP system. With weak quorum (W=1, R=1), you get high availability but sacrifice consistency — you might read stale data. This is an AP system. Systems like Cassandra let you choose per request, allowing you to optimize for consistency or availability based on the use case."

---

## 11. Quick Reference

```
What is Quorum?
  Minimum nodes that must agree for operation to succeed
  Ensures consistency despite failures
  Majority agreement

Core concepts:
  N: Total nodes
  W: Write quorum (nodes that must ack write)
  R: Read quorum (nodes that must respond to read)

Quorum configurations:
  - Majority (W + R > N): Strong consistency
  - Read-heavy (W = N, R = 1): Fast reads
  - Write-heavy (W = 1, R = N): Fast writes
  - Eventual (W = 1, R = 1): High availability

Common setup:
  N = 3, W = 2, R = 2
  Tolerates 1 node failure
  Strong consistency

Quorum in systems:
  - Cassandra/DynamoDB: Tunable consistency
  - Raft: Majority for consensus
  - Paxos: Majority in each phase

Sloppy quorum:
  Write to temporary nodes if preferred nodes down
  Higher availability, weaker consistency
  Hinted handoff for eventual consistency

Challenges:
  - Split brain (use odd number of nodes)
  - Latency (waiting for multiple nodes)
  - Conflict resolution (last-write-wins, vector clocks)

When to use:
  ✅ Distributed databases
  ✅ Consensus systems
  ✅ Need consistency despite failures
  
  ❌ Single-node systems
  ❌ Need lowest latency
  ❌ Eventual consistency acceptable

Best practices:
  - Use odd number of nodes (3, 5, 7)
  - Majority quorum for strong consistency
  - Tune per use case (read-heavy vs write-heavy)
  - Monitor quorum failures
  - Use timestamps for conflict resolution
```
