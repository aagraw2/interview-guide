
### **Priority Levels**

- **P1** = most commonly asked / must-know for interviews
    
- **P2** = occasionally asked / nice to know
    
- **P3** = less common / advanced or niche
    

| S. No | Priority | System / Product              | Key Concepts Tested                                                                                          | Notes                                                                            |
| ----- | -------- | ----------------------------- | ------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------- |
| 1     | P1       | Ticket Booking System         | Preventing double booking (Redis NX + DB lock), seat hold TTL, CDC for search sync, flash sale handling      | [Notes](example-systems/Ticket%20Booking%20System%20Design.md)                   |
| 2     | P1       | Ecommerce                     | Inventory oversell prevention, cart in Redis, price snapshot, saga pattern, CDC, flash sale Redis DECR gate  | [Notes](example-systems/Ecommerce%20System%20Design.md)                          |
| 3     | P1       | Netflix                       | ABR streaming (HLS/DASH), CDN as primary delivery, video encoding pipeline, Kafka for async processing       | [Notes](example-systems/Netflix%20System%20Design.md)                            |
| 4     | P1       | Hotel Reservation             | Date range availability, double booking prevention, Redis lock + DB transaction, CDC, geo search             | [Notes](example-systems/Hotel%20Booking%20System%20Design.md)                    |
| 5     | P1       | Food Delivery                 | Driver assignment matching, real-time location tracking, Kafka event bus, saga pattern, geo search           | [Notes](example-systems/Food%20Delivery%20System%20Design.md)                    |
| 6     | P1       | Chat System                   | WebSocket vs polling, message ordering (TIMEUUID), offline delivery, fan-out for groups, Redis WS registry   | [Notes](example-systems/Chat%20Application%20System%20Design.md)                 |
| 7     | P1       | Job Scheduler                 | Watcher polling (SELECT FOR UPDATE SKIP LOCKED), duplicate execution prevention, stuck job detection, Kafka  | [Notes](example-systems/Job%20Scheduler%20System%20Design.md)                    |
| 8     | P1       | Google Docs                   | OT vs CRDT, Redis as canonical copy, op log as event source, snapshot + op replay, distributed lock          | [Notes](example-systems/Collaborative%20Editor%20System%20Design.md)             |
| 9     | P1       | Ride Matching (Uber/Ola)      | Zookeeper lock for driver assignment, Redis GEOADD/GEORADIUS, WebSocket location updates, surge pricing      | [Notes](example-systems/Ride%20Booking%20System%20Design.md)                     |
| 10    | P1       | Social Media                  | Fan-out problem (push vs pull vs hybrid), content moderation pipeline, follow graph, counter accuracy        | [Notes](example-systems/Social%20Media%20System%20Design.md)                     |
| 11    | P1       | Leaderboard                   | Redis ZSET operations, two-consumer pattern (DB + Redis), stream processing for aggregations, hot key        | [Notes](example-systems/Leaderboard%20System%20Design.md)                        |
| 12    | P1       | Notification System           | Outbox pattern, three-tier Kafka topics (critical/standard/promotional), OTP service, idempotency            | [Notes](example-systems/Notification%20System%20Design.md)                       |
| 13    | P1       | Logstash (Log Aggregation)    | Kafka as ingestion buffer, Flink stream processing, Cassandra as primary store, hot-warm-cold tiering        | [Notes](example-systems/Logstash%20%28Log%20Aggregation%29%20System%20Design.md) |
| 14    | P1       | Google Drive                  | 6-step chunked upload, content-addressed chunks, delta sync, file_version_chunks table, client Watcher       | [Notes](example-systems/Google%20Drive%20System%20Design.md)                     |
| 15    | P2       | Payment System                | PaymentIntent + checkout session, PCI Zone tokenization, connector pattern, outbox, double-entry ledger      | [Notes](example-systems/Payment%20System%20Design.md)                            |
| 16    | P2       | SendGrid/ SES (Email Service) | DMARC/DKIM/SPF, suppression list, transactional vs marketing separation, bounce/complaint handling           | [Notes](example-systems/Email%20Service%20System%20Design.md)                    |
| 17    | P2       | URL Shortener                 | Base62 encoding, unique ID generation (Redis counter vs Zookeeper/Snowflake), caching, DB sharding           | [Notes](example-systems/TinyURL%20System%20Design.md)                            |
| 18    | P2       | Stock Trading System          | Order book (price-time priority), Exchange Gateway, Validator + Kafka topics, fund reservation               | [Notes](example-systems/Stock%20Trading%20System%20Design.md)                    |
| 19    | P2       | Short Video Platform (TikTok) | Fan-out problem, For You Page (recommendation), ABR streaming, video encoding pipeline, counter accuracy     | [Notes](example-systems/TikTok%20System%20Design.md)                             |
| 20    | P2       | Digital Wallet                | Double-entry bookkeeping, double-spend prevention (SELECT FOR UPDATE), idempotency, fraud detection          | [Notes](example-systems/Digital%20Wallet%20System%20Design.md)                   |
| 21    | P2       | S3-like Storage               | Content-addressed chunks, erasure coding vs replication, 11 nines durability, metadata vs data separation    | [Notes](example-systems/S3-like%20Storage%20System%20Design.md)                  |
| 22    | P2       | Live Streaming                | RTMP ingest, HLS segment TTL, CDN pull rate, viewer count approximation, LL-HLS for low latency              | [Notes](example-systems/Live%20Streaming%20System%20Design.md)                   |
| 23    | P2       | YouTube                       | UGC vs curated (Netflix), fan-out problem, view counter accuracy, subscription fan-out, CDN delivery         | [Notes](example-systems/YouTube%20System%20Design.md)                            |
| 24    | P2       | Leetcode                      | Sandbox security (Docker isolation), async execution, verdict determination, execution queue scaling         | [Notes](example-systems/Leetcode%20System%20Design.md)                           |
| 25    | P2       | MyGate                        | Offline-first guard app, OTP/QR entry, visitor log partitioning, real-time notification latency              | [Notes](example-systems/MyGate%20System%20Design.md)                             |
| 26    | P2       | Coursera                      | Course moderation workflow, Permission Sync Service, max vs last position tracking, enterprise multi-tenancy | [Notes](example-systems/Coursera%20%28Online%20Learning%29%20System%20Design.md) |
| 27    | P2       | Ad Click Aggregation          | Lambda architecture, Kafka partitioning by adId, deduplication, fraud detection at scale, late events        | [Notes](example-systems/Ad%20Click%20Aggregation%20System%20Design.md)           |
| 28    | P2       | Whiteboard                    | CRDT vs OT, element-based canvas, infinite canvas, cursor presence, offline drawing sync                     | [Notes](example-systems/Whiteboard%20System%20Design.md)                         |
| 29    | P3       | Gmail (Email Client)          | SMTP protocol, SPF/DKIM/DMARC, MX records, internal vs external routing, spam filtering, inbound SMTP        | [Notes](example-systems/Gmail%20System%20Design.md)                              |
| 30    | P3       | Google Maps                   | Tile pre-rendering, Contraction Hierarchies for routing, map matching, real-time traffic aggregation         | [Notes](example-systems/Google%20Maps%20System%20Design.md)                      |
| 31    | P3       | Reconciliation System         | Exact vs fuzzy matching, T+1/T+2 settlement, idempotent runs, Lambda architecture, discrepancy resolution    | [Notes](example-systems/Reconciliation%20System%20Design.md)                     |
| 32    | P3       | Payout System                 | Bank API rate limits, idempotency (no double payout), retry with backoff, dead letter queue, KYC validation  | [Notes](example-systems/Payout%20System%20Design.md)                             |
| 33    | P3       | Escrow System                 | State machine transitions, segregated holding accounts, double release prevention, dispute arbitration       | [Notes](example-systems/Escrow%20System%20Design.md)                             |
| 34    | P3       | Recommendation System         | Two-stage pipeline (recall + ranking), ANN search, cold start problem, collaborative filtering               | [Notes](example-systems/Recommendation%20System%20Design.md)                     |
| 35    | P3       | Search Engine                 | Inverted index, TF-IDF, PageRank, crawl freshness, deduplication (SimHash), index sharding                   | [Notes](example-systems/Search%20Engine%20System%20Design.md)                    |
| 36    | P3       | Multi-tenant SaaS             | Silo vs pool vs bridge tenancy models, noisy neighbor, GDPR deletion, connection pooling, schema migrations  | [Notes](example-systems/Multi-tenant%20SaaS%20System%20Design.md)                |

---

## Summary
- Start with `P1` systems during preparation, then move to `P2` and `P3` to broaden coverage.
