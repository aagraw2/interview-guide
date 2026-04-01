## 1. What is "Timeout Everywhere"?

"Timeout Everywhere" means every network call, external dependency, or blocking operation must have a timeout. Without timeouts, a slow or unresponsive service can cause threads to hang indefinitely, exhausting resources and bringing down the entire system.

```
Without timeout:
  request = http.get("http://slow-service.com/api")
  → Waits forever if service doesn't respond
  → Thread blocked indefinitely
  → Eventually all threads blocked
  → System unresponsive

With timeout:
  request = http.get("http://slow-service.com/api", timeout=5)
  → Waits maximum 5 seconds
  → Raises timeout exception
  → Thread released
  → System remains responsive
```

**Purpose:** Prevent resource exhaustion, fail fast, maintain system responsiveness.

---

## 2. Why Timeouts Matter

### Problem: Hanging threads

```
Web server with 100 threads:
  - Service B stops responding (network issue, deadlock, etc.)
  - 100 requests to Service B, no timeout
  → All 100 threads wait forever
  → No threads available for new requests
  → Entire system unresponsive

One unresponsive dependency brings down the entire system
```

### Solution: Timeouts

```
Web server with 100 threads:
  - Service B stops responding
  - 100 requests to Service B, timeout=5s
  → After 5 seconds, all requests timeout
  → Threads released
  → System returns errors but remains responsive

Fail fast instead of hanging
```

---

## 3. Types of Timeouts

### Connection timeout

Time to establish a connection.

```
connection_timeout = 2 seconds

If can't connect within 2 seconds:
  → Raise connection timeout error
  → Don't wait indefinitely

Typical values: 1-5 seconds
```

### Read timeout (socket timeout)

Time to receive data after connection established.

```
read_timeout = 5 seconds

Connection established
Waiting for response...
If no data received within 5 seconds:
  → Raise read timeout error

Typical values: 5-30 seconds
```

### Request timeout (end-to-end)

Total time for the entire request.

```
request_timeout = 10 seconds

Includes:
  - Connection time
  - Request sending time
  - Server processing time
  - Response receiving time

If total time > 10 seconds:
  → Raise request timeout error

Typical values: 10-60 seconds
```

---

## 4. Implementation

### Python requests

```python
import requests

# Connection timeout: 2s, Read timeout: 5s
response = requests.get(
    "http://api.example.com/data",
    timeout=(2, 5)
)

# Single timeout for both
response = requests.get(
    "http://api.example.com/data",
    timeout=5
)

# No timeout (bad!)
response = requests.get(
    "http://api.example.com/data"
)  # Can hang forever
```

### Java

```java
// HttpClient with timeouts
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(2))
    .build();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("http://api.example.com/data"))
    .timeout(Duration.ofSeconds(5))
    .build();

HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
```

### Database queries

```python
# PostgreSQL
cursor.execute("SELECT * FROM users", timeout=5)

# MongoDB
collection.find({}, max_time_ms=5000)

# Redis
redis.get("key", socket_timeout=5)
```

### Message queue consumers

```python
# Kafka
consumer.poll(timeout_ms=1000)

# RabbitMQ
method, properties, body = channel.basic_get(queue='my_queue', timeout=5)
```

---

## 5. Timeout Values

### Too short

```
Timeout: 100ms
Service latency: 200ms (normal)

Result:
  → All requests timeout
  → False positives
  → Poor user experience

Timeout should be > p99 latency
```

### Too long

```
Timeout: 60 seconds
Service is down

Result:
  → Threads blocked for 60 seconds
  → Slow failure detection
  → Resource exhaustion

Timeout should be < acceptable wait time
```

### Choosing timeout values

```
Measure service latency:
  p50: 100ms
  p95: 500ms
  p99: 1000ms

Set timeout:
  timeout = p99 × 2 = 2 seconds

Allows for some variance while failing fast on real issues

Adjust based on:
  - Service SLA
  - User experience requirements
  - Resource constraints
```

---

## 6. Cascading Timeouts

Timeouts should decrease as you go deeper in the call chain.

### Problem: Timeout too long

```
Client → Service A (timeout: 10s) → Service B (timeout: 10s)

Service B hangs:
  - Service A waits 10s for Service B
  - Client waits 10s for Service A
  → Total: 20s (client timeout exceeded)
```

### Solution: Decreasing timeouts

```
Client → Service A (timeout: 10s) → Service B (timeout: 5s)

Service B hangs:
  - Service A waits 5s for Service B
  - Service A returns error to client after 5s
  - Client receives error after 5s
  → Total: 5s (within client timeout)

Rule: Each level should have shorter timeout than the level above
```

### Example

```
Client timeout: 10 seconds
  ↓
Service A timeout: 8 seconds
  ↓
Service B timeout: 6 seconds
  ↓
Database timeout: 4 seconds

Each level has buffer for processing time
```

---

## 7. Timeout Propagation

Pass timeout context through the call chain.

### Without propagation

```
Client → Service A (timeout: 10s) → Service B (timeout: 10s)

Client timeout: 10s
Service A receives request at t=0
Service A calls Service B at t=5s (after 5s of processing)
Service B timeout: 10s

Problem:
  - Client timeout at t=10s
  - Service B still has 5s left
  → Service B continues processing after client gave up
  → Wasted resources
```

### With propagation

```
Client → Service A (timeout: 10s) → Service B (timeout: 5s)

Client timeout: 10s
Service A receives request at t=0 with deadline=t+10s
Service A calls Service B at t=5s with deadline=t+10s
Service B calculates remaining time: 10s - 5s = 5s
Service B timeout: 5s

Service B knows client deadline and adjusts timeout
```

### Implementation (gRPC)

```python
# Client sets deadline
response = stub.GetUser(
    request,
    timeout=10  # Deadline = now + 10s
)

# Server receives deadline in context
def GetUser(self, request, context):
    # Check remaining time
    remaining = context.time_remaining()
    
    # Call downstream with remaining time
    response = downstream_service.call(
        request,
        timeout=remaining - 1  # Leave 1s buffer
    )
```

---

## 8. Timeout + Retry

Combine timeouts with retries for better resilience.

### Timeout without retry

```
Request timeout: 5s
Service latency: 6s (temporarily slow)

Result:
  → Timeout after 5s
  → Return error
  → Request failed (might have succeeded with retry)
```

### Timeout with retry

```
Request timeout: 5s
Max attempts: 3
Total timeout: 15s

Attempt 1: Timeout after 5s
Attempt 2: Timeout after 5s
Attempt 3: Success in 3s

Result:
  → Request succeeded on 3rd attempt
  → Better user experience
```

### Implementation

```python
def call_with_timeout_and_retry(func, timeout=5, max_attempts=3):
    for attempt in range(1, max_attempts + 1):
        try:
            return func(timeout=timeout)
        except TimeoutError:
            if attempt == max_attempts:
                raise
            # Retry with same timeout
```

---

## 9. Monitoring Timeouts

### Metrics to track

```
Timeout rate:
  timeouts / total_requests
  Alert if > 1%

Timeout by service:
  service_a_timeouts: 0.5%
  service_b_timeouts: 5% ← Problem!
  service_c_timeouts: 0.2%

Latency percentiles:
  p50: 100ms
  p95: 500ms
  p99: 2000ms ← Close to timeout (5s)
  p99.9: 5000ms ← Hitting timeout

If p99 is close to timeout:
  → Increase timeout or optimize service
```

### Alerting

```
Alert: Service B timeout rate > 5%
  - Check Service B health
  - Check network issues
  - Check if Service B is overloaded
  - Consider increasing timeout (temporary)
  - Optimize Service B (permanent)
```

---

## 10. Common Mistakes

### 1. No timeout

```python
# Bad: Can hang forever
response = requests.get("http://api.example.com/data")

# Good: Always set timeout
response = requests.get("http://api.example.com/data", timeout=5)
```

### 2. Timeout too long

```python
# Bad: 5 minutes is too long
response = requests.get("http://api.example.com/data", timeout=300)

# Good: Reasonable timeout
response = requests.get("http://api.example.com/data", timeout=10)
```

### 3. Same timeout everywhere

```python
# Bad: Same timeout for all services
TIMEOUT = 10

service_a_response = requests.get("http://service-a/api", timeout=TIMEOUT)
service_b_response = requests.get("http://service-b/api", timeout=TIMEOUT)

# Good: Different timeouts based on service characteristics
service_a_response = requests.get("http://service-a/api", timeout=5)  # Fast service
service_b_response = requests.get("http://service-b/api", timeout=30)  # Slow service
```

### 4. Not handling timeout exceptions

```python
# Bad: Timeout exception crashes the application
response = requests.get("http://api.example.com/data", timeout=5)

# Good: Handle timeout gracefully
try:
    response = requests.get("http://api.example.com/data", timeout=5)
except requests.Timeout:
    return {"error": "Service temporarily unavailable"}, 503
```

---

## 11. Common Interview Questions + Answers

### Q: Why is it important to set timeouts on all network calls?

> "Without timeouts, a slow or unresponsive service can cause threads to hang indefinitely, eventually exhausting all available threads and making the entire system unresponsive. One bad dependency can bring down the whole system. Timeouts ensure that threads are released after a reasonable wait time, allowing the system to fail fast and remain responsive. It's better to return an error quickly than to hang indefinitely."

### Q: What's the difference between connection timeout and read timeout?

> "Connection timeout is the time to establish a TCP connection to the server — typically 1-5 seconds. Read timeout is the time to receive data after the connection is established — typically 5-30 seconds. You need both because a service might accept connections quickly but take a long time to respond. For example, connection_timeout=2s means give up if can't connect within 2 seconds, and read_timeout=10s means give up if no response within 10 seconds after connecting."

### Q: How do you choose appropriate timeout values?

> "Measure the service's latency distribution and set the timeout to about 2x the p99 latency. For example, if p99 is 1 second, set timeout to 2 seconds. This allows for some variance while failing fast on real issues. Also consider the user experience — users typically won't wait more than 10-30 seconds. Adjust based on the service's SLA and your resource constraints. Monitor timeout rates and adjust if you see too many false positives or if threads are being exhausted."

### Q: What are cascading timeouts and why do they matter?

> "Cascading timeouts means each level in the call chain should have a shorter timeout than the level above it. For example, if the client has a 10-second timeout, Service A should have an 8-second timeout, and Service B should have a 6-second timeout. This ensures that if Service B times out, Service A has time to handle the error and return a response to the client before the client's timeout expires. Without this, you can have situations where the client times out but downstream services continue processing, wasting resources."

---

## 12. Interview Tricks & Pitfalls

### ✅ Trick 1: Mention both connection and read timeouts

Knowing the difference shows you understand the details of network communication.

### ✅ Trick 2: Discuss cascading timeouts

Timeouts should decrease as you go deeper. This shows you understand distributed systems.

### ✅ Trick 3: Combine with retries

Timeouts + retries work well together. Mentioning this shows you understand resilience patterns.

### ❌ Pitfall 1: Forgetting to set timeouts

Always set timeouts. This is the most common mistake.

### ❌ Pitfall 2: Using the same timeout everywhere

Different services have different characteristics. Tailor timeouts to each service.

### ❌ Pitfall 3: Not monitoring timeout rates

Set timeouts but don't monitor them. You need metrics to know if timeouts are too short or too long.

---

## 13. Quick Reference

```
What is "Timeout Everywhere"?
  Every network call must have a timeout
  Prevents threads from hanging indefinitely
  Fail fast instead of hanging

Why timeouts matter:
  Without: One slow service exhausts all threads
  With: Threads released after timeout, system responsive

Types of timeouts:
  Connection: Time to establish connection (1-5s)
  Read: Time to receive data after connection (5-30s)
  Request: Total time for entire request (10-60s)

Implementation:
  Python: requests.get(url, timeout=(2, 5))
  Java: HttpClient with connectTimeout and timeout
  Database: cursor.execute(query, timeout=5)
  Message queue: consumer.poll(timeout_ms=1000)

Choosing timeout values:
  Measure service latency (p50, p95, p99)
  Set timeout = p99 × 2
  Adjust based on SLA and user experience

Cascading timeouts:
  Each level should have shorter timeout than level above
  Client: 10s → Service A: 8s → Service B: 6s → DB: 4s
  Ensures upstream has time to handle errors

Timeout propagation:
  Pass deadline through call chain
  Downstream calculates remaining time
  Prevents wasted work after client timeout

Timeout + Retry:
  Timeout: Fail fast on slow responses
  Retry: Try again on transient failures
  Combine for better resilience

Monitoring:
  Timeout rate (alert if > 1%)
  Timeout by service (identify problem services)
  Latency percentiles (p99 close to timeout?)

Common mistakes:
  ❌ No timeout (can hang forever)
  ❌ Timeout too long (slow failure detection)
  ❌ Same timeout everywhere (not tailored)
  ❌ Not handling timeout exceptions

Best practices:
  - Always set timeouts
  - Use different timeouts for different services
  - Implement cascading timeouts
  - Combine with retries and circuit breakers
  - Monitor timeout rates
  - Handle timeout exceptions gracefully
```
