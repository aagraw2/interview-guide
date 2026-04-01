## 1. What is Event-Driven Architecture?

Event-Driven Architecture (EDA) is a design pattern where services communicate by producing and consuming events rather than direct synchronous calls.

```
Traditional (Request-Response):
  Service A → HTTP call → Service B
  Service A waits for response
  Tight coupling

Event-Driven:
  Service A → Publish event → Event Bus
  Service B → Subscribe to event → Process
  Loose coupling, async
```

**Benefits:** Loose coupling, scalability, flexibility, resilience.

---

## 2. Core Components

### Event

An immutable record of something that happened.

```
{
  "event_id": "123",
  "event_type": "OrderPlaced",
  "timestamp": "2024-01-01T10:00:00Z",
  "data": {
    "order_id": "456",
    "user_id": "789",
    "total": 99.99
  }
}

Events are facts (past tense):
  ✅ OrderPlaced, UserRegistered, PaymentProcessed
  ❌ PlaceOrder, RegisterUser, ProcessPayment (commands)
```

### Event Producer

Service that publishes events.

```
Order Service:
  1. User places order
  2. Save order to database
  3. Publish "OrderPlaced" event
  4. Return success to user
```

### Event Consumer

Service that subscribes to and processes events.

```
Inventory Service:
  1. Subscribe to "OrderPlaced" events
  2. Receive event
  3. Reserve inventory
  4. Publish "InventoryReserved" event

Email Service:
  1. Subscribe to "OrderPlaced" events
  2. Receive event
  3. Send confirmation email
```

### Event Bus

Infrastructure for routing events (Kafka, RabbitMQ, SNS/SQS).

```
Producers → Event Bus → Consumers

Event Bus responsibilities:
  - Store events (durability)
  - Route to subscribers
  - Guarantee delivery
  - Maintain order (per partition)
```

---

## 3. Event-Driven vs Request-Response

### Request-Response (Synchronous)

```
Order Service → Inventory Service: Reserve inventory
                ← Response: Success
Order Service → Payment Service: Process payment
                ← Response: Success
Order Service → Shipping Service: Ship order
                ← Response: Success

Problems:
  ❌ Tight coupling (Order Service knows all services)
  ❌ Cascading failures (if Shipping fails, order fails)
  ❌ Slow (sequential, waits for each response)
  ❌ Hard to add new services
```

### Event-Driven (Asynchronous)

```
Order Service → Publish "OrderPlaced" event

Inventory Service ← Subscribe → Reserve inventory → Publish "InventoryReserved"
Payment Service ← Subscribe → Process payment → Publish "PaymentProcessed"
Shipping Service ← Subscribe → Ship order → Publish "OrderShipped"

Benefits:
  ✅ Loose coupling (services don't know each other)
  ✅ Resilient (failures isolated)
  ✅ Fast (async, no waiting)
  ✅ Easy to add new services (just subscribe)
```

---

## 4. Event Patterns

### Event Notification

Notify other services that something happened.

```
Order Service:
  Publish "OrderPlaced" event

Email Service:
  Subscribe to "OrderPlaced"
  Send confirmation email

Analytics Service:
  Subscribe to "OrderPlaced"
  Update metrics

Minimal data in event (just notification)
Consumers fetch details if needed
```

### Event-Carried State Transfer

Include full state in event.

```
{
  "event_type": "OrderPlaced",
  "data": {
    "order_id": "456",
    "user_id": "789",
    "items": [...],
    "total": 99.99,
    "shipping_address": {...}
  }
}

Consumers have all data needed
No need to call back to producer
```

### Event Sourcing

Store all changes as events (event log is source of truth).

```
Traditional:
  users table: {id: 1, name: "Alice", email: "alice@example.com"}
  Update overwrites previous state

Event Sourcing:
  events table:
    {event: "UserCreated", data: {id: 1, name: "Alice"}}
    {event: "EmailUpdated", data: {id: 1, email: "alice@example.com"}}
  
  Current state = replay all events

Benefits:
  - Full audit trail
  - Time travel (replay to any point)
  - Debug (see all changes)
```

---

## 5. Implementation Example

### Producer (Order Service)

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

def place_order(order):
    # Save to database
    db.save_order(order)
    
    # Publish event
    event = {
        'event_type': 'OrderPlaced',
        'event_id': str(uuid.uuid4()),
        'timestamp': datetime.utcnow().isoformat(),
        'data': {
            'order_id': order.id,
            'user_id': order.user_id,
            'total': order.total
        }
    }
    
    producer.send('orders', value=event)
    producer.flush()
    
    return order
```

### Consumer (Inventory Service)

```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    group_id='inventory-service',
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for message in consumer:
    event = message.value
    
    if event['event_type'] == 'OrderPlaced':
        order_id = event['data']['order_id']
        
        # Reserve inventory
        reserve_inventory(order_id)
        
        # Publish event
        publish_event('InventoryReserved', {'order_id': order_id})
```

---

## 6. Choreography vs Orchestration

### Choreography (Event-Driven)

Services react to events independently.

```
Order Service → "OrderPlaced"
  ↓
Inventory Service → "InventoryReserved"
  ↓
Payment Service → "PaymentProcessed"
  ↓
Shipping Service → "OrderShipped"

No central coordinator
Each service knows what to do
```

**Pros:**
- Loose coupling
- No single point of failure
- Easy to add services

**Cons:**
- Hard to understand flow
- Difficult to debug
- No central view

### Orchestration (Saga Pattern)

Central orchestrator coordinates services.

```
Order Orchestrator:
  1. Command Inventory: Reserve
  2. Command Payment: Process
  3. Command Shipping: Ship

Central coordinator
Explicit workflow
```

**Pros:**
- Easy to understand
- Easy to debug
- Central control

**Cons:**
- Tight coupling to orchestrator
- Single point of failure
- Orchestrator complexity

---

## 7. Challenges

### Eventual consistency

```
Problem:
  Order Service: Order placed
  Inventory Service: Not yet processed event
  User queries inventory: Shows old state

Solution:
  - Accept eventual consistency
  - Show "processing" status
  - Use CQRS (separate read/write models)
```

### Event ordering

```
Problem:
  Event 1: Update user email to "alice@example.com"
  Event 2: Update user email to "alice@newdomain.com"
  
  Consumer receives out of order:
    Event 2 → email = "alice@newdomain.com"
    Event 1 → email = "alice@example.com" (wrong!)

Solution:
  - Kafka partitioning (same key → same partition → ordered)
  - Event versioning (timestamp, sequence number)
  - Idempotent consumers
```

### Duplicate events

```
Problem:
  Producer sends event twice (retry)
  Consumer processes twice

Solution:
  - Idempotent consumers (check if already processed)
  - Deduplication (store event IDs)
  - Exactly-once semantics (Kafka transactions)
```

### Error handling

```
Problem:
  Consumer fails to process event
  What happens to event?

Solutions:
  1. Retry (with exponential backoff)
  2. Dead letter queue (after max retries)
  3. Manual intervention (alert, investigate)
```

---

## 8. Monitoring

### Metrics

```
Event lag:
  Producer offset: 1000
  Consumer offset: 950
  Lag: 50 events
  Alert if lag > 1000

Event processing time:
  p95: 100ms
  Alert if > 1000ms

Event failure rate:
  Failures: 5%
  Alert if > 10%

Dead letter queue size:
  Size: 10 events
  Alert if > 100
```

### Distributed tracing

```
Trace events across services:

Order Service → "OrderPlaced" (trace_id: abc123)
  ↓
Inventory Service → "InventoryReserved" (trace_id: abc123)
  ↓
Payment Service → "PaymentProcessed" (trace_id: abc123)

Tools: Jaeger, Zipkin, OpenTelemetry
```

---

## 9. Common Interview Questions + Answers

### Q: What are the benefits of event-driven architecture?

> "Event-driven architecture provides loose coupling — services don't need to know about each other, they just publish and subscribe to events. This makes it easy to add new services without modifying existing ones. It's also more resilient since failures are isolated — if one consumer fails, others continue processing. It's scalable since consumers can be scaled independently. Finally, it provides flexibility — you can add new functionality by adding new consumers without changing producers."

### Q: What's the difference between choreography and orchestration?

> "Choreography is event-driven where services react to events independently with no central coordinator. Each service knows what to do when it sees an event. It's loosely coupled but hard to understand the overall flow. Orchestration uses a central coordinator that explicitly commands services in sequence. It's easier to understand and debug but creates tight coupling to the orchestrator. Use choreography for simple workflows and orchestration for complex workflows that need centralized control."

### Q: How do you handle eventual consistency in event-driven systems?

> "Accept that data will be eventually consistent, not immediately consistent. Show users 'processing' or 'pending' status while events propagate. Use CQRS to separate read and write models — the write model is immediately consistent, the read model is eventually consistent. Implement idempotent consumers so duplicate events don't cause issues. Use event versioning with timestamps to handle out-of-order events. For critical operations, use synchronous calls or sagas with compensating transactions."

### Q: How do you ensure event ordering in a distributed system?

> "Use Kafka partitioning with a consistent partition key — events with the same key go to the same partition and are processed in order. For example, partition by user_id to ensure all events for a user are ordered. Include sequence numbers or timestamps in events to detect out-of-order delivery. Make consumers idempotent so processing events multiple times or out of order doesn't cause issues. For strict ordering across all events, use a single partition, but this limits scalability."

---

## 10. Quick Reference

```
What is EDA?
  Services communicate via events (async)
  Loose coupling, scalability, resilience

Core components:
  Event: Immutable record of what happened
  Producer: Publishes events
  Consumer: Subscribes and processes events
  Event Bus: Routes events (Kafka, RabbitMQ)

Event patterns:
  Notification: Minimal data, consumers fetch details
  State Transfer: Full state in event
  Event Sourcing: Event log is source of truth

EDA vs Request-Response:
  EDA: Async, loose coupling, resilient
  Request-Response: Sync, tight coupling, cascading failures

Choreography vs Orchestration:
  Choreography: Services react independently (loose coupling)
  Orchestration: Central coordinator (easy to understand)

Challenges:
  Eventual consistency: Accept delay, show "processing"
  Event ordering: Kafka partitioning, sequence numbers
  Duplicates: Idempotent consumers, deduplication
  Errors: Retry, dead letter queue, alerts

Implementation:
  Kafka: High throughput, partitioning, durability
  RabbitMQ: Flexible routing, multiple protocols
  SNS/SQS: Managed, serverless

Monitoring:
  Event lag (alert if > 1000)
  Processing time (alert if > 1s)
  Failure rate (alert if > 10%)
  Dead letter queue size

Best practices:
  - Use past tense for events (OrderPlaced)
  - Include event ID and timestamp
  - Make consumers idempotent
  - Use partitioning for ordering
  - Monitor lag and failures
  - Implement dead letter queues
  - Use distributed tracing
```
