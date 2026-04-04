# Social Media System Design

## System Overview
A social media platform (think Twitter / Instagram / Facebook) where users post content, follow others, receive a personalized feed, interact via likes/comments/shares, and get real-time notifications.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- User registration, authentication, profile management
- Create posts (text, images, videos)
- Follow/unfollow users
- Personalized home feed (posts from followed users + recommendations)
- Like, comment, share posts
- Search users and posts
- Real-time notifications (likes, comments, follows, mentions)
- Direct messaging (basic)
- Trending topics/hashtags

### Non-Functional Requirements
- Availability: 99.99%
- Latency: <200ms feed load; <500ms post creation
- Scalability: 1B+ users, 500M DAU, 500M posts/day
- Read >> Write: feed reads vastly outnumber post creations
- Eventual consistency acceptable for feed and counts

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 1B users, 500M DAU
- 500M posts/day (text + media)
- Each user follows 200 people on average
- Each user reads feed 10 times/day
- 20% posts contain media (avg 2MB)

### Traffic
```
Posts/sec           = 500M / 86400 ≈ 5.8K/sec
Feed reads/sec      = 500M × 10 / 86400 ≈ 58K/sec
Likes/sec           = 500M × 20 likes/day / 86400 ≈ 116K/sec

Media uploads/sec   = 5.8K × 0.2 = 1.16K/sec
```

### Storage
```
Posts/day           = 500M × 200B text = 100GB/day → ~36TB/year
Media/day           = 500M × 0.2 × 2MB = 200TB/day → S3
User profiles       = 1B × 1KB = 1TB
Follow graph        = 1B users × 200 follows × 16B = 3.2TB
```

### Memory (Cache)
```
Hot feed cache      = 500M DAU × 20 posts × 200B = 2TB (top 10M users = 40GB)
Trending topics     = small, easily cached
```

## 3. Core Components

**API Gateway** — Auth, rate limiting, routing

**User Service** — Registration, login, profile management; writes to User DB

**Content Service** — Post creation and deletion; validates content; publishes to Kafka `raw_post` topic; does not write to DB directly (async pipeline handles that)

**Moderator Service** — Kafka consumer on `raw_post`; runs content moderation (spam detection, hate speech, NSFW classifiers); publishes to `filtered_post` (approved) or `blocked_post` (rejected); keeps post creation fast by doing moderation async

**Post Consumer Service** — Kafka consumer on `filtered_post`; writes approved posts to Post DB and media to S3; triggers search indexing and notification for mentions

**Feed Service** — Serves personalized home feed; reads from Feed Cache (Redis) first, falls back to Feed DB, then pull-based reconstruction

**Fan-out Service** — Kafka consumer; fetches follower list from Follower Cache (Redis) → Follower DB; pushes postId to followers' Feed Cache and Feed DB (hybrid push/pull)

**Follower Service** — Manages follow/unfollow relationships; writes to Follower DB; invalidates Follower Cache on change

**Engagement Service** — Likes (with reaction types), comments, shares; writes to Engagement DB; Kafka events for counter updates and notifications

**Search Service** — User and post search; Elasticsearch; CDC from Post DB

**Notification Service** — Kafka consumer; real-time push notifications via WebSocket + FCM/APNs

**Trending Service** — Computes trending hashtags from engagement events; Redis ZSET

**Media Service** — Handles image/video upload; stores to S3; returns CDN URLs

**Post DB (Cassandra)** — Approved post records, partitioned by userId

**User DB (MySQL)** — User profiles, credentials

**Follower DB (PostgreSQL)** — Follow relationships; separate from User DB due to different access patterns (graph traversal, high write volume)

**Engagement DB (Cassandra)** — Likes (with reaction_type), comments — high write throughput

**Feed Cache (Redis)** — Pre-computed feeds per user (top 200 postIds); primary read path

**Feed DB (Cassandra)** — Durable backing store for feeds; survives Redis eviction; Feed Cache rebuilt from here on miss

**Follower Cache (Redis)** — Cached top follower lists per user; used by Fan-out Service to avoid hitting Follower DB on every post

**S3 + CDN** — Media storage and delivery

**Kafka** — Three post topics: `raw_post`, `filtered_post`, `blocked_post`; plus engagement events, notification fan-out

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| Cassandra (Post DB) | High write throughput, partition by userId, time-series ordering |
| MySQL (User DB) | Structured user data, ACID |
| PostgreSQL (Follower DB) | Follow graph — relational queries, ACID, separate from user data due to scale |
| Cassandra (Engagement DB) | Billions of likes/comments, append-only, high throughput |
| Cassandra (Feed DB) | Durable feed storage, partition by userId, survives Redis eviction |
| Redis | Feed Cache, Follower Cache, trending topics, sessions |
| Elasticsearch | Full-text search on posts and users |
| S3 + CDN | Media storage, global delivery |

### Cassandra — posts

| Field | Type |
|---|---|
| user_id | UUID (partition key) |
| post_id | TIMEUUID (clustering) |
| post_type | VARCHAR (text / image / video) |
| content_text | TEXT |
| media_url | TEXT |
| thumbnail_url | TEXT |
| like_count | COUNTER |
| share_count | COUNTER |
| comment_count | COUNTER |
| metadata | JSONB |

### MySQL — users

| Field | Type |
|---|---|
| user_id | UUID (PK) |
| username | VARCHAR, unique |
| email | VARCHAR, unique |
| password_hash | VARCHAR |
| bio | TEXT |
| profile_pic_url | TEXT |
| follower_count | INT |
| following_count | INT |
| created_at | TIMESTAMP |

### PostgreSQL — followers

| Field | Type |
|---|---|
| follow_id | UUID (PK) |
| follower_id | UUID |
| following_id | UUID |
| status | ENUM (active / blocked) |
| timestamp | TIMESTAMP |
| metadata | JSONB |

### Cassandra — engagement (likes)

| Field | Type |
|---|---|
| post_id | UUID (partition key) |
| like_id | TIMEUUID (clustering) |
| user_id | UUID |
| reaction_type | VARCHAR (like / love / haha / angry / sad) |
| metadata | JSONB |

### Cassandra — engagement (comments)

| Field | Type |
|---|---|
| post_id | UUID (partition key) |
| comment_id | TIMEUUID (clustering) |
| user_id | UUID |
| content | TEXT |
| like_count | COUNTER |
| metadata | JSONB |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `feed:{userId}` | List | ordered postIds (top 200) | 3600s |
| `followers:{userId}` | List | top follower userIds | 300s |
| `trending:hashtags` | ZSET | hashtag → score | 300s |
| `post:meta:{postId}` | String | post JSON | 600s |
| `session:{sessionId}` | String | userId | 86400s |

## 5. Key Flows

### 5.1 Auth

1. Register → User Service → write to MySQL → return JWT
2. Login → validate → JWT (1hr) + refresh token → session in Redis
3. API Gateway validates JWT on every request

### 5.2 Post Creation & Moderation

```
Client → Content Service → Kafka: raw_post
                                      ↓
                              Moderator Service
                              (spam / NSFW / hate speech check)
                                      ↓
                    filtered_post          blocked_post
                          ↓                     ↓
                Post Consumer Svc         notify user (rejected)
                (write PostDB + S3)
                          ↓
                  Kafka: post_created → Fan-out, Search, Notification
```

1. User creates post → Content Service validates format, publishes to `raw_post`
2. Returns postId immediately (async — post not yet live)
3. Moderator Service consumes `raw_post`: runs ML classifiers (spam, NSFW, hate speech)
4. Approved → publish to `filtered_post`; rejected → publish to `blocked_post`, notify user
5. Post Consumer Service consumes `filtered_post`: writes to Post DB (Cassandra), media to S3
6. Publishes `post_created` event → Fan-out Service, Search Service, Notification Service (for mentions)

### 5.3 Feed Generation — The Core Problem

**Push model (fan-out on write):**
- When user posts, push postId to all followers' feed caches in Redis
- Fast reads (pre-computed), expensive writes
- Problem: celebrity with 100M followers → 100M Redis writes per post

**Pull model (fan-out on read):**
- On feed request, fetch recent posts from all followed users
- Cheap writes, expensive reads
- Problem: user follows 1000 people → 1000 DB queries per feed load

**Hybrid model (used by Twitter/Instagram):**
- Push for regular users (< 1M followers): fan-out to followers' caches
- Pull for celebrities (> 1M followers): fetch on demand at read time
- Merge both at read time

**Flow (hybrid):**
1. User requests feed → Feed Service
2. Check Redis `feed:{userId}` (pre-computed from push)
3. Fetch recent posts from followed celebrities (pull, last 20 posts each)
4. Merge + rank by recency/engagement
5. Return top 20 posts
6. Populate/update Redis cache

### 5.4 Fan-out Service

1. Consumes `post_created` from Kafka
2. Fetches follower list from **Follower Cache** (Redis `followers:{userId}`) — avoids hitting Follower DB on every post
3. Cache miss: query Follower DB, populate cache (TTL 5min)
4. For each follower (if poster has < 1M followers — push model):
   - `LPUSH feed:{followerId} {postId}` in Redis Feed Cache
   - `LTRIM feed:{followerId} 0 199` (keep top 200)
   - Write to Feed DB (Cassandra) for durability
5. Skip fan-out for celebrities (> 1M followers) — their followers pull on read
6. Batch Redis writes using pipeline; process follower batches in parallel

**Why Feed DB alongside Feed Cache:**
Redis TTL evicts feeds for inactive users. Feed DB (Cassandra) is the durable backing store — Feed Cache rebuilt from Feed DB on miss, not from Post DB. Faster recovery, less load on Post DB.

### 5.5 Engagement (Likes, Comments)

1. User likes post → Engagement Service
2. Write to Cassandra `likes` table
3. Publish `LIKE` event to Kafka
4. Kafka consumer: `INCR` like_count in Cassandra (COUNTER type) — batched every 30s
5. Notification Service: push notification to post author

### 5.6 Trending Topics

1. Engagement Service publishes hashtag events to Kafka
2. Trending Service consumes: `ZINCRBY trending:hashtags {score} {hashtag}` in Redis
3. Score = weighted sum of posts + likes + shares in rolling 1hr window
4. `ZREVRANGE trending:hashtags 0 9` returns top 10 trending
5. Refreshed every 5 min

### 5.7 Search

1. CDC from Cassandra Post DB → Kafka → Elasticsearch consumer
2. User searches "climate change" → Search Service → Elasticsearch
3. Full-text on post content + hashtags; user search on username/bio
4. Returns ranked results

## 6. Key Interview Concepts

### Fan-out Problem
The most important concept for social media. Celebrity with 100M followers posts → 100M feed cache writes. Solutions:
- Hybrid: push for regular users, pull for celebrities
- Threshold: typically 1M followers = celebrity
- At read time: merge pre-computed feed (push) + celebrity posts (pull)

### Feed Ranking
Chronological feed is simple but not engaging. Ranked feed considers:
- Recency (newer posts score higher)
- Engagement (posts with many likes/comments score higher)
- Relationship strength (close friends score higher)
- Content type preference (user watches more videos → videos ranked higher)

### Follow Graph Storage
Follower relationships are stored in a **separate PostgreSQL DB** (not in User DB). Reasons:
- The follow graph is a completely different access pattern — high write volume, graph traversal queries
- At 1B users × 200 follows = 200B edges, it dwarfs user profile data
- Separating it lets both scale independently

Two query patterns both need indexes:
- "Who does user A follow?" → `SELECT following_id WHERE follower_id = A`
- "Who follows user A?" → `SELECT follower_id WHERE following_id = A`

**Follower Cache** in Redis caches top follower lists per user (TTL 5min). Fan-out Service reads from cache first — avoids a DB query on every post from a popular user.

### Counter Accuracy
500M likes/day = 5.8K likes/sec. Writing to Cassandra COUNTER on every like creates hot partitions for viral posts. Solution: buffer in Redis (`INCR post:likes:{postId}`), batch-flush to Cassandra every 30s. Slight staleness acceptable.

### Content Moderation Pipeline
At scale, you can't moderate synchronously — it would add 500ms+ to every post. Async pipeline via Kafka:
- Post creation returns immediately with `postId`
- Moderator Service runs ML classifiers in the background
- Approved posts go live within seconds; rejected posts never reach Post DB
- Three Kafka topics keep the flows clean: `raw_post` → `filtered_post` / `blocked_post`
- Moderator Service scales independently based on post volume

### Reaction Types
Modern platforms have reaction types beyond just "like" (love, haha, angry, sad). Stored as `reaction_type` in the Engagement DB. Enables richer analytics ("this post got 10K angry reactions") and different notification copy ("X reacted 😂 to your post").

### CAP Trade-off
Social media favors Availability + Partition tolerance (AP). A user seeing a post 2 seconds late is fine. A user unable to post is not. Cassandra with eventual consistency is the right choice for posts and engagement.

## 7. Failure Scenarios

### Fan-out Service Crash
- Detection: Kafka consumer lag grows; feeds not updated
- Recovery: Kafka retains events; Fan-out Service restarts and catches up; users see slightly stale feeds
- Prevention: multiple Fan-out Service instances in consumer group

### Feed Cache Miss (Redis Failure)
- Impact: all feed requests hit pull path (slower but functional)
- Recovery: Redis Sentinel failover; cache rebuilds as users request feeds
- Prevention: Redis Cluster; feed is reconstructable from Post DB

### Cassandra Hot Partition (Viral Post)
- Scenario: post with 10M likes/sec creates hot partition
- Recovery: Redis counter absorbs writes; batch-flush to Cassandra
- Prevention: COUNTER type with batched updates; shard hot posts across multiple partitions

### Celebrity Posts Not in Feed
- Scenario: user follows celebrity but pull path fails
- Recovery: graceful degradation — show pre-computed push feed without celebrity posts; retry pull
- Prevention: celebrity post pull is a separate service with its own cache and fallback
