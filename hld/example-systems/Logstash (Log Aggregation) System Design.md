# Log Aggregation System Design (Logstash / ELK Stack)

## System Overview
A centralized log aggregation and analysis platform (think ELK Stack — Elasticsearch, Logstash, Kibana / Datadog / Splunk) that collects logs from distributed services, processes and indexes them, and provides search, alerting, and dashboards for observability.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Collect logs from all services (application logs, access logs, error logs)
- Parse, enrich, and normalize log entries
- Full-text search across logs with filters (service, level, time range)
- Real-time alerting on error patterns (e.g., error rate > 1%)
- Dashboards and visualizations
- Log retention policy (hot: 7 days, warm: 30 days, cold: 1 year)
- Distributed tracing correlation (trace_id across services)

### Non-Functional Requirements
- Throughput: 1M+ log events/sec
- Latency: logs searchable within 30s of generation
- Availability: 99.9% (slightly lower than core services — logs are observability, not product)
- Scalability: petabytes of log data
- Durability: logs must not be lost (at-least-once delivery)

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 1000 microservices, each generating 1K log lines/sec = 1M log lines/sec
- Average log line: 500B
- Hot retention: 7 days; warm: 30 days; cold: 1 year

### Traffic
```
Log ingestion/sec   = 1M lines/sec
Ingestion bandwidth = 1M × 500B = 500MB/sec

Search queries/sec  = 1K/sec (engineers, dashboards, alerts)
```

### Storage
```
Logs/day            = 1M × 500B × 86400 = 43TB/day
Hot (7 days)        = 43TB × 7 = 301TB
Warm (30 days)      = 43TB × 30 = 1.3PB
Cold (1 year)       = 43TB × 365 = 15.7PB → compressed ~3PB
```

## 3. Core Components

**Log Shippers (Filebeat / Fluentd)** — Lightweight agents running on every service host; tail log files; forward to Kafka; handle backpressure and buffering

**Kafka** — Durable log ingestion buffer; decouples shippers from processors; handles burst traffic; partitioned by service/topic

**Log Processor (Logstash / custom)** — Kafka consumer; parses raw log lines; extracts structured fields (timestamp, level, service, trace_id); enriches (adds host metadata, geo-IP); normalizes format; writes to Elasticsearch

**Elasticsearch** — Primary search and analytics store; inverted index on all log fields; hot-warm-cold tiering

**Kibana** — Visualization and dashboard layer; queries Elasticsearch; used by engineers for log search and dashboards

**Alerting Service** — Monitors Elasticsearch for alert conditions (error rate, latency spikes); sends alerts via PagerDuty/Slack/email

**Index Lifecycle Manager (ILM)** — Manages index rollover and tiering: hot → warm → cold → delete based on age and size

**S3 (Cold Storage)** — Long-term log archive; compressed; searchable via Athena for compliance queries

**Redis** — Alert state, deduplication of repeated alerts, rate limiting

## 4. Data Flow Architecture

```
Services → Log Files
               ↓
         Log Shipper (Filebeat)
               ↓
            Kafka
               ↓
         Log Processor (Logstash)
               ↓
         Elasticsearch (hot index)
               ↓ (ILM after 7 days)
         Elasticsearch (warm index)
               ↓ (ILM after 30 days)
              S3 (cold archive)
```

## 5. Database / Storage Design

### Selection Reasoning

| Store | Why |
|---|---|
| Kafka | Durable ingestion buffer; handles 500MB/sec burst; decouples shippers from processors |
| Elasticsearch | Inverted index for full-text log search; aggregations for dashboards; native time-series support |
| S3 | Cost-effective cold storage; compressed; Athena for ad-hoc queries on old logs |
| Redis | Alert deduplication, rate limiting |

### Elasticsearch Index Structure

One index per day per service (or per day for all services):
- `logs-2024-01-15` — all logs for Jan 15
- Rolled over daily by ILM
- Hot tier: SSD nodes, 7-day retention
- Warm tier: HDD nodes, 30-day retention
- Cold tier: frozen/searchable snapshots, 1-year retention

### Log Document Schema (Elasticsearch)

| Field | Type |
|---|---|
| @timestamp | date |
| service | keyword |
| level | keyword (INFO / WARN / ERROR / DEBUG) |
| message | text (full-text indexed) |
| trace_id | keyword |
| span_id | keyword |
| host | keyword |
| environment | keyword (prod / staging) |
| http_status | integer, nullable |
| latency_ms | integer, nullable |
| user_id | keyword, nullable |
| error_type | keyword, nullable |
| stack_trace | text, nullable |

## 6. Key Flows

### 6.1 Log Collection & Ingestion

1. Service writes log line to stdout or log file
2. Filebeat agent on same host tails the file
3. Filebeat buffers locally (handles service restarts without log loss)
4. Filebeat publishes to Kafka topic `logs-{service}` or `logs-all`
5. Kafka retains messages for 24hr (buffer for processor lag)

**Backpressure handling:**
- If Kafka is slow: Filebeat buffers on disk (configurable size)
- If Filebeat disk full: oldest buffered logs dropped (acceptable — logs are observability, not critical data)

### 6.2 Log Processing & Indexing

1. Log Processor (Logstash) consumes from Kafka
2. Parse raw log line:
   - Grok pattern matching for structured extraction
   - JSON parsing for services that log in JSON
3. Enrich:
   - Add `host`, `environment`, `datacenter` from metadata
   - Geo-IP lookup for IP addresses
   - Normalize timestamp to UTC
4. Filter:
   - Drop DEBUG logs in production (reduce volume)
   - Mask PII (email, phone numbers)
5. Write to Elasticsearch hot index
6. Commit Kafka offset after successful write

### 6.3 Log Search

1. Engineer opens Kibana, searches: `service:payment-service AND level:ERROR AND @timestamp:[now-1h TO now]`
2. Kibana translates to Elasticsearch query
3. Elasticsearch searches inverted index across hot + warm indices
4. Returns matching log lines with highlighting
5. Engineer can filter, aggregate, visualize

**Distributed tracing:**
- Search by `trace_id` to see all logs across services for a single request
- `trace_id:abc123` returns logs from API Gateway, Order Service, Payment Service, DB — all correlated

### 6.4 Alerting

1. Alerting Service runs scheduled queries against Elasticsearch every 1 min
2. Example rule: `error rate for payment-service > 1% in last 5 min`
3. Query: `service:payment-service AND level:ERROR` count / total count
4. If threshold exceeded: check Redis dedup key (prevent alert storm)
5. If not recently alerted: send to PagerDuty/Slack
6. Set Redis dedup key with 15min TTL (suppress duplicate alerts)

### 6.5 Index Lifecycle Management (ILM)

```
Day 0–7:   Hot index (SSD, full replicas, fast search)
Day 7–30:  Warm index (HDD, reduced replicas, slower search)
Day 30+:   Snapshot to S3, delete from Elasticsearch
```

ILM automatically:
- Rolls over index when size > 50GB or age > 1 day
- Moves to warm tier after 7 days
- Snapshots to S3 after 30 days
- Deletes from Elasticsearch after 30 days

For compliance queries on old logs: use AWS Athena to query S3 directly (SQL on compressed log files).

## 7. Key Interview Concepts

### Why Kafka Between Shippers and Processors
Without Kafka: if Log Processor is slow or down, Filebeat backs up → service disk fills → service crashes. Kafka acts as a durable buffer — Filebeat writes fast, Processor reads at its own pace. Kafka retains 24hr of logs — processor can lag and catch up.

### Elasticsearch Inverted Index for Logs
Every word in every log message is indexed. Search for "NullPointerException" returns all logs containing that string in milliseconds across billions of log lines. This is why Elasticsearch is the standard for log search — full-text search at scale.

### Hot-Warm-Cold Tiering
Logs from the last 7 days are searched most frequently (debugging recent issues). Older logs are rarely searched. Tiering saves cost:
- Hot (SSD): fast, expensive — recent logs
- Warm (HDD): slower, cheaper — last 30 days
- Cold (S3): very cheap — compliance archive

### Structured Logging
Services should log in JSON format with consistent fields (`trace_id`, `user_id`, `latency_ms`). This makes parsing trivial (no Grok patterns needed) and enables powerful aggregations (avg latency by service, error count by user).

### Sampling for High-Volume Services
At 1M logs/sec, storing everything is expensive. Solution: sample DEBUG/INFO logs (keep 10%), keep all WARN/ERROR. Reduces volume by 80% while preserving all actionable signals.

### Distributed Tracing
Each request gets a `trace_id` at the entry point (API Gateway). All services propagate `trace_id` in their logs. Searching by `trace_id` shows the complete journey of a request across all services — essential for debugging distributed systems.

## 8. Failure Scenarios

### Log Processor Crash
- Detection: Kafka consumer lag grows
- Recovery: restart processor; Kafka retains messages (24hr); processor catches up
- Prevention: multiple processor instances in consumer group; auto-restart

### Elasticsearch Overload
- Detection: indexing latency spikes; search timeouts
- Recovery: reduce indexing rate (backpressure to Kafka); scale Elasticsearch cluster
- Prevention: index rate limiting; separate clusters for indexing vs search; ILM keeps index sizes manageable

### Kafka Full (Disk)
- Detection: Kafka disk usage alert
- Recovery: increase retention period or disk; scale Kafka cluster
- Prevention: monitor disk usage; auto-scale; Filebeat drops oldest buffered logs if disk full (acceptable)

### Alert Storm
- Scenario: 1000 services all error simultaneously → 1000 alerts in 1 min
- Recovery: alert deduplication in Redis; group alerts by service/type; send summary alert
- Prevention: alert grouping rules; dedup TTL (15min); escalation policies
