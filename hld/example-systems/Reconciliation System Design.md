# Reconciliation System Design

## System Overview
A financial reconciliation system that compares transaction records across multiple systems (internal ledger, payment gateway, bank statements) to detect discrepancies, flag mismatches, and ensure financial data integrity — critical for fintech, payments, and banking platforms.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Ingest transaction data from multiple sources (internal DB, payment gateway reports, bank statements)
- Match transactions across sources by amount, reference ID, timestamp
- Detect and flag discrepancies (missing transactions, amount mismatches, duplicates)
- Generate reconciliation reports (daily, weekly)
- Alert finance team on unresolved discrepancies
- Manual resolution workflow for flagged items
- Audit trail of all reconciliation runs

### Non-Functional Requirements
- Accuracy: 100% — every discrepancy must be detected
- Latency: Reconciliation run completes within 1hr for daily batch
- Scalability: 10M+ transactions/day
- Durability: Reconciliation results and audit logs must never be lost
- Idempotency: Re-running reconciliation must not create duplicate records

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 10M transactions/day across all sources
- 3 data sources: internal ledger, payment gateway, bank
- Average transaction record: 500B
- Discrepancy rate: ~0.1% = 10K discrepancies/day

### Traffic
```
Transactions/day    = 10M × 3 sources = 30M records to process
Processing rate     = 30M / 3600s (1hr window) = 8.3K records/sec

Discrepancies/day   = 10K records to flag and store
```

### Storage
```
Transaction records = 30M × 500B = 15GB/day
Reconciliation runs = 365 runs/year × 15GB = 5.5TB/year
Discrepancy log     = 10K/day × 1KB = 10MB/day (small)
```

## 3. Core Components

**API Gateway** — Auth, routing for manual resolution UI and report APIs

**Data Ingestion Service** — Pulls/receives transaction data from each source:
- Internal ledger: direct DB query or CDC
- Payment gateway: SFTP file download or API pull
- Bank: SFTP file (MT940 / CSV format)
Normalizes all records to a common schema; writes to Staging DB

**Reconciliation Engine** — Core batch processing service; reads from Staging DB; matches records across sources; writes matches and discrepancies to Reconciliation DB

**Matching Service** — Implements matching logic:
- Exact match: same reference ID + amount + currency
- Fuzzy match: same amount + currency within ±1hr timestamp window
- Unmatched: present in one source but not others

**Discrepancy Service** — Manages flagged discrepancies; tracks resolution status; notifies finance team

**Report Service** — Generates daily/weekly reconciliation reports; exports to CSV/PDF

**Notification Service** — Alerts finance team on unresolved discrepancies via email/Slack

**Staging DB (PostgreSQL)** — Normalized transaction records from all sources, per reconciliation run

**Reconciliation DB (PostgreSQL)** — Match results, discrepancy records, resolution history

**Audit Log (Cassandra)** — Immutable log of every reconciliation run and action

**Redis** — Idempotency keys, run state, rate limiting

**S3** — Raw source files (bank statements, gateway reports), generated reports

**Kafka** — Triggers reconciliation runs, discrepancy alerts

## 4. Database Design

### PostgreSQL — staging_transactions

| Field | Type |
|---|---|
| staging_id | UUID (PK) |
| run_id | UUID |
| source | ENUM (internal / gateway / bank) |
| reference_id | VARCHAR |
| amount | DECIMAL(18,2) |
| currency | VARCHAR |
| transaction_time | TIMESTAMP |
| status | VARCHAR |
| raw_data | JSONB |
| normalized_at | TIMESTAMP |

### PostgreSQL — reconciliation_runs

| Field | Type |
|---|---|
| run_id | UUID (PK) |
| run_date | DATE |
| status | ENUM (running / completed / failed) |
| total_records | INT |
| matched_count | INT |
| discrepancy_count | INT |
| started_at | TIMESTAMP |
| completed_at | TIMESTAMP |

### PostgreSQL — matches

| Field | Type |
|---|---|
| match_id | UUID (PK) |
| run_id | UUID |
| internal_txn_id | VARCHAR |
| gateway_txn_id | VARCHAR |
| bank_txn_id | VARCHAR |
| amount | DECIMAL(18,2) |
| currency | VARCHAR |
| match_type | ENUM (exact / fuzzy) |
| matched_at | TIMESTAMP |

### PostgreSQL — discrepancies

| Field | Type |
|---|---|
| discrepancy_id | UUID (PK) |
| run_id | UUID |
| type | ENUM (missing_in_gateway / missing_in_bank / amount_mismatch / duplicate) |
| reference_id | VARCHAR |
| internal_amount | DECIMAL, nullable |
| gateway_amount | DECIMAL, nullable |
| bank_amount | DECIMAL, nullable |
| status | ENUM (open / investigating / resolved / waived) |
| resolved_by | UUID, nullable |
| resolved_at | TIMESTAMP, nullable |
| notes | TEXT |

### Cassandra — audit_log

| Field | Type |
|---|---|
| run_id | UUID (partition key) |
| event_time | TIMESTAMP (clustering) |
| event_type | TEXT |
| actor_id | UUID |
| details | TEXT (JSON) |

## 5. Key Flows

### 5.1 Data Ingestion

1. Scheduled job triggers at end of day (or T+1 morning)
2. Data Ingestion Service pulls from each source:
   - Internal ledger: query `transactions` table for date range
   - Payment gateway: download CSV/JSON report via API or SFTP
   - Bank: download MT940/CSV statement via SFTP
3. Normalize all records to common schema (reference_id, amount, currency, timestamp)
4. Write to `staging_transactions` with `run_id`
5. Publish `INGESTION_COMPLETE` to Kafka → triggers Reconciliation Engine

### 5.2 Reconciliation Engine

```
For each run:
  1. Load all staging records grouped by source
  2. Build lookup maps: reference_id → record for each source
  3. For each internal transaction:
     a. Exact match: find same reference_id in gateway + bank
     b. If found: write to matches table (exact)
     c. If not found: fuzzy match by amount + currency ± 1hr
     d. If fuzzy match: write to matches (fuzzy), flag for review
     e. If no match: write to discrepancies (missing_in_gateway or missing_in_bank)
  4. Find gateway/bank records with no internal match → discrepancies (missing_in_internal)
  5. Find duplicate reference_ids → discrepancies (duplicate)
  6. Update reconciliation_run with counts and status=completed
```

**Matching algorithm:**
```
Phase 1 - Exact match:
  For each internal txn:
    IF gateway[ref_id] exists AND bank[ref_id] exists:
      IF amounts match: EXACT_MATCH
      ELSE: AMOUNT_MISMATCH discrepancy

Phase 2 - Fuzzy match (unmatched records):
  For remaining unmatched:
    Find gateway records with same amount ± $0.01 within ±1hr
    If found: FUZZY_MATCH (flag for human review)

Phase 3 - Unmatched:
  Remaining records → MISSING discrepancies
```

### 5.3 Discrepancy Resolution

1. Finance team reviews open discrepancies in dashboard
2. For each discrepancy:
   - Investigate: check raw transaction data, contact gateway/bank
   - Resolve: mark as resolved with notes (e.g., "timing difference, matched next day")
   - Waive: mark as waived with justification (e.g., "bank fee, expected")
3. All actions logged to Cassandra audit log
4. Resolved discrepancies trigger refund/adjustment if needed

### 5.4 Report Generation

1. Report Service queries Reconciliation DB for run results
2. Generates summary: total transactions, match rate, discrepancy breakdown by type
3. Exports to CSV/PDF, stores to S3
4. Emails report to finance team

## 6. Key Interview Concepts

### Why Reconciliation is Hard
Data comes from multiple independent systems with different:
- Reference ID formats (internal: UUID, gateway: alphanumeric, bank: numeric)
- Timestamps (different timezones, settlement vs transaction time)
- Amount precision (gateway may round differently)
- Latency (bank statement may be T+1 or T+2)

Reconciliation must handle all these variations while maintaining 100% accuracy.

### Exact vs Fuzzy Matching
Exact match (same reference ID) is fast and reliable. Fuzzy match (same amount + time window) is needed when reference IDs don't align across systems. Fuzzy matches are flagged for human review — automated resolution is risky for financial data.

### Idempotency of Reconciliation Runs
Re-running reconciliation for the same date must not create duplicate records. Solution: `run_id` is deterministic (`hash(date + sources)`). Before processing, check if `run_id` already exists in `reconciliation_runs`. If exists and completed, skip. If exists and failed, re-run from last checkpoint.

### T+1 / T+2 Settlement
Bank statements often reflect settlement date, not transaction date. A transaction on Monday may appear in Tuesday's bank statement. Reconciliation must handle this by matching across a rolling window (e.g., ±2 days) for bank matching.

### Double-Entry Verification
For internal ledger: sum of all debits must equal sum of all credits for any time period. This is a quick sanity check before external reconciliation. If internal books don't balance, there's a bug in the payment system itself.

## 7. Failure Scenarios

### Data Source Unavailable (Gateway/Bank)
- Impact: reconciliation run incomplete for that source
- Recovery: retry ingestion with backoff; run partial reconciliation with available sources; flag missing source in report
- Prevention: multiple retry attempts; alert if source unavailable after 3 retries

### Reconciliation Engine Crash Mid-Run
- Recovery: idempotent run design — restart from beginning; staging data already loaded; re-processing produces same results
- Prevention: checkpoint progress to DB; resume from last checkpoint on restart

### Amount Mismatch Due to FX
- Scenario: transaction in USD, gateway reports in EUR at different exchange rate
- Recovery: normalize all amounts to base currency using exchange rate at transaction time; store both original and normalized amounts
- Prevention: always store original currency + amount; normalize separately

### Large Discrepancy Volume
- Scenario: system bug causes 100K discrepancies in one run
- Recovery: alert finance team immediately; pause auto-resolution; manual review required
- Prevention: anomaly detection — alert if discrepancy rate > 1% (normal is 0.1%)
