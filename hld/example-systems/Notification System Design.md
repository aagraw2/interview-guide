# Notification System Design

## System Overview
A scalable notification delivery system that sends push notifications, emails, and SMS to users across multiple channels — handling high throughput, user preferences, deduplication, retry logic, and delivery tracking. Used as an internal platform service by other systems (order updates, chat messages, marketing campaigns).

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Send notifications via multiple channels: push (FCM/APNs), email, SMS
- Support notification types: transactional (order confirmed) and marketing (promotions)
- User notification preferences (opt-in/out per channel per type)
- Deduplication — same notification not sent twice
- Retry on delivery failure
- Delivery status tracking (sent / delivered / failed / opened)
- Scheduled and bulk notifications (marketing campaigns)
- Rate limiting per user (no spam)
- Template management for notification content

### Non-Functional Requirements
- Throughput: 1M+ notifications/sec for bulk campaigns
- Latency: <1s for transactional notifications; minutes acceptable for marketing
- Availability: 99.99%
- Durability: No notification lost before delivery attempt
- Scalability: 1B+ users, multiple notification channels

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 1B users
- 10M transactional notifications/day (order updates, alerts)
- 100M marketing notifications/day (campaigns, promotions)
- Channel split: 60% push, 30% email, 10% SMS
- Retry rate: 5% of notifications fail first attempt

### Traffic
```
Total notifications/day     = 110M
Notifications/sec (avg)     = 110M / 86400 ≈ 1.3K/sec
Campaign burst              = 100M notifications in 1hr = 27.8K/sec

Push (FCM/APNs)/sec         = 27.8K × 0.6 = 16.7K/sec
Email/sec                   = 27.8K × 0.3 = 8.3K/sec
SMS/sec                     = 27.8K × 0.1 = 2.8K/sec
```

### Storage
```
Notification records/day    = 110M × 300B = 33GB/day → ~12TB/year
User preferences            = 1B × 500B = 500GB
Templates                   = thousands × 10KB = small
Delivery logs               = 110M × 200B = 22GB/day
```

## 3. Core Components

**API Gateway** — Auth, rate limiting, routing, round-robin for notification submission API

**Notification Service** — Receives notification requests from internal services; validates; writes to Notification DB and Notification Outbox in the same DB transaction; CDC publishes outbox entries to Kafka (outbox pattern — prevents lost events on crash)

**Template Service** — Manages versioned notification templates with `is_active` flag; renders content with user-specific variables; reads from Template DB

**User Preference Service** — Manages user notification preferences per channel per type; publishes preference changes to Kafka `user_preference` topic; User Preference Consumer updates UserPref DB

**OTP Service** — Generates and validates one-time passwords for auth flows; separate from general notification pipeline; uses SMS/email channel workers

**Delivery Consumer** — Kafka consumer on `delivery_status` topic; receives delivery receipts from channel providers; updates Notification DB status

**User Preference Consumer** — Kafka consumer on `user_preference` topic; updates UserPref DB; ensures preference changes are durable even if direct DB write fails

**Channel Providers:**
- SMS Service Provider → Twilio / MSG91
- Email Service Provider → SendGrid / Amazon SES
- In-App Service Provider → FCM / APNs

**Reporting Service** — Consumes notification events; writes to analytics store (BigQuery); powers delivery rate, open rate, campaign performance dashboards

**Retry Service** — Monitors failed notifications; re-queues with exponential backoff to retry topics

**Notification DB (Cassandra)** — Notification records and delivery status; partition by `client_id`

**Notification Outbox** — Transactional outbox table (same DB as Notification DB); CDC agent reads and publishes to Kafka; guarantees at-least-once Kafka delivery

**Template DB (PostgreSQL)** — Versioned templates with active/inactive flag

**UserPref DB (PostgreSQL)** — User preferences per channel; `preferences: {"email": true, "sms": false, "push": true}`

**UserPref Cache (Redis)** — Cached user preferences; checked before every notification send

**Kafka** — Three-tier topic structure:
- `notifications.critical.email/sms/push` — OTP, security alerts (highest priority)
- `notifications.standard.email/sms/push` — transactional (order updates, receipts)
- `notifications.promotional.email/sms/push` — marketing campaigns
- `notifications.bulk.email` — bulk sends
- `notifications.retry` — failed notifications pending retry
- `notifications.dlq` — dead letter queue (exhausted retries)
- `user_preference` — preference update events
- `delivery_status` — delivery receipts from providers

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| Cassandra (Notification DB) | High write throughput for notification records and delivery logs; time-series; partition by client_id |
| PostgreSQL (UserPref DB) | Structured preferences per channel — ACID, relational |
| PostgreSQL (Template DB) | Versioned templates, is_active flag, structured |
| Redis (UserPref Cache) | Sub-ms preference lookup on every notification send |
| Kafka | Three-tier priority × channel topic matrix; durable queue; independent scaling per topic |

### Cassandra — notifications

Partition key: `client_id`

| Field | Type |
|---|---|
| notification_id | UUID (PK) |
| client_id | UUID (partition key) |
| external_user_id | UUID |
| template_id | UUID |
| channel | ENUM (push / email / sms / in_app) |
| payload | JSONB |
| status | ENUM (PENDING / SCHEDULED / SENT / DELIVERED / FAILED / CANCELLED) |
| priority | ENUM (high / normal / low) |
| scheduled_at | TIMESTAMP, nullable |
| last_updated_at | TIMESTAMP |
| metadata | JSONB |

### PostgreSQL — templates

| Field | Type |
|---|---|
| template_id | UUID (PK) |
| name | VARCHAR |
| type | ENUM (transactional / promotional) |
| channel | ENUM (push / email / sms) |
| content | TEXT |
| variables | JSONB (list of variable names) |
| version | INT |
| is_active | BOOLEAN |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

### PostgreSQL — user_preferences

| Field | Type |
|---|---|
| id | UUID (PK) |
| client_id | UUID |
| external_user_id | UUID |
| preferences | JSONB (`{"email": true, "sms": false, "push": true}`) |
| updated_at | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `userpref:{userId}` | String | preferences JSON | 300s |
| `dedup:{idempotencyKey}` | String | notificationId | 86400s |
| `rate:{userId}` | Counter | notifications sent in window | 3600s |

## 5. Key Flows

### 5.1 Transactional Notification (e.g., Order Confirmed)

```
Order Service → Notification Service
                        ↓
    Validate + dedup check (Redis idempotency key)
                        ↓
    DB transaction:
      Write to Notification DB (status=PENDING)
      Write to Notification Outbox
                        ↓
    CDC reads Outbox → publishes to Kafka
    (notifications.standard.email / sms / push)
                        ↓
    Channel worker consumes:
      Check UserPref Cache → UserPref DB
      If channel enabled: call provider
                        ↓
    Provider delivers → publishes to delivery_status topic
                        ↓
    Delivery Consumer updates Notification DB (status=DELIVERED)
```

1. Caller sends `POST /notify` with `{clientId, userId, templateId, channel, payload, priority, idempotencyKey}`
2. Notification Service checks Redis dedup: `SET dedup:{key} 1 NX EX 86400` — if exists, return cached result
3. DB transaction: write to Notification DB (`status = PENDING`) + write to Outbox table atomically
4. CDC agent reads Outbox, publishes to appropriate Kafka topic based on `priority × channel`
5. Channel worker consumes, checks UserPref Cache for user's channel preference
6. If opted out: skip, update status to `CANCELLED`
7. If opted in: render template, call provider (FCM/APNs/SendGrid/Twilio)
8. Provider delivers and sends receipt → published to `delivery_status` topic
9. Delivery Consumer updates Notification DB status

### 5.2 Why the Outbox Pattern
Risk without outbox: Notification Service writes to DB, then crashes before publishing to Kafka → notification record exists but never sent. With outbox:
- DB write + outbox write are in the same transaction — both succeed or both fail
- CDC reads the outbox and publishes to Kafka independently
- Even if service crashes after DB write, CDC will still publish
- Guarantees at-least-once Kafka delivery

### 5.3 Three-Tier Kafka Topics

```
Priority × Channel matrix:

notifications.critical.push   ← OTP, security alerts
notifications.critical.sms
notifications.critical.email

notifications.standard.push   ← order updates, receipts
notifications.standard.sms
notifications.standard.email

notifications.promotional.push ← marketing campaigns
notifications.promotional.sms
notifications.promotional.email

notifications.bulk.email       ← bulk sends (newsletters)
notifications.retry            ← failed, pending retry
notifications.dlq              ← dead letter (exhausted retries)
```

Workers drain `critical` topics before `standard`; `standard` before `promotional`. This ensures OTPs are never delayed by a marketing campaign burst.

### 5.4 OTP Flow

1. User requests OTP (login, transaction confirmation)
2. OTP Service generates 6-digit code, stores in Redis with 5min TTL
3. Publishes to `notifications.critical.sms` or `notifications.critical.email`
4. Channel worker delivers immediately (critical priority, no queue wait)
5. User submits OTP → OTP Service validates against Redis value
6. On success: delete Redis key (single use)

### 5.5 User Preference Update

1. User updates preferences in app → User Preference Service
2. Publishes to Kafka `user_preference` topic
3. User Preference Consumer consumes → updates UserPref DB
4. Invalidates Redis UserPref Cache: `DEL userpref:{userId}`
5. Next notification for this user fetches fresh preferences

Using Kafka for preference updates (rather than direct DB write) ensures durability — if the DB write fails, the event is retained in Kafka and retried.

### 5.6 Bulk Campaign

1. Marketing team creates campaign with template, segment, schedule
2. At scheduled time: campaign job publishes user batches to `notifications.promotional.*` topics
3. Channel workers process at their own rate (rate-limited to avoid provider throttling)
4. Progress tracked via Reporting Service consuming notification events

### 5.7 Retry Logic

1. Channel worker fails → publishes to `notifications.retry` with `retry_count`
2. Retry Service applies exponential backoff: 1min, 5min, 30min, 2hr
3. After max retries (3): publish to `notifications.dlq`, update status = `FAILED`
4. Dead letter: alert ops, available for manual re-trigger

## 6. Key Interview Concepts

### Outbox Pattern — Guaranteed Kafka Delivery
The classic problem: service writes to DB, then crashes before publishing to Kafka → event lost. Solution: write to DB and an Outbox table in the same transaction. CDC agent reads the Outbox and publishes to Kafka independently. Even if the service crashes, CDC will publish. Guarantees at-least-once delivery without distributed transactions.

### Three-Tier Priority × Channel Topics
Flat topic structure (`notifications.email`) means a marketing campaign burst delays OTPs. Priority × channel matrix solves this:
- Workers drain `critical` before `standard` before `promotional`
- Each topic scales independently
- OTP (critical) is never blocked by a 100M-user campaign (promotional)

### Idempotency
Same notification must not be sent twice on retry. `idempotency_key = hash(sourceEventId + userId + channel)` checked in Redis before processing. If key exists: return original result. TTL = 24hr covers the retry window.

### Channel Abstraction
Callers specify `{userId, templateId, priority}` — not the channel. Notification Service decides channels based on user preferences. Decouples callers from channel implementation. Adding a new channel (WhatsApp) requires no changes to callers.

### Template Versioning
Templates have a `version` field and `is_active` flag. Multiple versions can exist; only `is_active = true` version is used. Allows A/B testing of notification copy and safe rollback if a template causes issues.

### OTP as Critical Priority
OTPs must be delivered in seconds — user is waiting at a login screen. Publishing to `notifications.critical.sms` ensures it's processed before any standard or promotional notification. OTP Service also stores the code in Redis with TTL for fast validation.

### Rate Limiting Per User
Prevent notification spam: max N notifications/hr per user. Redis counter: `INCR rate:{userId}` with 1hr TTL. Transactional and critical notifications bypass rate limit; promotional notifications are rate-limited.

### Delivery Tracking via Separate Consumer
Delivery Consumer is separate from channel workers — it only handles status updates from providers. This separation means delivery tracking scales independently and a slow status update doesn't block new sends.

## 7. Failure Scenarios

### Notification Service Crash After DB Write (Before Kafka Publish)
- Without outbox: notification record exists but never sent
- With outbox: CDC reads outbox and publishes to Kafka regardless of service state
- Recovery: automatic — CDC catches up on restart

### FCM/APNs Outage
- Impact: push notifications not delivered
- Recovery: retry queue with backoff; fall back to SMS/email if configured; alert ops
- Prevention: circuit breaker on push worker; monitor provider status

### Email Provider Rate Limit
- Detection: SendGrid returns 429
- Recovery: exponential backoff retry; switch to backup provider (SES)
- Prevention: stay within provider limits; spread campaign sends over time

### Kafka Consumer Lag
- Impact: notification delivery delayed
- Recovery: scale up channel workers; Kafka retains messages; critical topics prioritized
- Prevention: monitor consumer lag per topic; auto-scale on lag threshold

### User Preference Cache Stale
- Scenario: user opts out but Redis cache still shows opted-in → notification sent
- Recovery: short TTL (5min) limits window; on opt-out, actively invalidate cache via User Preference Consumer
- Prevention: cache invalidation on preference change; TTL as safety net

### Dead Letter Queue Buildup
- Scenario: provider outage causes many notifications to exhaust retries
- Recovery: ops team re-triggers after provider recovery; bulk re-queue from DLQ
- Prevention: monitor DLQ depth; alert on threshold
