
# DRY, KISS, YAGNI
[DRY, KISS, YAGNI](lld/DRY,%20KISS,%20YAGNI.md)

## SOLID Principles
[SOLID Principles](lld/SOLID%20Principles.md)

## Architecture Styles
[Architecture Styles](lld/Architecture%20Styles.md)

## OOP Concepts
[OOP Concepts](lld/OOP%20Concepts.md)

## Concurrency Basics
[Concurrency Basics](lld/Concurrency%20Basics.md)

## Database Design Basics
[Database Design Basics](lld/Database%20Design%20Basics.md)

## API Design Basics
[API Design Basics](lld/API%20Design%20Basics.md)

## Domain Driven Design
[Domain Driven Design](lld/Domain%20Driven%20Design.md)

## Design Patterns
[Design Patterns](lld/Design%20Patterns.md)

## Example Systems
[LLD Systems](lld/LLD%20Systems.md)

---

# Interview Template

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

Add key attributes:
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

Mention:
- Separation of concerns (each layer has a clear responsibility)

---

## 5. Apply Design Patterns (IMPORTANT)

Ask yourself:
#### Creational Patterns
- Builder → Do I need step-by-step construction with optional configs?
- Singleton → Do I need a single shared instance globally?
- Factory → Is object creation complex or based on conditions?
#### Structural Patterns
- Adapter → Do I need to make incompatible interfaces work together?
- Decorator → Do I want to dynamically add behavior without modifying the base class?
- Facade → Do I need to simplify a complex subsystem?
- Composite → Am I dealing with tree-like structures (part-whole hierarchy)?
#### Behavioral Patterns
- Strategy → Do I have interchangeable logic/algorithms?
- Observer → Do I need event-based notifications to multiple listeners?
- Command → Do I need to encapsulate actions (queue, undo/redo, logging)?
- State → Does behavior change based on internal state?
- Chain of Responsibility → Do I want to pass request through multiple handlers?

Explicitly say:
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

## 11. Golden Lines to Say

- “Let’s separate concerns using service and repository layers”
- “This logic can be abstracted using Strategy pattern”
- “We should make this idempotent”
- “We need locking to avoid race conditions”
- “We can make this async for better performance”