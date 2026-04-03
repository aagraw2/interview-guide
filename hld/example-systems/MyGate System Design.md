# MyGate (Gated Community Management) System Design

## System Overview
A gated community management platform (think MyGate / NoBrokerHood) that manages visitor entry/exit, resident notifications, delivery management, staff attendance, and community communication for apartment complexes and gated societies.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Resident registration and apartment mapping
- Visitor pre-approval and entry management (guard app)
- Real-time notification to resident when visitor arrives
- Resident approves/denies entry via app
- Delivery management (package tracking)
- Domestic staff management (maid, cook, driver) with attendance
- Community announcements and notices
- Emergency alerts
- Visitor log and history

### Non-Functional Requirements
- Availability: 99.99% — gate entry must always work (even offline)
- Latency: <2s for visitor notification to reach resident
- Scalability: 10K+ societies, 5M+ residents
- Offline resilience: Guard app must work even without internet
- Security: Visitor data privacy, OTP-based entry

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 10K societies, avg 500 apartments each = 5M residents
- 20 visitor entries/society/day = 200K entries/day
- 50 delivery entries/society/day = 500K deliveries/day
- Peak: morning (8–10am) and evening (5–8pm) — 5× normal

### Traffic
```
Entry events/sec (avg)  = 700K / 86400 ≈ 8/sec
Entry events/sec (peak) = 40/sec

Notifications/sec       = 8 × 2 (entry + exit) = 16/sec
Push notifications      = 16 × 1 resident avg = 16/sec (trivial)
```

### Storage
```
Residents           = 5M × 1KB = 5GB
Visitor logs/day    = 700K × 500B = 350MB/day → ~128GB/year
Staff records       = 5M × 500B = 2.5GB
Photos (visitor)    = 700K/day × 100KB = 70GB/day → S3
```

## 3. Core Components

**API Gateway** — Auth, rate limiting, routing

**Resident Service** — Resident registration, apartment mapping, profile management

**Visitor Service** — Visitor entry/exit management; OTP generation; entry log

**Guard App Backend** — Serves the guard-facing app; handles offline sync; entry/exit recording

**Notification Service** — Real-time push notifications to residents (FCM/APNs); WebSocket for in-app alerts

**Delivery Service** — Package tracking, delivery agent management, resident notification

**Staff Service** — Domestic staff registration, attendance tracking, entry/exit log

**Community Service** — Announcements, notices, polls, emergency alerts

**Pre-approval Service** — Residents pre-approve expected visitors; generates OTP or QR code

**Analytics Service** — Society-level reports, visitor trends, staff attendance reports

**Resident DB (PostgreSQL)** — Residents, apartments, societies, admin roles

**Visitor DB (Cassandra)** — Visitor logs — high write throughput, time-series, partition by societyId

**Staff DB (PostgreSQL)** — Staff profiles, attendance records

**Redis** — Active OTPs (TTL), pre-approvals, session store, real-time entry state

**S3** — Visitor photos, staff photos, community documents

**Kafka** — Entry events, notification fan-out, analytics

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL (Resident DB) | Structured resident/apartment data, ACID, relational |
| Cassandra (Visitor DB) | High write throughput for entry logs, time-series, partition by society |
| PostgreSQL (Staff DB) | Staff profiles and attendance, relational queries |
| Redis | OTP store (TTL), pre-approvals, active sessions |
| S3 | Visitor/staff photos, documents |
| Kafka | Entry events for notifications and analytics |

### PostgreSQL — societies

| Field | Type |
|---|---|
| society_id | UUID (PK) |
| name | VARCHAR |
| address | TEXT |
| city | VARCHAR |
| admin_id | UUID |
| created_at | TIMESTAMP |

### PostgreSQL — apartments

| Field | Type |
|---|---|
| apartment_id | UUID (PK) |
| society_id | UUID (FK → societies) |
| block | VARCHAR |
| flat_number | VARCHAR |
| floor | INT |

### PostgreSQL — residents

| Field | Type |
|---|---|
| resident_id | UUID (PK) |
| apartment_id | UUID (FK → apartments) |
| name | VARCHAR |
| phone | VARCHAR, unique |
| email | VARCHAR |
| role | ENUM (owner / tenant / family) |
| created_at | TIMESTAMP |

### Cassandra — visitor_log

Partition key: `society_id`, Clustering: `entry_time DESC`

| Field | Type |
|---|---|
| society_id | UUID (partition key) |
| entry_time | TIMESTAMP (clustering) |
| visitor_id | UUID |
| visitor_name | VARCHAR |
| visitor_phone | VARCHAR |
| apartment_id | UUID |
| resident_id | UUID |
| entry_type | TEXT (visitor / delivery / staff) |
| status | TEXT (pending / approved / denied / exited) |
| photo_url | TEXT |
| otp | VARCHAR, nullable |
| exit_time | TIMESTAMP, nullable |

### PostgreSQL — staff

| Field | Type |
|---|---|
| staff_id | UUID (PK) |
| name | VARCHAR |
| phone | VARCHAR |
| role | VARCHAR (maid / cook / driver / security) |
| society_id | UUID |
| photo_url | TEXT |
| status | ENUM (active / inactive) |

### PostgreSQL — staff_attendance

| Field | Type |
|---|---|
| attendance_id | UUID (PK) |
| staff_id | UUID (FK → staff) |
| apartment_id | UUID |
| entry_time | TIMESTAMP |
| exit_time | TIMESTAMP, nullable |
| date | DATE |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `otp:{societyId}:{phone}` | String | OTP code | 300s (5 min) |
| `preapproval:{token}` | String | `{residentId, visitorName, validUntil}` | 86400s |
| `session:{sessionId}` | String | userId | 86400s |
| `entry:pending:{apartmentId}` | String | visitor entry JSON | 120s |

## 5. Key Flows

### 5.1 Auth

1. Resident downloads app, registers with phone number → OTP verification → account created
2. Society admin maps resident to apartment
3. JWT issued on login; session stored in Redis
4. Guard app uses separate auth (guard credentials per society)

### 5.2 Visitor Entry Flow

```
Visitor arrives at gate → Guard scans/enters visitor details
                               ↓
                    Visitor Service creates entry record (status=pending)
                               ↓
                    Notification Service sends push to resident
                               ↓
                    Resident approves/denies via app
                               ↓
                    Guard app receives decision in real-time
                               ↓
                    Entry logged; gate opened or denied
```

1. Guard enters visitor name + phone (or scans QR pre-approval)
2. Visitor Service creates entry in Cassandra (`status = pending`)
3. Publishes `VISITOR_ARRIVED` to Kafka
4. Notification Service sends push notification to resident (FCM/APNs)
5. Resident sees notification: "John Doe at gate. Allow?" → taps Allow/Deny
6. Decision sent to Visitor Service → updates entry status
7. Guard app receives real-time update (WebSocket or polling)
8. Gate opened (if approved) or visitor turned away

**OTP-based entry (pre-approved):**
1. Resident pre-approves visitor → Pre-approval Service generates OTP
2. OTP stored in Redis with TTL (5 min or 24hr)
3. Visitor shows OTP at gate → Guard enters OTP
4. Visitor Service validates OTP from Redis → auto-approve without resident notification

### 5.3 Offline Guard App

Critical requirement: gate must work even without internet.

1. Guard app syncs pre-approvals and staff list to local SQLite DB on startup
2. If internet unavailable: guard can still:
   - Verify pre-approved visitors (OTP check against local cache)
   - Log staff entry/exit locally
   - Record visitor entries locally
3. On reconnect: sync local entries to server (conflict resolution: server wins for status, local wins for new entries)

### 5.4 Delivery Management

1. Delivery agent arrives → Guard logs delivery (agent name, company, apartment)
2. Resident notified: "Package from Amazon for Flat 302"
3. Resident can: accept (agent goes up), hold at gate, or reject
4. Package status tracked: arrived → held / delivered
5. Resident can mark package collected

### 5.5 Staff Attendance

1. Staff arrives → Guard app scans staff QR code or enters phone
2. Staff Service records entry: `staff_attendance` row with `entry_time`
3. Resident notified: "Your maid Sunita has arrived"
4. On exit: Guard records exit time
5. Monthly attendance report generated for residents

### 5.6 Community Announcements

1. Society admin posts announcement → Community Service
2. Stored in Community DB
3. Kafka event → Notification Service sends push to all residents in society
4. Residents see announcement in app feed

## 6. Key Interview Concepts

### Offline-First Guard App
The guard app is the most critical component — if it fails, the gate can't function. Offline-first design:
- Pre-approvals and staff list synced to local SQLite on app start
- All entry/exit events written locally first, synced to server when online
- Conflict resolution: last-write-wins for status updates; append-only for new entries

### OTP vs QR Code for Pre-approval
- OTP: 6-digit code, easy to communicate verbally, stored in Redis with TTL
- QR code: scannable, no verbal communication needed, contains signed token
- Both stored in Redis with TTL; validated at gate without resident notification

### Visitor Log Partitioning
Cassandra partition by `society_id` — all entries for a society on one partition. Efficient for "show all visitors today for this society" queries. Clustering by `entry_time DESC` for chronological display. Hot partition risk for very large societies — shard by `society_id + date` if needed.

### Real-time Notification Latency
Resident must see notification within 2s of visitor arriving. Path: Guard app → API → Kafka → Notification Service → FCM → Resident phone. FCM delivery: typically <1s. Total: 1–2s. Acceptable.

### Privacy Considerations
Visitor photos stored in S3 with access control — only the resident and society admin can view. Visitor phone numbers masked in logs after 30 days. GDPR/data retention policies configurable per society.

## 7. Failure Scenarios

### Push Notification Failure (FCM/APNs)
- Impact: resident doesn't receive visitor notification
- Recovery: fallback to SMS (Twilio); guard can call resident directly; entry held as pending
- Prevention: retry push 3 times; SMS as fallback after 30s

### Guard App Offline
- Recovery: offline mode with local SQLite; pre-approvals work; new visitors logged locally and synced on reconnect
- Prevention: offline-first design is the primary design, not a fallback

### Redis Failure (OTP Store Lost)
- Impact: OTP-based pre-approvals fail; residents must manually approve
- Recovery: Redis Sentinel failover (<30s); OTPs lost — residents re-generate or manually approve
- Prevention: Redis Cluster + AOF; OTP loss is a minor UX issue, not a safety issue

### Cassandra Node Failure
- Impact: some visitor log writes may fail
- Recovery: RF=3, QUORUM writes continue; hinted handoff replays on recovery
- Prevention: multi-AZ deployment
