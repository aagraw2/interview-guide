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

**API Gateway** — Auth, rate limiting, routing

**User Service** — Registration, login, profile management

**Course Service** — Course and lecture CRUD; instructor tools; writes to Course DB; CDC to Elasticsearch

**Enrollment Service** — Handles enrollment (free and paid); integrates with Payment Service for paid courses; writes to Enrollment DB

**Video Service** — Video upload pipeline (same as Netflix/YouTube); encoding, S3 storage, CDN delivery; manifest generation

**Progress Service** — Tracks learner progress per lecture, quiz, assignment; writes to Progress DB; triggers certificate generation on completion

**Quiz Service** — Quiz creation, submission, auto-grading; writes to Quiz DB

**Certificate Service** — Generates PDF certificates on course completion; stores to S3; sends to learner

**Search Service** — Course discovery via Elasticsearch; full-text on title, description, instructor; CDC from Course DB

**Review Service** — Course ratings and reviews; async rating recalculation

**Discussion Service** — Per-course forums; threaded discussions; moderation

**Recommendation Service** — Personalized course recommendations based on enrollment history and interests

**Notification Service** — Kafka consumer; enrollment confirmations, assignment deadlines, certificate ready

**Course DB (PostgreSQL)** — Course metadata, lectures, curriculum structure

**Enrollment DB (PostgreSQL)** — Enrollment records, payment status

**Progress DB (Cassandra)** — Per-user per-lecture progress; high write throughput

**Quiz DB (PostgreSQL)** — Quiz definitions, submissions, grades

**S3 + CDN** — Video storage and delivery, certificates, course assets

**Redis** — Course metadata cache, progress cache, session store

**Kafka** — Enrollment events, progress events, certificate triggers, notifications

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL (Course DB) | Structured course hierarchy, ACID, relational |
| PostgreSQL (Enrollment DB) | ACID for enrollment + payment linkage |
| Cassandra (Progress DB) | High write throughput (millions of progress updates/day), partition by userId |
| PostgreSQL (Quiz DB) | Structured quiz data, grading, ACID |
| Redis | Course cache, progress cache, sessions |
| Elasticsearch | Full-text course search |
| S3 + CDN | Video and certificate storage |

### PostgreSQL — courses

| Field | Type |
|---|---|
| course_id | UUID (PK) |
| instructor_id | UUID |
| title | VARCHAR |
| description | TEXT |
| category | VARCHAR |
| level | ENUM (beginner / intermediate / advanced) |
| price | DECIMAL, nullable (null = free) |
| thumbnail_url | TEXT |
| avg_rating | DECIMAL |
| enrollment_count | INT |
| is_published | BOOLEAN |
| created_at | TIMESTAMP |

### PostgreSQL — lectures

| Field | Type |
|---|---|
| lecture_id | UUID (PK) |
| course_id | UUID (FK → courses) |
| title | VARCHAR |
| order_index | INT |
| video_url | TEXT (CDN URL) |
| duration_sec | INT |
| is_preview | BOOLEAN |

### PostgreSQL — enrollments

| Field | Type |
|---|---|
| enrollment_id | UUID (PK) |
| user_id | UUID |
| course_id | UUID |
| status | ENUM (active / completed / refunded) |
| payment_id | UUID, nullable |
| enrolled_at | TIMESTAMP |
| completed_at | TIMESTAMP, nullable |

### Cassandra — progress

Partition key: `user_id`, Clustering: `course_id, lecture_id`

| Field | Type |
|---|---|
| user_id | UUID (partition key) |
| course_id | UUID (clustering) |
| lecture_id | UUID (clustering) |
| watched_seconds | INT |
| completed | BOOLEAN |
| last_watched_at | TIMESTAMP |

### PostgreSQL — quiz_submissions

| Field | Type |
|---|---|
| submission_id | UUID (PK) |
| quiz_id | UUID |
| user_id | UUID |
| answers | JSONB |
| score | DECIMAL |
| passed | BOOLEAN |
| submitted_at | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `course:meta:{courseId}` | String | course JSON | 600s |
| `progress:{userId}:{courseId}` | String | completion % | 300s |
| `session:{sessionId}` | String | userId | 86400s |

## 5. Key Flows

### 5.1 Auth

1. Register → User Service → write to User DB → return JWT
2. Login → validate → JWT (1hr) + refresh token → session in Redis
3. API Gateway validates JWT on every request

### 5.2 Course Enrollment

**Free course:**
1. User clicks Enroll → Enrollment Service
2. Check: not already enrolled
3. Write enrollment record to PostgreSQL
4. Publish `ENROLLMENT_CREATED` to Kafka → Notification Service sends confirmation

**Paid course:**
1. User clicks Enroll → Enrollment Service → Payment Service
2. Payment processed (see Payment System Design)
3. On payment success: write enrollment record
4. Publish event → Notification Service

### 5.3 Video Streaming

Same pipeline as Netflix:
1. Instructor uploads video → Video Service → S3 → encoding pipeline → multiple bitrates
2. Learner opens lecture → Video Service returns CDN manifest URL
3. Client fetches HLS manifest → CDN segments → ABR streaming
4. Progress tracked: client sends progress update every 30s

### 5.4 Progress Tracking

1. Learner watches video → client sends progress update every 30s: `{userId, lectureId, watchedSeconds}`
2. Progress Service writes to Cassandra (partition by userId)
3. Updates Redis cache: `progress:{userId}:{courseId}` = completion %
4. On lecture completion (watched > 90%): mark `completed = true`
5. Check course completion: all lectures + quizzes completed?
6. If complete: publish `COURSE_COMPLETED` to Kafka → Certificate Service

### 5.5 Certificate Generation

1. Certificate Service consumes `COURSE_COMPLETED` event
2. Generates PDF certificate with learner name, course name, date, unique ID
3. Stores to S3: `certificates/{userId}/{courseId}.pdf`
4. Updates enrollment record with certificate URL
5. Notification Service sends email with certificate download link

### 5.6 Quiz Flow

1. Learner submits quiz answers → Quiz Service
2. Auto-grade: compare answers to answer key
3. Write submission to PostgreSQL
4. If passed: mark quiz as completed in Progress DB
5. If failed: allow retry (configurable attempts)
6. Quiz score contributes to course completion criteria

### 5.7 Course Search & Discovery

1. CDC from Course DB → Kafka → Elasticsearch consumer
2. User searches "machine learning" → Search Service → Elasticsearch
3. Full-text on title, description, instructor name; filter by category, level, price, rating
4. Returns ranked results (by relevance × rating × enrollment count)
5. Recommendation Service suggests courses based on enrollment history

## 6. Key Interview Concepts

### Video Delivery at Scale
Same as Netflix — pre-encoded multiple bitrates, HLS/DASH, CDN delivery. Key difference: educational videos are watched repeatedly (popular lectures cached at CDN edge for months). Cache hit rate very high for popular courses.

### Progress Tracking Write Volume
10M DAU × 5 lectures/day × 2 updates/lecture = 100M progress writes/day. Cassandra handles this easily (partition by userId). Redis cache reduces read load for "show my progress" queries.

### Certificate Uniqueness & Verification
Each certificate has a unique ID (UUID). Employers can verify at `coursera.com/verify/{certificateId}`. Certificate Service stores mapping `certificateId → {userId, courseId, completedAt}` in PostgreSQL. PDF generated with QR code linking to verification URL.

### Instructor Analytics
Instructors need enrollment counts, completion rates, revenue. Kafka consumer aggregates events into analytics DB (InfluxDB or PostgreSQL). Dashboard queries pre-aggregated data. Not in real-time critical path.

### Refund Policy
Learner requests refund within 30 days → Enrollment Service checks policy → Payment Service initiates refund → enrollment status = `refunded` → access revoked. Certificate revoked if issued.

### Content Delivery Optimization
Popular courses (top 1000) pre-warmed on CDN. Lecture videos have long TTL (content never changes after upload). Thumbnails and course assets served from CDN. Only metadata queries hit origin servers.

## 7. Failure Scenarios

### Video Encoding Failure
- Recovery: Kafka retains encoding jobs; another encoder picks up; idempotent encoding
- Lecture stays in `processing` status; instructor notified of delay

### Progress DB (Cassandra) Failure
- Impact: progress updates fail; video streaming unaffected
- Recovery: RF=3, QUORUM writes continue; client buffers progress updates locally and retries
- Prevention: multi-AZ Cassandra; progress loss of a few seconds is acceptable

### Certificate Generation Failure
- Impact: certificate not generated immediately
- Recovery: Kafka retains `COURSE_COMPLETED` event; Certificate Service retries on restart
- Prevention: idempotent certificate generation (same courseId + userId = same certificate)

### Payment Failure on Enrollment
- Recovery: enrollment not created; user notified; retry with idempotency key
- Prevention: idempotency key prevents double charge on retry
