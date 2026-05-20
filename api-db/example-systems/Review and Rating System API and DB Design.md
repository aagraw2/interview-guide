# Review and Rating System API and DB Design

## Problem Statement

Design the API and database schema for a review and rating system where users can rate products/services, write reviews, and see aggregate scores.

**Core requirements:**
- Submit/update/delete reviews
- 1 rating per user per target entity
- Compute aggregate rating and count
- Support moderation and abuse reporting

**Scale:** 200M reviews, high read traffic for product pages

---

## API Design

### Core Endpoints
```http
POST /api/v1/reviews
PATCH /api/v1/reviews/{reviewId}
DELETE /api/v1/reviews/{reviewId}
GET /api/v1/entities/{entityId}/reviews?sort=recent&cursor=...&limit=20
GET /api/v1/entities/{entityId}/rating-summary
POST /api/v1/reviews/{reviewId}/report
```

### Example: Create Review
```json
POST /api/v1/reviews
{
  "entityId": "prd_901",
  "rating": 4,
  "title": "Great value",
  "body": "Good quality for the price."
}
```

```json
201 Created
{
  "reviewId": "rev_121",
  "status": "PUBLISHED"
}
```

### Design Decisions
- Enforce unique `(entity_id, user_id)` to prevent duplicate ratings
- Cursor pagination for large review lists
- Denormalized rating summary for fast reads

---

## Database Schema

```sql
CREATE TABLE reviews (
    id                  VARCHAR(64) PRIMARY KEY,
    entity_id           VARCHAR(64) NOT NULL,
    user_id             VARCHAR(64) NOT NULL,
    rating              SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title               VARCHAR(200),
    body                TEXT,
    status              VARCHAR(24) NOT NULL, -- PUBLISHED, HIDDEN, PENDING
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (entity_id, user_id)
);

CREATE INDEX idx_reviews_entity_created ON reviews (entity_id, created_at DESC);
CREATE INDEX idx_reviews_entity_rating ON reviews (entity_id, rating);

CREATE TABLE rating_summaries (
    entity_id           VARCHAR(64) PRIMARY KEY,
    avg_rating          NUMERIC(3,2) NOT NULL,
    rating_count        BIGINT NOT NULL,
    star_1_count        BIGINT NOT NULL DEFAULT 0,
    star_2_count        BIGINT NOT NULL DEFAULT 0,
    star_3_count        BIGINT NOT NULL DEFAULT 0,
    star_4_count        BIGINT NOT NULL DEFAULT 0,
    star_5_count        BIGINT NOT NULL DEFAULT 0,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE review_reports (
    id                  BIGSERIAL PRIMARY KEY,
    review_id           VARCHAR(64) NOT NULL REFERENCES reviews(id),
    reported_by         VARCHAR(64) NOT NULL,
    reason              VARCHAR(100) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Trade-offs and Considerations

- Real-time aggregate updates are simple but can add write contention.
- Async recomputation is scalable but introduces temporary staleness.
- Moderation workflow can be rule-based first, then ML-assisted at scale.
