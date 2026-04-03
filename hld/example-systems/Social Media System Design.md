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

**User Service** — Registration, login, profile management, follow graph

**Post Service** — Post creation, deletion, media upload; writes to Post DB; publishes to Kafka

**Feed Service** — Generates and serves personalized home feed; reads from Feed Cache or builds on demand

**Fan-out Service** — Kafka consumer; distributes new posts to followers' feed caches (push model)

**Timeline Service** — Serves user's own posts (profile page); reads from Post DB

**Engagement Service** — Likes, comments, shares, retweets; writes to Engagement DB; Kafka events for counters

**Search Service** — User and post search; Elasticsearch; CDC from Post DB

**Notification Service** — Kafka consumer; real-time push notifications via WebSocket + FCM/APNs

**Trending Service** — Computes trending hashtags and topics from engagement events; Redis ZSET

**Media Service** — Handles image/video upload; stores to S3; triggers encoding for videos; returns CDN URLs

**Post DB (Cassandra)** — Posts, partitioned by userId; high read/write throughput

**User DB (MySQL)** — User profiles, follow relationships

**Engagement DB (Cassandra)** — Likes, comments — high write throughput, time-series

**Feed Cache (Redis)** — Pre-computed feeds per user (top 200 posts)

**S3 + CDN** — Media storage and delivery

**Kafka** — Post events, engagement events, notification fan-out

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| Cassandra (Post DB) | High write throughput, partition by userId, time-series ordering |
| MySQL (User DB) | Structured user data, follow graph, ACID |
| Cassandra (Engagement) | Billions of likes/comments, append-only |
| Redis | Feed cache, trending topics, sessions |
| Elasticsearch | Full-text search on posts and users |
| S3 + CDN | Media storage, global delivery |

### Cassandra — posts

Partition key: `user_id`, Clustering: `post_id DESC` (TIMEUUID)

| Field | Type |
|---|---|
| user_id | UUID (partition key) |
| post_id | TIMEUUID (clustering) |
| content | TEXT |
| media_urls | LIST\<TEXT\> |
| like_count | COUNTER |
| comment_count | COUNTER |
| share_count | COUNTER |
| hashtags | LIST\<VARCHAR\> |
| created_at | TIMESTAMP |

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

### MySQL — follows

| Field | Type |
|---|---|
| follower_id | UUID |
| followee_id | UUID |
| created_at | TIMESTAMP |
| PRIMARY KEY | (follower_id, followee_id) |

### Cassandra — comments

| Field | Type |
|---|---|
| post_id | UUID (partition key) |
| comment_id | TIMEUUID (clustering) |
| user_id | UUID |
| content | TEXT |
| created_at | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `feed:{userId}` | List | ordered postIds (top 200) | 3600s |
| `trending:hashtags` | ZSET | hashtag → score | 300s |
| `post:meta:{postId}` | String | post JSON | 600s |
| `session:{sessionId}` | String | userId | 86400s |

## 5. Key Flows

### 5.1 Auth

1. Register → User Service → write to MySQL → return JWT
2. Login → validate → JWT (1hr) + refresh token → session in Redis
3. API Gateway validates JWT on every request

### 5.2 Post Creation

1. User creates post → Post Service
2. Validate content (length, media count)
3. If media: upload to S3 via Media Service, get CDN URLs
4. Write post to Cassandra (partition by userId)
5. Publish `POST_CREATED` event to Kafka
6. Return postId to client

**Kafka consumers:**
- Fan-out Service: push post to followers' feed caches
- Search Service: index post in Elasticsearch
- Trending Service: update hashtag counts
- Notification Service: notify mentioned users

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

1. Consumes `POST_CREATED` from Kafka
2. Fetches follower list from MySQL (paginated for large follower counts)
3. For each follower (if < 1M followers threshold):
   - `LPUSH feed:{followerId} {postId}`
   - `LTRIM feed:{followerId} 0 199` (keep top 200)
4. Skip celebrities' followers (they pull on read)

**Optimization:** batch Redis writes using pipeline; process followers in parallel workers

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
MySQL `follows` table with composite PK `(follower_id, followee_id)`. Two query patterns:
- "Who does user A follow?" → `SELECT followee_id WHERE follower_id = A`
- "Who follows user A?" → `SELECT follower_id WHERE followee_id = A`
Both need indexes. For very large graphs (1B+ edges), consider graph DB (Neo4j) or denormalized Cassandra tables.

### Counter Accuracy
500M likes/day = 5.8K likes/sec. Writing to Cassandra COUNTER on every like creates hot partitions for viral posts. Solution: buffer in Redis (`INCR post:likes:{postId}`), batch-flush to Cassandra every 30s. Slight staleness acceptable.

### Soft Delete
Posts are never hard-deleted immediately. Mark as `deleted = true`, hide from feeds. Async job purges after 30 days. Allows recovery from accidental deletion and compliance with legal holds.

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
