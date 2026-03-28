# 🧩 Low Level Design Template

---

## 1. Clarify Requirements (2–3 mins)

### Functional Requirements
- What exactly should the system do?
- Core user actions

### Non-Functional Requirements
- Scalability (users, QPS)
- Performance (latency expectations)
- Consistency vs availability
- Concurrency needs

### Constraints / Assumptions
- Single machine vs distributed
- Data size
- Edge cases

---

## 2. Identify Core Entities (3–5 mins)

List the main objects (nouns in the problem):

- Entity 1
- Entity 2
- Entity 3

👉 Add key attributes:
- id
- state
- relationships

---

## 3. Define APIs / Use Cases (2–3 mins)

Write clear method signatures:

- createX()
- updateX()
- getX()
- deleteX()

Example:
- createOrder(userId, items)
- getOrder(orderId)

---

## 4. High-Level Class Design (5–10 mins)

### Models (Data)
- Plain objects (POJOs)
- Only fields + minimal logic

### Services (Business Logic)
- Core logic lives here

### Repositories (Data Access)
- DB interaction abstraction

👉 Mention:
- Separation of concerns (each layer has a clear responsibility)

---

## 5. Apply Design Patterns (IMPORTANT)

Ask yourself:

- Strategy → Do I have interchangeable logic?
- State → Does behavior change based on state?
- Observer → Do I need notifications/events?
- Factory → Object creation complexity?
- Singleton → Shared resource?

👉 Explicitly say:
“This part can use Strategy pattern because logic can vary”

---

## 6. Define Relationships (Diagram Thinking)

Explain relationships:

- One-to-one
- One-to-many
- Composition vs aggregation

Example:
- Order → contains → OrderItems
- User → has → Orders

---

## 7. Write Core Flows (MOST IMPORTANT)

Pick 1–2 flows and go deep:

### Example Flow:
createOrder()

Step 1: Validate input  
Step 2: Check inventory  
Step 3: Reserve items  
Step 4: Create order  
Step 5: Persist  
Step 6: Return response  

👉 This is where you stand out.

---

## 8. Handle Edge Cases

- Concurrency (race conditions)
- Duplicate requests (idempotency)
- Failures (retry / rollback)

Example:
- Payment → idempotency key
- Booking → locking

---

## 9. Optimize (if time)

- Caching (Redis)
- Indexing
- Async processing (queues)
- Pagination

---

## 10. Complexity Analysis

Mention briefly:

- Time complexity of key operations
- Space complexity (if relevant)

---

## 🧠 Mental Checklist (Quick Recall)

- Entities ✅  
- APIs ✅  
- Classes (Model + Service + Repo) ✅  
- Patterns ✅  
- Flow ✅  
- Edge cases ✅  

---

## 🔥 Golden Lines to Say

- “Let’s separate concerns using service and repository layers”
- “This logic can be abstracted using Strategy pattern”
- “We should make this idempotent”
- “We need locking to avoid race conditions”
- “We can make this async for better performance”

---

# 🏗️ Low Level Design Practice Sheet

| Name | Design Patterns Used | Entities | Approach | Time Complexities | Priority |
|------|---------------------|----------|----------|-------------------|----------|
| Parking Lot System | Strategy<br>Factory<br>Singleton | ParkingLot<br>Floor<br>ParkingSpot<br>Vehicle<br>Ticket<br>FeeCalculator | Maintain maps of spotType to available spots.<br>Factory creates vehicles.<br>Strategy is used for fee calculation.<br>Ticket is created on entry and closed on exit. | parkVehicle O(1)<br>unparkVehicle O(1)<br>findSpot O(1) | P1 |
| LRU Cache Design | HashMap + DoublyLinkedList<br>Singleton (optional) | LRUCache<br>Node | Use hashmap for lookup.<br>Doubly linked list for recency ordering.<br>Move node to head on access.<br>Remove tail on eviction. | get O(1)<br>put O(1)<br>evict O(1) | P1 |
| Rate Limiter Design | Strategy<br>Builder<br>Factory | RateLimiter<br>Request<br>RateLimiterConfig<br>TokenBucket<br>SlidingWindow<br>FixedWindow | Implement algorithms via strategy pattern.<br>Implement RateLimiter via builder pattern.<br>Maintain counters or token buckets per user or key. | allowRequest O(1) | P1 |
| Elevator System Design | State<br>Strategy<br>Observer | Elevator<br>ElevatorController<br>Floor<br>Request<br>Direction<br>ElevatorState | State pattern models moving, idle, and door states.<br>Controller assigns best elevator using scheduling strategy. | requestElevator O(n elevators)<br>moveElevator O(1) | P1 |
| BookMyShow System | Observer<br>Factory<br>Repository | User<br>Movie<br>Theatre<br>Screen<br>Seat<br>Show<br>Booking<br>Payment | Maintain seat availability per show.<br>Lock seats during booking.<br>Observer is used for notifications. | searchMovie O(n)<br>bookSeat O(1)<br>payment O(1) | P1 |
| Splitwise Design | Strategy<br>Factory | User<br>Expense<br>ExpenseSplit<br>BalanceSheet<br>ExpenseService | Maintain user balances.<br>Strategy handles equal, exact, and percent split. | addExpense O(n users)<br>settleBalance O(n) | P1 |
| Warehouse Design | Strategy<br>Repository<br>Observer | Warehouse<br>Product<br>InventoryItem<br>Order<br>OrderItem<br>InventoryService<br>OrderService<br>AllocationStrategy | Manage inventory across warehouses.<br>Allocate storage and fulfill orders using pluggable strategies. | addStock O(log n)<br>reserveInventory O(n)<br>createOrder O(n items) | P1 |
| Pub-Sub System | Strategy<br>Observer<br>Factory | Topic<br>Publisher<br>Subscriber<br>Broker<br>Message | Broker maintains topic to subscribers map.<br>Publishers send messages to topics.<br>Subscribers register or unregister dynamically.<br>Strategy is used for delivery.<br>Asynchronous queue is used for decoupling. | publish O(n subscribers)<br>subscribe O(1)<br>unsubscribe O(1) | P1 |
| Webhook Delivery System | Retry<br>Strategy<br>Observer | Webhook<br>Event<br>DeliveryAttempt<br>Queue<br>Worker | Events trigger webhook creation.<br>Requests are queued for asynchronous delivery.<br>Retry is handled with exponential backoff.<br>Idempotency key prevents duplicates.<br>Dead-letter queue is used for failures. | enqueue O(1)<br>delivery O(n retries)<br>lookup O(1) | P1 |
| Ledger System (Double Entry) | Factory<br>Strategy<br>Repository | Account<br>Transaction<br>Entry (debit or credit)<br>Ledger | Each transaction creates two entries.<br>The sum must always be zero.<br>Ledger is append-only for audit.<br>Transaction boundaries ensure consistency. | createTransaction O(1)<br>getBalance O(n) or O(1) cached | P1 |
| Payment Retry + Idempotency | Idempotency<br>Retry<br>State | PaymentRequest<br>Payment<br>IdempotencyKey<br>RetryPolicy | Store idempotency key to response mapping.<br>Retries use exponential backoff.<br>State machine ensures correct transitions.<br>Avoid duplicate payments. | processPayment O(1)<br>retry O(k)<br>lookup O(1) | P1 |
| Notification System | Observer<br>Strategy<br>Factory | User<br>Notification<br>Channel (Email, SMS, Push)<br>Publisher | Users subscribe to events.<br>Events trigger notification fanout.<br>Strategy selects channel.<br>Async processing via queue. | notify O(n users)<br>subscribe O(1) | P1 |
| Feature Toggle System | Strategy<br>Factory<br>Singleton | FeatureFlag<br>User<br>Rule<br>EvaluationEngine | Flags stored centrally.<br>Evaluation based on rules.<br>Strategy for rollout.<br>Cached locally for low latency. | evaluate O(1)<br>update O(1) | P1 |
| Circuit Breaker | State<br>Strategy<br>Singleton | CircuitBreaker<br>Request<br>Metrics<br>State | Track failures and success rates.<br>Open circuit on threshold breach.<br>Half-open allows trial requests.<br>Fallback strategy when open. | request O(1)<br>state check O(1) | P1 |
| Cab Booking System | Strategy<br>Observer<br>Singleton | Rider<br>Driver<br>Ride<br>Location<br>RideManager | Match nearest driver.<br>Observer for ride updates. | requestRide O(n drivers)<br>acceptRide O(1) | P2 |
| Digital Wallet System | Strategy<br>Facade | User<br>Wallet<br>Transaction<br>PaymentMethod | Wallet maintains balance and history.<br>Strategy for payment modes. | addMoney O(1)<br>transferMoney O(1)<br>getBalance O(1) | P2 |
| Task Scheduler Design | Strategy<br>Command | Task<br>Scheduler<br>Worker<br>Queue | Priority queue for scheduling.<br>Workers execute tasks. | scheduleTask O(log n)<br>executeTask O(1) | P2 |
| File System Design | Composite<br>Iterator | File<br>Directory<br>FileSystem | Tree structure using composite pattern. | addFile O(1)<br>listFiles O(n)<br>delete O(n) | P2 |
| Logging Framework Design | ChainOfResponsibility<br>Singleton | Logger<br>LogProcessor<br>LogLevel<br>Appender | Chain processes request until handled. | logMessage O(n processors) | P2 |
| Vending Machine | State<br>Strategy | VendingMachine<br>Item<br>Slot<br>Coin<br>State | State handles lifecycle transitions. | selectItem O(1)<br>insertCoin O(1)<br>dispense O(1) | P3 |
| Snake and Ladder Game | Strategy (optional) | Game<br>Board<br>Player<br>Dice<br>Snake<br>Ladder | Mapping of snakes and ladders.<br>Turn-based flow. | movePlayer O(1) | P3 |
| Chess Game | Strategy<br>State | Board<br>Piece<br>Player<br>Move | Each piece has its own movement logic. | movePiece O(1)<br>validateMove O(1) | P3 |
| Tic Tac Toe | Simple Strategy | Board<br>Player<br>Game | Counters for fast win detection. | move O(1)<br>checkWinner O(1) | P3 |