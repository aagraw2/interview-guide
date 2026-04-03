# YouTube System Design

## System Overview
A video sharing platform where users upload, discover, and watch long-form videos with comments, likes, subscriptions, and a recommendation-driven home feed. Similar to Netflix for streaming architecture but with user-generated content, public discovery, and monetization.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- User registration, authentication, channel management
- Upload videos (up to 12hr / 256GB)
- Watch videos with adaptive quality
- Search videos by title, description, channel
- Subscribe to channels; personalized home feed
- Like, dislike, comment on videos
- Video recommendations
- Notifications for new uploads from subscribed channels

### Non-Functional Requirements
- Availability: 99.99%
- Latency: <200ms feed load; video starts within 1s
- Scalability: 2B+ users, 500M+ videos, 1B+ hours watched/day
- Read >> Write: views vastly outnumber uploads
- Durability: uploaded videos must never be lost

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 2B users, 500M DAU
- 500 hours of video uploaded per minute
- 1B hours watched/day
- Average video: 10 min, encoded to 5 quality variants at avg 3GB total
- Read:Write = 10000:1

### Traffic
```
Uploads/min         = 500 hours = 30K min of video/min
Uploads/sec         = 500/sec (raw video minutes)

Views/day           = 1B hours × 60min / 10min avg = 6B views/day
Views/sec           = 6B / 86400 ≈ 69K/sec → CDN

Feed requests/sec   = 500M × 5 / 86400 ≈ 29K/sec
```

### Storage
```
Uploads/day         = 500hr/min × 60min × 24hr = 720K hr/day
                    = 720K × 60min × 3GB/10min ≈ 13PB/day (encoded)
Total catalog       = 500M videos × 3GB = 1.5EB
```

## 3. Core Components

**API Gateway** — Auth, rate limiting, routing

**User Service** — Registration, login, channel management, subscription graph

**Video Upload Service** — Receives raw video upload; stores to S3; triggers processing pipeline

**Video Processing Pipeline** — Same as Netflix/TikTok: Chunker → Encoder (multiple bitrates) → S3 → Video DB update

**Video Service** — Video metadata CRUD; view count updates; CDC to Elasticsearch

**Feed Service** — Personalized home feed; mix of subscribed channels + recommendations

**Recommendation Service** — ML-based video ranking; considers watch history, likes, search, trending

**Search Service** — Full-text video search via Elasticsearch; CDC from Video DB

**Playback Service** — Returns manifest URL for video streaming; validates auth

**Engagement Service** — Likes, dislikes, comments; writes to Engagement DB; Kafka events for counters

**Notification Service** — Kafka consumer; push/email for new uploads, comments, milestones

**Analytics Service** — Kafka consumer; processes view events for watch time, retention, revenue

**Video DB (Cassandra)** — Video metadata, view/like counters

**User DB (MySQL)** — Users, channels, subscriptions

**Engagement DB (Cassandra)** — Comments, likes — high write throughput

**Feed Cache (Redis)** — Pre-computed feeds per user

**S3 + CDN** — Video storage and global delivery

**Kafka** — Async event bus for all downstream processing

## 4. Database Design

### Cassandra — videos

| Field | Type |
|---|---|
| video_id | UUID (PK) |
| channel_id | UUID |
| title | VARCHAR |
| description | TEXT |
| manifest_url | TEXT |
| thumbnail_url | TEXT |
| duration_sec | INT |
| view_count | COUNTER |
| like_count | COUNTER |
| status | TEXT (processing / public / private / deleted) |
| published_at | TIMESTAMP |

### MySQL — users / channels

| Field | Type |
|---|---|
| user_id | UUID (PK) |
| channel_name | VARCHAR |
| email | VARCHAR, unique |
| password_hash | VARCHAR |
| subscriber_count | INT |
| created_at | TIMESTAMP |

### MySQL — subscriptions

| Field | Type |
|---|---|
| subscriber_id | UUID |
| channel_id | UUID |
| subscribed_at | TIMESTAMP |

### Cassandra — comments

| Field | Type |
|---|---|
| video_id | UUID (partition key) |
| comment_id | TIMEUUID (clustering) |
| user_id | UUID |
| content | TEXT |
| like_count | COUNTER |
| created_at | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `feed:{userId}` | List | ordered videoIds | 3600s |
| `trending:global` | ZSET | videoId → score | 300s |
| `video:meta:{videoId}` | String | metadata JSON | 600s |
| `session:{sessionId}` | String | userId | 86400s |

## 5. Key Flows

### 5.1 Video Upload & Processing
Same pipeline as Netflix — raw upload → S3 → Chunker → Kafka → parallel Encoder instances → encoded segments to S3 → manifest → Video DB update → CDC to Elasticsearch.

Large file handling: resumable uploads via multipart (same as S3 design). Client uploads in 5MB chunks; server reassembles.

### 5.2 Feed Generation
Same hybrid fan-out as TikTok:
- Push for channels with <100K subscribers
- Pull + Recommendation for large channels
- Recommendation Service fills with personalized content

### 5.3 Video Streaming
Identical to Netflix — manifest file → CDN → ABR streaming. See Netflix design.

### 5.4 View Count
View events published to Kafka. Analytics Service aggregates and batch-updates Cassandra COUNTER every 30s. Prevents hot-key writes on viral videos. Slight staleness (30s) acceptable.

### 5.5 Search
CDC from Cassandra Video DB → Kafka → Elasticsearch consumer. Search Service queries Elasticsearch with full-text + filters (duration, upload date, channel). Autocomplete via prefix search.

## 6. Key Interview Concepts

### YouTube vs Netflix Architecture
Both use the same video encoding and CDN delivery pipeline. Key differences:
- YouTube: user-generated content (UGC) — anyone uploads; Netflix: curated professional content
- YouTube: public discovery, search, recommendations are core; Netflix: subscription-gated
- YouTube: view counts, likes, comments are public engagement signals; Netflix: private watch history
- YouTube: monetization (ads) requires view tracking; Netflix: subscription model

### Subscription Fan-out
Channel with 100M subscribers uploads video. Pushing to all 100M feed caches = 100M Redis writes. Same fan-out problem as TikTok. Hybrid approach: push to small subscriber counts, pull for large channels.

### Video Counter Accuracy vs Performance
Viral video gets 1M views/sec. Writing to Cassandra COUNTER 1M times/sec creates a hot partition. Solution: buffer view events in Kafka, batch-aggregate every 30s, write aggregated count to Cassandra. Acceptable staleness for view counts.

### Recommendation at Scale
YouTube's recommendation system is one of the most complex ML systems. At HLD level: two-stage pipeline:
1. Candidate generation: retrieve ~100 candidate videos from user's watch history, subscriptions, trending
2. Ranking: ML model scores and ranks candidates by predicted watch time
3. Result: top 20 videos for feed

## 7. Failure Scenarios

### Encoding Pipeline Failure
Same as Netflix/TikTok — Kafka retains jobs, another encoder picks up, idempotent encoding.

### Feed Cache Miss
Fall back to pull-based feed from subscriptions + trending. Recommendation Service generates on demand.

### View Count Kafka Lag
View counts temporarily stale. Kafka retains events; Analytics Service catches up. No data loss.

### Search Index Stale
CDC replays missed changes on Elasticsearch recovery. Search shows slightly old results briefly.
