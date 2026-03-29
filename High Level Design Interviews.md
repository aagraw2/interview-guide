
## Components
[[HLD Components]]

## Example Systems
[[HLD Systems]]

---

# Interview Template

## 1. Clarify Requirements (2–3 mins)

### Functional Requirements

- What exactly should the system do?
    
- Core user actions / workflows.
    
- Expected throughput / request types.
    

### Non-Functional Requirements

- Scalability → number of users, QPS, data size.
    
- Performance → latency expectations.
    
- Reliability / Availability / Consistency needs.
    
- Security / Compliance requirements.
    

### Constraints / Assumptions

- Single vs distributed system.
    
- Storage / network limits.
    
- Data retention / regulatory requirements.
    

---

## 2. Identify Core Components (3–5 mins)

- Break system into high-level building blocks.
    
- Examples:
    
    - Frontend / client
        
    - API Gateway
        
    - Application / Service Layer
        
    - Database / Storage Layer
        
    - Caching Layer
        
    - Message Queue / Event Bus
        
    - Analytics / Logging / Monitoring
        
- Mention interactions between components.
    

---

## 3. Define APIs & Interfaces (2–3 mins)

- Expose main system interactions.
    
- Examples:
    

```text
POST /createOrder
GET /order/{id}
POST /payment
```

- Define request / response structure.
    
- Mention synchronous vs asynchronous calls.
    

---

## 4. Data Modeling & Storage (3–5 mins)

- Identify entities and relationships (high-level ER diagram).
    
- Decide storage type:
    
    - SQL vs NoSQL
        
    - Key-Value, Document, Graph, Time-Series
        
- Consider:
    
    - Sharding / partitioning
        
    - Replication / high availability
        
    - Indexing / caching
        

---

## 5. High-Level Component Interactions (5–10 mins)

- Draw a block diagram:
    
    - Frontend → API Gateway → Service → Database
        
    - Include caches, queues, background workers
        
- Show major flows:
    
    - Read / write path
        
    - Event handling / async processing
        
- Discuss bottlenecks and scalability points.
    

---

## 6. Apply System Design Patterns

- **Scaling / Distribution**
    
    - Load Balancer, Consistent Hashing, CDNs
        
- **Data Handling**
    
    - Caching, Replication, Partitioning, Queues
        
- **Reliability**
    
    - Circuit Breaker, Retry, Dead Letter Queues
        
- **Security**
    
    - Secrets Manager, Authentication, Authorization
        
- Mention where each pattern fits and why.
    

---

## 7. Address Edge Cases & Failures

- Single points of failure.
    
- Network / DB partitions.
    
- Traffic spikes → auto-scaling.
    
- Data corruption / retry mechanisms.
    

---

## 8. Trade-offs & Decisions

- SQL vs NoSQL → consistency vs flexibility.
    
- Synchronous vs asynchronous → latency vs throughput.
    
- Replication factor → durability vs cost.
    
- Caching → memory vs staleness.
    
- Be ready to justify choices.
    

---

## 9. Monitoring & Observability

- Logging / metrics / alerts.
    
- Health checks / dashboards.
    
- SLA monitoring.
    

---

## 10. Scaling Strategy

- Horizontal vs vertical scaling.
    
- Partitioning / sharding strategy.
    
- CDN / edge servers.
    
- Queueing / batching.
    

---

## 11. Complexity Analysis

- Expected QPS / throughput.
    
- Latency expectations.
    
- Storage growth & retention considerations.
    

---

## 12. Golden Lines to Say

- “We’ll separate concerns into services and data layers.”
    
- “We can use caching to reduce DB load.”
    
- “Asynchronous processing via queues will improve throughput.”
    
- “Sharding helps us scale horizontally without impacting latency.”
    
- “Circuit breaker prevents cascading failures.”
    
- “Let’s discuss trade-offs between consistency, availability, and partition tolerance.”
    
