## 1. What is Retry with Exponential Backoff?

Retry with exponential backoff is a strategy where failed requests are retried with increasing delays between attempts. The delay doubles (or increases exponentially) after each failure.

```
Attempt 1: Immediate
Attempt 2: Wait 1 second
Attempt 3: Wait 2 seconds
Attempt 4: Wait 4 seconds
Attempt 5: Wait 8 seconds

Formula: delay = base_delay × 2^(attempt - 1)
```

**Purpose:** Prevent overwhelming a failing service, avoid thundering herd, give service time to recover.

---

## 2. Why Exponential Backoff?

### Without retry (bad)

```
Request fails → Return error immediately
User sees error → Poor experience
Transient failures not handled
```

### With constant retry (bad)

```
Request fails → Retry every 1 second
Service is down → 1000 clients retry every 1 second
→ 1000 requests/second hammering the service
→ Service can't recover (thundering herd)
```

### With exponential backoff (good)

```
Request fails → Retry with increasing delays
1000 clients retry:
  After 1s: 1000 requests
  After 2s: 500 requests (half gave up or succeeded)
  After 4s: 250 requests
  After 8s: 125 requests
→ Load decreases over time
→ Service has time to recover
```

---

## 3. Exponential Backoff Formula

### Basic formula

```
delay = base_delay × 2^(attempt - 1)

Example (base_delay = 1 second):
  Attempt 1: 1 × 2^0 = 1 second
  Attempt 2: 1 × 2^1 = 2 seconds
  Attempt 3: 1 × 2^2 = 4 seconds
  Attempt 4: 1 × 2^3 = 8 seconds
  Attempt 5: 1 × 2^4 = 16 seconds
```

### With maximum delay

```
delay = min(max_delay, base_delay × 2^(attempt - 1))

Example (base_delay = 1s, max_delay = 60s):
  Attempt 1: min(60, 1) = 1 second
  Attempt 2: min(60, 2) = 2 seconds
  Attempt 3: min(60, 4) = 4 seconds
  ...
  Attempt 7: min(60, 64) = 60 seconds
  Attempt 8: min(60, 128) = 60 seconds (capped)
```

### With jitter (randomization)

```
delay = random(0, base_delay × 2^(attempt - 1))

Or:
delay = base_delay × 2^(attempt - 1) × random(0.5, 1.5)

Why jitter?
  Without: All clients retry at the same time (synchronized)
  With: Clients retry at different times (spread out)
  → Prevents thundering herd
```

---

## 4. Implementation

### Python example

```python
import time
import random

def retry_with_backoff(func, max_attempts=5, base_delay=1, max_delay=60):
    for attempt in range(1, max_attempts + 1):
        try:
            return func()
        except Exception as e:
            if attempt == max_attempts:
                raise  # Last attempt, give up
            
            # Calculate delay with exponential backoff
            delay = min(max_delay, base_delay * (2 ** (attempt - 1)))
            
            # Add jitter (randomize ±50%)
            jitter = delay * random.uniform(0.5, 1.5)
            
            print(f"Attempt {attempt} failed: {e}. Retrying in {jitter:.2f}s...")
            time.sleep(jitter)

# Usage
def call_api():
    response = requests.get("https://api.example.com/data")
    response.raise_for_status()
    return response.json()

result = retry_with_backoff(call_api)
```

### Decorator pattern

```python
def retry(max_attempts=5, base_delay=1, max_delay=60):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts:
                        raise
                    
                    delay = min(max_delay, base_delay * (2 ** (attempt - 1)))
                    jitter = delay * random.uniform(0.5, 1.5)
                    time.sleep(jitter)
        return wrapper
    return decorator

@retry(max_attempts=3, base_delay=1)
def call_api():
    response = requests.get("https://api.example.com/data")
    response.raise_for_status()
    return response.json()
```

---

## 5. When to Retry

### Retryable errors (transient failures)

```
✅ Network timeouts
✅ Connection refused (service temporarily down)
✅ 500 Internal Server Error (server bug, might work on retry)
✅ 502 Bad Gateway (upstream service down)
✅ 503 Service Unavailable (overloaded, might recover)
✅ 504 Gateway Timeout (upstream timeout)
✅ Rate limit exceeded (429, retry after cooldown)
```

### Non-retryable errors (permanent failures)

```
❌ 400 Bad Request (invalid input, won't change)
❌ 401 Unauthorized (invalid credentials)
❌ 403 Forbidden (no permission)
❌ 404 Not Found (resource doesn't exist)
❌ 422 Unprocessable Entity (validation error)
❌ Client-side errors (invalid JSON, etc.)
```

### Implementation

```python
RETRYABLE_STATUS_CODES = {500, 502, 503, 504, 429}

def should_retry(exception):
    if isinstance(exception, requests.Timeout):
        return True
    if isinstance(exception, requests.ConnectionError):
        return True
    if isinstance(exception, requests.HTTPError):
        return exception.response.status_code in RETRYABLE_STATUS_CODES
    return False

def retry_with_backoff(func, max_attempts=5):
    for attempt in range(1, max_attempts + 1):
        try:
            return func()
        except Exception as e:
            if not should_retry(e) or attempt == max_attempts:
                raise
            # ... backoff logic
```

---

## 6. Jitter Strategies

### Full jitter

```
delay = random(0, base_delay × 2^attempt)

Example (base_delay = 1s, attempt = 3):
  delay = random(0, 8) seconds
  → Could be anywhere from 0 to 8 seconds

Pros:
  ✅ Maximum spread (best for thundering herd)
  ✅ Simple

Cons:
  ❌ Can be too short (0 seconds)
```

### Equal jitter

```
temp = base_delay × 2^attempt
delay = temp / 2 + random(0, temp / 2)

Example (base_delay = 1s, attempt = 3):
  temp = 8
  delay = 4 + random(0, 4) = 4 to 8 seconds

Pros:
  ✅ Good spread
  ✅ Minimum delay guaranteed (temp / 2)

Cons:
  ❌ Slightly more complex
```

### Decorrelated jitter

```
delay = random(base_delay, previous_delay × 3)

Example:
  Attempt 1: random(1, 3) = 2 seconds
  Attempt 2: random(1, 6) = 4 seconds
  Attempt 3: random(1, 12) = 7 seconds

Pros:
  ✅ Good spread
  ✅ Adapts to previous delay

Cons:
  ❌ More complex
  ❌ Can grow unbounded (need max cap)
```

---

## 7. Retry Budget

Limit the percentage of requests that can be retried to prevent retry storms.

### Problem: Retry amplification

```
Service A calls Service B:
  - 1000 requests/sec
  - 10% failure rate
  - Each request retried 3 times

Without retry budget:
  Original: 1000 requests/sec
  Failures: 100 requests/sec
  Retries: 100 × 3 = 300 requests/sec
  Total: 1300 requests/sec (30% increase)

If Service B is already overloaded:
  → Retries make it worse
  → More failures
  → More retries
  → Cascading failure
```

### Solution: Retry budget

```
Retry budget: 10% of requests

Allowed retries per second:
  1000 × 0.10 = 100 retries/sec

If failures exceed budget:
  → Stop retrying
  → Fail fast
  → Prevent retry storm

Implementation:
  retry_count = 0
  request_count = 0
  
  if retry_count / request_count < 0.10:
      retry()
  else:
      fail_fast()
```

---

## 8. Idempotency

Retries require idempotent operations.

### Non-idempotent (bad)

```
POST /orders
{
  "user_id": 123,
  "item": "Laptop",
  "quantity": 1
}

Request fails after order created but before response sent
Client retries → Creates duplicate order
```

### Idempotent (good)

```
POST /orders
{
  "idempotency_key": "abc123",
  "user_id": 123,
  "item": "Laptop",
  "quantity": 1
}

Server:
  if order_exists(idempotency_key):
      return existing_order
  else:
      create_order()

Retry is safe → Returns existing order
```

---

## 9. Circuit Breaker + Retry

Combine circuit breaker with retry for better resilience.

### Circuit breaker states

```
Closed (normal):
  - Requests pass through
  - Track failure rate
  - If failure rate > threshold → Open

Open (failing):
  - Requests fail immediately (no retry)
  - After timeout → Half-open

Half-open (testing):
  - Allow a few test requests
  - If successful → Closed
  - If failed → Open
```

### Integration with retry

```
def call_with_retry_and_circuit_breaker(func):
    if circuit_breaker.is_open():
        raise CircuitBreakerOpenError("Service unavailable")
    
    try:
        return retry_with_backoff(func)
    except Exception as e:
        circuit_breaker.record_failure()
        raise

Circuit breaker prevents retries when service is known to be down
Retry handles transient failures when service is up
```

---

## 10. AWS SDK Retry Strategy

### Default retry strategy

```
Max attempts: 3
Base delay: 100ms
Max delay: 20 seconds
Backoff: Exponential with full jitter

Retryable errors:
  - Throttling (429)
  - Server errors (500, 502, 503, 504)
  - Connection errors
  - Timeouts

Example:
  Attempt 1: Immediate
  Attempt 2: random(0, 200ms)
  Attempt 3: random(0, 400ms)
```

### Adaptive retry mode

```
Monitors success/failure rate
Adjusts retry behavior dynamically

High success rate:
  → More aggressive retries

High failure rate:
  → Back off more (prevent overload)

Automatically adapts to service health
```

---

## 11. Common Interview Questions + Answers

### Q: Why use exponential backoff instead of constant delay?

> "Constant delay causes the thundering herd problem — if 1000 clients retry every second, the failing service gets hammered with 1000 requests per second and can't recover. Exponential backoff spreads out retries over time. After the first retry, half the clients might succeed or give up, so the second retry has fewer requests. This gives the service time to recover. Adding jitter randomizes retry times so clients don't retry simultaneously."

### Q: What is jitter and why is it important?

> "Jitter is randomization added to the retry delay. Without jitter, all clients that fail at the same time will retry at the same time, creating synchronized spikes of traffic. With jitter, each client waits a slightly different amount of time, spreading out the retries. For example, instead of all clients waiting exactly 4 seconds, they wait between 2 and 6 seconds. This prevents the thundering herd problem and helps the service recover smoothly."

### Q: Which errors should you retry and which shouldn't you?

> "Retry transient failures that might succeed on retry: network timeouts, connection errors, 500/502/503/504 server errors, and 429 rate limits. Don't retry permanent failures that won't change: 400 bad request, 401 unauthorized, 403 forbidden, 404 not found, and validation errors. Retrying permanent failures wastes resources and delays the error response to the user. Always check the error type before retrying."

### Q: What is a retry budget and why is it needed?

> "A retry budget limits the percentage of requests that can be retried, typically 10-20%. Without a budget, retries can amplify load — if 10% of requests fail and each is retried 3 times, that's 30% more load on an already struggling service. This can cause a retry storm where retries make the problem worse. A retry budget prevents this by failing fast once the retry limit is reached, protecting the service from retry amplification."

---

## 12. Interview Tricks & Pitfalls

### ✅ Trick 1: Always mention jitter

Jitter prevents thundering herd. Always mention it when discussing exponential backoff.

### ✅ Trick 2: Know which errors to retry

Transient vs permanent failures is a common interview question. Know the difference.

### ✅ Trick 3: Discuss idempotency

Retries require idempotent operations. Mentioning idempotency keys shows depth.

### ❌ Pitfall 1: Retrying non-idempotent operations

Retrying POST /orders without idempotency key creates duplicates. Always mention idempotency.

### ❌ Pitfall 2: Unlimited retries

Always set a maximum number of attempts. Unlimited retries can cause infinite loops.

### ❌ Pitfall 3: Forgetting retry budget

Retries can amplify load. Mention retry budget to prevent retry storms.

---

## 13. Quick Reference

```
What is exponential backoff?
  Retry with increasing delays: 1s, 2s, 4s, 8s, 16s, ...
  Formula: delay = base_delay × 2^(attempt - 1)

Why exponential?
  Constant delay → Thundering herd (all clients retry together)
  Exponential → Spread out retries (give service time to recover)

Jitter:
  Randomize delay to prevent synchronized retries
  Full jitter: random(0, delay)
  Equal jitter: delay/2 + random(0, delay/2)
  Prevents thundering herd

When to retry:
  ✅ Network timeouts, connection errors
  ✅ 500, 502, 503, 504 (server errors)
  ✅ 429 (rate limit, retry after cooldown)
  ❌ 400, 401, 403, 404 (client errors)
  ❌ Validation errors

Implementation:
  max_attempts = 5
  base_delay = 1 second
  max_delay = 60 seconds
  Add jitter (randomize ±50%)

Retry budget:
  Limit retries to 10-20% of requests
  Prevents retry amplification
  Fail fast when budget exceeded

Idempotency:
  Retries require idempotent operations
  Use idempotency keys for POST/PUT
  Server deduplicates based on key

Circuit breaker + retry:
  Circuit breaker: Fail fast when service is down
  Retry: Handle transient failures
  Combine for better resilience

AWS SDK:
  Max attempts: 3
  Base delay: 100ms
  Exponential with full jitter
  Adaptive mode (adjusts to service health)

Best practices:
  - Always add jitter
  - Set max attempts (typically 3-5)
  - Set max delay (typically 60s)
  - Only retry transient failures
  - Use idempotency keys
  - Implement retry budget
  - Combine with circuit breaker
```
