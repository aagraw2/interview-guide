## 1. What is Backpressure?

Backpressure is a mechanism to prevent a fast producer from overwhelming a slow consumer by signaling the producer to slow down or stop sending data.

```
Without backpressure:
  Producer: 10,000 messages/sec
  Consumer: 1,000 messages/sec
  → Queue grows unbounded → Out of memory → System crash

With backpressure:
  Producer: 10,000 messages/sec
  Consumer: 1,000 messages/sec → Signals producer to slow down
  Producer: Reduces to 1,000 messages/sec
  → Queue stays bounded → System stable
```

**Purpose:** Prevent resource exhaustion, maintain system stability, graceful degradation.

---

## 2. Why Backpressure Matters

### Problem: Unbounded queues

```
API server receives 10,000 requests/sec
Worker processes 1,000 requests/sec
Queue grows: 0 → 9,000 → 18,000 → 27,000 → ...

After 10 seconds: 90,000 requests in queue
Memory: 90,000 × 1KB = 90 MB
After 100 seconds: 900,000 requests → 900 MB
→ Out of memory → Crash
```

### Solution: Bounded queues + backpressure

```
Queue capacity: 10,000 requests

When queue is full:
  Option 1: Block producer (wait for space)
  Option 2: Drop new requests (return 503)
  Option 3: Drop old requests (FIFO eviction)

Producer slows down or clients retry later
→ System stays stable
```

---

## 3. Backpressure Strategies

### 1. Block producer

```
Producer tries to add to queue:
  if queue.full():
      wait()  # Block until space available
  queue.add(item)

Pros:
  ✅ No data loss
  ✅ Simple to implement

Cons:
  ❌ Producer blocked (can't do other work)
  ❌ Can cause deadlock
  ❌ Cascading slowdown
```

### 2. Drop new items

```
Producer tries to add to queue:
  if queue.full():
      return Error("Queue full")  # Drop new item
  queue.add(item)

Pros:
  ✅ Producer not blocked
  ✅ Protects system from overload

Cons:
  ❌ Data loss (new items dropped)
  ❌ Client must retry
```

### 3. Drop old items

```
Producer tries to add to queue:
  if queue.full():
      queue.remove_oldest()  # Make space
  queue.add(item)

Pros:
  ✅ Always accept new items
  ✅ Useful for real-time data (old data less valuable)

Cons:
  ❌ Data loss (old items dropped)
  ❌ Not suitable for all use cases
```

### 4. Reject with error

```
HTTP 503 Service Unavailable
Retry-After: 60

Client receives error and retries later

Pros:
  ✅ Client aware of overload
  ✅ Can retry with backoff
  ✅ System protected

Cons:
  ❌ Client must handle retries
  ❌ User experience degraded
```

---

## 4. Backpressure in Different Systems

### HTTP APIs

```
Server overloaded:
  → Return 503 Service Unavailable
  → Include Retry-After header
  → Client backs off and retries

Rate limiting:
  → Return 429 Too Many Requests
  → Include X-RateLimit-Reset header
  → Client waits until rate limit resets

Load shedding:
  → Drop low-priority requests
  → Serve high-priority requests only
```

### Message queues

```
Kafka:
  Producer: buffer.memory = 32 MB
  If buffer full → Block or throw exception
  
  Consumer: max.poll.records = 500
  Consumer controls how many messages to fetch

RabbitMQ:
  Queue length limit: 10,000 messages
  If full → Reject new messages or drop old messages
  
  Consumer prefetch: 100 messages
  Consumer controls how many messages to buffer

SQS:
  Queue depth monitoring
  If depth > threshold → Trigger auto-scaling
  Add more consumers to drain queue
```

### Reactive Streams

```
Publisher-Subscriber model with backpressure:

Subscriber: request(100)  # Request 100 items
Publisher: onNext(item1), onNext(item2), ..., onNext(item100)
Subscriber: request(50)   # Request 50 more
Publisher: onNext(item101), ..., onNext(item150)

Subscriber controls flow rate
Publisher never sends more than requested

Examples: RxJava, Project Reactor, Akka Streams
```

---

## 5. TCP Flow Control (Built-in Backpressure)

### How TCP handles backpressure

```
Receiver has buffer: 64 KB

Sender sends data:
  → Receiver buffer fills up
  → Receiver sends ACK with window size = 0
  → Sender stops sending (backpressure)

Receiver processes data:
  → Buffer has space
  → Receiver sends ACK with window size = 32 KB
  → Sender resumes sending (up to 32 KB)

This is automatic backpressure at the transport layer
```

### Application-level backpressure

```
TCP provides backpressure, but:
  - Only at network level
  - Doesn't know about application semantics
  - Can't prioritize requests

Application-level backpressure:
  - Reject requests before TCP buffer fills
  - Prioritize important requests
  - Return meaningful errors (503, 429)
```

---

## 6. Bounded Queues

### Unbounded queue (bad)

```python
queue = Queue()  # No size limit

while True:
    item = receive_request()
    queue.put(item)  # Always succeeds
    # Queue grows unbounded → Out of memory
```

### Bounded queue (good)

```python
queue = Queue(maxsize=10000)  # Size limit

while True:
    item = receive_request()
    try:
        queue.put(item, block=False)  # Don't block
    except Full:
        return 503  # Queue full, reject request
```

### Bounded queue with timeout

```python
queue = Queue(maxsize=10000)

while True:
    item = receive_request()
    try:
        queue.put(item, timeout=1)  # Wait up to 1 second
    except Full:
        return 503  # Still full after 1 second
```

---

## 7. Load Shedding

Drop low-priority requests when overloaded.

### Priority-based shedding

```
Request priorities:
  Critical: Payment processing
  High: User-facing API
  Medium: Analytics
  Low: Background jobs

When overloaded:
  1. Drop low-priority requests first
  2. Then medium-priority
  3. Keep critical and high-priority

Implementation:
  if queue_depth > 8000:
      if request.priority == LOW:
          return 503
  if queue_depth > 9000:
      if request.priority in [LOW, MEDIUM]:
          return 503
```

### Probabilistic shedding

```
Load factor = current_load / max_capacity

Drop probability = max(0, (load_factor - 0.8) / 0.2)

Examples:
  Load 70%: Drop 0% (0.7 < 0.8)
  Load 80%: Drop 0% ((0.8 - 0.8) / 0.2 = 0)
  Load 90%: Drop 50% ((0.9 - 0.8) / 0.2 = 0.5)
  Load 100%: Drop 100% ((1.0 - 0.8) / 0.2 = 1.0)

Gradually increase drop rate as load increases
```

---

## 8. Backpressure in Microservices

### Service A calls Service B

```
Without backpressure:
  Service A: 10,000 RPS
  Service B: Can handle 1,000 RPS
  → Service B overwhelmed → Crashes
  → Service A gets errors → Retries
  → More load on Service B → Cascading failure

With backpressure:
  Service B: Returns 503 when overloaded
  Service A: Sees 503 → Backs off → Reduces RPS
  → Service B recovers
  → Service A gradually increases RPS
```

### Circuit breaker + backpressure

```
Service A → Circuit Breaker → Service B

Circuit breaker:
  - Monitors Service B error rate
  - If error rate > 50% → Open circuit
  - Stop sending requests to Service B
  - Return error immediately (fail fast)

After timeout:
  - Half-open: Send a few test requests
  - If successful → Close circuit (resume traffic)
  - If failed → Open circuit again

This prevents Service A from overwhelming Service B
```

---

## 9. Monitoring Backpressure

### Metrics to track

```
Queue depth:
  Current: 7,500 / 10,000 (75% full)
  Alert if > 80% for 5 minutes

Request rejection rate:
  503 errors: 5% of requests
  Alert if > 10%

Consumer lag (Kafka):
  Lag: 50,000 messages behind
  Alert if lag > 100,000

Processing time:
  p95 latency: 500ms
  Alert if > 1000ms (consumers falling behind)
```

### Auto-scaling based on queue depth

```
CloudWatch alarm:
  Metric: SQS ApproximateNumberOfMessages
  Threshold: > 10,000
  Action: Add 2 more consumers

When queue depth increases:
  → Auto-scaling adds consumers
  → Queue drains faster
  → System stabilizes
```

---

## 10. Common Interview Questions + Answers

### Q: What is backpressure and why is it important?

> "Backpressure is a mechanism to prevent a fast producer from overwhelming a slow consumer by signaling the producer to slow down. Without backpressure, if a producer sends 10,000 messages per second but the consumer can only process 1,000, the queue grows unbounded until the system runs out of memory and crashes. Backpressure maintains system stability by using bounded queues, rejecting requests with 503 errors, or blocking the producer until the consumer catches up."

### Q: How would you implement backpressure in an HTTP API?

> "When the server is overloaded, return 503 Service Unavailable with a Retry-After header telling clients when to retry. Use bounded queues for request processing — if the queue is full, immediately return 503 instead of accepting more requests. Implement rate limiting with 429 Too Many Requests for per-client limits. Monitor queue depth and auto-scale workers when the queue grows. For critical requests, implement load shedding to drop low-priority requests first while keeping high-priority requests."

### Q: What's the difference between backpressure and rate limiting?

> "Rate limiting prevents a single client from sending too many requests, typically enforced per API key or IP address. Backpressure prevents the entire system from being overwhelmed regardless of the source. Rate limiting is proactive (prevent abuse), while backpressure is reactive (respond to overload). You need both — rate limiting to prevent individual clients from monopolizing resources, and backpressure to protect the system when aggregate load exceeds capacity."

### Q: How does TCP implement backpressure?

> "TCP uses flow control with a receive window. The receiver advertises how much buffer space it has available. If the receiver's buffer fills up, it sends an ACK with window size 0, telling the sender to stop. When the receiver processes data and frees buffer space, it sends an updated window size, allowing the sender to resume. This is automatic backpressure at the transport layer, but applications still need application-level backpressure to handle semantic concerns like request prioritization."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Mention bounded queues

Unbounded queues lead to out-of-memory errors. Always use bounded queues with backpressure.

### ✅ Trick 2: Discuss load shedding

Drop low-priority requests first when overloaded. This shows you understand graceful degradation.

### ✅ Trick 3: Know TCP flow control

TCP has built-in backpressure. Mentioning this shows you understand the full stack.

### ❌ Pitfall 1: Thinking backpressure is the same as rate limiting

They're different. Rate limiting is per-client, backpressure is system-wide.

### ❌ Pitfall 2: Forgetting about cascading failures

Without backpressure, overload cascades through the system. Mention circuit breakers.

### ❌ Pitfall 3: Blocking indefinitely

Blocking the producer can cause deadlocks. Use timeouts or non-blocking rejection.

---

## 12. Quick Reference

```
What is backpressure?
  Signal producer to slow down when consumer is overwhelmed
  Prevents resource exhaustion and system crashes

Why it matters:
  Without: Unbounded queue growth → Out of memory → Crash
  With: Bounded queues → Stable system

Strategies:
  1. Block producer (wait for space)
  2. Drop new items (reject with error)
  3. Drop old items (FIFO eviction)
  4. Reject with 503/429 (client retries)

HTTP APIs:
  503 Service Unavailable (overloaded)
  429 Too Many Requests (rate limited)
  Retry-After header (when to retry)

Message queues:
  Kafka: Buffer limit, block or throw exception
  RabbitMQ: Queue length limit, reject or drop
  SQS: Monitor queue depth, auto-scale consumers

Reactive Streams:
  Subscriber requests N items
  Publisher sends at most N items
  Subscriber controls flow rate

TCP flow control:
  Receiver advertises window size
  Window size 0 → Sender stops
  Automatic backpressure at transport layer

Bounded queues:
  ✅ Queue(maxsize=10000)
  ❌ Queue() (unbounded)

Load shedding:
  Drop low-priority requests when overloaded
  Keep critical requests
  Graceful degradation

Microservices:
  Service B returns 503 when overloaded
  Service A backs off
  Circuit breaker prevents cascading failures

Monitoring:
  Queue depth (alert if > 80%)
  Rejection rate (alert if > 10%)
  Consumer lag (Kafka)
  Processing time (p95 latency)

Auto-scaling:
  Monitor queue depth
  Add consumers when queue grows
  Remove consumers when queue shrinks

Backpressure vs Rate limiting:
  Backpressure: System-wide overload protection
  Rate limiting: Per-client abuse prevention
  Need both
```
