# LLD Systems Quick Reference

One-page mental model per system. Core entities, key design patterns used, and what the interviewer is actually testing.

---

## Parking Lot System

**Core problem:** Allocate the right slot for a vehicle, generate a ticket, calculate fee on exit.

**Entities:** `Vehicle`, `VehicleType (enum)`, `ParkingSlot`, `Floor`, `Ticket`, `ParkingLot`, `ParkingService`

**Patterns used:**
- Strategy: `SlotAllocationStrategy` (nearest slot, random, etc.) and `FeeStrategy` (hourly, slab-based)
- Singleton: `ParkingLot` — single source of truth for all floors and slots
- Facade: `ParkingService` — simplifies entry/exit flow for the client
- Factory: `VehicleFactory` — decouples vehicle creation

**Key design decisions:**
- `ParkingSlot` has `VehicleType` and `isFree` — slot is typed, not generic
- `Ticket` stores entry time, vehicle, and slot reference — all you need for exit
- Fee calculation is pluggable via `FeeStrategy` — easy to add slab pricing without touching core logic
- Slot search is O(F × S) worst case — acceptable for typical lot sizes

**Interviewer is testing:** Strategy pattern for pricing and allocation, Singleton for the lot, how you model the ticket lifecycle.

---

## LRU Cache

**Core problem:** O(1) get and put with automatic eviction of least recently used item.

**Entities:** `Node (key, value, prev, next)`, `DoublyLinkedList`, `LRUCache`

**Patterns used:**
- Strategy: `EvictionPolicy` interface — swap LRU for LFU without changing cache logic
- Singleton: `LRUCache` — single cache instance

**Key design decisions:**
- HashMap gives O(1) key lookup. DLL gives O(1) move-to-front and remove-tail.
- Dummy head and tail nodes eliminate null checks in DLL operations
- On get: move node to head (MRU). On put: add to head, evict tail if over capacity.
- `EvictionPolicy` interface makes it easy to extend to LFU or TTL-based eviction

**Core code pattern:**
```
get(key):
  if not in map → return -1
  move node to head
  return value

put(key, value):
  if exists → update + move to head
  else → create node + add to head + add to map
  if size > capacity → remove tail + remove from map
```

**Interviewer is testing:** HashMap + DLL combination, why both are needed, O(1) proof.

---

## Rate Limiter

**Core problem:** Allow at most N requests per user per time window. Reject excess requests.

**Entities:** `RateLimiter (interface)`, `TokenBucketLimiter`, `SlidingWindowLimiter`, `FixedWindowLimiter`, `RateLimitService`

**Patterns used:**
- Strategy: `RateLimiter` interface — swap algorithms without changing service
- Factory: create the right limiter based on config

**Algorithms:**
- Token Bucket: bucket refills at fixed rate, each request consumes one token. Allows bursts up to bucket size.
- Sliding Window: track request timestamps in a queue, count requests in last N seconds. More accurate, more memory.
- Fixed Window: count requests per fixed time window (e.g., per minute). Simple but allows burst at window boundary.

**Key design decisions:**
- Per-user state stored in Redis for distributed rate limiting
- Token bucket is the most common interview answer — explain burst allowance
- Sliding window is most accurate but O(requests) memory per user

**Interviewer is testing:** Token bucket vs sliding window trade-offs, distributed rate limiting with Redis.

---

## Elevator System

**Core problem:** Dispatch elevators to floors efficiently, handle multiple elevators and requests.

**Entities:** `Elevator`, `ElevatorState (enum: IDLE, MOVING_UP, MOVING_DOWN)`, `Request (floor, direction)`, `ElevatorController`, `DispatchStrategy`

**Patterns used:**
- State: `Elevator` changes behavior based on state (IDLE, MOVING_UP, MOVING_DOWN)
- Strategy: `DispatchStrategy` — SCAN algorithm, nearest elevator, etc.
- Observer: `ElevatorController` observes floor button presses and dispatches

**Key design decisions:**
- SCAN algorithm (elevator algorithm): elevator moves in one direction, picks up all requests in that direction, then reverses
- Each elevator maintains a sorted set of destination floors
- Controller assigns request to elevator with minimum cost (distance + direction alignment)
- State transitions: IDLE → MOVING_UP/DOWN on request, back to IDLE when no more requests

**Interviewer is testing:** State pattern for elevator states, SCAN algorithm, how you model the dispatch problem.

---

## BookMyShow (LLD)

**Core problem:** Book seats for a show — check availability, hold seat, confirm booking.

**Entities:** `Movie`, `Theatre`, `Screen`, `Show`, `Seat`, `SeatType (enum)`, `Booking`, `Payment`, `User`

**Patterns used:**
- Strategy: `PricingStrategy` — different prices for different seat types
- Factory: `BookingFactory` — creates booking with correct state
- Observer: notify user on booking confirmation/cancellation
- Singleton: `BookingService` — single entry point for all bookings

**Key design decisions:**
- `Seat` has `SeatType` (REGULAR, PREMIUM, VIP) and `SeatStatus` (AVAILABLE, HELD, BOOKED)
- Hold seat before payment — release if payment fails or times out
- `Booking` has status enum: PENDING → CONFIRMED → CANCELLED
- Concurrency: `synchronized` on seat status update or optimistic locking

**Interviewer is testing:** Seat hold lifecycle, status enums, pricing strategy, concurrency handling.

---

## Splitwise

**Core problem:** Track expenses in a group, calculate who owes whom, minimize the number of transactions to settle.

**Entities:** `User`, `Group`, `Expense`, `ExpenseSplit`, `Balance`, `SplitStrategy`, `SettlementService`

**Patterns used:**
- Strategy: `SplitStrategy` — equal split, exact amounts, percentage split, shares
- Factory: `ExpenseFactory` — creates expense with correct split type

**Key design decisions:**
- Balance is a net amount per user pair — not per expense
- On each expense: update balance map `(userA, userB) → netAmount`
- Settlement minimization: greedy algorithm — find max creditor and max debtor, settle between them, repeat
- `SplitStrategy` interface: `calculateSplits(expense, participants) → List<Split>`

**Settlement algorithm:**
```
while creditors and debtors exist:
  take max creditor (most owed) and max debtor (owes most)
  settle min(credit, debt) between them
  update balances, repeat
```

**Interviewer is testing:** Strategy pattern for split types, balance calculation, settlement minimization algorithm.

---

## Warehouse Management System

**Core problem:** Track inventory across locations, handle inbound/outbound, allocate storage.

**Entities:** `Item`, `Location`, `Inventory`, `InboundOrder`, `OutboundOrder`, `AllocationStrategy`, `WarehouseService`

**Patterns used:**
- Strategy: `AllocationStrategy` — FIFO, FEFO (first expired first out), nearest location
- Observer: notify when stock falls below threshold
- Factory: `OrderFactory` — creates inbound/outbound orders

**Key design decisions:**
- `Inventory` tracks `(itemId, locationId) → quantity` — fine-grained location tracking
- FEFO for perishables: always pick item expiring soonest
- Reservation: reserve inventory on order creation, deduct on dispatch
- `Location` has type (rack, bin, zone) and capacity

**Interviewer is testing:** Strategy for allocation, inventory reservation pattern, FIFO vs FEFO.

---

## Pub-Sub System

**Core problem:** Publishers send messages to topics, subscribers receive messages for their subscribed topics.

**Entities:** `Topic`, `Message`, `Publisher`, `Subscriber`, `Subscription`, `MessageBroker`, `MessageQueue`

**Patterns used:**
- Observer: `MessageBroker` notifies all subscribers when a message is published
- Strategy: `DeliveryStrategy` — at-most-once, at-least-once, exactly-once
- Factory: `TopicFactory` — creates topics with correct configuration

**Key design decisions:**
- `MessageBroker` maintains `topic → List<Subscriber>` map
- Each subscriber has its own queue — decouples delivery speed
- Message has `id`, `topic`, `payload`, `timestamp`, `status`
- Retry on delivery failure with exponential backoff
- Dead letter queue for messages that fail after max retries

**Interviewer is testing:** Observer pattern for pub-sub, per-subscriber queues, retry and DLQ design.

---

## Webhook Delivery System

**Core problem:** Deliver HTTP callbacks to registered URLs reliably, retry on failure.

**Entities:** `Webhook`, `WebhookEvent`, `Delivery`, `DeliveryStatus (enum)`, `RetryPolicy`, `WebhookService`

**Patterns used:**
- Strategy: `RetryPolicy` — exponential backoff, fixed interval
- Observer: event triggers webhook delivery
- Chain of Responsibility: delivery pipeline (validate → sign → send → record)

**Key design decisions:**
- Delivery is async — event stored first, worker delivers
- Retry with exponential backoff: `delay = base * 2^attempt`
- HMAC signature on payload — receiver can verify authenticity
- `Delivery` tracks attempt count, last attempt time, status
- Dead letter after max retries — alert the webhook owner

**Interviewer is testing:** Async delivery, retry with backoff, HMAC signing, delivery status tracking.

---

## Notification System (LLD)

**Core problem:** Send notifications via multiple channels (push, SMS, email) based on user preferences.

**Entities:** `Notification`, `NotificationChannel (enum)`, `NotificationTemplate`, `UserPreference`, `NotificationService`, `ChannelHandler`

**Patterns used:**
- Strategy: `ChannelHandler` interface — `EmailHandler`, `SMSHandler`, `PushHandler`
- Factory: `ChannelHandlerFactory` — returns right handler for channel type
- Observer: events trigger notifications
- Template Method: `NotificationService.send()` — common flow, channel-specific delivery

**Key design decisions:**
- User preferences determine which channels to use
- Template system: `NotificationTemplate` with placeholders, filled at send time
- Rate limiting per user per channel — prevent spam
- Retry on channel failure, fallback to secondary channel

**Interviewer is testing:** Strategy for channels, factory for handler selection, template design.

---

## Cab Booking System

**Core problem:** Match riders to nearby drivers, track trip lifecycle.

**Entities:** `Rider`, `Driver`, `Trip`, `TripStatus (enum)`, `Location`, `MatchingStrategy`, `PricingStrategy`, `TripService`

**Patterns used:**
- Strategy: `MatchingStrategy` (nearest driver, highest rated) and `PricingStrategy` (base fare + distance + time)
- State: `Trip` transitions through states (REQUESTED → MATCHED → STARTED → COMPLETED → CANCELLED)
- Observer: notify rider and driver on status changes
- Factory: `TripFactory` — creates trip with correct initial state

**Key design decisions:**
- Driver location updated periodically, stored in memory (or Redis in real system)
- Matching: find drivers within radius, rank by distance + rating, assign first available
- `TripStatus` state machine: each transition has validation (can't go COMPLETED → STARTED)
- Fare calculation: `baseFare + (distanceRate × km) + (timeRate × minutes) × surgeFactor`

**Interviewer is testing:** State pattern for trip lifecycle, strategy for matching and pricing, observer for notifications.

---

## Food Ordering System

**Core problem:** Browse restaurants, add items to cart, place order, track delivery.

**Entities:** `Restaurant`, `MenuItem`, `Cart`, `CartItem`, `Order`, `OrderStatus (enum)`, `DeliveryAgent`, `OrderService`

**Patterns used:**
- State: `Order` transitions (PLACED → ACCEPTED → PREPARING → OUT_FOR_DELIVERY → DELIVERED)
- Strategy: `DeliveryAssignmentStrategy` — nearest agent, load-balanced
- Observer: notify customer on order status changes
- Factory: `OrderFactory` — creates order from cart

**Key design decisions:**
- Cart is transient — not persisted until order is placed
- Price snapshot at order time — menu price changes don't affect existing orders
- `Order` contains snapshot of items and prices, not references to menu items
- Delivery agent assignment after restaurant accepts

**Interviewer is testing:** State pattern for order lifecycle, price snapshot, observer for status updates.

---

## Logging Framework

**Core problem:** Log messages at different levels, route to different appenders, format output.

**Entities:** `Logger`, `LogLevel (enum)`, `LogMessage`, `Appender`, `Formatter`, `LoggerFactory`

**Patterns used:**
- Chain of Responsibility: log levels — DEBUG → INFO → WARN → ERROR, each handler decides to process or pass up
- Strategy: `Formatter` (JSON, plain text, structured) and `Appender` (console, file, remote)
- Factory: `LoggerFactory.getLogger(className)` — returns logger for that class
- Singleton: `LogManager` — manages all loggers and their configuration

**Key design decisions:**
- Logger hierarchy: `com.app.service` inherits config from `com.app` which inherits from root
- Level filtering: logger only processes messages at or above its configured level
- Multiple appenders per logger — same message can go to console and file
- Async appender: buffer messages, flush in background — reduces latency impact

**Interviewer is testing:** Chain of Responsibility for level filtering, Strategy for formatters/appenders, logger hierarchy.

---

## File System

**Core problem:** Model files and directories, support CRUD operations, navigate the tree.

**Entities:** `FileSystemNode (abstract)`, `File`, `Directory`, `Permission`, `FileSystem`

**Patterns used:**
- Composite: `FileSystemNode` — `File` is leaf, `Directory` is composite. Both support `getSize()`, `delete()`, `rename()`.
- Iterator: traverse directory tree
- Factory: `FileSystemNodeFactory` — creates file or directory

**Key design decisions:**
- `Directory` contains `List<FileSystemNode>` — uniform treatment of files and directories
- `getSize()` on directory recursively sums children — classic composite operation
- Permission model: owner/group/other with read/write/execute bits
- Path resolution: split by `/`, traverse from root

**Interviewer is testing:** Composite pattern, recursive operations on tree, permission modeling.

---

## Digital Wallet (LLD)

**Core problem:** Transfer money between wallets, maintain accurate balances, prevent double spend.

**Entities:** `Wallet`, `Transaction`, `TransactionType (enum)`, `TransactionStatus (enum)`, `LedgerEntry`, `WalletService`

**Patterns used:**
- Strategy: `FraudDetectionStrategy` — rule-based, ML-based
- Observer: notify user on transaction completion
- Factory: `TransactionFactory`

**Key design decisions:**
- Double-entry: every transfer creates two ledger entries (debit source, credit destination)
- Balance = sum of all ledger entries for wallet — never stored directly, always computed
- Idempotency key on every transaction — prevents duplicate transfers
- `TransactionStatus`: PENDING → COMPLETED / FAILED / REVERSED
- Optimistic locking on wallet balance update — check version before commit

**Interviewer is testing:** Double-entry bookkeeping, idempotency, optimistic locking.

---

## Payment System (LLD)

**Core problem:** Process payments through multiple gateways, handle failures, maintain audit trail.

**Entities:** `Payment`, `PaymentStatus (enum)`, `PaymentGateway (interface)`, `PaymentMethod`, `Refund`, `PaymentService`

**Patterns used:**
- Strategy: `PaymentGateway` interface — `StripeGateway`, `RazorpayGateway`, etc.
- Factory: `PaymentGatewayFactory` — selects gateway based on method/region
- Chain of Responsibility: payment processing pipeline (validate → fraud check → charge → record)
- Observer: notify on payment success/failure

**Key design decisions:**
- Gateway abstraction: `PaymentGateway.charge(amount, method) → PaymentResult`
- Idempotency key passed to every gateway call
- `Payment` has status state machine: INITIATED → PROCESSING → COMPLETED / FAILED / REFUNDED
- Retry logic in service layer, not gateway layer
- Audit log: every status transition recorded with timestamp

**Interviewer is testing:** Strategy for gateway abstraction, state machine for payment status, idempotency.

---

## Ledger System (Double Entry Accounting)

**Core problem:** Record all financial transactions with double-entry bookkeeping, ensure books always balance.

**Entities:** `Account`, `AccountType (enum: ASSET, LIABILITY, EQUITY, REVENUE, EXPENSE)`, `JournalEntry`, `LedgerEntry`, `LedgerService`

**Patterns used:**
- Factory: `JournalEntryFactory` — creates balanced journal entries
- Observer: trigger reconciliation on large transactions

**Key design decisions:**
- Every journal entry has equal debits and credits — books always balance
- Debit increases ASSET/EXPENSE, decreases LIABILITY/EQUITY/REVENUE
- Credit increases LIABILITY/EQUITY/REVENUE, decreases ASSET/EXPENSE
- Balance = sum of all entries — never stored, always computed
- Immutable entries — never update or delete, only add reversals

**Interviewer is testing:** Double-entry mechanics, immutable ledger, account type debit/credit rules.

---

## Task Scheduler

**Core problem:** Execute tasks at scheduled times, support recurring tasks, handle failures.

**Entities:** `Task`, `TaskStatus (enum)`, `Schedule`, `RecurrenceRule`, `TaskExecutor`, `SchedulerService`

**Patterns used:**
- Strategy: `ExecutionStrategy` — immediate, delayed, recurring
- Command: `Task` encapsulates the work to be done
- Observer: notify on task completion/failure
- Factory: `TaskFactory` — creates tasks with correct schedule type

**Key design decisions:**
- Priority queue ordered by `nextRunAt` — always execute earliest task first
- Recurring tasks: after execution, compute `nextRunAt` from recurrence rule and re-enqueue
- Retry on failure: `retryCount < maxRetries → reschedule with backoff`
- Task status state machine: PENDING → RUNNING → COMPLETED / FAILED / RETRYING
- Idempotency: task has unique ID, executor checks before running

**Interviewer is testing:** Command pattern for tasks, priority queue scheduling, retry with backoff, recurring task design.

---

## Circuit Breaker

**Core problem:** Prevent cascading failures by stopping calls to a failing service and recovering gracefully.

**Entities:** `CircuitBreaker`, `CircuitState (enum: CLOSED, OPEN, HALF_OPEN)`, `CircuitBreakerConfig`, `CircuitBreakerRegistry`

**Patterns used:**
- State: `CircuitBreaker` transitions between CLOSED, OPEN, HALF_OPEN
- Proxy: `CircuitBreaker` wraps the actual service call
- Strategy: `FailureDetectionStrategy` — count-based, rate-based

**Key design decisions:**
- CLOSED: normal operation, track failure count
- OPEN: reject all calls immediately, start timeout timer
- HALF_OPEN: allow one probe request — if success → CLOSED, if failure → OPEN
- Failure threshold: open after N failures in last M seconds
- Recovery timeout: how long to stay OPEN before trying HALF_OPEN

**State transitions:**
```
CLOSED → OPEN: failure rate exceeds threshold
OPEN → HALF_OPEN: after recovery timeout
HALF_OPEN → CLOSED: probe request succeeds
HALF_OPEN → OPEN: probe request fails
```

**Interviewer is testing:** State pattern for circuit states, transition logic, half-open probe mechanism.

---

## Vending Machine

**Core problem:** Accept coins, dispense items, handle insufficient funds and out-of-stock.

**Entities:** `VendingMachine`, `VendingState (enum)`, `Item`, `Coin`, `Inventory`, `Display`

**Patterns used:**
- State: machine behaves differently in IDLE, HAS_MONEY, DISPENSING, OUT_OF_STOCK states
- Strategy: `CoinAcceptanceStrategy` — which coins are valid
- Singleton: `VendingMachine`

**Key design decisions:**
- State determines what actions are valid — inserting coin in DISPENSING state is rejected
- `HAS_MONEY` state tracks inserted amount
- On selection: check item availability + sufficient funds → dispense + return change
- Change calculation: greedy algorithm with available coin denominations

**State transitions:**
```
IDLE → HAS_MONEY: coin inserted
HAS_MONEY → DISPENSING: item selected + sufficient funds
HAS_MONEY → IDLE: cancel + return coins
DISPENSING → IDLE: item dispensed + change returned
```

**Interviewer is testing:** State pattern, state transition validation, change calculation.

---

## Chess Game

**Core problem:** Model the board, pieces, valid moves, check/checkmate detection.

**Entities:** `Board`, `Piece (abstract)`, `King/Queen/Rook/Bishop/Knight/Pawn`, `Player`, `Move`, `GameState`, `ChessGame`

**Patterns used:**
- Strategy: each `Piece` subclass implements `getValidMoves()` — polymorphism over switch
- Command: `Move` encapsulates a move, supports undo
- Observer: notify on check, checkmate, stalemate
- Factory: `PieceFactory` — creates pieces by type

**Key design decisions:**
- `Piece` is abstract with `getValidMoves(Board board) → List<Position>`
- Each piece knows its own movement rules — no central switch statement
- Check detection: after every move, check if opponent's king is under attack
- Checkmate: no valid moves AND king is in check
- Stalemate: no valid moves AND king is NOT in check

**Interviewer is testing:** Polymorphism for piece movement, check/checkmate detection, Command for move history.

---

## Snake and Ladder Game

**Core problem:** Model the board, players, dice, snakes and ladders, turn-based gameplay.

**Entities:** `Board`, `Player`, `Dice`, `Snake`, `Ladder`, `Cell`, `Game`

**Patterns used:**
- Strategy: `DiceStrategy` — fair dice, loaded dice, multiple dice
- Observer: notify on game events (player wins, hits snake, climbs ladder)
- Factory: `BoardFactory` — creates board with snakes and ladders

**Key design decisions:**
- `Cell` has optional `Snake` or `Ladder` — null means normal cell
- `Snake` has head and tail positions, `Ladder` has bottom and top
- Turn: roll dice → move player → check cell → apply snake/ladder → check win
- Multiple players: queue-based turn management

**Interviewer is testing:** Clean entity modeling, Strategy for dice, Observer for events.

---

## Tic Tac Toe

**Core problem:** Two players take turns, detect win/draw, support different board sizes.

**Entities:** `Board`, `Player`, `Mark (enum: X, O, EMPTY)`, `WinStrategy`, `Game`

**Patterns used:**
- Strategy: `WinStrategy` — row/column/diagonal check, extensible for different win conditions
- Factory: `PlayerFactory` — human or AI player
- Template Method: `Game.play()` — fixed turn loop, pluggable win check

**Key design decisions:**
- `Board` is N×N grid of `Mark` — parameterized for different sizes
- Win check after every move — only check rows/cols/diagonals involving last move
- Draw: board full and no winner
- AI player: minimax algorithm for optimal play

**Interviewer is testing:** Clean separation of board, player, and game logic. Strategy for win condition.

---

## LLD Interview Cheat Sheet

| System | Core entities | Key patterns |
|--------|--------------|--------------|
| Parking Lot | Vehicle, Slot, Ticket, Floor | Strategy (allocation + fee), Singleton, Facade |
| LRU Cache | Node, DLL, HashMap | Strategy (eviction), Singleton |
| Rate Limiter | Bucket/Window, RateLimiter | Strategy (algorithm), Factory |
| Elevator | Elevator, Request, Controller | State, Strategy (dispatch), Observer |
| BookMyShow | Show, Seat, Booking | Strategy (pricing), State (booking), Observer |
| Splitwise | Expense, Balance, Split | Strategy (split type), Factory |
| Warehouse | Item, Location, Inventory | Strategy (allocation), Observer |
| Pub-Sub | Topic, Message, Subscriber | Observer, Strategy (delivery), Factory |
| Webhook | Webhook, Delivery, RetryPolicy | Strategy (retry), Observer, Chain |
| Notification | Notification, ChannelHandler | Strategy (channel), Factory, Observer |
| Cab Booking | Trip, Driver, Rider | State (trip), Strategy (matching + pricing) |
| Food Ordering | Order, Cart, DeliveryAgent | State (order), Strategy (assignment) |
| Logging | Logger, Appender, Formatter | Chain of Responsibility, Strategy, Singleton |
| File System | File, Directory, Node | Composite, Iterator, Factory |
| Digital Wallet | Wallet, LedgerEntry | Double-entry, Observer |
| Payment System | Payment, Gateway | Strategy (gateway), State, Chain |
| Ledger | Account, JournalEntry | Factory, Observer |
| Task Scheduler | Task, Schedule | Command, Strategy, Observer |
| Circuit Breaker | CircuitBreaker, State | State, Proxy, Strategy |
| Vending Machine | VendingMachine, Item, Coin | State, Strategy, Singleton |
| Chess | Board, Piece, Move | Strategy (moves), Command, Observer |
| Snake and Ladder | Board, Player, Dice | Strategy (dice), Observer, Factory |
| Tic Tac Toe | Board, Player, Mark | Strategy (win), Template Method |

---

## Recurring Patterns Across LLD Systems

**State pattern** shows up whenever an entity has a lifecycle. Order, Trip, Payment, Circuit Breaker, Vending Machine, Elevator. The rule: each state handles its own transitions, the context just delegates.

**Strategy pattern** shows up whenever behavior needs to be swappable. Pricing, allocation, matching, channel selection, eviction policy. The rule: if you have an if-else chain choosing between algorithms, extract to Strategy.

**Observer pattern** shows up whenever one thing changing should notify others. Booking confirmation, order status, payment result. The rule: subject maintains a list, notifies all on change.

**Singleton** for the central manager. ParkingLot, LRUCache, LogManager. The rule: use Bill Pugh inner class pattern, not lazy initialization.

**Factory** whenever object creation logic is non-trivial or type-dependent. VehicleFactory, PieceFactory, ChannelHandlerFactory.

**Enums for status/type.** Always use enums for `VehicleType`, `OrderStatus`, `TripStatus`, `CircuitState`. Never use strings.

**Idempotency** for anything that can be retried. Webhook delivery, payment, task execution. Always have a unique ID and check before executing.
