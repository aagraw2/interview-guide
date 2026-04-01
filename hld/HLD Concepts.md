
### **Priority Levels**

- **P1** = most commonly asked / must-know for interviews
    
- **P2** = occasionally asked / good to know
    
- **P3** = less common / advanced or niche

## Data & Storage

| Concept / Component          | Description                                                                                                                                                    | Priority | Notes | Video Tutorial |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ----- | -------------- |
| CAP Theorem                  | A distributed system can guarantee only 2 of: Consistency, Availability, Partition Tolerance. Know where real systems sit (Cassandra = AP, HBase = CP).        | P1       | [Notes](concepts/CAP%20Theorem.md) |                |
| SQL Database                 | ACID transactions, strong consistency, relational model. Know indexing, query planning, connection pooling. Use when joins or strict consistency are needed.   | P1       | [Notes](concepts/SQL%20Database.md) |                |
| NoSQL Database               | Schema-flexible, horizontally scalable. Know the 4 families: document (MongoDB), key-value (DynamoDB), column (Cassandra), graph (Neo4j).                      | P1       | [Notes](concepts/NoSQL%20Database.md) |                |
| Database Indexing            | B-tree for range queries, hash for equality, composite and covering indexes. Know when indexes hurt write performance.                                         | P1       | [Notes](concepts/Database%20Indexing.md) |                |
| Database Replication         | Leader-follower (sync vs async), multi-leader, leaderless (Dynamo-style). Understand replication lag, eventual consistency, read-your-writes guarantees.       | P1       | [Notes](concepts/Replication.md) |                |
| Sharding / Partitioning      | Horizontal data split across nodes. Range, hash, directory-based strategies. Know hot-spot avoidance, cross-shard joins, resharding.                           | P1       | [Notes](concepts/Sharding%20and%20Partitioning.md) |                |
| Caching                      | In-memory layer (Redis, Memcached). Write-through, write-back, write-around policies. Eviction: LRU, LFU, TTL. Cache stampede and thundering herd mitigations. | P1       | [Notes](concepts/Caching.md) |                |
| Key-Value Store              | Design internals: hash table, LSM tree + SSTables, bloom filters, compaction (leveled vs tiered). Foundation for many distributed storage systems.             | P1       | [Notes](concepts/Key-Value%20Store.md) |                |
| ACID vs BASE                 | ACID = strong consistency (SQL). BASE = basically available, eventually consistent (NoSQL). Know trade-offs and when each is acceptable.                       | P1       | [Notes](concepts/ACID%20vs%20BASE.md) |                |
| Database Selection           | Framework for choosing the right DB: read/write ratio, consistency needs, query patterns, scale, operational complexity. Core interview discussion.            | P1       | [Notes](concepts/Database%20Selection.md) |                |
| Read Replicas                | Route reads to replicas to scale read throughput. Understand replication lag, stale reads, use in CQRS patterns.                                               | P2       | [Notes](concepts/Read%20Replicas.md) |                |
| Write-Ahead Log (WAL)        | Append-only log written before applying changes to storage. Enables crash recovery, replication, and change data capture (CDC).                                | P2       | [Notes](concepts/Write-Ahead%20Log%20(WAL).md) |                |
| Bloom Filter                 | Probabilistic set membership check — no false negatives, possible false positives. Used in RocksDB, Cassandra to avoid unnecessary disk lookups.               | P2       | [Notes](concepts/Bloom%20Filter.md) |                |
| Column-Family Store          | Wide-column stores (Cassandra, HBase). Data stored by column families, optimized for time-series and write-heavy workloads.                                    | P2       | [Notes](concepts/Column-Family%20Store.md) |                |
| Time-Series Database         | Optimized for timestamped data (InfluxDB, TimescaleDB). Know compression, retention policies, downsampling.                                                    | P2       | [Notes](concepts/Time-Series%20Database.md) |                |
| Search Index (Elasticsearch) | Inverted index for full-text search. Know sharding search indexes, relevance scoring, eventual consistency with the write path.                                | P2       | [Notes](concepts/Search%20Index%20(Elasticsearch).md) |                |
| Object / Blob Storage        | Immutable object storage (S3-compatible). Metadata separation, chunk replication, erasure coding basics, presigned URLs.                                       | P2       | [Notes](concepts/Object%20Storage.md) |                |
| Event Sourcing               | Append-only event log as source of truth. Rebuild state by replaying events. Pairs naturally with CQRS.                                                        | P3       | [Notes](concepts/Event%20Sourcing.md) |                |
| CQRS                         | Separate read and write models. Write model handles commands; read model is a denormalized projection optimized for queries.                                   | P3       | [Notes](concepts/CQRS.md) |                |
| Graph Database               | Nodes and edges for relationship-heavy data (Neo4j). Know traversal queries, use cases: social graphs, fraud detection, recommendations.                       | P3       | [Notes](concepts/Graph%20Database.md) |                |
| Vector Database              | Stores high-dimensional embeddings (Pinecone, Weaviate). Approximate Nearest Neighbor (ANN) search for AI/ML retrieval use cases.                              | P3       | [Notes](concepts/Vector%20Database.md) |                |
| Data Warehouse               | OLAP workloads, columnar storage (Redshift, BigQuery, Snowflake). Optimized for analytics, aggregations, not transactional writes.                             | P3       | [Notes](concepts/Data%20Warehouse.md) |                |
| Change Data Capture (CDC)    | Stream DB changes (inserts/updates/deletes) to downstream systems via WAL tailing (Debezium). Enables event-driven pipelines without polling.                  | P3       | [Notes](concepts/Change%20Data%20Capture%20(CDC).md) |                |

---

## Network & Routing

| Concept / Component              | Description                                                                                                                                  | Priority | Notes | Video Tutorial |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ----- | -------------- |
| Load Balancer                    | Distribute traffic across servers. Know L4 (TCP) vs L7 (HTTP). Algorithms: round-robin, least-connections, IP hash. Sticky sessions.         | P1       | [Notes](concepts/Load%20Balancing.md) |                |
| CDN                              | Edge caching for static and dynamic content. Push vs pull model, cache invalidation, origin shield, PoP (points of presence).                | P1       | [Notes](concepts/CDN.md) |                |
| API Gateway                      | Single entry point for clients. Handles auth, routing, rate limiting, SSL termination, request/response transformation.                      | P1       | [Notes](concepts/API%20Gateway.md) |                |
| DNS                              | How DNS resolution works, TTL trade-offs, GeoDNS for routing by region, authoritative vs recursive resolvers.                                | P2       | [Notes](concepts/DNS.md) |                |
| Reverse Proxy                    | Sits in front of servers — handles SSL, caching, compression, request routing. Nginx/HAProxy as examples.                                    | P2       | [Notes](concepts/Reverse%20Proxy.md) |                |
| Service Discovery                | How services find each other. Client-side (Eureka) vs server-side (Consul, AWS ALB). Health checks, deregistration on failure.               | P2       | [Notes](concepts/Service%20Discovery.md) |                |
| gRPC                             | Binary RPC protocol over HTTP/2. Supports streaming, strong typing via protobuf, low latency. Preferred for internal service communication.  | P3       | [Notes](concepts/gRPC.md) |                |

---

## Scalability

| Concept / Component            | Description                                                                                                                                   | Priority | Notes | Video Tutorial |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ----- | -------------- |
| Consistent Hashing             | Distribute keys across nodes with minimal remapping when nodes join or leave. Know virtual nodes for load balance. Core to Dynamo, Cassandra. | P1       | [Notes](concepts/Consistent%20Hashing.md) |                |
| Microservices                  | Service decomposition, data ownership per service, sync (REST/gRPC) vs async (queue) communication, avoiding distributed monoliths.           | P1       | [Notes](concepts/Microservices.md) |                |
| Horizontal vs Vertical Scaling | Vertical = bigger machine. Horizontal = more machines. Know stateless design requirements for horizontal scaling.                             | P1       | [Notes](concepts/Horizontal%20vs%20Vertical%20Scaling.md) |                |
| Unique ID Generator            | Snowflake IDs (timestamp + machine ID + sequence), UUIDs, ULID. Know monotonicity, global uniqueness, clock skew issues.                      | P2       | [Notes](concepts/Unique%20ID%20Generator.md) |                |
| Fanout                         | Push vs pull feed generation. Fanout-on-write for low-follower users, fanout-on-read for celebrities. Hybrid approach.                        | P2       | [Notes](concepts/Fanout.md) |                |
| Auto-Scaling                   | Dynamic capacity adjustment based on metrics (CPU, queue depth). Horizontal pod autoscaler, warm pool for cold-start mitigation.              | P2       | [Notes](concepts/Auto-Scaling.md) |                |
| Geo-Distribution               | Multi-region active-active vs active-passive. Data residency, cross-region replication latency, conflict resolution strategies.               | P2       | [Notes](concepts/Geo-Distribution.md) |                |
| Stateless Services             | Design services with no local state so any instance can handle any request. Required for horizontal scaling and rolling deployments.          | P2       | [Notes](concepts/Stateless%20Services.md) |                |


---

## Reliability & Fault Tolerance

| Concept / Component            | Description                                                                                                                                       | Priority | Notes                                                     | Video Tutorial |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | --------------------------------------------------------- | -------------- |
| Rate Limiting                  | Protect services from overload and abuse. Token bucket, leaky bucket, fixed window, sliding window log, sliding window counter.                   | P1       | [Notes](concepts/Rate%20Limiting.md)                      |                |
| Idempotency                    | Safe retries via idempotency keys. Critical for payments, order placement, any at-least-once delivery system.                                     | P1       | [Notes](concepts/Idempotency.md)                          |                |
| Circuit Breaker                | Prevent cascading failures. Closed → Open → Half-open states. Pair with retry + exponential backoff + jitter.                                     | P1       | [Notes](concepts/Circuit%20Breaker.md)                    |                |
| Distributed Locking            | Exclusive access across nodes. Redlock (Redis), ZooKeeper, etcd. Know fencing tokens to handle lock expiry edge cases.                            | P2       | [Notes](concepts/Distributed%20Locking.md)                |                |
| Health Checks & Heartbeats     | Active (probe endpoint) vs passive (track error rates) health checks. Gossip protocols (Cassandra, Serf) for liveness detection.                  | P2       | [Notes](concepts/Health%20Checks%20and%20Heartbeats.md)   |                |
| Backpressure                   | Signal upstream to slow down when consumer is overwhelmed. Reactive streams, bounded queues, queue-depth-based flow control.                      | P2       | [Notes](concepts/Backpressure.md)                         |                |
| Retry with Exponential Backoff | Retry failed requests with increasing delay + random jitter to avoid thundering herd. Know max retry caps.                                        | P2       | [Notes](concepts/Retry%20with%20Exponential%20Backoff.md) |                |
| Bulkhead Pattern               | Isolate failures by partitioning resources (thread pools, connection pools) per downstream dependency. Like watertight ship compartments.         | P2       | [Notes](concepts/Bulkhead%20Pattern.md)                   |                |
| Timeout Everywhere             | Every network call needs a timeout. Know the difference between connection timeout and read timeout. Prevents resource exhaustion.                | P2       | [Notes](concepts/Timeout%20Everywhere.md)                 |                |
| Saga Pattern                   | Distributed transactions without 2PC. Choreography (event-driven) vs orchestration (central coordinator). Compensating transactions for rollback. | P2       | [Notes](concepts/Saga%20Pattern.md)                       |                |
| Failover & Fallback            | Automatic switch to backup on failure. Graceful degradation: return cached/stale data or a default response rather than an error.                 | P2       | [Notes](concepts/Failover%20and%20Fallback.md)            |                |
| Chaos Engineering              | Intentionally inject failures (Netflix Chaos Monkey) to validate resilience. Key for reliability SLAs.                                            | P3       | [Notes](concepts/Chaos%20Engineering.md) |                |
| Checkpointing                  | Periodic state snapshots in stream processing (Kafka offsets, Flink checkpoints) for fault recovery without full reprocessing.                    | P3       | [Notes](concepts/Checkpointing.md) |                |
| Quorum                         | In distributed systems, a read or write succeeds when acknowledged by a majority (quorum) of nodes. Used in Raft, Paxos, Dynamo.                  | P3       | [Notes](concepts/Quorum.md) |                |

---

## Async, Messaging & Real-Time

| Concept / Component             | Description                                                                                                                                      | Priority | Notes | Video Tutorial |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | -------- | ----- | -------------- |
| Message Queue                   | Async decoupling (Kafka, RabbitMQ, SQS). At-least-once vs exactly-once delivery, consumer groups, dead-letter queues (DLQ), ordering guarantees. | P1       | [Notes](concepts/Message%20Queue.md) |                |
| Pub/Sub                         | Fan-out topics to multiple subscribers (Kafka topics, Google Pub/Sub, SNS). Decouples producers from consumers.                                  | P1       | [Notes](concepts/Message%20Queue.md) |                |
| WebSockets                      | Full-duplex persistent connection for real-time push (chat, live scores). Know connection scalability and sticky routing requirement.            | P1       | [Notes](concepts/WebSockets.md) |                |
| File Upload / Download          | Presigned URLs, chunked and resumable uploads (TUS protocol), multipart upload to S3-compatible storage.                                         | P2       | [Notes](concepts/File%20Upload%20and%20Download.md) |                |
| Server-Sent Events (SSE)        | Server-to-client one-way push over HTTP. Simpler than WebSocket. Good for notifications, live feeds where client doesn't send data.              | P2       | [Notes](concepts/Server-Sent%20Events%20(SSE).md) |                |
| Long Polling                    | Client holds connection open until server has data or timeout. Fallback when WebSocket/SSE not available.                                        | P2       | [Notes](concepts/Long%20Polling.md) |                |
| Distributed Scheduler           | Cron-like job execution across distributed workers. At-most-once vs at-least-once job execution, leader election for deduplication.              | P2       | [Notes](concepts/Distributed%20Scheduler.md) |                |
| Stream Processing               | Process events as they arrive (Kafka Streams, Flink, Spark Streaming). Know windowing (tumbling, sliding, session), exactly-once semantics.      | P2       | [Notes](concepts/Stream%20Processing.md) |                |
| Event-Driven Architecture       | Services communicate via events rather than direct calls. Loose coupling, better scalability, but harder to trace and debug.                     | P2       | [Notes](concepts/Event-Driven%20Architecture.md) |                |
| Message Ordering & Partitioning | Kafka guarantees order within a partition. Use a consistent partition key (e.g. user_id) to maintain order for a given entity.                   | P3       | [Notes](concepts/Message%20Ordering%20and%20Partitioning.md) |                |
| Exactly-Once Semantics          | Achieved in Kafka via idempotent producers + transactional APIs. Costly — prefer at-least-once + idempotent consumers when possible.             | P3       | [Notes](concepts/Exactly-Once%20Semantics.md) |                |

---

## Security & Identity

| Concept / Component | Description | Priority | Notes | Video Tutorial |
| --- | --- | --- | --- | --- |
| Authentication / Authorization | JWT, OAuth 2.0, session tokens. RBAC and ABAC. Know token refresh flows, token revocation, and inter-service auth (mTLS, service accounts). | P1 | [Notes](concepts/Authentication%20and%20Authorization.md) |  |
| API Keys & Secrets Management | API key issuance, hashing keys at rest, rotation. Secrets stored in vaults (AWS Secrets Manager, HashiCorp Vault) — never in code. | P1 | [Notes](concepts/API%20Keys%20and%20Secrets%20Management.md) |  |
| HTTPS / TLS | Encrypt data in transit. Know TLS handshake basics, certificate pinning, mTLS for service-to-service auth. | P1 | [Notes](concepts/HTTPS%20and%20TLS.md) |  |
| OAuth 2.0 / OpenID Connect | OAuth 2.0 for delegation (authorization). OIDC for identity (authentication). Know authorization code flow, PKCE, client credentials flow. | P2 | [Notes](concepts/OAuth%202.0%20and%20OpenID%20Connect.md) |  |
| SSO (Single Sign-On) | One login across multiple services. SAML for enterprise, OIDC for modern apps. Know SP-initiated vs IdP-initiated flows. | P2 | [Notes](concepts/SSO%20(Single%20Sign-On).md) |  |
| Encryption at Rest | Encrypt stored data (AES-256). Key management: envelope encryption, KMS. Know who holds the keys and key rotation. | P2 | [Notes](concepts/Encryption%20at%20Rest.md) |  |
| DDoS Protection | Anycast absorption, rate limiting at edge, IP reputation lists, CAPTCHAs. Know the difference between volumetric, protocol, and application-layer attacks. | P2 | [Notes](concepts/DDoS%20Protection.md) |  |
| Input Validation & Sanitization | Prevent SQL injection, XSS, SSRF. Validate at the boundary — don't trust client input. Use parameterized queries and output encoding. | P2 | [Notes](concepts/Input%20Validation%20and%20Sanitization.md) |  |
| Zero Trust Architecture | Never trust, always verify. Every service-to-service call is authenticated and authorized regardless of network location. | P3 | [Notes](concepts/Zero%20Trust%20Architecture.md) |  |
| Audit Logging | Tamper-evident log of all sensitive actions (who did what, when). Required for compliance (SOC2, GDPR). Append-only, ideally immutable. | P3 | [Notes](concepts/Audit%20Logging.md) |  |

---

## Infrastructure & Deployment

| Concept / Component | Description | Priority | Notes | Video Tutorial |
| --- | --- | --- | --- | --- |
| Microservices Architecture | Decompose by business domain. Each service owns its data. Understand the cost: distributed tracing, service discovery, network latency. | P1 | [Notes](concepts/Microservices.md) |  |
| Containers & Kubernetes | Container-based deployment, pod scheduling, horizontal pod autoscaler, rolling updates, resource limits. Know liveness vs readiness probes. | P2 | [Notes](concepts/Containers%20and%20Kubernetes.md) |  |
| Service Mesh | Sidecar proxy (Istio, Linkerd) handles mTLS, retries, circuit breaking, traffic shaping at the infrastructure layer. | P2 | [Notes](concepts/Service%20Mesh.md) |  |
| CI/CD Pipeline | Automated build, test, deploy. Blue-green deployments, canary releases, feature flags for safe rollouts. | P2 | [Notes](concepts/CI-CD%20Pipeline.md) |  |
| Blue-Green / Canary Deployments | Blue-green: switch all traffic to new version. Canary: route a small % first, monitor, then gradually increase. Reduces risk. | P2 | [Notes](concepts/Blue-Green%20and%20Canary%20Deployments.md) |  |
| Observability (Metrics, Logs, Traces) | The three pillars. Metrics for alerting, logs for debugging, distributed traces for latency attribution. Know OpenTelemetry. | P2 | [Notes](concepts/Observability%20(Metrics,%20Logs,%20Traces).md) |  |
| Configuration Management | Separate config from code. Feature flags, environment-specific configs, dynamic config (LaunchDarkly, AWS AppConfig). | P3 | [Notes](concepts/Configuration%20Management.md) |  |
| Infrastructure as Code | Terraform, Pulumi, CloudFormation. Reproducible, version-controlled infrastructure. Know idempotency in IaC. | P3 | [Notes](concepts/Infrastructure%20as%20Code.md) |  |

---

## Algorithms & Estimation

| Concept / Component             | Description                                                                                                                               | Priority | Notes | Video Tutorial |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | -------- | ----- | -------------- |
| Back-of-the-Envelope Estimation | Estimate QPS, storage, bandwidth. Know powers of 2, latency numbers (L1 cache ~1ns, disk seek ~10ms, cross-region ~150ms).                | P1       | [Notes](concepts/Back-of-the-Envelope%20Estimation.md) |                |
| Consistent Hashing (algorithm)  | Modular hash ring, virtual nodes for even distribution, O(log N) lookup. Know why naive mod-N hashing fails on node changes.              | P1       | [Notes](concepts/Consistent%20Hashing.md) |                |
| LRU Cache                       | Doubly linked list + hash map for O(1) get and put. Know LFU variant (min-heap or frequency buckets).                                     | P1       | [Notes](concepts/LRU%20Cache.md) |                |
| Sliding Window / Token Bucket   | Sliding window counter for rate limiting. Token bucket for burst tolerance. Know the difference and when to use each.                     | P1       | [Notes](concepts/Sliding%20Window%20and%20Token%20Bucket.md) |                |
| Merkle Tree                     | Hash tree for efficient data verification and synchronization. Used in Git, blockchain, Cassandra anti-entropy repair.                    | P2       | [Notes](concepts/Merkle%20Tree.md) |                |
| Gossip Protocol                 | Each node periodically exchanges state with a random peer. Eventual convergence with no central coordinator. Used in Cassandra, DynamoDB. | P2       | [Notes](concepts/Gossip%20Protocol.md) |                |
| Geohash / S2                    | Encode lat/lng into a string (Geohash) or cell hierarchy (S2). Enable efficient proximity queries by comparing prefixes.                  | P2       | [Notes](concepts/Geohash%20and%20S2.md) |                |
