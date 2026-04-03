# Netflix System Design

## System Overview
A video streaming platform where users subscribe, browse and search content, and stream high-quality video on demand — with adaptive bitrate streaming, a global CDN, and a content upload/encoding pipeline for creators.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- User registration, authentication, and subscription management
- Browse and search movies/shows
- Stream video with adaptive quality based on network speed
- Resume playback from where user left off
- Content upload, processing, and encoding pipeline (internal)
- Payment and subscription billing

### Non-Functional Requirements
- Availability: 99.99% — streaming must always work
- Latency: <200ms to start playback; seamless adaptive quality switching
- Scalability: 200M+ subscribers, 100M+ concurrent streams at peak
- Throughput: petabytes of video served daily
- Durability: uploaded content must never be lost
- Security: DRM (Digital Rights Management), auth on all streaming requests

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 200M subscribers, 100M DAU
- 100M concurrent streams at peak (2hr avg watch time)
- Average bitrate: 5 Mbps (mix of SD/HD/4K)
- Video catalog: 10K titles, each encoded at 5 resolutions × 2 codecs = 10 variants
- Average video size per variant: 4GB (2hr movie at 5Mbps)
- New content uploads: 100 videos/day

### Traffic
```
Concurrent streams       = 100M
Bandwidth required       = 100M × 5 Mbps = 500 Tbps  → served by CDN, not origin
CDN origin pull          = ~1% of traffic = 5 Tbps

Search/browse requests   = 100M DAU × 10 req/day / 86400 ≈ 11.5K/sec
```

### Storage
```
Per video (all variants) = 10 variants × 4GB = 40GB
Full catalog             = 10K titles × 40GB = 400TB
With replication (3×)    = 1.2PB on S3

New uploads/day          = 100 videos × 40GB = 4TB/day
```

### Memory (Cache)
```
Hot content metadata     = 10K titles × 10KB = 100MB (trivially cacheable)
User watch history       = 200M × 1KB = 200GB
Active sessions          = 100M × 500B = 50GB
```

## 3. Core Components

**LB + API Gateway** — Routing, authentication, authorization, rate limiting, round-robin load balancing across services

**User Service** — Registration, login, JWT issuance, profile management; reads/writes to User DB (MySQL)

**Subscription Service** — Manages subscription plans, validity, and status; integrates with Payment Service; reads/writes to Subscription DB (MySQL)

**Payment Service** — Handles billing, payment gateway integration, invoices; writes to Payment DB (MySQL)

**Search Service** — Content discovery via Elasticsearch; full-text search on title, description, cast; kept in sync with Video DB via CDC

**Play Service** — Core streaming service; validates auth + subscription, fetches manifest file from Video DB, returns CDN URLs for video segments to client

**Uploader** — Internal service for content ingestion; receives raw video uploads, stores to S3, triggers Chunker

**Chunker** — Splits raw video into small segments (2–10s each), generates manifest files (.m3u8 for HLS, .mpd for DASH), publishes to Kafka for encoding

**Video Encoder** — Kafka consumer; encodes each chunk into multiple bitrates (480p, 720p, 1080p, 4K) and codecs (H.264, H.265); stores encoded segments back to S3

**Consumer Service** — Kafka consumer for general events (watch history, analytics, recommendations)

**Notification Service** — Kafka consumer; sends emails/push for new content, billing alerts, account activity

**Video DB (MongoDB)** — Stores video metadata: title, description, manifest file URLs, available resolutions

**User DB (MySQL)** — User profiles, credentials, watch history, resume positions

**Subscription DB (MySQL)** — Subscription plans, user subscription status, expiry

**Payment DB (MySQL)** — Payment records, invoices, billing history

**S3** — Raw uploaded videos + all encoded video segments and manifest files

**CDN** — Serves video segments to users globally; caches segments at edge nodes close to users; primary delivery mechanism for all streaming traffic

**Elasticsearch** — Full-text search index for content discovery; synced from Video DB via CDC

**Kafka** — Async event bus for video processing pipeline and downstream consumers

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| MongoDB (Video DB) | Flexible schema for video metadata, nested manifest info, varying attributes per title |
| MySQL (User DB) | Structured user data, ACID for auth and watch history, relational |
| MySQL (Subscription DB) | ACID for subscription state — billing correctness is critical |
| MySQL (Payment DB) | PCI-DSS compliance, ACID transactions, audit trail |
| Elasticsearch | Full-text search on title/description/cast, faceted filtering by genre/year |
| S3 | Petabyte-scale video storage, high durability, CDN integration |
| Redis | Sessions, hot metadata cache, rate limiting |
| Kafka | Video processing pipeline, event fan-out to consumers |

### MongoDB — videos

| Field | Type |
|---|---|
| video_id | ObjectId (PK) |
| title | STRING |
| description | TEXT |
| manifest_file | STRING (S3 URL to .m3u8 / .mpd) |
| available_resolutions | ARRAY\<STRING\> (480p, 720p, 1080p, 4k) |
| available_codecs | ARRAY\<STRING\> (H.264, H.265) |
| duration_seconds | INT |
| genre | ARRAY\<STRING\> |
| cast | ARRAY\<STRING\> |
| release_year | INT |
| thumbnail_url | STRING |
| created_at | TIMESTAMP |

### MySQL — users

| Field | Type |
|---|---|
| user_id | UUID (PK) |
| name | VARCHAR |
| email | VARCHAR, unique |
| password_hash | VARCHAR |
| subscription_status | ENUM (active / inactive / trial) |
| expiry_date | TIMESTAMP |
| created_at | TIMESTAMP |

### MySQL — watch_history

| Field | Type |
|---|---|
| id | UUID (PK) |
| user_id | UUID (FK → users) |
| video_id | VARCHAR |
| watched_duration_sec | INT |
| last_watched_at | TIMESTAMP |

### MySQL — subscriptions

| Field | Type |
|---|---|
| subscription_id | UUID (PK) |
| user_id | UUID |
| name | VARCHAR (Basic / Standard / Premium) |
| price | DECIMAL |
| currency | VARCHAR |
| validity | INT (days) |
| created_at | TIMESTAMP |

### MySQL — payments

| Field | Type |
|---|---|
| payment_id | UUID (PK) |
| user_id | UUID |
| amount | DECIMAL |
| currency | VARCHAR |
| status | ENUM (pending / success / failed / refunded) |
| time | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `session:{sessionId}` | String | `{userId, deviceId}` | 86400s |
| `video:meta:{videoId}` | String | video metadata JSON | 3600s |
| `user:resume:{userId}:{videoId}` | String | last watched position (seconds) | 30 days |
| `rate:{userId}` | Counter | request count | 60s window |

## 5. Key Flows

### 5.1 Auth & Subscription Check

1. User registers → User Service → write to MySQL User DB → return JWT
2. Login → validate credentials → generate JWT (1hr) + refresh token → store session in Redis
3. Every request: API Gateway validates JWT + Redis session
4. Play Service additionally checks `subscription_status` and `expiry_date` before serving content — expired subscription → 403

### 5.2 Content Search & Browse

```
Client → API Gateway → Search Service → Elasticsearch
                                              ↓
                                    ranked content list
                                              ↓
                       Metadata enrichment from Redis cache → MongoDB fallback
                                              ↓
                              Thumbnails served via CDN (S3)
```

1. User searches "Stranger Things" → Search Service
2. Elasticsearch query: full-text on `title`, `description`, `cast` + filters (genre, year)
3. Returns ranked list of matching titles
4. Client fetches full metadata from Redis cache (TTL 1hr) → fallback to MongoDB
5. Thumbnails served from CDN

**CDC Sync (Video DB → Elasticsearch):**
- New content added to MongoDB → CDC captures change → Kafka → Elasticsearch consumer updates index
- Eventual consistency — search index may lag a few seconds after new content is added

### 5.3 Video Playback (Play Service)

```
Client → API Gateway → Play Service
                            ↓
              Validate JWT + subscription status
                            ↓
              Fetch manifest file URL from Video DB (Redis → MongoDB)
                            ↓
              Return manifest file URL to client
                            ↓
              Client fetches manifest (.m3u8 / .mpd) from CDN
                            ↓
              Client fetches video segments from CDN
                            ↓ (CDN miss)
              CDN pulls segment from S3 origin, caches at edge
```

1. User clicks play → `GET /play/{videoId}`
2. Play Service validates JWT and active subscription
3. Fetches manifest file URL from Redis cache → fallback to MongoDB
4. Returns manifest URL (pointing to CDN)
5. Client downloads manifest file from CDN — manifest lists all segment URLs at all quality levels
6. Client picks starting quality based on current bandwidth estimate
7. Client fetches video segments from CDN edge node (nearest to user)
8. CDN serves from cache; on miss, pulls from S3 and caches for subsequent users

### 5.4 Adaptive Bitrate Streaming (ABR)

This is how Netflix adjusts quality seamlessly as network conditions change.

**How it works:**
- Video is pre-encoded into multiple bitrates: 480p (1Mbps), 720p (3Mbps), 1080p (5Mbps), 4K (15Mbps)
- Each quality level is split into small segments of 2–10 seconds
- Manifest file (.m3u8 for HLS / .mpd for DASH) lists all segments at all quality levels

**HLS Manifest example:**
```
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=1000000,RESOLUTION=640x360
http://cdn.example.com/low.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720
http://cdn.example.com/medium.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
http://cdn.example.com/high.m3u8
```

**Segment-level manifest:**
```
#EXTM3U
#EXT-X-TARGETDURATION:10
http://cdn.example.com/seg1.ts   (2k)
http://cdn.example.com/seg2.ts   (2k)
http://cdn.example.com/seg3.ts   (4k)  ← quality switched up here
```

**ABR algorithm (client-side):**
1. Client measures download speed of each segment
2. Maintains a buffer (e.g., 30s of video pre-downloaded)
3. If buffer is healthy and speed is high → request next segment at higher quality
4. If buffer is draining (slow network) → drop to lower quality
5. Quality switches happen at segment boundaries — seamless to user

**Example:**
```
User starts on WiFi (10Mbps) → client requests 1080p segments
User switches to mobile (2Mbps) → buffer starts draining
Client detects → switches to 480p for next segment
User returns to WiFi → gradually steps back up: 480p → 720p → 1080p
```

### 5.5 Video Upload & Encoding Pipeline

```
Uploader → S3 (raw video)
               ↓
           Chunker
               ↓ (splits into 2-10s segments, generates manifest)
           Kafka (segment jobs)
               ↓
         Video Encoder (multiple instances)
               ↓ (encodes each segment at all bitrates + codecs)
           S3 (encoded segments)
               ↓
         Video DB updated (manifest URLs, available resolutions)
               ↓
         CDC → Elasticsearch (content now searchable)
```

1. Content team uploads raw video → Uploader stores to S3
2. Chunker splits video into 2–10s segments
3. Generates master manifest file referencing all segments
4. Publishes encoding jobs to Kafka (one job per segment per quality level)
5. Video Encoder instances consume jobs in parallel:
   - Encode segment to H.264 and H.265
   - At 480p, 720p, 1080p, 4K
   - Compress to target bitrate
   - Store encoded segment to S3
6. Once all segments encoded: update Video DB with manifest URL and available resolutions
7. CDC propagates to Elasticsearch — content becomes searchable

**Why parallel encoding via Kafka:**
A 2hr movie at 10 quality variants = thousands of segment jobs. Kafka allows N encoder instances to process jobs in parallel — encoding time scales with number of encoders, not video length.

### 5.6 Resume Playback

1. User stops watching at 45:32 → Play Service writes `user:resume:{userId}:{videoId} = 2732` to Redis (TTL 30 days)
2. Async: Consumer Service flushes Redis resume positions to MySQL `watch_history` every 5 min
3. User reopens video → Play Service reads resume position from Redis → returns to client
4. Client seeks to position before requesting first segment

### 5.7 Subscription & Payment Flow

1. User selects plan → Subscription Service creates subscription record in MySQL
2. Calls Payment Service with `{userId, amount, idempotencyKey}`
3. Payment Service calls external gateway
4. On success: write to Payment DB, publish `PAYMENT_SUCCESS` to Kafka
5. Subscription Service consumes event → sets `subscription_status = active`, `expiry_date = now + validity`
6. Notification Service sends confirmation email
7. On renewal: cron job checks expiring subscriptions, triggers Payment Service for auto-renewal
8. On failure: `subscription_status = inactive`, user loses streaming access

## 6. Key Interview Concepts

### HLS vs DASH
Both are adaptive bitrate streaming protocols — video split into segments, manifest file describes available qualities.

- HLS (HTTP Live Streaming): Apple standard, `.m3u8` manifest, `.ts` segments; native support on iOS/Safari
- DASH (Dynamic Adaptive Streaming over HTTP): open standard, `.mpd` manifest, `.mp4` segments; used on Android, browsers

Netflix uses both depending on device. The architecture is identical — only the manifest format and segment container differ.

### Why CDN is Everything
500 Tbps of streaming traffic cannot come from origin servers. CDN edge nodes cache video segments close to users — a user in Mumbai gets segments from a Mumbai edge node, not a US data center. CDN hit rate for popular content is >99%. Origin (S3) only serves cache misses and new/unpopular content.

### Segment Size Trade-off
- Smaller segments (2s): faster quality switching, more HTTP requests, higher overhead
- Larger segments (10s): fewer requests, slower to react to network changes, larger buffer jumps

Netflix uses 2–4s segments for responsive ABR. The manifest file lists hundreds of segment URLs — client fetches them sequentially, switching quality at any boundary.

### Why Kafka for Encoding Pipeline
A 2hr movie generates thousands of encoding jobs (segments × quality levels × codecs). Kafka distributes jobs across N encoder instances in parallel. Adding more encoders reduces total encoding time linearly. Without Kafka, a single encoder would take hours; with 100 encoders, minutes.

### MongoDB for Video Metadata
Video metadata has a flexible, nested structure — manifest URLs, multiple resolutions, cast arrays, genre arrays. MongoDB's document model fits naturally. No joins needed — a single document fetch returns everything needed for playback. Contrast with relational tables which would require joins across videos, resolutions, cast, etc.

### CDC for Search Sync
Video DB (MongoDB) → CDC → Kafka → Elasticsearch. Same pattern as ecommerce and ticket booking. Decouples content ingestion from search indexing. If Elasticsearch is down, content is still stored in MongoDB and CDC replays missed changes on recovery.

### DRM (Digital Rights Management)
Video segments are encrypted. Client must obtain a license key from a DRM server (Widevine for Android/Chrome, FairPlay for Apple) before decryption. License is tied to the user's subscription status — expired subscription = no license = can't decrypt. This prevents downloaded segments from being played without a valid subscription.

### Watch History & Resume — Write Optimization
100M users updating watch position every few seconds = massive write load. Solution: write to Redis first (fast, in-memory), batch flush to MySQL every 5 minutes. Acceptable to lose last 5 min of position on Redis failure — user resumes from 5 min earlier, minor UX issue.

### CAP Trade-off
- Streaming (Play Service): AP — availability over consistency; slightly stale manifest URL is fine
- Subscription check: CP — must be consistent; expired user must not stream
- Watch history: AP — eventual consistency fine; losing a few seconds of position is acceptable
- Payments: CP — strong consistency, ACID

## 7. Failure Scenarios

### CDN Node Failure
- Impact: users in that region experience buffering or quality drops
- Recovery: CDN automatically reroutes to next nearest edge node; client ABR algorithm drops quality to compensate for higher latency
- Prevention: CDN provider has redundant PoPs; S3 origin always available as fallback

### S3 Region Outage
- Impact: CDN cache misses can't be served; new uploads fail
- Recovery: S3 cross-region replication; CDN serves from cache for popular content (high hit rate means most users unaffected); new uploads queued and retried
- Prevention: multi-region S3 replication; CDN cache TTL long enough to survive brief S3 outage

### Video Encoder Failure (Mid-Job)
- Impact: some video segments not encoded; content partially available
- Recovery: Kafka retains unacknowledged jobs; another encoder instance picks up the job; idempotent encoding (re-encoding same segment produces same output)
- Prevention: multiple encoder instances; dead letter queue for repeatedly failing jobs; alert ops team

### Play Service Failure
- Impact: users can't start new streams; active streams continue (client has segments buffered)
- Recovery: load balancer routes to healthy instances; stateless service restarts quickly; manifest URLs cached in Redis so recovery is fast
- Prevention: multiple Play Service instances; health checks; auto-scaling

### Subscription Check Inconsistency
- Scenario: user cancels subscription but Play Service serves stale cached status
- Recovery: subscription status cache TTL is short (60s max); on cache miss, always check MySQL; on cancellation, actively invalidate cache
- Prevention: subscription changes publish invalidation event to Kafka → Play Service clears cache immediately

### MySQL Primary Failure (User / Subscription DB)
- Impact: login, subscription checks, watch history writes fail
- Recovery: promote replica (<30s); Redis sessions keep active users streaming during failover; watch history writes queue in Redis and flush after recovery
- Prevention: synchronous replication; automated failover; Redis as buffer for writes

### Kafka Lag (Encoding Pipeline)
- Impact: new content takes longer to become available; existing content unaffected
- Recovery: scale up encoder instances; Kafka retains jobs; encoding catches up
- Prevention: monitor encoding queue depth; auto-scale encoders based on queue lag; separate Kafka topics for encoding vs user events

### Payment Failure on Renewal
- Detection: payment gateway returns failure on auto-renewal
- Recovery: retry 3 times over 3 days; notify user via email/push; after grace period, set `subscription_status = inactive`; user loses streaming access
- Prevention: idempotency key on renewal; store payment method securely for retry; dunning management (retry logic with escalating notifications)
