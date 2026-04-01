## 1. What Is Rate Limiting?

Rate limiting controls how many requests a client can make in a given time window. It protects services from overload and abuse.

```
Without rate limiting:
  Malicious client → 100,000 req/sec → Server overwhelmed → Service down

With rate limiting:
  Client → 1,000 req/sec allowed → Excess requests rejected (HTTP 429)
```

**Goals:**
- Prevent abuse (DDoS, scraping, brute force)
- Ensure fair resource allocation
- Protect backend from overload
- Enforce API pricing tiers

---

## 2. Where to Apply Rate Limiting

### API Gateway

Most common location. Centralized enforcement before requests reach backend.

```
Client → [API Gateway + Rate Limiter] → Backend Services
              ↓
         Reject if over limit
```

### Application layer

Rate limiting in the application code itself.

```python
@rate_limit(max_requests=100, window=60)  # 100 req/min
def api_endpoint():
    return process_request()
```

### Load balancer

Some load balancers (Nginx, HAProxy) support rate limiting.

### CDN / WAF

Cloudflare, AWS WAF can rate limit at the edge before traffic reaches your infrastructure.

---

## 3. Rate Limiting Algorithms

### Fixed Window Counter

Count requests in fixed time windows.

```
Window: 60 seconds
Limit: 100 requests

00:00 - 00:59 → 100 requests allowed
01:00 - 01:59 → counter resets, 100 more allowed
```

**Implementation:**

```python
def is_allowed(user_id):
    key = f"rate_limit:{user_id}:{current_minute()}"
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, 60)  # expire after 60 seconds
    return count <= 100
```

**Pros:** Simple, memory efficient. **Cons:** Burst at window boundaries.

**Boundary problem:**

```
00:59 → 100 requests
01:00 → 100 requests (new window)
Total: 200 requests in 2 seconds (burst)
```

---

### Sliding Window Log

Track timestamp of each request. Count requests in the last N seconds.

```
Limit: 100 requests per 60 seconds

Current time: 10:05:30
Count requests between 10:04:30 and 10:05:30
If count < 100 → allow
```

**Implementation:**

```python
def is_allowed(user_id):
    key = f"rate_limit:{user_id}"
    now = time.time()
    window_start = now - 60
    
    # Remove old entries
    redis.zremrangebyscore(key, 0, window_start)
    
    # Count entries in window
    count = redis.zcard(key)
    
    if count < 100:
        redis.zadd(key, {now: now})
        redis.expire(key, 60)
        return True
    return False
```

**Pros:** Accurate, no boundary burst. **Cons:** Memory intensive (stores every request timestamp).

---

### Sliding Window Counter

Hybrid approach. Combines fixed window efficiency with sliding window accuracy.

```
Current window: 01:00 - 01:59 (45 seconds elapsed)
Previous window: 00:00 - 00:59

Estimated count = 
  (previous_window_count × (60 - 45) / 60) + current_window_count
  = (previous_count × 0.25) + current_count
```

**Implementation:**

```python
def is_allowed(user_id):
    current_window = current_minute()
    previous_window = current_window - 1
    
    current_count = redis.get(f"rate_limit:{user_id}:{current_window}") or 0
    previous_count = redis.get(f"rate_limit:{user_id}:{previous_window}") or 0
    
    elapsed_in_current = seconds_elapsed_in_current_minute()
    weight = (60 - elapsed_in_current) / 60
    
    estimated_count = previous_count * weight + current_count
    
    if estimated_count < 100:
        redis.incr(f"rate_limit:{user_id}:{current_window}")
        return True
    return False
```

**Pros:** Memory efficient, smooth rate limiting. **Cons:** Approximate (not perfectly accurate).

**This is the most commonly used algorithm in production.**

---

### Token Bucket

Bucket holds tokens. Each request consumes a token. Tokens refill at a constant rate.

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/second

Request arrives:
  If tokens available → consume 1 token, allow request
  If no tokens → reject request

Tokens refill continuously (10/sec) up to capacity (100)
```

**Implementation:**

```python
def is_allowed(user_id):
    key = f"rate_limit:{user_id}"
    now = time.time()
    
    # Get current state
    data = redis.hgetall(key)
    tokens = float(data.get('tokens', 100))
    last_refill = float(data.get('last_refill', now))
    
    # Refill tokens
    elapsed = now - last_refill
    tokens = min(100, tokens + elapsed * 10)  # 10 tokens/sec, max 100
    
    if tokens >= 1:
        tokens -= 1
        redis.hset(key, 'tokens', tokens)
        redis.hset(key, 'last_refill', now)
        redis.expire(key, 60)
        return True
    return False
```

**Pros:** Allows bursts (up to bucket capacity), smooth refill. **Cons:** More complex, requires storing state.

**Use for:** APIs that need to allow occasional bursts but limit sustained rate.

---

### Leaky Bucket

Requests enter a bucket. Bucket "leaks" (processes) at a constant rate.

```
Bucket capacity: 100 requests
Leak rate: 10 requests/second

Requests arrive at any rate → added to bucket
Bucket processes 10 req/sec
If bucket full → reject new requests
```

**Difference from token bucket:** Leaky bucket enforces a constant output rate. Token bucket allows bursts.

**Use for:** Smoothing bursty traffic into a steady stream.

---

## 4. Distributed Rate Limiting

In a distributed system with multiple servers, rate limiting must be coordinated.

### Centralized counter (Redis)

All servers check a shared Redis counter.

```
Server A → Redis (increment counter) → check limit
Server B → Redis (increment counter) → check limit
Server C → Redis (increment counter) → check limit
```

**Pros:** Accurate, consistent across servers. **Cons:** Redis is a single point of failure, adds latency.

**Most common approach in production.**

---

### Local counters with sync

Each server maintains local counters, periodically syncs with central store.

```
Server A: local count = 30
Server B: local count = 25
Server C: local count = 20

Every 10 seconds → sync to Redis → total = 75
```

**Pros:** Low latency (local check), reduces Redis load. **Cons:** Approximate, can exceed limit temporarily.

---

### Sticky sessions

Route each user to the same server (via load balancer).

```
User 123 → always Server A
User 456 → always Server B
```

**Pros:** Simple, no coordination needed. **Cons:** Uneven load distribution, doesn't work with auto-scaling.

---

## 5. Rate Limiting Strategies

### Per-user rate limiting

Each user has their own limit.

```
User A: 1,000 req/hour
User B: 1,000 req/hour
User C: 1,000 req/hour
```

**Use for:** Authenticated APIs, SaaS products.

---

### Per-IP rate limiting

Limit by client IP address.

```
IP 1.2.3.4: 100 req/min
IP 5.6.7.8: 100 req/min
```

**Use for:** Public APIs, unauthenticated endpoints.

**Problem:** NAT and proxies — many users share one IP.

---

### Per-API-key rate limiting

Each API key has a limit (often tiered).

```
Free tier:     1,000 req/day
Pro tier:      100,000 req/day
Enterprise:    Unlimited
```

**Use for:** Public APIs with pricing tiers.

---

### Global rate limiting

Limit total traffic to the service.

```
Service-wide: 100,000 req/sec across all users
```

**Use for:** Protecting backend capacity.

---

### Tiered rate limiting

Different limits for different endpoints.

```
GET /api/users:     1,000 req/min  (cheap)
POST /api/reports:  10 req/min     (expensive)
```

---

## 6. Response to Rate Limit Exceeded

### HTTP 429 Too Many Requests

Standard response when limit exceeded.

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640000000

{
  "error": "Rate limit exceeded. Try again in 60 seconds."
}
```

### Headers

```
X-RateLimit-Limit:     Maximum requests allowed in window
X-RateLimit-Remaining: Requests remaining in current window
X-RateLimit-Reset:     Unix timestamp when limit resets
Retry-After:           Seconds until client can retry
```

### Client behavior

```python
response = requests.get(url)

if response.status_code == 429:
    retry_after = int(response.headers.get('Retry-After', 60))
    time.sleep(retry_after)
    response = requests.get(url)  # retry
```

---

## 7. Advanced Techniques

### Adaptive rate limiting

Adjust limits based on system load.

```
Normal load:  1,000 req/min per user
High load:    500 req/min per user (throttle)
Critical:     100 req/min per user (aggressive throttle)
```

**Trigger:** CPU > 80%, queue depth > 10,000, error rate > 5%.

---

### Priority-based rate limiting

Different limits for different user tiers.

```
Free users:       100 req/hour
Paid users:       10,000 req/hour
Enterprise:       No limit
```

---

### Quota-based rate limiting

Long-term quotas (monthly) in addition to short-term rate limits.

```
Rate limit:  1,000 req/min
Quota:       10,000,000 req/month

User can burst to 1,000 req/min, but total monthly usage capped
```

---

### Concurrency limiting

Limit concurrent requests instead of rate.

```
User A: max 10 concurrent requests
If 10 requests in-flight → reject new requests until one completes
```

**Use for:** Expensive operations (report generation, large queries).

---

## 8. Rate Limiting in Microservices

### Gateway-level rate limiting

API Gateway enforces limits before routing to services.

```
Client → [API Gateway + Rate Limiter] → Service A
                                      → Service B
                                      → Service C
```

**Pros:** Centralized, protects all services. **Cons:** Gateway is a bottleneck.

---

### Service-level rate limiting

Each service enforces its own limits.

```
Client → API Gateway → Service A [Rate Limiter]
                    → Service B [Rate Limiter]
```

**Pros:** Fine-grained control, no single point of failure. **Cons:** More complex, harder to coordinate.

---

### Hybrid approach

Gateway enforces coarse limits, services enforce fine-grained limits.

```
Gateway: 10,000 req/min per user (prevent abuse)
Service A: 100 req/min for expensive endpoint (protect service)
```

---

## 9. Common Interview Questions + Answers

### Q: How would you implement rate limiting for a public API?

> "I'd use a sliding window counter algorithm with Redis as the backing store. Each API key gets a counter in Redis with a key like `rate_limit:{api_key}:{current_minute}`. On each request, I increment the counter and check if it exceeds the limit. I'd also track the previous minute's count to smooth the window. The API Gateway would enforce this before routing to backend services. I'd return HTTP 429 with Retry-After and X-RateLimit headers when the limit is exceeded. For different pricing tiers, I'd store the limit per API key in a database and look it up on each request."

### Q: What's the difference between token bucket and leaky bucket?

> "Token bucket allows bursts up to the bucket capacity, then refills tokens at a constant rate. It's good for APIs that need to allow occasional bursts but limit sustained rate. Leaky bucket processes requests at a constant rate regardless of input rate — it smooths bursty traffic into a steady stream. Token bucket is more common for API rate limiting because it's more user-friendly — users can burst when needed. Leaky bucket is better for smoothing traffic to a backend that can't handle bursts."

### Q: How do you handle rate limiting in a distributed system?

> "I'd use Redis as a centralized counter that all servers check. Each server increments a shared counter in Redis and checks if it exceeds the limit. This ensures consistency across servers. To reduce Redis load and latency, I could use local counters that sync periodically, accepting that the limit might be slightly exceeded temporarily. For very high scale, I'd shard the rate limiter by user ID — different Redis instances handle different users, reducing load on any single instance."

### Q: What happens if Redis goes down?

> "I'd have a Redis replica for failover. If both fail, I have two options: fail open (allow all requests, risking overload) or fail closed (reject all requests, ensuring protection). The choice depends on the use case. For a public API under attack, fail closed. For an internal API, fail open. I'd also implement circuit breakers — if Redis is consistently failing, bypass rate limiting temporarily and alert ops."

---

## 10. Interview Tricks & Pitfalls

### ✅ Trick 1: Always mention the algorithm

Don't just say "I'll use rate limiting." Specify:

> "I'll use a sliding window counter algorithm — it's memory efficient like fixed window but avoids the boundary burst problem. It's the best balance of accuracy and performance for most use cases."

### ✅ Trick 2: Discuss the response

Rate limiting isn't just rejecting requests:

> "When the limit is exceeded, I'll return HTTP 429 with Retry-After and X-RateLimit-* headers so clients know when they can retry. I'd also log these events to detect abuse patterns."

### ✅ Trick 3: Consider different dimensions

Show you understand rate limiting is multi-dimensional:

> "I'd apply rate limiting at multiple levels: per-user (1,000 req/min), per-IP (100 req/min for unauthenticated), and global (100,000 req/sec service-wide). This protects against both individual abuse and coordinated attacks."

### ❌ Pitfall 1: Forgetting about distributed systems

If you have multiple servers, you need coordinated rate limiting. Don't assume a single server.

### ❌ Pitfall 2: Not handling Redis failure

If you propose Redis-based rate limiting, the interviewer will ask "what if Redis fails?" Have a failover strategy.

### ❌ Pitfall 3: Ignoring the user experience

Rate limiting should be transparent and helpful. Return proper headers, clear error messages, and reasonable limits.

---

## 11. Quick Reference

```
Rate Limiting = control request rate to prevent overload and abuse

Algorithms:
  Fixed Window:         Simple, but boundary burst problem
  Sliding Window Log:   Accurate, but memory intensive
  Sliding Window Counter: Best balance (most common)
  Token Bucket:         Allows bursts, smooth refill
  Leaky Bucket:         Constant output rate

Distributed:
  Centralized (Redis):  Accurate, consistent
  Local + sync:         Low latency, approximate
  Sticky sessions:      Simple, but limits scaling

Strategies:
  Per-user:    Each user has own limit
  Per-IP:      Limit by IP (watch for NAT)
  Per-API-key: Tiered limits (free/pro/enterprise)
  Global:      Service-wide capacity protection
  Tiered:      Different limits per endpoint

Response:
  HTTP 429 Too Many Requests
  Headers: X-RateLimit-Limit, X-RateLimit-Remaining, Retry-After

Advanced:
  Adaptive:     Adjust limits based on system load
  Priority:     Different limits per user tier
  Quota:        Long-term limits (monthly)
  Concurrency:  Limit concurrent requests

Implementation:
  API Gateway:  Centralized enforcement
  Service-level: Fine-grained control
  Hybrid:       Gateway + service limits
```
