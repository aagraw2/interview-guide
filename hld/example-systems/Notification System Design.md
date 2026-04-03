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

**API Gateway** — Auth, rate limiting, routing for notification submission API

**Notification API Service** — Receives notification requests from internal services; validates; writes to Notification DB; publishes to Kafka

**Preference Service** — Manages user notification preferences per channel per type; cached in Redis

**Template Service** — Manages notification templates; renders content with user-specific variables

**Router Service** — Kafka consumer; reads notification from queue; checks user preferences; routes to appropriate channel workers

**Push Worker** — Sends push notifications via FCM (Android) and APNs (iOS); handles device token management

**Email Worker** — Sends emails via email provider (SendGrid / SES); handles bounces and unsubscribes

**SMS Worker** — Sends SMS via provider (Twilio / SNS); handles delivery receipts

**Retry Service** — Monitors failed notifications; re-queues with exponential backoff

**Delivery Tracker** — Tracks delivery status per notification; updates Notification DB; handles webhooks from providers

**Campaign Service** — Manages bulk marketing campaigns; segments users; schedules sends; rate-limits to avoid overwhelming channel workers

**Notification DB (Cassandra)** — Notification records, delivery status; high write throughput

**Preference DB (PostgreSQL)** — User preferences, opt-outs, device tokens

**Template DB (PostgreSQL)** — Notification templates

**Redis** — User preference cache, deduplication keys, rate limiting, campaign state

**Kafka** — Notification queue (separate topics per channel), retry queue, dead letter queue

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| Cassandra (Notification DB) | High write throughput for notification records and delivery logs; time-series |
| PostgreSQL (Preference DB) | Structured preferences, device tokens, opt-outs — ACID, relational |
| Redis | Preference cache (hot path), dedup keys (TTL), rate limiting |
| Kafka | Durable notification queue; separate topics per channel for independent scaling |

### Cassandra — notifications

Partition key: `user_id`, Clustering: `notification_id DESC`

| Field | Type |
|---|---|
| user_id | UUID (partition key) |
| notification_id | TIMEUUID (clustering) |
| type | VARCHAR (order_update / marketing / alert / chat) |
| channel | ENUM (push / email / sms) |
| title | VARCHAR |
| body | TEXT |
| data | JSONB (deep link, action) |
| status | ENUM (pending / sent / delivered / failed / opened) |
| idempotency_key | VARCHAR |
| created_at | TIMESTAMP |
| sent_at | TIMESTAMP, nullable |

### PostgreSQL — user_preferences

| Field | Type |
|---|---|
| user_id | UUID |
| notification_type | VARCHAR |
| channel | ENUM (push / email / sms) |
| enabled | BOOLEAN |
| updated_at | TIMESTAMP |

### PostgreSQL — device_tokens

| Field | Type |
|---|---|
| token_id | UUID (PK) |
| user_id | UUID |
| platform | ENUM (ios / android / web) |
| token | VARCHAR, unique |
| is_active | BOOLEAN |
| last_used | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `pref:{userId}:{type}:{channel}` | String | enabled (0/1) | 300s |
| `dedup:{idempotencyKey}` | String | notificationId | 86400s |
| `rate:{userId}` | Counter | notifications sent in window | 3600s |
| `campaign:progress:{campaignId}` | String | sent count | while active |

## 5. Key Flows

### 5.1 Transactional Notification (e.g., Order Confirmed)

```
Order Service → Notification API Service
                        ↓
    Validate + dedup check (Redis idempotency key)
                        ↓
    Write to Cassandra (status=pending)
                        ↓
    Publish to Kafka: notifications topic
                        ↓
    Router Service consumes:
      Check user preferences (Redis cache)
      If push enabled: publish to push-notifications topic
      If email enabled: publish to email-notifications topic
                        ↓
    Push Worker → FCM/APNs → delivery receipt
    Email Worker → SendGrid → delivery webhook
                        ↓
    Delivery Tracker updates status in Cassandra
```

1. Order Service calls `POST /notify` with `{userId, type: order_update, title, body, idempotencyKey}`
2. Notification API validates and checks Redis dedup: `SET dedup:{key} 1 NX EX 86400`
3. If duplicate: return existing notificationId (no re-send)
4. Write to Cassandra with `status = pending`
5. Publish to Kafka `notifications` topic
6. Router Service consumes:
   - Fetch user preferences from Redis (TTL 5min) → fallback to PostgreSQL
   - If user opted out of this type/channel: skip
   - Route to appropriate channel topic
7. Channel worker sends via provider
8. Delivery Tracker receives webhook/callback, updates status

### 5.2 Push Notification (FCM/APNs)

1. Push Worker consumes from `push-notifications` Kafka topic
2. Fetch user's device tokens from PostgreSQL (or Redis cache)
3. For each active device token:
   - Call FCM API (Android) or APNs API (iOS)
   - FCM/APNs returns delivery status
4. On success: update `status = sent`
5. On failure (invalid token): mark token as inactive in PostgreSQL
6. On failure (rate limit / server error): publish to retry queue

**Device token management:**
- Tokens expire or become invalid when user uninstalls app
- FCM/APNs return `InvalidRegistration` error for stale tokens
- Push Worker marks stale tokens as inactive immediately

### 5.3 Email Notification

1. Email Worker consumes from `email-notifications` topic
2. Fetch user's email from User DB
3. Render template with user-specific variables (name, order details)
4. Call SendGrid/SES API
5. SendGrid sends webhook on delivery/bounce/open
6. Delivery Tracker updates status

**Bounce handling:**
- Hard bounce (invalid email): mark email as invalid, disable email channel for user
- Soft bounce (mailbox full): retry after 1hr, max 3 retries

### 5.4 Bulk Campaign

1. Marketing team creates campaign: `{segment: all_users, type: promotion, template, schedule}`
2. Campaign Service segments users (query User DB with filters)
3. At scheduled time: Campaign Service publishes user batches to Kafka
4. Router Service processes each user: check preferences, route to channel workers
5. Rate limiting: Campaign Service throttles to max N notifications/sec to avoid overwhelming providers
6. Progress tracked in Redis: `INCR campaign:progress:{campaignId}`

**Why rate limiting campaigns:**
- FCM/APNs have rate limits per app
- Email providers have sending limits
- Sending 100M emails in 1 minute would get the domain blacklisted
- Spread over 1hr = 27.8K/sec — manageable

### 5.5 Retry Logic

1. Channel worker publishes failed notification to `retry` Kafka topic with `retry_count`
2. Retry Service applies exponential backoff: 1min, 5min, 30min, 2hr
3. After max retries (3): publish to `dead_letter` topic, update status = `failed`
4. Dead letter notifications: alert ops, available for manual re-trigger

### 5.6 User Preferences

1. User updates preferences in app → Preference Service → write to PostgreSQL
2. Invalidate Redis cache: `DEL pref:{userId}:*`
3. Router Service reads from Redis on every notification (TTL 5min)
4. Opt-out is respected immediately (cache invalidated on change)

## 6. Key Interview Concepts

### Idempotency
Same notification must not be sent twice. Solution: `idempotency_key` (e.g., `hash(orderId + userId + type)`) checked in Redis before processing. If key exists: return existing result. TTL = 24hr (covers retry window).

### Channel Abstraction
Notification system is channel-agnostic. Callers specify `{userId, type, content}` — not the channel. Router Service decides which channel(s) to use based on user preferences. This decouples callers from channel implementation details.

### Kafka Topic Per Channel
Separate Kafka topics for push, email, SMS allow:
- Independent scaling (push needs more workers than SMS)
- Independent retry policies (email retries differently than push)
- Independent monitoring (email bounce rate vs push delivery rate)

### Rate Limiting Per User
Prevent notification spam: max 10 notifications/hr per user. Redis counter: `INCR rate:{userId}` with 1hr TTL. If count > 10: drop notification (or queue for next window). Transactional notifications bypass rate limit; marketing notifications are rate-limited.

### Delivery Tracking
Providers send webhooks on delivery/open/bounce. Delivery Tracker receives webhooks, updates Cassandra. Enables analytics: delivery rate, open rate, bounce rate per campaign. Also enables debugging: "why didn't user X receive notification Y?"

### Priority Queues
Transactional notifications (order confirmed, OTP) must be delivered immediately. Marketing notifications can wait. Solution: separate Kafka topics by priority — high-priority topic processed first; workers drain high-priority before consuming low-priority.

## 7. Failure Scenarios

### FCM/APNs Outage
- Impact: push notifications not delivered
- Recovery: retry queue with backoff; fall back to email/SMS if configured; alert ops
- Prevention: monitor FCM/APNs status; circuit breaker on push worker

### Email Provider Rate Limit
- Detection: SendGrid returns 429
- Recovery: exponential backoff retry; switch to backup provider (SES)
- Prevention: stay within provider limits; spread campaign sends over time

### Kafka Consumer Lag (Router Service)
- Impact: notification delivery delayed
- Recovery: scale up Router Service instances; Kafka retains messages
- Prevention: monitor consumer lag; auto-scale on lag threshold

### User Preference Cache Stale
- Scenario: user opts out but Redis cache still shows opted-in → notification sent
- Recovery: short TTL (5min) limits window; on opt-out, actively invalidate cache
- Prevention: cache invalidation on preference change; TTL as safety net

### Dead Letter Queue Buildup
- Scenario: provider outage causes many notifications to exhaust retries
- Recovery: ops team re-triggers after provider recovery; bulk re-queue from dead letter
- Prevention: monitor dead letter queue depth; alert on threshold
