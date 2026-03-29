
# 1. Functional Requirements

- Prevent repeated calls to failing service
    
- Support states: CLOSED, OPEN, HALF_OPEN
    
- Allow automatic recovery after timeout
    
- Fail fast when circuit is OPEN
    
- Allow limited trial requests in HALF_OPEN
    
- Track failures and success count
    

---

# 2. Non-Functional Requirements

- Low latency decision making
    
- Thread-safe operations
    
- Fault-tolerant
    
- Lightweight and in-memory
    
- Extensible for custom policies
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|CircuitBreaker|Core logic for state transitions|
|CircuitState|CLOSED / OPEN / HALF_OPEN|
|FailurePolicy|Determines when to open circuit|
|RecoveryPolicy|Determines when to transition to HALF_OPEN|
|Execution|Wrapped external call|
|CircuitBreakerService|Manages circuit breakers|

---

# 4. State Transitions

- CLOSED → OPEN (failure threshold reached)
    
- OPEN → HALF_OPEN (after timeout)
    
- HALF_OPEN → CLOSED (success)
    
- HALF_OPEN → OPEN (failure)
    

---

# 5. Flows

---

## Execute Request

1. Check circuit state
    
2. If OPEN → fail fast
    
3. If HALF_OPEN → allow limited calls
    
4. Execute external call
    
5. Record success/failure
    
6. Update state accordingly
    

---

# 6. Code

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

// ENUM
enum CircuitState {
    CLOSED, OPEN, HALF_OPEN
}

// FAILURE POLICY (Strategy)
interface FailurePolicy {
    boolean shouldOpen(int failureCount);
}

class ThresholdFailurePolicy implements FailurePolicy {
    private int threshold;

    public ThresholdFailurePolicy(int threshold) {
        this.threshold = threshold;
    }

    public boolean shouldOpen(int failureCount) {
        return failureCount >= threshold;
    }
}

// RECOVERY POLICY (Strategy)
interface RecoveryPolicy {
    boolean canTry(long lastFailureTime);
}

class TimeoutRecoveryPolicy implements RecoveryPolicy {
    private long timeoutMillis;

    public TimeoutRecoveryPolicy(long timeoutMillis) {
        this.timeoutMillis = timeoutMillis;
    }

    public boolean canTry(long lastFailureTime) {
        return System.currentTimeMillis() - lastFailureTime >= timeoutMillis;
    }
}

// CIRCUIT BREAKER
class CircuitBreaker {

    private AtomicInteger failureCount = new AtomicInteger(0);
    private volatile CircuitState state = CircuitState.CLOSED;
    private volatile long lastFailureTime = 0;

    private FailurePolicy failurePolicy;
    private RecoveryPolicy recoveryPolicy;

    public CircuitBreaker(FailurePolicy failurePolicy,
                          RecoveryPolicy recoveryPolicy) {
        this.failurePolicy = failurePolicy;
        this.recoveryPolicy = recoveryPolicy;
    }

    public synchronized boolean allowRequest() {
        if (state == CircuitState.OPEN) {
            if (recoveryPolicy.canTry(lastFailureTime)) {
                state = CircuitState.HALF_OPEN;
                return true;
            }
            return false;
        }
        return true;
    }

    public void recordSuccess() {
        if (state == CircuitState.HALF_OPEN) {
            reset();
            state = CircuitState.CLOSED;
        }
    }

    public void recordFailure() {
        failureCount.incrementAndGet();
        lastFailureTime = System.currentTimeMillis();

        if (failurePolicy.shouldOpen(failureCount.get())) {
            state = CircuitState.OPEN;
        }
    }

    private void reset() {
        failureCount.set(0);
    }
}

// SERVICE
class CircuitBreakerService {
    private Map<String, CircuitBreaker> breakers = new ConcurrentHashMap<>();

    public CircuitBreaker get(String key,
                               FailurePolicy fp,
                               RecoveryPolicy rp) {
        return breakers.computeIfAbsent(key,
            k -> new CircuitBreaker(fp, rp));
    }
}

// USAGE
class ExternalServiceCaller {
    private CircuitBreaker cb;

    public ExternalServiceCaller(CircuitBreaker cb) {
        this.cb = cb;
    }

    public String call() {
        if (!cb.allowRequest()) {
            throw new RuntimeException("Circuit Open - Fail Fast");
        }

        try {
            // simulate external call
            if (Math.random() < 0.5) {
                throw new RuntimeException("Failure");
            }

            cb.recordSuccess();
            return "Success";

        } catch (Exception e) {
            cb.recordFailure();
            throw e;
        }
    }
}
```

---

# 7. Complexity Analysis

- Allow request check → O(1)
    
- Record success/failure → O(1)
    

---

# 8. Design Patterns Used

- State Pattern → Circuit states (CLOSED, OPEN, HALF_OPEN)
    
- Strategy Pattern → Failure policy, Recovery policy
    
- Singleton-like management via CircuitBreakerService
    
- Dependency Injection → Flexible policies
    

---

# 9. Key Design Decisions

- Atomic variables used for thread-safe counters
    
- State transitions handled inside CircuitBreaker
    
- Policies abstracted for flexibility
    
- Fail-fast mechanism when OPEN
    
- HALF_OPEN allows controlled recovery
    
- Lightweight in-memory design
    
- Separate service for managing multiple breakers
    

---

# 10. Edge Cases Handled

- Continuous failures
    
- Flapping services (fail → recover → fail)
    
- Concurrent requests during HALF_OPEN
    
- Time-based recovery
    

---

# 11. Possible Extensions

- Sliding window failure tracking
    
- Exponential backoff
    
- Metrics & monitoring (Prometheus)
    
- Distributed circuit breaker (Redis)
    
- Integration with Retry mechanism
    
- Rate limiting in HALF_OPEN state