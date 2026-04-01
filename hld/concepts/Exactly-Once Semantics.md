## 1. What is Exactly-Once Semantics?

Exactly-once semantics ensures that each message is processed exactly one time, even in the presence of failures — no duplicates, no data loss.

```
At-most-once:
  Send message → May be lost
  No duplicates, but data loss possible

At-least-once:
  Send message → Retry on failure
  No data loss, but duplicates possible

Exactly-once:
  Send message → Processed exactly once
  No data loss, no duplicates
  Hardest to achieve
```

**Key principle:** Guarantee each message is processed once and only once.

---

## 2. Core Concepts

### Delivery Semantics

```
At-most-once:
  Producer sends message
  No retry on failure
  Message may be lost
  
  Use case: Metrics, logs (loss acceptable)

At-least-once:
  Producer sends message
  Retry on failure
  Message may be duplicated
  
  Use case: Most systems (with idempotent consumers)

Exactly-once:
  Producer sends message
  Idempotent + transactional
  Message processed exactly once
  
  Use case: Financial transactions, critical data
```

### Idempotency

Processing the same message multiple times has the same effect as processing it once.

```
Non-idempotent:
  balance += 100
  Process twice: balance += 200 (wrong!)

Idempotent:
  if not processed(transaction_id):
      balance += 100
      mark_processed(transaction_id)
  
  Process twice: balance += 100 (correct!)
```

### Transactional Processing

Combine message processing and state update in a transaction.

```
BEGIN TRANSACTION
  Process message
  Update database
  Commit offset
COMMIT

If crash: Transaction rolls back, reprocess
If success: All committed, no duplicate
```

---

## 3. Achieving Exactly-Once

### Idempotent Producer (Kafka)

```
Producer retries don't create duplicates:

Without idempotence:
  Send message → Network error → Retry → Duplicate

With idempotence:
  Producer ID + Sequence number
  Broker deduplicates based on (producer_id, sequence)
  Retry → Broker detects duplicate → Ignores

Enable:
  producer = KafkaProducer(
      enable_idempotence=True
  )
```

### Transactional Producer (Kafka)

```
Atomic writes across partitions:

producer = KafkaProducer(
    transactional_id='my-transactional-id',
    enable_idempotence=True
)

producer.init_transactions()
producer.begin_transaction()
try:
    producer.send('topic1', value='msg1')
    producer.send('topic2', value='msg2')
    producer.commit_transaction()
except:
    producer.abort_transaction()

Both messages committed or neither
```

### Transactional Consumer (Kafka)

```
Read-process-write in transaction:

consumer = KafkaConsumer(
    isolation_level='read_committed'  # Only read committed messages
)

producer = KafkaProducer(
    transactional_id='my-transactional-id'
)

for message in consumer:
    producer.begin_transaction()
    try:
        # Process message
        result = process(message.value)
        
        # Write result
        producer.send('output-topic', value=result)
        
        # Commit offset
        producer.send_offsets_to_transaction(
            {TopicPartition(message.topic, message.partition): message.offset + 1},
            consumer.group_id
        )
        
        producer.commit_transaction()
    except:
        producer.abort_transaction()
```

---

## 4. Implementation Examples

### Idempotent Consumer (Database)

```python
import psycopg2
from kafka import KafkaConsumer

consumer = KafkaConsumer('orders', group_id='order-processor')
conn = psycopg2.connect("postgresql://localhost/mydb")

def process_order(order_id, amount):
    cursor = conn.cursor()
    
    # Check if already processed (idempotency)
    cursor.execute("""
        SELECT 1 FROM processed_orders WHERE order_id = %s
    """, [order_id])
    
    if cursor.fetchone():
        print(f"Order {order_id} already processed, skipping")
        return
    
    # Process order
    cursor.execute("""
        BEGIN;
        UPDATE accounts SET balance = balance + %s WHERE id = 1;
        INSERT INTO processed_orders (order_id, processed_at) VALUES (%s, NOW());
        COMMIT;
    """, [amount, order_id])
    
    conn.commit()

for message in consumer:
    order = json.loads(message.value)
    process_order(order['id'], order['amount'])
    consumer.commit()
```

### Exactly-Once with Database Transaction

```python
from kafka import KafkaConsumer
import psycopg2

consumer = KafkaConsumer(
    'orders',
    group_id='order-processor',
    enable_auto_commit=False
)

conn = psycopg2.connect("postgresql://localhost/mydb")

for message in consumer:
    order = json.loads(message.value)
    
    cursor = conn.cursor()
    try:
        # Begin transaction
        cursor.execute("BEGIN")
        
        # Process order
        cursor.execute("""
            UPDATE accounts SET balance = balance + %s WHERE id = %s
        """, [order['amount'], order['account_id']])
        
        # Store offset in same transaction
        cursor.execute("""
            INSERT INTO kafka_offsets (topic, partition, offset, group_id)
            VALUES (%s, %s, %s, %s)
            ON CONFLICT (topic, partition, group_id)
            DO UPDATE SET offset = %s
        """, [
            message.topic,
            message.partition,
            message.offset,
            'order-processor',
            message.offset
        ])
        
        # Commit transaction
        cursor.execute("COMMIT")
        
        # Commit Kafka offset
        consumer.commit()
        
    except Exception as e:
        cursor.execute("ROLLBACK")
        print(f"Error: {e}")
        # Don't commit offset, will reprocess
```

---

## 5. Exactly-Once in Different Systems

### Kafka Streams

```java
Properties props = new Properties();
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, 
          StreamsConfig.EXACTLY_ONCE_V2);

StreamsBuilder builder = new StreamsBuilder();
KStream<String, String> orders = builder.stream("orders");

orders
    .mapValues(order -> processOrder(order))
    .to("processed-orders");

KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();

// Exactly-once processing guaranteed
```

### Apache Flink

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// Enable checkpointing for exactly-once
env.enableCheckpointing(60000);
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

DataStream<Order> orders = env.addSource(new FlinkKafkaConsumer<>(...));

orders
    .map(order -> processOrder(order))
    .addSink(new FlinkKafkaProducer<>(...));

env.execute();

// Exactly-once via checkpointing + two-phase commit
```

### AWS Lambda + SQS

```python
def lambda_handler(event, context):
    for record in event['Records']:
        message_id = record['messageId']
        
        # Check if already processed (idempotency)
        if is_processed(message_id):
            continue
        
        # Process message
        process_message(record['body'])
        
        # Mark as processed
        mark_processed(message_id)
    
    # SQS automatically deletes messages after successful return
    return {'statusCode': 200}

# Idempotency ensures exactly-once even with retries
```

---

## 6. Challenges

### Performance overhead

```
Problem:
  Exactly-once requires transactions
  Transactions are slower than at-least-once

Impact:
  - Lower throughput
  - Higher latency
  - More resource usage

Trade-off:
  Use exactly-once only when necessary
  Use at-least-once + idempotent consumers otherwise
```

### Complexity

```
Problem:
  Exactly-once is complex to implement
  Requires transactional support
  Harder to debug

Solutions:
  - Use framework support (Kafka Streams, Flink)
  - Start with at-least-once + idempotency
  - Upgrade to exactly-once only if needed
```

### Cross-system transactions

```
Problem:
  Exactly-once across different systems (Kafka + database)
  Requires distributed transactions (2PC)

Solutions:
  - Store offsets in same database (single transaction)
  - Use idempotency keys
  - Accept at-least-once + idempotent consumers
```

---

## 7. Exactly-Once vs Idempotency

### Exactly-Once

```
Guarantee: Message processed exactly once
Mechanism: Transactions, deduplication
Cost: High (performance, complexity)

Use when:
  - Financial transactions
  - Critical data
  - Cannot tolerate duplicates
```

### At-Least-Once + Idempotency

```
Guarantee: Message processed at least once
Mechanism: Retries + idempotent consumers
Cost: Low (simple, fast)

Use when:
  - Most systems
  - Can make consumers idempotent
  - Performance matters

Example:
  Update user profile (idempotent)
  Increment counter (use idempotency key)
```

---

## 8. When to Use Exactly-Once

### Good fit

```
✅ Financial transactions (payments, transfers)
✅ Inventory management (stock updates)
✅ Billing systems (charge once)
✅ Critical data (no duplicates acceptable)

Examples:
  - Payment processing
  - Order fulfillment
  - Account balance updates
```

### Poor fit

```
❌ Metrics and logs (duplicates acceptable)
❌ Idempotent operations (can use at-least-once)
❌ Performance-critical (overhead too high)

Examples:
  - Page view tracking
  - Log aggregation
  - Cache updates (idempotent)
```

---

## 9. Best Practices

### Start with at-least-once

```
1. Implement at-least-once delivery
2. Make consumers idempotent
3. Test with duplicates
4. Upgrade to exactly-once only if needed

Most systems don't need exactly-once
At-least-once + idempotency is sufficient
```

### Use idempotency keys

```
Include unique ID in each message:
  {
    "idempotency_key": "order_123_attempt_1",
    "order_id": 123,
    "amount": 99.99
  }

Consumer checks key before processing:
  if not processed(idempotency_key):
      process_message()
      mark_processed(idempotency_key)
```

### Store offsets with data

```
Store Kafka offsets in same database as data:

BEGIN TRANSACTION
  Process message
  Update data
  Update offset
COMMIT

Single transaction ensures exactly-once
```

---

## 10. Common Interview Questions + Answers

### Q: What is exactly-once semantics and why is it hard to achieve?

> "Exactly-once means each message is processed exactly one time — no duplicates, no data loss. It's hard because distributed systems have failures. If you send a message and don't get an acknowledgment, you don't know if it was received. If you retry, you might create a duplicate. If you don't retry, you might lose data. Achieving exactly-once requires idempotent producers to prevent duplicate sends, transactional consumers to atomically process and commit offsets, and often distributed transactions across multiple systems."

### Q: What's the difference between exactly-once and at-least-once with idempotent consumers?

> "Exactly-once uses transactions to guarantee no duplicates at the infrastructure level. At-least-once with idempotent consumers allows duplicates but makes processing idempotent so duplicates have no effect. For example, with exactly-once, Kafka ensures each message is delivered once. With at-least-once plus idempotency, Kafka may deliver duplicates, but your consumer checks an idempotency key and skips already-processed messages. The latter is simpler and faster, and sufficient for most use cases."

### Q: How does Kafka achieve exactly-once semantics?

> "Kafka uses three mechanisms: First, idempotent producers with producer IDs and sequence numbers prevent duplicate sends. Second, transactional producers allow atomic writes across multiple partitions. Third, transactional consumers read only committed messages and can atomically process a message, write output, and commit the offset in a single transaction. This ensures that even with retries and failures, each message is processed exactly once. However, it requires enabling transactions and has performance overhead."

### Q: When should you use exactly-once vs at-least-once?

> "Use exactly-once for financial transactions, inventory updates, or billing where duplicates cause real problems. Use at-least-once with idempotent consumers for everything else — it's simpler, faster, and sufficient when you can make operations idempotent. For example, updating a user profile is naturally idempotent, so at-least-once is fine. Charging a credit card is not idempotent, so you need exactly-once or idempotency keys. Most systems should start with at-least-once and only upgrade to exactly-once if duplicates cause issues."

---

## 11. Quick Reference

```
What is Exactly-Once?
  Each message processed exactly one time
  No duplicates, no data loss
  Hardest delivery semantic to achieve

Delivery semantics:
  - At-most-once: May lose messages, no duplicates
  - At-least-once: No loss, may duplicate
  - Exactly-once: No loss, no duplicates

Achieving exactly-once:
  1. Idempotent producer (prevent duplicate sends)
  2. Transactional producer (atomic writes)
  3. Transactional consumer (atomic process + commit)

Kafka exactly-once:
  - enable_idempotence=True
  - transactional_id='...'
  - isolation_level='read_committed'
  - Atomic: process + write + commit offset

Idempotency:
  Same message processed multiple times = same effect
  Check idempotency key before processing
  Store processed IDs in database

Exactly-once vs Idempotency:
  Exactly-once: Infrastructure guarantee (transactions)
  Idempotency: Application guarantee (check before process)
  
  Most systems: At-least-once + idempotency (simpler, faster)

Challenges:
  - Performance overhead (transactions slower)
  - Complexity (harder to implement)
  - Cross-system (requires distributed transactions)

When to use:
  ✅ Financial transactions
  ✅ Inventory management
  ✅ Billing systems
  ✅ Critical data
  
  ❌ Metrics and logs
  ❌ Idempotent operations
  ❌ Performance-critical

Best practices:
  - Start with at-least-once + idempotency
  - Use idempotency keys
  - Store offsets with data (single transaction)
  - Upgrade to exactly-once only if needed
  - Test with duplicates
```
