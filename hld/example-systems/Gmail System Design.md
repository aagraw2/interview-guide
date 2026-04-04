# Gmail (Email Client) System Design

## System Overview
A full-featured email client and server (think Gmail / Outlook) where users compose, send, and receive emails — with SMTP relay for outbound delivery, inbound SMTP for receiving, spam/malware filtering, attachment scanning, full-text search, and contact autocomplete.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- User registration and authentication
- Compose and send emails (to internal and external recipients)
- Receive inbound emails from external SMTP servers
- Inbox, drafts, sent, spam folders (mailbox management)
- Attachment upload and download
- Full-text email search
- Contact autocomplete (personal contacts + directory)
- Spam and malware filtering
- Read/unread status, labels, threading

### Non-Functional Requirements
- Availability: 99.99%
- Latency: <500ms for send; <200ms for inbox load
- Scalability: 1B+ users, billions of emails/day
- Durability: Emails must never be lost
- Security: DMARC/DKIM/SPF, TLS, spam filtering, malware scanning

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 1B users, 500M DAU
- 10 emails sent/user/day = 5B emails/day
- Average email size: 75KB (with attachments avg)
- 60% emails are spam (filtered before delivery)
- Read:Write ratio = 5:1

### Traffic
```
Emails sent/sec     = 5B / 86400 ≈ 58K/sec
Inbound SMTP/sec    = 58K/sec (external + internal)
Search queries/sec  = 500M × 5 / 86400 ≈ 29K/sec
```

### Storage
```
Emails/day          = 5B × 75KB = 375TB/day
Attachments         = stored in S3 separately
User mailbox avg    = 15GB per user
Total               = 1B × 15GB = 15EB
```

## 3. Core Components

**Load Balancer & API Gateway** — Auth, rate limiting, routing for web/mobile clients

**User Service** — Registration, login, JWT issuance; User DB (userId, email, password_hash, metadata); User Cache (Redis) for fast lookups; sharded by email using consistent hashing

**Compose Service** — Handles email composition; saves drafts to Draft DB; validates recipients

**Mail Send Service** — Orchestrates outbound email sending; validates, routes to Outbox Consumer

**Outbox Consumer Service** — Kafka consumer; processes outbound emails; calls Spam/Malware Filtering, Policy Check, Validation DB checks; routes to Outbox Delivery Orchestrator

**Outbox Delivery Orchestrator** — Routes email to correct destination:
- Internal recipient (same domain): deliver directly to recipient's Mailbox DB
- External recipient: forward to SMTP Relay Workers

**SMTP Relay Workers** — Pool of workers that connect to external recipient SMTP servers; handle SMTP handshake, TLS, retry on failure; manage sending IP reputation

**Inbound SMTP Service** — Receives emails from external SMTP servers (Outlook, Yahoo, etc.); validates sender (SPF/DKIM/DMARC); passes to Kafka for processing

**Attachment Scanner** — Scans attachments for malware/viruses; validates file types; stores clean attachments to S3; returns signed URL

**Spam/Malware Filtering Service** — ML-based spam scoring + rule-based filters; checks sender reputation, content patterns, attachment hashes; marks as spam or blocks

**Policy Check** — Enforces organizational policies: attachment size limits, blocked domains, DLP (data loss prevention) rules

**Validation DB** — Stores validation rules: `sender_user_id, recipient_email, policy_user_id, route_type (INTERNAL/EXTERNAL), spam_check, attachment_check, status`

**Search Service** — Full-text email search via Elasticsearch; indexes subject, body, sender, recipients

**Aggregator Service** — CDC pipeline from Mailbox DB to Elasticsearch for search indexing

**Mailbox DB** — Per-user mailbox storage: `Outbox_Email` and `MailboxItem` tables

**Mailbox Metadata DB** — Metadata about mailbox items (read status, labels, folder, thread_id)

**Draft DB** — Temporary storage for unsent drafts

**MX Cache** — Caches MX record lookups (DNS → mail server for a domain); avoids repeated DNS queries

**MX Resolver** — Resolves MX records for external domains; used by SMTP Relay Workers to find recipient's mail server

**Notification Service** — Kafka consumer; sends push/in-app notifications for new emails

**Kafka** — Two topics: `outbound_send_request` and `inbound_send_request`

**S3** — Attachment storage; signed URLs for download; virus-scanned before storage

**Elasticsearch** — Full-text search index for emails

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL (Mailbox DB) | Per-user email storage, ACID, relational queries for threading |
| PostgreSQL (Draft DB) | Temporary draft storage, ACID |
| PostgreSQL (Validation DB) | Policy and routing rules, ACID |
| Redis (User Cache) | Fast user lookup, session store; sharded by email using consistent hashing |
| Elasticsearch | Full-text search on subject, body, sender |
| S3 | Attachment storage, large blobs, CDN-friendly |
| Kafka | Async email processing pipeline |

### PostgreSQL — mailbox (Outbox_Email + MailboxItem)

| Field | Type |
|---|---|
| message_id | UUID (PK) |
| user_id | UUID |
| thread_id | UUID |
| sender | VARCHAR |
| recipients | JSONB (to, cc, bcc) |
| subject | VARCHAR |
| body | TEXT |
| attachment_urls | JSONB |
| folder | ENUM (inbox / sent / drafts / spam / trash) |
| is_read | BOOLEAN |
| labels | ARRAY\<VARCHAR\> |
| created_at | TIMESTAMP |

### PostgreSQL — validation_db

| Field | Type |
|---|---|
| message_id | UUID |
| sender_user_id | UUID |
| recipient_email | VARCHAR |
| policy_user_id | UUID |
| route_type | ENUM (INTERNAL / EXTERNAL) |
| spam_check | ENUM (PASS / FAIL / PENDING) |
| attachment_check | ENUM (PASS / FAIL / PENDING) |
| status | VARCHAR |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `user:{email}` | String | userId + metadata | 3600s |
| `mx:{domain}` | String | MX record | 86400s |
| `session:{sessionId}` | String | userId | 86400s |

## 5. Key Flows

### 5.1 Auth

1. User logs in → User Service → validate credentials → JWT + session in Redis
2. API Gateway validates JWT on every request
3. User Cache sharded by email using consistent hashing — 25/50 most sent emails cached per user for autocomplete

### 5.2 Outbound Email Send Flow

```
Client → Mail Send Service → Kafka (outbound_send_request)
                                        ↓
                              Outbox Consumer Service
                              (spam check, policy check, validation)
                                        ↓
                              Outbox Delivery Orchestrator
                                        ↓
                    Internal?                    External?
                       ↓                              ↓
              Mailbox DB (recipient)         SMTP Relay Workers
                                                      ↓
                                              MX Resolver → MX Cache
                                                      ↓
                                              Receiver SMTP Server
```

1. User composes email, clicks Send → Mail Send Service
2. Publishes to Kafka `outbound_send_request`
3. Outbox Consumer Service consumes:
   - Spam/Malware Filtering: score email content + attachments
   - Policy Check: attachment size, blocked domains, DLP rules
   - Write to Validation DB with check results
4. If checks pass → Outbox Delivery Orchestrator
5. Determine route: internal (same domain) or external
6. Internal: write directly to recipient's Mailbox DB
7. External: SMTP Relay Workers look up MX record (MX Cache → MX Resolver → DNS), connect to recipient's SMTP server, deliver via SMTP with TLS

**Steps to connect to external SMTP server:**
1. DNS MX record lookup for recipient domain (e.g., `outlook.com`)
2. TCP connection to MX server on port 25
3. SMTP handshake (EHLO)
4. STARTTLS (upgrade to TLS)
5. AUTH (if required)
6. MAIL FROM, RCPT TO, DATA
7. SMTP 250 OK → delivery confirmed

### 5.3 Inbound Email Receive Flow

```
External SMTP → Inbound SMTP Service
                        ↓
              Validate sender (SPF/DKIM/DMARC)
                        ↓
              Kafka (inbound_send_request)
                        ↓
              Spam/Malware Filtering
              Attachment Scanner → S3
                        ↓
              Delivery Consumer → Mailbox DB
                        ↓
              Notification Service → push to recipient
```

1. External server (Outlook, Yahoo) connects to our SMTP server
2. Inbound SMTP Service validates: SPF (sender IP authorized?), DKIM (signature valid?), DMARC (policy check)
3. Publishes to Kafka `inbound_send_request`
4. Spam/Malware Filtering scores the email
5. Attachment Scanner scans attachments, stores clean ones to S3
6. Delivery Consumer writes to recipient's Mailbox DB
7. Notification Service sends push notification to recipient

### 5.4 Attachment Handling

1. User attaches file → Attachment Scanner
2. Virus scan (ClamAV or similar)
3. File type validation (block .exe, .bat, etc.)
4. Store to S3: `attachments/{userId}/{messageId}/{filename}`
5. Return signed URL (valid 24hr for download)
6. Attachment URL stored in email record

### 5.5 Email Search

1. Aggregator Service (CDC) captures new emails from Mailbox DB → Elasticsearch
2. User searches "invoice from Amazon" → Search Service → Elasticsearch
3. Full-text on subject, body, sender, recipients; filter by date, folder, label
4. Returns matching emails with snippets

### 5.6 Contact Autocomplete

When user types in To field:
1. Check local cache: `user:{userId}:contacts` — 25/50 most recently emailed contacts
2. Cache hit: return suggestions instantly
3. Cache miss: scan UserDB for contacts
4. Miss: scan the UserDB table (broader directory search)

**Scenarios:**
- Sending to unknown address (first time): no autosuggestion
- Sending to outside org: no autosuggestion (security)
- Sending to unknown (first time override): show suggestion on click/enter
- Sending to known (within org): show contact (on click or enter)

## 6. Key Interview Concepts

### SMTP Protocol Flow
SMTP is the protocol for email transfer between servers. Key steps: DNS MX lookup → TCP connection → EHLO → STARTTLS → MAIL FROM → RCPT TO → DATA → 250 OK. Understanding this is essential for explaining how emails actually travel between Gmail and Outlook.

### SPF / DKIM / DMARC
Three DNS-based mechanisms that prove email legitimacy:
- **SPF:** "Only these IPs can send from our domain" — DNS TXT record listing authorized sending IPs
- **DKIM:** Cryptographic signature on email headers — receiving server verifies with public key in DNS
- **DMARC:** Policy for what to do if SPF/DKIM fail (reject/quarantine/none) + reporting

Without these, emails go to spam or are rejected. All three must be configured for good deliverability.

### MX Records and Routing
MX (Mail Exchange) records in DNS tell senders which server handles email for a domain. `dig MX gmail.com` returns Google's mail servers. SMTP Relay Workers look up MX records to find where to deliver external emails. MX Cache avoids repeated DNS lookups for popular domains.

### Internal vs External Routing
Internal emails (user@gmail.com → user2@gmail.com) never leave the system — delivered directly to Mailbox DB. External emails (user@gmail.com → user@outlook.com) go through SMTP Relay Workers. This distinction is important for latency and security.

### Spam Filtering at Scale
5B emails/day, 60% spam = 3B spam emails to filter. Two-layer approach:
- Fast rules (IP reputation, known spam domains, header checks) — block at SMTP level
- ML model (content analysis, behavioral patterns) — score remaining emails
- Threshold: score > 0.8 → spam folder; score > 0.95 → block entirely

### Mailbox Sharding
1B users × 15GB = 15EB of mailbox data. Shard by `user_id` (hash-based). All emails for a user on one shard — efficient inbox queries. Hot users (high email volume) may need dedicated shards.

### User Cache with Consistent Hashing
User Service shards the User Cache across Redis nodes using consistent hashing on email address. This ensures the same user always hits the same cache node, and adding/removing nodes only moves a fraction of keys.

### Outbox Pattern for Reliability
Mail Send Service writes to Draft DB and publishes to Kafka. If service crashes after DB write but before Kafka publish, the email is lost. Solution: outbox pattern — write to DB and outbox table in same transaction; CDC publishes to Kafka. Guarantees at-least-once delivery.

## 7. Failure Scenarios

### SMTP Relay Worker Failure
- Detection: delivery failures, health check
- Recovery: Kafka retains unacknowledged messages; another worker picks up; retry with exponential backoff
- Prevention: multiple relay workers; dead letter queue for permanently failed deliveries

### Inbound SMTP Overload (Email Bomb)
- Detection: inbound connection rate spikes
- Recovery: rate limit per sending IP; reject connections exceeding threshold; greylisting (temporary reject → legitimate servers retry)
- Prevention: IP reputation filtering at SMTP level before processing

### Spam Filter False Positive
- Scenario: legitimate email marked as spam
- Recovery: user moves to inbox → feedback signal to ML model; whitelist sender
- Prevention: conservative threshold for blocking; aggressive threshold only for spam folder

### Elasticsearch Failure
- Impact: search unavailable; email delivery unaffected
- Recovery: graceful degradation — show "search unavailable"; CDC replays missed emails on recovery
- Prevention: Elasticsearch cluster with replicas

### Mailbox DB Failure
- Impact: email delivery and inbox reads fail
- Recovery: promote replica (<30s); Kafka retains inbound emails; delivery retried after recovery
- Prevention: synchronous replication; automated failover; Kafka as durable buffer
