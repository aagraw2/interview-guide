## 1. Why Unique IDs?

In distributed systems, you need globally unique identifiers for entities (users, orders, posts, etc.) that can be generated independently across multiple servers without coordination.

```
Problem with auto-increment IDs:
  Server 1: INSERT INTO orders → id = 1
  Server 2: INSERT INTO orders → id = 1 (collision!)

Need: Globally unique IDs generated without coordination
```

**Requirements:**
- Globally unique (no collisions)
- Sortable by time (for range queries, pagination)
- High throughput (millions of IDs per second)
- Low latency (generate in <1ms)
- Compact (fit in 64 bits for database efficiency)

---

## 2. UUID (Universally Unique Identifier)

128-bit random identifier with extremely low collision probability.

### UUID v4 (random)

```
Example: 550e8400-e29b-41d4-a716-446655440000

Format: 8-4-4-4-12 hex digits (32 hex chars = 128 bits)

Generation:
  - 122 random bits
  - 6 bits for version and variant
  - Collision probability: ~1 in 2^122 (negligible)

Pros:
  ✅ No coordination needed
  ✅ Can generate offline
  ✅ Extremely low collision probability

Cons:
  ❌ Not sortable by time
  ❌ Large (128 bits = 16 bytes)
  ❌ Poor database index performance (random inserts)
  ❌ Not human-readable
```

### UUID v1 (timestamp + MAC address)

```
Example: 6ba7b810-9dad-11d1-80b4-00c04fd430c8

Components:
  - 60-bit timestamp (100-nanosecond intervals since 1582)
  - 48-bit MAC address
  - 14-bit clock sequence
  - 6-bit version/variant

Pros:
  ✅ Sortable by time (timestamp first)
  ✅ No coordination needed

Cons:
  ❌ Exposes MAC address (privacy concern)
  ❌ Still 128 bits (large)
```

### UUID v7 (timestamp + random)

```
New standard (2024): Combines timestamp with random bits

Components:
  - 48-bit Unix timestamp (milliseconds)
  - 12-bit random
  - 62-bit random

Pros:
  ✅ Sortable by time
  ✅ No privacy concerns (no MAC address)
  ✅ Better database performance than v4

Cons:
  ❌ Still 128 bits
```

---

## 3. Snowflake ID (Twitter)

64-bit ID with timestamp, machine ID, and sequence number.

### Structure

```
┌─────────────────────────────────────────────────────────────┐
│ 1 bit │ 41 bits      │ 10 bits    │ 12 bits                 │
│ Sign  │ Timestamp    │ Machine ID │ Sequence                │
│ (0)   │ (ms)         │            │                         │
└─────────────────────────────────────────────────────────────┘

Total: 64 bits (fits in a long/bigint)

Components:
  - 1 bit: Always 0 (reserved for sign)
  - 41 bits: Timestamp in milliseconds since custom epoch
    → 2^41 ms = 69 years
  - 10 bits: Machine/datacenter ID
    → 2^10 = 1024 machines
  - 12 bits: Sequence number (per machine per millisecond)
    → 2^12 = 4096 IDs per ms per machine
```

### Example

```
Timestamp: 2024-01-01 00:00:00.000 (1704067200000 ms since epoch)
Machine ID: 5
Sequence: 123

Binary:
  0 | 0000000110001110101011110110000000000 | 0000000101 | 000001111011

Decimal: 7123456789012345678

Capacity:
  4096 IDs/ms × 1000 ms/s = 4 million IDs/second per machine
  × 1024 machines = 4 billion IDs/second total
```

### Pros and cons

```
Pros:
  ✅ Sortable by time (timestamp first)
  ✅ Compact (64 bits)
  ✅ High throughput (4M IDs/sec per machine)
  ✅ No coordination needed (each machine independent)
  ✅ Good database performance (sequential inserts)

Cons:
  ❌ Requires machine ID assignment (coordination at startup)
  ❌ Clock skew issues (if clocks drift)
  ❌ Limited to 1024 machines (10-bit machine ID)
  ❌ Sequence overflow if >4096 IDs in same millisecond
```

---

## 4. Clock Skew Problem

### Problem

```
Machine 1 clock: 2024-01-01 00:00:00.000
Machine 2 clock: 2024-01-01 00:00:01.000 (1 second ahead)

IDs generated:
  Machine 1: 7123456789012345678 (timestamp: 00:00:00.000)
  Machine 2: 7123456793307312973 (timestamp: 00:00:01.000)

Machine 2's ID is larger even though it was generated earlier!
  → Breaks time-based sorting
```

### Solutions

#### 1. NTP (Network Time Protocol)

```
Sync all machines to a central time server
  - Typical accuracy: ±10ms
  - Good enough for most use cases

Limitation: Can't prevent all clock skew
```

#### 2. Wait for clock to catch up

```
If current_time < last_timestamp:
  # Clock moved backward
  Wait until current_time >= last_timestamp
  Then generate ID

Limitation: Blocks ID generation during wait
```

#### 3. Reject requests during clock skew

```
If current_time < last_timestamp:
  # Clock moved backward
  Throw error: "Clock skew detected"
  Client retries on another machine

Limitation: Temporary unavailability
```

#### 4. Use logical clocks (Lamport timestamps)

```
Instead of wall-clock time, use logical counter
  - Increment on every ID generation
  - Sync counters across machines

Limitation: Loses absolute time information
```

---

## 5. ULID (Universally Unique Lexicographically Sortable Identifier)

128-bit ID that's sortable and more compact than UUID when encoded.

### Structure

```
┌──────────────────────────────────────────────────────────┐
│ 48 bits          │ 80 bits                                │
│ Timestamp (ms)   │ Random                                 │
└──────────────────────────────────────────────────────────┘

Total: 128 bits

Encoded as 26-character Crockford Base32 string:
  01ARZ3NDEKTSV4RRFFQ69G5FAV

Components:
  - 48-bit timestamp (milliseconds since Unix epoch)
    → 2^48 ms = 8925 years
  - 80-bit random
    → 2^80 = 1.2 × 10^24 combinations per millisecond
```

### Example

```
Timestamp: 2024-01-01 00:00:00.000
Random: [80 random bits]

ULID: 01HN3Z8QKXYZ9ABCDEFGHJKMNP

Sorting:
  01HN3Z8QKXYZ9ABCDEFGHJKMNP (2024-01-01 00:00:00.000)
  01HN3Z8QKYABC123456789XYZW (2024-01-01 00:00:00.001)
  01HN3Z8QKZDEF456789ABCXYZW (2024-01-01 00:00:00.002)

Lexicographically sorted = time sorted
```

### Pros and cons

```
Pros:
  ✅ Sortable by time
  ✅ No coordination needed
  ✅ Compact string representation (26 chars vs 36 for UUID)
  ✅ Case-insensitive (Crockford Base32)
  ✅ No special characters (URL-safe)

Cons:
  ❌ Still 128 bits (larger than Snowflake)
  ❌ Not as compact as 64-bit IDs
```

---

## 6. Database Auto-Increment with Offset

Use database auto-increment with different offsets per server.

### Setup

```
Server 1:
  auto_increment_offset = 1
  auto_increment_increment = 3
  IDs: 1, 4, 7, 10, 13, ...

Server 2:
  auto_increment_offset = 2
  auto_increment_increment = 3
  IDs: 2, 5, 8, 11, 14, ...

Server 3:
  auto_increment_offset = 3
  auto_increment_increment = 3
  IDs: 3, 6, 9, 12, 15, ...

No collisions across servers
```

### Pros and cons

```
Pros:
  ✅ Simple to implement
  ✅ Compact (32 or 64 bits)
  ✅ Sortable by time (mostly)

Cons:
  ❌ Requires coordination (assign offsets)
  ❌ Hard to add/remove servers (change increment)
  ❌ Gaps in sequence (not consecutive)
  ❌ Exposes number of servers (security concern)
```

---

## 7. Ticket Server (Flickr)

Centralized ID generation service.

### Architecture

```
Application servers → Ticket server → Generate ID

Ticket server:
  - Single MySQL database with auto-increment
  - REPLACE INTO tickets (stub) VALUES ('a')
  - Returns LAST_INSERT_ID()

High availability:
  - Two ticket servers with different offsets
  - Server 1: odd IDs (1, 3, 5, ...)
  - Server 2: even IDs (2, 4, 6, ...)
```

### Pros and cons

```
Pros:
  ✅ Simple to implement
  ✅ Compact (64 bits)
  ✅ Sortable by time

Cons:
  ❌ Single point of failure (even with HA)
  ❌ Network latency (extra hop)
  ❌ Bottleneck (all requests go through ticket server)
  ❌ Not suitable for high throughput
```

---

## 8. Comparison

|Method|Size|Sortable|Coordination|Throughput|Use Case|
|---|---|---|---|---|---|
|UUID v4|128 bits|❌|None|Very high|General purpose, offline generation|
|UUID v7|128 bits|✅|None|Very high|Modern replacement for v4|
|Snowflake|64 bits|✅|Machine ID|Very high (4M/sec)|Twitter, Instagram, Discord|
|ULID|128 bits|✅|None|Very high|URL-safe, human-readable|
|Auto-increment + offset|32/64 bits|✅|Offset assignment|High|Small clusters|
|Ticket server|64 bits|✅|Centralized|Low|Legacy systems|

---

## 9. Implementation: Snowflake in Python

```python
import time
import threading

class SnowflakeIDGenerator:
    def __init__(self, machine_id, epoch=1704067200000):
        self.machine_id = machine_id  # 10 bits (0-1023)
        self.epoch = epoch  # Custom epoch (2024-01-01)
        self.sequence = 0
        self.last_timestamp = -1
        self.lock = threading.Lock()
    
    def generate(self):
        with self.lock:
            timestamp = int(time.time() * 1000)  # Current time in ms
            
            if timestamp < self.last_timestamp:
                # Clock moved backward
                raise Exception("Clock moved backward")
            
            if timestamp == self.last_timestamp:
                # Same millisecond, increment sequence
                self.sequence = (self.sequence + 1) & 0xFFF  # 12 bits
                if self.sequence == 0:
                    # Sequence overflow, wait for next millisecond
                    while timestamp <= self.last_timestamp:
                        timestamp = int(time.time() * 1000)
            else:
                # New millisecond, reset sequence
                self.sequence = 0
            
            self.last_timestamp = timestamp
            
            # Build ID
            id = ((timestamp - self.epoch) << 22) | \
                 (self.machine_id << 12) | \
                 self.sequence
            
            return id

# Usage
generator = SnowflakeIDGenerator(machine_id=5)
id1 = generator.generate()  # 7123456789012345678
id2 = generator.generate()  # 7123456789012345679
```

---

## 10. Common Interview Questions + Answers

### Q: Why not use UUID for database primary keys?

> "UUIDs are 128 bits (16 bytes), which is large compared to 64-bit integers. More importantly, UUID v4 is random, which causes poor database performance — random inserts fragment B-tree indexes and require more disk I/O. Sequential IDs like Snowflake or UUID v7 are better because they insert at the end of the index, minimizing fragmentation. For high-throughput systems, use 64-bit Snowflake IDs for better performance."

### Q: How does Snowflake ID handle clock skew?

> "Snowflake IDs are vulnerable to clock skew. If a machine's clock moves backward, it could generate IDs with older timestamps than previously generated IDs, breaking time-based sorting. Solutions include: using NTP to sync clocks (±10ms accuracy), waiting for the clock to catch up before generating IDs, or rejecting requests and failing over to another machine. In practice, NTP is usually sufficient for most use cases."

### Q: What's the difference between Snowflake and ULID?

> "Snowflake is 64 bits with timestamp, machine ID, and sequence number. It requires machine ID assignment but is very compact. ULID is 128 bits with timestamp and random bits. It doesn't require coordination and is URL-safe with a compact string representation (26 characters). Use Snowflake for high-throughput systems where 64 bits is important. Use ULID when you want sortability without coordination and don't mind the larger size."

### Q: How would you design a distributed ID generator for Instagram?

> "I'd use Snowflake IDs. Each application server gets a unique machine ID (0-1023). IDs are generated locally without coordination, achieving 4 million IDs per second per server. The 64-bit IDs are sortable by time, which is important for Instagram's feed pagination. To handle clock skew, I'd use NTP to sync clocks and reject requests if the clock moves backward significantly. For machine ID assignment, I'd use a service discovery system like Consul or ZooKeeper to assign IDs on server startup."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Know Snowflake structure

Snowflake is the most common interview topic. Know the bit layout: 1 sign + 41 timestamp + 10 machine + 12 sequence.

### ✅ Trick 2: Discuss clock skew

Clock skew is a critical issue with timestamp-based IDs. Always mention NTP and how to handle backward clock movement.

### ✅ Trick 3: Compare multiple approaches

Don't just describe one method. Compare UUID, Snowflake, and ULID with trade-offs. This shows breadth.

### ❌ Pitfall 1: Thinking UUIDs are always good

UUID v4 is random and causes poor database performance. Mention this limitation and suggest Snowflake or UUID v7 instead.

### ❌ Pitfall 2: Forgetting about coordination

Snowflake requires machine ID assignment. ULID and UUID don't. Know which methods need coordination.

### ❌ Pitfall 3: Ignoring throughput limits

Snowflake can generate 4096 IDs per millisecond per machine. If you need more, you need more machines or a different approach.

---

## 12. Quick Reference

```
Why unique IDs?
  Globally unique identifiers without coordination
  Requirements: Unique, sortable, high throughput, compact

UUID v4:
  128 bits, random
  Pros: No coordination, low collision
  Cons: Not sortable, large, poor DB performance

UUID v7:
  128 bits, timestamp + random
  Pros: Sortable, no coordination
  Cons: Still 128 bits

Snowflake (Twitter):
  64 bits: 1 sign + 41 timestamp + 10 machine + 12 sequence
  Capacity: 4M IDs/sec per machine
  Pros: Sortable, compact, high throughput
  Cons: Requires machine ID, clock skew issues

Clock skew:
  Problem: Clock moves backward → breaks sorting
  Solutions: NTP sync, wait for clock, reject requests

ULID:
  128 bits: 48 timestamp + 80 random
  26-char Crockford Base32 string
  Pros: Sortable, no coordination, URL-safe
  Cons: Larger than Snowflake (128 vs 64 bits)

Auto-increment + offset:
  Server 1: 1, 4, 7, 10, ... (offset=1, increment=3)
  Server 2: 2, 5, 8, 11, ... (offset=2, increment=3)
  Pros: Simple, compact
  Cons: Hard to scale, requires coordination

Ticket server:
  Centralized ID generation
  Pros: Simple, sortable
  Cons: Single point of failure, bottleneck

Comparison:
  UUID v4: 128 bits, not sortable, no coordination
  UUID v7: 128 bits, sortable, no coordination
  Snowflake: 64 bits, sortable, machine ID needed
  ULID: 128 bits, sortable, no coordination

Best practices:
  - Use Snowflake for high-throughput systems (64 bits)
  - Use ULID for URL-safe, human-readable IDs
  - Use UUID v7 for general purpose (modern replacement for v4)
  - Avoid UUID v4 for database primary keys (poor performance)
  - Use NTP to sync clocks (handle clock skew)
```
