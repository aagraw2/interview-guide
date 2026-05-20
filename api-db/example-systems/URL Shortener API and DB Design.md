# URL Shortener API and DB Design

## Problem Statement

Design the API and database schema for a URL shortening service (like bit.ly). Users submit a long URL and receive a short code. Anyone with the short code can be redirected to the original URL.

**Core requirements:**
- Shorten a long URL to a short code
- Redirect short code to original URL
- Track click counts per short URL
- Optional: custom aliases, expiry dates, analytics

**Scale:** 100M URLs created per day, 10B redirects per day (read-heavy, ~100:1 read/write ratio)

---

## API Design

### Endpoints

#### Create Short URL
```
POST /api/v1/urls
Authorization: Bearer <token>

Request:
{
  "longUrl": "https://example.com/very/long/path?with=params",
  "customAlias": "my-link",        // optional
  "expiresAt": "2025-12-31T00:00:00Z"  // optional
}

Response 201:
{
  "shortCode": "abc123",
  "shortUrl": "https://short.ly/abc123",
  "longUrl": "https://example.com/very/long/path?with=params",
  "createdAt": "2024-01-15T10:00:00Z",
  "expiresAt": "2025-12-31T00:00:00Z"
}

Errors:
- 400: Invalid URL format
- 409: Custom alias already taken
- 422: URL exceeds maximum length
```

#### Redirect (Public)
```
GET /{shortCode}

Response 301/302: Location: <longUrl>

Errors:
- 404: Short code not found
- 410: Short URL has expired
```

#### Get URL Stats
```
GET /api/v1/urls/{shortCode}/stats
Authorization: Bearer <token>

Response 200:
{
  "shortCode": "abc123",
  "longUrl": "https://example.com/...",
  "clickCount": 15234,
  "createdAt": "2024-01-15T10:00:00Z",
  "lastClickedAt": "2024-01-20T08:30:00Z"
}
```

#### List User's URLs
```
GET /api/v1/urls?page=1&pageSize=20
Authorization: Bearer <token>

Response 200:
{
  "items": [...],
  "total": 150,
  "page": 1,
  "pageSize": 20
}
```

#### Delete Short URL
```
DELETE /api/v1/urls/{shortCode}
Authorization: Bearer <token>

Response 204
```

### Design Decisions
- Use `301 Permanent Redirect` for SEO; use `302 Temporary Redirect` if click tracking is needed (browser won't cache 302)
- Idempotency: same long URL from same user returns existing short code (dedup)
- Rate limit creation endpoint: 100 URLs/hour per user

---

## Database Schema

### urls table
```sql
CREATE TABLE urls (
    id          BIGSERIAL PRIMARY KEY,
    short_code  VARCHAR(10) NOT NULL UNIQUE,
    long_url    TEXT NOT NULL,
    user_id     BIGINT REFERENCES users(id),
    click_count BIGINT NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at  TIMESTAMPTZ,
    is_active   BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE INDEX idx_urls_short_code ON urls (short_code);
CREATE INDEX idx_urls_user_id ON urls (user_id);
CREATE INDEX idx_urls_expires_at ON urls (expires_at) WHERE expires_at IS NOT NULL;
```

### url_clicks table (analytics)
```sql
CREATE TABLE url_clicks (
    id          BIGSERIAL PRIMARY KEY,
    url_id      BIGINT NOT NULL REFERENCES urls(id),
    clicked_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ip_hash     VARCHAR(64),    -- hashed for privacy
    user_agent  VARCHAR(500),
    referrer    VARCHAR(2000)
);

CREATE INDEX idx_url_clicks_url_id ON url_clicks (url_id);
CREATE INDEX idx_url_clicks_clicked_at ON url_clicks (clicked_at);
```

### users table
```sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    api_key     VARCHAR(64) UNIQUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Trade-offs and Considerations

### Short Code Generation
- **Random**: Generate 6-char base62 string, check for collision in DB. Simple but collision probability grows with scale.
- **Hash-based**: MD5/SHA256 of long URL, take first 6 chars. Deterministic but collision-prone.
- **Counter-based**: Auto-increment ID encoded in base62. No collisions, but sequential codes are guessable.
- **Recommendation**: Counter-based with base62 encoding for uniqueness guarantees.

### Click Count Updates
- Direct `UPDATE urls SET click_count = click_count + 1` on every redirect creates write contention at scale.
- Better: Write clicks to `url_clicks` table asynchronously; aggregate periodically.
- Or: Use Redis counter (`INCR short_code:clicks`) and sync to DB in batches.

### Caching
- Cache `short_code → long_url` mapping in Redis with TTL matching the URL's expiry.
- Cache hit rate should be very high (popular URLs get most traffic).
- On cache miss, query DB and populate cache.

### Read vs Write Scaling
- Reads (redirects) vastly outnumber writes — scale read replicas aggressively.
- Writes (URL creation) are low volume — single primary is sufficient initially.
- Consider CDN-level caching for the redirect endpoint.
