## 1. What is Observability?

**Observability** is the ability to understand the internal state of a system by examining its external outputs.

```
Monitoring:     "Is the system working?" (known unknowns)
                → Check predefined metrics, alert on thresholds

Observability:  "Why is the system behaving this way?" (unknown unknowns)
                → Explore data to understand unexpected behavior
```

**The Three Pillars of Observability:**

```
1. Metrics:  Numerical measurements over time (CPU, latency, error rate)
2. Logs:     Timestamped records of events (errors, requests, state changes)
3. Traces:   Request flow across services (distributed tracing)
```

---

## 2. Metrics

### What are Metrics?

Numerical measurements aggregated over time.

```
Examples:
  - Request rate (requests per second)
  - Error rate (errors per second)
  - Latency (p50, p95, p99)
  - CPU usage (%)
  - Memory usage (MB)
  - Queue depth (number of messages)
```

---

### Types of Metrics

#### Counter

Monotonically increasing value. Resets to zero on restart.

```
Examples:
  - Total requests
  - Total errors
  - Total bytes sent

Usage:
  - Calculate rate: requests per second
  - Calculate percentage: error rate = errors / requests
```

---

#### Gauge

Current value that can go up or down.

```
Examples:
  - CPU usage (%)
  - Memory usage (MB)
  - Active connections
  - Queue depth

Usage:
  - Monitor current state
  - Alert on thresholds (CPU > 80%)
```

---

#### Histogram

Distribution of values (latency, request size).

```
Examples:
  - Request latency (p50, p95, p99)
  - Request size (bytes)
  - Response size (bytes)

Usage:
  - Understand distribution (most requests fast, some slow)
  - Alert on percentiles (p99 > 1s)
```

---

#### Summary

Similar to histogram, but calculates percentiles on the client side.

```
Histogram:  Server calculates percentiles (more accurate)
Summary:    Client calculates percentiles (less load on server)
```

---

### Metric Dimensions (Labels)

Add context to metrics with labels.

```
http_requests_total{method="GET", status="200", endpoint="/api/users"}
http_requests_total{method="POST", status="500", endpoint="/api/orders"}

Query:
  - Total requests: sum(http_requests_total)
  - Requests by endpoint: sum(http_requests_total) by (endpoint)
  - Error rate: sum(http_requests_total{status=~"5.."}) / sum(http_requests_total)
```

**Best practice:** Use low-cardinality labels (method, status, endpoint). Avoid high-cardinality labels (user_id, request_id).

---

### Prometheus Example

```yaml
# Prometheus config
scrape_configs:
  - job_name: 'myapp'
    static_configs:
      - targets: ['localhost:8080']
```

**Application exposes metrics:**

```python
from prometheus_client import Counter, Histogram, start_http_server

# Counter
requests_total = Counter('http_requests_total', 'Total requests', ['method', 'status'])

# Histogram
request_duration = Histogram('http_request_duration_seconds', 'Request duration')

@app.route('/api/users')
def get_users():
    with request_duration.time():
        # Handle request
        requests_total.labels(method='GET', status='200').inc()
        return users

# Expose metrics on /metrics
start_http_server(8080)
```

**Query in Prometheus:**

```promql
# Request rate (per second)
rate(http_requests_total[5m])

# Error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# p99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

---

### Grafana Dashboards

Visualize metrics from Prometheus.

```
Dashboard panels:
  - Request rate (line chart)
  - Error rate (line chart)
  - Latency (p50, p95, p99) (line chart)
  - CPU usage (gauge)
  - Memory usage (gauge)
```

---

## 3. Logs

### What are Logs?

Timestamped records of events.

```
Examples:
  2026-04-01T12:00:00Z INFO  Request received: GET /api/users
  2026-04-01T12:00:01Z ERROR Database connection failed: timeout
  2026-04-01T12:00:02Z INFO  Response sent: 200 OK
```

---

### Structured Logging

Use structured formats (JSON) instead of plain text.

```
Bad (plain text):
  2026-04-01T12:00:00Z Request received: GET /api/users user_id=123

Good (JSON):
  {
    "timestamp": "2026-04-01T12:00:00Z",
    "level": "INFO",
    "message": "Request received",
    "method": "GET",
    "path": "/api/users",
    "user_id": 123
  }
```

**Benefits:**

- Easy to parse and query
- Consistent format across services
- Can add context (user_id, request_id, trace_id)

---

### Log Levels

```
TRACE:  Very detailed (function entry/exit)
DEBUG:  Detailed information for debugging
INFO:   General information (request received, job completed)
WARN:   Warning (deprecated API used, retry attempted)
ERROR:  Error (request failed, database connection lost)
FATAL:  Critical error (service crashing)
```

**Best practice:** Use INFO in production. Use DEBUG for troubleshooting.

---

### Centralized Logging

Aggregate logs from all services in one place.

```
┌─────────┐  ┌─────────┐  ┌─────────┐
│Service A│  │Service B│  │Service C│
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     │ Ship logs  │            │
     ▼            ▼            ▼
┌────────────────────────────────────┐
│      Log Aggregator (Fluentd)      │
└────────────────┬───────────────────┘
                 │
                 ▼
┌────────────────────────────────────┐
│   Log Storage (Elasticsearch)      │
└────────────────┬───────────────────┘
                 │
                 ▼
┌────────────────────────────────────┐
│   Log Viewer (Kibana)              │
└────────────────────────────────────┘
```

**ELK Stack:** Elasticsearch (storage), Logstash/Fluentd (aggregation), Kibana (viewer).

---

### Example: Fluentd Config

```yaml
<source>
  @type tail
  path /var/log/myapp/*.log
  pos_file /var/log/fluentd/myapp.pos
  tag myapp
  <parse>
    @type json
  </parse>
</source>

<match myapp>
  @type elasticsearch
  host elasticsearch.example.com
  port 9200
  index_name myapp
</match>
```

---

### Querying Logs

**Elasticsearch query:**

```json
{
  "query": {
    "bool": {
      "must": [
        {"match": {"level": "ERROR"}},
        {"range": {"timestamp": {"gte": "now-1h"}}}
      ]
    }
  }
}
```

**Result:** All ERROR logs in the last hour.

---

### Log Retention

Logs consume storage. Set retention policies.

```
Hot tier:   Last 7 days (fast SSD, frequently queried)
Warm tier:  8-30 days (slower storage, occasionally queried)
Cold tier:  31-90 days (archive, rarely queried)
Delete:     > 90 days
```

---

## 4. Traces

### What are Distributed Tracing?

Track a request as it flows through multiple services.

```
User request → API Gateway → Service A → Service B → Database
                  ↓              ↓           ↓
               Span 1         Span 2      Span 3
                  ↓              ↓           ↓
              Trace ID       Trace ID    Trace ID
```

**Trace:** End-to-end request flow. **Span:** Single operation within a trace.

---

### Trace Structure

```
Trace ID: abc123

Span 1: API Gateway
  - Start: 12:00:00.000
  - End:   12:00:00.150
  - Duration: 150ms
  - Parent: None

Span 2: Service A
  - Start: 12:00:00.010
  - End:   12:00:00.100
  - Duration: 90ms
  - Parent: Span 1

Span 3: Service B
  - Start: 12:00:00.020
  - End:   12:00:00.080
  - Duration: 60ms
  - Parent: Span 2

Span 4: Database
  - Start: 12:00:00.030
  - End:   12:00:00.070
  - Duration: 40ms
  - Parent: Span 3
```

**Visualization:**

```
API Gateway [====================] 150ms
  Service A [============]         90ms
    Service B [========]           60ms
      Database [====]              40ms
```

---

### OpenTelemetry

Standard for instrumenting applications.

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger import JaegerExporter

# Set up tracer
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Export to Jaeger
jaeger_exporter = JaegerExporter(
    agent_host_name="localhost",
    agent_port=6831,
)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

# Instrument code
@app.route('/api/users')
def get_users():
    with tracer.start_as_current_span("get_users"):
        # Call downstream service
        with tracer.start_as_current_span("call_service_b"):
            response = requests.get("http://service-b/api/data")
        return response.json()
```

---

### Trace Context Propagation

Pass trace ID and span ID across services via HTTP headers.

```
Service A → Service B

Request headers:
  traceparent: 00-abc123-def456-01
               ││  │      │      └─ Flags
               ││  │      └──────── Parent span ID
               ││  └─────────────── Trace ID
               │└────────────────── Version

Service B extracts trace ID and creates child span
```

---

### Jaeger UI

Visualize traces.

```
Features:
  - Search traces by service, operation, duration
  - View trace timeline (spans, durations)
  - Identify slow operations
  - Detect errors
```

---

### Sampling

Tracing every request is expensive. Sample a percentage.

```
Head-based sampling:
  - Decide at the start of the trace (sample 10% of requests)
  - Simple, but may miss rare errors

Tail-based sampling:
  - Decide at the end of the trace (sample all errors, 1% of successes)
  - More accurate, but more complex
```

---

## 5. Correlation

### Correlating Metrics, Logs, and Traces

Use common identifiers to link data.

```
Request ID: req-123
Trace ID:   trace-abc

Metrics:
  http_requests_total{request_id="req-123"}

Logs:
  {"timestamp": "...", "request_id": "req-123", "trace_id": "trace-abc", "message": "Request failed"}

Traces:
  Trace ID: trace-abc
```

**Workflow:**

```
1. Alert fires: p99 latency > 1s
2. Check metrics: Which endpoint is slow?
3. Check logs: Any errors for that endpoint?
4. Check traces: Which service is slow?
5. Drill into slow service logs
```

---

## 6. Observability Tools

### Metrics

```
Prometheus:  Open-source, pull-based, PromQL query language
Grafana:     Visualization, dashboards
Datadog:     Commercial, all-in-one (metrics, logs, traces)
New Relic:   Commercial, APM (Application Performance Monitoring)
```

---

### Logs

```
ELK Stack:   Elasticsearch, Logstash/Fluentd, Kibana
Splunk:      Commercial, powerful search
Datadog:     Commercial, all-in-one
Loki:        Open-source, designed for Kubernetes
```

---

### Traces

```
Jaeger:      Open-source, CNCF project
Zipkin:      Open-source, Twitter origin
Datadog APM: Commercial, all-in-one
New Relic:   Commercial, APM
Lightstep:   Commercial, advanced analysis
```

---

### All-in-One

```
Datadog:     Metrics, logs, traces, APM
New Relic:   Metrics, logs, traces, APM
Dynatrace:   Metrics, logs, traces, APM, AI-powered
```

---

## 7. Common Interview Questions + Answers

### Q: What are the three pillars of observability?

> "The three pillars are metrics, logs, and traces. Metrics are numerical measurements over time like request rate, error rate, and latency. They're good for monitoring trends and alerting. Logs are timestamped records of events like errors and state changes. They're good for debugging specific issues. Traces track a request as it flows through multiple services, showing which service is slow or failing. They're good for understanding distributed systems. Together, they give you a complete picture of system health and behavior."

---

### Q: What's the difference between monitoring and observability?

> "Monitoring is about checking known metrics and alerting on thresholds — it answers 'is the system working?' Observability is about exploring data to understand unexpected behavior — it answers 'why is the system behaving this way?' Monitoring is reactive (alert fires, you investigate). Observability is proactive (you can ask arbitrary questions about the system). Monitoring works for known issues. Observability helps you debug unknown issues."

---

### Q: How do you correlate metrics, logs, and traces?

> "Use common identifiers like request ID and trace ID. When a request comes in, generate a request ID and trace ID. Include them in metrics labels, log fields, and trace context. When an alert fires, I check metrics to identify the problem area, then search logs for that request ID to see errors, then view the trace to see which service is slow. This lets me quickly drill down from high-level metrics to specific request details."

---

### Q: How do you handle high-cardinality metrics?

> "High-cardinality labels like user ID or request ID create too many unique metric series, overwhelming the metrics system. I avoid them by using low-cardinality labels like endpoint, method, and status. For high-cardinality data, I use logs or traces instead. For example, I don't create a metric per user, but I log user IDs and query logs to analyze per-user behavior. I also use sampling for traces to reduce cardinality."

---

## 8. Interview Tricks & Pitfalls

### ✅ Trick 1: Know the three pillars

Metrics, logs, and traces. Be able to explain each and when to use them.

---

### ✅ Trick 2: Mention correlation

Show you understand how to link metrics, logs, and traces using common identifiers.

---

### ✅ Trick 3: Talk about sampling

Tracing every request is expensive. Mention sampling strategies (head-based, tail-based).

---

### ❌ Pitfall 1: Confusing monitoring and observability

Monitoring is checking known metrics. Observability is exploring data to understand unknown issues.

---

### ❌ Pitfall 2: High-cardinality metrics

Don't use high-cardinality labels (user ID, request ID) in metrics. Use logs or traces instead.

---

### ❌ Pitfall 3: Ignoring structured logging

Plain text logs are hard to parse. Always use structured logging (JSON) for centralized logging.

---

## 9. Quick Reference

```
Observability: Understand system state by examining external outputs

Three Pillars:
  1. Metrics:  Numerical measurements (CPU, latency, error rate)
  2. Logs:     Timestamped events (errors, requests)
  3. Traces:   Request flow across services

Metrics:
  Types:       Counter, Gauge, Histogram, Summary
  Labels:      Add context (method, status, endpoint)
  Tools:       Prometheus, Grafana, Datadog
  Query:       PromQL (rate, sum, histogram_quantile)

Logs:
  Format:      Structured (JSON) > plain text
  Levels:      TRACE, DEBUG, INFO, WARN, ERROR, FATAL
  Tools:       ELK Stack, Splunk, Datadog, Loki
  Retention:   Hot (7d), Warm (30d), Cold (90d), Delete

Traces:
  Structure:   Trace (end-to-end) → Spans (operations)
  Context:     Propagate trace ID via HTTP headers (traceparent)
  Tools:       Jaeger, Zipkin, Datadog APM, New Relic
  Sampling:    Head-based (sample at start), Tail-based (sample at end)

Correlation:
  Use common identifiers (request ID, trace ID)
  Link metrics, logs, traces for debugging

Monitoring vs Observability:
  Monitoring:     "Is it working?" (known unknowns)
  Observability:  "Why is it behaving this way?" (unknown unknowns)

Best Practices:
  ✅ Use structured logging (JSON)
  ✅ Low-cardinality metric labels
  ✅ Correlate with request ID / trace ID
  ✅ Sample traces (don't trace everything)
  ✅ Set log retention policies
  ✅ Use percentiles for latency (p50, p95, p99)
  ✅ Alert on metrics, debug with logs and traces

Tools:
  Metrics:     Prometheus, Grafana, Datadog
  Logs:        ELK Stack, Splunk, Loki
  Traces:      Jaeger, Zipkin, Datadog APM
  All-in-One:  Datadog, New Relic, Dynatrace
```
