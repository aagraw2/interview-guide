## 1. What is Stream Processing?

Stream processing is the real-time processing of continuous data streams as events arrive, rather than batch processing stored data.

```
Batch processing:
  Collect data for 24 hours → Process all at once → Results
  Latency: 24 hours

Stream processing:
  Event arrives → Process immediately → Results
  Latency: Milliseconds to seconds
```

**Use cases:** Real-time analytics, fraud detection, monitoring, ETL, event-driven applications.

---

## 2. Stream Processing vs Batch Processing

|Feature|Stream Processing|Batch Processing|
|---|---|---|
|Data|Unbounded (continuous)|Bounded (finite dataset)|
|Latency|Milliseconds to seconds|Minutes to hours|
|Processing|Event-by-event|All data at once|
|State|Stateful (windowing)|Stateless or full state|
|Use case|Real-time analytics|Historical analysis|
|Examples|Kafka Streams, Flink|Spark Batch, MapReduce|

---

## 3. Core Concepts

### Event time vs Processing time

```
Event time: When event actually occurred
Processing time: When event is processed

Example:
  Event: User clicked at 10:00:00 (event time)
  Network delay: 5 seconds
  Processed at 10:00:05 (processing time)

Use event time for accurate analytics
```

### Windowing

Group events into finite chunks for aggregation.

```
Tumbling window (non-overlapping):
  [00:00-00:05], [00:05-00:10], [00:10-00:15]
  Each event in exactly one window

Sliding window (overlapping):
  [00:00-00:05], [00:01-00:06], [00:02-00:07]
  Each event in multiple windows

Session window (activity-based):
  Group events by user session
  Window closes after inactivity timeout
```

### Watermarks

Track progress of event time to handle late events.

```
Watermark: "All events before time T have been seen"

Example:
  Watermark at 10:05:00
  → Can close windows ending before 10:05:00
  → Late events after 10:05:00 are dropped or handled separately
```

---

## 4. Kafka Streams

### Simple example

```java
StreamsBuilder builder = new StreamsBuilder();

// Input stream
KStream<String, String> orders = builder.stream("orders");

// Transform
KStream<String, Long> orderCounts = orders
    .groupByKey()
    .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
    .count()
    .toStream();

// Output stream
orderCounts.to("order-counts");

KafkaStreams streams = new KafkaStreams(builder.build(), config);
streams.start();
```

### Stateful processing

```java
// Count orders per user
KTable<String, Long> userOrderCounts = orders
    .groupBy((key, value) -> value.getUserId())
    .count();

// Join streams
KStream<String, Order> orders = builder.stream("orders");
KTable<String, User> users = builder.table("users");

KStream<String, EnrichedOrder> enriched = orders
    .join(users,
        (order, user) -> new EnrichedOrder(order, user));
```

---

## 5. Apache Flink

### DataStream API

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// Source
DataStream<Event> events = env
    .addSource(new FlinkKafkaConsumer<>("events", schema, properties));

// Transform
DataStream<Result> results = events
    .keyBy(Event::getUserId)
    .window(TumblingEventTimeWindows.of(Time.minutes(5)))
    .aggregate(new CountAggregator());

// Sink
results.addSink(new FlinkKafkaProducer<>("results", schema, properties));

env.execute("Stream Processing Job");
```

### Exactly-once semantics

```
Flink provides exactly-once processing:
  1. Checkpointing: Periodic snapshots of state
  2. Two-phase commit: Transactional sinks
  3. Idempotent writes: Safe to replay

On failure:
  → Restore from last checkpoint
  → Replay events from checkpoint
  → No data loss, no duplicates
```

---

## 6. Windowing Examples

### Tumbling window

```
Count events per 5-minute window:

Events:
  10:00:01 → Window 1
  10:00:30 → Window 1
  10:04:59 → Window 1
  10:05:00 → Window 2
  10:05:30 → Window 2

Window 1 [10:00-10:05]: 3 events
Window 2 [10:05-10:10]: 2 events
```

### Sliding window

```
Count events in 5-minute window, sliding every 1 minute:

Events:
  10:00:01 → Windows [10:00-10:05]
  10:00:30 → Windows [10:00-10:05]
  10:01:00 → Windows [10:00-10:05], [10:01-10:06]
  10:05:00 → Windows [10:01-10:06], [10:02-10:07], ..., [10:05-10:10]

Window [10:00-10:05]: 3 events
Window [10:01-10:06]: 2 events
```

### Session window

```
Group events by user session (timeout: 30 minutes):

User A events:
  10:00:00 → Session 1
  10:10:00 → Session 1 (within 30 min)
  10:20:00 → Session 1 (within 30 min)
  11:00:00 → Session 2 (> 30 min gap)

Session 1: 3 events (10:00-10:20)
Session 2: 1 event (11:00)
```

---

## 7. State Management

### Local state

```
Each worker maintains local state:
  - In-memory (fast)
  - Backed by RocksDB (persistent)
  - Checkpointed periodically

Example: Count per user
  State: {user_id: count}
  user_123: 5
  user_456: 3
```

### State backends

```
Flink state backends:
  1. MemoryStateBackend: In-memory (fast, limited size)
  2. FsStateBackend: File system (larger, slower)
  3. RocksDBStateBackend: Embedded DB (largest, disk-based)

Choose based on state size and performance needs
```

### Checkpointing

```
Periodic snapshots of state:
  1. Pause processing
  2. Save state to durable storage (HDFS, S3)
  3. Resume processing

On failure:
  → Restore from last checkpoint
  → Replay events from checkpoint offset

Checkpoint interval: 1-10 minutes
```

---

## 8. Handling Late Events

### Allowed lateness

```
Window: [10:00-10:05]
Watermark: 10:05:00
Allowed lateness: 1 minute

Event arrives at 10:05:30 with event time 10:04:00:
  → Within allowed lateness
  → Update window result
  → Emit updated result

Event arrives at 10:06:30 with event time 10:04:00:
  → Beyond allowed lateness
  → Drop or send to side output
```

### Side outputs

```
Late events sent to separate stream:

Main output: On-time events
Side output: Late events

Process late events separately:
  - Log for monitoring
  - Store in database
  - Trigger manual reconciliation
```

---

## 9. Common Patterns

### Filtering

```
Filter high-value orders:

orders
  .filter(order -> order.getAmount() > 1000)
  .to("high-value-orders");
```

### Aggregation

```
Sum order amounts per user:

orders
  .groupBy(Order::getUserId)
  .window(TumblingEventTimeWindows.of(Time.hours(1)))
  .sum("amount");
```

### Join

```
Enrich orders with user data:

orders
  .join(users)
  .where(Order::getUserId)
  .equalTo(User::getId)
  .window(TumblingEventTimeWindows.of(Time.minutes(5)))
  .apply((order, user) -> new EnrichedOrder(order, user));
```

### Deduplication

```
Remove duplicate events:

events
  .keyBy(Event::getId)
  .window(TumblingEventTimeWindows.of(Time.minutes(5)))
  .reduce((e1, e2) -> e1);  // Keep first
```

---

## 10. Common Interview Questions + Answers

### Q: What's the difference between stream processing and batch processing?

> "Stream processing handles unbounded continuous data in real-time with millisecond to second latency, processing events as they arrive. Batch processing handles bounded finite datasets with minute to hour latency, processing all data at once. Use stream processing for real-time analytics, fraud detection, and monitoring where low latency matters. Use batch processing for historical analysis, data warehousing, and complex transformations where latency is acceptable. Modern systems like Spark and Flink support both."

### Q: What are the different types of windows in stream processing?

> "Tumbling windows are non-overlapping fixed-size windows like [00:00-00:05], [00:05-00:10]. Each event belongs to exactly one window. Sliding windows overlap, like [00:00-00:05], [00:01-00:06], useful for moving averages. Session windows group events by activity with an inactivity timeout, useful for user sessions. Choose tumbling for periodic aggregations, sliding for continuous metrics, and session for user behavior analysis."

### Q: How does Flink achieve exactly-once semantics?

> "Flink uses distributed snapshots (checkpointing) to periodically save the state of all operators and Kafka offsets. On failure, it restores from the last checkpoint and replays events from that point. For sinks, it uses two-phase commit to ensure transactional writes. The combination of checkpointing, replay, and transactional sinks guarantees each event is processed exactly once with no data loss or duplicates. This works even with failures and restarts."

### Q: How do you handle late-arriving events in stream processing?

> "Use watermarks to track event time progress and define allowed lateness. Events within the allowed lateness window update the result and emit a correction. Events beyond allowed lateness are either dropped or sent to a side output for separate handling. For example, with a 5-minute window and 1-minute allowed lateness, events up to 1 minute late update the window, but events more than 1 minute late go to side output for logging or manual reconciliation."

---

## 11. Quick Reference

```
What is stream processing?
  Real-time processing of continuous data streams
  Process events as they arrive
  Latency: Milliseconds to seconds

Stream vs Batch:
  Stream: Unbounded, real-time, event-by-event
  Batch: Bounded, delayed, all-at-once

Core concepts:
  Event time: When event occurred
  Processing time: When event processed
  Windowing: Group events into finite chunks
  Watermarks: Track event time progress

Window types:
  Tumbling: Non-overlapping [00:00-00:05], [00:05-00:10]
  Sliding: Overlapping [00:00-00:05], [00:01-00:06]
  Session: Activity-based (inactivity timeout)

Kafka Streams:
  Java library for stream processing
  Stateful processing (count, join, aggregate)
  Exactly-once semantics

Apache Flink:
  Distributed stream processing engine
  DataStream API
  Exactly-once with checkpointing
  Low latency, high throughput

State management:
  Local state (in-memory or RocksDB)
  Checkpointing (periodic snapshots)
  Restore on failure

Late events:
  Allowed lateness: Update window within threshold
  Side outputs: Separate stream for late events
  Drop or log events beyond threshold

Common patterns:
  Filtering: Select events matching criteria
  Aggregation: Sum, count, average per window
  Join: Combine streams by key
  Deduplication: Remove duplicate events

Exactly-once:
  Checkpointing: Save state periodically
  Two-phase commit: Transactional sinks
  Replay: Restore and replay from checkpoint

Best practices:
  - Use event time for accuracy
  - Set appropriate watermarks
  - Handle late events gracefully
  - Choose right window type
  - Monitor lag and throughput
  - Test failure scenarios
```
