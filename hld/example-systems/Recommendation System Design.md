# Recommendation System Design

## System Overview
A recommendation engine that suggests relevant items (products, videos, articles, friends) to users based on their behavior, preferences, and similarity to other users — used across ecommerce, streaming, social media, and content platforms.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Generate personalized item recommendations for each user
- Support multiple recommendation types: collaborative filtering, content-based, trending, similar items
- Real-time updates based on recent user actions (clicks, purchases, watches)
- A/B testing of recommendation algorithms
- Explain recommendations ("Because you watched X")

### Non-Functional Requirements
- Latency: <100ms for recommendation serving
- Scalability: 1B+ users, 500M+ items
- Freshness: Recommendations updated within minutes of user action
- Availability: 99.99% — fall back to trending if personalization unavailable

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 1B users, 500M items
- 100M DAU, each requests recommendations 10 times/day
- User interaction events: 1B/day (clicks, purchases, watches, likes)
- Model retraining: daily batch + real-time updates

### Traffic
```
Recommendation requests/sec = 100M × 10 / 86400 ≈ 11.6K/sec
Interaction events/sec      = 1B / 86400 ≈ 11.6K/sec
```

### Storage
```
User-item interaction matrix = 1B users × 500 interactions avg × 8B = 4TB
Item embeddings              = 500M × 128 dimensions × 4B = 256GB
User embeddings              = 1B × 128 dimensions × 4B = 512GB
Pre-computed recommendations = 1B users × 20 items × 8B = 160GB (Redis)
```

## 3. Core Components

**API Gateway** — Auth, rate limiting, routing

**Recommendation Service** — Serves recommendations; reads from pre-computed cache; falls back to real-time model if cache miss

**Event Ingestion Service** — Receives user interaction events (click, purchase, watch, like, skip); publishes to Kafka

**Feature Store** — Stores user and item features (embeddings, metadata, behavioral signals); serves features to models

**Batch Training Pipeline** — Daily job; trains collaborative filtering and content-based models on full interaction history; generates user/item embeddings; stores to Feature Store

**Real-time Update Pipeline** — Kafka consumer; processes recent interactions; updates user embeddings incrementally; refreshes pre-computed recommendations for active users

**Candidate Generation Service** — Retrieves top-K candidate items for a user using approximate nearest neighbor (ANN) search on embeddings

**Ranking Service** — Re-ranks candidates using a more complex ML model (considers context, diversity, business rules); produces final top-N recommendations

**A/B Testing Service** — Routes users to different recommendation algorithms; tracks metrics per variant

**Pre-computed Cache (Redis)** — Stores top-20 recommendations per user; refreshed periodically

**Feature Store DB (Cassandra)** — User and item embeddings, behavioral features

**Interaction DB (Cassandra)** — Raw interaction events, user history

**Model Store (S3)** — Trained model artifacts

**Kafka** — Interaction event stream, real-time update triggers

## 4. Database Design

### Cassandra — user_interactions

Partition key: `user_id`, Clustering: `event_time DESC`

| Field | Type |
|---|---|
| user_id | UUID (partition key) |
| event_time | TIMESTAMP (clustering) |
| item_id | UUID |
| event_type | TEXT (click / purchase / watch / like / skip) |
| context | TEXT (JSON — device, page, position) |

### Cassandra — item_features

| Field | Type |
|---|---|
| item_id | UUID (PK) |
| category | VARCHAR |
| tags | LIST\<VARCHAR\> |
| embedding | BLOB (128-dim float vector) |
| popularity_score | FLOAT |
| updated_at | TIMESTAMP |

### Cassandra — user_features

| Field | Type |
|---|---|
| user_id | UUID (PK) |
| embedding | BLOB (128-dim float vector) |
| top_categories | LIST\<VARCHAR\> |
| recent_items | LIST\<UUID\> (last 100) |
| updated_at | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `recs:{userId}` | List | ordered itemIds (top 20) | 3600s |
| `trending:global` | ZSET | itemId → score | 300s |
| `trending:{category}` | ZSET | itemId → score | 300s |
| `similar:{itemId}` | List | similar itemIds | 86400s |

## 5. Key Flows

### 5.1 Recommendation Serving

```
Client → Recommendation Service
              ↓
    Check Redis: recs:{userId}  ← <1ms
              ↓ (cache hit)
    Return pre-computed recommendations
              ↓ (cache miss)
    Candidate Generation: ANN search on user embedding
              ↓
    Ranking Service: re-rank candidates
              ↓
    Return top-N; populate cache
```

1. Client requests recommendations for `userId`
2. Recommendation Service checks Redis cache (TTL 1hr)
3. Cache hit: return immediately (<1ms)
4. Cache miss: real-time generation
   - Fetch user embedding from Feature Store
   - Candidate Generation: ANN search finds top-500 similar items
   - Ranking Service: ML model scores and re-ranks to top-20
   - Populate Redis cache
5. Return recommendations with explanation metadata

### 5.2 Two-Stage Recommendation Pipeline

**Stage 1 — Candidate Generation (Recall):**
- Goal: retrieve ~500 relevant candidates from 500M items fast
- Method: Approximate Nearest Neighbor (ANN) search on user embedding
- Tools: FAISS, ScaNN, or Pinecone vector database
- Speed: <10ms for 500M items

**Stage 2 — Ranking (Precision):**
- Goal: rank 500 candidates to top-20 using rich features
- Method: gradient boosted trees or neural ranking model
- Features: user-item affinity, recency, diversity, business rules (promoted items, inventory)
- Speed: <50ms for 500 candidates

This two-stage approach is used by YouTube, Netflix, Amazon, TikTok.

### 5.3 Collaborative Filtering

"Users who liked X also liked Y."

**Matrix Factorization:**
- User-item interaction matrix: rows = users, columns = items, values = interaction strength
- Factorize into user embeddings (U) and item embeddings (V)
- Predicted affinity = U[user] · V[item] (dot product)
- Users with similar embeddings have similar tastes

**Training:** daily batch job on full interaction history (Spark / distributed training)
**Inference:** ANN search on pre-computed embeddings

### 5.4 Content-Based Filtering

"You liked X, here are items similar to X."

- Item features: category, tags, description embeddings (BERT/sentence transformers)
- User profile: weighted average of features of items user interacted with
- Recommend items with high cosine similarity to user profile

### 5.5 Real-Time Updates

1. User clicks item → Event Ingestion Service → Kafka
2. Real-time Update Pipeline consumes event
3. Updates user embedding incrementally (online learning or re-fetch recent history)
4. Invalidates Redis cache for that user: `DEL recs:{userId}`
5. Next recommendation request triggers fresh generation with updated embedding

### 5.6 Trending Recommendations

For new users (cold start) or cache miss fallback:
- Trending Service aggregates interaction events per item in rolling 1hr window
- Updates Redis ZSET: `ZINCRBY trending:global {score} {itemId}`
- Score = weighted sum of clicks + purchases + shares in last hour
- Recommendation Service falls back to trending if no personalized recs available

## 6. Key Interview Concepts

### Cold Start Problem
New user has no interaction history → no personalized recommendations.
Solutions:
1. Onboarding: ask user to select interests/categories
2. Demographic-based: recommend popular items in user's age/location group
3. Trending: show globally popular items
4. Gradually personalize as user interacts

New item has no interactions → not recommended by collaborative filtering.
Solutions:
1. Content-based: use item features (category, description) to find similar items
2. Exploration: occasionally show new items to gather interaction data

### ANN (Approximate Nearest Neighbor) Search
Finding exact nearest neighbors in 128-dim space across 500M items is too slow (O(N)). ANN algorithms (HNSW, IVF) trade small accuracy loss for massive speed gain — O(log N) or O(1) with indexing. FAISS (Facebook) and ScaNN (Google) are standard tools.

### Diversity vs Relevance
Pure relevance optimization leads to filter bubbles — user only sees items similar to what they've seen. Solution: diversity constraints in ranking (e.g., max 3 items from same category in top-20). Serendipity: occasionally recommend slightly outside user's comfort zone.

### Implicit vs Explicit Feedback
- Explicit: ratings, likes, reviews — clear signal but rare
- Implicit: clicks, watch time, purchases, skips — abundant but noisy (click ≠ like)
- Watch time is a stronger signal than click (user actually consumed the content)
- Skip is a negative signal

### Exploration vs Exploitation (Bandit Problem)
Exploitation: recommend items you know the user likes (safe, less discovery).
Exploration: occasionally recommend new items to learn user preferences (risky, potentially more valuable).
Solution: epsilon-greedy or Thompson Sampling — explore with probability ε, exploit otherwise.

## 7. Failure Scenarios

### Feature Store Failure
- Impact: real-time recommendation generation fails; cache misses return trending
- Recovery: Recommendation Service falls back to trending/popular items; Feature Store is stateless, restarts quickly
- Prevention: Redis cache absorbs most traffic; Feature Store only hit on cache miss

### Model Training Failure
- Impact: recommendations become stale (yesterday's model)
- Recovery: keep last N model versions; roll back to previous version; alert ML team
- Prevention: model validation before deployment; canary rollout of new models

### Cache Miss Storm (Redis Failure)
- Impact: all requests hit Candidate Generation + Ranking — high latency
- Recovery: Redis Sentinel failover; cache rebuilds as requests come in
- Prevention: Redis Cluster; pre-warm cache for top 10M active users after Redis recovery

### Recommendation Bias / Feedback Loop
- Scenario: model recommends popular items → users click → items become more popular → model recommends even more → filter bubble
- Recovery: diversity constraints; periodic re-exploration; inject random items
- Prevention: monitor recommendation diversity metrics; A/B test diversity vs engagement trade-off
