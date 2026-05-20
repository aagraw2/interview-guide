# Analytics Dashboard API and DB Design

## Problem Statement

Design the API and database schema for an analytics dashboard that shows business metrics (traffic, conversions, revenue) across time ranges with filters.

**Core requirements:**
- Ingest events from multiple sources
- Query aggregated metrics by time range and dimensions
- Support near real-time charts (minute/hour/day granularity)
- Allow saved dashboards with reusable widgets

**Scale:** 2M events/min ingestion, read-heavy aggregate queries

---

## API Design

### Core Endpoints
```http
POST /api/v1/events
POST /api/v1/metrics/query
GET /api/v1/dashboards/{dashboardId}
POST /api/v1/dashboards
PATCH /api/v1/dashboards/{dashboardId}
```

### Example: Query Metrics
```json
POST /api/v1/metrics/query
{
  "metric": "revenue",
  "timeRange": { "from": "2026-05-01T00:00:00Z", "to": "2026-05-20T00:00:00Z" },
  "granularity": "day",
  "groupBy": ["country", "channel"],
  "filters": [{ "field": "plan", "op": "IN", "values": ["pro", "enterprise"] }]
}
```

```json
200 OK
{
  "series": [
    { "bucket": "2026-05-01", "country": "IN", "channel": "organic", "value": 18345.5 }
  ]
}
```

### Design Decisions
- Separate ingestion API from query API to isolate workloads
- Async validation for event schema evolution
- Query endpoint enforces max time window to protect cluster

---

## Database Schema

```sql
CREATE TABLE raw_events (
    id              BIGSERIAL PRIMARY KEY,
    event_id        VARCHAR(64) NOT NULL UNIQUE,
    event_type      VARCHAR(64) NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL,
    user_id         VARCHAR(64),
    properties      JSONB NOT NULL
);

CREATE INDEX idx_raw_events_occurred_at ON raw_events (occurred_at);
CREATE INDEX idx_raw_events_event_type_time ON raw_events (event_type, occurred_at);

CREATE TABLE metric_aggregates_hourly (
    bucket_start    TIMESTAMPTZ NOT NULL,
    metric_name     VARCHAR(64) NOT NULL,
    dimension_key   VARCHAR(200) NOT NULL,
    value_sum       DOUBLE PRECISION NOT NULL DEFAULT 0,
    value_count     BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (bucket_start, metric_name, dimension_key)
);

CREATE TABLE dashboards (
    id              VARCHAR(64) PRIMARY KEY,
    owner_id        VARCHAR(64) NOT NULL,
    name            VARCHAR(120) NOT NULL,
    layout_json     JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Trade-offs and Considerations

- Pre-aggregation reduces query latency but increases ingestion complexity.
- Time-series stores can outperform general RDBMS for large scans.
- Late-arriving events require backfill jobs for aggregate correction.
