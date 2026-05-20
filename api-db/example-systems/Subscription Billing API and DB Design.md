# Subscription Billing API and DB Design

## Problem Statement

Design the API and database schema for a subscription billing system (similar to Stripe Billing basics) where users subscribe to plans, are billed on recurring cycles, and can upgrade/downgrade or cancel.

**Core requirements:**
- Create and manage subscription plans
- Start, pause, resume, cancel subscriptions
- Generate recurring invoices and track payment status
- Handle plan changes with proration

**Scale:** 5M active subscriptions, monthly billing spikes, strict correctness for money flows

---

## API Design

### Core Endpoints
```http
POST /api/v1/plans
POST /api/v1/subscriptions
GET /api/v1/subscriptions/{subscriptionId}
PATCH /api/v1/subscriptions/{subscriptionId}
POST /api/v1/subscriptions/{subscriptionId}/cancel
POST /api/v1/invoices/{invoiceId}/pay
GET /api/v1/customers/{customerId}/invoices?cursor=...&limit=20
```

### Example: Create Subscription
```json
POST /api/v1/subscriptions
{
  "customerId": "cus_123",
  "planId": "plan_pro_monthly",
  "paymentMethodId": "pm_789",
  "idempotencyKey": "9f8f2c3e-..."
}
```

```json
201 Created
{
  "subscriptionId": "sub_456",
  "status": "ACTIVE",
  "currentPeriodStart": "2026-05-01T00:00:00Z",
  "currentPeriodEnd": "2026-06-01T00:00:00Z",
  "nextInvoiceId": "inv_111"
}
```

### Design Decisions
- Use idempotency keys for all payment-affecting POST APIs
- Webhook endpoint for async payment gateway updates
- Cursor pagination for invoice history

---

## Database Schema

```sql
CREATE TABLE plans (
    id              VARCHAR(64) PRIMARY KEY,
    name            VARCHAR(120) NOT NULL,
    amount_cents    BIGINT NOT NULL,
    currency        VARCHAR(8) NOT NULL,
    interval_unit   VARCHAR(16) NOT NULL, -- month/year
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE subscriptions (
    id                      VARCHAR(64) PRIMARY KEY,
    customer_id             VARCHAR(64) NOT NULL,
    plan_id                 VARCHAR(64) NOT NULL REFERENCES plans(id),
    status                  VARCHAR(24) NOT NULL, -- ACTIVE, PAUSED, CANCELED, PAST_DUE
    current_period_start    TIMESTAMPTZ NOT NULL,
    current_period_end      TIMESTAMPTZ NOT NULL,
    cancel_at_period_end    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_subscriptions_customer_status ON subscriptions (customer_id, status);
CREATE INDEX idx_subscriptions_period_end ON subscriptions (current_period_end);

CREATE TABLE invoices (
    id                  VARCHAR(64) PRIMARY KEY,
    subscription_id     VARCHAR(64) NOT NULL REFERENCES subscriptions(id),
    amount_due_cents    BIGINT NOT NULL,
    amount_paid_cents   BIGINT NOT NULL DEFAULT 0,
    currency            VARCHAR(8) NOT NULL,
    status              VARCHAR(24) NOT NULL, -- DRAFT, OPEN, PAID, VOID
    due_at              TIMESTAMPTZ NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_invoices_subscription_created ON invoices (subscription_id, created_at DESC);
```

---

## Trade-offs and Considerations

- Proration can be computed at change time (simpler) or via ledger events (more auditable).
- Payment retries should use exponential backoff with terminal failure states.
- Use immutable invoice line items for auditability.
