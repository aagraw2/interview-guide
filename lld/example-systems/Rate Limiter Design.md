
## 1. Functional Requirements

- Limit number of requests per client (user/IP/API key)
    
- Support configurable rate limits (e.g., 100 req/min)
    
- Allow/Reject incoming requests based on limit
    
- Support multiple rate limiting strategies (Token Bucket, Fixed Window, etc.)
    
- Reset limits based on time window
    
- Build flexible configurations for different clients
    

---

## 2. Non-Functional Requirements

- Low latency decision per request (O(1))
    
- High concurrency support
    
- Scalable for distributed systems
    
- Extensible for new rate limiting strategies
    
- Thread-safe operations
    
- Configurable and maintainable
    

---

## 3. Core Entities

|Entity|Responsibility|
|---|---|
|Client|Represents requester (user/IP/API key)|
|RateLimitConfig|Holds limit, refill rate, and window config|
|RateLimiter|Interface for rate limiting logic|
|TokenBucket|Token bucket implementation|
|FixedWindowCounter|Fixed window implementation|
|RateLimiterFactory|Creates appropriate limiter|
|RateLimiterService|Handles request validation|

---

## 4. Flows (Use Cases)

### Allow Request (Main Flow)

1. Request arrives with client identifier
    
2. Fetch RateLimitConfig for client
    
3. Fetch corresponding RateLimiter
    
4. Call `allowRequest()`
    
5. If allowed → forward request
    
6. If rejected → block request
    

---

### Token Refill (Supporting Flow)

1. Calculate elapsed time since last refill
    
2. Add tokens based on refill rate
    
3. Cap tokens to max capacity
    

---

### Fixed Window Reset (Supporting Flow)

1. Check if current window expired
    
2. Reset counter if expired
    
3. Continue counting in new window
    

---

### Config Creation (Supporting Flow)

1. Use Builder to construct RateLimitConfig
    
2. Set required parameters (capacity, refill rate, window)
    
3. Build immutable config object
    

---

## 5. Code

```java
import java.util.*;

// MODELS

// Builder Pattern: flexible and readable config creation
class RateLimitConfig {
    final int capacity;
    final int refillRate;
    final int windowSizeInSeconds;

    private RateLimitConfig(Builder builder) {
        this.capacity = builder.capacity;
        this.refillRate = builder.refillRate;
        this.windowSizeInSeconds = builder.windowSizeInSeconds;
    }

    public static class Builder {
        private int capacity;
        private int refillRate;
        private int windowSizeInSeconds;

        public Builder setCapacity(int capacity) {
            this.capacity = capacity;
            return this;
        }

        public Builder setRefillRate(int refillRate) {
            this.refillRate = refillRate;
            return this;
        }

        public Builder setWindowSize(int windowSizeInSeconds) {
            this.windowSizeInSeconds = windowSizeInSeconds;
            return this;
        }

        public RateLimitConfig build() {
            return new RateLimitConfig(this);
        }
    }
}

// Strategy Pattern
interface RateLimiter {
    boolean allowRequest(String clientId);
}

// Token Bucket Implementation
class TokenBucketRateLimiter implements RateLimiter {

    private class Bucket {
        int tokens;
        long lastRefillTime;

        Bucket(int capacity) {
            this.tokens = capacity;
            this.lastRefillTime = System.currentTimeMillis();
        }
    }

    private Map<String, Bucket> buckets = new HashMap<>();
    private int capacity;
    private int refillRate;

    public TokenBucketRateLimiter(int capacity, int refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
    }

    @Override
    public synchronized boolean allowRequest(String clientId) {
        Bucket bucket = buckets.computeIfAbsent(clientId, k -> new Bucket(capacity));

        refill(bucket);

        if (bucket.tokens > 0) {
            bucket.tokens--;
            return true;
        }
        return false;
    }

    private void refill(Bucket bucket) {
        long now = System.currentTimeMillis();
        long elapsed = (now - bucket.lastRefillTime) / 1000;

        int tokensToAdd = (int) (elapsed * refillRate);
        if (tokensToAdd > 0) {
            bucket.tokens = Math.min(capacity, bucket.tokens + tokensToAdd);
            bucket.lastRefillTime = now;
        }
    }
}

// Fixed Window Implementation
class FixedWindowRateLimiter implements RateLimiter {

    private class Window {
        int count;
        long startTime;
    }

    private Map<String, Window> windows = new HashMap<>();
    private int capacity;
    private int windowSize;

    public FixedWindowRateLimiter(int capacity, int windowSize) {
        this.capacity = capacity;
        this.windowSize = windowSize;
    }

    @Override
    public synchronized boolean allowRequest(String clientId) {
        long now = System.currentTimeMillis();

        Window window = windows.computeIfAbsent(clientId, k -> {
            Window w = new Window();
            w.startTime = now;
            return w;
        });

        if (now - window.startTime > windowSize * 1000) {
            window.count = 0;
            window.startTime = now;
        }

        if (window.count < capacity) {
            window.count++;
            return true;
        }
        return false;
    }
}

// Factory Pattern
class RateLimiterFactory {
    public static RateLimiter createLimiter(String type, RateLimitConfig config) {
        switch (type) {
            case "TOKEN_BUCKET":
                return new TokenBucketRateLimiter(config.capacity, config.refillRate);
            case "FIXED_WINDOW":
                return new FixedWindowRateLimiter(config.capacity, config.windowSizeInSeconds);
            default:
                throw new IllegalArgumentException("Invalid limiter type");
        }
    }
}

// SERVICES

// Singleton Pattern
class RateLimiterService {
    private static RateLimiterService instance;

    private Map<String, RateLimiter> limiterMap = new HashMap<>();

    private RateLimiterService() {}

    public static RateLimiterService getInstance() {
        if (instance == null) {
            synchronized (RateLimiterService.class) {
                if (instance == null) {
                    instance = new RateLimiterService();
                }
            }
        }
        return instance;
    }

    // Register client with builder config
    public void registerClient(String clientId, String type, RateLimitConfig config) {
        RateLimiter limiter = RateLimiterFactory.createLimiter(type, config);
        limiterMap.put(clientId, limiter);
    }

    // Main flow
    public boolean allowRequest(String clientId) {
        RateLimiter limiter = limiterMap.get(clientId);
        if (limiter == null) throw new RuntimeException("Client not registered");

        return limiter.allowRequest(clientId);
    }
}
```

---

## 6. Complexity Analysis

### Allow Request

- HashMap lookup → O(1)
    
- Strategy execution → O(1)
    

**Overall: O(1)**

---

### Token Refill

- Time computation → O(1)
    
- Token update → O(1)
    

**Overall: O(1)**

---

### Fixed Window Reset

- Window check → O(1)
    
- Reset → O(1)
    

**Overall: O(1)**

---

### Config Build

- Builder object creation → O(1)
    

**Overall: O(1)**

---

## 7. Key Design Decisions

- Builder Pattern ensures clean and flexible configuration creation
    
- Strategy Pattern allows pluggable rate limiting algorithms
    
- Factory Pattern centralizes limiter instantiation
    
- Singleton ensures single service instance
    
- Per-client limiter isolation improves scalability
    
- In-memory storage ensures low latency; extensible to Redis for distributed setups