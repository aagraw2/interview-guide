## 1. What is Event Sourcing?

Event Sourcing is a pattern where you store all changes to application state as a sequence of events rather than storing just the current state.

```
Traditional (State-Based):
  users table: {id: 1, name: "Alice", email: "alice@example.com"}
  Update overwrites previous state
  No history of changes

Event Sourcing (Event-Based):
  events table:
    {event: "UserCreated", data: {id: 1, name: "Alice"}}
    {event: "EmailUpdated", data: {id: 1, email: "alice@example.com"}}
    {event: "NameUpdated", data: {id: 1, name: "Alice Smith"}}
  
  Current state = replay all events
  Full history preserved
```

**Key principle:** Events are the source of truth, not the current state.

---

## 2. Core Concepts

### Event

An immutable fact about something that happened.

```
{
  "event_id": "evt_123",
  "aggregate_id": "user_456",
  "event_type": "EmailUpdated",
  "timestamp": "2024-01-01T10:00:00Z",
  "version": 5,
  "data": {
    "old_email": "alice@old.com",
    "new_email": "alice@new.com"
  }
}

Event properties:
  - Immutable (never modified or deleted)
  - Past tense (EmailUpdated, not UpdateEmail)
  - Includes all data needed to apply change
  - Versioned (for schema evolution)
```

### Event Store

Append-only log of all events.

```
Event Store:
  - Append-only (no updates or deletes)
  - Ordered by timestamp/sequence
  - Indexed by aggregate_id
  - Supports event replay

Popular implementations:
  - EventStoreDB
  - Kafka (as event log)
  - PostgreSQL (with append-only table)
  - DynamoDB (with sort key for ordering)
```

### Aggregate

A cluster of domain objects treated as a single unit.

```
User Aggregate:
  - Aggregate ID: user_456
  - Events:
      UserCreated
      EmailUpdated
      PasswordChanged
      AccountSuspended
  
  Current state = replay all events for user_456

Order Aggregate:
  - Aggregate ID: order_789
  - Events:
      OrderCreated
      ItemAdded
      ItemRemoved
      OrderPlaced
      PaymentProcessed
      OrderShipped
```

### Projection (Read Model)

Materialized view of current state built from events.

```
Event Store (write):
  UserCreated → EmailUpdated → NameUpdated

Projection (read):
  users table: {id: 1, name: "Alice Smith", email: "alice@new.com"}

Projection is eventually consistent
Rebuilt by replaying events
```

---

## 3. How It Works

### Write Path

```
1. User action: Update email
2. Load events for aggregate (user_456)
3. Replay events to get current state
4. Validate command (business rules)
5. Generate new event (EmailUpdated)
6. Append event to event store
7. Publish event to event bus
8. Return success

Code:
  events = event_store.get_events(aggregate_id="user_456")
  user = User.from_events(events)
  
  if user.can_update_email(new_email):
      event = EmailUpdated(user_id=user.id, new_email=new_email)
      event_store.append(event)
      event_bus.publish(event)
```

### Read Path

```
1. Query projection (read model)
2. Return current state

Code:
  user = users_table.get(user_id=456)
  return user

Projection is updated asynchronously:
  1. Subscribe to events
  2. Apply event to projection
  3. Update read model
```

---

## 4. Benefits

### Complete audit trail

```
Every change is recorded:
  - Who made the change
  - When it happened
  - What changed
  - Why (if included in event)

Use cases:
  - Compliance (SOC2, GDPR)
  - Debugging (what went wrong?)
  - Analytics (user behavior)
```

### Time travel

```
Replay events to any point in time:

What was user's email on Jan 1?
  Replay events up to Jan 1
  
How many orders were placed last month?
  Replay OrderPlaced events for last month

Useful for:
  - Debugging production issues
  - Testing (replay production events in staging)
  - Analytics (historical queries)
```

### Event replay

```
Rebuild projections from scratch:

1. Delete projection
2. Replay all events
3. Projection rebuilt

Use cases:
  - Fix bugs in projection logic
  - Add new projections
  - Migrate to new schema
```

### Natural fit for event-driven architecture

```
Events are already published:
  - Other services can subscribe
  - No additional integration needed
  - Loose coupling

Example:
  EmailUpdated event:
    - User service: Update projection
    - Email service: Send confirmation
    - Analytics service: Track change
```

---

## 5. Implementation Example

### Event Store (PostgreSQL)

```sql
CREATE TABLE events (
    event_id UUID PRIMARY KEY,
    aggregate_id VARCHAR(255) NOT NULL,
    aggregate_type VARCHAR(100) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    version INTEGER NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    UNIQUE(aggregate_id, version)
);

CREATE INDEX idx_aggregate ON events(aggregate_id, version);
```

### Append Event

```python
def append_event(aggregate_id, event_type, event_data, expected_version):
    event = {
        'event_id': str(uuid.uuid4()),
        'aggregate_id': aggregate_id,
        'aggregate_type': 'User',
        'event_type': event_type,
        'event_data': event_data,
        'version': expected_version + 1,
        'timestamp': datetime.utcnow()
    }
    
    try:
        db.execute("""
            INSERT INTO events 
            (event_id, aggregate_id, aggregate_type, event_type, 
             event_data, version, timestamp)
            VALUES (%s, %s, %s, %s, %s, %s, %s)
        """, event.values())
    except UniqueViolation:
        raise ConcurrencyException("Version conflict")
    
    event_bus.publish(event)
    return event
```

### Load Events

```python
def get_events(aggregate_id):
    rows = db.execute("""
        SELECT event_type, event_data, version, timestamp
        FROM events
        WHERE aggregate_id = %s
        ORDER BY version ASC
    """, [aggregate_id])
    
    return [Event(**row) for row in rows]
```

### Rebuild State

```python
class User:
    def __init__(self, user_id):
        self.id = user_id
        self.name = None
        self.email = None
        self.version = 0
    
    @staticmethod
    def from_events(events):
        user = User(events[0].data['user_id'])
        for event in events:
            user.apply(event)
        return user
    
    def apply(self, event):
        if event.event_type == 'UserCreated':
            self.name = event.data['name']
            self.email = event.data['email']
        elif event.event_type == 'EmailUpdated':
            self.email = event.data['new_email']
        elif event.event_type == 'NameUpdated':
            self.name = event.data['new_name']
        
        self.version = event.version
```

### Update Projection

```python
def update_projection(event):
    if event.event_type == 'UserCreated':
        db.execute("""
            INSERT INTO users (id, name, email)
            VALUES (%s, %s, %s)
        """, [event.data['user_id'], event.data['name'], event.data['email']])
    
    elif event.event_type == 'EmailUpdated':
        db.execute("""
            UPDATE users SET email = %s WHERE id = %s
        """, [event.data['new_email'], event.aggregate_id])
    
    elif event.event_type == 'NameUpdated':
        db.execute("""
            UPDATE users SET name = %s WHERE id = %s
        """, [event.data['new_name'], event.aggregate_id])

# Subscribe to events
event_bus.subscribe('User.*', update_projection)
```

---

## 6. Challenges

### Event schema evolution

```
Problem:
  Old events: {event: "EmailUpdated", email: "alice@example.com"}
  New events: {event: "EmailUpdated", old_email: "...", new_email: "..."}
  
  How to handle old events?

Solutions:
  1. Upcasting: Transform old events to new schema on read
  2. Versioning: Include version in event, handle both versions
  3. Weak schema: Use flexible schema (JSON)

Example (upcasting):
  def load_event(raw_event):
      if raw_event.version == 1:
          # Old schema
          return EmailUpdated(new_email=raw_event.data['email'])
      else:
          # New schema
          return EmailUpdated(
              old_email=raw_event.data['old_email'],
              new_email=raw_event.data['new_email']
          )
```

### Eventual consistency

```
Problem:
  Event appended to event store
  Projection not yet updated
  Read returns stale data

Solutions:
  1. Accept eventual consistency (most common)
  2. Read from event store (slow)
  3. Wait for projection update (complex)
  4. Return "processing" status
```

### Event store size

```
Problem:
  Millions of events per aggregate
  Slow to replay all events

Solutions:
  1. Snapshots: Periodic state snapshots
  2. Archiving: Move old events to cold storage
  3. Compaction: Merge events (if possible)

Snapshot example:
  Events: 1-1000 (archived)
  Snapshot at event 1000: {name: "Alice", email: "alice@example.com"}
  Events: 1001-1050 (active)
  
  Rebuild state:
    Load snapshot (event 1000)
    Replay events 1001-1050
```

### Querying

```
Problem:
  Event store is append-only
  Hard to query (e.g., "find all users with email domain @example.com")

Solution:
  Use projections (read models)
  Build multiple projections for different queries
  
Example:
  Projection 1: users table (by user_id)
  Projection 2: users_by_email table (by email)
  Projection 3: users_by_domain table (by email domain)
```

---

## 7. Snapshots

### What are snapshots?

Periodic state checkpoints to avoid replaying all events.

```
Without snapshots:
  Events: 1-10000
  Rebuild state: Replay all 10000 events (slow)

With snapshots:
  Events: 1-9000 (archived)
  Snapshot at event 9000: {current state}
  Events: 9001-10000 (active)
  
  Rebuild state:
    Load snapshot (event 9000)
    Replay events 9001-10000 (fast)
```

### Implementation

```python
def save_snapshot(aggregate_id, state, version):
    db.execute("""
        INSERT INTO snapshots (aggregate_id, state, version, timestamp)
        VALUES (%s, %s, %s, %s)
        ON CONFLICT (aggregate_id) DO UPDATE
        SET state = %s, version = %s, timestamp = %s
    """, [aggregate_id, state, version, datetime.utcnow()])

def load_aggregate(aggregate_id):
    # Load latest snapshot
    snapshot = db.execute("""
        SELECT state, version FROM snapshots
        WHERE aggregate_id = %s
    """, [aggregate_id])
    
    if snapshot:
        user = User.from_snapshot(snapshot.state)
        start_version = snapshot.version + 1
    else:
        user = User(aggregate_id)
        start_version = 1
    
    # Load events after snapshot
    events = db.execute("""
        SELECT * FROM events
        WHERE aggregate_id = %s AND version >= %s
        ORDER BY version ASC
    """, [aggregate_id, start_version])
    
    for event in events:
        user.apply(event)
    
    return user

# Save snapshot every 100 events
if user.version % 100 == 0:
    save_snapshot(user.id, user.to_dict(), user.version)
```

---

## 8. Event Sourcing vs Traditional

### Traditional (CRUD)

```
Pros:
  ✅ Simple to understand
  ✅ Easy to query
  ✅ Immediate consistency
  ✅ Less storage

Cons:
  ❌ No audit trail
  ❌ No time travel
  ❌ Lost history
  ❌ Hard to debug
```

### Event Sourcing

```
Pros:
  ✅ Complete audit trail
  ✅ Time travel
  ✅ Event replay
  ✅ Natural fit for event-driven architecture

Cons:
  ❌ Complex to implement
  ❌ Eventual consistency
  ❌ More storage
  ❌ Schema evolution challenges
```

---

## 9. When to Use Event Sourcing

### Good fit

```
✅ Audit requirements (financial, healthcare)
✅ Complex business logic (need to replay)
✅ Event-driven architecture
✅ Time travel needed (debugging, analytics)
✅ Collaborative editing (conflict resolution)

Examples:
  - Banking (transaction history)
  - E-commerce (order lifecycle)
  - Version control (Git)
  - Collaborative docs (Google Docs)
```

### Poor fit

```
❌ Simple CRUD applications
❌ Immediate consistency required
❌ Complex queries on current state
❌ Team unfamiliar with pattern

Examples:
  - Simple blog
  - Static content management
  - Read-heavy applications
```

---

## 10. Common Interview Questions + Answers

### Q: What is event sourcing and why would you use it?

> "Event sourcing stores all changes as a sequence of events rather than just the current state. Instead of updating a user's email in place, you append an EmailUpdated event. The current state is derived by replaying all events. You'd use it when you need a complete audit trail, like in financial systems, or when you need time travel capabilities for debugging. It's also a natural fit for event-driven architectures since events are already being published."

### Q: How do you handle event schema evolution?

> "Use versioning in events and upcasting on read. Include a version field in each event. When reading old events, transform them to the new schema. For example, if EmailUpdated originally had just 'email' but now needs 'old_email' and 'new_email', the upcaster fills in old_email as null for old events. Alternatively, use weak schemas like JSON that can handle missing fields gracefully."

### Q: What are the main challenges with event sourcing?

> "First, eventual consistency — projections lag behind events, so reads may be stale. Second, event store size grows indefinitely, so you need snapshots to avoid replaying millions of events. Third, schema evolution is tricky since events are immutable. Fourth, querying is hard since the event store is append-only, so you need multiple projections for different query patterns. Finally, it's complex to implement and requires team buy-in."

### Q: How do you handle concurrency in event sourcing?

> "Use optimistic locking with version numbers. Each event has a version that increments. When appending an event, check that the expected version matches the latest version in the store. If not, someone else modified the aggregate concurrently, so reject the write and let the client retry. This is similar to compare-and-swap. The database enforces uniqueness on (aggregate_id, version) to prevent race conditions."

---

## 11. Quick Reference

```
What is Event Sourcing?
  Store all changes as events (append-only log)
  Current state = replay all events
  Events are source of truth

Core concepts:
  Event: Immutable fact (past tense)
  Event Store: Append-only log
  Aggregate: Cluster of domain objects
  Projection: Materialized view (read model)

How it works:
  Write: Append event to event store
  Read: Query projection (eventually consistent)
  Rebuild: Replay events to reconstruct state

Benefits:
  - Complete audit trail
  - Time travel (replay to any point)
  - Event replay (rebuild projections)
  - Natural fit for event-driven architecture

Challenges:
  - Eventual consistency
  - Event store size (use snapshots)
  - Schema evolution (versioning, upcasting)
  - Querying (use projections)

Snapshots:
  Periodic state checkpoints
  Avoid replaying all events
  Save every N events (e.g., 100)

When to use:
  ✅ Audit requirements
  ✅ Time travel needed
  ✅ Event-driven architecture
  ✅ Complex business logic
  
  ❌ Simple CRUD
  ❌ Immediate consistency required
  ❌ Complex queries on current state

Best practices:
  - Use past tense for events
  - Include version and timestamp
  - Make events immutable
  - Use snapshots for performance
  - Build multiple projections
  - Handle schema evolution
  - Monitor projection lag
```
