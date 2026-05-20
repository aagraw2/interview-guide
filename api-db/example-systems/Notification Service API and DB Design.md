# Notification Service API and DB Design

## Problem Statement

Design the API and database schema for a notification service that sends messages across channels (email, SMS, push), supports templates, retries, and user preferences.

**Core requirements:**
- Send single and bulk notifications
- Support multiple channels and fallback order
- Track delivery status and retry failures
- Respect user opt-in/opt-out preferences

**Scale:** 50M notifications/day with burst traffic

---

## API Design

### Core Endpoints
```http
POST /api/v1/notifications
POST /api/v1/notifications/bulk
GET /api/v1/notifications/{notificationId}
POST /api/v1/templates
PATCH /api/v1/preferences/{userId}
```

### Example: Send Notification
```json
POST /api/v1/notifications
{
  "userId": "usr_123",
  "templateId": "tpl_order_shipped",
  "channels": ["push", "email"],
  "variables": { "orderId": "ORD-001", "eta": "2 days" },
  "idempotencyKey": "f4ea4bd9-..."
}
```

```json
202 Accepted
{
  "notificationId": "ntf_456",
  "status": "QUEUED"
}
```

### Design Decisions
- 202 Accepted because send is async via queue
- Channel adapters isolate provider-specific integrations
- Idempotency key prevents duplicate notifications

---

## Database Schema

```sql
CREATE TABLE notification_templates (
    id                  VARCHAR(64) PRIMARY KEY,
    name                VARCHAR(120) NOT NULL,
    channel             VARCHAR(16) NOT NULL, -- EMAIL, SMS, PUSH
    subject_template    TEXT,
    body_template       TEXT NOT NULL,
    version             INTEGER NOT NULL DEFAULT 1,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE notification_requests (
    id                  VARCHAR(64) PRIMARY KEY,
    user_id             VARCHAR(64) NOT NULL,
    template_id         VARCHAR(64) REFERENCES notification_templates(id),
    payload_json        JSONB NOT NULL,
    status              VARCHAR(24) NOT NULL, -- QUEUED, SENT, FAILED
    idempotency_key     VARCHAR(128) UNIQUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notification_requests_user_time ON notification_requests (user_id, created_at DESC);
CREATE INDEX idx_notification_requests_status_time ON notification_requests (status, created_at);

CREATE TABLE delivery_attempts (
    id                  BIGSERIAL PRIMARY KEY,
    notification_id     VARCHAR(64) NOT NULL REFERENCES notification_requests(id),
    channel             VARCHAR(16) NOT NULL,
    provider            VARCHAR(32) NOT NULL,
    attempt_no          INTEGER NOT NULL,
    status              VARCHAR(24) NOT NULL,
    provider_message_id VARCHAR(128),
    error_code          VARCHAR(64),
    attempted_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE notification_preferences (
    user_id             VARCHAR(64) PRIMARY KEY,
    email_enabled       BOOLEAN NOT NULL DEFAULT TRUE,
    sms_enabled         BOOLEAN NOT NULL DEFAULT TRUE,
    push_enabled        BOOLEAN NOT NULL DEFAULT TRUE,
    quiet_hours_json    JSONB
);
```

---

## Trade-offs and Considerations

- Queue-based async delivery improves throughput and resilience.
- Separate delivery attempts table enables audit and retry analytics.
- Provider outages require fallback channels and dead-letter handling.
