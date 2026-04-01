## 1. What is CQRS?

CQRS (Command Query Responsibility Segregation) is a pattern that separates read and write operations into different models.

```
Traditional (Single Model):
  User → API → Single Model → Database
  Same model for reads and writes
  
CQRS (Separate Models):
  User → Write API → Command Model → Write DB
  User → Read API → Query Model → Read DB
  
  Different models optimized for different purposes
```

**Key principle:** Reads and writes have different requirements, so use different models.

---

## 2. Core Concepts

### Command (Write)

An intent to change state.

```
Commands:
  - PlaceOrder
  - UpdateEmail
  - CancelSubscription
  
Command properties:
  - Imperative (verb)
  - Can be rejected (validation)
  - Changes state
  - Returns success/failure

Example:
  {
    "command": "PlaceOrder",
    "order_id": "123",
    "user_id": "456",
    "items": [...]
  }
```

### Query (Read)

A request for data.

```
Queries:
  - GetUserById
  - SearchOrders
  - GetOrderHistory
  
Query properties:
  - Never changes state
  - Always succeeds (or returns empty)
  - Can be cached
  - Optimized for reading

Example:
  {
    "query": "GetUserById",
    "user_id": "456"
  }
```

### Command Model (Write Model)

Optimized for writes and business logic.

```
Write Model:
  - Normalized schema
  - Strong consistency
  - ACID transactions
  - Business rules enforced
  - Optimized for writes

Example (Order):
  orders table: {id, user_id, status, created_at}
  order_items table: {id, order_id, product_id, quantity}
  
  Normalized, enforces referential integrity
```

### Query Model (Read Model)

Optimized for reads and queries.

```
Read Model:
  - Denormalized schema
  - Eventual consistency
  - No business logic
  - Optimized for reads
  - Multiple read models for different queries

Example (Order):
  order_summary table: {
    id, user_id, user_name, user_email,
    total_items, total_price, status, created_at
  }
  
  Denormalized, no joins needed
```

---

## 3. How It Works

### Write Path

```
1. User sends command (PlaceOrder)
2. Command handler validates
3. Apply business rules
4. Update write model (database)
5. Publish event (OrderPlaced)
6. Return success

Code:
  def place_order(command):
      # Validate
      if not validate_order(command):
          raise ValidationError()
      
      # Apply business rules
      order = Order.create(command)
      
      # Save to write model
      db.save(order)
      
      # Publish event
      event_bus.publish(OrderPlaced(order))
      
      return {"status": "success", "order_id": order.id}
```

### Read Path

```
1. User sends query (GetOrderById)
2. Query handler fetches from read model
3. Return data

Code:
  def get_order(query):
      # Fetch from read model (denormalized)
      order = read_db.get(query.order_id)
      return order

No business logic, just data retrieval
```

### Synchronization

```
Write model → Event → Read model

1. Command updates write model
2. Event published
3. Event handler updates read model

Example:
  OrderPlaced event:
    - Write model: orders table updated
    - Event published
    - Read model: order_summary table updated
    
Read model is eventually consistent
```

---

## 4. Benefits

### Optimized for different workloads

```
Write model:
  - Normalized (avoid data duplication)
  - Strong consistency (ACID)
  - Complex business logic
  
Read model:
  - Denormalized (fast queries, no joins)
  - Eventual consistency (acceptable for reads)
  - Simple data retrieval

Example:
  Write: SQL database (PostgreSQL)
  Read: NoSQL database (MongoDB) or cache (Redis)
```

### Independent scaling

```
Read-heavy system:
  - Scale read model (add replicas)
  - Write model stays small
  
Write-heavy system:
  - Scale write model (sharding)
  - Read model stays small

Example:
  E-commerce:
    - 90% reads (product catalog, search)
    - 10% writes (orders, inventory)
    
    Scale read model with CDN + Redis
    Write model on single PostgreSQL instance
```

### Multiple read models

```
Different queries need different schemas:

Read model 1: User profile
  {id, name, email, created_at}
  
Read model 2: User orders
  {user_id, orders: [{id, total, status}]}
  
Read model 3: User analytics
  {user_id, total_orders, total_spent, last_order_date}

Each optimized for specific query pattern
```

### Simplified queries

```
Without CQRS:
  SELECT u.name, u.email, COUNT(o.id) as order_count, SUM(o.total) as total_spent
  FROM users u
  LEFT JOIN orders o ON u.id = o.user_id
  GROUP BY u.id
  
  Complex join, slow

With CQRS:
  SELECT * FROM user_analytics WHERE user_id = 456
  
  Simple query, fast (denormalized)
```

---

## 5. Implementation Example

### Command Handler

```python
class PlaceOrderCommandHandler:
    def handle(self, command):
        # Validate
        if not self.validate(command):
            raise ValidationError("Invalid order")
        
        # Check inventory
        if not inventory_service.check_availability(command.items):
            raise BusinessRuleViolation("Items not available")
        
        # Create order (write model)
        order = Order(
            id=command.order_id,
            user_id=command.user_id,
            items=command.items,
            status="pending"
        )
        
        # Save to write database
        write_db.save(order)
        
        # Publish event
        event_bus.publish(OrderPlaced(
            order_id=order.id,
            user_id=order.user_id,
            items=order.items,
            total=order.total
        ))
        
        return {"status": "success", "order_id": order.id}
```

### Query Handler

```python
class GetOrderQueryHandler:
    def handle(self, query):
        # Fetch from read model (denormalized)
        order = read_db.query("""
            SELECT * FROM order_summary
            WHERE order_id = %s
        """, [query.order_id])
        
        return order

# No business logic, just data retrieval
```

### Event Handler (Sync Read Model)

```python
class OrderPlacedEventHandler:
    def handle(self, event):
        # Update read model
        read_db.execute("""
            INSERT INTO order_summary 
            (order_id, user_id, user_name, total_items, total_price, status)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, [
            event.order_id,
            event.user_id,
            self.get_user_name(event.user_id),
            len(event.items),
            event.total,
            "pending"
        ])

# Subscribe to events
event_bus.subscribe("OrderPlaced", OrderPlacedEventHandler())
```

---

## 6. CQRS + Event Sourcing

CQRS pairs naturally with Event Sourcing.

```
Event Sourcing:
  - Write model = event store (append-only log)
  - Read model = projections (materialized views)

CQRS:
  - Command model = event store
  - Query model = projections

They complement each other:
  - Event sourcing provides write model
  - CQRS provides read models
```

### Example

```
Command: PlaceOrder
  ↓
Event Store: OrderPlaced event appended
  ↓
Projections (Read Models):
  - order_summary table
  - user_orders table
  - order_analytics table

Query: GetOrderById
  ↓
Read from order_summary table
```

---

## 7. Challenges

### Eventual consistency

```
Problem:
  Command executed (write model updated)
  Read model not yet updated
  Query returns stale data

Solutions:
  1. Accept eventual consistency (most common)
  2. Return "processing" status
  3. Poll until read model updated
  4. Use version numbers to detect staleness

Example:
  User places order
  Redirect to "Order placed! Processing..."
  Poll order status until "confirmed"
```

### Complexity

```
Problem:
  Two models to maintain
  Synchronization logic
  More code

When to use CQRS:
  ✅ Complex domain with different read/write patterns
  ✅ High read/write ratio (need independent scaling)
  ✅ Multiple query patterns
  
  ❌ Simple CRUD applications
  ❌ Immediate consistency required
  ❌ Small team (maintenance overhead)
```

### Data duplication

```
Problem:
  Same data in write and read models
  Storage overhead

Mitigation:
  - Read models are projections (can be rebuilt)
  - Use cheaper storage for read models (S3, Redis)
  - Archive old data in write model
```

### Synchronization failures

```
Problem:
  Event published but read model update fails
  Read model out of sync

Solutions:
  1. Retry with exponential backoff
  2. Dead letter queue for failed events
  3. Rebuild read model from events
  4. Monitor lag between write and read models

Monitoring:
  - Event lag (time between write and read update)
  - Failed event count
  - Read model rebuild time
```

---

## 8. CQRS Patterns

### Simple CQRS

```
Same database, different models:

Write Model:
  Normalized tables (orders, order_items)
  
Read Model:
  Denormalized views (order_summary)

Synchronization:
  Database triggers or application code
```

### CQRS with Separate Databases

```
Different databases:

Write Database:
  PostgreSQL (ACID, strong consistency)
  
Read Database:
  MongoDB (denormalized, fast queries)
  
Synchronization:
  Event bus (Kafka)
```

### CQRS with Event Sourcing

```
Write Model:
  Event store (append-only log)
  
Read Model:
  Projections (materialized views)
  
Synchronization:
  Event replay
```

---

## 9. When to Use CQRS

### Good fit

```
✅ Different read/write patterns
✅ High read/write ratio (need independent scaling)
✅ Complex queries (need denormalization)
✅ Multiple query patterns (need multiple read models)
✅ Event-driven architecture

Examples:
  - E-commerce (product catalog, orders)
  - Social media (feeds, posts)
  - Analytics dashboards
```

### Poor fit

```
❌ Simple CRUD applications
❌ Immediate consistency required
❌ Low read/write ratio
❌ Single query pattern
❌ Small team (maintenance overhead)

Examples:
  - Simple blog
  - Internal tools
  - Prototypes
```

---

## 10. Common Interview Questions + Answers

### Q: What is CQRS and why would you use it?

> "CQRS separates read and write operations into different models. The write model is optimized for business logic and consistency, while the read model is optimized for queries and performance. You'd use it when reads and writes have very different requirements — for example, in an e-commerce system where you have complex order processing logic but need fast product catalog queries. It allows you to scale reads and writes independently and use different databases for each."

### Q: What's the difference between CQRS and event sourcing?

> "CQRS is about separating read and write models, while event sourcing is about storing all changes as events. They're independent patterns but work well together. You can use CQRS without event sourcing by just having separate read and write databases. You can use event sourcing without CQRS by using the same model for reads and writes. But they complement each other — event sourcing provides the write model (event store) and CQRS provides the read models (projections)."

### Q: How do you handle eventual consistency in CQRS?

> "Accept that the read model will lag behind the write model. Show users a 'processing' status after writes. Use polling or WebSockets to notify when the read model is updated. Include version numbers in responses so clients can detect stale data. For critical operations, read from the write model directly, though this defeats the purpose of CQRS. Most importantly, design the UX to accommodate eventual consistency — users are usually okay with a slight delay if you communicate it clearly."

### Q: What are the main challenges with CQRS?

> "First, eventual consistency — the read model lags behind writes, so you need to handle stale data. Second, complexity — you're maintaining two models and synchronization logic. Third, data duplication — the same data exists in both models. Fourth, synchronization failures — events can fail to update the read model, so you need retry logic and monitoring. Finally, it's overkill for simple applications — only use CQRS when you have genuinely different read and write requirements."

---

## 11. Quick Reference

```
What is CQRS?
  Separate read and write models
  Command model (write) + Query model (read)
  Different models for different purposes

Core concepts:
  Command: Intent to change state (PlaceOrder)
  Query: Request for data (GetOrderById)
  Command Model: Optimized for writes (normalized)
  Query Model: Optimized for reads (denormalized)

How it works:
  Write: Command → Write model → Event → Read model
  Read: Query → Read model → Data
  Synchronization: Eventually consistent

Benefits:
  - Optimized for different workloads
  - Independent scaling
  - Multiple read models
  - Simplified queries

Challenges:
  - Eventual consistency
  - Complexity (two models)
  - Data duplication
  - Synchronization failures

CQRS + Event Sourcing:
  Write model = Event store
  Read model = Projections
  Natural fit

When to use:
  ✅ Different read/write patterns
  ✅ High read/write ratio
  ✅ Complex queries
  ✅ Multiple query patterns
  
  ❌ Simple CRUD
  ❌ Immediate consistency required
  ❌ Small team

Best practices:
  - Accept eventual consistency
  - Monitor synchronization lag
  - Use multiple read models
  - Rebuild read models from events
  - Keep write model normalized
  - Keep read models denormalized
```
