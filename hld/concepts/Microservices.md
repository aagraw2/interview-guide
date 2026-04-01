## 1. What Are Microservices?

Microservices is an architectural style where an application is composed of small, independent services that communicate over a network.

```
Monolith:
  ┌─────────────────────────────┐
  │  Single Application         │
  │  - User Management          │
  │  - Order Processing         │
  │  - Payment                  │
  │  - Inventory                │
  │  - Notifications            │
  └─────────────────────────────┘
  Single codebase, single deployment

Microservices:
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  User    │  │  Order   │  │ Payment  │
  │ Service  │  │ Service  │  │ Service  │
  └──────────┘  └──────────┘  └──────────┘
  ┌──────────┐  ┌──────────┐
  │Inventory │  │  Notify  │
  │ Service  │  │ Service  │
  └──────────┘  └──────────┘
  Independent services, independent deployments
```

**Key characteristics:**
- Small, focused services (single responsibility)
- Independent deployment
- Own their data (no shared database)
- Communicate via APIs (REST, gRPC, messaging)
- Technology agnostic (each service can use different tech stack)

---

## 2. Microservices vs Monolith

### Monolith

```
Single application:
  - All code in one codebase
  - Single database
  - Deploy entire application together
  - Scale entire application together
```

**Pros:**
- Simple to develop initially
- Easy to test (everything in one place)
- Simple deployment (one artifact)
- No network latency between components

**Cons:**
- Hard to scale (must scale entire app)
- Slow deployments (small change requires full redeploy)
- Technology lock-in (entire app uses same stack)
- Team coordination overhead (everyone works in same codebase)
- Failure cascades (one bug can crash entire app)

---

### Microservices

```
Multiple services:
  - Each service has own codebase
  - Each service has own database
  - Deploy services independently
  - Scale services independently
```

**Pros:**
- Independent scaling (scale only what needs it)
- Independent deployment (deploy one service without affecting others)
- Technology flexibility (use best tool for each job)
- Team autonomy (teams own services end-to-end)
- Fault isolation (one service failure doesn't crash everything)

**Cons:**
- Distributed system complexity
- Network latency between services
- Data consistency challenges (no ACID across services)
- Harder to test (integration testing complex)
- Operational overhead (many services to monitor, deploy, debug)

---

## 3. Service Boundaries

How do you split a monolith into microservices?

### Domain-Driven Design (DDD)

Organize services around business domains.

```
E-commerce system:

User Service:
  - User registration
  - Authentication
  - Profile management

Order Service:
  - Create order
  - Order status
  - Order history

Payment Service:
  - Process payment
  - Refunds
  - Payment methods

Inventory Service:
  - Stock levels
  - Reserve items
  - Restock

Notification Service:
  - Email notifications
  - SMS notifications
  - Push notifications
```

**Each service owns a business capability.**

---

### Bounded contexts

Each service has its own model of shared concepts.

```
"User" means different things in different contexts:

User Service:
  User = {id, email, password, profile}

Order Service:
  User = {id, name, shipping_address}

Payment Service:
  User = {id, billing_address, payment_methods}

Each service has its own "User" table with only the data it needs
```

**No shared database. Each service owns its data.**

---

### Service size

**"Micro" doesn't mean tiny.** Size is about scope, not lines of code.

```
Too small:
  - CRUD service per entity (UserService, AddressService, PhoneService)
  - Too many network calls
  - Distributed monolith

Too large:
  - Service does too many things
  - Hard to understand and maintain
  - Defeats purpose of microservices

Right size:
  - Service owns a business capability
  - Team can understand and maintain it
  - Can be rewritten in 2 weeks if needed
```

**Rule of thumb:** If a service requires multiple teams to maintain, it's too large.

---

## 4. Communication Patterns

### Synchronous (Request-Response)

Service A calls Service B and waits for response.

**REST:**

```
Order Service → HTTP POST /payments → Payment Service
                ← 200 OK {"payment_id": "123"}
```

**gRPC:**

```
Order Service → ProcessPayment(order_id, amount) → Payment Service
                ← PaymentResponse{payment_id: "123", status: "success"}
```

**Pros:** Simple, immediate response. **Cons:** Tight coupling, cascading failures, higher latency.

---

### Asynchronous (Event-Driven)

Service A publishes an event, Service B subscribes and reacts.

```
Order Service → [Event: OrderCreated] → Message Queue
                                            ↓
                                    Payment Service (subscribes)
                                    Inventory Service (subscribes)
                                    Notification Service (subscribes)
```

**Example:**

```
1. Order Service: Order created → publish "OrderCreated" event
2. Payment Service: Receives event → process payment
3. Inventory Service: Receives event → reserve items
4. Notification Service: Receives event → send confirmation email

All happen independently, in parallel
```

**Pros:** Loose coupling, fault tolerance, scalability. **Cons:** Eventual consistency, harder to debug, complexity.

---

### When to use which

```
Synchronous (REST/gRPC):
  ✅ Need immediate response
  ✅ Simple request-response flow
  ✅ Low latency critical
  Example: Get user profile, validate payment

Asynchronous (Events):
  ✅ Fire-and-forget operations
  ✅ Multiple services need to react
  ✅ Can tolerate eventual consistency
  Example: Order placed, user registered, payment completed
```

---

## 5. Data Management

### Database per service

Each service owns its database. No shared database.

```
User Service → [User DB]
Order Service → [Order DB]
Payment Service → [Payment DB]
```

**Why:**
- Services are independent (schema changes don't affect others)
- Can choose best database for each service
- Fault isolation (one DB failure doesn't affect all services)

**Challenge:** No joins across services, no distributed transactions.

---

### Data consistency

**Problem:** How to keep data consistent across services?

**Example:**

```
Create order:
  1. Order Service: Create order record
  2. Inventory Service: Decrement stock
  3. Payment Service: Charge customer

What if step 3 fails? Order created, stock decremented, but payment failed.
```

**Solutions:**

---

### Saga pattern

Distributed transaction via compensating actions.

**Choreography (event-driven):**

```
1. Order Service: Create order → publish "OrderCreated"
2. Inventory Service: Decrement stock → publish "StockReserved"
3. Payment Service: Charge customer
   - Success → publish "PaymentCompleted"
   - Failure → publish "PaymentFailed"
4. Inventory Service: Listens to "PaymentFailed" → restore stock (compensate)
5. Order Service: Listens to "PaymentFailed" → cancel order (compensate)
```

**Orchestration (coordinator):**

```
Order Orchestrator:
  1. Call Inventory Service: Reserve stock
  2. Call Payment Service: Charge customer
  3. If payment fails:
     - Call Inventory Service: Restore stock
     - Call Order Service: Cancel order
```

**Choreography:** Decentralized, services react to events. **Orchestration:** Centralized, coordinator manages flow.

---

### Event sourcing

Store events instead of current state.

```
Traditional:
  orders table: {id: 1, status: "completed", amount: 99.99}

Event sourcing:
  events table:
    {event: "OrderCreated", order_id: 1, amount: 99.99}
    {event: "PaymentProcessed", order_id: 1}
    {event: "OrderShipped", order_id: 1}
    {event: "OrderCompleted", order_id: 1}

Current state = replay all events
```

**Benefits:** Full audit trail, can rebuild state, time travel. **Cons:** Complexity, eventual consistency.

---

## 6. Service Discovery

How do services find each other?

### Client-side discovery

Client queries service registry, then calls service directly.

```
1. Order Service → Service Registry: "Where is Payment Service?"
2. Service Registry → "Payment Service: 10.0.1.5:8080, 10.0.1.6:8080"
3. Order Service → Calls 10.0.1.5:8080 directly
```

**Examples:** Netflix Eureka, Consul

---

### Server-side discovery

Client calls load balancer, load balancer queries registry.

```
1. Order Service → Load Balancer: "Call Payment Service"
2. Load Balancer → Service Registry: "Where is Payment Service?"
3. Load Balancer → Routes to 10.0.1.5:8080
```

**Examples:** AWS ALB, Kubernetes Service

---

### Service mesh

Sidecar proxy handles all service-to-service communication.

```
Order Service → [Envoy Proxy] → [Envoy Proxy] → Payment Service
                     ↓                  ↓
              Service Discovery    Service Discovery
```

**Examples:** Istio, Linkerd

**Benefits:** Service discovery, load balancing, retries, circuit breaking, observability — all handled by infrastructure.

---

## 7. API Gateway

Single entry point for external clients.

```
External Clients → [API Gateway] → Microservices
                        ↓
                   Auth, Rate Limiting,
                   Routing, SSL
```

**Responsibilities:**
- Authentication & authorization
- Rate limiting
- Request routing
- SSL termination
- API composition (aggregate multiple services)

**See API Gateway doc for details.**

---

## 8. Observability

Distributed systems are hard to debug. Need strong observability.

### Distributed tracing

Track requests across services.

```
Request ID: abc123

API Gateway (5ms) → Order Service (20ms) → Payment Service (50ms)
                                        → Inventory Service (30ms)

Total: 105ms
Bottleneck: Payment Service (50ms)
```

**Tools:** Jaeger, Zipkin, AWS X-Ray

---

### Centralized logging

Aggregate logs from all services.

```
All services → [Log Aggregator] → [Search/Analysis]
               (Elasticsearch,      (Kibana)
                Splunk)
```

**Correlation ID:** Include request ID in all logs to trace a request across services.

---

### Metrics

Monitor health of each service.

```
Per service:
  - Request rate (requests/sec)
  - Error rate (5xx responses)
  - Latency (p50, p95, p99)
  - CPU, memory usage

Tools: Prometheus, Grafana, Datadog
```

---

## 9. Deployment Strategies

### Independent deployment

Each service deployed independently.

```
Deploy Order Service v2:
  - No need to deploy other services
  - Other services continue running v1
```

**Enables:** Faster releases, smaller changes, less risk.

---

### Blue-Green deployment

Run two versions, switch traffic.

```
Blue (v1): 100% traffic
Green (v2): 0% traffic (deployed, tested)

Switch:
Blue (v1): 0% traffic
Green (v2): 100% traffic

If issues: switch back to Blue
```

---

### Canary deployment

Gradually roll out to subset of users.

```
v1: 95% traffic
v2: 5% traffic (canary)

Monitor metrics. If good:
v1: 50% traffic
v2: 50% traffic

Eventually:
v1: 0% traffic
v2: 100% traffic
```

---

## 10. Common Pitfalls

### Distributed monolith

Microservices that are tightly coupled.

```
Order Service → calls Payment Service → calls Inventory Service → calls User Service

All services must be deployed together
One service down → entire flow breaks
```

**This is worse than a monolith.** You have microservices complexity without the benefits.

**Fix:** Use async communication, design for failure, loose coupling.

---

### Too many services

```
100 microservices for a small team
→ Operational nightmare
→ Can't keep track of what's where
→ Debugging is impossible
```

**Start with a monolith. Split when you have a clear reason.**

---

### Shared database

```
Order Service → [Shared DB] ← Payment Service

Services coupled via database
Schema changes affect multiple services
```

**This defeats the purpose of microservices.**

---

### Chatty services

```
Order Service:
  1. Call User Service (get user)
  2. Call Inventory Service (check stock)
  3. Call Pricing Service (get price)
  4. Call Tax Service (calculate tax)
  5. Call Payment Service (charge)

5 network calls → high latency
```

**Fix:** API composition at gateway, or use async events.

---

## 11. When to Use Microservices

### Use microservices when:

```
✅ Large team (> 20 developers)
✅ Different parts scale differently
✅ Need independent deployment
✅ Different tech stacks for different services
✅ Have DevOps maturity (CI/CD, monitoring, etc.)
```

### Don't use microservices when:

```
❌ Small team (< 5 developers)
❌ Simple application
❌ No clear service boundaries
❌ Lack DevOps infrastructure
❌ Can't handle distributed system complexity
```

**Start with a monolith. Split when you have a clear need.**

---

## 12. Common Interview Questions + Answers

### Q: What are microservices and what are their benefits?

> "Microservices is an architectural style where an application is composed of small, independent services that communicate over a network. Each service owns a business capability, has its own database, and can be deployed independently. Benefits include independent scaling (scale only what needs it), independent deployment (deploy one service without affecting others), technology flexibility (use best tool for each job), team autonomy (teams own services end-to-end), and fault isolation (one service failure doesn't crash everything). The tradeoff is distributed system complexity — network latency, data consistency challenges, and operational overhead."

### Q: How do microservices communicate with each other?

> "Two main patterns: synchronous (request-response) and asynchronous (event-driven). Synchronous uses REST or gRPC — Service A calls Service B and waits for a response. It's simple but creates tight coupling. Asynchronous uses message queues or pub/sub — Service A publishes an event, Service B subscribes and reacts independently. It's more complex but provides loose coupling and better fault tolerance. I'd use synchronous for operations that need immediate responses (like fetching user data) and asynchronous for operations where multiple services need to react (like order placement)."

### Q: How do you handle transactions across microservices?

> "You can't use traditional ACID transactions across services since each service has its own database. Instead, use the Saga pattern — a sequence of local transactions with compensating actions for rollback. There are two approaches: choreography (event-driven, services react to events) and orchestration (central coordinator manages the flow). For example, when creating an order: Order Service creates the order, Inventory Service reserves stock, Payment Service charges the customer. If payment fails, Inventory Service restores stock and Order Service cancels the order. This provides eventual consistency with compensation."

### Q: When would you choose microservices over a monolith?

> "Choose microservices when you have a large team that needs to work independently, different parts of the system scale differently, you need independent deployment cycles, or you want to use different technologies for different services. But you need DevOps maturity — CI/CD pipelines, monitoring, distributed tracing. Don't use microservices for small teams or simple applications — the operational overhead isn't worth it. I'd start with a well-structured monolith and split into microservices when you have clear service boundaries and a team that can handle the complexity."

---

## 13. Interview Tricks & Pitfalls

### ✅ Trick 1: Mention the tradeoffs

Don't just list benefits:

> "Microservices provide independent scaling and deployment, but they add distributed system complexity — network latency, data consistency challenges, and operational overhead. You need strong DevOps practices to make them work."

### ✅ Trick 2: Discuss service boundaries

Show you understand how to split services:

> "I'd use domain-driven design to identify service boundaries — each service owns a business capability like User Management, Order Processing, or Payment. Each service has its own database with only the data it needs. This ensures loose coupling."

### ✅ Trick 3: Address the distributed monolith problem

Shows you know the common pitfall:

> "The worst outcome is a distributed monolith — microservices that are tightly coupled and must be deployed together. This gives you all the complexity of microservices without the benefits. To avoid this, I'd use async communication where possible and design each service to be independently deployable."

### ❌ Pitfall 1: Saying "microservices are always better"

Microservices have significant tradeoffs. Don't dismiss monoliths:

> "For a small team or simple application, a monolith is often the right choice. It's simpler to develop, test, and deploy. I'd only move to microservices when there's a clear need — like independent scaling or team autonomy."

### ❌ Pitfall 2: Not mentioning observability

Distributed systems are hard to debug:

> "With microservices, I'd invest heavily in observability — distributed tracing to track requests across services, centralized logging with correlation IDs, and metrics for each service. Without these, debugging is nearly impossible."

### ❌ Pitfall 3: Forgetting about data consistency

If you propose microservices, be ready to discuss transactions:

> "Since each service has its own database, I can't use ACID transactions across services. I'd use the Saga pattern with compensating transactions for operations that span multiple services."

---

## 14. Quick Reference

```
Microservices = small, independent services communicating over network

Key characteristics:
  - Single responsibility (one business capability)
  - Own their data (database per service)
  - Independent deployment
  - Communicate via APIs (REST, gRPC, events)
  - Technology agnostic

vs Monolith:
  Monolith: Simple, single deployment, shared database
  Microservices: Complex, independent deployment, distributed data

Service boundaries:
  - Domain-Driven Design (organize by business domain)
  - Bounded contexts (each service has own model)
  - Right size: team can maintain, can rewrite in 2 weeks

Communication:
  Synchronous (REST/gRPC): Immediate response, tight coupling
  Asynchronous (Events): Loose coupling, eventual consistency

Data management:
  - Database per service (no shared DB)
  - Saga pattern (distributed transactions with compensation)
  - Event sourcing (store events, rebuild state)

Service discovery:
  - Client-side: Client queries registry
  - Server-side: Load balancer queries registry
  - Service mesh: Sidecar proxy handles all

Observability:
  - Distributed tracing (Jaeger, Zipkin)
  - Centralized logging (Elasticsearch, Splunk)
  - Metrics (Prometheus, Grafana)

Deployment:
  - Independent deployment
  - Blue-green (switch versions)
  - Canary (gradual rollout)

Common pitfalls:
  - Distributed monolith (tight coupling)
  - Too many services (operational overhead)
  - Shared database (defeats purpose)
  - Chatty services (high latency)

When to use:
  ✅ Large team, different scaling needs, independent deployment
  ❌ Small team, simple app, no DevOps maturity

Start with monolith, split when needed
```
