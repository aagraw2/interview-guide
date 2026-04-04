# Log Aggregation System Design (Logstash / ELK Stack)

## System Overview
A centralized log aggregation platform (think Datadog / Splunk / a custom ELK stack) that collects logs from distributed services across multiple clients, processes and indexes them via a stream processing pipeline, and provides search, alerting, and dashboards for observability. Designed as a multi-tenant service where each client onboards and sends their logs independently.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Client onboarding with token-based authentication
- Collect logs from services via two paths: real-time agent streaming and batch file upload
- Parse, enrich, normalize, and deduplicate log entries
- Full-text search across logs with filters (service, level, time range, trace_id)
- Real-time alerting on error patterns (e.g., error rate > 1%)
- Alert preference management per client (xmatter / email / SMS / pager)
- Log retention policy (hot search: 14 days in Elasticsearch; long-term: 45+ days in S3)
- Distributed tracing correlation (trace_id across services)
- Cron-based cleanup of old logs from primary DB

### Non-Functional Requirements
- Throughput: 1M+ log events/sec
- Latency: logs searchable within 30s of generation
- Availability: 99.9%
- Scalability: petabytes of log data, multiple tenants
- Durability: logs must not be lost (at-least-once delivery)

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 1000 client services, each generating 1K log lines/sec = 1M log lines/sec
- Average log line: 500B
- Elasticsearch retention: 14 days (recent/hot search)
- Cassandra retention: 45 days (primary store)
- S3: all data indefinitely (cold archive)

### Traffic
```
Log ingestion/sec   = 1M lines/sec
Ingestion bandwidth = 1M × 500B = 500MB/sec

Search queries/sec  = 1K/sec (engineers, dashboards, alerts)
```

### Storage
```
Logs/day            = 1M × 500B × 86400 = 43TB/day
Elasticsearch (14d) = 43TB × 14 = 602TB
Cassandra (45d)     = 43TB × 45 = 1.9PB
S3 (cold, 1yr)      = 43TB × 365 ≈ 15.7PB → compressed ~3PB
```

## 3. Core Components

**LB + API Gateway** — Authentication (token validation), authorization, rate limiting, routing

**Client Onboarding Service** — Registers new clients; issues `clientId + tokenId`; writes to Client DB; manages token TTL and preferences

**Agent Forwarder (FluentBit / OTEL / custom agent)** — Lightweight agent running on each service host; tails log files in real-time; forwards to Agent Based Service; handles local buffering and backpressure

**File Upload / Batch Ingest** — Offline ingestion path for bulk historical log uploads; clients upload log files directly; File Ingestion Service processes them

**Agent Based Service** — Receives real-time log streams from Agent Forwarders; validates client token; publishes to Kafka `raw_logs` topic

**File Ingestion Service** — Receives batch file uploads; parses and publishes to Kafka `raw_logs` topic; same downstream pipeline as real-time path

**Kafka** — Durable ingestion buffer; two topics: `raw_logs` (all incoming logs) and `alert_topic` (processed logs matching alert rules); decouples ingestion from processing

**Apache Flink** — Stateful stream processor consuming from Kafka `raw_logs`; four steps per log event:
1. Validation (schema check, required fields)
2. Parse & normalize (extract structured fields, normalize timestamp)
3. Enrich (add host metadata, environment, namespace, pod_name)
4. Deduplicate (drop duplicate log_ids within a time window)

**Log DB (Cassandra)** — Primary durable log store; two table designs for different query patterns; 45-day retention enforced by Cron Service

**Elasticsearch** — Search layer for recent logs (14-day retention); inverted index for full-text search; fed by Flink after processing

**S3** — Cold storage for all logs beyond 45 days; compressed; queryable via Athena for compliance

**Alert Service** — Consumes from Kafka `alert_topic`; evaluates alert rules against AlertPref DB; sends via xmatter / email / SMS / pager

**AlertPref DB** — Per-client alert preferences and rules (thresholds, channels, escalation)

**Search Service** — Handles search queries from clients; queries Elasticsearch for recent logs; falls back to Cassandra for older logs within 45-day window

**Notification Service** — Delivers alert notifications via configured channels

**Cron Service** — Scheduled job that deletes logs older than X days from Cassandra; keeps primary DB size bounded

**Client DB (PostgreSQL)** — Client registry: `clientId, clientName, token, tokenTTL, env, pref, metadata`

## 4. Data Flow Architecture

```
Real-time path:
  Services → Agent Forwarder (FluentBit/OTEL)
                    ↓
           Agent Based Service
                    ↓
                 Kafka (raw_logs)
                    ↓
             Apache Flink
         (validate → parse → enrich → dedup)
                    ↓              ↓
             Cassandra DB    Elasticsearch (14d)
                    ↓              ↓
                   S3 (cold, 45d+)

Batch path:
  Client uploads file → File Ingestion Service → Kafka (raw_logs) → same pipeline

Alert path:
  Flink → Kafka (alert_topic) → Alert Service → Notification Service
```

## 5. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| Cassandra (Log DB) | Primary log store; high write throughput (1M/sec); partition by service_name or trace_id for efficient queries; 45-day retention |
| Elasticsearch | Full-text search on recent logs (14 days); inverted index; aggregations for dashboards |
| S3 | Cost-effective cold archive; all data beyond 45 days; Athena for compliance queries |
| PostgreSQL (Client DB) | Multi-tenant client registry; ACID; small dataset |
| Kafka | Durable ingestion buffer; decouples agents from Flink; handles burst traffic |

### Cassandra — logs_by_service

Partition key: `service_name`, Clustering: `log_date DESC, timestamp DESC`

| Field | Type |
|---|---|
| service_name | VARCHAR (partition key) |
| log_date | DATE (partition key) |
| timestamp | TIMESTAMP (clustering DESC) |
| log_id | UUID |
| log_level | VARCHAR (INFO / WARN / ERROR / DEBUG) |
| host | VARCHAR |
| env | VARCHAR |
| message | TEXT |
| namespace | VARCHAR |
| pod_name | VARCHAR |
| trace_id | UUID |

### Cassandra — logs_by_traceId

Partition key: `trace_id`, Clustering: `timestamp ASC`

| Field | Type |
|---|---|
| trace_id | UUID (partition key) |
| timestamp | TIMESTAMP (clustering) |
| log_id | UUID |
| log_date | DATE |
| service_name | VARCHAR |
| log_level | VARCHAR |
| message | TEXT |

### Elasticsearch — logs index (14-day retention)

| Field | Type |
|---|---|
| @timestamp | date |
| service_name | keyword |
| log_level | keyword |
| message | text (full-text indexed) |
| trace_id | keyword |
| host | keyword |
| env | keyword |
| namespace | keyword |
| pod_name | keyword |
| log_id | keyword |

### PostgreSQL — clients

| Field | Type |
|---|---|
| client_id | UUID (PK) |
| client_name | VARCHAR |
| token | VARCHAR |
| token_ttl | TIMESTAMP |
| env | VARCHAR |
| pref | JSONB |
| metadata | JSONB |

## 6. Key Flows

### 6.1 Client Onboarding

1. New client registers → Client Onboarding Service
2. Generates `clientId + tokenId` with TTL
3. Writes to Client DB (PostgreSQL)
4. Returns credentials to client
5. Client configures Agent Forwarder with token for authentication

### 6.2 Real-Time Log Ingestion

1. Service writes log to stdout / log file
2. Agent Forwarder (FluentBit/OTEL) tails file, buffers locally
3. Forwards to Agent Based Service with client token
4. Agent Based Service validates token against Client DB (or Redis cache)
5. Publishes to Kafka `raw_logs` topic (partitioned by `clientId` or `service_name`)

### 6.3 Batch / Offline Ingestion

1. Client uploads historical log file to File Ingestion Service
2. Service parses file, validates format
3. Publishes log events to Kafka `raw_logs` — same topic as real-time
4. Same Flink pipeline processes both paths identically

### 6.4 Stream Processing (Apache Flink)

Flink consumes from Kafka `raw_logs` and applies four steps per event:

1. **Validation** — check required fields (timestamp, service_name, log_level); drop malformed events
2. **Parse & Normalize** — extract structured fields from raw log string; normalize timestamp to UTC; standardize log level format
3. **Enrich** — add `host`, `env`, `namespace`, `pod_name` from service metadata; add `clientId` from token context
4. **Deduplicate** — check `log_id` in a sliding time window (5 min); drop duplicates (handles agent retry storms)

After processing:
- Write to Cassandra `logs_by_service` and `logs_by_traceId`
- Write to Elasticsearch (14-day index)
- Publish matching alert conditions to Kafka `alert_topic`

### 6.5 Log Search

**Recent logs (< 14 days):**
1. Client queries Search Service: `service:payment AND level:ERROR AND time:[now-1h TO now]`
2. Search Service queries Elasticsearch — fast full-text search
3. Returns matching log lines with highlighting

**Older logs (14–45 days):**
1. Search Service queries Cassandra `logs_by_service` with time range filter
2. Slower but functional for compliance/debugging

**By trace_id (any age up to 45 days):**
1. Search Service queries Cassandra `logs_by_traceId` — all logs for a request across all services
2. Returns correlated log lines in timestamp order

**Cold archive (> 45 days):**
- Query S3 via Athena (SQL on compressed files)
- Used for compliance audits, not regular debugging

### 6.6 Alerting

1. Flink evaluates alert rules on the stream (e.g., error rate > 1% in 5-min window)
2. Publishes matching events to Kafka `alert_topic`
3. Alert Service consumes, checks AlertPref DB for client's alert configuration
4. Sends via configured channel: xmatter / email / SMS / pager
5. Deduplication: suppress repeated alerts for same condition within 15 min

### 6.7 Log Cleanup (Cron Service)

1. Cron Service runs daily
2. Deletes Cassandra rows where `log_date < now - 45 days`
3. Ensures Cassandra storage stays bounded
4. S3 retains all data indefinitely (cheap cold storage)

## 7. Key Interview Concepts

### Why Cassandra as Primary Store (Not Elasticsearch)
Elasticsearch is optimized for search but struggles with write throughput at 1M events/sec sustained. Cassandra handles this easily — append-only writes, partition by service_name + date. Elasticsearch is kept as a 14-day search layer only. This separation gives the best of both: Cassandra for durability and write scale, Elasticsearch for fast search on recent data.

### Two Cassandra Table Designs
Two common query patterns require two tables (denormalization):
- "Show me all ERROR logs for payment-service today" → `logs_by_service` (partition by service_name + date)
- "Show me all logs for trace_id abc123 across all services" → `logs_by_traceId` (partition by trace_id)

Same data written to both tables by Flink. This is the standard Cassandra pattern — model tables around query patterns, not data structure.

### Two Ingestion Paths
Real-time (agent) and batch (file upload) both feed the same Kafka topic. Flink processes both identically. This is important for:
- Onboarding new clients who want to ingest historical logs
- Disaster recovery (re-ingest from S3 backup)
- Offline environments where agents can't stream in real-time

### Apache Flink vs Logstash
Logstash is simpler but stateless — can't do stateful deduplication or windowed aggregations. Flink is stateful — can maintain a deduplication window, compute error rates over time windows for alerting, and handle exactly-once processing. For multi-tenant high-throughput scenarios, Flink is the right choice.

### Kafka as Ingestion Buffer
Without Kafka: if Flink is slow or down, agents back up → service disk fills → service crashes. Kafka acts as a durable buffer — agents write fast, Flink reads at its own pace. Kafka retains 24hr of logs — Flink can lag and catch up.

### Distributed Tracing via trace_id
Each request gets a `trace_id` at the entry point (API Gateway). All services propagate it in their logs. `logs_by_traceId` table in Cassandra makes this query O(1) — all logs for a request in one partition, ordered by timestamp. Essential for debugging distributed systems.

### Hot-Warm-Cold Tiering
- Hot (Elasticsearch, 14 days): fast full-text search, SSD, expensive
- Warm (Cassandra, 45 days): structured queries by service/trace, cheaper
- Cold (S3, indefinite): compliance archive, very cheap, queryable via Athena

### Structured Logging
Services should log in JSON with consistent fields (`trace_id`, `service_name`, `log_level`). Makes Flink parsing trivial — no Grok patterns needed. Enables powerful aggregations (avg latency by service, error count by namespace).

### Sampling for High-Volume Services
At 1M logs/sec, storing everything is expensive. Sample DEBUG/INFO logs (keep 10%), keep all WARN/ERROR. Reduces volume by 80% while preserving all actionable signals. Applied in Flink's filter step.

## 8. Failure Scenarios

### Flink Crash
- Detection: Kafka consumer lag grows on `raw_logs`
- Recovery: Flink restarts from last checkpoint; Kafka retains messages (24hr); catches up
- Prevention: Flink checkpointing every 30s; multiple task managers; auto-restart

### Cassandra Node Failure
- Impact: some log writes fail; search on affected partitions degraded
- Recovery: RF=3, QUORUM writes continue on remaining replicas; hinted handoff replays on recovery
- Prevention: multi-AZ deployment; Elasticsearch still serves recent log search during Cassandra degradation

### Elasticsearch Overload
- Detection: indexing latency spikes; search timeouts
- Recovery: reduce indexing rate (backpressure from Flink); scale Elasticsearch cluster
- Prevention: Elasticsearch only handles 14-day window — bounded dataset; separate clusters for indexing vs search

### Agent Forwarder Disk Full
- Scenario: Kafka slow → agent buffers fill → service disk fills
- Recovery: agent drops oldest buffered logs (acceptable — logs are observability, not critical data)
- Prevention: monitor agent buffer size; alert before disk full; Kafka auto-scaling

### Alert Storm
- Scenario: 1000 services all error simultaneously → 1000 alerts in 1 min
- Recovery: Alert Service deduplicates by condition + client; sends summary alert
- Prevention: 15-min dedup TTL; alert grouping rules; escalation policies
