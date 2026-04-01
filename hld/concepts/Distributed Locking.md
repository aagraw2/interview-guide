## 1. What is Distributed Locking?

Distributed locking ensures that only one process across multiple servers can access a shared resource at a time. It's the distributed equivalent of a mutex or semaphore.

```
Problem:
  Server 1: Process order #123
  Server 2: Process order #123 (at the same time)
  → Order processed twice!

Solution: Distributed lock
  Server 1: Acquire lock for order #123 → Success → Process
  Server 2: Try to acquire lock for order #123 → Fail (locked) → Wait
```

**Use cases:** Prevent duplicate processing, leader election, resource coordination, scheduled jobs.

---

## 2. Why Distributed Locks?

### Single-server locking (doesn't work in distributed systems)

```python
lock = threading.Lock()

def process_order(order_id):
    with lock:
        # Only one thread on THIS server can execute
        process(order_id)

Problem:
  Server 1 and Server 2 each have their own lock
  → Both can process the same order simultaneously
```

### Distributed locking (works across servers)

```python
def process_order(order_id):
    lock = redis.lock(f"order:{order_id}", timeout=30)
    if lock.acquire(blocking=False):
        try:
            process(order_id)
        finally:
            lock.release()
    else:
        print("Order already being processed")

Now:
  Server 1 acquires lock in Redis → Success
  Server 2 tries to acquire same lock → Fail (already locked)
```

---

## 3. Redis-based Locking

### Simple lock (SET with NX and EX)

```
Acquire lock:
  SET lock:order:123 "server1" NX EX 30
  
  NX: Only set if key doesn't exist
  EX 30: Expire after 30 seconds (auto-release)
  
  Returns: OK (acquired) or nil (already locked)

Release lock:
  DEL lock:order:123
  
  But check ownership first (don't delete someone else's lock)
```

### Lock with ownership check

```lua
-- Acquire (Lua script for atomicity)
if redis.call("EXISTS", KEYS[1]) == 0 then
    redis.call("SET", KEYS[1], ARGV[1], "EX", ARGV[2])
    return 1
else
    return 0
end

-- Release (only if we own the lock)
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end

Usage:
  lock_id = generate_unique_id()  # e.g., UUID
  acquired = redis.eval(acquire_script, ["lock:order:123"], [lock_id, 30])
  
  if acquired:
      process_order()
      redis.eval(release_script, ["lock:order:123"], [lock_id])
```

### Why ownership check matters

```
Without ownership check:
  1. Server 1 acquires lock (expires in 30s)
  2. Server 1 takes 35s to process (lock expires at 30s)
  3. Server 2 acquires lock (Server 1's lock expired)
  4. Server 1 finishes, releases lock (deletes Server 2's lock!)
  5. Server 3 acquires lock (Server 2 still processing)
  → Two servers processing simultaneously

With ownership check:
  4. Server 1 tries to release lock, but lock_id doesn't match → Don't delete
  → Server 2's lock remains intact
```

---

## 4. Redlock Algorithm

Distributed lock algorithm for Redis clusters (multiple Redis instances).

### Problem with single Redis instance

```
Single Redis:
  Server 1 acquires lock in Redis
  Redis crashes
  Server 2 acquires lock in new Redis (no memory of old lock)
  → Both servers have lock

Need: Lock across multiple Redis instances
```

### Redlock algorithm

```
Setup: 5 independent Redis instances (not replicated)

Acquire lock:
  1. Get current time (T1)
  2. Try to acquire lock in all 5 instances (with timeout)
  3. Calculate time taken (T2 - T1)
  4. If acquired in majority (3 of 5) AND time < lock TTL:
       → Lock acquired
     Else:
       → Release all locks, retry

Release lock:
  Release lock in all 5 instances

Example:
  Lock TTL: 30 seconds
  Acquire in instances 1, 2, 3 (success)
  Acquire in instances 4, 5 (timeout)
  Time taken: 2 seconds
  3 of 5 acquired AND 2s < 30s → Lock acquired
```

### Controversy

```
Martin Kleppmann's critique:
  - Clock skew can break Redlock
  - GC pauses can cause issues
  - Not safe for correctness-critical applications

Salvatore Sanfilippo's (Redis creator) response:
  - Redlock is for efficiency, not correctness
  - Use fencing tokens for correctness

Consensus: Use Redlock for efficiency (prevent duplicate work)
           Use ZooKeeper/etcd for correctness (prevent data corruption)
```

---

## 5. Fencing Tokens

Prevent stale lock holders from corrupting data.

### Problem: Lock expiry during processing

```
1. Server 1 acquires lock (token=1, expires in 30s)
2. Server 1 starts processing (takes 35s)
3. Lock expires at 30s
4. Server 2 acquires lock (token=2)
5. Server 2 starts processing
6. Server 1 finishes, writes to database (stale lock holder!)
7. Server 2 finishes, writes to database
→ Data corruption (Server 1's write is stale)
```

### Solution: Fencing tokens

```
Lock server returns monotonically increasing token:
  Server 1 acquires lock → token=1
  Server 2 acquires lock → token=2
  Server 3 acquires lock → token=3

Resource (database) rejects writes with old tokens:
  Server 1 writes with token=1
  Server 2 writes with token=2 (accepted, token > 1)
  Server 1 writes with token=1 (rejected, token < 2)

Database:
  current_token = 0
  
  def write(data, token):
      if token > current_token:
          current_token = token
          save(data)
      else:
          raise StaleTokenError
```

### Implementation

```
ZooKeeper:
  - Sequential znodes provide monotonic tokens
  - Create /locks/order-123-0000000001
  - Next lock: /locks/order-123-0000000002
  - Token = sequence number

etcd:
  - Lease with revision number
  - Revision increases monotonically
  - Token = revision
```

---

## 6. ZooKeeper for Distributed Locking

### How it works

```
1. Create ephemeral sequential znode:
   /locks/order-123-0000000001
   
   Ephemeral: Deleted when client disconnects
   Sequential: Automatically numbered

2. Get all children of /locks:
   [order-123-0000000001, order-123-0000000002, ...]

3. If my znode is the smallest:
   → I have the lock
   Else:
   → Watch the znode before mine
   → Wait for notification

4. When done, delete my znode (release lock)

5. Next waiter gets notified, checks if it's now the smallest
```

### Example

```
Server 1: Create /locks/order-123-0000000001 → Smallest → Lock acquired
Server 2: Create /locks/order-123-0000000002 → Not smallest → Wait
Server 3: Create /locks/order-123-0000000003 → Not smallest → Wait

Server 1 finishes, deletes 0000000001
Server 2 gets notification, checks → Now smallest → Lock acquired

Server 2 finishes, deletes 0000000002
Server 3 gets notification, checks → Now smallest → Lock acquired
```

### Advantages

```
✅ Automatic cleanup (ephemeral nodes deleted on disconnect)
✅ Fair ordering (FIFO queue)
✅ No polling (watch notifications)
✅ Fencing tokens (sequence numbers)
✅ Strong consistency (ZooKeeper guarantees)
```

---

## 7. etcd for Distributed Locking

### How it works

```
1. Create lease (TTL):
   lease = etcd.lease(ttl=30)

2. Put key with lease:
   etcd.put("/locks/order-123", "server1", lease=lease)

3. Check if key was created (compare-and-swap):
   If key didn't exist before → Lock acquired
   Else → Lock already held

4. Keep lease alive (heartbeat):
   etcd.keep_alive(lease)

5. Release lock:
   etcd.delete("/locks/order-123")
   etcd.revoke(lease)
```

### Example

```python
import etcd3

etcd = etcd3.client()

def acquire_lock(key, ttl=30):
    lease = etcd.lease(ttl)
    success = etcd.put_if_not_exists(key, "server1", lease)
    if success:
        return lease
    else:
        return None

def release_lock(key, lease):
    etcd.delete(key)
    etcd.revoke_lease(lease)

# Usage
lease = acquire_lock("/locks/order-123")
if lease:
    try:
        process_order()
    finally:
        release_lock("/locks/order-123", lease)
```

---

## 8. Common Pitfalls

### 1. Lock not released (deadlock)

```
Problem:
  Server acquires lock
  Server crashes before releasing
  → Lock held forever

Solution:
  - Always set TTL/expiry (auto-release)
  - Use ephemeral nodes (ZooKeeper)
  - Use leases (etcd)
```

### 2. Lock expires during processing

```
Problem:
  Lock TTL: 30 seconds
  Processing takes: 35 seconds
  → Lock expires, another server acquires it

Solutions:
  - Increase TTL (but longer recovery time on crash)
  - Extend lock periodically (heartbeat)
  - Use fencing tokens (prevent stale writes)
```

### 3. Clock skew

```
Problem:
  Server 1 clock: 10:00:00
  Server 2 clock: 10:00:30 (30 seconds ahead)
  
  Server 1 acquires lock (expires at 10:00:30 by its clock)
  Server 2 sees lock expired (already 10:00:30 by its clock)
  → Both have lock

Solution:
  - Use NTP to sync clocks
  - Use logical clocks (ZooKeeper, etcd)
  - Use Redlock (majority consensus)
```

### 4. Split brain

```
Problem:
  Network partition separates Redis master from replicas
  Server 1 acquires lock on master
  Master fails, replica promoted
  Server 2 acquires lock on new master (no memory of old lock)
  → Both have lock

Solution:
  - Use Redlock (multiple independent instances)
  - Use ZooKeeper/etcd (consensus-based)
  - Accept eventual consistency (for efficiency, not correctness)
```

---

## 9. When to Use Distributed Locks

### Good use cases

```
✅ Prevent duplicate processing (idempotency)
   - Process payment only once
   - Send email only once

✅ Leader election
   - One server handles scheduled jobs
   - One server processes queue

✅ Resource coordination
   - Exclusive access to shared resource
   - Rate limiting (global counter)

✅ Scheduled jobs
   - Cron job runs on only one server
```

### Bad use cases (alternatives)

```
❌ Database transactions
   → Use database locks (SELECT FOR UPDATE)

❌ Optimistic concurrency
   → Use version numbers or ETags

❌ Rate limiting
   → Use token bucket or sliding window (no lock needed)

❌ Idempotency
   → Use idempotency keys in database (unique constraint)
```

---

## 10. Common Interview Questions + Answers

### Q: How do you implement distributed locking with Redis?

> "Use SET with NX (only if not exists) and EX (expiry) flags. Generate a unique lock ID (UUID) and store it as the value. When releasing, check that the lock ID matches before deleting to ensure you don't delete someone else's lock. Always set an expiry to prevent deadlocks if the process crashes. For higher reliability, use Redlock across multiple Redis instances, acquiring the lock in a majority (3 of 5) to tolerate failures."

### Q: What's the problem with lock expiry during processing?

> "If processing takes longer than the lock TTL, the lock expires while you're still working. Another process can then acquire the lock and start processing the same task. When the first process finishes, it might write stale data. Solutions include: increasing the TTL, extending the lock periodically with heartbeats, or using fencing tokens where the resource rejects writes with old tokens. Fencing tokens are the most robust solution for correctness-critical applications."

### Q: What's the difference between Redis locks and ZooKeeper locks?

> "Redis locks are simpler and faster but less reliable — a single Redis failure can lose locks, and clock skew can cause issues. Redlock improves reliability by using multiple Redis instances. ZooKeeper locks are more reliable with strong consistency guarantees, automatic cleanup via ephemeral nodes, fair ordering, and built-in fencing tokens via sequence numbers. Use Redis for efficiency (prevent duplicate work), ZooKeeper for correctness (prevent data corruption)."

### Q: How do fencing tokens prevent stale writes?

> "Fencing tokens are monotonically increasing numbers issued with each lock. When a process acquires a lock, it gets a token (e.g., token=5). The resource (database) tracks the highest token seen and rejects writes with lower tokens. If a process holds a stale lock (token=5) and another process acquires a new lock (token=6), the resource accepts writes from token=6 but rejects writes from token=5. This prevents stale lock holders from corrupting data even if their lock expired."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Mention ownership check

Always check lock ownership before releasing. This shows you understand the edge cases.

### ✅ Trick 2: Discuss fencing tokens

Fencing tokens are the robust solution for correctness. Mentioning them shows depth.

### ✅ Trick 3: Know the trade-offs

Redis (fast, less reliable) vs ZooKeeper (slower, more reliable). Know when to use each.

### ❌ Pitfall 1: Forgetting TTL

Always set a TTL to prevent deadlocks. Forgetting this is a common mistake.

### ❌ Pitfall 2: Thinking locks solve everything

Distributed locks are complex and have failure modes. Often there are simpler alternatives (database constraints, optimistic locking).

### ❌ Pitfall 3: Ignoring clock skew

Clock skew can break Redis locks. Mention NTP or use systems with logical clocks (ZooKeeper, etcd).

---

## 12. Quick Reference

```
What is distributed locking?
  Ensure only one process across multiple servers accesses a resource
  Use cases: Prevent duplicates, leader election, resource coordination

Redis locking:
  SET lock:key value NX EX 30
  NX: Only if not exists
  EX: Expiry (auto-release)
  Always check ownership before releasing

Redlock:
  5 independent Redis instances
  Acquire in majority (3 of 5)
  More reliable than single Redis
  Controversy: Not safe for correctness-critical apps

Fencing tokens:
  Monotonically increasing numbers
  Resource rejects writes with old tokens
  Prevents stale lock holders from corrupting data
  Implementation: ZooKeeper sequence numbers, etcd revisions

ZooKeeper:
  Ephemeral sequential znodes
  Automatic cleanup on disconnect
  Fair ordering (FIFO)
  Built-in fencing tokens
  Strong consistency

etcd:
  Leases with TTL
  Keep-alive heartbeat
  Revision numbers for fencing
  Strong consistency

Common pitfalls:
  1. Lock not released → Always set TTL
  2. Lock expires during processing → Extend lock or use fencing tokens
  3. Clock skew → Use NTP or logical clocks
  4. Split brain → Use Redlock or consensus systems

When to use:
  ✅ Prevent duplicate processing
  ✅ Leader election
  ✅ Resource coordination
  ✅ Scheduled jobs
  ❌ Database transactions (use DB locks)
  ❌ Optimistic concurrency (use version numbers)

Redis vs ZooKeeper:
  Redis: Fast, simple, less reliable (efficiency)
  ZooKeeper: Slower, complex, more reliable (correctness)

Best practices:
  - Always set TTL/expiry
  - Check ownership before releasing
  - Use fencing tokens for correctness
  - Sync clocks with NTP
  - Consider alternatives (DB constraints, optimistic locking)
```
