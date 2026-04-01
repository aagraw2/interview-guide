## 1. What are Failover and Fallback?

**Failover:** Automatically switch to a backup system when the primary system fails.

**Fallback:** Return degraded functionality or cached data when a service is unavailable, instead of failing completely.

```
Failover:
  Primary database fails → Automatically switch to replica
  → Full functionality maintained

Fallback:
  Recommendation service fails → Return popular items instead
  → Degraded functionality, but system still works
```

**Purpose:** High availability, graceful degradation, maintain user experience during failures.

---

## 2. Failover

### Active-Passive Failover

One primary system handles traffic, backup is on standby.

```
Normal operation:
  Primary: Active (serving traffic)
  Secondary: Passive (standby, replicating data)

Primary fails:
  1. Detect failure (health checks)
  2. Promote secondary to primary
  3. Route traffic to new primary
  4. Old primary becomes secondary (when recovered)

Failover time: 30 seconds to 5 minutes
```

**Pros:**
- Simple
- No split-brain (only one active)
- Lower cost (secondary can be smaller)

**Cons:**
- Wasted capacity (secondary idle)
- Failover time (downtime during switch)
- Data loss risk (replication lag)

### Active-Active Failover

Both systems handle traffic simultaneously.

```
Normal operation:
  Primary: Active (serving 50% traffic)
  Secondary: Active (serving 50% traffic)

Primary fails:
  1. Detect failure
  2. Route all traffic to secondary
  3. Secondary handles 100% traffic

Failover time: Seconds (just routing change)
```

**Pros:**
- No wasted capacity
- Fast failover (no promotion needed)
- No data loss (both have same data)

**Cons:**
- More expensive (both fully provisioned)
- Complex (data synchronization)
- Split-brain risk (both think they're primary)

---

## 3. Database Failover

### MySQL/PostgreSQL

```
Setup:
  Primary: Read/write
  Replica: Read-only (async replication)

Failover:
  1. Primary fails
  2. Promote replica to primary
  3. Update DNS or connection string
  4. Applications reconnect to new primary

Replication lag:
  Async: 1-5 seconds lag → Data loss possible
  Sync: No lag → Slower writes, lower availability

Tools: MySQL Group Replication, PostgreSQL Patroni
```

### MongoDB

```
Replica set:
  Primary: Read/write
  Secondary 1: Read-only
  Secondary 2: Read-only

Failover:
  1. Primary fails
  2. Secondaries elect new primary (majority vote)
  3. Applications automatically reconnect
  4. Failover time: 10-30 seconds

Automatic failover (no manual intervention)
```

### DynamoDB / Cosmos DB

```
Multi-region active-active:
  US East: Read/write
  EU West: Read/write
  Asia: Read/write

Region fails:
  → Traffic automatically routed to other regions
  → No failover needed (already active)
  → Eventual consistency (conflict resolution)
```

---

## 4. Application Failover

### Load balancer health checks

```
Load balancer:
  - Health check every 10 seconds
  - If 3 consecutive failures → Mark unhealthy
  - Stop routing traffic to unhealthy instance

Instance 1: Healthy → Receives traffic
Instance 2: Unhealthy → No traffic
Instance 3: Healthy → Receives traffic

Automatic failover at load balancer level
```

### DNS failover (Route 53)

```
Primary: us-east-1 (health check: passing)
Secondary: us-west-2 (health check: passing)

Primary fails:
  1. Health check detects failure
  2. Update DNS to point to secondary
  3. TTL expires (60 seconds)
  4. Clients resolve to secondary

Failover time: TTL + detection time (1-2 minutes)
```

### Kubernetes

```
Deployment with 3 replicas:
  Pod 1: Running
  Pod 2: Running
  Pod 3: Running

Pod 1 fails:
  1. Liveness probe fails
  2. Kubernetes kills pod
  3. Kubernetes starts new pod
  4. New pod passes readiness probe
  5. Added to service endpoints

Automatic self-healing
```

---

## 5. Fallback Strategies

### Return cached data

```
def get_recommendations(user_id):
    try:
        return recommendation_service.get(user_id)
    except ServiceUnavailable:
        # Fallback: Return cached recommendations
        cached = cache.get(f"recommendations:{user_id}")
        if cached:
            return cached
        # Fallback: Return popular items
        return get_popular_items()

Stale data is better than no data
```

### Return default values

```
def get_user_preferences(user_id):
    try:
        return preferences_service.get(user_id)
    except ServiceUnavailable:
        # Fallback: Return default preferences
        return {
            "theme": "light",
            "language": "en",
            "notifications": True
        }

Reasonable defaults maintain functionality
```

### Degrade functionality

```
def search_products(query):
    try:
        # Full search with ML ranking
        return search_service.search(query, use_ml=True)
    except ServiceUnavailable:
        # Fallback: Simple keyword search
        return database.query(
            "SELECT * FROM products WHERE name LIKE %s",
            f"%{query}%"
        )

Degraded functionality is better than no functionality
```

### Return partial results

```
def get_user_dashboard(user_id):
    results = {}
    
    try:
        results["profile"] = profile_service.get(user_id)
    except:
        results["profile"] = None  # Partial failure
    
    try:
        results["orders"] = order_service.get(user_id)
    except:
        results["orders"] = None  # Partial failure
    
    try:
        results["recommendations"] = recommendation_service.get(user_id)
    except:
        results["recommendations"] = []  # Partial failure
    
    return results

Show what's available, hide what's not
```

---

## 6. Circuit Breaker + Fallback

Combine circuit breaker with fallback for better resilience.

```
def get_recommendations(user_id):
    if circuit_breaker.is_open():
        # Circuit open: Don't even try, use fallback immediately
        return get_cached_recommendations(user_id)
    
    try:
        result = recommendation_service.get(user_id)
        circuit_breaker.record_success()
        return result
    except ServiceUnavailable:
        circuit_breaker.record_failure()
        # Fallback
        return get_cached_recommendations(user_id)

Circuit breaker prevents wasting time on known failures
Fallback provides degraded functionality
```

---

## 7. Split-Brain Problem

When both primary and secondary think they're primary.

### Problem

```
Network partition:
  Primary: Can't reach secondary → Thinks secondary is down
  Secondary: Can't reach primary → Thinks primary is down
  
Both promote themselves to primary:
  → Both accept writes
  → Data diverges
  → Conflict when partition heals

This is called "split-brain"
```

### Solution: Quorum

```
3-node cluster:
  Node 1, Node 2, Node 3

Network partition:
  Partition A: Node 1, Node 2 (majority)
  Partition B: Node 3 (minority)

Only partition with majority can elect primary:
  → Partition A elects Node 1 as primary
  → Partition B can't elect (no majority)
  → No split-brain

Requires odd number of nodes (3, 5, 7)
```

### Solution: Fencing

```
Primary fails:
  1. Secondary promotes itself
  2. Secondary "fences" old primary (prevents writes)
  3. Secondary becomes new primary

Fencing methods:
  - STONITH (Shoot The Other Node In The Head)
  - Revoke storage access
  - Disable network interface
  - Power off old primary

Ensures only one primary at a time
```

---

## 8. Monitoring Failover

### Metrics

```
Failover count:
  Last 24 hours: 2 failovers
  Alert if > 5 (frequent failovers indicate instability)

Failover duration:
  Last failover: 45 seconds
  Alert if > 2 minutes

Replication lag:
  Current: 2 seconds
  Alert if > 10 seconds (data loss risk)

Health check failures:
  Instance 1: 0 failures
  Instance 2: 3 failures (about to be marked unhealthy)
```

### Alerting

```
Alert: Primary database failed over to replica
  - Check why primary failed
  - Verify replica is handling load
  - Monitor replication lag
  - Plan to restore primary

Alert: Frequent failovers (5 in 1 hour)
  - Indicates instability
  - Check health check configuration
  - Check resource utilization
  - May need to increase thresholds
```

---

## 9. Testing Failover

### Chaos engineering

```
Randomly kill instances in production:
  - Netflix Chaos Monkey
  - Verify automatic failover works
  - Verify monitoring detects failures
  - Verify alerts fire correctly

Test in production (with safeguards):
  - Start with non-critical services
  - Limit blast radius
  - Have rollback plan
```

### Disaster recovery drills

```
Quarterly DR drill:
  1. Simulate primary region failure
  2. Failover to secondary region
  3. Verify all services work
  4. Measure failover time
  5. Document issues
  6. Improve runbooks

Practice makes perfect
```

---

## 10. Common Interview Questions + Answers

### Q: What's the difference between failover and fallback?

> "Failover is switching to a backup system when the primary fails, maintaining full functionality. For example, if the primary database fails, failover promotes the replica to primary. Fallback is returning degraded functionality when a service is unavailable. For example, if the recommendation service fails, fallback returns cached recommendations or popular items. Failover maintains full functionality but requires redundant infrastructure. Fallback accepts degraded functionality but is simpler and cheaper."

### Q: What's the difference between active-passive and active-active failover?

> "Active-passive has one primary handling traffic and a standby backup. When the primary fails, the backup is promoted, which takes 30 seconds to 5 minutes. Active-active has both systems handling traffic simultaneously. When one fails, the other handles all traffic immediately. Active-passive is simpler and cheaper but has failover time and wasted capacity. Active-active is faster and uses capacity efficiently but is more complex and expensive. Use active-passive for cost savings and active-active for zero downtime."

### Q: What is the split-brain problem and how do you prevent it?

> "Split-brain occurs when both primary and secondary think they're primary, usually due to network partition. Both accept writes, causing data divergence. Prevention: use quorum-based systems where only the partition with a majority can elect a primary. For example, in a 3-node cluster, the partition with 2 nodes can elect a primary while the partition with 1 node cannot. Also use fencing to forcibly disable the old primary when promoting a new one. This ensures only one primary at a time."

### Q: How would you implement fallback for a recommendation service?

> "I'd use a layered fallback strategy. First, try the recommendation service with a circuit breaker. If it fails or the circuit is open, return cached recommendations from Redis. If cache is empty, return popular items from the database. If database is slow, return a hardcoded list of bestsellers. Each layer provides progressively more degraded but still useful functionality. The key is to never return an error — always return something useful, even if it's not personalized."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Know active-passive vs active-active

This is a common interview question. Know the trade-offs and when to use each.

### ✅ Trick 2: Discuss split-brain

Split-brain is a critical problem in failover systems. Mentioning quorum and fencing shows depth.

### ✅ Trick 3: Combine failover and fallback

Use failover for infrastructure (databases) and fallback for application logic (services). This shows you understand both.

### ❌ Pitfall 1: Thinking failover is instant

Failover takes time (seconds to minutes). Don't assume zero downtime without active-active.

### ❌ Pitfall 2: Forgetting about data loss

Async replication has lag. Failover can lose recent writes. Mention this trade-off.

### ❌ Pitfall 3: Not testing failover

Untested failover will fail when you need it. Always mention testing and chaos engineering.

---

## 12. Quick Reference

```
Failover:
  Automatically switch to backup when primary fails
  Maintains full functionality

Fallback:
  Return degraded functionality when service unavailable
  Better than complete failure

Active-Passive:
  Primary active, secondary standby
  Failover time: 30s-5min
  Pros: Simple, cheaper
  Cons: Wasted capacity, downtime

Active-Active:
  Both systems handle traffic
  Failover time: Seconds
  Pros: Fast, no wasted capacity
  Cons: Expensive, complex

Database failover:
  MySQL/PostgreSQL: Promote replica (manual or automatic)
  MongoDB: Automatic election (10-30s)
  DynamoDB: Multi-region active-active (no failover needed)

Application failover:
  Load balancer: Health checks, automatic routing
  DNS: Route 53 health checks (1-2min failover)
  Kubernetes: Self-healing (liveness/readiness probes)

Fallback strategies:
  1. Return cached data (stale is better than none)
  2. Return default values (reasonable defaults)
  3. Degrade functionality (simple search vs ML search)
  4. Return partial results (show what's available)

Circuit breaker + Fallback:
  Circuit open → Use fallback immediately
  Circuit closed → Try service, fallback on failure

Split-brain:
  Problem: Both think they're primary
  Solution: Quorum (majority vote), Fencing (disable old primary)

Monitoring:
  Failover count (alert if > 5/day)
  Failover duration (alert if > 2min)
  Replication lag (alert if > 10s)
  Health check failures

Testing:
  Chaos engineering (kill instances randomly)
  DR drills (quarterly failover tests)
  Practice makes perfect

Best practices:
  - Use active-active for zero downtime
  - Use active-passive for cost savings
  - Implement layered fallbacks
  - Test failover regularly
  - Monitor replication lag
  - Prevent split-brain with quorum
```
