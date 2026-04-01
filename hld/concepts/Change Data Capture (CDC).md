## 1. What is Change Data Capture (CDC)?

Change Data Capture is a pattern for tracking and streaming database changes (inserts, updates, deletes) to downstream systems in real-time.

```
Traditional (Polling):
  Every 5 minutes:
    SELECT * FROM orders WHERE updated_at > last_check
  
  Problems:
    - Polling overhead
    - Missed deletes
    - Stale data (up to 5 minutes)

CDC (Streaming):
  Database → CDC → Event Stream → Consumers
  
  Benefits:
    - Real-time (milliseconds)
    - Captures all changes (including deletes)
    - No polling overhead
```

**Key principle:** Stream database changes as events without polling.

---

## 2. Core Concepts

### Change Event

A record of a database change.

```
Insert event:
  {
    "operation": "INSERT",
    "table": "orders",
    "before": null,
    "after": {
      "id": 123,
      "user_id": 456,
      "total": 99.99,
      "status": "pending"
    },
    "timestamp": "2024-01-01T10:00:00Z"
  }

Update event:
  {
    "operation": "UPDATE",
    "table": "orders",
    "before": {
      "id": 123,
      "status": "pending"
    },
    "after": {
      "id": 123,
      "status": "confirmed"
    },
    "timestamp": "2024-01-01T10:05:00Z"
  }

Delete event:
  {
    "operation": "DELETE",
    "table": "orders",
    "before": {
      "id": 123,
      "user_id": 456,
      "total": 99.99,
      "status": "confirmed"
    },
    "after": null,
    "timestamp": "2024-01-01T10:10:00Z"
  }
```

### Write-Ahead Log (WAL)

Database log of all changes before they're applied.

```
PostgreSQL WAL:
  - Every change written to WAL first
  - Then applied to database
  - WAL used for crash recovery
  - CDC reads WAL to capture changes

MySQL Binlog:
  - Binary log of all changes
  - Used for replication
  - CDC reads binlog
```

### CDC Connector

Tool that reads database logs and publishes events.

```
Debezium (most popular):
  - Reads WAL/binlog
  - Publishes to Kafka
  - Supports PostgreSQL, MySQL, MongoDB, etc.

AWS DMS (Database Migration Service):
  - Managed CDC service
  - Publishes to Kinesis, S3, etc.
```

---

## 3. How It Works

### Log-Based CDC (Recommended)

```
1. Database writes change to WAL/binlog
2. CDC connector reads WAL/binlog
3. Connector publishes event to Kafka
4. Consumers process events

Example (PostgreSQL):
  1. INSERT INTO orders (id, total) VALUES (123, 99.99)
  2. PostgreSQL writes to WAL
  3. Debezium reads WAL
  4. Debezium publishes to Kafka topic "orders"
  5. Consumers receive event

Benefits:
  - Real-time (milliseconds)
  - No database overhead (just read logs)
  - Captures all changes (including deletes)
  - No schema changes needed
```

### Trigger-Based CDC

```
1. Create database trigger on table
2. Trigger writes change to audit table
3. CDC reads audit table
4. CDC publishes event

Example (PostgreSQL):
  CREATE TRIGGER orders_cdc
  AFTER INSERT OR UPDATE OR DELETE ON orders
  FOR EACH ROW EXECUTE FUNCTION log_change();

Problems:
  - Database overhead (trigger execution)
  - Schema changes needed
  - Can impact performance
```

### Query-Based CDC (Polling)

```
1. Periodically query for changes
2. Track last processed timestamp
3. Publish changes as events

Example:
  SELECT * FROM orders
  WHERE updated_at > '2024-01-01T10:00:00Z'

Problems:
  - Polling overhead
  - Missed deletes (no updated_at)
  - Stale data (polling interval)
```

---

## 4. Use Cases

### Data Synchronization

```
Sync data between databases:
  PostgreSQL (OLTP) → CDC → Redshift (OLAP)
  
  Real-time data warehouse updates
  No batch ETL needed

Example:
  1. Order placed in PostgreSQL
  2. CDC captures change
  3. Event published to Kafka
  4. Consumer updates Redshift
```

### Cache Invalidation

```
Invalidate cache when data changes:
  Database → CDC → Cache invalidation
  
  Keep cache in sync with database

Example:
  1. User profile updated in database
  2. CDC captures change
  3. Event published
  4. Consumer invalidates Redis cache
```

### Event-Driven Architecture

```
Database changes trigger business logic:
  Database → CDC → Event bus → Services
  
  Decouple services from database

Example:
  1. Order status updated to "shipped"
  2. CDC captures change
  3. Event published
  4. Email service sends notification
  5. Analytics service updates metrics
```

### Search Index Synchronization

```
Keep search index in sync:
  Database → CDC → Elasticsearch
  
  Real-time search updates

Example:
  1. Product added to database
  2. CDC captures change
  3. Event published
  4. Consumer indexes in Elasticsearch
```

---

## 5. Implementation Example (Debezium + Kafka)

### Setup Debezium Connector

```json
{
  "name": "orders-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "localhost",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "password",
    "database.dbname": "mydb",
    "database.server.name": "myserver",
    "table.include.list": "public.orders",
    "plugin.name": "pgoutput"
  }
}
```

### Enable WAL in PostgreSQL

```sql
-- postgresql.conf
wal_level = logical
max_replication_slots = 4
max_wal_senders = 4

-- Create replication slot
SELECT pg_create_logical_replication_slot('debezium', 'pgoutput');

-- Grant permissions
GRANT SELECT ON ALL TABLES IN SCHEMA public TO debezium_user;
GRANT REPLICATION SLAVE ON *.* TO debezium_user;
```

### Consume Events (Python)

```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'myserver.public.orders',
    bootstrap_servers=['localhost:9092'],
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for message in consumer:
    event = message.value
    
    operation = event['payload']['op']  # c=create, u=update, d=delete
    
    if operation == 'c':  # Insert
        after = event['payload']['after']
        print(f"Order created: {after['id']}")
        
    elif operation == 'u':  # Update
        before = event['payload']['before']
        after = event['payload']['after']
        print(f"Order updated: {before} → {after}")
        
    elif operation == 'd':  # Delete
        before = event['payload']['before']
        print(f"Order deleted: {before['id']}")
```

### Update Cache on Change

```python
import redis

redis_client = redis.Redis(host='localhost', port=6379)

for message in consumer:
    event = message.value
    operation = event['payload']['op']
    
    if operation == 'c' or operation == 'u':
        # Update cache
        after = event['payload']['after']
        redis_client.set(f"order:{after['id']}", json.dumps(after))
        
    elif operation == 'd':
        # Invalidate cache
        before = event['payload']['before']
        redis_client.delete(f"order:{before['id']}")
```

---

## 6. Event Format (Debezium)

### Full Event Structure

```json
{
  "schema": {...},
  "payload": {
    "before": {
      "id": 123,
      "status": "pending"
    },
    "after": {
      "id": 123,
      "status": "confirmed"
    },
    "source": {
      "version": "1.9.0",
      "connector": "postgresql",
      "name": "myserver",
      "ts_ms": 1609459200000,
      "snapshot": "false",
      "db": "mydb",
      "schema": "public",
      "table": "orders",
      "txId": 12345,
      "lsn": 67890
    },
    "op": "u",  // c=create, u=update, d=delete, r=read (snapshot)
    "ts_ms": 1609459200000
  }
}
```

### Operation Types

```
c (create): INSERT
  before: null
  after: new row

u (update): UPDATE
  before: old row
  after: new row

d (delete): DELETE
  before: old row
  after: null

r (read): Initial snapshot
  before: null
  after: existing row
```

---

## 7. Challenges

### Schema evolution

```
Problem:
  Database schema changes (add/remove column)
  Consumers may break

Solutions:
  - Schema registry (Confluent Schema Registry)
  - Backward-compatible changes
  - Version events
  - Consumer handles missing fields

Example:
  Old schema: {id, name}
  New schema: {id, name, email}
  
  Consumer:
    email = event.get('email', None)  # Default to None
```

### Event ordering

```
Problem:
  Events may arrive out of order
  
Solutions:
  - Kafka partitioning (same key → same partition → ordered)
  - Event versioning (timestamp, sequence number)
  - Idempotent consumers

Example:
  Partition by order_id:
    All events for order 123 → partition 0 → ordered
```

### Initial snapshot

```
Problem:
  CDC starts, need to capture existing data
  
Solution:
  Debezium takes initial snapshot:
    1. Lock table (briefly)
    2. Read all rows
    3. Publish as "read" events
    4. Start streaming changes
    
  Can be slow for large tables
```

### Backpressure

```
Problem:
  Database changes faster than consumers can process
  
Solutions:
  - Scale consumers (more instances)
  - Batch processing
  - Increase Kafka partitions
  - Monitor lag
```

---

## 8. CDC Tools

### Debezium

```
Open-source, most popular
Supports PostgreSQL, MySQL, MongoDB, SQL Server, Oracle
Publishes to Kafka

Pros:
  ✅ Open-source
  ✅ Wide database support
  ✅ Active community
  
Cons:
  ❌ Requires Kafka
  ❌ Self-hosted (ops overhead)
```

### AWS DMS (Database Migration Service)

```
Managed CDC service
Publishes to Kinesis, S3, Redshift, etc.

Pros:
  ✅ Managed (no ops)
  ✅ Integrates with AWS
  
Cons:
  ❌ AWS-only
  ❌ Expensive
  ❌ Less flexible
```

### Maxwell's Daemon

```
Lightweight CDC for MySQL
Publishes to Kafka, Kinesis, etc.

Pros:
  ✅ Lightweight
  ✅ Easy to set up
  
Cons:
  ❌ MySQL only
  ❌ Less features than Debezium
```

### Airbyte

```
Open-source data integration
Supports CDC for many databases
Publishes to various destinations

Pros:
  ✅ Many connectors
  ✅ Easy to use
  
Cons:
  ❌ Less mature than Debezium
```

---

## 9. When to Use CDC

### Good fit

```
✅ Real-time data synchronization
✅ Cache invalidation
✅ Event-driven architecture
✅ Search index updates
✅ Data warehouse updates

Examples:
  - Sync OLTP to OLAP (real-time analytics)
  - Invalidate cache on database change
  - Trigger business logic on data change
  - Update Elasticsearch on database change
```

### Poor fit

```
❌ Batch processing (use ETL)
❌ Simple use cases (use polling)
❌ No real-time requirement

Examples:
  - Daily reports (use batch ETL)
  - Low-frequency updates (use polling)
```

---

## 10. Common Interview Questions + Answers

### Q: What is Change Data Capture and why would you use it?

> "CDC is a pattern for streaming database changes to downstream systems in real-time by reading the database's write-ahead log. Instead of polling the database for changes, CDC captures every insert, update, and delete as it happens and publishes it as an event. You'd use it for real-time data synchronization like keeping a data warehouse up-to-date, cache invalidation when data changes, or building event-driven architectures where database changes trigger business logic."

### Q: How does log-based CDC work?

> "Log-based CDC reads the database's write-ahead log or binary log. When you write to the database, it first writes to the log for durability, then applies the change. A CDC connector like Debezium reads this log, converts changes into events, and publishes them to Kafka. This approach has minimal database overhead since you're just reading logs, captures all changes including deletes, and provides real-time updates with millisecond latency."

### Q: What are the challenges with CDC?

> "First, schema evolution — when the database schema changes, consumers may break, so you need a schema registry and backward-compatible changes. Second, event ordering — events can arrive out of order, so use Kafka partitioning with consistent keys. Third, initial snapshot — capturing existing data can be slow for large tables. Fourth, backpressure — if the database changes faster than consumers can process, you need to scale consumers or batch processing. Finally, operational complexity — running Debezium and Kafka requires ops expertise."

### Q: When would you use CDC vs polling?

> "Use CDC when you need real-time updates, high throughput, or need to capture deletes. CDC has minimal database overhead and provides millisecond latency. Use polling for simple use cases with low frequency updates, when you don't need real-time data, or when you can't access database logs. Polling is simpler to implement but has overhead, can miss deletes, and has stale data between polls. For most production systems with real-time requirements, CDC is the better choice."

---

## 11. Quick Reference

```
What is CDC?
  Stream database changes (inserts, updates, deletes)
  Real-time, no polling
  Reads write-ahead log (WAL)

How it works:
  Database → WAL → CDC connector → Kafka → Consumers

Change event:
  {operation, table, before, after, timestamp}
  Operations: INSERT, UPDATE, DELETE

CDC approaches:
  - Log-based (recommended): Read WAL/binlog
  - Trigger-based: Database triggers
  - Query-based: Polling (not true CDC)

Use cases:
  - Data synchronization (OLTP → OLAP)
  - Cache invalidation
  - Event-driven architecture
  - Search index updates

Popular tools:
  - Debezium (open-source, Kafka)
  - AWS DMS (managed, AWS)
  - Maxwell's Daemon (MySQL)
  - Airbyte (data integration)

Event format (Debezium):
  op: c (create), u (update), d (delete), r (read)
  before: old row
  after: new row
  source: metadata

Challenges:
  - Schema evolution (use schema registry)
  - Event ordering (use partitioning)
  - Initial snapshot (can be slow)
  - Backpressure (scale consumers)

When to use:
  ✅ Real-time synchronization
  ✅ Cache invalidation
  ✅ Event-driven architecture
  ✅ High throughput
  
  ❌ Batch processing
  ❌ Low frequency updates
  ❌ No real-time requirement

Best practices:
  - Use log-based CDC (not triggers)
  - Partition by entity ID
  - Use schema registry
  - Make consumers idempotent
  - Monitor lag
  - Handle schema evolution
```
