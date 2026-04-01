## 1. What are Health Checks?

Health checks are automated tests that verify if a service is functioning correctly. They help detect failures and remove unhealthy instances from load balancing rotation.

```
Load balancer:
  Every 10 seconds: GET /health on each server
  
  Server 1: 200 OK → Healthy → Send traffic
  Server 2: 500 Error → Unhealthy → Stop sending traffic
  Server 3: Timeout → Unhealthy → Stop sending traffic
```

**Purpose:** Detect failures, enable automatic recovery, maintain high availability.

---

## 2. Types of Health Checks

### Liveness check

Is the service running?

```
Check: Can the process respond to requests?

Example:
  GET /health → 200 OK (alive)
  GET /health → Timeout (dead)

Kubernetes liveness probe:
  If fails → Restart container

Simple check:
  - HTTP endpoint returns 200
  - TCP connection succeeds
  - Process is running
```

### Readiness check

Is the service ready to handle traffic?

```
Check: Is the service fully initialized and ready?

Example:
  Service starting up:
    - Database connection: Not ready
    - Cache warming: Not ready
    → GET /ready → 503 Service Unavailable

  Service fully started:
    - Database connection: Ready
    - Cache warming: Complete
    → GET /ready → 200 OK

Kubernetes readiness probe:
  If fails → Remove from service endpoints (no traffic)
  If passes → Add to service endpoints (receive traffic)
```

### Startup check

Is the service finished starting up?

```
Check: Has the service completed initialization?

Example:
  Service starting (takes 2 minutes):
    - GET /startup → 503 (not ready)
    - Liveness check disabled during startup
  
  Service started:
    - GET /startup → 200 OK
    - Liveness check now enabled

Kubernetes startup probe:
  Protects slow-starting containers from being killed by liveness probe
```

---

## 3. Health Check Endpoints

### Simple health check

```
GET /health

Response:
  200 OK
  {
    "status": "UP"
  }

Implementation:
  def health():
      return {"status": "UP"}, 200

Just checks if the service can respond
```

### Detailed health check

```
GET /health

Response:
  200 OK
  {
    "status": "UP",
    "checks": {
      "database": "UP",
      "redis": "UP",
      "disk_space": "UP",
      "memory": "UP"
    },
    "timestamp": "2024-01-01T00:00:00Z"
  }

Or if unhealthy:
  503 Service Unavailable
  {
    "status": "DOWN",
    "checks": {
      "database": "DOWN",
      "redis": "UP",
      "disk_space": "UP",
      "memory": "UP"
    },
    "timestamp": "2024-01-01T00:00:00Z"
  }
```

### Implementation

```python
def health():
    checks = {
        "database": check_database(),
        "redis": check_redis(),
        "disk_space": check_disk_space(),
        "memory": check_memory()
    }
    
    all_healthy = all(status == "UP" for status in checks.values())
    status = "UP" if all_healthy else "DOWN"
    status_code = 200 if all_healthy else 503
    
    return {
        "status": status,
        "checks": checks,
        "timestamp": datetime.utcnow().isoformat()
    }, status_code

def check_database():
    try:
        db.execute("SELECT 1")
        return "UP"
    except:
        return "DOWN"

def check_redis():
    try:
        redis.ping()
        return "UP"
    except:
        return "DOWN"
```

---

## 4. Health Check Configuration

### Frequency

```
Too frequent:
  - Every 1 second
  - High load on service
  - False positives (temporary glitches)

Too infrequent:
  - Every 5 minutes
  - Slow failure detection
  - Unhealthy instances serve traffic longer

Typical: 10-30 seconds
```

### Timeout

```
Too short:
  - 1 second timeout
  - False positives (slow responses marked as failures)

Too long:
  - 30 second timeout
  - Slow failure detection

Typical: 2-5 seconds
```

### Failure threshold

```
Consecutive failures before marking unhealthy:
  - 1 failure: Too sensitive (false positives)
  - 5 failures: Too slow (50 seconds at 10s interval)

Typical: 2-3 consecutive failures
```

### Success threshold

```
Consecutive successes before marking healthy:
  - 1 success: Too quick (flapping)
  - 5 successes: Too slow

Typical: 2 consecutive successes
```

---

## 5. Heartbeats

Heartbeats are periodic signals sent by a service to indicate it's alive.

### Pull-based (health checks)

```
Monitor pulls status from service:
  Monitor → GET /health → Service
  
  Every 10 seconds, monitor checks service

Pros:
  ✅ Monitor controls frequency
  ✅ Service doesn't need to know about monitor
  ✅ Easy to add new monitors

Cons:
  ❌ Monitor must reach service (network issues)
  ❌ Load on service (many monitors)
```

### Push-based (heartbeats)

```
Service pushes status to monitor:
  Service → POST /heartbeat → Monitor
  
  Every 30 seconds, service sends heartbeat

Pros:
  ✅ Service controls frequency
  ✅ Less load on service (one push vs many pulls)
  ✅ Works with NAT/firewall (outbound only)

Cons:
  ❌ Service must know about monitor
  ❌ If service crashes, no heartbeat sent
```

### TTL-based heartbeats

```
Service registers with TTL:
  POST /register {service_id: "server1", ttl: 60}
  
Service sends heartbeat:
  PUT /heartbeat/server1
  → Resets TTL to 60 seconds

Monitor:
  If no heartbeat for 60 seconds → Mark as dead

Example: Consul, etcd, ZooKeeper
```

---

## 6. Kubernetes Health Checks

### Liveness probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-app:latest
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30  # Wait 30s before first check
      periodSeconds: 10         # Check every 10s
      timeoutSeconds: 5         # Timeout after 5s
      failureThreshold: 3       # Restart after 3 failures
```

If liveness probe fails → Kubernetes restarts container

### Readiness probe

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
  successThreshold: 2  # Need 2 successes to mark ready
```

If readiness probe fails → Remove from service endpoints (no traffic)

### Startup probe

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 30  # 30 * 10s = 5 minutes to start
```

Protects slow-starting containers from liveness probe

---

## 7. Load Balancer Health Checks

### AWS Application Load Balancer (ALB)

```
Health check configuration:
  Protocol: HTTP
  Path: /health
  Port: 8080
  Interval: 30 seconds
  Timeout: 5 seconds
  Healthy threshold: 2 (consecutive successes)
  Unhealthy threshold: 2 (consecutive failures)

Behavior:
  Healthy: Send traffic
  Unhealthy: Stop sending traffic (but keep instance running)
```

### Nginx

```nginx
upstream backend {
  server 10.0.1.1:8080 max_fails=3 fail_timeout=30s;
  server 10.0.1.2:8080 max_fails=3 fail_timeout=30s;
  server 10.0.1.3:8080 max_fails=3 fail_timeout=30s;
}

max_fails: Mark unhealthy after 3 failures
fail_timeout: Retry after 30 seconds

Passive health check (monitors actual requests)
```

### HAProxy

```
backend servers
  option httpchk GET /health
  server server1 10.0.1.1:8080 check inter 10s fall 3 rise 2
  server server2 10.0.1.2:8080 check inter 10s fall 3 rise 2

inter: Check every 10 seconds
fall: Mark unhealthy after 3 failures
rise: Mark healthy after 2 successes

Active health check (dedicated health check requests)
```

---

## 8. Distributed Systems Health Checks

### Gossip protocol (Cassandra, Serf)

```
Each node periodically exchanges state with random peers:
  Node 1 → Node 3: "I'm alive, Node 2 is alive"
  Node 3 → Node 5: "I'm alive, Node 1 is alive, Node 2 is alive"

If no gossip from Node 2 for 30 seconds:
  → Mark Node 2 as suspected
  → Continue gossiping suspicion
  → If confirmed by multiple nodes → Mark as dead

Pros:
  ✅ Decentralized (no single point of failure)
  ✅ Scales to thousands of nodes
  ✅ Detects network partitions

Cons:
  ❌ Eventual consistency (takes time to propagate)
  ❌ False positives (network delays)
```

### Consensus-based (ZooKeeper, etcd)

```
Ephemeral nodes:
  Client creates /services/server1 (ephemeral)
  Client maintains session with heartbeats
  If no heartbeat for 30 seconds → Session expires → Node deleted

Watchers:
  Other clients watch /services
  Get notified when nodes are added/removed

Pros:
  ✅ Strong consistency
  ✅ Automatic cleanup
  ✅ Reliable failure detection

Cons:
  ❌ Requires consensus cluster (complexity)
  ❌ Limited scalability (hundreds of nodes)
```

---

## 9. Health Check Best Practices

### 1. Check dependencies

```
Don't just return 200 OK:
  def health():
      return {"status": "UP"}, 200

Check critical dependencies:
  def health():
      db_ok = check_database()
      cache_ok = check_redis()
      
      if db_ok and cache_ok:
          return {"status": "UP"}, 200
      else:
          return {"status": "DOWN"}, 503
```

### 2. Separate liveness and readiness

```
Liveness (can the service recover?):
  - Process is running
  - Not deadlocked
  - Can respond to requests

Readiness (is the service ready for traffic?):
  - Database connected
  - Cache warmed
  - Dependencies available

Don't mix them:
  ❌ Liveness checks database → Database down → Restart container → Still down → Restart loop
  ✅ Readiness checks database → Database down → Remove from load balancer → Wait for recovery
```

### 3. Fast health checks

```
Health check should be fast (<1 second):
  ✅ Ping database (SELECT 1)
  ✅ Check Redis (PING)
  ❌ Run complex query
  ❌ Call external API

Slow health checks:
  - Increase load on service
  - False positives (timeout)
  - Slow failure detection
```

### 4. Don't cascade failures

```
Bad:
  Service A health check → Calls Service B
  Service B is down → Service A marked unhealthy
  → Both services removed from load balancer

Good:
  Service A health check → Only checks local state
  Service A handles Service B failures gracefully (circuit breaker)
  → Service A stays healthy, degrades functionality
```

---

## 10. Common Interview Questions + Answers

### Q: What's the difference between liveness and readiness probes?

> "Liveness checks if the service is alive and can recover — if it fails, Kubernetes restarts the container. Readiness checks if the service is ready to handle traffic — if it fails, Kubernetes removes it from service endpoints but doesn't restart it. For example, liveness checks if the process is running and not deadlocked, while readiness checks if dependencies like the database are available. You should never check external dependencies in liveness probes because it can cause restart loops."

### Q: How do you implement health checks for a service with database dependencies?

> "I'd implement separate liveness and readiness endpoints. Liveness (/health) just checks if the service can respond, returning 200 OK. Readiness (/ready) checks if the database is connected by running a simple query like SELECT 1, and returns 200 if successful or 503 if the database is down. This way, if the database is temporarily unavailable, the service is removed from load balancing but not restarted, allowing it to recover when the database comes back."

### Q: What's the difference between active and passive health checks?

> "Active health checks are dedicated requests sent by the load balancer to a health endpoint, like GET /health every 10 seconds. Passive health checks monitor actual traffic — if 3 consecutive requests to a server fail, mark it unhealthy. Active checks detect failures faster and work even with no traffic, but add load. Passive checks have no overhead but only detect failures when there's traffic. Most systems use active checks for critical services and passive checks for efficiency."

### Q: How do gossip protocols detect failures in distributed systems?

> "In gossip protocols like Cassandra's, each node periodically exchanges state with random peers, sharing information about which nodes are alive. If a node doesn't receive gossip from another node for a timeout period (e.g., 30 seconds), it marks that node as suspected. The suspicion is gossiped to other nodes. If multiple nodes confirm the suspicion, the node is marked as dead. This is decentralized and scales well, but has eventual consistency — it takes time for failure information to propagate across the cluster."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Separate liveness and readiness

This is a common interview topic. Know the difference and when to use each.

### ✅ Trick 2: Mention dependency checks

Health checks should verify critical dependencies (database, cache), not just return 200 OK.

### ✅ Trick 3: Discuss failure thresholds

Don't mark unhealthy on first failure. Use 2-3 consecutive failures to avoid false positives.

### ❌ Pitfall 1: Checking external dependencies in liveness

This causes restart loops. Only check local state in liveness probes.

### ❌ Pitfall 2: Slow health checks

Health checks should be fast (<1 second). Slow checks cause false positives and increase load.

### ❌ Pitfall 3: Cascading failures

Don't call other services in health checks. This can cascade failures across the system.

---

## 12. Quick Reference

```
What are health checks?
  Automated tests to verify service is functioning
  Detect failures, enable automatic recovery

Types:
  Liveness: Is the service alive? (restart if fails)
  Readiness: Is the service ready for traffic? (remove from LB if fails)
  Startup: Has the service finished starting? (protect slow starts)

Health check endpoint:
  GET /health → 200 OK (healthy) or 503 (unhealthy)
  Check dependencies: database, cache, disk, memory

Configuration:
  Frequency: 10-30 seconds
  Timeout: 2-5 seconds
  Failure threshold: 2-3 consecutive failures
  Success threshold: 2 consecutive successes

Heartbeats:
  Pull-based: Monitor checks service (health checks)
  Push-based: Service sends heartbeat to monitor
  TTL-based: Service registers with TTL, sends periodic heartbeat

Kubernetes:
  Liveness: Restart container if fails
  Readiness: Remove from service if fails
  Startup: Protect slow-starting containers

Load balancers:
  AWS ALB: Active health checks (dedicated requests)
  Nginx: Passive health checks (monitor actual traffic)
  HAProxy: Active health checks (dedicated requests)

Distributed systems:
  Gossip: Decentralized, eventual consistency (Cassandra, Serf)
  Consensus: Strong consistency, automatic cleanup (ZooKeeper, etcd)

Best practices:
  1. Check dependencies (database, cache)
  2. Separate liveness and readiness
  3. Fast health checks (<1 second)
  4. Don't cascade failures (don't call other services)
  5. Use failure thresholds (2-3 consecutive failures)

Liveness vs Readiness:
  Liveness: Can the service recover? (restart if fails)
  Readiness: Is the service ready? (remove from LB if fails)
  Never check external dependencies in liveness (restart loops)

Active vs Passive:
  Active: Dedicated health check requests (faster detection, more load)
  Passive: Monitor actual traffic (no overhead, slower detection)
```
