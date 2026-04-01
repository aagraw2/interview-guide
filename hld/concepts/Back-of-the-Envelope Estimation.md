## 1. What Is Back-of-the-Envelope Estimation?

Quick, approximate calculations to understand system scale and feasibility.

**Purpose:**
- Estimate QPS (queries per second)
- Calculate storage requirements
- Determine bandwidth needs
- Validate system design decisions

**Not about precision** — about getting the right order of magnitude.

---

## 2. Key Numbers to Memorize

### Powers of 2

```
2^10 = 1,024      ≈ 1 thousand (1 KB)
2^20 = 1,048,576  ≈ 1 million  (1 MB)
2^30 = 1,073,741,824 ≈ 1 billion (1 GB)
2^40 ≈ 1 trillion (1 TB)
```

**Use approximations:** 1 KB = 1,000 bytes, 1 MB = 1,000 KB, 1 GB = 1,000 MB

---

### Latency Numbers

```
L1 cache reference:              0.5 ns
Branch mispredict:               5 ns
L2 cache reference:              7 ns
Mutex lock/unlock:               100 ns
Main memory reference:           100 ns
Compress 1KB with Snappy:        10,000 ns = 10 μs
Send 2KB over 1 Gbps network:    20,000 ns = 20 μs
Read 1 MB sequentially from memory: 250,000 ns = 250 μs
Round trip within same datacenter: 500,000 ns = 0.5 ms
Disk seek:                       10,000,000 ns = 10 ms
Read 1 MB sequentially from disk: 30,000,000 ns = 30 ms
Send packet CA → Netherlands → CA: 150,000,000 ns = 150 ms
```

**Key takeaways:**
- Memory is fast (100 ns)
- Disk is slow (10 ms seek)
- Network within datacenter is fast (0.5 ms)
- Cross-region network is slow (150 ms)

---

### Time Conversions

```
1 second = 1,000 milliseconds (ms)
1 ms = 1,000 microseconds (μs)
1 μs = 1,000 nanoseconds (ns)

1 day = 86,400 seconds ≈ 100,000 seconds (10^5)
1 month = 30 days ≈ 2.5 million seconds (2.5 × 10^6)
1 year = 365 days ≈ 31.5 million seconds (3 × 10^7)
```

---

### Availability Numbers

```
99% availability = 3.65 days downtime/year
99.9% (three nines) = 8.76 hours downtime/year
99.99% (four nines) = 52.6 minutes downtime/year
99.999% (five nines) = 5.26 minutes downtime/year
```

---

## 3. Estimation Framework

### Step 1: Clarify requirements

```
Ask:
  - How many users?
  - How many requests per user per day?
  - Read vs write ratio?
  - Data retention period?
  - Peak vs average traffic?
```

---

### Step 2: Calculate QPS

```
Formula:
  Daily Active Users (DAU) × Actions per user per day / 86,400 seconds

Example: Twitter
  - 200 million DAU
  - Each user views 100 tweets per day
  - Read QPS = 200M × 100 / 86,400 ≈ 230,000 QPS
  
  - Each user posts 2 tweets per day
  - Write QPS = 200M × 2 / 86,400 ≈ 4,600 QPS

Peak QPS = Average QPS × 2 (or 3)
  - Peak read QPS ≈ 460,000 QPS
  - Peak write QPS ≈ 9,200 QPS
```

---

### Step 3: Calculate storage

```
Formula:
  Records per day × Record size × Retention period

Example: Twitter
  - 400 million tweets per day (200M users × 2 tweets)
  - Average tweet size: 300 bytes (text + metadata)
  - Retention: 5 years
  
  Storage = 400M × 300 bytes × 365 days × 5 years
          = 400M × 300 × 1,825
          = 219 TB
          ≈ 220 TB

With replication (3x):
  Total storage = 220 TB × 3 = 660 TB
```

---

### Step 4: Calculate bandwidth

```
Formula:
  QPS × Average request/response size

Example: Twitter
  Read bandwidth:
    - 230,000 QPS
    - Average response: 1 KB (tweet data)
    - Bandwidth = 230,000 × 1 KB = 230 MB/s
  
  Write bandwidth:
    - 4,600 QPS
    - Average request: 300 bytes
    - Bandwidth = 4,600 × 300 bytes = 1.4 MB/s
```

---

## 4. Common Estimation Patterns

### URL Shortener (like bit.ly)

```
Requirements:
  - 100 million URLs shortened per month
  - Read:Write ratio = 100:1
  - URL stored for 10 years

Write QPS:
  100M / (30 days × 86,400 sec) = 100M / 2.6M ≈ 40 QPS

Read QPS:
  40 QPS × 100 = 4,000 QPS

Storage:
  - 100M URLs/month × 12 months × 10 years = 12 billion URLs
  - Each URL: 500 bytes (original URL + short URL + metadata)
  - Total: 12B × 500 bytes = 6 TB

Bandwidth:
  - Write: 40 QPS × 500 bytes = 20 KB/s
  - Read: 4,000 QPS × 500 bytes = 2 MB/s
```

---

### Video Streaming (like YouTube)

```
Requirements:
  - 1 billion users
  - Each user watches 5 videos per day
  - Average video: 5 minutes, 50 MB

Read QPS:
  1B users × 5 videos / 86,400 sec ≈ 58,000 QPS

Storage (new videos per day):
  - 1 million new videos per day
  - Average size: 50 MB
  - Daily storage: 1M × 50 MB = 50 TB/day
  - Yearly: 50 TB × 365 = 18 PB/year

Bandwidth:
  - 58,000 videos playing simultaneously
  - Each video: 50 MB / 300 sec = 170 KB/s
  - Total: 58,000 × 170 KB/s = 9.8 GB/s
```

---

### Social Media Feed (like Facebook)

```
Requirements:
  - 2 billion users
  - 500 million DAU
  - Each user views feed 10 times per day
  - Each feed shows 20 posts

Read QPS:
  500M × 10 / 86,400 ≈ 58,000 QPS

Posts fetched per second:
  58,000 QPS × 20 posts = 1.16 million posts/sec

Storage (posts):
  - 500M users × 2 posts per day = 1 billion posts/day
  - Each post: 1 KB (text + metadata)
  - Daily: 1B × 1 KB = 1 TB/day
  - Yearly: 365 TB

Images:
  - 50% of posts have images
  - Average image: 200 KB
  - Daily: 500M posts × 200 KB = 100 TB/day
  - Yearly: 36.5 PB
```

---

## 5. Estimation Shortcuts

### Rule of thumb: 1 million requests per day

```
1 million requests per day ≈ 12 QPS

Derivation:
  1,000,000 / 86,400 ≈ 11.6 ≈ 12 QPS

Use this for quick mental math:
  10 million requests/day ≈ 120 QPS
  100 million requests/day ≈ 1,200 QPS
```

---

### Rule of thumb: 1 billion users

```
If 1 billion users, assume:
  - 10% are DAU = 100 million DAU
  - Each DAU makes 10 requests/day
  - Total: 1 billion requests/day ≈ 12,000 QPS
```

---

### Rule of thumb: Peak traffic

```
Peak QPS = Average QPS × 2 (or 3)

Some systems have higher peaks:
  - E-commerce (Black Friday): 10x
  - News sites (breaking news): 100x
  - Gaming (new release): 50x
```

---

## 6. Memory Estimation

### Cache sizing

```
Question: How much memory for caching 20% of daily requests?

Example: Twitter
  - 230,000 read QPS
  - 86,400 seconds/day
  - Total reads: 230,000 × 86,400 ≈ 20 billion reads/day
  - Cache 20%: 4 billion reads
  - Each cached item: 1 KB
  - Memory needed: 4B × 1 KB = 4 TB

With 80% cache hit rate:
  - Reduces DB load from 230,000 QPS to 46,000 QPS
```

---

### Server memory

```
Typical server: 64 GB RAM

How many servers for 4 TB cache?
  4 TB / 64 GB = 4,000 GB / 64 GB ≈ 63 servers

With replication (3x):
  63 × 3 = 189 servers
```

---

## 7. Database Estimation

### Number of database servers

```
Single PostgreSQL server:
  - Can handle ~10,000 QPS (reads)
  - Can handle ~5,000 QPS (writes)

Example: Twitter (230,000 read QPS, 4,600 write QPS)
  
  Read replicas needed:
    230,000 / 10,000 = 23 read replicas
  
  Write capacity:
    4,600 / 5,000 = 1 primary (sufficient)
  
  Total: 1 primary + 23 read replicas = 24 DB servers
```

---

### Sharding calculation

```
When to shard?
  - Single DB can't handle QPS
  - Data doesn't fit on one server

Example: 10 TB data, 2 TB per server
  Shards needed: 10 TB / 2 TB = 5 shards

With replication (3x per shard):
  5 shards × 3 replicas = 15 DB servers
```

---

## 8. Network Bandwidth

### Bandwidth calculation

```
1 Gbps = 125 MB/s
10 Gbps = 1.25 GB/s

Example: Video streaming (9.8 GB/s)
  10 Gbps links needed: 9.8 GB/s / 1.25 GB/s ≈ 8 links

With redundancy (2x):
  16 × 10 Gbps links
```

---

## 9. Cost Estimation

### AWS rough costs (2024)

```
Compute (EC2):
  - t3.medium (2 vCPU, 4 GB): $30/month
  - m5.large (2 vCPU, 8 GB): $70/month
  - m5.xlarge (4 vCPU, 16 GB): $140/month

Storage (S3):
  - $0.023/GB/month
  - 1 TB = $23/month
  - 1 PB = $23,000/month

Database (RDS):
  - db.m5.large: $120/month
  - db.m5.xlarge: $240/month

Bandwidth:
  - Ingress: Free
  - Egress: $0.09/GB
  - 1 TB egress = $90
```

---

## 10. Interview Tips

### Tip 1: Show your work

```
Don't just give the answer. Show the calculation:

"Let's calculate QPS:
  - 100 million DAU
  - 10 requests per user per day
  - Total: 1 billion requests per day
  - QPS = 1B / 86,400 ≈ 12,000 QPS
  - Peak QPS ≈ 24,000 QPS"
```

---

### Tip 2: Round aggressively

```
Don't say: 11,574 QPS
Say: ~12,000 QPS or ~10,000 QPS

Don't say: 219.7 TB
Say: ~220 TB or ~200 TB

Order of magnitude matters, not precision
```

---

### Tip 3: State assumptions

```
"I'm assuming:
  - 10% of users are daily active
  - Each user makes 10 requests per day
  - Peak traffic is 2x average
  - Read:Write ratio is 100:1"

Interviewer can correct if assumptions are wrong
```

---

### Tip 4: Sanity check

```
After calculation, ask yourself:
  - Does this make sense?
  - Is 1 million QPS reasonable for a startup? (No)
  - Is 10 QPS reasonable for Facebook? (No)

If it seems off, recheck your math
```

---

## 11. Common Interview Questions + Answers

### Q: Estimate the storage needed for Instagram photos.

> "Let's break this down. Instagram has about 1 billion users. Assume 10% are daily active, so 100 million DAU. Each user posts 1 photo per day on average. That's 100 million photos per day. Each photo is about 2 MB after compression. Daily storage is 100M × 2 MB = 200 TB per day. Over 5 years, that's 200 TB × 365 × 5 = 365 PB. With 3x replication for availability, total storage is about 1 exabyte. We'd also need thumbnails, which are smaller, maybe adding 20% more, so roughly 1.2 exabytes total."

### Q: How many servers do you need to handle 100,000 QPS?

> "It depends on the operation, but let's assume it's a simple read from cache. A single server with Redis can handle about 100,000 QPS for simple GET operations. So theoretically, 1 server could handle it. But for high availability, I'd use at least 3 servers with sharding and replication. If it's a database read with joins, a single PostgreSQL server handles about 10,000 QPS, so I'd need 10 read replicas. For writes, PostgreSQL handles about 5,000 QPS, so I'd need 20 primary shards with replication."

### Q: Estimate the bandwidth needed for a video conferencing app with 1 million concurrent users.

> "Video conferencing typically uses about 1-2 Mbps per user for decent quality. Let's use 1.5 Mbps. With 1 million concurrent users, that's 1M × 1.5 Mbps = 1.5 Tbps total bandwidth. But this is peer-to-peer or through a media server. If using a centralized media server, each user sends and receives, so it's 1.5 Tbps × 2 = 3 Tbps. That's 375 GB/s. With 10 Gbps links, we'd need 375 / 1.25 = 300 links. In practice, we'd use a distributed architecture with regional media servers to reduce bandwidth at any single location."

---

## 12. Quick Reference

```
Powers of 2:
  2^10 ≈ 1 thousand (1 KB)
  2^20 ≈ 1 million (1 MB)
  2^30 ≈ 1 billion (1 GB)
  2^40 ≈ 1 trillion (1 TB)

Latency:
  Memory: 100 ns
  Disk seek: 10 ms
  Network (same DC): 0.5 ms
  Network (cross-region): 150 ms

Time:
  1 day ≈ 100,000 seconds (10^5)
  1 year ≈ 31.5 million seconds (3 × 10^7)

QPS calculation:
  QPS = DAU × Actions per day / 86,400
  Peak QPS = Average QPS × 2 (or 3)

Storage calculation:
  Storage = Records × Size × Retention period

Bandwidth calculation:
  Bandwidth = QPS × Request/Response size

Shortcuts:
  1 million requests/day ≈ 12 QPS
  1 billion users → 100M DAU → 12,000 QPS (10 requests/day)

Server capacity:
  PostgreSQL: ~10,000 read QPS, ~5,000 write QPS
  Redis: ~100,000 simple GET QPS
  Typical server: 64 GB RAM

Network:
  1 Gbps = 125 MB/s
  10 Gbps = 1.25 GB/s

Interview tips:
  ✅ Show your work
  ✅ Round aggressively
  ✅ State assumptions
  ✅ Sanity check results
```
