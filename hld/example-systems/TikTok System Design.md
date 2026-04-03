# TikTok (Short Video Platform) System Design

## System Overview
A short-form video platform where users upload, discover, and stream short videos (15s–3min) with a personalized feed powered by a recommendation engine, real-time engagement (likes, comments, shares), and a global CDN for low-latency video delivery.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- User registration and authentication
- Upload short videos (up to 3 min)
- Personalized video feed (For You Page)
- Like, comment, share, follow
- Search videos and users
- Live streaming (basic)
- Notifications (new follower, like, comment)

### Non-Functional Requirements
- Availability: 99.99%
- Latency: <200ms feed load; video starts within 1s
- Scalability: 1B+ users, 500M DAU, 1B+ videos served/day
- Read >> Write: feed reads vastly outnumber uploads
- Durability: uploaded videos must never be lost

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 500M DAU, each watches 50 videos/day, uploads 0.01 videos/day
- Average video size (raw): 100MB; encoded: 50MB across all resolutions
- Average video duration: 30s at 5Mbps = ~18MB per quality variant

### Traffic
```
Video views/day     = 500M × 50 = 25B views/day
Video views/sec     = 25B / 86400 ≈ 289K/sec → served by CDN

Uploads/day         = 500M × 0.01 = 5M videos/day
Uploads/sec         = 5M / 86400 ≈ 58/sec

Feed requests/sec   = 500M × 10 / 86400 ≈ 58K/sec
```

### Storage
```
Raw uploads/day     = 5M × 100MB = 500TB/day
Encoded/day         = 5M × 50MB  = 250TB/day
5-year storage      ≈ 250TB × 365 × 5 ≈ 456PB
```

## 3. Core Components

**API Gateway** — Auth, rate limiting, routing

**User Service** — Registration, login, profile, follow graph

**Video Upload Service** — Receives raw video, stores to S3, triggers encoding pipeline

**Video Processing Pipeline** — Chunker splits video into segments; Encoder encodes to multiple bitrates (360p, 720p, 1080p) and codecs; stores to S3; updates Video DB

**Feed Service** — Generates personalized For You Page; reads from pre-computed feed cache or calls Recommendation Service

**Recommendation Service** — ML-based ranking of videos for each user; considers watch history, likes, follows, trending; writes pre-computed feeds to Redis/feed cache

**Video Streaming Service** — Serves manifest files and video segments; validates auth; returns CDN URLs

**Engagement Service** — Handles likes, comments, shares; writes to Engagement DB; publishes events to Kafka

**Search Service** — Video and user search via Elasticsearch; synced via CDC

**Notification Service** — Kafka consumer; sends push/email notifications

**CDN** — Global edge delivery of video segments; primary delivery mechanism

**Video DB (Cassandra)** — Video metadata, view counts, engagement counts

**User DB (MySQL)** — User profiles, follow relationships

**Engagement DB (Cassandra)** — Likes, comments, shares — high write throughput

**Feed Cache (Redis)** — Pre-computed feeds per user (top 200 videos)

**S3** — Raw and encoded video storage

**Kafka** — Async event bus for engagement events, encoding pipeline, notifications

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| Cassandra (Video DB) | High read/write throughput for video metadata and counters |
| MySQL (User DB) | Structured user data, follow graph, ACID |
| Cassandra (Engagement) | Billions of likes/comments, append-only, time-series |
| Redis | Pre-computed feed cache, trending videos, session store |
| Elasticsearch | Full-text video/user search |
| S3 + CDN | Petabyte video storage, global delivery |

### Cassandra — videos

| Field | Type |
|---|---|
| video_id | UUID (PK) |
| creator_id | UUID |
| title | VARCHAR |
| description | TEXT |
| s3_url | TEXT |
| manifest_url | TEXT |
| duration_sec | INT |
| resolution_variants | LIST\<VARCHAR\> |
| view_count | COUNTER |
| like_count | COUNTER |
| status | TEXT (processing / active / deleted) |
| created_at | TIMESTAMP |

### MySQL — users

| Field | Type |
|---|---|
| user_id | UUID (PK) |
| username | VARCHAR, unique |
| email | VARCHAR, unique |
| password_hash | VARCHAR |
| follower_count | INT |
| following_count | INT |
| created_at | TIMESTAMP |

### MySQL — follows

| Field | Type |
|---|---|
| follower_id | UUID |
| followee_id | UUID |
| created_at | TIMESTAMP |

### Cassandra — comments

| Field | Type |
|---|---|
| video_id | UUID (partition key) |
| comment_id | TIMEUUID (clustering) |
| user_id | UUID |
| content | TEXT |
| created_at | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `feed:{userId}` | List | ordered videoIds (top 200) | 3600s |
| `trending:global` | ZSET | videoId → score | 300s |
| `video:meta:{videoId}` | String | video metadata JSON | 600s |
| `session:{sessionId}` | String | userId | 86400s |

## 5. Key Flows

### 5.1 Video Upload & Processing

1. Creator uploads video → Video Upload Service → store raw to S3
2. Trigger encoding pipeline via Kafka
3. Chunker splits into 2–5s segments
4. Encoder produces 360p / 720p / 1080p variants in parallel (same as Netflix)
5. Encoded segments stored to S3; manifest file generated
6. Video DB updated with `status = active`, manifest URL
7. CDC → Elasticsearch (video now searchable)
8. Feed/Recommendation Service notified to consider for followers' feeds

### 5.2 Feed Generation (For You Page)

Two approaches:

**Push (fan-out on write):**
- When creator posts, push video to all followers' feed caches
- Fast reads (pre-computed), expensive writes for creators with millions of followers
- Good for users with small follower counts

**Pull (fan-out on read):**
- On feed request, fetch recent videos from followed creators + Recommendation Service
- Expensive reads, cheap writes
- Good for mega-influencers (avoid fan-out to 100M followers)

**Hybrid (TikTok approach):**
- Push for creators with <10K followers
- Pull + Recommendation for mega-influencers
- Recommendation Service fills gaps with trending/personalized content

### 5.3 Video Streaming

Same as Netflix — manifest file → CDN segments → ABR streaming. See Netflix design for full detail.

### 5.4 Engagement (Likes, Comments)

1. User likes video → Engagement Service → write to Cassandra
2. Publish `LIKE` event to Kafka
3. Kafka consumer updates `like_count` counter in Video DB (Cassandra COUNTER type — atomic increment)
4. Notification Service sends push to creator

## 6. Key Interview Concepts

### Fan-out Problem
A creator with 100M followers posts a video. Pushing to all 100M feed caches = 100M Redis writes. Solution: hybrid approach — push to small follower counts, pull + recommend for large. Threshold typically 10K–100K followers.

### For You Page vs Following Feed
Following feed = videos from accounts you follow (pull-based, straightforward). For You Page = ML-ranked mix of followed + recommended content. TikTok's FYP is what drives engagement — recommendation quality is the product differentiator.

### Video Counter Accuracy
Cassandra COUNTER type provides atomic increments without locking. For extreme scale, use approximate counting: increment a Redis counter (fast), batch-flush to Cassandra every 30s. Slight staleness in view counts is acceptable.

### CDN Strategy
Same as Netflix — video segments cached at edge nodes globally. Cache hit rate >99% for popular videos. Long TTL on segments (they never change after encoding). Short TTL on manifest files (may update with new quality variants).

## 7. Failure Scenarios

### Encoding Pipeline Failure
- Recovery: Kafka retains encoding jobs; another encoder picks up; idempotent encoding
- Video stays in `processing` status until complete; creator notified of delay

### Feed Cache Miss
- Recovery: fall back to pull-based feed generation from followed creators + trending
- Recommendation Service generates feed on demand; populate cache for next request

### CDN Node Failure
- Recovery: CDN reroutes to next nearest edge; ABR drops quality to compensate
- S3 origin always available as fallback

### Recommendation Service Failure
- Recovery: fall back to chronological feed from followed creators
- Graceful degradation — users see less personalized but functional feed
