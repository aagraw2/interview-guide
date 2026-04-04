# Payment System Design

## System Overview
A payment gateway platform (think Stripe / Razorpay) that processes card payments through a PaymentIntent → Checkout Session → PCI-scoped tokenization → Processor Connector pipeline — handling authorization, capture, settlement, refunds, and fraud detection with strong consistency and PCI-DSS compliance.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Merchant onboarding and API key management
- Create payment intents and hosted checkout sessions
- Accept card payments with PCI-compliant tokenization
- Route payments to multiple processor connectors (Razorpay, PayU, Fiserv)
- Payment authorization and capture (two-step for cards)
- Async callback handling from processors
- Refunds (full and partial)
- Recurring payments / subscriptions
- Payout to merchants
- Payment status tracking and webhooks to merchants
- Reconciliation with processors (T+1)

### Non-Functional Requirements
- Availability: 99.999% — payment downtime = revenue loss
- Latency: <500ms for payment authorization
- Consistency: Strong — no double charge, no missed payment
- Durability: Every payment event permanently recorded
- Security: PCI-DSS Level 1 compliance, HSM-backed tokenization, TLS everywhere
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

Processor API calls     = 1157/sec to Razorpay/PayU/Fiserv
```

### Storage
```
Transactions/day        = 10M × 1KB = 10GB/day → ~3.6TB/year
Ledger entries          = 10M × 3 entries × 300B = 9GB/day
Audit log               = 10M × 500B = 5GB/day
```

## 3. Core Components

**API Gateway & Load Balancer** — Auth (API key / JWT), rate limiting, routing, TLS termination; PCI-DSS boundary for all inbound traffic

**PaymentIntent Service** — First step in the payment flow; merchant creates a payment intent with amount, currency, order details; returns `payment_intent_id`; writes to PaymentIntent DB

**Checkout Session Service** — Creates a checkout session tied to a payment intent; stores session state in Redis with TTL; returns a hosted checkout URL (`https://pay.gateway.com/checkout/sess_{id}`); validates: session exists, not expired, intent valid, merchant matched, session ACTIVE

**Checkout Frontend Service** — Serves the hosted checkout HTML/JS page at the session URL; client-side card form; never touches raw card data on merchant's servers (PCI scope reduction)

**Checkout Backend Service** — Handles checkout form submission; validates session state before processing; calls Tokenization Service in PCI Zone; calls Orchestrator Service

**PCI Zone — Tokenization Service** — Isolated zone with strict network controls; four steps:
1. Validate request
2. Generate card fingerprint (hash of PAN for dedup)
3. Vault card in HSM (Hardware Security Module)
4. Encrypt card using HSM → return `encrypted_card_token`

**Orchestrator Service** — Routes payment to the correct processor connector based on merchant config, currency, payment method; reads session data from Redis; calls Processor Gateway

**Processor Gateway** — Abstraction layer over multiple payment processors; routes to connector services

**Connector Services** — Adapters to specific processors:
- `RazorPayConnectorSvc` → Razorpay API
- `PayUConnectorSvc` → PayU API
- `FiservConnectorSvc` → Fiserv API

**Collector Callback Service** — Receives async callbacks from payment processors (success/failure); publishes to Kafka `payment.processor.callback.status` topic

**Fraud Detection Service** — Real-time scoring per transaction; blocks high-risk payments; ML-based + rule-based velocity checks

**Refund Service** — Processes full/partial refunds via processor connectors

**Settlement Service** — T+1 batch settlement with processors; reconciles via Reconciliation Service

**Reconciliation Service** — Compares Payment Processor DB records with processor reports; flags discrepancies

**Ledger Service** — Double-entry bookkeeping; immutable record of all fund movements

**Webhook Service** — Delivers payment status updates to merchants; Kafka-backed with retry

**PaymentIntent DB (PostgreSQL)** — `payment_intent_id, status (SENT/DONE), amount, currency, merchant_id, payment_method, order_id, customer, card_token, metadata`

**Payment Processor DB (PostgreSQL)** — Processor-level records: `txn_id, intent_id, status (SENT/DONE), amount, currency, merchant_id, payment_method, card_token, metadata`

**Merchant DB (PostgreSQL)** — Merchant profiles, API keys, processor routing config, fee structure

**Card DB (HSM-backed Vault)** — Encrypted card data; Hardware Security Module for key management; isolated in PCI Zone

**Ledger DB (PostgreSQL)** — Immutable double-entry ledger

**Redis** — Checkout session state (TTL), idempotency keys, fraud velocity signals, payment status cache

**Kafka** — `payment.processor.callback.status` (async processor callbacks), `payment.processor.status` (T+1 settlement status)

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL (PaymentIntent DB) | ACID for intent state transitions; relational |
| PostgreSQL (Payment Processor DB) | Processor-level records; reconciliation queries |
| PostgreSQL (Merchant DB) | Merchant config, routing rules, ACID |
| PostgreSQL (Ledger DB) | Immutable double-entry; financial compliance |
| HSM-backed Card Vault | PCI-DSS requirement; hardware encryption; isolated network zone |
| Redis | Session state (TTL), idempotency keys, fraud velocity checks |
| Kafka | Async processor callbacks; T+1 settlement events |

### PostgreSQL — payment_intents

| Field | Type |
|---|---|
| payment_intent_id | UUID (PK) |
| status | ENUM (SENT / DONE / FAILED / CANCELLED) |
| amount | DECIMAL(18,2) |
| currency | VARCHAR(3) |
| merchant_id | UUID |
| payment_method | ENUM (CARD / UPI / BANK / WALLET) |
| order_id | UUID |
| customer | JSONB |
| card_token | VARCHAR, nullable |
| metadata | JSONB |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

### PostgreSQL — payment_processor_records

| Field | Type |
|---|---|
| txn_id | UUID (PK) |
| intent_id | UUID (FK → payment_intents) |
| status | ENUM (SENT / DONE / FAILED) |
| amount | DECIMAL(18,2) |
| currency | VARCHAR(3) |
| merchant_id | UUID |
| payment_method | VARCHAR |
| card_token | VARCHAR |
| processor | VARCHAR (razorpay / payu / fiserv) |
| processor_txn_id | VARCHAR |
| metadata | JSONB |
| created_at | TIMESTAMP |

### PostgreSQL — ledger (immutable)

| Field | Type |
|---|---|
| entry_id | UUID (PK) |
| txn_id | UUID |
| account_type | ENUM (customer / merchant / platform / processor) |
| entry_type | ENUM (debit / credit) |
| amount | DECIMAL(18,2) |
| currency | VARCHAR |
| event_type | VARCHAR (authorization / capture / refund / fee) |
| created_at | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `checkout:session:{sessionId}` | String | session state JSON | 1800s (30 min) |
| `idempotency:{key}` | String | txn_id | 86400s |
| `fraud:velocity:{customerId}` | Counter | txn count in window | 3600s |
| `fraud:velocity:{ip}` | Counter | txn count in window | 3600s |
| `payment:status:{intentId}` | String | status JSON | 300s |

## 5. Key Flows

### 5.1 Full Payment Flow (Card)

```
Merchant → PaymentIntent Service → PaymentIntent DB
                ↓ (returns payment_intent_id)
         Checkout Session Service → Redis (session state)
                ↓ (returns hosted checkout URL)
         Customer opens URL → Checkout Frontend Service (HTML/JS)
                ↓ (customer enters card details)
         Checkout Backend Service
           - validate session (Redis)
           - call Tokenization Service (PCI Zone)
                ↓ (returns encrypted_card_token)
         Orchestrator Service → Processor Gateway
           → RazorPayConnectorSvc / PayUConnectorSvc / FiservConnectorSvc
                ↓ (async callback)
         Collector Callback Service → Kafka (payment.processor.callback.status)
                ↓
         Payment Processor DB updated
         Ledger entries written
         Webhook to merchant
```

**Step by step:**

1. Merchant calls `POST /payment-intents` with `{amount, currency, orderId, customer}`
2. PaymentIntent Service creates intent (`status = SENT`), returns `payment_intent_id`
3. Merchant calls `POST /checkout-sessions` with `payment_intent_id`
4. Checkout Session Service creates session in Redis (TTL 30 min), returns hosted URL
5. Customer opens URL → Checkout Frontend serves card form (merchant's domain never sees card data)
6. Customer submits card → Checkout Backend validates session state:
   - Session exists in Redis
   - Not expired
   - Intent valid and matches
   - Merchant matched
   - Session state is ACTIVE
7. Checkout Backend calls Tokenization Service (PCI Zone):
   - Validate request
   - Generate card fingerprint
   - Vault card in HSM
   - Return `encrypted_card_token`
8. Checkout Backend calls Orchestrator with `{intentId, cardToken, amount}`
9. Orchestrator reads merchant routing config from Merchant DB
10. Routes to appropriate Connector (e.g., RazorPayConnectorSvc)
11. Connector calls Razorpay API synchronously or async
12. Processor responds → Collector Callback Service receives callback
13. Publishes to Kafka `payment.processor.callback.status`
14. Consumer updates Payment Processor DB, PaymentIntent DB (`status = DONE`)
15. Ledger entries written; webhook sent to merchant

### 5.2 Session Validation (Checkout Backend)

Before processing any payment, Checkout Backend validates:
- Session exists in Redis (not expired)
- `intent_id` in session matches request
- `merchant_id` in session matches API key
- Session `state = ACTIVE` (not already used or cancelled)

This prevents replay attacks, session hijacking, and double-processing.

### 5.3 PCI Zone — Tokenization

The PCI Zone is an isolated network segment with strict ingress/egress rules. Only Checkout Backend can call into it. Steps:
1. Validate request (required fields, format)
2. Generate card fingerprint: `SHA256(PAN + expiry)` — used to detect duplicate cards without storing PAN
3. Vault: store encrypted PAN in HSM-backed Card DB
4. Encrypt card token using HSM key → return `encrypted_card_token`

The token is what flows through all other systems. Raw PAN never leaves the PCI Zone.

### 5.4 Processor Connector Pattern

Orchestrator selects connector based on:
- Merchant's preferred processor
- Payment method (some processors don't support all methods)
- Currency (some processors are region-specific)
- Failover: if primary processor is down, route to secondary

Each connector is an independent service — adding a new processor (e.g., Stripe) means adding a new connector without changing Orchestrator logic.

### 5.5 Async Callback Handling

Processors often respond asynchronously (especially for bank transfers, UPI):
1. Connector sends payment request to processor
2. Processor returns `202 Accepted` with `processor_txn_id`
3. Processor later sends callback to Collector Callback Service
4. Collector validates callback signature (HMAC)
5. Publishes to Kafka `payment.processor.callback.status`
6. Consumer updates Payment Processor DB and PaymentIntent DB
7. Webhook sent to merchant

### 5.6 Refund Flow

1. Merchant calls `POST /payments/{intentId}/refund`
2. Refund Service validates: intent is DONE, refund amount ≤ original
3. Calls appropriate Connector to initiate refund at processor
4. Processor confirms → update records, write ledger credit entry
5. Webhook to merchant

### 5.7 T+1 Reconciliation

1. Settlement Service runs daily batch
2. Downloads settlement report from each processor
3. Reconciliation Service compares Payment Processor DB records with report
4. Flags discrepancies (missing transactions, amount mismatches)
5. Publishes `payment.processor.status` events to Kafka
6. Finance team reviews flagged items

## 6. Key Interview Concepts

### PaymentIntent Pattern (Stripe-style)
Creating a PaymentIntent before checkout separates "intent to pay" from "actual payment". Benefits:
- Merchant can create intent server-side without exposing API keys to frontend
- Intent can be updated (amount change, currency change) before checkout
- Idempotent: same intent can be retried without creating duplicate charges
- Status tracking: `SENT → DONE / FAILED` gives clear lifecycle

### Hosted Checkout (PCI Scope Reduction)
If merchant's frontend handles card data, they're in PCI scope (expensive audit). Hosted checkout: card form is served from the payment gateway's domain. Merchant's servers never see raw card data. Merchant is out of PCI scope. This is how Stripe Checkout and Razorpay's hosted page work.

### PCI Zone Isolation
The Tokenization Service and Card DB live in a strictly isolated network zone:
- Only Checkout Backend can call in (no direct internet access)
- HSM (Hardware Security Module) handles all encryption/decryption
- Card fingerprint enables dedup without storing PAN elsewhere
- Audit logs of every vault access

### Connector Pattern for Multi-Processor
Different processors have different APIs, auth mechanisms, and response formats. Connector Services abstract this:
- Orchestrator speaks one internal interface
- Each connector translates to processor-specific protocol
- Adding a new processor = new connector, no Orchestrator changes
- Failover: Orchestrator can retry on a different connector if one fails

### Idempotency
Network timeout → merchant retries → risk of double charge. Solution:
- Merchant generates `idempotency_key` before first attempt
- Stored in Redis (24hr TTL) and DB (unique constraint)
- On retry: same key → return original result, no new charge
- Connectors pass idempotency key to processors (processors also support it)

### Double-Entry Ledger
Every payment creates multiple ledger entries:
```
Card payment $100:
  DEBIT  customer_account    $100
  CREDIT platform_holding    $100

On settlement:
  DEBIT  platform_holding    $97   (after 3% fee)
  CREDIT merchant_account    $97
  DEBIT  platform_holding    $3
  CREDIT platform_revenue    $3
```

### Tokenization & PCI-DSS
Storing raw card numbers (PAN) requires PCI-DSS Level 1 compliance. Solution: tokenize immediately. Replace PAN with a random token. Store PAN only in HSM-backed vault. All other systems use tokens — they're outside PCI scope.

### Authorization vs Capture
Authorization: "Can this card pay $X?" — funds held, not transferred. Capture: "Transfer the $X." Two-step allows:
- Pre-authorization for hotels/car rentals
- Capture only on shipment
- Partial capture

## 7. Failure Scenarios

### Processor Timeout (Async Callback Not Received)
- Detection: no callback within SLA (e.g., 30s for card, 5min for UPI)
- Recovery: poll processor status API; if confirmed, update DB; if not, mark as failed and retry
- Prevention: idempotency key prevents double charge on retry; Kafka retains callback events

### Checkout Session Expired
- Scenario: customer takes >30 min to complete checkout
- Recovery: return session expired error; merchant creates new session; customer re-enters card
- Prevention: 30-min TTL is generous; warn customer at 25 min

### Tokenization Service (PCI Zone) Failure
- Impact: new card payments fail; existing token payments work
- Recovery: HSM-backed vault is highly available (multi-AZ); brief unavailability → queue checkout requests
- Prevention: multiple HSM instances; geographic redundancy; PCI Zone has its own HA setup

### Orchestrator Routes to Wrong Processor
- Scenario: processor routing config stale in cache
- Recovery: Orchestrator reads fresh config from Merchant DB on cache miss; connector returns error → retry on alternate processor
- Prevention: short cache TTL for routing config; circuit breaker per connector

### Reconciliation Discrepancy
- Scenario: payment shows DONE in our DB but missing in processor's settlement report
- Recovery: Reconciliation Service flags it; finance team investigates; may be timing difference (T+1 vs T+2)
- Prevention: retain raw processor callbacks; compare against settlement report; alert on discrepancy rate > 0.1%

### PostgreSQL Failure
- Impact: payment writes fail; Kafka callbacks not acknowledged
- Recovery: promote replica (<30s); retry from Kafka; idempotency prevents duplicates
- Prevention: synchronous replication; automated failover
