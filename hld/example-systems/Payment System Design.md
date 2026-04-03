# Payment System Design

## System Overview
A payment processing platform (think Stripe / Razorpay / PayPal) that processes card payments, bank transfers, and digital wallet transactions — handling authorization, capture, settlement, refunds, and fraud detection with strong consistency and PCI-DSS compliance.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Accept payments via card (credit/debit), bank transfer, UPI, wallet
- Payment authorization and capture (two-step for cards)
- Refunds (full and partial)
- Recurring payments / subscriptions
- Payout to merchants
- Payment status tracking
- Fraud detection and prevention
- Multi-currency support

### Non-Functional Requirements
- Availability: 99.999% — payment downtime = revenue loss
- Latency: <500ms for payment authorization
- Consistency: Strong — no double charge, no missed payment
- Durability: Every payment event permanently recorded
- Security: PCI-DSS Level 1 compliance, tokenization, encryption
- Idempotency: Retried payments must not result in double charge

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 100M merchants, 1B end users
- 10M transactions/day average; 100M during peak (Black Friday)
- Average transaction: $50
- Card split: 60% card, 25% UPI/bank, 15% wallet
- Refund rate: 2%

### Traffic
```
Transactions/sec (avg)  = 10M / 86400 ≈ 116/sec
Transactions/sec (peak) = 100M / 86400 ≈ 1157/sec

Authorization calls     = 1157/sec to card networks (Visa/Mastercard)
```

### Storage
```
Transactions/day        = 10M × 1KB = 10GB/day → ~3.6TB/year
Ledger entries          = 10M × 3 entries × 300B = 9GB/day
Audit log               = 10M × 500B = 5GB/day
```

## 3. Core Components

**API Gateway** — Auth, rate limiting, routing; TLS termination; PCI-DSS boundary

**Payment API Service** — Entry point for payment requests; validates; creates payment record; publishes to Kafka; returns immediately (async for some flows)

**Authorization Service** — Calls card network (Visa/Mastercard) or bank for authorization; handles 3DS (3D Secure) for card payments

**Capture Service** — Captures authorized amount (for two-step card payments); called after merchant confirms order

**Refund Service** — Processes full/partial refunds; calls card network or bank for reversal

**Fraud Detection Service** — Real-time scoring per transaction; blocks high-risk payments; ML-based + rule-based

**Tokenization Service** — Replaces sensitive card data (PAN) with tokens; PCI-DSS scope reduction; stores card data in secure vault

**Settlement Service** — Batches transactions for settlement with card networks; T+1 or T+2 settlement

**Payout Service** — Disburses funds to merchants; see Payout System Design

**Ledger Service** — Double-entry bookkeeping; immutable record of all fund movements

**Webhook Service** — Delivers payment status updates to merchants via webhooks

**Payment DB (PostgreSQL)** — Payment records, status, metadata

**Ledger DB (PostgreSQL)** — Immutable double-entry ledger

**Card Vault (HSM-backed)** — Encrypted card data storage; Hardware Security Module for key management

**Audit Log (Cassandra)** — Immutable audit trail; every action logged

**Redis** — Idempotency keys, payment state cache, fraud signals, rate limiting

**Kafka** — Payment events, settlement batches, webhook delivery, fraud events

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL (Payment DB) | ACID for payment state transitions; relational queries; audit |
| PostgreSQL (Ledger DB) | Immutable double-entry; financial compliance |
| Cassandra (Audit Log) | Append-only, high write throughput, immutable |
| Redis | Idempotency keys (TTL), payment state cache, fraud velocity checks |
| HSM-backed Vault | PCI-DSS requirement for card data; hardware encryption |

### PostgreSQL — payments

| Field | Type |
|---|---|
| payment_id | UUID (PK) |
| merchant_id | UUID |
| customer_id | UUID, nullable |
| amount | DECIMAL(18,2) |
| currency | VARCHAR(3) |
| status | ENUM (created / authorized / captured / failed / refunded / cancelled) |
| payment_method | ENUM (card / upi / bank / wallet) |
| card_token | VARCHAR, nullable (token, not actual card) |
| authorization_code | VARCHAR, nullable |
| network_txn_id | VARCHAR, nullable |
| idempotency_key | VARCHAR, unique |
| failure_reason | TEXT, nullable |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

### PostgreSQL — refunds

| Field | Type |
|---|---|
| refund_id | UUID (PK) |
| payment_id | UUID (FK → payments) |
| amount | DECIMAL(18,2) |
| status | ENUM (pending / success / failed) |
| reason | TEXT |
| network_refund_id | VARCHAR, nullable |
| created_at | TIMESTAMP |

### PostgreSQL — ledger (immutable)

| Field | Type |
|---|---|
| entry_id | UUID (PK) |
| payment_id | UUID |
| account_type | ENUM (customer / merchant / platform / card_network) |
| entry_type | ENUM (debit / credit) |
| amount | DECIMAL(18,2) |
| currency | VARCHAR |
| event_type | VARCHAR (authorization / capture / refund / fee) |
| created_at | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `idempotency:{key}` | String | payment_id | 86400s |
| `payment:status:{paymentId}` | String | status JSON | 300s |
| `fraud:velocity:{customerId}` | Counter | txn count in window | 3600s |
| `fraud:velocity:{ip}` | Counter | txn count in window | 3600s |

## 5. Key Flows

### 5.1 Card Payment — Authorization + Capture (Two-Step)

```
Merchant → Payment API Service
                ↓
    Idempotency check (Redis)
                ↓
    Tokenize card (if raw card data) → Card Vault
                ↓
    Fraud Detection Service (sync, <100ms)
                ↓
    Authorization Service → Card Network (Visa/MC)
                ↓
    On auth success: payment.status = authorized
                ↓
    Merchant confirms order → Capture Service → Card Network
                ↓
    On capture: payment.status = captured
    Ledger entries written
    Webhook sent to merchant
```

**Authorization:**
1. Merchant sends `POST /payments` with `{amount, currency, cardToken, idempotencyKey}`
2. Idempotency check: `SET idempotency:{key} 1 NX EX 86400` — if exists, return cached result
3. Tokenization: if raw card number provided, tokenize → store in vault, use token
4. Fraud Detection: score transaction (<100ms); block if high risk
5. Authorization Service calls Visa/Mastercard API: "Can this card pay $X?"
6. Card network checks: card valid, sufficient funds, not blocked
7. On approval: `authorization_code` returned; `payment.status = authorized`
8. Funds held on customer's card (not yet transferred)

**Capture:**
9. Merchant confirms order shipped → `POST /payments/{id}/capture`
10. Capture Service calls card network: "Collect the authorized $X"
11. On success: `payment.status = captured`; ledger entries written
12. Funds transferred from customer to merchant (via settlement)

**Why two-step?**
Authorization holds funds without transferring. Capture transfers. Allows merchants to authorize at order time and capture at shipment — customer isn't charged until goods are sent.

### 5.2 UPI / Bank Transfer

1. Customer initiates UPI payment → Payment API Service
2. Fraud check
3. UPI Service calls NPCI (National Payments Corporation of India) UPI API
4. Customer approves on their UPI app (push notification)
5. NPCI confirms → `payment.status = captured` (UPI is single-step, no separate capture)
6. Ledger entries written; webhook to merchant

### 5.3 Refund Flow

1. Merchant initiates refund: `POST /payments/{id}/refund` with `{amount}`
2. Refund Service validates: payment is captured, refund amount ≤ original amount
3. Calls card network / bank for reversal
4. On success: write refund record; update payment status; write ledger credit entry
5. Funds returned to customer (T+3–5 days for cards)
6. Webhook to merchant

### 5.4 Fraud Detection

Real-time checks on every transaction:
- **Velocity checks:** `INCR fraud:velocity:{customerId}` — if > 10 txns/hr → flag
- **IP velocity:** `INCR fraud:velocity:{ip}` — if > 50 txns/hr from same IP → block
- **Amount anomaly:** transaction amount >> customer's historical average → flag
- **Card BIN check:** card issuer country vs customer location mismatch → flag
- **ML model:** trained on historical fraud patterns; scores 0–100; >80 = block, 50–80 = 3DS challenge

### 5.5 3D Secure (3DS)

For medium-risk transactions, redirect customer to card issuer for additional authentication (OTP, biometric):
1. Authorization Service detects 3DS required
2. Return 3DS redirect URL to merchant
3. Customer completes 3DS on issuer's page
4. Issuer returns authentication result
5. Authorization Service proceeds with auth using 3DS result
6. Liability shifts to issuer if fraud occurs (merchant protected)

### 5.6 Settlement

Daily batch process:
1. Settlement Service queries all `captured` payments from previous day
2. Groups by card network (Visa, Mastercard, etc.)
3. Submits settlement file to each network
4. Networks transfer funds to platform's bank account (T+1)
5. Platform disburses to merchants (T+2) via Payout Service

## 6. Key Interview Concepts

### Idempotency is Critical
Network timeout → merchant retries → risk of double charge. Solution:
- Merchant generates `idempotency_key` before first attempt
- Payment API stores key in Redis (24hr TTL) and DB (unique constraint)
- On retry: same key → return original result, no new charge
- Card networks also support idempotency keys — pass through

### Tokenization & PCI-DSS
Storing raw card numbers (PAN) requires PCI-DSS Level 1 compliance — extremely expensive. Solution: tokenize immediately on receipt. Replace PAN with a random token. Store PAN in HSM-backed vault (separate, highly secured system). All other systems use tokens — they're outside PCI scope.

### Authorization vs Capture
Authorization: "Can this card pay $X?" — funds held, not transferred. Capture: "Transfer the $X." — funds moved. Two-step allows:
- Pre-authorization for hotels/car rentals (hold funds, capture actual amount later)
- Capture only on shipment (customer not charged for cancelled orders)
- Partial capture (order partially fulfilled)

### Double-Entry Ledger
Every payment creates multiple ledger entries:
```
Card payment $100:
  DEBIT  customer_account    $100
  CREDIT platform_holding    $100  (during settlement period)
  
On settlement:
  DEBIT  platform_holding    $97   (after 3% fee)
  CREDIT merchant_account    $97
  DEBIT  platform_holding    $3
  CREDIT platform_revenue    $3    (fee)
```

### Saga Pattern for Payment + Order
Payment and order creation are separate services. If payment succeeds but order creation fails:
- Compensating transaction: refund the payment
- Kafka events: `PAYMENT_CAPTURED` → Order Service creates order; if fails → `ORDER_FAILED` → Payment Service refunds

### Webhook Reliability
Merchants need payment status updates. Webhooks must be delivered reliably:
- Kafka consumer delivers webhooks
- Retry with exponential backoff (1min, 5min, 30min, 2hr, 24hr)
- Dead letter queue after max retries
- Merchant can also poll `GET /payments/{id}` for status

## 7. Failure Scenarios

### Card Network Timeout
- Detection: no response within 3s
- Recovery: retry with same idempotency key; card network returns same result if already processed; after 3 retries, return `payment_failed`
- Prevention: idempotency key prevents double charge on retry

### Fraud Service Failure
- Recovery: fail open (allow payment with enhanced logging) or fail closed (block) based on risk tolerance; circuit breaker; Fraud Service is stateless, restarts quickly
- Prevention: multiple Fraud Service instances; cached fraud signals in Redis survive brief outage

### PostgreSQL Failure
- Impact: payment writes fail; Kafka events not acknowledged
- Recovery: promote replica (<30s); retry from Kafka; idempotency prevents duplicates
- Prevention: synchronous replication; automated failover

### Settlement Failure
- Impact: merchants not paid on time
- Recovery: retry settlement file submission; manual intervention if persistent
- Prevention: settlement is idempotent (same file = same result); monitor settlement status

### Tokenization Vault Failure
- Impact: new card payments fail (can't tokenize); existing token payments work
- Recovery: vault is highly available (HSM-backed, multi-AZ); brief unavailability → queue new card payments
- Prevention: vault is the most hardened component; multiple HSM instances; geographic redundancy
