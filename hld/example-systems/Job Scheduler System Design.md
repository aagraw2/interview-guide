# Job Scheduler System Design

## System Overview
A distributed job scheduling system that allows clients to submit one-time or recurring jobs (cron-style), executes them reliably at the scheduled time, handles retries on failure, and provides job status visibility — similar to AWS EventBridge Scheduler or Quartz Scheduler at scale.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Submit, edit, and cancel jobs
- Schedule jobs for a specific time (one-time) or on a recurring schedule (cron expression)
- Execute jobs reliably at or near the scheduled time
- Retry failed jobs with configurable retry count
- Track job status (queued / running / success / failed / dead)
- Search and filter jobs by status, type, schedule
- Cancel a running job

### Non-Functional Requirements
- Availability: 99.99% — jobs must never be silently dropped
- Latency: Jobs should execute within a few seconds of their scheduled time
- Scalability: Millions of jobs scheduled, thousands executing concurrently
- Durability: Job definitions and state must survive any single component failure
- Idempotency: A job must not execute more than once per scheduled trigger
- At-least-once delivery: A job must execute at least once even if executor crashes

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 10M active scheduled jobs
- 100K job executions/day on average
- Peak: 10K jobs due at the same second (e.g., all cron jobs set to run at midnight)
- Average job payload: 1KB
- Job history retained for 30 days
- Retry attempts: up to 3 per job

### Traffic
```
Job submissions/sec     = 100K / 86400 ≈ 1.2/sec (low write)
Job executions/sec      = 100K / 86400 ≈ 1.2/sec avg
Peak executions/sec     = 10K (midnight spike)

Watcher poll interval   = every 20s
Jobs fetched per poll   = all jobs where schedule_time <= now + 20s
                        = up to 10K jobs per poll cycle
```

### Storage
```
Job definitions     = 10M × 1KB = 10GB
Job run history     = 100K/day × 30 days × 2KB = 6GB rolling
Total               = ~16GB (very manageable, fits in a single DB)
```

### Memory (Redis)
```
last_polled_time    = single key (negligible)
Cancel requests     = active jobs × 100B ≈ small
Executor heartbeats = N executors × 100B ≈ small
```

## 3. Core Components

**LB + API Gateway** — Authentication, authorization, rate limiting, routing

**Job Service** — Submit, edit, cancel job definitions; writes to Job DB (PostgreSQL)

**Job Search Service** — Search and filter jobs by status, type, schedule, name; queries Job DB directly (or Elasticsearch for large scale)

**Watcher** — The scheduler brain; polls Job DB every 20s for jobs due to run; publishes them to Kafka; tracks `last_polled_time` in Redis; also detects stuck running jobs

**Kafka** — Three topics: `run` (new jobs to execute), `retry` (failed jobs to retry), `dead` (jobs that exhausted retries)

**Job Consumer Service** — Kafka consumer; receives jobs from `run` and `retry` topics; distributes to Executor instances (batch of 1–100); updates Job DB with retry count via Consumer Service

**Executor Service** — Pool of worker instances (101 shown as 1–100 range); pulls job from Job Consumer, executes the actual job logic; updates job status in Redis every 10s (heartbeat); writes final status to Job DB; handles cancel signal from Redis

**Job DB (PostgreSQL)** — Source of truth for all job definitions and run history

**Redis** — `last_polled_time` for Watcher coordination; cancel request signals (`job: {id}, request: cancel, TTL`); executor heartbeats; status update buffer (every 10s)

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL (Job DB) | ACID for job state transitions, row-level locking to prevent duplicate execution, complex queries for Watcher polling, reliable source of truth |
| Redis | Watcher coordination (last_polled_time), cancel signals with TTL, executor heartbeats — all ephemeral, fast, TTL-based |
| Kafka | Durable job dispatch queue; decouples Watcher from Executors; topic-per-state (run/retry/dead) enables independent processing |

### PostgreSQL — jobs (job definitions)

| Field | Type |
|---|---|
| job_id | UUID (PK) |
| job_name | VARCHAR |
| job_type | VARCHAR (one-time / recurring) |
| schedule_time | TIMESTAMP (next execution time) |
| cron_expression | VARCHAR, nullable (for recurring) |
| payload | JSONB |
| status | ENUM (queued / running / success / failed / dead) |
| retry_count | INT (current attempt number) |
| max_retries | INT |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

### PostgreSQL — job_runs (execution history)

| Field | Type |
|---|---|
| run_id | UUID (PK) |
| job_id | UUID (FK → jobs) |
| status | ENUM (queued / running / success / failed) |
| start_time | TIMESTAMP |
| end_time | TIMESTAMP, nullable |
| executor_id | VARCHAR |
| attempt_number | INT |
| error_msg | TEXT, nullable |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `watcher:last_polled` | String | TIMESTAMP | — |
| `job:cancel:{jobId}` | String | `"cancel"` | 300s (TTL — executor checks this) |
| `executor:heartbeat:{executorId}:{jobId}` | String | timestamp | 30s (refreshed every 10s) |

## 5. Key Flows

### 5.1 Auth

1. Client registers → API Gateway → User Service → JWT issued
2. JWT validated on every request at API Gateway
3. Job submission and cancellation require auth
4. Job search can be scoped to authenticated user's jobs

### 5.2 Job Submission

1. Client submits job: `POST /jobs` with `{name, type, scheduleTime, cronExpression, payload, maxRetries}`
2. API Gateway → Job Service
3. Validate: schedule time in future, valid cron expression if recurring
4. Write to Job DB with `status = queued`, `schedule_time = next execution time`
5. Return `jobId` to client

**For recurring jobs:**
- `cron_expression` stored in DB
- After each execution, Watcher or Job Service computes next `schedule_time` from cron expression and resets `status = queued`

### 5.3 Watcher — The Scheduler Brain

The Watcher is the most critical component. It runs continuously, polling for jobs due to execute.

```
Every 20 seconds:
  1. Read last_polled_time from Redis
  2. Query Job DB:
     SELECT * FROM jobs
     WHERE status = 'queued'
       AND schedule_time <= now + 20s
       AND schedule_time > last_polled_time
     FOR UPDATE SKIP LOCKED   ← prevents duplicate pickup if multiple Watchers
  3. Mark fetched jobs as status = 'running' in DB (atomic with the SELECT)
  4. Publish each job to Kafka topic: 'run'
  5. Update last_polled_time in Redis to now
```

**Also detects stuck jobs:**
- Query jobs where `status = 'running'` AND `updated_at < now - 30s` (no heartbeat)
- These are jobs whose Executor crashed mid-execution
- Re-queue them: `status = queued` (or push to `retry` topic if retry count > 0)

**Why poll every 20s:**
- Balances DB load vs scheduling precision
- Jobs execute within 20s of their scheduled time (acceptable for most use cases)
- For sub-second precision: use Redis Sorted Set (see alternatives section)

**Multiple Watcher instances:**
- `SELECT FOR UPDATE SKIP LOCKED` ensures two Watcher instances never pick up the same job
- Only one Watcher processes each job even if multiple are running for HA

### 5.4 Job Execution Flow

```
Kafka (run topic)
      ↓
Job Consumer Service
      ↓ (batch 1–100)
Executor Service (one of N instances)
      ↓
Execute job logic
      ↓
Update Redis heartbeat every 10s
      ↓
On complete: write final status to Job DB
             publish result to Kafka (success → done, failure → retry/dead)
```

1. Job Consumer Service consumes from Kafka `run` topic
2. Distributes jobs to available Executor instances (batch size 1–100)
3. Executor picks up job, updates `job_runs` with `status = running`, `start_time = now`
4. Executor runs job logic (HTTP call, script, function invocation, etc.)
5. Every 10s: Executor writes heartbeat to Redis: `SET executor:heartbeat:{executorId}:{jobId} {timestamp} EX 30`
6. On success:
   - Update `jobs.status = success` in Job DB
   - Update `job_runs.status = success`, `end_time = now`
   - If recurring: compute next `schedule_time`, reset `status = queued`
7. On failure:
   - If `retry_count < max_retries`: increment `retry_count`, publish to Kafka `retry` topic
   - If `retry_count >= max_retries`: update `status = dead`, publish to Kafka `dead` topic

### 5.5 Job Cancellation

1. Client sends `DELETE /jobs/{jobId}` → Job Service
2. Job Service writes cancel signal to Redis: `SET job:cancel:{jobId} "cancel" EX 300`
3. Updates `jobs.status = cancelled` in Job DB

**If job is queued (not yet executing):**
- Watcher skips it on next poll (status no longer `queued`)
- Done

**If job is running (Executor has it):**
- Executor checks Redis for `job:cancel:{jobId}` every few seconds during execution
- On detecting cancel signal: stops execution, updates `job_runs.status = cancelled`
- TTL on Redis key ensures cleanup even if Executor crashes

### 5.6 Retry Flow

1. Executor publishes failed job to Kafka `retry` topic with incremented `retry_count`
2. Job Consumer Service consumes from `retry` topic
3. Applies backoff delay (e.g., exponential: 1min, 5min, 30min) before re-dispatching
4. Dispatches to Executor
5. If all retries exhausted: publish to `dead` topic
6. Dead letter jobs: alert ops team, available for manual inspection and re-trigger

### 5.7 Job Search

1. Client queries `GET /jobs?status=failed&type=recurring` → Job Search Service
2. Queries Job DB with filters on `status`, `job_type`, `schedule_time`, `job_name`
3. Returns paginated list of jobs with run history
4. For large scale: index Job DB in Elasticsearch for full-text search on `job_name`, `payload`

## 6. Key Interview Concepts

### Preventing Duplicate Execution — The Core Problem
The hardest problem: Watcher polls and publishes job to Kafka. What if Watcher crashes after publishing but before marking job as `running`? On restart, it polls again and publishes the same job twice → two Executors run the same job.

**Solution: atomic status transition**
```sql
UPDATE jobs SET status = 'running', updated_at = now()
WHERE job_id = ? AND status = 'queued'
RETURNING *
```
Only one update succeeds (PostgreSQL row-level lock). If Watcher publishes to Kafka and then crashes, the job is already `running` in DB — on restart, Watcher won't pick it up again.

**`SELECT FOR UPDATE SKIP LOCKED`** for multiple Watcher instances: each Watcher locks the rows it's processing; other Watchers skip locked rows and move on. No coordination needed between Watchers.

### At-Least-Once vs Exactly-Once
- At-least-once: job may execute more than once (on crash + retry). Acceptable if jobs are idempotent.
- Exactly-once: much harder — requires distributed transactions between Kafka and DB.

This design provides at-least-once. Job owners should design their jobs to be idempotent (safe to run twice). For critical non-idempotent jobs: use a deduplication key in the job payload and check before executing.

### Watcher Polling vs Push Alternatives

**Current approach: DB polling every 20s**
- Simple, reliable, works with any DB
- Scheduling precision: ±20s
- DB load: one query every 20s (manageable)

**Alternative 1: Redis Sorted Set**
- Store jobs as `ZADD scheduled_jobs {timestamp} {jobId}`
- Watcher does `ZRANGEBYSCORE scheduled_jobs 0 {now}` to get due jobs
- Sub-second precision, very fast
- Trade-off: Redis is not the source of truth — need to sync with DB; data loss on Redis failure

**Alternative 2: Amazon SQS with delay queues**
- SQS supports message delay up to 15 minutes
- For longer delays: store in DB, push to SQS when within 15min window
- Managed service — no Watcher needed
- Trade-off: vendor lock-in, 15min delay limit, less control

### Stuck Job Detection
Executor crashes mid-job → job stays `running` forever. Watcher detects this by checking jobs where `status = running` AND `updated_at < now - 30s` (heartbeat expired). These are re-queued. Heartbeat interval (10s) and stuck threshold (30s) are tunable — balance between false positives and detection speed.

### Kafka Topic Design
Three topics map to three job states:
- `run`: new jobs ready to execute
- `retry`: failed jobs to retry (with backoff)
- `dead`: jobs that exhausted all retries

Separate topics allow independent consumer scaling and monitoring. Dead letter topic enables ops team to inspect and manually re-trigger failed jobs.

### Executor Scaling
Job Consumer Service distributes jobs in batches of 1–100 to Executor instances. Executors are stateless workers — scale horizontally. Each Executor handles one job at a time (or a small pool). Total throughput = N executors × jobs/sec per executor. Auto-scale based on Kafka consumer lag.

### Cron Expression Parsing
After a recurring job executes, compute next `schedule_time`:
```
cron: "0 9 * * 1-5"  (9am every weekday)
last run: Monday 9:00am
next run: Tuesday 9:00am  → update schedule_time in DB, reset status = queued
```
Standard cron libraries (e.g., `cron-parser`) handle this. Next schedule_time computed by Job Service on submission and by Executor after each run.

### Job Priority
Not in base design but common follow-up: add a `priority` field to jobs. Kafka doesn't natively support priority queues. Solutions:
- Multiple Kafka topics by priority (high/medium/low), consumers poll high-priority first
- Priority queue in Job Consumer Service before dispatching to Executors

### CAP Trade-off
Job Scheduler favors Consistency over Availability for job dispatch — it's worse to run a job twice than to delay it by a few seconds. PostgreSQL with row-level locking ensures exactly one Watcher claims each job. Redis is AP — used only for ephemeral coordination (heartbeats, cancel signals), not for job state.

## 7. Failure Scenarios

### Watcher Crash Mid-Poll
- Scenario: Watcher fetches jobs, marks them `running` in DB, publishes to Kafka, then crashes before updating `last_polled_time` in Redis
- Recovery: on restart, Watcher reads stale `last_polled_time` → re-queries the same window → jobs already `running` in DB are skipped (status filter); no duplicate execution
- Prevention: update `last_polled_time` only after successful Kafka publish; idempotent DB status transition

### Executor Crash Mid-Execution
- Detection: heartbeat in Redis expires (no update for 30s); Watcher detects `status = running` with stale `updated_at`
- Recovery: Watcher re-queues job (`status = queued`); Job Consumer re-dispatches to another Executor; retry count incremented
- Prevention: heartbeat TTL (30s) > heartbeat interval (10s) gives 3 missed heartbeats before detection

### Kafka Unavailable
- Impact: Watcher can't publish jobs; executions delayed
- Recovery: Watcher retries publish with backoff; jobs remain `queued` in DB (not lost); when Kafka recovers, Watcher resumes normal polling
- Prevention: Kafka cluster with replication; jobs in DB are the durable source — Kafka is a dispatch mechanism, not storage

### PostgreSQL Primary Failure
- Impact: Watcher can't poll; Job Service can't accept submissions; Executors can't update status
- Recovery: promote replica (<30s); Watcher and Executors retry DB operations; in-flight Executor jobs continue running (they don't need DB mid-execution, only at completion)
- Prevention: synchronous replication; automated failover; Executors buffer final status update and retry

### Duplicate Execution (Kafka Redelivery)
- Scenario: Executor processes job, writes success to DB, but Kafka offset not committed → Kafka redelivers job → second Executor picks it up
- Recovery: second Executor attempts `UPDATE jobs SET status='running' WHERE status='queued'` → fails (already `success`) → Executor detects and skips
- Prevention: Executor checks current job status in DB before executing; idempotent status transition

### Job Never Executes (Missed Schedule)
- Scenario: Watcher was down during a job's scheduled window
- Recovery: on restart, Watcher queries jobs where `status = queued AND schedule_time < now` — catches up on all missed jobs and dispatches them
- Prevention: multiple Watcher instances with `SKIP LOCKED`; monitor Watcher health; alert if polling gap > 1 min

### Executor Pool Exhausted (All Busy)
- Scenario: 10K jobs due at midnight, only 100 Executors available
- Recovery: jobs queue in Kafka `run` topic; Executors process as fast as they can; jobs execute slightly late but not lost
- Prevention: auto-scale Executors based on Kafka consumer lag; pre-scale before known peak times (e.g., midnight cron spike)

### Cancel Signal Missed
- Scenario: Redis TTL on cancel key expires before Executor checks it
- Recovery: Job DB `status = cancelled` is the authoritative state; Executor checks DB status at job completion — if `cancelled`, discards result
- Prevention: Executor checks both Redis (fast) and DB (authoritative) for cancel signal; TTL (300s) is generous for any reasonable job duration
