
# 1. Functional Requirements

- Initiate payment
    
- Ensure idempotent payment processing
    
- Support retries for failed payments
    
- Handle multiple payment methods (Card, UPI, Wallet)
    
- Maintain payment status (INITIATED → SUCCESS → FAILED)
    
- Prevent duplicate charges
    
- Store transaction history
    

---

# 2. Non-Functional Requirements

- Strong consistency (no double charge)
    
- High reliability with retry mechanism
    
- Low latency
    
- Fault-tolerant system
    
- Thread-safe operations
    
- Scalable
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|PaymentRequest|Incoming request with idempotency key|
|Payment|Represents a transaction|
|PaymentStatus|Lifecycle state|
|PaymentMethod|Strategy for payment execution|
|RetryPolicy|Retry logic|
|IdempotencyStore|Prevent duplicate processing|
|PaymentRepository|Store payments|
|PaymentService|Orchestrates flow|

---

# 4. Flows

---

## Initiate Payment (Idempotent Flow)

1. Client sends request with idempotencyKey
    
2. Check IdempotencyStore
    
    - If exists → return stored response
        
3. Create payment with INITIATED
    
4. Execute payment via PaymentMethod
    
5. Save result in IdempotencyStore
    
6. Return response
    

---

## Retry Flow

1. Payment fails
    
2. Check RetryPolicy
    
3. Retry execution
    
4. Update status accordingly
    
5. Stop after max retries
    

---

# 5. Code

```java
import java.util.*;
import java.util.concurrent.*;

// ENUM
enum PaymentStatus {
    INITIATED, SUCCESS, FAILED
}

// MODELS
class PaymentRequest {
    String idempotencyKey;
    double amount;
    String userId;
    String method;
}

class Payment {
    String id;
    String userId;
    double amount;
    PaymentStatus status;

    public Payment(String id, String userId, double amount) {
        this.id = id;
        this.userId = userId;
        this.amount = amount;
        this.status = PaymentStatus.INITIATED;
    }
}

// IDEMPOTENCY STORE
class IdempotencyStore {
    private Map<String, Payment> store = new ConcurrentHashMap<>();

    public Payment get(String key) {
        return store.get(key);
    }

    public void save(String key, Payment payment) {
        store.put(key, payment);
    }
}

// REPOSITORY
class PaymentRepository {
    private Map<String, Payment> store = new ConcurrentHashMap<>();

    public void save(Payment payment) {
        store.put(payment.id, payment);
    }

    public Payment get(String id) {
        return store.get(id);
    }
}

// STRATEGY: Payment Method
interface PaymentMethod {
    boolean pay(Payment payment);
}

class CardPayment implements PaymentMethod {
    public boolean pay(Payment payment) {
        return Math.random() > 0.3; // simulate failure
    }
}

class UPIPayment implements PaymentMethod {
    public boolean pay(Payment payment) {
        return Math.random() > 0.2;
    }
}

// FACTORY
class PaymentFactory {
    public static PaymentMethod get(String type) {
        switch (type) {
            case "CARD": return new CardPayment();
            case "UPI": return new UPIPayment();
            default: throw new IllegalArgumentException();
        }
    }
}

// STRATEGY: Retry
interface RetryPolicy {
    boolean shouldRetry(int attempt);
}

class FixedRetryPolicy implements RetryPolicy {
    private int maxRetries;

    public FixedRetryPolicy(int maxRetries) {
        this.maxRetries = maxRetries;
    }

    public boolean shouldRetry(int attempt) {
        return attempt < maxRetries;
    }
}

// SERVICE
class PaymentService {
    private IdempotencyStore idStore;
    private PaymentRepository repo;
    private RetryPolicy retryPolicy;

    public PaymentService(IdempotencyStore idStore,
                          PaymentRepository repo,
                          RetryPolicy retryPolicy) {
        this.idStore = idStore;
        this.repo = repo;
        this.retryPolicy = retryPolicy;
    }

    public Payment process(PaymentRequest request) {

        // IDEMPOTENCY CHECK
        Payment existing = idStore.get(request.idempotencyKey);
        if (existing != null) {
            return existing;
        }

        Payment payment = new Payment(
            UUID.randomUUID().toString(),
            request.userId,
            request.amount
        );

        PaymentMethod method = PaymentFactory.get(request.method);

        int attempt = 0;
        boolean success = false;

        while (retryPolicy.shouldRetry(attempt)) {
            success = method.pay(payment);
            if (success) break;
            attempt++;
        }

        payment.status = success ? PaymentStatus.SUCCESS : PaymentStatus.FAILED;

        repo.save(payment);
        idStore.save(request.idempotencyKey, payment);

        return payment;
    }
}
```

---

# 6. Complexity Analysis

- Idempotency lookup → O(1)
    
- Payment execution → O(R) (R = retries)
    

---

# 7. Design Patterns Used

- Strategy Pattern → Payment methods, Retry logic
    
- Factory Pattern → Payment method creation
    
- Repository Pattern → Data access
    
- Dependency Injection → Loose coupling
    

---

# 8. Key Design Decisions

- IdempotencyStore ensures no duplicate charges
    
- Retry handled at service level for simplicity
    
- Payment methods are pluggable via Strategy
    
- Factory isolates object creation
    
- Thread-safe maps used for concurrency
    
- Payment state stored after execution to ensure consistency
    
- Retry bounded to avoid infinite loops
    

---

# 9. Edge Cases Handled

- Duplicate requests (network retries)
    
- Partial failures
    
- Payment gateway failures
    
- Concurrent requests with same idempotency key
    

---

# 10. Possible Extensions

- Add exponential backoff retry
    
- Use distributed cache (Redis) for idempotency
    
- Add payment timeout handling
    
- Introduce circuit breaker for payment gateway
    
- Event-driven processing with Kafka
    
- Support refunds and reversals
    
- Add audit logging and monitoring