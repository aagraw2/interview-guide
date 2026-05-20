# Blog Platform API and DB Design

## System Overview
A multi-author blogging platform (think Medium or Dev.to) where writers publish posts, readers discover content through tags and full-text search, and the community engages through comments and reactions. Posts move through a draft/published lifecycle with slug-based URLs for SEO.

## 1. Requirements

### Functional Requirements
- Authors can create, edit, and publish posts with rich text content
- Posts have slug-based URLs (e.g., `/posts/my-first-post-abc123`)
- Tag system with many-to-many relationship between posts and tags
- Readers can comment on published posts (threaded replies)
- Like/reaction system for posts and comments
- Full-text search across post titles and content
- Author profiles with follower/following relationships

### Non-Functional Requirements
- Availability: 99.9%
- Read-heavy: 100:1 read-to-write ratio
- Latency: <100ms for post reads (cached); <500ms for search
- SEO: Stable slug-based URLs; canonical URLs for duplicate content
- Scalability: 10M posts, 100M readers, 1B page views/month

## 2. Scale Estimation

```
DAU                   = 5M readers, 100K authors
Posts published/day   = 50K
Page views/day        = 33M (1B/month)
Reads/sec             = 33M / 86400 ≈ 380/sec
Peak reads/sec        ≈ 2,000/sec

Post avg size         = 10KB (text + metadata)
Total post storage    = 10M × 10KB = 100GB
Comment storage       = 100M × 500B = 50GB
```

## 3. API Design

### Key Endpoints

#### Create / Update Post
```
POST /api/v1/posts
Authorization: Bearer <token>

Request:
{
  "title": "Understanding PostgreSQL Indexes",
  "content": "## Introduction\n\nIndexes are...",
  "content_format": "markdown",
  "tags": ["postgresql", "databases", "performance"],
  "status": "DRAFT",
  "canonical_url": null
}

Response 201:
{
  "id": "post_abc",
  "slug": "understanding-postgresql-indexes-abc",
  "title": "Understanding PostgreSQL Indexes",
  "status": "DRAFT",
  "author": { "id": "user_xyz", "username": "dbexpert" },
  "tags": ["postgresql", "databases", "performance"],
  "created_at": "2025-06-01T10:00:00Z"
}
```

#### Publish Post
```
PATCH /api/v1/posts/{postId}
Authorization: Bearer <token>

Request: { "status": "PUBLISHED" }

Response 200:
{
  "id": "post_abc",
  "slug": "understanding-postgresql-indexes-abc",
  "status": "PUBLISHED",
  "published_at": "2025-06-01T12:00:00Z",
  "url": "https://blog.example.com/posts/understanding-postgresql-indexes-abc"
}
```

#### Get Post by Slug
```
GET /api/v1/posts/{slug}

Response 200:
{
  "id": "post_abc",
  "title": "Understanding PostgreSQL Indexes",
  "content": "...",
  "author": { "id": "user_xyz", "username": "dbexpert", "avatar_url": "..." },
  "tags": ["postgresql", "databases"],
  "like_count": 342,
  "comment_count": 28,
  "read_time_minutes": 8,
  "published_at": "2025-06-01T12:00:00Z"
}
```

#### Search Posts
```
GET /api/v1/posts/search?q=postgresql+indexes&tags=databases&page=1

Response 200:
{
  "posts": [
    {
      "id": "post_abc",
      "title": "Understanding PostgreSQL Indexes",
      "excerpt": "...Indexes are critical for query performance...",
      "author": { "username": "dbexpert" },
      "tags": ["postgresql"],
      "published_at": "2025-06-01T12:00:00Z"
    }
  ],
  "total": 47
}
```

#### Add Comment
```
POST /api/v1/posts/{postId}/comments
Authorization: Bearer <token>

Request:
{
  "content": "Great explanation of B-tree indexes!",
  "parent_comment_id": null   // null = top-level, set for replies
}

Response 201:
{
  "id": "cmt_xyz",
  "content": "Great explanation of B-tree indexes!",
  "author": { "username": "reader123" },
  "parent_comment_id": null,
  "like_count": 0,
  "created_at": "2025-06-01T13:00:00Z"
}
```

### Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| POST | /api/v1/posts | Create post (draft) |
| GET | /api/v1/posts | List published posts (feed) |
| GET | /api/v1/posts/{slug} | Get post by slug |
| PATCH | /api/v1/posts/{postId} | Update post or publish |
| DELETE | /api/v1/posts/{postId} | Delete post |
| GET | /api/v1/posts/search | Full-text search |
| POST | /api/v1/posts/{postId}/comments | Add comment |
| GET | /api/v1/posts/{postId}/comments | List comments |
| POST | /api/v1/posts/{postId}/like | Like a post |
| DELETE | /api/v1/posts/{postId}/like | Unlike a post |
| GET | /api/v1/tags/{tag}/posts | Posts by tag |

## 4. Database Schema

```sql
CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    username        VARCHAR(50) NOT NULL UNIQUE,
    email           VARCHAR(255) NOT NULL UNIQUE,
    bio             TEXT,
    avatar_url      VARCHAR(500),
    follower_count  INT NOT NULL DEFAULT 0,
    following_count INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE posts (
    id              BIGSERIAL PRIMARY KEY,
    author_id       BIGINT NOT NULL REFERENCES users(id),
    title           VARCHAR(500) NOT NULL,
    slug            VARCHAR(550) NOT NULL UNIQUE,
    content         TEXT NOT NULL,
    content_format  VARCHAR(20) NOT NULL DEFAULT 'markdown',  -- markdown, html
    excerpt         TEXT,                -- auto-generated or manual
    status          VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
                    -- DRAFT, PUBLISHED, ARCHIVED, UNLISTED
    canonical_url   VARCHAR(500),        -- for cross-posted content
    read_time_minutes INT,
    like_count      INT NOT NULL DEFAULT 0,
    comment_count   INT NOT NULL DEFAULT 0,
    view_count      BIGINT NOT NULL DEFAULT 0,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_posts_author_id ON posts (author_id);
CREATE INDEX idx_posts_status_published_at ON posts (status, published_at DESC)
    WHERE status = 'PUBLISHED';
CREATE INDEX idx_posts_slug ON posts (slug);
-- Full-text search index on title + content
CREATE INDEX idx_posts_fts ON posts
    USING GIN (to_tsvector('english', title || ' ' || COALESCE(excerpt, '')));

CREATE TABLE tags (
    id      BIGSERIAL PRIMARY KEY,
    name    VARCHAR(100) NOT NULL UNIQUE,
    slug    VARCHAR(110) NOT NULL UNIQUE,
    post_count INT NOT NULL DEFAULT 0
);

CREATE INDEX idx_tags_name ON tags (name);

-- Many-to-many: posts ↔ tags
CREATE TABLE post_tags (
    post_id BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    tag_id  BIGINT NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);

CREATE INDEX idx_post_tags_tag_id ON post_tags (tag_id);

-- Threaded comments (one level of nesting via parent_comment_id)
CREATE TABLE comments (
    id                  BIGSERIAL PRIMARY KEY,
    post_id             BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    author_id           BIGINT NOT NULL REFERENCES users(id),
    parent_comment_id   BIGINT REFERENCES comments(id),
    content             TEXT NOT NULL,
    like_count          INT NOT NULL DEFAULT 0,
    is_deleted          BOOLEAN NOT NULL DEFAULT FALSE,  -- soft delete
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_comments_post_id ON comments (post_id, created_at);
CREATE INDEX idx_comments_parent_id ON comments (parent_comment_id);

-- Post likes (one per user per post)
CREATE TABLE post_likes (
    user_id     BIGINT NOT NULL REFERENCES users(id),
    post_id     BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, post_id)
);

CREATE INDEX idx_post_likes_post_id ON post_likes (post_id);

-- Author follows
CREATE TABLE follows (
    follower_id BIGINT NOT NULL REFERENCES users(id),
    followee_id BIGINT NOT NULL REFERENCES users(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id),
    CHECK (follower_id != followee_id)
);

CREATE INDEX idx_follows_followee_id ON follows (followee_id);

-- Post revision history
CREATE TABLE post_revisions (
    id          BIGSERIAL PRIMARY KEY,
    post_id     BIGINT NOT NULL REFERENCES posts(id),
    title       VARCHAR(500) NOT NULL,
    content     TEXT NOT NULL,
    revised_by  BIGINT NOT NULL REFERENCES users(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_post_revisions_post_id ON post_revisions (post_id, created_at DESC);
```

### Schema Summary

| Table | Purpose |
|-------|---------|
| posts | Post content with draft/published lifecycle |
| tags | Global tag registry |
| post_tags | Many-to-many post-tag association |
| comments | Threaded comments with soft delete |
| post_likes | One-like-per-user constraint |
| follows | Author follow relationships |
| post_revisions | Edit history for posts |

## 5. Key Design Decisions

### 5.1 Slug Generation and Uniqueness

Slugs are generated from the title at publish time: `"My Post Title"` → `"my-post-title"`. Collisions are resolved by appending a short random suffix: `"my-post-title-a3f2"`. The slug is immutable after publishing to preserve SEO and inbound links.

```sql
-- Slug uniqueness enforced at DB level
slug VARCHAR(550) NOT NULL UNIQUE
```

Drafts can have a temporary slug; it's finalized on first publish.

### 5.2 Full-Text Search with PostgreSQL GIN

PostgreSQL's built-in full-text search handles moderate scale without Elasticsearch:

```sql
SELECT id, title, excerpt,
       ts_rank(to_tsvector('english', title || ' ' || COALESCE(excerpt, '')),
               plainto_tsquery('english', $1)) AS rank
FROM posts
WHERE status = 'PUBLISHED'
  AND to_tsvector('english', title || ' ' || COALESCE(excerpt, ''))
      @@ plainto_tsquery('english', $1)
ORDER BY rank DESC
LIMIT 20;
```

For larger scale, sync to Elasticsearch via CDC. The GIN index covers the excerpt (not full content) to keep index size manageable.

### 5.3 Denormalized Counters

`like_count`, `comment_count`, and `view_count` on the `posts` table are denormalized for fast reads. Updated atomically:
```sql
UPDATE posts SET like_count = like_count + 1 WHERE id = $1;
```
Periodic reconciliation jobs correct any drift. `view_count` is updated asynchronously via a queue to avoid write contention on popular posts.

### 5.4 Draft/Published State Machine

```
DRAFT → PUBLISHED → ARCHIVED
DRAFT → UNLISTED (published but not in feeds)
PUBLISHED → DRAFT (unpublish)
```

Only `PUBLISHED` posts appear in feeds and search. `UNLISTED` posts are accessible by direct URL but excluded from discovery. `published_at` is set once and never changed (even if unpublished and republished) to preserve the original publication date.

### 5.5 Tag Counter Maintenance

`tags.post_count` is a denormalized counter. It's incremented/decremented when posts are tagged/untagged. A background job reconciles counts nightly:
```sql
UPDATE tags t SET post_count = (
    SELECT COUNT(*) FROM post_tags pt
    JOIN posts p ON pt.post_id = p.id
    WHERE pt.tag_id = t.id AND p.status = 'PUBLISHED'
);
```

## 6. Failure Scenarios

### Slug Collision on Publish
- **Impact**: Two authors publish posts with identical titles simultaneously; unique constraint violation
- **Recovery**: Catch constraint violation; append a different random suffix and retry
- **Prevention**: Generate slug with random suffix by default; unique index catches the rare collision

### Full-Text Search Index Lag
- **Impact**: Newly published posts don't appear in search immediately
- **Recovery**: GIN index updates synchronously on INSERT/UPDATE — no lag for PostgreSQL FTS
- **Prevention**: For Elasticsearch sync, use CDC with at-least-once delivery; deduplicate on index

### Like Count Drift
- **Impact**: `like_count` on posts diverges from actual count in `post_likes`
- **Recovery**: Nightly reconciliation job recomputes counts from source table
- **Prevention**: Use atomic `UPDATE ... SET like_count = like_count + 1`; wrap in transaction with `post_likes` insert

### Deleted Author's Posts
- **Impact**: Author account deleted; posts become orphaned
- **Recovery**: Soft-delete users; posts remain with `[deleted]` author display; never hard-delete authors with published posts
- **Prevention**: Check for published posts before allowing account deletion; require explicit content transfer or deletion
