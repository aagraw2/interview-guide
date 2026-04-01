## 1. What is the Saga Pattern?

The Saga pattern manages distributed transactions across multiple services without using traditional two-phase commit (2PC). A saga is a sequence of local transactions where each transaction updates a single service and publishes an event or message to trigger the next step.

```
Traditional distributed transaction (2PC):
  Coordinator → Prepare phase → All services vote
  Coordinator → Commit phase → All services commit
  Problem: Blocking, low availability, complex

Saga pattern:
  Service A → Local transaction → Publish event
  Service B → Local transaction → Publish event
  Service C → Local transaction → Done
  If any step fails → Compensating transactions (rollback)
```

**Purpose:** Maintain data consistency across services without distributed transactions, improve availability, handle long-running transactions.

---

## 2. Why Sagas?

### Problem: Distributed transactions don't scale

```
Order service + Payment service + Inventory service

Two-phase commit (2PC):
  1. Coordinator: "Prepare to commit"
  2. All services: Lock resources, vote yes/no
  3. Coordinator: "Commit" or "Abort"
  4. All services: Commit or rollback

Problems:
  ❌ Blocking (services wait for coordinator)
  ❌ Single point of failure (coordinator)
  ❌ Low availability (all services must be up)
  ❌ Doesn't work across organizational boundaries
```

### Solution: Sagas

```
Order saga:
  1. Create order (Order service)
  2. Reserve inventory (Inventory service)
  3. Process payment (Payment service)
  4. Ship order (Shipping service)

If payment fails:
  → Compensate: Release inventory, cancel order
```

---

## 3. Saga Coordination

### Choreography (Event-driven)

Services publish events and listen to events from other services. No central coordinator.

```
1. Order service: Create order → Publish "OrderCreated"
2. Inventory service: Listen "OrderCreated" → Reserve inventory → Publish "InventoryReserved"
3. Payment service: Listen "InventoryReserved" → Process payment → Publish "PaymentProcessed"
4. Shipping service: Listen "PaymentProcessed" → Ship order → Publish "OrderShipped"

If payment fails:
  Payment service: Publish "PaymentFailed"
  Inventory service: Listen "PaymentFailed" → Release inventory
  Order service: Listen "PaymentFailed" → Cancel order
```

**Pros:**
- Decoupled services
- No single point of failure
- Good for simple workflows

**Cons:**
- Hard to understand (logic spread across services)
- Difficult to debug (no central view)
- Cyclic dependencies (services depend on each other)

### Orchestration (Command-driven)

Central orchestrator tells each service what to do. Orchestrator maintains state.

```
Order Orchestrator:
  1. Command Order service: Create order
  2. Command Inventory service: Reserve inventory
  3. Command Payment service: Process payment
  4. Command Shipping service: Ship order

If payment fails:
  Orchestrator:
    - Command Inventory service: Release inventory
    - Command Order service: Cancel order
```

**Pros:**
- Centralized logic (easy to understand)
- Easy to debug (orchestrator has full view)
- No cyclic dependencies

**Cons:**
- Orchestrator is a single point of failure
- Orchestrator can become complex
- Services coupled to orchestrator

---

## 4. Compensating Transactions

Undo the effects of previous steps when a saga fails.

### Forward recovery (retry)

```
Step fails → Retry until it succeeds

Example:
  Payment service temporarily down
  → Retry payment
  → Eventually succeeds
  → Continue saga

Use when: Failure is transient
```

### Backward recovery (compensate)

```
Step fails → Undo previous steps

Example:
  Payment fails (insufficient funds)
  → Release inventory (compensate step 2)
  → Cancel order (compensate step 1)

Use when: Failure is permanent
```

### Compensating transaction design

```
Transaction: Reserve inventory (quantity -= 10)
Compensation: Release inventory (quantity += 10)

Transaction: Charge credit card ($100)
Compensation: Refund credit card ($100)

Transaction: Send email
Compensation: Send cancellation email (can't unsend)

Not all operations are perfectly compensatable
```

---

## 5. Saga State Machine

Track saga progress and handle failures.

### States

```
PENDING → INVENTORY_RESERVED → PAYMENT_PROCESSED → SHIPPED → COMPLETED

If payment fails:
  PAYMENT_PROCESSED → PAYMENT_FAILED → COMPENSATING → CANCELLED
```

### State transitions

```
State: PENDING
  Action: Reserve inventory
  Success → INVENTORY_RESERVED
  Failure → CANCELLED

State: INVENTORY_RESERVED
  Action: Process payment
  Success → PAYMENT_PROCESSED
  Failure → COMPENSATING (release inventory)

State: COMPENSATING
  Action: Release inventory
  Success → CANCELLED
  Failure → RETRY_COMPENSATING
```

---

## 6. Implementation Example

### Orchestration with state machine

```python
class OrderSaga:
    def __init__(self, order_id):
        self.order_id = order_id
        self.state = "PENDING"
    
    def execute(self):
        try:
            # Step 1: Create order
            self.create_order()
            self.state = "ORDER_CREATED"
            
            # Step 2: Reserve inventory
            self.reserve_inventory()
            self.state = "INVENTORY_RESERVED"
            
            # Step 3: Process payment
            self.process_payment()
            self.state = "PAYMENT_PROCESSED"
            
            # Step 4: Ship order
            self.ship_order()
            self.state = "COMPLETED"
            
        except InventoryException:
            self.compensate_order()
        except PaymentException:
            self.compensate_inventory()
            self.compensate_order()
        except ShippingException:
            self.compensate_payment()
            self.compensate_inventory()
            self.compensate_order()
    
    def create_order(self):
        order_service.create(self.order_id)
    
    def reserve_inventory(self):
        inventory_service.reserve(self.order_id)
    
    def process_payment(self):
        payment_service.charge(self.order_id)
    
    def ship_order(self):
        shipping_service.ship(self.order_id)
    
    def compensate_order(self):
        order_service.cancel(self.order_id)
        self.state = "CANCELLED"
    
    def compensate_inventory(self):
        inventory_service.release(self.order_id)
    
    def compensate_payment(self):
        payment_service.refund(self.order_id)
```

---

## 7. Saga Persistence

Store saga state to handle failures and restarts.

### Saga log

```
saga_id: 12345
state: INVENTORY_RESERVED
steps_completed:
  - create_order (timestamp: 2024-01-01 10:00:00)
  - reserve_inventory (timestamp: 2024-01-01 10:00:01)
next_step: process_payment
compensations_if_fail:
  - release_inventory
  - cancel_order
```

### Recovery

```
On orchestrator restart:
  1. Load all in-progress sagas from database
  2. For each saga:
     - Check current state
     - Resume from next step
     - Or compensate if failed
```

---

## 8. Saga vs 2PC

|Feature|Saga|Two-Phase Commit (2PC)|
|---|---|---|
|Consistency|Eventual|Immediate|
|Availability|High|Low|
|Isolation|No (dirty reads possible)|Yes|
|Complexity|High (compensations)|Moderate|
|Performance|High (no locking)|Low (locking)|
|Failure handling|Compensating transactions|Rollback|
|Use case|Microservices, long transactions|Monoliths, short transactions|

---

## 9. Saga Isolation Issues

Sagas don't provide isolation — other transactions can see intermediate states.

### Lost update

```
Saga 1: Read balance ($100) → Deduct $50 → Write balance ($50)
Saga 2: Read balance ($100) → Deduct $30 → Write balance ($70)

Final balance: $70 (should be $20)
Saga 1's update lost
```

**Solution:** Optimistic locking (version numbers)

### Dirty read

```
Saga 1: Reserve inventory (quantity = 90)
Saga 2: Read inventory (quantity = 90) → Reserve 10 more
Saga 1: Fails → Compensate (quantity = 100)

Saga 2 saw dirty data (quantity = 90 was temporary)
```

**Solution:** Semantic lock (mark inventory as "reserved")

---

## 10. Common Interview Questions + Answers

### Q: What is the Saga pattern and when would you use it?

> "The Saga pattern manages distributed transactions across microservices without using two-phase commit. A saga is a sequence of local transactions where each service updates its own database and publishes an event to trigger the next step. If any step fails, compensating transactions undo the previous steps. Use sagas for long-running transactions across services, like order processing that involves inventory, payment, and shipping. Sagas provide high availability and performance but only eventual consistency."

### Q: What's the difference between choreography and orchestration in sagas?

> "Choreography is event-driven — services publish and listen to events with no central coordinator. It's decoupled but hard to understand and debug since logic is spread across services. Orchestration uses a central orchestrator that commands each service what to do and maintains the saga state. It's easier to understand and debug but the orchestrator is a single point of failure. Use choreography for simple workflows and orchestration for complex workflows that need centralized control."

### Q: How do compensating transactions work?

> "Compensating transactions undo the effects of previous steps when a saga fails. For example, if payment fails after reserving inventory, the compensation releases the inventory and cancels the order. Not all operations are perfectly compensatable — you can't unsend an email, only send a cancellation email. Design compensations carefully and make them idempotent since they might be retried. Use forward recovery (retry) for transient failures and backward recovery (compensate) for permanent failures."

### Q: What are the isolation issues with sagas?

> "Sagas don't provide isolation like database transactions do. Other transactions can see intermediate states, causing issues like lost updates and dirty reads. For example, if Saga 1 reserves inventory and then fails and compensates, Saga 2 might have seen the temporarily reduced inventory. Solutions include optimistic locking with version numbers, semantic locks to mark resources as reserved, and designing compensations to be idempotent. Accept that sagas provide eventual consistency, not strong consistency."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Know both choreography and orchestration

This is a common interview question. Know the trade-offs and when to use each.

### ✅ Trick 2: Discuss compensating transactions

Compensations are the key to sagas. Mentioning idempotency and that not all operations are compensatable shows depth.

### ✅ Trick 3: Mention isolation issues

Sagas don't provide isolation. Mentioning lost updates and dirty reads shows you understand the limitations.

### ❌ Pitfall 1: Thinking sagas provide strong consistency

Sagas provide eventual consistency, not strong consistency. Don't confuse them with ACID transactions.

### ❌ Pitfall 2: Forgetting about idempotency

Compensations might be retried. They must be idempotent.

### ❌ Pitfall 3: Not persisting saga state

The orchestrator can crash. Saga state must be persisted to resume after restart.

---

## 12. Quick Reference

```
What is the Saga pattern?
  Manage distributed transactions without 2PC
  Sequence of local transactions
  Compensating transactions on failure

Why sagas?
  2PC doesn't scale (blocking, low availability)
  Sagas: High availability, eventual consistency

Choreography (Event-driven):
  Services publish/listen to events
  No central coordinator
  Pros: Decoupled, no SPOF
  Cons: Hard to understand, difficult to debug

Orchestration (Command-driven):
  Central orchestrator commands services
  Orchestrator maintains state
  Pros: Easy to understand, easy to debug
  Cons: SPOF, orchestrator complexity

Compensating transactions:
  Undo effects of previous steps
  Forward recovery: Retry (transient failures)
  Backward recovery: Compensate (permanent failures)
  Must be idempotent

Saga state machine:
  Track progress: PENDING → RESERVED → PAID → SHIPPED → COMPLETED
  Handle failures: PAID → FAILED → COMPENSATING → CANCELLED

Saga persistence:
  Store saga state in database
  Resume after orchestrator restart
  Replay or compensate in-progress sagas

Saga vs 2PC:
  Saga: Eventual consistency, high availability
  2PC: Strong consistency, low availability

Isolation issues:
  Lost updates: Use optimistic locking
  Dirty reads: Use semantic locks
  Accept eventual consistency

Best practices:
  - Choose choreography for simple workflows
  - Choose orchestration for complex workflows
  - Design idempotent compensations
  - Persist saga state
  - Handle partial failures gracefully
  - Monitor saga completion rates
```
