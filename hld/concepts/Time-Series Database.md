## 1. What is a Time-Series Database?

A time-series database (TSDB) is optimized for storing and querying data points indexed by time. Each data point consists of a timestamp, a metric name, and a value, often with additional tags/labels.

```
Examples:
  - InfluxDB
  - TimescaleDB (PostgreSQL extension)
  - Prometheus
  - OpenTSDB
  - Amazon Timestream
```

**Core use case:** Monitoring, metrics, IoT sensor data, financial tick data — anything where you're tracking how values change over time.

---

## 2. Time-Series Data Model

### Data point structure

```
timestamp | metric_name | value | tags
----------|-------------|-------|------------------
1640000000| cpu_usage   | 75.2  | host=server1, region=us-east
1640000060| cpu_usage   | 78.5  | host=server1, region=us-east
1640000120| cpu_usage   | 72.1  | host=server1, region=us-east
```

### InfluxDB data model

```
measurement: cpu_usage
tags: {host: server1, region: us-east}
fields: {value: 75.2}
timestamp: 1640000000

Line protocol:
  cpu_usage,host=server1,region=us-east value=75.2 1640000000
```

### Prometheus data model

```
metric_name{label1="value1", label2="value2"} value timestamp

Example:
  http_requests_total{method="GET", status="200"} 1234 1640000000
```

---

## 3. Why Not Use a Regular Database?

### Problem 1: Write throughput

Time-series data has extremely high write rates.

```
1000 servers × 100 metrics/server × 1 sample/sec = 100,000 writes/sec

Traditional SQL database:
  - B-tree index updates on every write (random I/O)
  - Row-level locking
  - Transaction overhead
  → Can't handle 100k writes/sec

Time-series database:
  - Append-only writes (sequential I/O)
  - Batch writes
  - No locks
  → Easily handles millions of writes/sec
```

### Problem 2: Storage efficiency

Time-series data is highly compressible.

```
Raw data (1 million points, 8 bytes each):
  1M × 8 bytes = 8 MB

With delta encoding + compression:
  500 KB (16x compression)

SQL database: Stores raw values
TSDB: Specialized compression algorithms
```

### Problem 3: Query patterns

Time-series queries are different from SQL queries.

```
SQL query:
  SELECT * FROM users WHERE age > 30

Time-series query:
  SELECT avg(cpu_usage) FROM metrics
  WHERE time > now() - 1h
  GROUP BY time(5m), host

Time-series queries:
  - Always filter by time range
  - Aggregate over time windows (5m, 1h, 1d)
  - Downsample for visualization
```

---

## 4. Storage Optimization

### Delta encoding

Store the difference between consecutive values instead of absolute values.

```
Raw values:
  1640000000, 1640000060, 1640000120, 1640000180
  (4 × 8 bytes = 32 bytes)

Delta encoding:
  1640000000, +60, +60, +60
  (8 bytes + 3 × 1 byte = 11 bytes)

Compression: 32 bytes → 11 bytes (3x)
```

### Run-length encoding

Store repeated values once with a count.

```
Raw values:
  75.2, 75.2, 75.2, 75.2, 78.5, 78.5

Run-length encoding:
  75.2 × 4, 78.5 × 2

Compression: 6 values → 2 values + counts
```

### Gorilla compression (Facebook)

Used by Prometheus. Combines delta-of-delta encoding with XOR compression.

```
Timestamps (delta-of-delta):
  1640000000, 1640000060, 1640000120, 1640000180
  Deltas: 60, 60, 60
  Delta-of-deltas: 0, 0 (constant interval)
  → Store as: base timestamp + interval

Values (XOR):
  75.2, 75.5, 75.8
  XOR(75.2, 75.5) = small difference → few bits
  XOR(75.5, 75.8) = small difference → few bits
  → Store only the differing bits

Compression: 12 bytes/point → 1.37 bytes/point (9x)
```

---

## 5. Data Retention and Downsampling

### Problem: Infinite storage growth

```
100,000 writes/sec × 86,400 sec/day = 8.6 billion points/day

At 10 bytes/point:
  8.6B × 10 bytes = 86 GB/day
  86 GB × 365 days = 31 TB/year

Storing raw data forever is impractical
```

### Solution 1: TTL (Time To Live)

Automatically delete old data after a specified duration.

```
InfluxDB retention policy:
  CREATE RETENTION POLICY "30_days"
  ON "metrics"
  DURATION 30d
  REPLICATION 1

Data older than 30 days is automatically deleted
```

### Solution 2: Downsampling

Aggregate high-resolution data into lower-resolution summaries.

```
Raw data (1-second resolution):
  Keep for 7 days

1-minute aggregates:
  Keep for 30 days

1-hour aggregates:
  Keep for 1 year

1-day aggregates:
  Keep forever
```

```
Example:
  Raw: 1640000000 → 75.2, 1640000001 → 75.5, 1640000002 → 75.8, ...
  1-minute aggregate: 1640000000 → avg=75.5, min=75.2, max=75.8
```

### Continuous aggregation (TimescaleDB)

```sql
CREATE MATERIALIZED VIEW cpu_usage_1h
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 hour', time) AS bucket,
  host,
  avg(value) AS avg_cpu,
  max(value) AS max_cpu
FROM cpu_usage
GROUP BY bucket, host;

Automatically updates as new data arrives
```

---

## 6. Querying Time-Series Data

### Time range queries

```
InfluxDB:
  SELECT mean(value)
  FROM cpu_usage
  WHERE time > now() - 1h
  GROUP BY time(5m), host

Prometheus:
  rate(http_requests_total[5m])
```

### Aggregation functions

```
Common aggregations:
  - avg, min, max, sum, count
  - percentile (p50, p95, p99)
  - rate (change per second)
  - derivative (rate of change)
```

### Windowing

```
Tumbling window (non-overlapping):
  [00:00-00:05], [00:05-00:10], [00:10-00:15]

Sliding window (overlapping):
  [00:00-00:05], [00:01-00:06], [00:02-00:07]

InfluxDB:
  GROUP BY time(5m)  -- tumbling window

Prometheus:
  rate(metric[5m])   -- sliding window
```

---

## 7. Indexing Strategies

### Time-based partitioning

Data is partitioned by time range (e.g., one partition per day).

```
Partitions:
  2024-01-01: [data for Jan 1]
  2024-01-02: [data for Jan 2]
  2024-01-03: [data for Jan 3]

Query: SELECT * FROM metrics WHERE time > '2024-01-02'
  → Only scan partitions 2024-01-02 and 2024-01-03
  → Skip older partitions
```

### Inverted index on tags

Tags (labels) are indexed for fast filtering.

```
Tag: host=server1
  → Points: [p1, p2, p3, ...]

Tag: region=us-east
  → Points: [p1, p4, p5, ...]

Query: WHERE host=server1 AND region=us-east
  → Intersect the two index lists: [p1]
```

### TimescaleDB hypertables

```sql
CREATE TABLE metrics (
  time TIMESTAMPTZ NOT NULL,
  host TEXT,
  value DOUBLE PRECISION
);

SELECT create_hypertable('metrics', 'time');

Automatically partitions by time (chunks)
  → Efficient time-range queries
  → Old chunks can be compressed or dropped
```

---

## 8. Prometheus Architecture

Prometheus is a popular open-source TSDB for monitoring.

### Pull-based model

```
Prometheus server:
  1. Scrape metrics from targets (HTTP endpoints)
  2. Store in local TSDB
  3. Evaluate alerting rules
  4. Serve queries (PromQL)

Targets (exporters):
  - Node exporter (system metrics)
  - Application metrics (/metrics endpoint)
  - Kubernetes metrics
```

### PromQL (Prometheus Query Language)

```
# Instant query (current value)
http_requests_total{job="api", status="200"}

# Range query (over time)
rate(http_requests_total[5m])

# Aggregation
sum by (status) (rate(http_requests_total[5m]))

# Percentile
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

### Limitations

- **Single-node:** Prometheus is not distributed (use Thanos or Cortex for scale)
- **No long-term storage:** Local storage only (use remote write to external TSDB)
- **Pull-only:** Can't push metrics directly (use Pushgateway for batch jobs)

---

## 9. InfluxDB vs TimescaleDB vs Prometheus

|Feature|InfluxDB|TimescaleDB|Prometheus|
|---|---|---|---|
|Type|Purpose-built TSDB|PostgreSQL extension|Monitoring-focused TSDB|
|Query language|InfluxQL / Flux|SQL|PromQL|
|Write model|Push (HTTP API)|Push (SQL INSERT)|Pull (scrape targets)|
|Clustering|InfluxDB Enterprise|PostgreSQL replication|Single-node (use Thanos)|
|Compression|High (Gorilla-like)|High (columnar)|High (Gorilla)|
|Retention|TTL + downsampling|TTL + continuous aggregates|TTL (local storage)|
|Use case|General time-series|SQL compatibility needed|Monitoring & alerting|

**Choose InfluxDB when:** You need a standalone TSDB with high write throughput and flexible querying.

**Choose TimescaleDB when:** You want SQL compatibility and PostgreSQL ecosystem (joins, transactions).

**Choose Prometheus when:** You're building a monitoring system with alerting and service discovery.

---

## 10. Use Cases

### 1. Infrastructure monitoring

CPU, memory, disk, network metrics from servers.

```
Metrics:
  cpu_usage{host=server1, region=us-east} 75.2
  memory_usage{host=server1, region=us-east} 8.5GB
  disk_io{host=server1, region=us-east} 1200 IOPS

Query: Show avg CPU usage per region over last 24h
```

### 2. Application performance monitoring (APM)

Request latency, error rates, throughput.

```
Metrics:
  http_request_duration_seconds{method=GET, status=200} 0.123
  http_requests_total{method=GET, status=200} 1234

Query: 95th percentile latency for GET requests
  histogram_quantile(0.95, http_request_duration_seconds)
```

### 3. IoT sensor data

Temperature, humidity, pressure from millions of devices.

```
Metrics:
  temperature{sensor_id=123, location=warehouse_a} 25.5
  humidity{sensor_id=123, location=warehouse_a} 60.2

Query: Show temperature trend for warehouse_a over last week
```

### 4. Financial tick data

Stock prices, trading volume, bid/ask spreads.

```
Metrics:
  stock_price{symbol=AAPL, exchange=NASDAQ} 150.25
  trading_volume{symbol=AAPL, exchange=NASDAQ} 1000000

Query: Show 1-minute candlestick chart for AAPL
```

---

## 11. Common Interview Questions + Answers

### Q: Why use a time-series database instead of a regular SQL database?

> "Time-series databases are optimized for high write throughput with append-only storage, specialized compression algorithms like delta encoding and Gorilla compression, and time-range queries with aggregation. A SQL database with B-tree indexes can't handle millions of writes per second, and it doesn't have built-in downsampling or retention policies. For time-series data, a TSDB can achieve 10-100x better write performance and 10x better compression."

### Q: How do time-series databases achieve high compression?

> "They use delta encoding to store differences between consecutive timestamps, run-length encoding for repeated values, and algorithms like Gorilla compression that combine delta-of-delta encoding with XOR compression. Since time-series data is highly regular — timestamps are often evenly spaced and values change slowly — these techniques can compress data by 10-20x compared to storing raw values."

### Q: What's the difference between downsampling and retention policies?

> "Retention policies automatically delete old data after a specified duration — for example, delete data older than 30 days. Downsampling aggregates high-resolution data into lower-resolution summaries — for example, convert 1-second data into 1-minute averages. You typically combine both: keep raw data for 7 days, 1-minute aggregates for 30 days, 1-hour aggregates for 1 year, and 1-day aggregates forever."

### Q: When would you use Prometheus vs InfluxDB?

> "Use Prometheus for monitoring and alerting with a pull-based model — it's designed to scrape metrics from targets and has built-in alerting rules. Use InfluxDB for general time-series data with a push-based model — it's better for IoT, financial data, or any use case where you're pushing data from many sources. Prometheus is single-node by default; InfluxDB has better clustering support."

---

## 12. Interview Tricks & Pitfalls

### ✅ Trick 1: Emphasize write optimization

Time-series databases are designed for high write throughput. Always mention append-only storage, batch writes, and no locking.

### ✅ Trick 2: Know the compression techniques

Delta encoding, run-length encoding, and Gorilla compression are common interview topics. Mentioning specific algorithms shows depth.

### ✅ Trick 3: Discuss retention and downsampling

Production time-series systems always have retention policies and downsampling. Mentioning this shows you understand operational concerns.

### ❌ Pitfall 1: Confusing TSDB with columnar databases

Time-series databases (InfluxDB, Prometheus) are for high-frequency writes and time-range queries. Columnar databases (Redshift, BigQuery) are for analytics with complex aggregations. Don't mix them up.

### ❌ Pitfall 2: Thinking TSDB is only for monitoring

Time-series databases are used for IoT, financial data, and any data indexed by time. Don't limit your answer to just monitoring.

### ❌ Pitfall 3: Forgetting about cardinality

High-cardinality tags (e.g., user_id with millions of unique values) can cause performance issues. Mentioning cardinality shows you understand the limitations.

---

## 13. Quick Reference

```
What is a time-series database?
  Optimized for data points indexed by time
  High write throughput, specialized compression, time-range queries
  Examples: InfluxDB, TimescaleDB, Prometheus, OpenTSDB

Data model:
  timestamp | metric_name | value | tags
  Tags = indexed dimensions (host, region, etc.)
  Fields = measured values (cpu_usage, temperature, etc.)

Why not SQL?
  SQL: B-tree indexes, row-level locking, transaction overhead
  TSDB: Append-only writes, batch inserts, no locks
  TSDB: 10-100x better write performance

Compression:
  Delta encoding: Store differences, not absolute values
  Run-length encoding: Store repeated values once
  Gorilla compression: Delta-of-delta + XOR (10-20x compression)

Retention and downsampling:
  Retention: Delete old data after TTL (e.g., 30 days)
  Downsampling: Aggregate high-res data into low-res summaries
  Example: Raw (7d) → 1m (30d) → 1h (1y) → 1d (forever)

Indexing:
  Time-based partitioning: One partition per day/week
  Inverted index on tags: Fast filtering by tag values
  TimescaleDB hypertables: Automatic time-based partitioning

Query patterns:
  Time-range queries: WHERE time > now() - 1h
  Aggregation: avg, min, max, percentile, rate
  Windowing: GROUP BY time(5m)

InfluxDB vs TimescaleDB vs Prometheus:
  InfluxDB    → General TSDB, push model, InfluxQL/Flux
  TimescaleDB → SQL compatibility, PostgreSQL extension
  Prometheus  → Monitoring, pull model, PromQL, alerting

Use cases:
  Infrastructure monitoring (CPU, memory, disk)
  APM (request latency, error rates)
  IoT (sensor data from millions of devices)
  Financial (stock prices, trading volume)

Cardinality:
  High-cardinality tags (millions of unique values) → performance issues
  Keep cardinality low (e.g., host, region, not user_id)
```
