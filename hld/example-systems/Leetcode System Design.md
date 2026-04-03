# Leetcode (Online Judge) System Design

## System Overview
An online coding platform where users solve programming problems, submit code that is compiled and executed in a sandboxed environment, and receive verdicts (Accepted, Wrong Answer, Time Limit Exceeded, etc.) — with a leaderboard, contests, and discussion forums.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- User registration and authentication
- Browse and search problems (by difficulty, tag, company)
- Write and submit code in multiple languages (Python, Java, C++, etc.)
- Execute code against test cases; return verdict and output
- Run custom test cases (without submitting)
- View submission history and accepted solutions
- Contests with real-time leaderboard
- Discussion forum per problem

### Non-Functional Requirements
- Availability: 99.9%
- Latency: Code execution result within 5s for most submissions
- Scalability: 5M+ users, 1M+ submissions/day
- Security: Code must run in isolated sandbox — no access to host system, network, or other users' data
- Fairness: Consistent execution environment for all users

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 5M users, 500K DAU
- 1M submissions/day, 100K during contest peaks
- Average execution time: 1–2s
- Average code size: 2KB
- 10 test cases per problem avg

### Traffic
```
Submissions/sec (avg)   = 1M / 86400 ≈ 12/sec
Submissions/sec (peak)  = 100K / 3600 ≈ 28/sec (contest hour)

Execution jobs/sec      = 28 × 10 test cases = 280 jobs/sec (peak)
```

### Storage
```
Problems            = 3000 × 10KB = 30MB (tiny)
Submissions/day     = 1M × 2KB code + 1KB metadata = 3GB/day → ~1TB/year
Test cases          = 3000 problems × 10 cases × 10KB = 300MB
```

## 3. Core Components

**API Gateway** — Auth, rate limiting (prevent submission spam), routing

**Problem Service** — Problem CRUD, test case management; reads/writes to Problem DB

**Submission Service** — Receives code submissions; validates; publishes to execution queue (Kafka); returns `submissionId` immediately (async)

**Code Execution Service (Judge)** — The core component; pulls jobs from Kafka; runs code in isolated sandbox; evaluates against test cases; writes verdict to Submission DB

**Sandbox** — Isolated execution environment per submission:
- Docker container with resource limits (CPU, memory, time)
- No network access
- Read-only filesystem except temp dir
- Killed after time limit

**Contest Service** — Manages contest lifecycle, scoring, real-time leaderboard (Redis ZSET)

**Discussion Service** — Forum threads per problem; comments, upvotes

**Search Service** — Problem search by title, tag, difficulty, company; Elasticsearch

**Notification Service** — Kafka consumer; sends submission result notification

**Problem DB (PostgreSQL)** — Problems, test cases, editorial

**Submission DB (PostgreSQL)** — Submission records, verdicts, code

**User DB (MySQL)** — Users, solved problems, stats

**Redis** — Contest leaderboard (ZSET), submission status polling, session store

**Kafka** — Submission queue, result events

**S3** — Large test case files, editorial assets

## 4. Database Design

### PostgreSQL — problems

| Field | Type |
|---|---|
| problem_id | UUID (PK) |
| title | VARCHAR |
| description | TEXT |
| difficulty | ENUM (easy / medium / hard) |
| tags | ARRAY\<VARCHAR\> |
| companies | ARRAY\<VARCHAR\> |
| acceptance_rate | DECIMAL |
| created_at | TIMESTAMP |

### PostgreSQL — test_cases

| Field | Type |
|---|---|
| test_id | UUID (PK) |
| problem_id | UUID (FK → problems) |
| input | TEXT |
| expected_output | TEXT |
| is_sample | BOOLEAN |
| time_limit_ms | INT |
| memory_limit_mb | INT |

### PostgreSQL — submissions

| Field | Type |
|---|---|
| submission_id | UUID (PK) |
| user_id | UUID |
| problem_id | UUID |
| language | VARCHAR |
| code | TEXT |
| status | ENUM (pending / running / accepted / wrong_answer / tle / mle / runtime_error / compile_error) |
| runtime_ms | INT, nullable |
| memory_mb | INT, nullable |
| submitted_at | TIMESTAMP |

### MySQL — users

| Field | Type |
|---|---|
| user_id | UUID (PK) |
| username | VARCHAR, unique |
| email | VARCHAR, unique |
| password_hash | VARCHAR |
| problems_solved | INT |
| rating | INT |
| created_at | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `submission:status:{submissionId}` | String | verdict JSON | 300s |
| `contest:leaderboard:{contestId}` | ZSET | userId → score | while contest active |
| `session:{sessionId}` | String | userId | 86400s |
| `rate:{userId}` | Counter | submission count | 60s |

## 5. Key Flows

### 5.1 Code Submission

```
Client → Submission Service
              ↓
    Validate: language supported, code size limit, rate limit
              ↓
    Write submission to DB (status=pending)
              ↓
    Publish to Kafka: {submissionId, code, language, problemId}
              ↓
    Return submissionId to client (202 Accepted)
              ↓
Client polls GET /submission/{id} every 2s
```

1. User submits code → Submission Service
2. Rate limit check (e.g., 5 submissions/min per user)
3. Write submission record to PostgreSQL with `status = pending`
4. Publish to Kafka `submissions` topic
5. Return `submissionId` immediately — async execution
6. Client polls for result every 2s (or WebSocket push)

### 5.2 Code Execution (Judge)

```
Kafka → Code Execution Service
              ↓
    Pull submission job
              ↓
    Spin up Docker sandbox
              ↓
    Compile code (if compiled language)
              ↓
    For each test case:
      Run code with input, enforce time + memory limits
      Compare output to expected
              ↓
    Determine verdict (all pass = Accepted, else first failure)
              ↓
    Write verdict to Submission DB
    Update Redis: submission:status:{id} = verdict
    Publish result to Kafka → Notification Service
```

**Sandbox constraints:**
- CPU: 1 core, time limit per test case (e.g., 1–3s)
- Memory: 256MB limit
- No network access (iptables rules)
- No filesystem writes outside /tmp
- Process killed on limit exceeded

**Verdict determination:**
```
Compile Error → stop, return CE
For each test case:
  TLE (time > limit) → return TLE
  MLE (memory > limit) → return MLE
  Runtime Error (crash, exception) → return RE
  Wrong Answer (output != expected) → return WA
All test cases pass → Accepted
```

### 5.3 Contest Leaderboard

Same as Leaderboard design — Redis ZSET per contest:
- On accepted submission: `ZINCRBY contest:leaderboard:{contestId} {score} {userId}`
- Score = problems solved × penalty time formula
- `ZREVRANGE` for top N; `ZREVRANK` for user's rank
- Real-time updates pushed via WebSocket

### 5.4 Custom Test Run

Same flow as submission but:
- Uses user-provided input instead of hidden test cases
- Not stored in submission history
- Lower priority in execution queue

## 6. Key Interview Concepts

### Sandbox Security
The most critical aspect. Code must not be able to:
- Access other users' data
- Make network calls (exfiltrate data, call external APIs)
- Fork-bomb or exhaust host resources
- Write to persistent storage

Solution: Docker container with:
- `--network none` (no network)
- `--memory 256m --cpus 0.5` (resource limits)
- `--read-only` filesystem with tmpfs for /tmp
- `seccomp` profile to restrict syscalls
- Run as non-root user
- Killed after time limit via `timeout` command

### Async Execution
Code execution takes 1–5s. Synchronous HTTP would hold connections open. Solution: async — return `submissionId` immediately, client polls or receives WebSocket push. Kafka queue absorbs submission bursts (contest start spike).

### Execution Queue Scaling
Contest start: 10K users submit simultaneously. Kafka buffers submissions. Code Execution Service scales horizontally — each instance handles one submission at a time (one Docker container). With 100 executor instances: 100 concurrent executions. Queue drains as executors process.

### Test Case Isolation
Each test case runs in a fresh process (not the same process reused). Prevents state leakage between test cases. Some judges reuse the process for performance — trade-off between speed and isolation.

### Plagiarism Detection
Post-submission async job: compare accepted solutions using code similarity algorithms (token-based, AST-based). Flag similar submissions for review. Not in critical path.

## 7. Failure Scenarios

### Code Execution Service Crash Mid-Execution
- Detection: Kafka message not acknowledged
- Recovery: Kafka redelivers to another executor; idempotent execution (submissionId dedup)
- Submission status remains `pending` until reprocessed

### Sandbox Escape Attempt
- Malicious code tries to break out of container
- Prevention: seccomp profile, non-root user, read-only filesystem, no network
- Detection: anomaly monitoring on container syscalls; alert security team

### Execution Queue Backup (Contest Spike)
- Detection: Kafka consumer lag grows; submission results delayed
- Recovery: auto-scale executor instances; users see "pending" status longer
- Prevention: pre-scale before known contest start times

### Wrong Verdict (Judge Bug)
- Scenario: judge incorrectly marks correct solution as WA
- Recovery: re-judge all submissions for affected problem; update verdicts
- Prevention: extensive test case validation; multiple judge instances cross-check results
