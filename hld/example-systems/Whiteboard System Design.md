# Whiteboard (Collaborative Drawing) System Design

## System Overview
A real-time collaborative whiteboard (think Miro / FigJam / Excalidraw) where multiple users simultaneously draw, add shapes, text, and sticky notes on an infinite canvas — with real-time sync, conflict resolution, and persistent state.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Create and manage whiteboards (boards)
- Real-time collaborative drawing (multiple users, same board)
- Drawing tools: pen, shapes, text, sticky notes, images
- Infinite canvas with pan and zoom
- Cursor presence (see where others are)
- Board sharing with permission levels (view / edit)
- Version history and undo/redo
- Export board as image/PDF

### Non-Functional Requirements
- Availability: 99.99%
- Latency: <50ms for local stroke to appear; <200ms for remote user to see it
- Scalability: 1M+ boards, 10M+ users, 100K concurrent collaborative sessions
- Consistency: All collaborators converge to the same board state
- Durability: Board state must never be lost

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 10M DAU, 100K concurrent collaborative sessions at peak
- Each session generates ~100 drawing ops/min (strokes, shapes)
- Average op size: 200B (coordinates, style, type)
- Average board size: 5MB (shapes, text, image references)
- Board snapshot every 5 min or 100 ops

### Traffic
```
Active sessions at peak  = 100K
Ops/sec                  = 100K × 100/60 ≈ 167K ops/sec
Presence updates/sec     = 100K × 2/sec = 200K/sec (cursor moves)
```

### Storage
```
Boards              = 1M × 5MB = 5TB
Op log/day          = 167K × 200B × 86400 = 2.9TB/day (rolling 30 days = 87TB)
Snapshots           = 1M boards × 5MB = 5TB
```

## 3. Core Components

**API Gateway** — Auth, rate limiting, routing

**WebSocket Gateway** — Persistent connections for real-time collaboration; consistent hashing on `boardId`

**Board Service** — Board CRUD, metadata, permissions, sharing; writes to Board DB

**Collaboration Service** — Core real-time engine; receives drawing ops via WebSocket; applies CRDT merge; broadcasts to collaborators; persists to op log

**CRDT Engine** — Conflict-free Replicated Data Type for drawing operations; runs inside Collaboration Service; ensures all clients converge to same state without central locking

**Snapshot Service** — Periodically computes and stores full board snapshots from op log; triggered by Kafka

**Presence Service** — Tracks cursor positions and active users per board; ephemeral, Redis-based

**Export Service** — Renders board to image/PDF on demand; reads from snapshot + recent ops

**Board DB (PostgreSQL)** — Board metadata, permissions, version info

**Op Log (Cassandra)** — Append-only drawing operation log; partition by boardId

**Snapshot Store (S3)** — Full board state snapshots (JSON)

**Redis** — Active board state cache, cursor presence, session store

**Kafka** — Op events for snapshot triggers, analytics

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL (Board DB) | Board metadata, permissions — ACID, relational |
| Cassandra (Op Log) | Append-only ops, high write throughput, time-series, partition by boardId |
| Redis | Active board state cache, cursor presence (ephemeral, TTL) |
| S3 | Board snapshots — large JSON blobs, durable |

### PostgreSQL — boards

| Field | Type |
|---|---|
| board_id | UUID (PK) |
| owner_id | UUID |
| title | VARCHAR |
| latest_version | BIGINT |
| last_snapshot_version | BIGINT |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

### PostgreSQL — board_permissions

| Field | Type |
|---|---|
| board_id | UUID |
| user_id | UUID, nullable |
| access_type | ENUM (owner / editor / viewer) |
| share_link_token | VARCHAR, nullable |

### Cassandra — op_log

Partition key: `board_id`, Clustering: `version ASC`

| Field | Type |
|---|---|
| board_id | UUID (partition key) |
| version | BIGINT (clustering) |
| op_type | TEXT (draw / erase / add_shape / move / delete / text) |
| op_payload | TEXT (JSON — coordinates, style, elementId) |
| author_id | UUID |
| client_id | UUID (dedup) |
| timestamp | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `board:state:{boardId}` | String | full board JSON | while active |
| `board:version:{boardId}` | String | current version | while active |
| `board:sessions:{boardId}` | Set | connected sessionIds | — |
| `presence:{boardId}:{userId}` | String | `{x, y, color, name}` | 5s (heartbeat) |
| `session:{sessionId}` | String | userId | 86400s |

## 5. Key Flows

### 5.1 Auth & Board Open

1. User logs in → JWT issued, session in Redis
2. User opens board → Board Service checks permission
3. Load latest snapshot from S3 (or Redis cache if hot)
4. Load ops since `last_snapshot_version` from Cassandra
5. Apply ops → reconstruct current board state
6. Return board state + current version to client
7. Client connects WebSocket to Collaboration Service (routed by boardId)

### 5.2 Real-Time Drawing (CRDT-based)

Whiteboard uses CRDT instead of OT (unlike Google Docs) because:
- Drawing ops are mostly commutative (adding a shape doesn't depend on position of other shapes)
- No central lock needed — each element has a globally unique ID
- Better for offline-first and P2P scenarios

**CRDT approach for whiteboard:**
- Each element (shape, stroke, text) has a globally unique `elementId` (UUID)
- Operations: `add(elementId, data)`, `move(elementId, newPos)`, `delete(elementId)`, `update(elementId, props)`
- Deletes use tombstones — element marked deleted, never truly removed from CRDT state
- Concurrent moves of same element: last-write-wins by timestamp (acceptable for drawing)

**Flow:**
1. User draws stroke → client generates `elementId` (UUID), creates op
2. Client sends op via WebSocket: `{boardId, elementId, opType, payload, version}`
3. Collaboration Service applies op to board state in Redis
4. Increments version, writes op to Cassandra op log
5. Broadcasts op to all other sessions in `board:sessions:{boardId}`
6. ACK sender with new version
7. Publish to Kafka (for snapshot trigger)

**No lock needed:** each op targets a specific `elementId`. Two users drawing different elements simultaneously → no conflict. Two users moving the same element → last-write-wins by timestamp.

### 5.3 Cursor Presence

1. Client sends cursor position every 100ms via WebSocket
2. Collaboration Service updates Redis: `SET presence:{boardId}:{userId} {x,y,color} EX 5`
3. Broadcasts cursor update to all other sessions
4. TTL = 5s — cursor disappears if user stops moving (idle/disconnected)
5. On board open: fetch all active cursors from Redis

### 5.4 Undo/Redo

Client-side undo/redo stack:
- Each client maintains local undo stack of ops it generated
- Undo: send inverse op (e.g., delete the element that was added)
- Redo: re-send the original op
- Collaborative undo: only undo your own ops (not others')
- Inverse ops go through normal CRDT flow — all clients see the undo

### 5.5 Snapshotting

1. Kafka consumer (Snapshot Service) triggers every 100 ops or 5 min per active board
2. Reads current board state from Redis
3. Serializes to JSON, writes to S3: `snapshots/{boardId}/{version}.json`
4. Updates `last_snapshot_version` in PostgreSQL

### 5.6 Export

1. User requests export → Export Service
2. Loads latest snapshot from S3
3. Applies any ops since snapshot from Cassandra
4. Renders board to canvas (server-side headless browser or canvas library)
5. Exports as PNG/PDF
6. Returns download URL (stored in S3)

## 6. Key Interview Concepts

### CRDT vs OT for Whiteboard
Google Docs uses OT (requires central server, lock per document). Whiteboards use CRDT because:
- Drawing elements are independent — adding shape A doesn't affect shape B
- CRDTs are commutative — order of ops doesn't matter for most drawing ops
- No central lock → lower latency, better offline support
- Trade-off: CRDT has higher memory overhead (tombstones for deleted elements)

### Element-Based vs Pixel-Based
Two approaches to representing whiteboard state:
- **Pixel-based:** store bitmap; simple but large, hard to edit individual elements
- **Element-based (vector):** store shapes, strokes as structured data; smaller, editable, scalable
This design uses element-based — each shape/stroke is a JSON object with coordinates and style.

### Infinite Canvas
Canvas has no fixed size. Client maintains viewport (pan + zoom). Server stores absolute coordinates. Client transforms coordinates based on current viewport. No server-side concept of "canvas size" — just a coordinate space.

### Consistent Hashing for Collaboration Service
Same as Google Docs design — WebSocket Gateway uses consistent hashing on `boardId` to route all connections for the same board to the same Collaboration Service instance. Warm in-memory board state, no cross-instance coordination for most ops.

### Offline Drawing
User draws offline → ops queued locally with `baseVersion = last known`. On reconnect: send queued ops to Collaboration Service. Server fetches ops since `baseVersion` from Cassandra. CRDT merges offline ops with server ops — no conflicts for independent elements. Concurrent moves of same element: last-write-wins.

### Presence Scalability
100K concurrent sessions × 2 cursor updates/sec = 200K Redis writes/sec. Redis handles this easily. Cursor updates are fire-and-forget — no ACK needed. TTL auto-cleans stale cursors.

## 7. Failure Scenarios

### Collaboration Service Crash
- Detection: WebSocket connections drop
- Recovery: clients reconnect; consistent hashing routes to new instance; new instance loads board state from Redis; clients resend pending ops
- Prevention: Redis holds board state independently of any instance

### Redis Failure (Board State Lost)
- Impact: active sessions lose in-memory state; all users effectively disconnected
- Recovery: Redis Sentinel failover (<30s); board state reconstructed from S3 snapshot + Cassandra ops on reconnect
- Prevention: Redis Cluster + AOF; board state is a cache, Cassandra is the truth

### Cassandra Op Log Write Failure
- Recovery: Collaboration Service does not ACK client until Cassandra write succeeds; client retries; CRDT dedup prevents duplicate ops
- Prevention: Cassandra RF=3, QUORUM writes

### Concurrent Move Conflict (Same Element)
- Scenario: two users move the same shape simultaneously
- Recovery: last-write-wins by timestamp; one user's move wins; other user sees their move reverted
- This is acceptable UX for drawing — users can see the conflict and re-move

### Large Board Performance
- Scenario: board with 100K elements; loading takes too long
- Recovery: lazy loading — load only elements in current viewport; load more as user pans
- Prevention: viewport-based rendering; server returns only elements in requested viewport bounds
