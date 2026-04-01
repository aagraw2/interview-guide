## 1. What Is a Circuit Breaker?

A circuit breaker prevents cascading failures by stopping requests to a failing service, giving it time to recover.

**Analogy:** Like an electrical circuit breaker that trips when there's too much current, preventing damage to the electrical system.

```
Without circuit breaker:
  Service A → Service B (failing) → timeout after 30s
  100 requests/sec → 100 threads waiting → Service A overwhelmed

With circuit breaker:
  Service A → Circuit Breaker → Service B (failing)
  Circuit breaker detects failures → stops sending requests
  Service A fails fast (no waiting) → remains healthy
```

---

## 2. Circuit Breaker States

A circuit breaker has three states:

### Closed (Normal operation)

```
Requests flow through normally
  Client → [Circuit Breaker: CLOSED] → Service
                                      ← Success

Circuit breaker tracks failures
If failure rate < threshold → stay closed
```

---

### Open (Blocking requests)

```
Too many failures detected → circuit opens
  Client → [Circuit Breaker: OPEN] → ✗ (request blocked)
           ← Fail fast (immediate error)

No requests sent to service
Service gets time to recover
```

---

### Half-Open (Testing recovery)

```
After timeout, circuit enters half-open state
  Client → [Circuit Breaker: HALF-OPEN] → Service (test request)
                                         ← Success?

If test succeeds → close circuit (resume normal operation)
If test fails → open circuit again (continue blocking)
```

---

## 3. State Transitions

```
                    ┌─────────┐
                    │ CLOSED  │ (normal)
                    └────┬────┘
                         │
                  Failure threshold
                    exceeded
                         │
                         ▼
                    ┌─────────┐
              ┌────▶│  OPEN   │ (blocking)
              │     └────┬────┘
              │          │
              │     Timeout expires
              │          │
              │          ▼
              │     ┌─────────┐
              │     │HALF-OPEN│ (testing)
              │     └────┬────┘
              │          │
              │    ┌─────┴─────┐
              │    │           │
         Test fails      Test succeeds
              │                │
              └────────────────┘
                                │
                                ▼
                           Back to CLOSED
```

---

## 4. Configuration Parameters

### Failure threshold

How many failures before opening the circuit?

```
Threshold: 5 failures in 10 seconds

Failures: 1, 2, 3, 4, 5 → Circuit opens
```

**Or percentage-based:**

```
Threshold: 50% failure rate in 10 seconds

10 requests: 6 failures, 4 successes → 60% failure rate → Circuit opens
```

---

### Timeout

How long to wait before testing recovery?

```
Circuit opens at 10:00:00
Timeout: 30 seconds
Circuit enters half-open at 10:00:30
```

**Too short:** Service hasn't recovered, circuit opens again immediately. **Too long:** Service recovered but circuit still blocking.

**Typical:** 30-60 seconds.

---

### Success threshold (half-open)

How many successful test requests before closing?

```
Half-open state:
  Test request 1: Success
  Test request 2: Success
  Test request 3: Success
  → 3 successes → Close circuit

Or:
  Test request 1: Success
  Test request 2: Failure
  → Open circuit again
```

---

## 5. Implementation Example

### Python with retry and circuit breaker

```python
from enum import Enum
import time

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=30, success_threshold=2):
        self.failure_threshold = failure_threshold
        self.timeout = timeout  # seconds
        self.success_threshold = success_threshold
        
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED
    
    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            # Check if timeout expired
            if time.time() - self.last_failure_time >= self.timeout:
                self.state = CircuitState.HALF_OPEN
                self.success_count = 0
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e
    
    def _on_success(self):
        self.failure_count = 0
        
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.success_threshold:
                self.state = CircuitState.CLOSED
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

# Usage
breaker = CircuitBreaker(failure_threshold=3, timeout=30)

def call_payment_service():
    response = requests.post("http://payment-service/charge", ...)
    return response.json()

try:
    result = breaker.call(call_payment_service)
except Exception as e:
    # Circuit open or service failed
    return fallback_response()
```

---

## 6. Circuit Breaker with Fallback

When circuit is open, return a fallback response.

```python
def get_user_recommendations(user_id):
    try:
        # Try recommendation service
        return breaker.call(recommendation_service.get, user_id)
    except CircuitBreakerOpen:
        # Fallback: return popular items
        return get_popular_items()
    except Exception:
        # Service failed
        return get_popular_items()
```

**Graceful degradation:** Service continues with reduced functionality.

---

## 7. Circuit Breaker + Retry + Timeout

Combine patterns for robust error handling.

```python
@retry(max_attempts=3, backoff=exponential)
@timeout(seconds=5)
@circuit_breaker(failure_threshold=5, timeout=30)
def call_external_service():
    return requests.get("http://external-service/api")

Flow:
  1. Timeout: Fail if request takes > 5 seconds
  2. Retry: Retry up to 3 times with exponential backoff
  3. Circuit breaker: If 5 failures, stop trying for 30 seconds
```

**Order matters:**
- Timeout (innermost): Prevents hanging requests
- Retry (middle): Retries transient failures
- Circuit breaker (outermost): Prevents cascading failures

---

## 8. Distributed Circuit Breaker

In a distributed system with multiple instances, circuit breaker state must be shared.

### Local circuit breaker (per instance)

```
Instance A: Circuit breaker (local state)
Instance B: Circuit breaker (local state)
Instance C: Circuit breaker (local state)

Each instance tracks failures independently
```

**Problem:** Inconsistent state. Instance A's circuit might be open while B's is closed.

---

### Shared circuit breaker (Redis)

```
All instances → [Redis: shared circuit breaker state]

State stored in Redis:
  - failure_count
  - last_failure_time
  - state (closed/open/half-open)

All instances see same state
```

**Implementation:**

```python
class DistributedCircuitBreaker:
    def __init__(self, redis_client, service_name):
        self.redis = redis_client
        self.key = f"circuit_breaker:{service_name}"
    
    def call(self, func, *args, **kwargs):
        state = self.redis.hget(self.key, "state") or "closed"
        
        if state == "open":
            last_failure = float(self.redis.hget(self.key, "last_failure_time"))
            if time.time() - last_failure >= 30:
                self.redis.hset(self.key, "state", "half_open")
            else:
                raise CircuitBreakerOpen()
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e
    
    def _on_failure(self):
        self.redis.hincrby(self.key, "failure_count", 1)
        self.redis.hset(self.key, "last_failure_time", time.time())
        
        failure_count = int(self.redis.hget(self.key, "failure_count"))
        if failure_count >= 5:
            self.redis.hset(self.key, "state", "open")
```

---

## 9. Circuit Breaker in Microservices

### Service mesh (Istio, Linkerd)

Circuit breaker configured at infrastructure level.

```yaml
# Istio DestinationRule
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

**Benefits:**
- No code changes
- Consistent across all services
- Centralized configuration

---

### Application-level (Resilience4j, Hystrix)

Circuit breaker in application code.

```java
// Resilience4j (Java)
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .slidingWindowSize(10)
    .build();

CircuitBreaker circuitBreaker = CircuitBreaker.of("paymentService", config);

Supplier<String> decoratedSupplier = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> callPaymentService());

String result = Try.ofSupplier(decoratedSupplier)
    .recover(throwable -> "fallback")
    .get();
```

---

## 10. Monitoring Circuit Breakers

Track circuit breaker state and metrics.

```
Metrics to monitor:
  - Circuit state (closed/open/half-open)
  - Failure rate
  - Number of times circuit opened
  - Time spent in open state
  - Fallback invocations

Alerts:
  - Circuit opened (service degraded)
  - Circuit frequently opening/closing (flapping)
  - High fallback rate
```

**Dashboard:**

```
Payment Service Circuit Breaker:
  State: OPEN
  Failure rate: 75% (last 1 minute)
  Time in open state: 45 seconds
  Fallback invocations: 1,234
```

---

## 11. Common Pitfalls

### Pitfall 1: Circuit breaker per instance

```
Bad:
  Each instance has local circuit breaker
  Instance A: circuit open
  Instance B: circuit closed
  Load balancer sends traffic to B → B gets overwhelmed

Good:
  Shared circuit breaker state (Redis)
  All instances see same state
```

---

### Pitfall 2: No fallback

```
Bad:
  Circuit opens → return 503 error → user sees error page

Good:
  Circuit opens → return fallback (cached data, default response)
  User sees degraded but functional service
```

---

### Pitfall 3: Too aggressive threshold

```
Bad:
  Threshold: 1 failure → circuit opens
  One transient error → circuit opens → service unavailable

Good:
  Threshold: 5 failures in 10 seconds
  Tolerates occasional errors
```

---

## 12. Common Interview Questions + Answers

### Q: What is a circuit breaker and why is it useful?

> "A circuit breaker prevents cascading failures in distributed systems. When a service starts failing, the circuit breaker detects it and stops sending requests to that service, failing fast instead of waiting for timeouts. This prevents the calling service from being overwhelmed by threads waiting for responses. The circuit breaker has three states: closed (normal), open (blocking requests), and half-open (testing recovery). After a timeout, it tests if the service has recovered. If yes, it closes the circuit and resumes normal operation. This pattern is critical for resilience in microservices architectures."

### Q: How does a circuit breaker differ from a retry mechanism?

> "Retries attempt the same request multiple times when it fails, hoping a transient error will resolve. Circuit breakers stop trying after detecting a pattern of failures, giving the failing service time to recover. They're complementary — use retries for transient errors (network blips) and circuit breakers for sustained failures (service down). The order matters: timeout (innermost) prevents hanging, retry (middle) handles transient errors, circuit breaker (outermost) prevents cascading failures. Without a circuit breaker, retries can make things worse by overwhelming an already struggling service."

### Q: What happens when a circuit breaker is open?

> "When the circuit is open, requests are immediately rejected without calling the downstream service. This is called 'failing fast.' Instead of waiting for a timeout, the calling service gets an immediate error and can return a fallback response — like cached data, default values, or a degraded experience. After a configured timeout (typically 30-60 seconds), the circuit enters half-open state and sends a test request. If it succeeds, the circuit closes and normal operation resumes. If it fails, the circuit stays open for another timeout period."

### Q: How do you implement a circuit breaker in a distributed system with multiple instances?

> "In a distributed system, circuit breaker state must be shared across instances, otherwise each instance tracks failures independently and you get inconsistent behavior. I'd use Redis to store the shared state — failure count, last failure time, and current state. All instances check and update this shared state. When one instance detects enough failures to open the circuit, all instances immediately see the open state and stop sending requests. This ensures consistent behavior across the cluster. The tradeoff is the extra Redis call, but it's worth it for consistency."

---

## 13. Interview Tricks & Pitfalls

### ✅ Trick 1: Mention the three states

Show you understand the full lifecycle:

> "The circuit breaker has three states: closed (normal operation), open (blocking requests after failures), and half-open (testing recovery). After a timeout in the open state, it enters half-open and sends test requests. If they succeed, it closes. If they fail, it opens again."

### ✅ Trick 2: Discuss fallbacks

Circuit breakers are more useful with fallbacks:

> "When the circuit is open, I'd return a fallback response instead of just an error. For recommendations, return popular items. For user profiles, return cached data. This provides graceful degradation — the service continues with reduced functionality rather than failing completely."

### ✅ Trick 3: Combine with other patterns

Shows you understand resilience holistically:

> "I'd combine circuit breaker with timeout and retry. Timeout prevents hanging requests, retry handles transient errors, and circuit breaker prevents cascading failures. The circuit breaker is the outermost layer — it tracks the overall health of the downstream service."

### ❌ Pitfall 1: Not mentioning distributed state

If you have multiple instances, address shared state:

> "With multiple instances, I'd use Redis to share circuit breaker state. Otherwise, each instance tracks failures independently and you get inconsistent behavior — one instance's circuit might be open while another's is closed."

### ❌ Pitfall 2: Forgetting about monitoring

Circuit breakers need visibility:

> "I'd monitor circuit breaker state and alert when circuits open. This indicates a downstream service is failing. I'd also track how often circuits open and close — frequent flapping indicates the threshold is too sensitive."

### ❌ Pitfall 3: Not discussing configuration

Show you understand the parameters:

> "I'd configure a failure threshold of 5 failures in 10 seconds — enough to tolerate occasional errors but quick enough to detect sustained failures. The timeout would be 30 seconds — long enough for the service to recover but short enough to resume quickly. In half-open state, I'd require 2 successful requests before closing the circuit."

---

## 14. Quick Reference

```
Circuit Breaker = prevent cascading failures by stopping requests to failing service

Three states:
  CLOSED:     Normal operation, requests flow through
  OPEN:       Blocking requests, fail fast
  HALF-OPEN:  Testing recovery with limited requests

State transitions:
  CLOSED → OPEN:      Failure threshold exceeded
  OPEN → HALF-OPEN:   Timeout expires
  HALF-OPEN → CLOSED: Test requests succeed
  HALF-OPEN → OPEN:   Test requests fail

Configuration:
  Failure threshold:  5 failures in 10 seconds (or 50% failure rate)
  Timeout:            30-60 seconds (time before testing recovery)
  Success threshold:  2-3 successful tests before closing

Benefits:
  - Prevents cascading failures
  - Fails fast (no waiting for timeouts)
  - Gives failing service time to recover
  - Enables graceful degradation with fallbacks

Fallback strategies:
  - Cached data
  - Default values
  - Degraded functionality
  - Error message (last resort)

Distributed systems:
  - Share state via Redis
  - All instances see same circuit state
  - Consistent behavior across cluster

Combine with:
  - Timeout (prevent hanging)
  - Retry (handle transient errors)
  - Fallback (graceful degradation)

Monitoring:
  - Circuit state (closed/open/half-open)
  - Failure rate
  - Times circuit opened
  - Fallback invocations

Tools:
  - Resilience4j (Java)
  - Hystrix (Java, deprecated but widely known)
  - Polly (.NET)
  - Service mesh (Istio, Linkerd)

Common pitfalls:
  - Local state per instance (use shared state)
  - No fallback (return error instead of degraded response)
  - Too aggressive threshold (opens on single error)

Key insight: Circuit breakers enable resilience by failing fast and allowing recovery
```
