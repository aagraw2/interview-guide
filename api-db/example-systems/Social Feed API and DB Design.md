# Social Feed API and DB Design

## Problem Statement

Design the API and database schema for a social media feed (like Twitter/X or Instagram). Users can post content, follow other users, and see a feed of posts from people they follow.

**Core requirements:**
- Create and delete posts
- Follow/unfollow users
- View a personalized feed (posts from followed users, reverse chronological)
- Like posts
- Optional: comments, retweets/reposts, notifications

**Scale:** 500M users, 100M posts/day, 1B feed reads/day

---

## API Design

### Endpoints

#### Create Post
```
POST /api/v1/posts
Authorization: Bearer <token>

Request:
{
  "content": "Hello world!",
  "mediaUrls": ["https://cdn.example.com/img/abc.jpg"]  // optional
}

Response 201:
{
  "id": "post_abc123",
  "content": "Hello world!",
  "authorId": "user_xyz",
  "likeCount": 0,
  "createdAt": "2024-01-15T10:00:00Z"
}
```

#### Get Feed
```
GET /api/v1/feed?cursor=<cursor>&limit=20
Authorization: Bearer <token>

Response 200:
{
  "posts": [
    {
      "id": "post_abc123",
      "content": "...",
      "author": { "id": "user_xyz", "username": "alice" },
      "likeCount": 42,
      "isLikedByMe": true,
      "createdAt": "2024-01-15T10:00:00Z"
    }
  ],
  "nextCursor": "eyJpZCI6MTAwfQ"
}
```

#### Follow / Unfollow
```
POST /api/v1/users/{userId}/follow
DELETE /api/v1/users/{userId}/follow
Authorization: Bearer <token>

Response 204
```

#### Like / Unlike Post
```
POST /api/v1/posts/{postId}/like
DELETE /api/v1/posts/{postId}/like
Authorization: Bearer <token>

Response 204
```

#### Get User Profile + Posts
```
GET /api/v1/users/{userId}
GET /api/v1/users/{userId}/posts?cursor=<cursor>&limit=20
```

---

## Database Schema

### users table
```sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    username    VARCHAR(50) NOT NULL UNIQUE,
    email       VARCHAR(255) NOT NULL UNIQUE,
    bio         TEXT,
    follower_count  INT NOT NULL DEFAULT 0,
    following_count INT NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### posts table
```sql
CREATE TABLE posts (
    id          BIGSERIAL PRIMARY KEY,
    author_id   BIGINT NOT NULL REFERENCES users(id),
    content     TEXT NOT NULL,
    like_count  INT NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at  TIMESTAMPTZ  -- soft delete
);

CREATE INDEX idx_posts_author_id_created_at ON posts (author_id, created_at DESC);
CREATE INDEX idx_posts_created_at ON posts (created_at DESC);
```

### follows table
```sql
CREATE TABLE follows (
    follower_id BIGINT NOT NULL REFERENCES users(id),
    followee_id BIGINT NOT NULL REFERENCES users(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id)
);

CREATE INDEX idx_follows_followee_id ON follows (followee_id);
```

### likes table
```sql
CREATE TABLE likes (
    user_id     BIGINT NOT NULL REFERENCES users(id),
    post_id     BIGINT NOT NULL REFERENCES posts(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, post_id)
);

CREATE INDEX idx_likes_post_id ON likes (post_id);
```

### feed_cache table (materialized feed — fan-out on write)
```sql
CREATE TABLE feed_cache (
    user_id     BIGINT NOT NULL REFERENCES users(id),
    post_id     BIGINT NOT NULL REFERENCES posts(id),
    created_at  TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (user_id, post_id)
);

CREATE INDEX idx_feed_cache_user_id_created_at ON feed_cache (user_id, created_at DESC);
```

---

## Trade-offs and Considerations

### Feed Generation: Fan-out on Write vs Fan-out on Read

**Fan-out on Write (Push model)**
- When a user posts, immediately write to all followers' feed caches
- Feed reads are fast (pre-computed)
- Problem: Celebrity users with millions of followers cause massive write amplification
- Solution: Hybrid — fan-out on write for regular users, fan-out on read for celebrities

**Fan-out on Read (Pull model)**
- Feed is computed at read time by querying posts from all followed users
- No write amplification
- Problem: Slow for users following many people (N+1 queries or expensive JOIN)

**Recommendation**: Hybrid approach — fan-out on write for users with < 10K followers, fan-out on read for celebrities.

### Denormalization
- `like_count` on posts table is denormalized — updated via counter increment
- `follower_count` / `following_count` on users table are denormalized
- Trade-off: faster reads, risk of count drift — use atomic increments and periodic reconciliation

### Cursor-Based Pagination
- Feed uses cursor-based pagination (not offset) to handle concurrent new posts
- Cursor encodes `(created_at, post_id)` for stable ordering

### Soft Deletes
- Posts use `deleted_at` for soft delete — preserves like/comment counts for analytics
- Filter `WHERE deleted_at IS NULL` in all queries
