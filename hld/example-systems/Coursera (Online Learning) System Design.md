# Coursera (Online Learning Platform) System Design

## System Overview
An online learning platform (think Coursera / Udemy / edX) where instructors create courses with video lectures, quizzes, and assignments — and learners enroll, watch videos, complete assessments, and earn certificates.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- User registration (learners and instructors)
- Course creation with video lectures, quizzes, assignments
- Course enrollment (free and paid)
- Video streaming with progress tracking
- Quizzes and auto-graded assignments
- Certificate generation on course completion
- Course search and discovery
- Reviews and ratings
- Discussion forums per course
- Instructor analytics (enrollment, completion rates)

### Non-Functional Requirements
- Availability: 99.99%
- Latency: <200ms for course browse; video starts within 1s
- Scalability: 100M+ learners, 100K+ courses, Read >> Write
- Durability: Progress and certificates must never be lost
- Video delivery: adaptive bitrate, global CDN

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 100M registered users, 10M DAU
- 100K courses, avg 20 video lectures each = 2M videos
- Average video: 10 min at 5Mbps = ~375MB per quality variant
- 5M video views/day, avg 8 min watched
- 100K course enrollments/day

### Traffic
```
Video views/sec         = 5M / 86400 ≈ 58/sec
Concurrent streams      = 58 × 8min × 60s = ~28K concurrent streams → CDN

Course browse/sec       = 10M × 10 / 86400 ≈ 1.2K/sec
Enrollment/sec          = 100K / 86400 ≈ 1.2/sec (low write)
```

### Storage
```
Videos (encoded)        = 2M × 375MB × 5 variants = 3.75PB → S3
Course metadata         = 100K × 50KB = 5GB
User progress           = 100M × 100 courses × 200B = 2TB
Certificates            = 10M × 100KB = 1TB → S3
```

## 3. Core Components

**API Gateway** — Auth, rate limiting, routing for students, instructors, and admins

**User Service** — Registration, login, profile management for students and instructors; writes to User DB

**Catalog Service** — Serves course catalog browsing (list, filter, sort); reads from Course DB; separate from search — handles structured browsing, not full-text

**Course Service** — Course and lecture CRUD for instructors; manages course status workflow (DRAFT → IN_REVIEW → PUBLISHED); writes to Course DB; CDC to Elasticsearch

**Course Moderator Service** — Admin/moderator reviews submitted courses; approves or rejects; updates course status; notifies instructor of decision

**Media Uploader Service** — Handles instructor video uploads; transcoding pipeline (multiple bitrates); thumbnail generation; stores to S3; updates lecture with CDN URL

**Video Playback Service** — Handles student video playback; validates enrollment/access; returns CDN manifest URL; separate from upload pipeline

**Enrollment Service** — Handles enrollment (free and paid); integrates with Payment Service; writes to Course Enrollment DB; triggers Permission Sync Service

**Permission Sync Service** — Syncs access rights after enrollment; writes to Access table; manages access types (LIFETIME / SUBSCRIPTION) and status (ACTIVE / REVOKED / EXPIRED / DEMO)

**User Progress Service** — Consumes user tracking events from Kafka; writes to Progress DB; tracks max watched position and last position separately

**Assignment Service** — Assignment creation, submission, grading (manual or auto); separate from Quiz Service

**Quiz Service** — Quiz and question management; auto-grading for MCQ/TRUE_FALSE; writes to Quiz DB

**Comment & Review Service** — Course ratings, reviews, and discussion comments; async rating recalculation on Course DB

**Course Search Service** — Full-text search via Elasticsearch; CDC from Course DB via Aggregator CDC pipeline

**Payment Service** — Payment processing for paid enrollments; writes to Payment DB; see Payment System Design

**Certificate Service** — Generates PDF certificates on course completion; stores to S3

**Notification Service** — Kafka consumer; enrollment confirmations, course approval/rejection, certificate ready

**Course DB (PostgreSQL)** — Course metadata, lectures, curriculum; includes course status and stats

**Quiz DB (PostgreSQL)** — Quiz definitions, questions, submissions

**Course Enrollment DB (PostgreSQL)** — Enrollment records, orders, payment linkage

**Access DB (PostgreSQL)** — Permission Sync records; access type and status per user per course

**Progress DB (MySQL)** — Per-user per-video progress with max and last position

**User DB (PostgreSQL)** — User and instructor profiles

**Payment DB (PostgreSQL)** — Payment records, orders

**S3 + CDN** — Video storage (chunks + rendered format), thumbnails, attachments, certificates

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL (Course DB) | Structured course hierarchy, ACID, relational, course status workflow |
| PostgreSQL (Enrollment / Access DB) | ACID for enrollment + access rights; relational |
| MySQL (Progress DB) | Per-user per-video progress; high write throughput; simple key-value-like access |
| PostgreSQL (Quiz DB) | Structured quiz/question data, grading, ACID |
| Redis | Course cache, session store |
| Elasticsearch | Full-text course search |
| S3 + CDN | Video chunks, thumbnails, certificates |

### PostgreSQL — courses

| Field | Type |
|---|---|
| course_id | UUID (PK) |
| owner_instructor_id | UUID |
| title | VARCHAR |
| description | TEXT |
| category_id | UUID |
| level | ENUM (BEGINNER / INTERMEDIATE / ADVANCED) |
| thumbnail_url | TEXT |
| avg_video_count | INT |
| status | ENUM (DRAFT / IN_REVIEW / PUBLISHED / REJECTED / ARCHIVED) |
| metadata | JSONB |

### PostgreSQL — course_stats (denormalized)

| Field | Type |
|---|---|
| course_id | UUID (PK) |
| avg_rating | DECIMAL |
| rating_count | INT |
| metadata | JSONB |

### PostgreSQL — instructor_profile

| Field | Type |
|---|---|
| user_id | UUID (PK) |
| name | VARCHAR |
| bio | TEXT |
| email | VARCHAR |
| password_hash | VARCHAR (encrypted) |
| course_count | INT |
| avg_rating | DECIMAL |
| metadata | JSONB |

### PostgreSQL — course_pricing

| Field | Type |
|---|---|
| course_id | UUID (PK/FK) |
| price_amount | DECIMAL |
| currency | VARCHAR |

### PostgreSQL — access (Permission Sync)

| Field | Type |
|---|---|
| access_id | UUID (PK) |
| user_id | UUID |
| course_id | UUID |
| access_type | ENUM (LIFETIME / SUBSCRIPTION) |
| status | ENUM (ACTIVE / REVOKED / EXPIRED / DEMO) |
| purchase_date | TIMESTAMP |
| metadata | JSONB |

### PostgreSQL — orders

| Field | Type |
|---|---|
| order_id | UUID (PK) |
| user_id | UUID |
| course_id | UUID |
| amount_total | DECIMAL |
| currency | VARCHAR |
| status | VARCHAR |
| metadata | JSONB |

### MySQL — progress

| Field | Type |
|---|---|
| user_id | UUID |
| course_id | UUID |
| video_id | UUID (if lesson is video) |
| duration_sec | INT |
| max_position_sec | INT (max watched point — for completion tracking) |
| last_position_sec | INT (resume point) |
| completed | BOOLEAN |
| metadata | JSONB |

### PostgreSQL — quizzes

| Field | Type |
|---|---|
| quiz_id | UUID (PK) |
| course_id | UUID |
| lesson_id | UUID |
| rating_count | INT |
| status | VARCHAR |
| points | INT |

### PostgreSQL — questions

| Field | Type |
|---|---|
| question_id | UUID (PK) |
| quiz_id | UUID |
| type | ENUM (MCQ / MULTI_SELECT / TRUE_FALSE / TEXT) |
| status | VARCHAR |
| points | INT |
| pass_score | DECIMAL |
| metadata | JSONB |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `course:meta:{courseId}` | String | course JSON | 600s |
| `access:{userId}:{courseId}` | String | access status | 300s |
| `session:{sessionId}` | String | userId | 86400s |

## 5. Key Flows

### 5.1 Auth

1. Register → User Service → write to User DB → return JWT
2. Login → validate → JWT (1hr) + refresh token → session in Redis
3. API Gateway validates JWT on every request
4. Instructor endpoints require `role = instructor`

### 5.2 Course Creation & Moderation Workflow

```
Instructor creates course → Course Service (status = DRAFT)
                                ↓
Instructor submits for review → status = IN_REVIEW
                                ↓
Course Moderator Service reviews content
                                ↓
Approved → status = PUBLISHED → Catalog + Search indexed
Rejected → status = REJECTED → instructor notified with reason
```

1. Instructor creates course with title, description, pricing → `status = DRAFT`
2. Instructor uploads videos → Media Uploader Service → S3 → transcoding → CDN URL stored in lecture
3. Instructor submits for review → `status = IN_REVIEW`
4. Course Moderator Service (admin) reviews content quality, policy compliance
5. On approval: `status = PUBLISHED`; CDC → Elasticsearch; Catalog Service serves it
6. On rejection: `status = REJECTED`; Notification Service informs instructor with reason

### 5.3 Course Enrollment & Access

**Free course:**
1. Student clicks Enroll → Enrollment Service
2. Write to Course Enrollment DB (order record)
3. Permission Sync Service creates Access record: `{access_type: LIFETIME, status: ACTIVE}`
4. Kafka event → Notification Service sends confirmation

**Paid course:**
1. Student clicks Enroll → Payment Service → payment gateway
2. On success: write order to Payment DB
3. Enrollment Service writes to Course Enrollment DB
4. Permission Sync Service creates Access record
5. Kafka event → Notification Service

**Access check on every video request:**
- Video Playback Service checks Redis `access:{userId}:{courseId}` → fallback to Access DB
- If `status = ACTIVE`: serve content
- If `status = REVOKED / EXPIRED`: return 403

### 5.4 Video Upload (Instructor)

1. Instructor uploads video → Media Uploader Service
2. Store raw video to S3
3. Trigger transcoding pipeline: encode to 360p / 720p / 1080p
4. Generate thumbnail
5. Store encoded chunks + thumbnail to S3
6. Update lecture record with CDN manifest URL

### 5.5 Video Playback (Student)

1. Student opens lecture → Video Playback Service
2. Validate access (Redis cache → Access DB)
3. Return CDN manifest URL (HLS/DASH)
4. Client fetches segments from CDN → ABR streaming
5. Client sends progress update every 30s: `{userId, videoId, currentPosition}`
6. User Progress Service consumes via Kafka → writes to Progress DB

### 5.6 Progress Tracking

Progress DB stores two position fields:
- `max_position_sec` — furthest point watched (used for completion detection)
- `last_position_sec` — where to resume on next open

1. Client sends `{userId, videoId, currentPosition}` every 30s
2. User Progress Service updates:
   - `last_position_sec = currentPosition` (always)
   - `max_position_sec = max(existing, currentPosition)` (only moves forward)
3. If `max_position_sec >= 0.9 × duration_sec`: mark `completed = true`
4. Check course completion: all required videos + quizzes completed?
5. If complete: publish `COURSE_COMPLETED` to Kafka → Certificate Service

### 5.7 Quiz Flow

1. Student submits quiz → Quiz Service
2. Auto-grade MCQ/MULTI_SELECT/TRUE_FALSE against answer key
3. TEXT answers flagged for manual review
4. If `score >= pass_score`: mark quiz completed in Progress DB
5. If failed: allow retry (configurable max attempts)

### 5.8 Certificate Generation

1. Certificate Service consumes `COURSE_COMPLETED` event
2. Generates PDF with student name, course name, date, unique certificate ID
3. Stores to S3: `certificates/{userId}/{courseId}.pdf`
4. Updates enrollment record with certificate URL
5. Notification Service sends email with download link
6. Verification URL: `coursera.com/verify/{certificateId}` → looks up in PostgreSQL

### 5.9 Course Search & Discovery

1. CDC from Course DB (on status → PUBLISHED) → Aggregator CDC pipeline → Elasticsearch
2. Student searches → Course Search Service → Elasticsearch
3. Full-text on title, description, instructor; filter by category, level, price, rating
4. Catalog Service handles structured browsing (category pages, featured courses)

## 6. Enterprise Clients (White-Label / B2B)

Large companies (e.g., Accenture, TCS) want to offer Coursera's platform to their employees under their own brand — custom subdomain, their own course catalog, employee-only access, and data isolation.

### 6.1 Tenancy Model

Use the **Bridge model** (shared compute, isolated DB per enterprise client):
- Shared application services (Course Service, Video Playback, etc.)
- Separate database schema or DB instance per enterprise client
- Enterprise client identified by subdomain: `accenture.coursera.com`

### 6.2 Custom Subdomain Routing

```
accenture.coursera.com → API Gateway
                              ↓
                    Extract tenantId from subdomain
                    (lookup: accenture → tenantId: T123)
                              ↓
                    Inject tenantId into request context
                    All DB queries scoped to tenantId
```

1. DNS: `*.coursera.com` wildcard CNAME → API Gateway
2. API Gateway extracts subdomain, looks up `tenantId` from Tenant Registry (Redis cache → PostgreSQL)
3. All downstream services receive `tenantId` in request context
4. DB queries always include `WHERE tenant_id = ?`

### 6.3 Data Isolation Options

**Option A — Schema per tenant (Bridge model):**
- Each enterprise gets their own PostgreSQL schema: `tenant_accenture.courses`, `tenant_accenture.progress`
- Shared DB cluster, isolated data
- Schema migrations run per tenant
- Good for 100s of enterprise clients

**Option B — DB per tenant (Silo model):**
- Each enterprise gets a dedicated DB instance
- Maximum isolation — GDPR, data residency compliance
- Higher cost, but enterprise clients often require this
- Good for large enterprises with strict compliance

### 6.4 Enterprise-Specific Features

**SSO Integration:**
- Enterprise clients use their own IdP (Okta, Azure AD, SAML)
- API Gateway validates enterprise SSO token, maps to internal userId
- No separate Coursera password needed for employees

**Custom Course Catalog:**
- Enterprise can have private courses (internal training) + public Coursera courses
- `courses.visibility = PRIVATE` scoped to `tenant_id`
- Public courses shared across all tenants (read-only)

**Employee Access Control:**
- Only employees of the enterprise can access the subdomain
- Validated via SSO or email domain whitelist (`@accenture.com`)
- Access revoked automatically when employee leaves (SSO deprovisioning)

**Analytics & Reporting:**
- Enterprise admin dashboard: completion rates, time spent, quiz scores per employee
- Data scoped to their tenant — they never see other companies' data
- Reports exported as CSV or via API

### 6.5 Tenant Registry

```
PostgreSQL — tenants
  tenant_id       UUID (PK)
  subdomain       VARCHAR, unique  (accenture)
  plan            ENUM (enterprise / team / individual)
  db_schema       VARCHAR          (tenant_accenture)
  sso_config      JSONB            (IdP metadata, SAML config)
  email_domain    VARCHAR          (@accenture.com)
  status          ENUM (active / suspended)
  created_at      TIMESTAMP
```

Redis caches `subdomain → tenantId` with 5min TTL for fast routing on every request.

### 6.6 Video Storage Isolation

Enterprise clients may require their training videos stored in a specific region (data residency):
- S3 bucket per enterprise client in their required region
- Media Uploader Service routes to tenant-specific S3 bucket based on `tenantId`
- CDN configured with tenant-specific origin

## 7. Key Interview Concepts

### Course Status Workflow
`DRAFT → IN_REVIEW → PUBLISHED / REJECTED → ARCHIVED`. Separating moderation from publishing keeps low-quality content off the platform. Course Moderator Service is the gatekeeper. CDC only fires to Elasticsearch on `PUBLISHED` — search index stays clean.

### max_position_sec vs last_position_sec
Two separate fields in Progress DB:
- `last_position_sec`: where to resume — always updated to current position
- `max_position_sec`: furthest point watched — only moves forward, never backward

This prevents a student from scrubbing to the end to mark a video complete. Completion requires `max_position_sec >= 90% of duration`.

### Permission Sync Service
Enrollment and access are separate concerns. Enrollment is a business record (payment, order). Access is a technical permission (can this user watch this video?). Permission Sync Service translates enrollment events into access records. Access types: LIFETIME (one-time purchase), SUBSCRIPTION (monthly), DEMO (free preview).

### Video Delivery at Scale
Same as Netflix — pre-encoded multiple bitrates, HLS/DASH, CDN delivery. Educational videos are watched repeatedly — popular lectures cached at CDN edge for months. Cache hit rate very high for popular courses.

### Certificate Uniqueness & Verification
Each certificate has a unique ID. Employers verify at `coursera.com/verify/{certificateId}`. Certificate Service stores `certificateId → {userId, courseId, completedAt}` in PostgreSQL. PDF has QR code linking to verification URL.

### Enterprise Multi-tenancy
Bridge model (shared compute, isolated DB schema) balances cost and isolation for most enterprise clients. Silo model (dedicated DB) for large enterprises with strict compliance. Subdomain routing via wildcard DNS + tenant registry lookup. SSO integration removes password management burden.

## 8. Failure Scenarios

### Video Encoding Failure
- Recovery: Kafka retains encoding jobs; another encoder picks up; idempotent encoding
- Lecture stays in `processing` status; instructor notified of delay

### Progress DB Failure
- Impact: progress updates fail; video streaming unaffected
- Recovery: client buffers progress updates locally and retries on reconnect
- Prevention: multi-AZ MySQL; progress loss of a few seconds is acceptable

### Certificate Generation Failure
- Recovery: Kafka retains `COURSE_COMPLETED` event; Certificate Service retries on restart
- Prevention: idempotent generation (same courseId + userId = same certificate)

### Payment Failure on Enrollment
- Recovery: enrollment not created; user notified; retry with idempotency key
- Prevention: idempotency key prevents double charge on retry

### Enterprise SSO Outage
- Impact: enterprise employees can't log in
- Recovery: fallback to Coursera credentials if configured; alert enterprise IT admin
- Prevention: SSO is enterprise's responsibility; Coursera can't control IdP availability
