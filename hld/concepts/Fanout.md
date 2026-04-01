## 1. What is Fanout?

Fanout is the process of delivering content (posts, notifications, updates) from one source to multiple recipients. The term comes from how a single input "fans out" to many outputs.

```
User posts a tweet:
  → Deliver to 1000 followers

Celebrity posts a tweet:
  → Deliver to 10 million followers

How do you efficiently deliver content to millions of users?
```

**Core use cases:** Social media feeds (Twitter, Instagram), notifications, activity streams, news feeds.

---

## 2. Fanout Strategies

### Fanout-on-Write (Push Model)

Write the content to all recipients' feeds immediately when it's created.

```
User A posts: "Hello world"
  → Write to User B's feed
  → Write to User C's feed
  → Write to User D's feed
  → ... (for all followers)

When User B opens their feed:
  → Read from their pre-computed feed (fast)
```

**Also called:** Push model, eager evaluation, pre-compute

### Fanout-on-Read (Pull Model)

Compute the feed when a user requests it by querying all followed users.

```
User B opens their feed:
  → Query: Get posts from users B follows (A, C, D, ...)
  → Merge and sort by timestamp
  → Return feed

No pre-computation, everything happens at read time
```

**Also called:** Pull model, lazy evaluation, on-demand

---

## 3. Fanout-on-Write (Push)

### How it works

```
1. User A creates post (id=123)

2. Get User A's followers:
   SELECT follower_id FROM followers WHERE user_id = A
   → Returns: [B, C, D, E, F]

3. Write post to each follower's feed:
   INSERT INTO feed (user_id, post_id) VALUES (B, 123)
   INSERT INTO feed (user_id, post_id) VALUES (C, 123)
   INSERT INTO feed (user_id, post_id) VALUES (D, 123)
   INSERT INTO feed (user_id, post_id) VALUES (E, 123)
   INSERT INTO feed (user_id, post_id) VALUES (F, 123)

4. User B reads feed:
   SELECT post_id FROM feed WHERE user_id = B ORDER BY created_at DESC LIMIT 20
   → Fast read (pre-computed)
```

### Pros

```
✅ Fast reads (feed is pre-computed)
✅ Simple read logic (just query user's feed)
✅ Good for users with few followers
✅ Consistent read performance
```

### Cons

```
❌ Slow writes (must write to all followers)
❌ Wasted work (write to inactive users)
❌ Celebrity problem (10M followers = 10M writes)
❌ Storage overhead (duplicate posts in many feeds)
```

### Celebrity problem

```
Celebrity with 10 million followers posts:
  → 10 million writes to feed tables
  → Takes minutes to complete
  → Blocks other operations
  → Wastes resources (most followers won't read it)

This is the main limitation of fanout-on-write
```

---

## 4. Fanout-on-Read (Pull)

### How it works

```
1. User A creates post (id=123)
   INSERT INTO posts (id, user_id, content) VALUES (123, A, "Hello")
   → Single write, no fanout

2. User B reads feed:
   a. Get users B follows:
      SELECT following_id FROM following WHERE user_id = B
      → Returns: [A, C, D, E, F]
   
   b. Get recent posts from followed users:
      SELECT * FROM posts
      WHERE user_id IN (A, C, D, E, F)
      ORDER BY created_at DESC
      LIMIT 20
   
   c. Return merged feed
```

### Pros

```
✅ Fast writes (single write, no fanout)
✅ No wasted work (only compute for active users)
✅ No celebrity problem (same cost for all users)
✅ No storage overhead (posts stored once)
```

### Cons

```
❌ Slow reads (must query and merge at read time)
❌ Complex read logic (join, merge, sort)
❌ Inconsistent read performance (depends on # of followed users)
❌ Database load (every read hits database)
```

### Hotspot problem

```
User follows 5000 users:
  → Query posts from 5000 users
  → Merge and sort
  → Very slow read

Celebrity's posts are queried by millions of users:
  → Database hotspot
  → Cache helps, but still high load
```

---

## 5. Hybrid Approach

Combine fanout-on-write and fanout-on-read based on user characteristics.

### Strategy

```
Regular users (< 1000 followers):
  → Fanout-on-write (push to all followers)

Celebrities (> 1000 followers):
  → No fanout (store post only)

Reading feed:
  1. Get pre-computed feed (from fanout-on-write)
  2. Merge with posts from celebrities (fanout-on-read)
  3. Sort and return
```

### Example: Twitter

```
User B follows:
  - 100 regular users (fanout-on-write)
  - 5 celebrities (fanout-on-read)

User B reads feed:
  1. Get pre-computed feed from 100 regular users (fast)
  2. Query recent posts from 5 celebrities (fast, cached)
  3. Merge and sort
  4. Return top 20 posts

Write:
  Regular user posts → fanout to followers (fast, few followers)
  Celebrity posts → no fanout (fast, single write)

Read:
  Fast for most users (pre-computed + small celebrity query)
```

### Pros

```
✅ Fast writes for celebrities (no fanout)
✅ Fast reads for most users (pre-computed)
✅ Balanced approach (best of both worlds)
```

### Cons

```
❌ Complex implementation (two code paths)
❌ Need to define "celebrity" threshold
❌ Edge cases (user crosses threshold)
```

---

## 6. Implementation Details

### Fanout-on-write with message queue

```
1. User A creates post
   → Publish event to message queue

2. Fanout workers consume events
   Worker 1: Write to followers 1-1000
   Worker 2: Write to followers 1001-2000
   Worker 3: Write to followers 2001-3000
   ...

3. Parallel processing
   → Faster fanout
   → Doesn't block user's request

Message queue:
  - Kafka, RabbitMQ, SQS
  - Decouples write from fanout
  - Enables async processing
```

### Feed storage

```
Option 1: Relational database
  feed table:
    user_id | post_id | created_at
    --------|---------|------------
    B       | 123     | 2024-01-01
    B       | 124     | 2024-01-02
    C       | 123     | 2024-01-01

  Pros: Simple, ACID guarantees
  Cons: Expensive joins, hard to scale

Option 2: Redis (sorted set)
  Key: feed:user_B
  Value: Sorted set of post IDs (score = timestamp)
  
  ZADD feed:user_B 1704067200 123
  ZADD feed:user_B 1704153600 124
  
  ZREVRANGE feed:user_B 0 19  # Get top 20 posts

  Pros: Very fast reads, easy to scale
  Cons: No ACID, limited query capabilities

Option 3: Cassandra (wide column)
  CREATE TABLE feed (
    user_id UUID,
    created_at TIMESTAMP,
    post_id UUID,
    PRIMARY KEY (user_id, created_at)
  ) WITH CLUSTERING ORDER BY (created_at DESC);

  Pros: Scalable, fast reads, time-series optimized
  Cons: Limited query flexibility
```

---

## 7. Optimizations

### 1. Limit fanout size

```
Only fanout to active users (logged in within 30 days)
  → Reduces wasted writes
  → Inactive users get fanout-on-read

Only fanout to first N posts in feed
  → Users rarely scroll past 100 posts
  → Older posts can be fetched on-demand
```

### 2. Batch writes

```
Instead of:
  INSERT INTO feed VALUES (B, 123)
  INSERT INTO feed VALUES (C, 123)
  INSERT INTO feed VALUES (D, 123)

Batch:
  INSERT INTO feed VALUES (B, 123), (C, 123), (D, 123)

Reduces database round-trips
```

### 3. Caching

```
Cache celebrity posts:
  Key: post:123
  Value: {user_id: A, content: "Hello", created_at: ...}

Cache user feeds:
  Key: feed:user_B
  Value: [123, 124, 125, ...]

Cache follower lists:
  Key: followers:user_A
  Value: [B, C, D, E, F]

Reduces database load
```

### 4. Pagination

```
Don't load entire feed at once
  → Load 20 posts per page
  → Fetch more on scroll

Cursor-based pagination:
  GET /feed?cursor=2024-01-01T00:00:00
  → Returns posts older than cursor
  → Efficient for time-series data
```

---

## 8. Real-World Examples

### Twitter

```
Hybrid approach:
  - Regular users: Fanout-on-write
  - Celebrities: Fanout-on-read
  - Feed: Merge both sources

Timeline:
  - Home timeline: Hybrid fanout
  - User timeline: Fanout-on-read (query user's posts)
  - Search: Fanout-on-read (query matching posts)
```

### Instagram

```
Fanout-on-write for most users
  - Posts written to followers' feeds
  - Stories have 24-hour TTL (auto-delete)

Feed ranking:
  - Not purely chronological
  - ML model ranks posts by engagement probability
  - Fanout writes post ID, ranking happens at read time
```

### Facebook

```
Hybrid approach with ranking:
  - Fanout-on-write for friends
  - Fanout-on-read for pages/groups
  - EdgeRank algorithm ranks posts at read time

News Feed:
  - Pre-computed feed (fanout-on-write)
  - Re-ranked on every read (personalized)
```

---

## 9. Fanout for Notifications

Similar problem: Deliver notifications to many users.

### Push notifications

```
User A likes User B's post:
  → Send push notification to User B

User A posts:
  → Send push notification to followers (fanout)

Challenges:
  - Don't spam users (rate limiting)
  - Batch notifications (User A liked 5 of your posts)
  - Respect user preferences (mute, do not disturb)
```

### Email notifications

```
Fanout-on-write:
  - Generate email for each recipient
  - Queue for delivery

Batching:
  - Don't send email for every event
  - Batch: "You have 10 new notifications"
  - Send digest emails (daily, weekly)
```

---

## 10. Common Interview Questions + Answers

### Q: What's the difference between fanout-on-write and fanout-on-read?

> "Fanout-on-write pre-computes the feed by writing each post to all followers' feeds immediately. This makes reads fast but writes slow, especially for celebrities with millions of followers. Fanout-on-read computes the feed on-demand by querying posts from followed users when the user requests their feed. This makes writes fast but reads slow. In practice, systems use a hybrid approach — fanout-on-write for regular users and fanout-on-read for celebrities."

### Q: How would you design Twitter's home timeline?

> "I'd use a hybrid approach. For regular users with fewer than 1000 followers, use fanout-on-write — when they post, write the post ID to all followers' feeds asynchronously via a message queue. For celebrities, don't fanout — just store the post. When a user reads their timeline, fetch their pre-computed feed and merge it with recent posts from celebrities they follow. Store feeds in Redis sorted sets for fast reads. Cache celebrity posts heavily since they're read by millions of users."

### Q: What's the celebrity problem in fanout-on-write?

> "The celebrity problem is when a user with millions of followers posts, and the system must write to millions of feeds. This takes a long time, wastes resources (most followers won't read it), and can overwhelm the database. The solution is to not fanout for celebrities — use fanout-on-read instead. When users read their feed, query recent posts from celebrities they follow and merge with their pre-computed feed."

### Q: How do you handle inactive users in fanout-on-write?

> "Don't fanout to inactive users — only write to users who have logged in within the last 30 days. This reduces wasted writes. For inactive users who return, compute their feed on-demand using fanout-on-read. You can also limit fanout to the first 100 posts in each user's feed, since users rarely scroll past that. Older posts can be fetched on-demand if needed."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Know both strategies

Fanout-on-write and fanout-on-read are both important. Know the trade-offs and when to use each.

### ✅ Trick 2: Mention the hybrid approach

Real systems use a hybrid approach. Mentioning this shows you understand practical implementations.

### ✅ Trick 3: Discuss the celebrity problem

The celebrity problem is a classic interview topic. Always mention it when discussing fanout-on-write.

### ❌ Pitfall 1: Thinking one strategy is always better

Neither fanout-on-write nor fanout-on-read is universally better. The best approach depends on the use case and user characteristics.

### ❌ Pitfall 2: Forgetting about async processing

Fanout-on-write should be async (message queue) to avoid blocking the user's request. Don't do synchronous fanout.

### ❌ Pitfall 3: Ignoring storage costs

Fanout-on-write duplicates posts in many feeds, increasing storage costs. Mention this trade-off.

---

## 12. Quick Reference

```
What is fanout?
  Deliver content from one source to multiple recipients
  Use cases: Social media feeds, notifications, activity streams

Fanout-on-write (Push):
  Pre-compute feed by writing to all followers immediately
  Pros: Fast reads, simple read logic
  Cons: Slow writes, celebrity problem, storage overhead

Fanout-on-read (Pull):
  Compute feed on-demand by querying followed users
  Pros: Fast writes, no celebrity problem
  Cons: Slow reads, complex read logic, database load

Celebrity problem:
  User with 10M followers posts → 10M writes
  Solution: Don't fanout for celebrities (use fanout-on-read)

Hybrid approach:
  Regular users: Fanout-on-write
  Celebrities: Fanout-on-read
  Read: Merge both sources

Implementation:
  Message queue: Async fanout (Kafka, RabbitMQ)
  Storage: Redis sorted sets, Cassandra, SQL
  Caching: Celebrity posts, user feeds, follower lists

Optimizations:
  - Limit fanout to active users (logged in within 30 days)
  - Limit fanout to first N posts (users rarely scroll past 100)
  - Batch writes (reduce database round-trips)
  - Pagination (load 20 posts per page)

Real-world:
  Twitter: Hybrid (regular users push, celebrities pull)
  Instagram: Fanout-on-write with ML ranking
  Facebook: Hybrid with EdgeRank algorithm

Notifications:
  Similar fanout problem
  Rate limiting (don't spam)
  Batching (digest emails)

Trade-offs:
  Fanout-on-write: Fast reads, slow writes
  Fanout-on-read: Fast writes, slow reads
  Hybrid: Balanced, but complex
```
