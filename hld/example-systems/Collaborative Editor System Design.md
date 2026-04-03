# Collaborative Editor System Design

## System Overview
A real-time collaborative document editor (think Google Docs) where multiple users simultaneously edit the same document, with changes merged conflict-free via Operational Transformation and persisted durably through an event-sourced op log.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- User registration and authentication
- Create, read, update, delete documents
- Real-time collaborative editing (multiple users, same document, simultaneously)
- Conflict-free merge of concurrent edits via OT
- Document versioning — snapshot on manual save and autosave (every 10/20s)
- Ability to view and restore previous versions

### Non-Functional Requirements
- Availability: 99.99% uptime
- Latency: <100ms for local edit to appear; <300ms for remote user to see it
- Scalability: 1B+ documents, 100M+ DAU, 10M+ concurrent editing sessions
- Consistency: All collaborators converge to the same document state
- Durability: No edit ever lost, even on crash
- Security: Document access enforced on every operation

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 100M DAU, avg 5 documents opened/day
- 10M concurrent active editing sessions at peak
- Each session generates ~20 ops/min while active
- Average op size: 50 bytes
- Average document size: 100KB
- Snapshot on every manual save + autosave every 10–20s

### Traffic
```
Active sessions at peak  = 10M
Ops/sec                  = 10M × 20/60 ≈ 3.3M ops/sec
Peak (2×)                ≈ 6.6M ops/sec
```

### Storage
```
Ops/day    = 10M sessions × 20 ops/min × 60 min × 8 hrs = 96B ops/day
           ≈ 96B × 50B = 4.8TB/day (raw ops log)
           → retain 30 days rolling = ~144TB

Snapshots  = 1B docs × 100KB avg = 100TB (S3)
```

### Memory (Redis)
```
Hot doc canonical copies = top 1M active docs × 100KB = 100GB
Active session metadata  = 10M × 500B = 5GB
```

## 3. Core Components

**LB + API Gateway** — SSL termination, JWT validation, rate limiting, routing for all HTTP requests

**WebSocket LB + Gateway** — Persistent WebSocket connections, auth, rate limiting, consistent hashing on `docId` to Document Editor Service

**Document Metadata Service** — Handles all HTTP requests to create/read/update document metadata; publishes metadata change events to Kafka

**Document Editor Service** — Core real-time engine; receives ops via WebSocket, applies OT, updates Redis canonical copy, publishes ops to Kafka, broadcasts to collaborators

**Metadata Consumer** — Kafka consumer; persists document metadata changes to Cassandra (VersionDB)

**Operation Consumer Service** — Kafka consumer; persists operations to Cassandra (Operations DB)

**Reconciliation Service (Replay)** — Reconstructs the final document state by replaying ops from Cassandra; used for version restore and recovery

**Redis (Canonical Copy)** — In-memory authoritative document state with TTL; the live working copy during active editing

**Kafka** — Async event bus decoupling the editor from persistence; two logical streams: metadata events and operation events

**S3** — Stores document snapshots (initial state and versioned saves)

**Cassandra — Operations DB** — Append-only op log; partition by `docId`, cluster by timestamp

**Cassandra — VersionDB** — Document metadata and version records

**CDN** — Serves static assets and document media

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| Cassandra (Ops DB) | Append-only op log, extremely high write throughput (3M+ ops/sec), time-series, partition by docId |
| Cassandra (VersionDB) | Document metadata and version history, high availability, flexible schema |
| Redis | Sub-ms canonical copy reads, TTL-based lifecycle, in-memory OT state |
| S3 | Cheap durable snapshot storage, CDN-friendly |
| Kafka | Decouples editor from persistence; durable event log for ops and metadata |

### Cassandra — operations

Partition key: `doc_id`, Clustering: `timestamp ASC`

| Field | Type |
|---|---|
| doc_id | UUID (partition key) |
| timestamp | TIMESTAMP (clustering) |
| operation_id | UUID |
| event | TEXT (JSON — type, position, content) |

### Cassandra — version_db (document metadata)

| Field | Type |
|---|---|
| id | UUID (PK) |
| doc_id | UUID |
| version_id | UUID |
| title | VARCHAR |
| url | TEXT (S3 snapshot URL) |
| created_by | UUID |
| created_time | TIMESTAMP |
| last_modified_time | TIMESTAMP |
| metadata | TEXT (JSON) |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `doc:canonical:{docId}` | String | full document content (blob) | TTL — evicted when doc goes inactive |
| `doc:sessions:{docId}` | Set | connected sessionIds | — |
| `session:{sessionId}` | String | `{userId, docId}` | 86400s |

## 5. Key Flows

### 5.1 Auth — Registration & Login

**Register:**
1. Client → API Gateway → User Service
2. Validate, hash password, write to DB
3. Return JWT + refresh token

**Login:**
1. Validate credentials, generate JWT (1hr) + refresh token
2. Store session in Redis
3. Return tokens

**Every request:** API Gateway validates JWT + session on both HTTP and WebSocket connections

### 5.2 Document Create & Open

**Create:**
1. Client → API Gateway → Document Metadata Service
2. Write initial metadata to VersionDB (Cassandra)
3. Write empty initial snapshot to S3
4. Return `docId` to client

**Open:**
1. Client → API Gateway → Document Metadata Service → fetch metadata + latest snapshot URL from VersionDB
2. Client downloads initial document content from S3 (via CDN)
3. Client establishes WebSocket connection to Document Editor Service (routed by `docId`)
4. Document Editor Service loads canonical copy from Redis if present; otherwise loads from S3 snapshot + replays any ops since that snapshot from Cassandra
5. Client begins sending/receiving ops

### 5.3 Real-Time Collaborative Editing (OT Flow)

The core problem: two users edit the same position simultaneously. OT resolves this by transforming concurrent ops so all clients converge to the same state.

```
Doc = "AC", both users at version 5

User A: insert(1, "B")  → "ABC"
User B: insert(1, "D")  → concurrent, same baseVersion

Server applies A first → version 6: "ABC"
Server transforms B against A → insert(2, "D") → version 7: "ABDC"

Both clients converge to "ABDC"
```

**Flow:**
1. User types → client generates op: `{docId, event: {type, position, content}, baseVersion}`
2. Client sends op via WebSocket to Document Editor Service
3. Document Editor Service acquires lock on `docId` (Redis SETNX, short TTL)
4. Fetches current version from Redis canonical copy
5. If `baseVersion < currentVersion`: transform op against all intermediate ops
6. Apply transformed op to Redis canonical copy, increment version
7. Release lock
8. Publish op to Kafka (Operation Consumer persists to Cassandra async)
9. Broadcast transformed op to all other sessions in `doc:sessions:{docId}`
10. ACK sender with new version

**Client-side:** maintains a pending ops queue; transforms pending ops against any incoming remote op before re-applying locally

### 5.4 Snapshotting & Versioning

Two triggers for creating a snapshot:
- Manual save by user
- Autosave every 10–20 seconds by the client

**Flow:**
1. Trigger fires → Document Editor Service reads current canonical copy from Redis
2. Writes snapshot blob to S3: `snapshots/{docId}/{versionId}.json`
3. Publishes metadata event to Kafka (metadata Kafka stream)
4. Metadata Consumer consumes event → writes new version record to VersionDB (Cassandra) with S3 URL, timestamp, versionId

### 5.5 Version Restore (Reconciliation Service)

1. User requests a previous version → Document Metadata Service fetches version list from VersionDB
2. User selects version V → Reconciliation Service loads snapshot from S3 for that version
3. Fetches all ops after that snapshot's timestamp from Cassandra Operations DB
4. Replays ops on top of snapshot → reconstructs exact document state at version V
5. Returns reconstructed document to client (read-only preview)
6. On confirm restore: reconstructed state written back as a new snapshot, flows through normal save path

### 5.6 Document Load After Redis TTL Expiry

When a document's canonical copy has been evicted from Redis (no active sessions):
1. First user opens doc → Document Editor Service cache miss on Redis
2. Fetches latest snapshot URL from VersionDB (Cassandra)
3. Downloads snapshot from S3
4. Fetches all ops after `last_snapshot_timestamp` from Cassandra Operations DB
5. Replays ops → reconstructs current state
6. Writes back to Redis canonical copy with TTL
7. Subsequent users on same doc hit Redis directly

## 6. Key Interview Concepts

### Operational Transformation (OT)
Every edit is an op: `insert(pos, text)` or `delete(pos, length)`. When two ops are concurrent (same `baseVersion`), the server transforms the second op's position to account for the first before applying. Requires a central server to serialize ops — this is why the Document Editor Service holds a per-doc lock.

Example: Doc = "AC"
- User A inserts "B" at pos 1 → "ABC"
- User B (same baseVersion) inserts "D" at pos 1 → without transform: "ADC" ✗
- After transform: insert at pos 2 → "ABDC" ✓

### CRDT as an Alternative
CRDTs assign a globally unique ID to every character; inserts/deletes are always commutative — no central coordinator needed. Used by Figma, Notion. Trade-off: more memory overhead, but better for P2P/offline-first. This design uses OT with a central server (simpler, Google Docs approach).

### Redis as Canonical Copy
Redis holds the live working document state during active editing. It's the single source of truth for the Document Editor Service while a doc is hot. TTL ensures it's evicted when inactive. On eviction, the document is reconstructed from S3 + Cassandra on next open. Redis is a cache — Cassandra + S3 are the durable sources.

### Kafka Decoupling
Document Editor Service doesn't write to Cassandra directly — it publishes to Kafka and returns immediately. This keeps the critical path (OT + Redis update + broadcast) fast. Cassandra writes happen async via consumers. Risk: if Kafka consumer lags, ops aren't persisted yet. Mitigation: monitor consumer lag; Kafka retains messages so nothing is lost.

### Op Log as Event Source
Cassandra Operations DB is an append-only log of every op ever applied. Snapshots are performance shortcuts — the full document can always be reconstructed by replaying all ops from the beginning. This is the event sourcing pattern: state = initial snapshot + all ops since.

### Snapshot + Op Replay
Replaying all ops from version 0 is too slow for large docs. Solution: periodic snapshots to S3. On open: load latest snapshot + replay only ops since that snapshot. Bounds load time regardless of document age or edit history length.

### Consistent Hashing for Document Editor Service
Document Editor Service is stateful (WebSocket connections + in-memory OT state). WebSocket Gateway uses consistent hashing on `docId` so all connections for the same document route to the same instance. Benefits: no cross-instance coordination for most ops, warm in-memory state. On scale-out, only docs in the moved hash range need to migrate.

### Distributed Lock per Document
OT requires serialized op application per document. Redis `SETNX` with short TTL acts as a distributed lock on `docId`. Consistent hashing means the same instance usually handles a doc, so lock contention is rare — it's a safety net for edge cases (instance restart, rebalancing).

### CAP Trade-off
Op processing is CP — consistency required, all collaborators must converge. If lock unavailable, writes fail rather than diverge. Presence/cursor data is AP — slight staleness is fine.

### Idempotency
Each op carries a `client_id` + `baseVersion`. On retry, Document Editor Service checks if the op already exists in the Cassandra log before applying. Prevents duplicate ops from appearing in the document.

## 7. Failure Scenarios

### Document Editor Service Instance Crash
- Detection: WebSocket connections drop, health check fails
- Recovery: clients reconnect with exponential backoff; consistent hashing routes to a new instance; new instance loads canonical copy from Redis (still warm) or reconstructs from S3 + Cassandra; clients resend pending ops from their local queue
- Prevention: Redis holds canonical copy independently of any instance

### Redis Failure (Canonical Copy Lost)
- Impact: all active editing sessions lose in-memory state; effectively a forced reconnect for all users
- Recovery: Redis Sentinel failover (<30s); on reconnect, Document Editor Service reconstructs doc from latest S3 snapshot + Cassandra ops since that snapshot; no data loss since Kafka/Cassandra are durable
- Prevention: Redis Cluster + AOF persistence; autosave every 10–20s limits how much needs to be replayed

### Kafka Consumer Lag (Operation Consumer)
- Impact: ops not yet persisted to Cassandra; if Redis also fails before consumer catches up, recent ops could be lost
- Recovery: Kafka retains messages (configurable retention); consumer catches up on recovery
- Prevention: monitor consumer lag; alert if lag exceeds 10s; Kafka retention set to at least 24hr

### Cassandra Write Failure
- Impact: ops lost if Kafka also fails (double failure)
- Recovery: Cassandra RF=3, QUORUM writes; failed writes retried by consumer with idempotency key
- Prevention: multi-datacenter replication; dead letter queue for failed writes

### Distributed Lock Expiry During Op Processing
- Scenario: Redis lock TTL expires mid-transform → two instances both acquire lock, apply same op twice
- Recovery: idempotency key on op deduplicates; version mismatch causes one instance to reject and retry
- Prevention: lock TTL > max op processing time; monitor lock acquisition latency

### Reconciliation Service Failure
- Impact: version restore unavailable; non-critical path, real-time editing unaffected
- Recovery: stateless service, restarts and retries; S3 and Cassandra data intact
- Prevention: health checks, auto-restart

### Long Reconnect After Disconnect
- Scenario: user disconnects for minutes, reconnects with stale `baseVersion`
- Recovery: Document Editor Service fetches all ops since client's `baseVersion` from Cassandra, transforms client's pending ops against them, returns updated state
- If gap is too large: return full current snapshot from Redis/S3, client reloads

### DDoS / Op Flooding
- Rate limit ops per user per document at WebSocket Gateway
- API Gateway rate limits per IP + per JWT
- Document Editor Service rejects ops exceeding per-doc rate threshold
