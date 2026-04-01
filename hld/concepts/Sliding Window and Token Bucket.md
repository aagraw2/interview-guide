## 1. Rate Limiting Algorithms Overview

Two most common algorithms for rate limiting:

```
Sliding Window:
  - Track requests in a time window
  - Window slides continuously
  - Accurate, smooth rate limiting

Token Bucket:
  - Bucket holds tokens
  - Tokens refill at constant rate
  - Allows bursts up to bucket capacity
```

**Both solve the same problem:** Limit request rate to prevent overload.

---

## 2. Sliding Window Counter

### How it works

```
Limit: 100 requests per minute

Current time: 10:05:30
Window: 10:04:30 to 10:05:30 (last 60 seconds)

Count requests in window:
  - If count < 100 → allow request
  - If count >= 100 → reject request
```

---

### Implementation (Redis)

```python
import time
import redis

def is_allowed_sliding_window(user_id, limit=100, window=60):
    redis_client = redis.Redis()
    key = f"rate_limit:{user_id}"
    now = time.time()
    window_start = now - window
    
    # Remove old entries outside window
    redis_client.zremrangebyscore(key, 0, window_start)
    
    # Count entries in window
    count = redis_client.zcard(key)
    
    if count < limit:
        # Add current request
        redis_client.zadd(key, {now: now})
        redis_client.expire(key, window)
        return True
    
    return False
```

**Data structure:** Sorted set (ZSET) with timestamps as scores.

---

### Pros and cons

```
Pros:
  ✅ Accurate (no boundary issues)
  ✅ Smooth rate limiting
  ✅ No burst at window boundaries

Cons:
  ❌ Memory intensive (stores every request timestamp)
  ❌ O(log n) operations (sorted set)
  ❌ Expensive for high-traffic systems
```

---

## 3. Sliding Window Counter (Optimized)

### How it works

```
Hybrid approach: combines fixed windows with sliding calculation

Current window: 10:05:00 - 10:05:59 (45 seconds elapsed)
Previous window: 10:04:00 - 10:04:59

Estimated count = 
  (previous_count × (60 - 45) / 60) + current_count
  = (previous_count × 0.25) + current_count
```

---

### Implementation

```python
def is_allowed_sliding_window_counter(user_id, limit=100, window=60):
    redis_client = redis.Redis()
    now = time.time()
    current_window = int(now / window) * window
    previous_window = current_window - window
    
    # Get counts
    current_key = f"rate_limit:{user_id}:{current_window}"
    previous_key = f"rate_limit:{user_id}:{previous_window}"
    
    current_count = int(redis_client.get(current_key) or 0)
    previous_count = int(redis_client.get(previous_key) or 0)
    
    # Calculate elapsed time in current window
    elapsed = now - current_window
    weight = (window - elapsed) / window
    
    # Estimate total count
    estimated_count = previous_count * weight + current_count
    
    if estimated_count < limit:
        redis_client.incr(current_key)
        redis_client.expire(current_key, window * 2)
        return True
    
    return False
```

---

### Pros and cons

```
Pros:
  ✅ Memory efficient (only 2 counters)
  ✅ O(1) operations
  ✅ Smooth rate limiting
  ✅ Scales to high traffic

Cons:
  ❌ Approximate (not perfectly accurate)
  ❌ Slightly more complex logic
```

**This is the most commonly used algorithm in production.**

---

## 4. Token Bucket

### How it works

```
Bucket:
  - Capacity: 100 tokens
  - Refill rate: 10 tokens/second

Request arrives:
  1. Refill tokens based on time elapsed
  2. If tokens >= 1:
     - Consume 1 token
     - Allow request
  3. If tokens < 1:
     - Reject request

Tokens refill continuously up to capacity
```

---

### Visual example

```
Time 0:  Bucket has 100 tokens (full)
Time 1:  Request → consume 1 token → 99 tokens
Time 2:  Request → consume 1 token → 98 tokens
...
Time 10: 10 requests → 90 tokens
Time 11: No requests → refill 10 tokens → 100 tokens (capped at capacity)

Burst scenario:
Time 0:  100 tokens
Time 1:  100 requests → consume 100 tokens → 0 tokens
Time 2:  Request → 0 tokens → REJECT
Time 3:  10 tokens refilled → 10 tokens
Time 4:  10 requests → consume 10 tokens → 0 tokens
```

---

### Implementation

```python
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.tokens = capacity
        self.last_refill = time.time()
    
    def is_allowed(self):
        now = time.time()
        
        # Refill tokens based on time elapsed
        elapsed = now - self.last_refill
        tokens_to_add = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now
        
        # Check if request allowed
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        
        return False
```

---

### Redis implementation

```python
def is_allowed_token_bucket(user_id, capacity=100, refill_rate=10):
    redis_client = redis.Redis()
    key = f"rate_limit:{user_id}"
    now = time.time()
    
    # Get current state
    data = redis_client.hgetall(key)
    tokens = float(data.get(b'tokens', capacity))
    last_refill = float(data.get(b'last_refill', now))
    
    # Refill tokens
    elapsed = now - last_refill
    tokens = min(capacity, tokens + elapsed * refill_rate)
    
    # Check if allowed
    if tokens >= 1:
        tokens -= 1
        redis_client.hset(key, 'tokens', tokens)
        redis_client.hset(key, 'last_refill', now)
        redis_client.expire(key, 60)
        return True
    
    return False
```

---

### Pros and cons

```
Pros:
  ✅ Allows bursts (up to bucket capacity)
  ✅ Smooth refill (continuous)
  ✅ Memory efficient (2 values: tokens + timestamp)
  ✅ Intuitive model

Cons:
  ❌ Burst can overwhelm downstream
  ❌ Slightly more complex than fixed window
```

---

## 5. Leaky Bucket

### How it works

```
Bucket:
  - Capacity: 100 requests
  - Leak rate: 10 requests/second

Request arrives:
  1. Add to bucket
  2. If bucket full → reject
  3. Bucket "leaks" (processes) at constant rate

Output rate is constant, regardless of input rate
```

---

### Visual example

```
Input (bursty):
  Time 0: 50 requests → bucket has 50
  Time 1: 100 requests → bucket has 140 (50 + 100 - 10 leaked)
  Time 2: 0 requests → bucket has 130 (140 - 10 leaked)
  Time 3: 0 requests → bucket has 120 (130 - 10 leaked)

Output (smooth):
  Constant 10 requests/second processed
```

---

### Implementation

```python
class LeakyBucket:
    def __init__(self, capacity, leak_rate):
        self.capacity = capacity
        self.leak_rate = leak_rate  # requests per second
        self.queue_size = 0
        self.last_leak = time.time()
    
    def is_allowed(self):
        now = time.time()
        
        # Leak (process) requests
        elapsed = now - self.last_leak
        leaked = elapsed * self.leak_rate
        self.queue_size = max(0, self.queue_size - leaked)
        self.last_leak = now
        
        # Check if bucket has space
        if self.queue_size < self.capacity:
            self.queue_size += 1
            return True
        
        return False
```

---

### Token bucket vs leaky bucket

```
Token Bucket:
  - Allows bursts (up to capacity)
  - Variable output rate
  - Tokens accumulate when idle
  - Better for APIs (allow occasional bursts)

Leaky Bucket:
  - Smooths bursts
  - Constant output rate
  - Requests queue up
  - Better for traffic shaping (network)
```

**For API rate limiting, token bucket is more common.**

---

## 6. Fixed Window Counter

### How it works

```
Limit: 100 requests per minute

Windows:
  10:00:00 - 10:00:59 → 100 requests allowed
  10:01:00 - 10:01:59 → 100 requests allowed (counter resets)
```

---

### Implementation

```python
def is_allowed_fixed_window(user_id, limit=100, window=60):
    redis_client = redis.Redis()
    now = time.time()
    window_key = int(now / window) * window
    key = f"rate_limit:{user_id}:{window_key}"
    
    count = redis_client.incr(key)
    if count == 1:
        redis_client.expire(key, window)
    
    return count <= limit
```

---

### Boundary problem

```
10:00:30 - 10:00:59: 100 requests (allowed)
10:01:00 - 10:01:29: 100 requests (allowed, new window)

Total: 200 requests in 60 seconds (burst at boundary)
```

**This is why sliding window is preferred.**

---

## 7. Comparison Table

|Algorithm|Accuracy|Memory|Complexity|Bursts|Use Case|
|---|---|---|---|---|---|
|Fixed Window|Low|O(1)|O(1)|Boundary burst|Simple, low traffic|
|Sliding Window Log|High|O(n)|O(log n)|No burst|Accurate, low traffic|
|Sliding Window Counter|Medium|O(1)|O(1)|Minimal burst|Production (most common)|
|Token Bucket|High|O(1)|O(1)|Allows burst|APIs, allow bursts|
|Leaky Bucket|High|O(1)|O(1)|Smooths burst|Traffic shaping|

---

## 8. Distributed Rate Limiting

### Challenge

```
Multiple servers, shared rate limit:
  - User makes 50 requests to Server A
  - User makes 50 requests to Server B
  - Total: 100 requests (at limit)

Without coordination:
  - Server A: 50 requests (allows)
  - Server B: 50 requests (allows)
  - Total: 100 requests (correct)

But if limit is 60:
  - Server A: 50 requests (allows)
  - Server B: 50 requests (allows)
  - Total: 100 requests (should reject 40)
```

---

### Solution: Centralized counter (Redis)

```
All servers check Redis:
  Server A → Redis (increment counter) → 50
  Server B → Redis (increment counter) → 100

Redis tracks global count
All servers see same state
```

---

### Solution: Local counters with sync

```
Each server tracks locally:
  Server A: 50 requests
  Server B: 50 requests

Periodically sync to Redis:
  Every 10 seconds → report to Redis
  Redis aggregates → total = 100

Tradeoff: Approximate, but reduces Redis load
```

---

## 9. Real-World Examples

### Stripe API

```
Rate limit: 100 requests per second

Uses token bucket:
  - Allows bursts up to 100 requests
  - Refills at 100 tokens/second
  - Returns headers:
    X-RateLimit-Limit: 100
    X-RateLimit-Remaining: 42
    X-RateLimit-Reset: 1640000000
```

---

### GitHub API

```
Rate limit: 5,000 requests per hour

Uses fixed window:
  - Resets every hour
  - Returns headers:
    X-RateLimit-Limit: 5000
    X-RateLimit-Remaining: 4999
    X-RateLimit-Reset: 1640000000
```

---

### AWS API Gateway

```
Rate limit: 10,000 requests per second
Burst: 5,000 requests

Uses token bucket:
  - Steady rate: 10,000 req/sec
  - Burst capacity: 5,000 requests
  - Allows temporary spikes
```

---

## 10. Common Interview Questions + Answers

### Q: Explain the difference between token bucket and leaky bucket.

> "Token bucket allows bursts up to the bucket capacity. Tokens refill at a constant rate, and requests consume tokens. If you have 100 tokens, you can make 100 requests immediately, then wait for refill. Leaky bucket smooths bursts by processing requests at a constant rate. Requests queue up in the bucket, and the bucket 'leaks' (processes) at a fixed rate. Token bucket has variable output rate and allows bursts, making it better for APIs. Leaky bucket has constant output rate and smooths bursts, making it better for traffic shaping in networks."

### Q: What's the boundary problem with fixed window counters?

> "Fixed window counters reset at fixed intervals. If the limit is 100 requests per minute, a user could make 100 requests at 10:00:59 and another 100 at 10:01:00, totaling 200 requests in 2 seconds. This burst at the window boundary can overwhelm the system. Sliding window counters solve this by continuously sliding the window, so the limit is enforced over any 60-second period, not just fixed minute boundaries."

### Q: How do you implement rate limiting in a distributed system?

> "I'd use Redis as a centralized counter that all servers check. For sliding window counter, each server increments counters in Redis for the current and previous time windows. For token bucket, each server reads and updates the token count and last refill time in Redis. This ensures all servers see the same rate limit state. The tradeoff is the extra Redis call on every request. To reduce load, I could use local counters that sync periodically to Redis, accepting that the limit might be slightly exceeded temporarily."

### Q: Which rate limiting algorithm would you use for an API?

> "I'd use sliding window counter for most cases. It's memory efficient (O(1)), has O(1) operations, provides smooth rate limiting without boundary bursts, and scales to high traffic. It's the best balance of accuracy, performance, and simplicity. If I specifically need to allow bursts (like allowing a client to catch up after being idle), I'd use token bucket. For simple, low-traffic scenarios, fixed window is fine. I'd avoid sliding window log (stores all timestamps) for high-traffic APIs due to memory overhead."

---

## 11. Quick Reference

```
Sliding Window Counter (most common):
  - Track counts in current and previous windows
  - Estimate count with weighted average
  - Memory: O(1), Time: O(1)
  - Smooth, no boundary burst
  - Use for: Production APIs

Token Bucket:
  - Bucket holds tokens, refills at constant rate
  - Requests consume tokens
  - Memory: O(1), Time: O(1)
  - Allows bursts up to capacity
  - Use for: APIs that allow bursts

Leaky Bucket:
  - Requests queue in bucket, leak at constant rate
  - Smooths bursts
  - Memory: O(1), Time: O(1)
  - Constant output rate
  - Use for: Traffic shaping

Fixed Window:
  - Count requests in fixed time windows
  - Memory: O(1), Time: O(1)
  - Boundary burst problem
  - Use for: Simple, low-traffic

Sliding Window Log:
  - Store all request timestamps
  - Memory: O(n), Time: O(log n)
  - Accurate, no burst
  - Use for: Low traffic, need accuracy

Distributed rate limiting:
  - Centralized counter (Redis)
  - All servers check same state
  - Ensures global limit enforced

Comparison:
  Accuracy:   Sliding Log > Token Bucket > Sliding Counter > Fixed Window
  Memory:     Fixed/Sliding Counter/Token Bucket (O(1)) > Sliding Log (O(n))
  Complexity: Fixed/Sliding Counter/Token Bucket (O(1)) > Sliding Log (O(log n))
  Bursts:     Token Bucket (allows) > Others (prevent)

Real-world:
  Stripe:  Token bucket (100 req/sec, allows bursts)
  GitHub:  Fixed window (5,000 req/hour)
  AWS:     Token bucket (10,000 req/sec, 5,000 burst)

Key insight: Sliding window counter is the production standard
```
