# Escrow System Design

## System Overview
An escrow service that holds funds from a buyer until predefined conditions are met, then releases to the seller — used in freelance platforms (Upwork), real estate, marketplace transactions, and M&A deals. Ensures neither party can be defrauded.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Buyer deposits funds into escrow
- Define release conditions (milestone completion, delivery confirmation, date)
- Seller fulfills conditions; buyer confirms
- Release funds to seller on confirmation
- Dispute resolution: hold funds pending arbitration
- Refund to buyer if conditions not met or dispute resolved in buyer's favor
- Full audit trail of all escrow actions

### Non-Functional Requirements
- Consistency: Strong — funds must never be double-released or lost
- Durability: Every state transition must be permanently recorded
- Availability: 99.99%
- Security: Funds held in segregated accounts; PCI-DSS compliance
- Auditability: Complete immutable log for regulatory compliance

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 1M active escrow contracts
- 100K new contracts/day
- Average escrow amount: $500
- Average duration: 14 days
- 10K disputes/day (1% of contracts)

### Traffic
```
New contracts/sec   = 100K / 86400 ≈ 1.2/sec (low write)
State transitions   = 100K × 5 avg transitions = 500K/day ≈ 6/sec
Dispute events      = 10K/day ≈ 0.1/sec
```

### Storage
```
Contracts           = 1M × 2KB = 2GB active
Contract history    = 100K/day × 365 × 2KB = 73GB/year
Ledger entries      = 500K/day × 300B = 150MB/day
Audit log           = 500K/day × 500B = 250MB/day
```

## 3. Core Components

**API Gateway** — Auth, rate limiting, routing

**Escrow Service** — Core orchestrator; manages escrow contract lifecycle; enforces state machine transitions

**Funding Service** — Handles buyer deposit into escrow; integrates with payment gateway; credits escrow holding account

**Release Service** — Releases funds to seller on condition fulfillment; integrates with payout system

**Refund Service** — Returns funds to buyer on cancellation or dispute resolution in buyer's favor

**Dispute Service** — Manages dispute lifecycle; holds funds during arbitration; implements arbitrator decision

**Condition Service** — Evaluates release conditions (milestone completion, delivery confirmation, timer expiry)

**Ledger Service** — Double-entry bookkeeping; immutable record of all fund movements

**Notification Service** — Kafka consumer; notifies buyer/seller of state changes, dispute updates

**Escrow DB (PostgreSQL)** — Contract records, state, conditions, parties

**Ledger DB (PostgreSQL)** — Immutable double-entry ledger

**Audit Log (Cassandra)** — Immutable audit trail of every action and state transition

**Redis** — Contract state cache, idempotency keys, session store

**Kafka** — State transition events, notification fan-out

## 4. Database Design

### PostgreSQL — escrow_contracts

| Field | Type |
|---|---|
| contract_id | UUID (PK) |
| buyer_id | UUID |
| seller_id | UUID |
| amount | DECIMAL(18,2) |
| currency | VARCHAR |
| state | ENUM (created / funded / in_progress / completed / disputed / refunded / cancelled) |
| release_conditions | JSONB |
| funded_at | TIMESTAMP, nullable |
| released_at | TIMESTAMP, nullable |
| expires_at | TIMESTAMP, nullable |
| idempotency_key | VARCHAR, unique |
| created_at | TIMESTAMP |

### PostgreSQL — milestones

| Field | Type |
|---|---|
| milestone_id | UUID (PK) |
| contract_id | UUID (FK → escrow_contracts) |
| description | TEXT |
| amount | DECIMAL(18,2) |
| status | ENUM (pending / submitted / approved / released) |
| submitted_at | TIMESTAMP, nullable |
| approved_at | TIMESTAMP, nullable |

### PostgreSQL — disputes

| Field | Type |
|---|---|
| dispute_id | UUID (PK) |
| contract_id | UUID (FK → escrow_contracts) |
| raised_by | UUID |
| reason | TEXT |
| status | ENUM (open / under_review / resolved_buyer / resolved_seller / escalated) |
| arbitrator_id | UUID, nullable |
| resolution_notes | TEXT, nullable |
| created_at | TIMESTAMP |
| resolved_at | TIMESTAMP, nullable |

### PostgreSQL — ledger (immutable)

| Field | Type |
|---|---|
| entry_id | UUID (PK) |
| contract_id | UUID |
| account_type | ENUM (buyer / escrow_holding / seller / platform) |
| entry_type | ENUM (debit / credit) |
| amount | DECIMAL(18,2) |
| event_type | VARCHAR (funded / released / refunded / fee) |
| created_at | TIMESTAMP |

### Cassandra — audit_log

| Field | Type |
|---|---|
| contract_id | UUID (partition key) |
| event_time | TIMESTAMP (clustering) |
| event_type | TEXT |
| actor_id | UUID |
| from_state | TEXT |
| to_state | TEXT |
| details | TEXT (JSON) |

## 5. Key Flows

### 5.1 Escrow Contract Lifecycle (State Machine)

```
CREATED → FUNDED → IN_PROGRESS → COMPLETED
                              ↘ DISPUTED → resolved → COMPLETED or REFUNDED
                              ↘ CANCELLED → REFUNDED
                              ↘ EXPIRED → REFUNDED
```

Every state transition is:
1. Validated (is transition allowed from current state?)
2. Written to PostgreSQL atomically with ledger entry
3. Logged to Cassandra audit log
4. Published to Kafka for notifications

### 5.2 Contract Creation & Funding

1. Buyer creates contract: `POST /escrow` with `{sellerId, amount, conditions, expiresAt}`
2. Escrow Service validates parties, creates contract (`state = created`)
3. Buyer initiates deposit → Funding Service → payment gateway
4. On payment success:
   - PostgreSQL transaction:
     - `contract.state = funded`
     - Ledger: DEBIT buyer_account, CREDIT escrow_holding_account
   - Publish `CONTRACT_FUNDED` to Kafka
5. Seller notified: "Funds secured in escrow, begin work"

### 5.3 Milestone Completion & Release

1. Seller completes work, submits milestone: `milestone.status = submitted`
2. Buyer reviews and approves: `milestone.status = approved`
3. Release Service triggered:
   - PostgreSQL transaction:
     - `milestone.status = released`
     - `contract.state = completed` (if all milestones done)
     - Ledger: DEBIT escrow_holding, CREDIT seller_account
     - Ledger: DEBIT escrow_holding, CREDIT platform_account (fee)
   - Payout Service initiates bank transfer to seller
4. Both parties notified

### 5.4 Dispute Flow

1. Either party raises dispute: `POST /escrow/{id}/dispute`
2. Dispute Service:
   - `contract.state = disputed` (funds frozen in escrow_holding)
   - Creates dispute record
3. Arbitrator assigned (human or automated)
4. Both parties submit evidence
5. Arbitrator resolves:
   - In buyer's favor: Refund Service → DEBIT escrow_holding, CREDIT buyer_account
   - In seller's favor: Release Service → DEBIT escrow_holding, CREDIT seller_account
6. Contract state updated to `completed` or `refunded`

### 5.5 Expiry & Auto-Refund

1. Scheduled job checks contracts where `expires_at < now AND state = funded/in_progress`
2. Refund Service initiates refund to buyer
3. Contract state → `refunded`
4. Both parties notified

## 6. Key Interview Concepts

### Escrow as a State Machine
Escrow contract is a strict state machine. Invalid transitions (e.g., `refunded → completed`) are rejected. This prevents race conditions where buyer and seller simultaneously trigger conflicting actions. State transitions are atomic PostgreSQL transactions.

### Segregated Holding Accounts
Escrow funds are held in a separate holding account (not mixed with platform operating funds). This is a regulatory requirement in many jurisdictions. Ledger tracks exact balance per contract in the holding account.

### Preventing Double Release
Two concurrent release requests for the same contract:
- PostgreSQL `UPDATE contracts SET state='completed' WHERE state='in_progress'` — only one succeeds
- Idempotency key on release request
- Ledger entry creation is atomic with state transition

### Dispute Arbitration
Automated arbitration: rule-based (e.g., if seller submitted proof of delivery, release to seller). Human arbitration: for complex cases, assign to human arbitrator with SLA. Escalation: if arbitrator doesn't resolve within SLA, escalate to senior arbitrator.

### Regulatory Compliance
Escrow services are regulated in many countries. Requirements:
- Segregated client funds (not mixed with company funds)
- Full audit trail (Cassandra immutable log)
- KYC/AML checks on parties
- Reporting to financial regulators

## 7. Failure Scenarios

### Payment Gateway Failure During Funding
- Recovery: idempotency key prevents double charge on retry; contract stays in `created` state until funding confirmed
- Prevention: webhook callback from gateway as confirmation fallback

### Release Service Crash Mid-Release
- Recovery: PostgreSQL transaction rolled back; contract stays in `in_progress`; retry release; idempotency key prevents double release
- Prevention: atomic DB transaction for state change + ledger entry

### Dispute Arbitrator Unavailable
- Recovery: SLA-based escalation; if no resolution in 7 days, auto-escalate; funds remain frozen (safe)
- Prevention: multiple arbitrators; automated rule-based resolution for clear-cut cases

### PostgreSQL Failure
- Recovery: promote replica; all in-flight operations retry; idempotency prevents duplicates
- Prevention: synchronous replication; automated failover; Cassandra audit log survives independently
