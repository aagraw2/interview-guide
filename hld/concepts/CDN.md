## 1. What Is a CDN?

A Content Delivery Network (CDN) is a geographically distributed network of servers that cache and serve content from locations close to users.

```
Without CDN:
  User in Tokyo → 200ms → Origin Server in US-East → 200ms → User
  Total: 400ms round trip

With CDN:
  User in Tokyo → 5ms → CDN Edge (Tokyo) → 5ms → User
  Total: 10ms round trip
```

**Core benefits:**
- Reduced latency (content served from nearby edge)
- Reduced origin load (CDN absorbs traffic)
- Improved availability (distributed, DDoS resilient)
- Better bandwidth efficiency

---

## 2. How CDN Works

### Basic flow

```
1. User requests https://example.com/image.jpg
2. DNS resolves to nearest CDN edge server (GeoDNS)
3. Edge server checks local cache:
   a. Cache HIT  → serve immediately
   b. Cache MISS → fetch from origin → cache → serve
4. Subsequent requests from that region → cache hit
```

### CDN architecture

```
                    Origin Server
                    (Your backend)
                          │
                          │ (cache miss)
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
   [CDN Edge]        [CDN Edge]        [CDN Edge]
   US-East           EU-West           APAC
        │                 │                 │
    Users (US)        Users (EU)        Users (Asia)
```

Each edge location (Point of Presence / PoP) caches content independently.

---

## 3. Push vs Pull CDN

### Pull CDN (Origin Pull)

CDN fetches content from origin on first request (cache miss).

```
First request:
  User → CDN Edge (miss) → Origin → CDN caches → User

Subsequent requests:
  User → CDN Edge (hit) → User
```

**Pros:**
- Simple setup — just point CDN at your origin
- No manual content upload
- Content automatically updated when origin changes

**Cons:**
- First request to each edge is slow (cache miss)
- Origin must handle cache miss traffic

**Use for:** Dynamic sites, frequently changing content, large catalogs

**Examples:** Cloudflare, AWS CloudFront (default), Fastly

---

### Push CDN

You manually upload content to the CDN. CDN doesn't fetch from origin.

```
Deploy process:
  Build assets → Upload to CDN → CDN distributes to all edges

User request:
  User → CDN Edge (always hit) → User
```

**Pros:**
- No cache misses (content pre-populated)
- Origin never hit by CDN
- Full control over what's cached

**Cons:**
- Manual upload on every deploy
- Storage costs (you're storing everything)
- Stale content if you forget to update

**Use for:** Static sites, infrequently changing assets, large files

**Examples:** AWS S3 + CloudFront, Netlify, Vercel

---

## 4. What Can Be Cached?

### Static assets (always cacheable)

```
Images:     .jpg, .png, .webp, .svg
Videos:     .mp4, .webm
Stylesheets: .css
Scripts:    .js
Fonts:      .woff2, .ttf
Documents:  .pdf
```

**Cache headers:**

```
Cache-Control: public, max-age=31536000, immutable
```

Cached for 1 year. Versioned filenames (e.g., `app.abc123.js`) ensure updates work.

---

### Dynamic content (conditionally cacheable)

```
HTML pages:     Cache with short TTL (60s) or use stale-while-revalidate
API responses:  Cache GET requests with appropriate TTL
Personalized:   Don't cache, or use Vary: Cookie header carefully
```

**Example: Caching HTML with freshness check**

```
Cache-Control: public, max-age=60, stale-while-revalidate=86400

Behavior:
  - Serve from cache for 60s
  - After 60s: serve stale version immediately, fetch fresh in background
  - User never waits for origin
```

---

### What NOT to cache

```
❌ User-specific data without Vary header
❌ POST/PUT/DELETE requests
❌ Responses with Set-Cookie (unless you know what you're doing)
❌ Sensitive data (unless encrypted and access-controlled)
❌ Real-time data (stock prices, live scores)
```

---

## 5. Cache Invalidation

The hardest problem in CDN usage: how do you update cached content?

### Time-based expiration (TTL)

Set `Cache-Control: max-age=3600` — content expires after 1 hour.

```
Pros: Simple, no coordination needed
Cons: Stale content for up to TTL duration
```

**Use for:** Content that changes infrequently or where staleness is acceptable.

---

### Versioned URLs (cache busting)

Change the URL when content changes.

```
Old: /static/app.js
New: /static/app.v2.js  (or app.abc123.js with content hash)

Cache-Control: public, max-age=31536000, immutable
```

Old URL stays cached forever. New URL is a cache miss → fetches fresh.

**This is the gold standard for static assets.**

---

### Purge / Invalidation API

Explicitly tell the CDN to delete cached content.

```
POST /purge
{
  "files": ["/api/products/123", "/images/product-123.jpg"]
}
```

**Pros:** Immediate update across all edges. **Cons:** Costs money (Cloudflare charges for purges beyond free tier), not instant (propagation takes seconds to minutes).

**Use for:** Urgent fixes, content takedowns, breaking news.

---

### Stale-while-revalidate

Serve stale content immediately while fetching fresh in the background.

```
Cache-Control: max-age=60, stale-while-revalidate=86400

Timeline:
  0-60s:    Serve from cache (fresh)
  60s-24h:  Serve from cache (stale), fetch fresh in background
  24h+:     Cache miss, fetch from origin
```

**Best of both worlds:** Users never wait, content stays reasonably fresh.

---

## 6. Cache Key and Vary Header

By default, CDN caches based on the full URL:

```
https://example.com/api/products?sort=price  → Cache key 1
https://example.com/api/products?sort=name   → Cache key 2
```

### Vary header

Tells CDN to include specific headers in the cache key.

```
Vary: Accept-Encoding

Cache keys:
  /api/products + Accept-Encoding: gzip     → Cache entry 1
  /api/products + Accept-Encoding: br       → Cache entry 2
  /api/products + Accept-Encoding: identity → Cache entry 3
```

**Common Vary headers:**

```
Vary: Accept-Encoding   (gzip vs brotli vs none)
Vary: Accept-Language   (en vs es vs fr)
Vary: User-Agent        (mobile vs desktop — use sparingly)
```

**Warning:** `Vary: Cookie` or `Vary: User-Agent` can fragment cache into thousands of entries, destroying hit rate.

---

## 7. Origin Shield

An additional caching layer between edge servers and origin.

```
Without Origin Shield:
  100 edge servers → cache miss → 100 requests to origin

With Origin Shield:
  100 edge servers → Origin Shield (1 cache miss) → 1 request to origin
```

**Benefits:**
- Reduces origin load during cache misses
- Protects origin from thundering herd
- Higher cache hit rate (shield has more traffic → warmer cache)

**Use when:** Origin is expensive to scale, or you have many edge locations.

**Examples:** CloudFront Origin Shield, Fastly shielding

---

## 8. CDN for Dynamic Content

CDNs aren't just for static files. Modern CDNs accelerate dynamic content too.

### Edge compute

Run code at the edge to generate responses without hitting origin.

```
Cloudflare Workers:
  User → Edge → Run JS function → Generate response
  No origin request needed

Use cases:
  - A/B testing (route to different origins)
  - Authentication checks
  - Personalization (inject user name into cached HTML)
  - API aggregation (call multiple backends, combine)
```

**Examples:** Cloudflare Workers, AWS Lambda@Edge, Fastly Compute@Edge

---

### Smart routing

CDN routes requests to the nearest healthy origin over optimized network paths.

```
User in Tokyo → CDN Edge (Tokyo) → Origin (US-East)
  Without CDN: public internet, 200ms
  With CDN: CDN's private backbone, 120ms
```

Even for cache misses, latency improves.

---

## 9. CDN Security Features

### DDoS protection

CDN absorbs attack traffic at the edge, far from your origin.

```
Attack: 1 Tbps DDoS
  Without CDN: Origin overwhelmed, site down
  With CDN: Distributed across 200+ PoPs, absorbed
```

### Web Application Firewall (WAF)

Filter malicious requests at the edge.

```
Block:
  - SQL injection attempts
  - XSS payloads
  - Known bot signatures
  - Rate limit by IP
```

### TLS termination

CDN handles HTTPS, reducing origin compute load.

```
User → HTTPS → CDN Edge → HTTP → Origin (internal network)
```

Origin doesn't need to decrypt TLS.

---

## 10. CDN Providers

|Provider|Strengths|Use Cases|
|---|---|---|
|Cloudflare|Free tier, DDoS protection, Workers (edge compute)|General purpose, security-focused|
|AWS CloudFront|Deep AWS integration, Lambda@Edge|AWS-native apps|
|Fastly|Real-time purging, VCL customization, edge compute|High-control, real-time needs|
|Akamai|Largest network, enterprise features|Enterprise, media streaming|
|Vercel|Optimized for Next.js, automatic|Modern web apps, Jamstack|
|Netlify|Integrated with Git, automatic deploys|Static sites, Jamstack|

---

## 11. Common Interview Questions + Answers

### Q: How does a CDN reduce latency?

> "A CDN caches content at edge servers geographically close to users. When a user in Tokyo requests an image, instead of fetching it from the origin server in the US (200ms round trip), they get it from a Tokyo edge server (5ms). The first request to each edge is a cache miss and hits the origin, but subsequent requests are served from cache. For dynamic content, even cache misses benefit from the CDN's optimized network backbone between edge and origin."

### Q: What's the difference between push and pull CDN?

> "Pull CDN fetches content from origin on cache miss — you just point the CDN at your origin and it automatically caches. It's simple but the first request to each edge is slow. Push CDN requires you to manually upload content to the CDN — no cache misses, but you must update it on every deploy. Pull is better for dynamic sites with frequently changing content. Push is better for static sites where you control the build process."

### Q: How do you handle cache invalidation?

> "The best approach is versioned URLs — use content hashes in filenames like `app.abc123.js` and set a long TTL. When content changes, the filename changes, so it's a new cache key. For content that can't be versioned, use short TTLs with stale-while-revalidate so users get fast responses while the CDN fetches fresh content in the background. For urgent updates, use the CDN's purge API, but that's a last resort since it costs money and takes time to propagate."

### Q: Can you cache API responses?

> "Yes, but carefully. Cache GET requests that return the same data for all users — like product catalogs, public posts, or reference data. Use appropriate TTLs based on how fresh the data needs to be. Don't cache user-specific data unless you use Vary: Cookie or separate cache keys per user. Never cache POST/PUT/DELETE. For personalized APIs, consider edge compute to inject user-specific data into cached templates."

---

## 12. Interview Tricks & Pitfalls

### ✅ Trick 1: Mention cache hit ratio

When discussing CDN, bring up the metric that matters:

> "The goal is a high cache hit ratio — ideally 90%+. This means 90% of requests are served from cache, only 10% hit the origin. To achieve this, use long TTLs for static assets, versioned URLs to avoid premature invalidation, and Origin Shield to consolidate cache misses."

### ✅ Trick 2: Connect CDN to the broader system

CDN isn't standalone:

> "I'd use CloudFront for static assets with versioned URLs and a 1-year TTL. For the API, I'd cache GET responses with a 60-second TTL and stale-while-revalidate. The CDN also provides DDoS protection and WAF rules to block malicious traffic before it reaches our origin."

### ✅ Trick 3: Know the cost model

Shows operational awareness:

> "CDN costs are based on bandwidth and requests. By caching aggressively and using Origin Shield, we reduce origin bandwidth costs. The CDN bandwidth cost is offset by improved performance and reduced origin infrastructure needs."

### ❌ Pitfall 1: Saying "CDN is only for static files"

Modern CDNs accelerate dynamic content, provide edge compute, and offer security features. Don't limit your answer to images and CSS.

### ❌ Pitfall 2: Forgetting about cache invalidation

If you propose caching, the interviewer will ask "what if the content changes?" Have the answer ready: versioned URLs for static assets, short TTLs with stale-while-revalidate for dynamic content.

### ❌ Pitfall 3: Not mentioning GeoDNS

The CDN needs to route users to the nearest edge. This happens via GeoDNS (DNS returns different IPs based on user location) or Anycast (same IP, routed to nearest PoP). Mention this to show you understand the routing layer.

---

## 13. Quick Reference

```
CDN = geographically distributed cache

Benefits:
  - Reduced latency (serve from nearby edge)
  - Reduced origin load (CDN absorbs traffic)
  - DDoS protection (distributed absorption)
  - Improved availability

Pull CDN:
  - CDN fetches from origin on cache miss
  - Simple setup, automatic updates
  - Use for: dynamic sites, frequently changing content

Push CDN:
  - Manually upload content to CDN
  - No cache misses, full control
  - Use for: static sites, infrequent changes

Cache invalidation:
  1. Versioned URLs (best for static assets)
  2. Short TTL + stale-while-revalidate (best for dynamic)
  3. Purge API (last resort, costs money)

Cache key:
  Default: full URL
  Vary header: include specific headers in key
  Warning: Vary: Cookie fragments cache

Origin Shield:
  Extra cache layer between edge and origin
  Reduces origin load, protects from thundering herd

Edge compute:
  Run code at edge (Cloudflare Workers, Lambda@Edge)
  Generate responses without origin request

Providers: Cloudflare, CloudFront, Fastly, Akamai, Vercel, Netlify
```
