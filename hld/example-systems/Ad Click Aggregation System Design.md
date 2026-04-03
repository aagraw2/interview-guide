# Ad Click Aggregation System Design

## System Overview
A real-time ad click aggregation system that ingests billions of click events, aggregates them by ad/campaign/time window, detects click fraud, and serves aggregated metrics to advertisers — similar to Google Ads or Facebook Ads analytics backend.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Ingest ad click events in real-time (adId, userId, timestamp, IP, device)
- Aggregate clicks by adId, campaignId, time window (per minute, hour, day)
- Detect and filter fraudulent clicks (bot traffic, click farms)
- Serve aggregated metrics to advertisers (click count, CTR, spend)
- Support queries: "clicks for ad X in last 1hr", "top 10 ads by clicks today"
- Billing: charge advertisers based on valid click counts

### Non-Functional Requirements
- Throughput: 1M+ click events/sec at peak
- Latency: aggregated metrics available within 1 min of click
- Accuracy: billing-critical — must not over-count or under-count valid clicks
- Idempotency: duplicate click events must not inflate counts
- Scalability: 100B+ clicks/day, 10M+ active ads

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 100B clicks/day (including fraudulent)
- 30% fraud rate → 70B valid clicks/day
- Peak: 1M clicks/sec (flash sale, major event)
- Average click event: 200B
- 10M active ads, 1M campaigns

### Traffic
```
Clicks/sec (avg)    = 100B / 86400 ≈ 1.16M/sec
Clicks/sec (peak)   = 3M/sec

Kafka throughput    = 3M × 200B = 600MB/sec
```

### Storage
```
Raw events/day      = 100B × 200B = 20TB/day
  → retain 30 days = 600TB rolling
Aggregated data     = 10M ads × 1440 min/day × 50B = 720GB/day
  → retain 2 years = ~525TB
```

## 3. Core Components

**API Gateway** — Auth, rate limiting, routing for click ingestion and query APIs

**Click Ingestion Service** — Receives click events; validates basic format; deduplicates; publishes to Kafka; returns 200 immediately (async)

**Kafka** — Durable click event stream; partitioned by `adId` for ordered processing per ad; high-throughput backbone

**Fraud Detection Service** — Kafka consumer; real-time fraud scoring per click (IP reputation, click velocity, user agent); filters fraudulent clicks; publishes valid clicks to separate Kafka topic

**Stream Aggregation Service** — Kafka consumer on valid clicks topic; maintains in-memory rolling windows (per minute, per hour); flushes aggregated counts to Aggregation DB

**Batch Aggregation Service** — Daily batch job; reprocesses raw events from cold storage for accurate billing reconciliation; handles late-arriving events

**Query Service** — Serves aggregated metrics to advertisers; reads from Aggregation DB and Redis cache

**Billing Service** — Reads daily aggregated valid click counts; generates invoices; integrates with payment system

**Click Event Store (Cassandra)** — Raw click events; append-only; partition by adId + date

**Aggregation DB (InfluxDB / Cassandra)** — Pre-aggregated click counts by adId + time window

**Redis** — Real-time aggregation buffer, deduplication cache, query result cache

**S3** — Cold storage for raw events (30-day retention); batch processing source

**Kafka** — Two topics: `raw-clicks` and `valid-clicks`

## 4. Database Design

### Cassandra — click_events (raw)

Partition key: `ad_id + date`, Clustering: `timestamp DESC`

| Field | Type |
|---|---|
| ad_id | UUID (partition key) |
| date | DATE (partition key) |
| click_id | UUID (clustering) |
| timestamp | TIMESTAMP |
| user_id | UUID, nullable |
| ip_address | VARCHAR |
| user_agent | VARCHAR |
| device_type | VARCHAR |
| is_fraud | BOOLEAN |
| session_id | VARCHAR |

### Cassandra / InfluxDB — click_aggregations

| Field | Type |
|---|---|
| ad_id | UUID |
| campaign_id | UUID |
| window_start | TIMESTAMP |
| window_type | TEXT (minute / hour / day) |
| total_clicks | BIGINT |
| valid_clicks | BIGINT |
| fraud_clicks | BIGINT |
| unique_users | BIGINT |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `dedup:{clickId}` | String | "1" | 3600s (dedup window) |
| `clicks:realtime:{adId}` | Counter | click count in current minute | 120s |
| `fraud:ip:{ip}` | Counter | clicks from IP in 1hr | 3600s |
| `query:cache:{queryHash}` | String | aggregation result JSON | 60s |

## 5. Key Flows

### 5.1 Click Ingestion

```
Ad click → Click Ingestion Service
                  ↓
    Validate: required fields present
    Dedup: check Redis dedup:{clickId}
                  ↓
    Publish to Kafka: raw-clicks topic (partitioned by adId)
                  ↓
    Return 200 OK immediately
```

1. User clicks ad → browser/app sends `POST /click` with `{clickId, adId, userId, ip, userAgent, timestamp}`
2. Click Ingestion Service validates format
3. Deduplication: `SET dedup:{clickId} 1 NX EX 3600` — if key exists, duplicate → discard
4. Publish to Kafka `raw-clicks` topic, partition key = `adId`
5. Return 200 immediately (async processing)

**Why partition by adId:** all clicks for the same ad go to the same Kafka partition → ordered processing → accurate per-ad aggregation without cross-partition coordination.

### 5.2 Fraud Detection

```
Kafka (raw-clicks) → Fraud Detection Service
                              ↓
    Score each click:
      - IP velocity: >100 clicks/hr from same IP → fraud
      - User velocity: >50 clicks/hr from same user → fraud
      - Bot detection: known bot user agents → fraud
      - Click pattern: clicks too fast (< 1s between clicks) → fraud
              ↓
    Valid click → publish to Kafka (valid-clicks)
    Fraud click → publish to Kafka (fraud-clicks) + update Cassandra is_fraud=true
```

**Fraud signals:**
- IP reputation: check against known bot IP lists
- Click velocity: `INCR fraud:ip:{ip}` in Redis; if > threshold → fraud
- User agent: known bot patterns (Googlebot, scrapers)
- Geographic anomaly: clicks from unexpected regions for targeted ad
- Time pattern: clicks at inhuman speed

### 5.3 Stream Aggregation

```
Kafka (valid-clicks) → Stream Aggregation Service
                                  ↓
    Maintain in-memory windows:
      per-minute: {adId → count} for current minute
      per-hour:   {adId → count} for current hour
                                  ↓
    On window close (every minute):
      Flush to Aggregation DB: {adId, window_start, valid_clicks}
      Update Redis real-time counter
```

1. Stream Aggregation Service maintains in-memory hash maps per time window
2. On each valid click: `window[adId]++`
3. Every minute: flush window to Aggregation DB (InfluxDB or Cassandra)
4. Hourly and daily aggregations computed from minute-level data
5. Late-arriving events (network delay): handled by allowing 5-min late window; events within 5 min of window close still counted

### 5.4 Batch Reconciliation

Daily batch job for billing accuracy:
1. Read all raw click events from S3 (previous day)
2. Re-apply fraud detection with updated models (more accurate than real-time)
3. Recompute daily aggregations
4. Compare with stream aggregations — flag discrepancies
5. Use batch results for billing (more accurate than stream)

This two-layer approach (stream for real-time display, batch for billing) is the Lambda Architecture pattern.

### 5.5 Advertiser Query

1. Advertiser queries "clicks for campaign X in last 24hr"
2. Query Service checks Redis cache (TTL 60s)
3. Cache miss: query Aggregation DB for hourly buckets in range
4. Sum up buckets, return result
5. Cache result in Redis

## 6. Key Interview Concepts

### Lambda Architecture
Two processing paths for different needs:
- **Speed layer (stream):** Kafka → Stream Aggregation → real-time metrics (approximate, low latency)
- **Batch layer:** S3 → Batch Aggregation → accurate billing metrics (exact, higher latency)
- **Serving layer:** merges both for queries

Stream gives fast approximate results; batch gives accurate final results. Billing uses batch; dashboards use stream.

### Kafka Partitioning by adId
All clicks for the same ad go to the same partition → same consumer instance processes them in order → accurate per-ad counting without distributed coordination. If partitioned randomly, clicks for the same ad could go to different consumers → need cross-consumer aggregation (complex).

### Deduplication
Same click event can arrive multiple times (network retry, browser retry). Solution: `clickId` (UUID generated client-side) + Redis `SET NX EX` with 1hr TTL. If `clickId` already seen → discard. TTL limits Redis memory usage.

### Late-Arriving Events
Network delays mean a click at 12:59:55 might arrive at 13:00:05 (after the 1pm window closed). Solution: allow 5-min late window — events arriving within 5 min of window close are still counted in that window. Events arriving later are counted in current window (acceptable approximation).

### Fraud Detection at Scale
1M clicks/sec through fraud detection. Must be fast (<1ms per click). Solution:
- Simple rules in-memory (IP velocity, user agent) — O(1) Redis lookups
- Complex ML models run async on sampled traffic
- Known fraud IPs/users cached in Redis for fast lookup

### Billing Accuracy
Advertisers pay per valid click. Over-counting = advertiser overpays (legal risk). Under-counting = revenue loss. Solution: batch reconciliation with more accurate fraud models; billing uses batch results; stream results are for display only.

## 7. Failure Scenarios

### Kafka Consumer Lag (Stream Aggregation)
- Impact: real-time metrics delayed; billing unaffected (uses batch)
- Recovery: scale up Stream Aggregation instances; Kafka retains events; catches up
- Prevention: monitor consumer lag; auto-scale on lag threshold

### Aggregation DB Failure
- Impact: real-time metric queries fail; stream aggregation buffers in memory
- Recovery: InfluxDB cluster failover; buffered aggregations flushed on recovery
- Prevention: InfluxDB clustering; Redis cache absorbs query load during brief outage

### Fraud Detection Service Failure
- Impact: all clicks treated as valid (fail open) or all blocked (fail closed)
- Recovery: fail open with enhanced logging; alert ops; Fraud Service is stateless, restarts quickly
- Prevention: multiple Fraud Detection instances; circuit breaker

### Duplicate Click Storm (DDoS)
- Scenario: attacker sends millions of clicks with same clickId
- Recovery: Redis `SET NX` deduplicates all duplicates; only first click processed
- Prevention: rate limiting at API Gateway per IP; CAPTCHA for suspicious traffic

### Late Batch Reconciliation Discrepancy
- Scenario: stream count = 1M, batch count = 950K (50K fraud detected in batch)
- Recovery: billing uses batch count; stream count updated retroactively in dashboard
- Prevention: this is expected behavior — stream is approximate, batch is authoritative
