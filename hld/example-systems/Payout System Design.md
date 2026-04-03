# Payout System Design

## System Overview
A payout system that disburses money from a platform to multiple recipients (sellers, drivers, freelancers, affiliates) — handling bulk payouts, bank transfers, UPI/wallet credits, scheduling, failure retries, and reconciliation. Used by platforms like Amazon Seller Payouts, Uber Driver Earnings, or Upwork Freelancer Payments.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Schedule and initiate payouts to recipients (bank transfer, UPI, wallet)
- Bulk payout processing (thousands of recipients in one batch)
- Payout status tracking (pending / processing / success / failed)
- Retry failed payouts automatically
- Hold/release payouts (fraud hold, compliance review)
- Payout history and statements for recipients
- Reconciliation with bank/payment gateway

### Non-Functional Requirements
- Availability: 99.99% — payout processing must not stop
- Consistency: Strong — no double payout, no missed payout
- Durability: Every payout instruction must be persisted before processing
- Idempotency: Retried payouts must not result in double disbursement
- Auditability: Complete audit trail for every payout action

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 1M recipients on platform
- 5M payouts/day (daily/weekly settlement cycles)
- Average payout: $50
- Peak: end of week/month settlement — 10× normal
- Payout methods: 60% bank transfer, 30% UPI, 10% wallet

### Traffic
```
Payouts/sec (avg)   = 5M / 86400 ≈ 58/sec
Payouts/sec (peak)  = 580/sec (settlement day)

Bank API calls/sec  = 58 × 0.6 = 35/sec (rate-limited by bank)
```

### Storage
```
Payout records/day  = 5M × 1KB = 5GB/day → ~1.8TB/year
Ledger entries      = 5M × 2 entries × 300B = 3GB/day
Audit log           = 5M × 500B = 2.5GB/day
```

## 3. Core Components

**API Gateway** — Auth, rate limiting, routing

**Payout Service** — Core orchestrator; creates payout instructions; manages payout lifecycle; idempotency enforcement

**Payout Scheduler** — Triggers scheduled payout batches (daily/weekly); reads eligible recipients from Payout DB; publishes to Kafka

**Payout Processor** — Kafka consumer; processes individual payout instructions; calls appropriate payment rail (bank/UPI/wallet); updates status

**Bank Transfer Service** — Integrates with NEFT/RTGS/IMPS/ACH APIs; handles bank-specific formats and rate limits

**UPI Service** — Integrates with UPI payment rails; handles VPA (Virtual Payment Address) validation and transfer

**Wallet Credit Service** — Credits internal wallet balance; direct DB update (no external API)

**Retry Service** — Monitors failed payouts; applies retry logic with backoff; re-publishes to Kafka

**Hold Service** — Manages payout holds (fraud, compliance, dispute); blocks payout until released

**Ledger Service** — Double-entry bookkeeping for all payout movements; immutable

**Reconciliation Service** — Matches payout records with bank/gateway confirmations

**Notification Service** — Kafka consumer; notifies recipients of payout status

**Payout DB (PostgreSQL)** — Payout instructions, status, recipient details

**Ledger DB (PostgreSQL)** — Double-entry ledger, immutable

**Audit Log (Cassandra)** — Immutable audit trail of every action

**Redis** — Idempotency keys, payout state cache, rate limiting per bank API

**Kafka** — Payout job queue, retry queue, dead letter queue, notification events

## 4. Database Design

### PostgreSQL — payouts

| Field | Type |
|---|---|
| payout_id | UUID (PK) |
| recipient_id | UUID |
| amount | DECIMAL(18,2) |
| currency | VARCHAR |
| method | ENUM (bank / upi / wallet) |
| status | ENUM (pending / processing / success / failed / held / cancelled) |
| idempotency_key | VARCHAR, unique |
| bank_reference | VARCHAR, nullable |
| retry_count | INT |
| max_retries | INT |
| scheduled_at | TIMESTAMP |
| processed_at | TIMESTAMP, nullable |
| failure_reason | TEXT, nullable |
| batch_id | UUID, nullable |

### PostgreSQL — payout_batches

| Field | Type |
|---|---|
| batch_id | UUID (PK) |
| total_count | INT |
| total_amount | DECIMAL |
| success_count | INT |
| failed_count | INT |
| status | ENUM (pending / processing / completed) |
| created_at | TIMESTAMP |

### PostgreSQL — recipients

| Field | Type |
|---|---|
| recipient_id | UUID (PK) |
| name | VARCHAR |
| bank_account | VARCHAR (encrypted) |
| ifsc_code | VARCHAR |
| upi_vpa | VARCHAR, nullable |
| kyc_status | ENUM (pending / verified / rejected) |
| payout_hold | BOOLEAN |
| created_at | TIMESTAMP |

### PostgreSQL — ledger (double-entry, immutable)

| Field | Type |
|---|---|
| entry_id | UUID (PK) |
| payout_id | UUID |
| account_id | VARCHAR |
| entry_type | ENUM (debit / credit) |
| amount | DECIMAL(18,2) |
| balance_after | DECIMAL |
| created_at | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `idempotency:{key}` | String | payout_id | 86400s |
| `payout:status:{payoutId}` | String | status | 300s |
| `bank:rate:{bankCode}` | Counter | API calls in window | 60s |

## 5. Key Flows

### 5.1 Single Payout Initiation

```
Client → Payout Service
              ↓
    Check idempotency key (Redis)
              ↓
    Validate recipient (KYC verified, no hold)
              ↓
    Write payout record (status=pending) to PostgreSQL
    Write ledger debit entry
              ↓
    Publish to Kafka payout queue
              ↓
    Return payoutId (202 Accepted)
```

1. Platform initiates payout with `idempotency_key = hash(recipientId + amount + date)`
2. Payout Service checks Redis — if key exists, return existing `payoutId` (no duplicate)
3. Validate: recipient KYC verified, no hold, sufficient platform balance
4. Write payout record to PostgreSQL (`status = pending`)
5. Write ledger debit entry (platform account debited)
6. Publish to Kafka `payouts` topic
7. Return `payoutId` immediately

### 5.2 Payout Processing

```
Kafka → Payout Processor
              ↓
    Update status = processing
              ↓
    Route to payment rail:
      Bank → Bank Transfer Service → NEFT/IMPS API
      UPI  → UPI Service → UPI rail
      Wallet → Wallet Credit Service → DB update
              ↓
    On success:
      Update status = success
      Write ledger credit entry (recipient credited)
      Publish SUCCESS event → Notification Service
    On failure:
      Update status = failed, store failure_reason
      Publish to retry queue if retry_count < max_retries
      Else publish to dead letter queue
```

1. Payout Processor consumes from Kafka
2. Updates `status = processing` in DB
3. Routes to appropriate payment rail based on `method`
4. Bank Transfer: calls NEFT/IMPS API with account + IFSC + amount
5. On bank API success: update `status = success`, `bank_reference = {bank_txn_id}`
6. Write ledger credit entry (recipient account credited)
7. Publish `PAYOUT_SUCCESS` to Kafka → Notification Service

### 5.3 Bulk Payout (Batch)

1. Payout Scheduler triggers at scheduled time (e.g., every Friday 6pm)
2. Queries eligible recipients: `SELECT * FROM payouts WHERE status=pending AND scheduled_at <= now`
3. Creates `payout_batch` record
4. Publishes all payout jobs to Kafka in bulk
5. Payout Processors consume in parallel (N processors × M payouts/sec)
6. Batch status updated as payouts complete

### 5.4 Retry Flow

1. Failed payout published to Kafka `retry` topic
2. Retry Service applies exponential backoff: 5min, 30min, 2hr, 24hr
3. After max retries (e.g., 3): publish to `dead_letter` topic
4. Dead letter payouts: alert ops team; manual investigation required
5. Ops can manually re-trigger or cancel

### 5.5 Payout Hold

1. Fraud/compliance system flags recipient → Hold Service
2. `UPDATE payouts SET status=held WHERE recipient_id=?`
3. Payout Processor skips held payouts
4. On hold release: status reset to `pending`, re-published to Kafka

## 6. Key Interview Concepts

### Idempotency is Critical
Bank APIs can time out. Retrying without idempotency = double payout. Solution:
- Platform generates `idempotency_key` before first attempt
- Payout Service stores key in Redis (24hr TTL) and DB (unique constraint)
- Bank APIs also support idempotency keys — pass through to prevent double transfer
- On retry: same key → same result, no new transfer

### Double-Entry Ledger
Every payout creates two ledger entries:
```
Payout $100 to seller:
  DEBIT  platform_account  $100  (platform balance decreases)
  CREDIT seller_account    $100  (seller balance increases)
```
Sum of all entries = 0. Enables reconciliation and audit.

### Bank API Rate Limits
Banks rate-limit API calls (e.g., 100 NEFT requests/sec). Solution:
- Redis counter per bank: `INCR bank:rate:{bankCode}` with 60s window
- If rate exceeded: queue payout for next window
- Multiple bank integrations: distribute payouts across banks to maximize throughput

### Payout Timing Windows
NEFT: batch processing, 30-min settlement windows (8am–7pm). IMPS: 24/7, instant. RTGS: large amounts, business hours. Payout Processor selects appropriate rail based on amount, urgency, and time of day.

### KYC Validation
Payout to unverified recipient is a compliance risk. Always check `kyc_status = verified` before processing. KYC verification is async (document upload → manual/automated review → status update). Payouts to unverified recipients are held until KYC completes.

## 7. Failure Scenarios

### Bank API Timeout
- Recovery: retry with same idempotency key; bank returns same result if already processed
- If bank processed but response lost: bank's idempotency key prevents double transfer; status updated on next retry

### Payout Processor Crash Mid-Processing
- Detection: Kafka message not acknowledged
- Recovery: Kafka redelivers; Payout Processor checks current status in DB — if already `success`, skip; if `processing`, re-attempt
- Prevention: idempotency key prevents double processing

### PostgreSQL Failure
- Impact: payout writes fail; Kafka messages not acknowledged; payouts not lost (Kafka retains)
- Recovery: promote replica; Payout Processor retries after DB recovery
- Prevention: synchronous replication; automated failover

### Insufficient Platform Balance
- Detection: ledger balance check before payout initiation
- Recovery: hold all pending payouts; alert finance team to top up platform account
- Prevention: minimum balance alerts; auto top-up from reserve account

### Dead Letter Queue Buildup
- Scenario: bank API down for hours → many payouts fail all retries → dead letter queue grows
- Recovery: ops team investigates; manually re-triggers payouts after bank recovery
- Prevention: monitor dead letter queue depth; alert if > threshold
