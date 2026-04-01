## 1. What is Geo-Distribution?

Geo-distribution (geographic distribution) means deploying your application across multiple geographic regions to serve users from the closest location, improving latency and providing disaster recovery.

```
Single region (US East):
  User in Tokyo → 150ms latency to US East

Multi-region (US East, Tokyo, London):
  User in Tokyo → 10ms latency to Tokyo region
  User in London → 15ms latency to London region
  User in New York → 20ms latency to US East region
```

**Benefits:** Lower latency, disaster recovery, data residency compliance, higher availability.

---

## 2. Latency by Distance

### Speed of light limit

```
Distance: New York to Tokyo = 10,900 km
Speed of light in fiber: ~200,000 km/s
Theoretical minimum: 10,900 / 200,000 = 54ms (one way)
Round trip: 108ms

Actual latency: 150-200ms (routing, processing, queuing)

No amount of optimization can beat physics
  → Need servers closer to users
```

### Typical latencies

```
Same datacenter: 1-5ms
Same region (different AZ): 5-10ms
Cross-region (same continent): 20-50ms
Cross-continent: 100-300ms

Examples:
  US East → US West: 60ms
  US East → Europe: 80ms
  US East → Asia: 150ms
  Europe → Asia: 200ms
```

---

## 3. Active-Active vs Active-Passive

### Active-Active

All regions serve traffic simultaneously.

```
US East: Serving US users
Europe: Serving European users
Asia: Serving Asian users

All regions are active and handle writes

Pros:
  ✅ Low latency for all users
  ✅ No failover needed (already active)
  ✅ Load distributed across regions

Cons:
  ❌ Complex (data replication, conflict resolution)
  ❌ Expensive (run full stack in all regions)
  ❌ Consistency challenges (CAP theorem)
```

### Active-Passive

One region serves traffic, others are standby.

```
US East: Active (serving all traffic)
Europe: Passive (standby, replicating data)
Asia: Passive (standby, replicating data)

On failure:
  US East fails → Failover to Europe

Pros:
  ✅ Simpler (single active region)
  ✅ Cheaper (passive regions can be smaller)
  ✅ Easier consistency (single write region)

Cons:
  ❌ Higher latency for distant users
  ❌ Failover time (minutes to hours)
  ❌ Wasted capacity (passive regions idle)
```

---

## 4. Routing Users to Regions

### GeoDNS

Route users to nearest region based on IP geolocation.

```
User in Tokyo:
  DNS query: api.example.com
  GeoDNS: Returns IP of Tokyo region (1.2.3.4)
  User connects to Tokyo region

User in London:
  DNS query: api.example.com
  GeoDNS: Returns IP of London region (5.6.7.8)
  User connects to London region

AWS Route 53 Geolocation Routing
Cloudflare Load Balancing
```

### Anycast

Multiple servers advertise the same IP address. Network routes to nearest server.

```
IP: 1.2.3.4 advertised from:
  - US East datacenter
  - Europe datacenter
  - Asia datacenter

User in Tokyo:
  Connects to 1.2.3.4
  Network routes to Asia datacenter (nearest)

Used by: Cloudflare, Google, AWS CloudFront

Pros:
  ✅ Automatic routing (no DNS needed)
  ✅ Fast failover (network reroutes)
  ✅ DDoS protection (traffic distributed)

Cons:
  ❌ Complex to set up (BGP routing)
  ❌ Limited to stateless protocols (TCP issues)
```

### Latency-based routing

Route to region with lowest latency (measured, not geographic).

```
User in Australia:
  Measure latency to all regions:
    US East: 200ms
    Europe: 250ms
    Asia: 50ms
  Route to Asia (lowest latency)

AWS Route 53 Latency Routing
```

---

## 5. Data Replication

### Synchronous replication

Write to all regions before acknowledging.

```
Write request:
  1. Write to US East
  2. Replicate to Europe (wait)
  3. Replicate to Asia (wait)
  4. Acknowledge to client

Pros:
  ✅ Strong consistency (all regions have same data)
  ✅ No data loss on region failure

Cons:
  ❌ High latency (wait for cross-region replication)
  ❌ Availability impact (if one region is slow/down)
```

### Asynchronous replication

Write to local region, replicate to others later.

```
Write request:
  1. Write to US East
  2. Acknowledge to client (fast)
  3. Replicate to Europe (async)
  4. Replicate to Asia (async)

Pros:
  ✅ Low latency (don't wait for replication)
  ✅ High availability (local region failure doesn't block writes)

Cons:
  ❌ Eventual consistency (regions temporarily out of sync)
  ❌ Data loss risk (if region fails before replication)
```

### Multi-region databases

```
DynamoDB Global Tables:
  - Active-active multi-region
  - Asynchronous replication (< 1 second)
  - Last-writer-wins conflict resolution

Aurora Global Database:
  - One primary region (writes)
  - Multiple secondary regions (reads)
  - Replication lag: < 1 second
  - Failover: < 1 minute

Cosmos DB:
  - Multi-region writes
  - Tunable consistency (strong, bounded staleness, eventual)
  - Automatic conflict resolution
```

---

## 6. Conflict Resolution

When multiple regions accept writes, conflicts can occur.

### Last-writer-wins (LWW)

```
US East: Update user.name = "Alice" at 10:00:00
Europe: Update user.name = "Bob" at 10:00:01

Conflict resolution:
  Europe's write is newer → user.name = "Bob"

Pros:
  ✅ Simple
  ✅ Deterministic

Cons:
  ❌ Data loss (Alice's write is lost)
  ❌ Depends on clock sync
```

### Version vectors

Track causality to detect conflicts.

```
US East: Update user.name = "Alice" (version: {US:1, EU:0})
Europe: Update user.name = "Bob" (version: {US:0, EU:1})

Conflict detected (concurrent writes)
  → Application must resolve (merge, prompt user, etc.)

Used by: Riak, Cassandra (with LWW timestamps)
```

### CRDTs (Conflict-free Replicated Data Types)

Data structures that automatically merge conflicts.

```
Counter CRDT:
  US East: Increment counter (local: +1)
  Europe: Increment counter (local: +1)
  Merge: Sum all increments → counter = 2

Set CRDT:
  US East: Add "Alice"
  Europe: Add "Bob"
  Merge: Union → {"Alice", "Bob"}

Used by: Redis Enterprise, Riak
```

---

## 7. Data Residency and Compliance

### GDPR (Europe)

```
Requirement:
  EU user data must be stored in EU

Implementation:
  - Route EU users to EU region
  - Store data only in EU datacenters
  - Don't replicate to non-EU regions

Challenges:
  - Disaster recovery (can only failover within EU)
  - Analytics (can't aggregate with non-EU data)
```

### Data sovereignty

```
China: Data must be stored in China
Russia: Data must be stored in Russia
India: Payment data must be stored in India

Implementation:
  - Separate database per country
  - No cross-border replication
  - Local compliance (encryption, audit logs)
```

---

## 8. Disaster Recovery

### RTO and RPO

```
RTO (Recovery Time Objective):
  How long can you be down?
  Example: 1 hour

RPO (Recovery Point Objective):
  How much data can you lose?
  Example: 5 minutes

Active-passive:
  RTO: 10-60 minutes (failover time)
  RPO: 1-5 minutes (replication lag)

Active-active:
  RTO: 0 (already active)
  RPO: 0 (synchronous replication) or seconds (async)
```

### Failover strategies

```
Manual failover:
  - Operator triggers failover
  - Update DNS to point to backup region
  - RTO: 30-60 minutes

Automatic failover:
  - Health checks detect failure
  - Automatically update DNS or routing
  - RTO: 5-15 minutes

Active-active:
  - No failover needed (already active)
  - RTO: 0
```

---

## 9. Cost Considerations

### Data transfer costs

```
Within region: Free or very cheap
Cross-region (same continent): $0.01-0.02/GB
Cross-continent: $0.02-0.12/GB

Example:
  1 TB/day cross-region replication
  $0.02/GB × 1000 GB × 30 days = $600/month

Optimization:
  - Compress data before replication
  - Replicate only critical data
  - Use regional caching (CDN)
```

### Compute costs

```
Active-active:
  Run full stack in all regions
  3 regions × $10,000/month = $30,000/month

Active-passive:
  Full stack in active region: $10,000/month
  Minimal stack in passive regions: $2,000/month × 2 = $4,000/month
  Total: $14,000/month

Savings: $16,000/month (53%)
```

---

## 10. CDN for Static Content

Use CDN for static assets (images, CSS, JS) instead of replicating entire application.

```
Application servers: Single region (US East)
CDN: Global edge locations (200+ locations)

User in Tokyo:
  - Dynamic API calls: 150ms to US East
  - Static assets: 10ms from Tokyo edge location

Benefits:
  ✅ Much cheaper than multi-region deployment
  ✅ Good enough for many use cases
  ✅ Simple (no data replication complexity)

Limitations:
  ❌ Dynamic content still has high latency
```

---

## 11. Common Interview Questions + Answers

### Q: What's the difference between active-active and active-passive multi-region deployment?

> "Active-active means all regions serve traffic simultaneously, providing low latency for all users and no failover needed. However, it's complex due to data replication and conflict resolution, and expensive since you run the full stack in all regions. Active-passive means one region serves traffic while others are standby, which is simpler and cheaper but has higher latency for distant users and requires failover time. Choose active-active for global applications with strict latency requirements, and active-passive for disaster recovery with cost constraints."

### Q: How do you handle data conflicts in a multi-region active-active system?

> "Conflicts occur when multiple regions accept writes to the same data. Solutions include last-writer-wins (simple but can lose data), version vectors (detect conflicts, application resolves), and CRDTs (conflict-free data types that automatically merge). For example, DynamoDB Global Tables uses last-writer-wins with timestamps. The choice depends on your consistency requirements — if you can tolerate eventual consistency and occasional data loss, LWW is simple. If you need stronger guarantees, use version vectors or CRDTs."

### Q: How would you design a globally distributed application for low latency?

> "I'd use active-active multi-region deployment with GeoDNS to route users to the nearest region. For data, I'd use a multi-region database like DynamoDB Global Tables with asynchronous replication for low write latency. For static assets, I'd use a CDN like CloudFront. For conflict resolution, I'd use last-writer-wins for simplicity. I'd also implement regional caching with Redis to reduce database load. The key trade-off is accepting eventual consistency for low latency — if strong consistency is required, I'd use synchronous replication but accept higher latency."

### Q: What are the main cost considerations for multi-region deployment?

> "The main costs are compute (running infrastructure in multiple regions), data transfer (cross-region replication), and storage (duplicating data). Active-active is expensive because you run the full stack in all regions. Active-passive is cheaper since passive regions can be smaller. Data transfer costs can be significant — $0.02-0.12/GB for cross-region traffic. To optimize, use active-passive for disaster recovery, compress data before replication, use CDN for static content, and replicate only critical data."

---

## 12. Interview Tricks & Pitfalls

### ✅ Trick 1: Know the latency numbers

Speed of light limits are fundamental. Mentioning that cross-continent latency is 100-300ms shows you understand the physics.

### ✅ Trick 2: Discuss active-active vs active-passive

This is a common interview question. Know the trade-offs and when to use each.

### ✅ Trick 3: Mention conflict resolution

Multi-region writes require conflict resolution. Mentioning LWW, version vectors, or CRDTs shows depth.

### ❌ Pitfall 1: Thinking multi-region is always better

Multi-region is complex and expensive. For many applications, single-region with CDN is sufficient.

### ❌ Pitfall 2: Forgetting about data residency

GDPR and data sovereignty laws require data to stay in specific regions. Don't forget compliance.

### ❌ Pitfall 3: Ignoring costs

Multi-region deployment can be very expensive. Always mention cost considerations.

---

## 13. Quick Reference

```
What is geo-distribution?
  Deploy application across multiple geographic regions
  Benefits: Lower latency, disaster recovery, compliance

Latency by distance:
  Same region: 1-10ms
  Cross-region (same continent): 20-50ms
  Cross-continent: 100-300ms
  Speed of light limit: Can't beat physics

Active-active:
  All regions serve traffic simultaneously
  Pros: Low latency, no failover, load distributed
  Cons: Complex, expensive, consistency challenges

Active-passive:
  One region active, others standby
  Pros: Simpler, cheaper, easier consistency
  Cons: Higher latency, failover time, wasted capacity

Routing:
  GeoDNS: Route by IP geolocation (Route 53)
  Anycast: Same IP, network routes to nearest (Cloudflare)
  Latency-based: Route to lowest latency region

Data replication:
  Synchronous: Wait for all regions (strong consistency, high latency)
  Asynchronous: Replicate later (eventual consistency, low latency)

Multi-region databases:
  DynamoDB Global Tables: Active-active, async replication
  Aurora Global: One primary, multiple secondaries
  Cosmos DB: Multi-region writes, tunable consistency

Conflict resolution:
  Last-writer-wins: Simple, data loss risk
  Version vectors: Detect conflicts, app resolves
  CRDTs: Automatic merge (counters, sets)

Data residency:
  GDPR: EU data must stay in EU
  China, Russia: Data must stay in country
  Implementation: Regional databases, no cross-border replication

Disaster recovery:
  RTO: Recovery time (how long down?)
  RPO: Recovery point (how much data loss?)
  Active-passive: RTO 10-60 min, RPO 1-5 min
  Active-active: RTO 0, RPO 0 or seconds

Cost:
  Compute: Active-active expensive (full stack in all regions)
  Data transfer: $0.02-0.12/GB cross-region
  Optimization: Active-passive, compress, CDN, replicate only critical data

CDN alternative:
  Application: Single region
  Static assets: Global CDN
  Good enough for many use cases, much cheaper
```
