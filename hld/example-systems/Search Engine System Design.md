# Search Engine System Design

## System Overview
A web-scale search engine (think Google / Bing) that crawls the web, indexes content, and serves relevant search results in milliseconds — covering web crawling, indexing, ranking, and query serving.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Crawl and index web pages continuously
- Full-text search with ranked results
- Autocomplete / query suggestions
- Spell correction
- Search result snippets with highlighted keywords
- Freshness: recently updated pages ranked appropriately
- Safe search filtering

### Non-Functional Requirements
- Latency: <200ms for search results
- Scale: 100B+ web pages indexed, 10B+ queries/day
- Freshness: popular pages re-crawled within hours; others within days/weeks
- Availability: 99.99%
- Relevance: results must be highly relevant to query

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 100B web pages indexed
- Average page size: 100KB raw, 10KB extracted text
- 10B queries/day, 100K queries/sec at peak
- 1B new/updated pages/day to crawl

### Traffic
```
Queries/sec (peak)  = 100K/sec
Crawl rate          = 1B pages/day = 11.6K pages/sec
Index writes/sec    = 11.6K/sec
```

### Storage
```
Raw pages           = 100B × 100KB = 10PB (compressed: ~2PB)
Extracted text      = 100B × 10KB  = 1PB
Inverted index      = ~500TB (compressed)
Page rank scores    = 100B × 8B = 800GB
URL frontier        = 10B URLs × 100B = 1TB
```

## 3. Core Components

**URL Frontier** — Priority queue of URLs to crawl; manages crawl scheduling, politeness (robots.txt), recrawl frequency

**Web Crawler** — Distributed crawlers that fetch web pages; respects robots.txt; stores raw HTML to S3; extracts links for URL Frontier

**Content Processor** — Extracts text from HTML; deduplication; language detection; stores processed content to Document Store

**Indexer** — Builds inverted index from processed documents; maps term → list of (docId, positions, frequency)

**Inverted Index Store** — Distributed index sharded by term; the core data structure for search

**Ranking Service** — Scores documents for a query using TF-IDF, PageRank, freshness, and ML signals; returns ranked list

**PageRank Calculator** — Offline batch job; computes PageRank from web graph (link structure); runs periodically

**Query Service** — Entry point for search queries; query parsing, spell correction, autocomplete; orchestrates index lookup + ranking

**Autocomplete Service** — Prefix-based query suggestions; backed by trie or Elasticsearch prefix search

**Snippet Generator** — Extracts relevant text snippet from document for display in results

**Document Store (Cassandra)** — Processed page content, metadata, PageRank scores

**Inverted Index (custom distributed store)** — Term → posting list (docId, TF, positions)

**URL Store (Cassandra)** — Crawled URLs, last crawl time, crawl frequency, status

**Redis** — Query cache, autocomplete cache, session store

**S3** — Raw crawled HTML, index snapshots

**Kafka** — Crawl job queue, index update stream

## 4. Database Design

### Cassandra — documents

| Field | Type |
|---|---|
| doc_id | UUID (PK) |
| url | TEXT |
| title | VARCHAR |
| content_hash | VARCHAR (for dedup) |
| extracted_text | TEXT |
| language | VARCHAR |
| page_rank | FLOAT |
| last_crawled | TIMESTAMP |
| outbound_links | LIST\<TEXT\> |

### Cassandra — url_frontier

| Field | Type |
|---|---|
| url_hash | VARCHAR (PK) |
| url | TEXT |
| priority | FLOAT |
| last_crawled | TIMESTAMP |
| crawl_frequency | INT (hours) |
| status | ENUM (pending / crawling / crawled / failed) |

### Inverted Index — posting list structure

```
term: "python"
→ posting list: [
    {docId: 123, tf: 5, positions: [10, 45, 102]},
    {docId: 456, tf: 3, positions: [5, 20]},
    ...
  ]
  (sorted by relevance score)
```

Sharded by term hash across N index servers. Each shard holds a subset of terms.

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `query:cache:{queryHash}` | String | result JSON | 300s |
| `autocomplete:{prefix}` | List | suggested queries | 3600s |
| `trending:queries` | ZSET | query → frequency | 3600s |

## 5. Key Flows

### 5.1 Web Crawling

```
URL Frontier → Crawler → fetch page → S3 (raw HTML)
                                    → extract links → URL Frontier
                                    → Content Processor → Document Store
                                    → Kafka (new doc) → Indexer
```

1. URL Frontier dequeues URL with highest priority
2. Crawler fetches page (respects robots.txt, rate limits per domain)
3. Stores raw HTML to S3
4. Extracts all outbound links → adds to URL Frontier (if not seen)
5. Content Processor: parse HTML, extract text, detect language, compute content hash
6. Deduplication: if content hash already exists, skip indexing
7. Store processed document to Cassandra
8. Publish to Kafka → Indexer

**Crawl priority:**
- High: popular pages (high PageRank), recently updated pages, news sites
- Low: rarely linked pages, slow-updating content

**Politeness:** max 1 request/sec per domain; respect Crawl-Delay in robots.txt

### 5.2 Indexing

```
Kafka → Indexer
           ↓
    Tokenize text (lowercase, remove stopwords, stem)
           ↓
    For each term: update posting list in Inverted Index
    {term → [(docId, tf, positions)]}
           ↓
    Update document metadata (title, URL, PageRank)
```

1. Indexer consumes document from Kafka
2. Tokenizes text: lowercase, remove stopwords ("the", "a"), stem ("running" → "run")
3. Computes TF (term frequency) per term in document
4. Updates inverted index: for each term, append `(docId, tf, positions)` to posting list
5. Posting lists sorted by relevance score (TF-IDF × PageRank) for fast retrieval

### 5.3 Query Processing

```
User query → Query Service
                  ↓
    Spell correction + query expansion
                  ↓
    Tokenize query terms
                  ↓
    For each term: fetch posting list from Inverted Index (parallel)
                  ↓
    Intersect posting lists (AND) or union (OR)
                  ↓
    Ranking Service: score each candidate doc
                  ↓
    Top-10 results → Snippet Generator
                  ↓
    Return results
```

1. User submits query "python web scraping"
2. Spell correction: "pythn" → "python"
3. Tokenize: ["python", "web", "scraping"]
4. Fetch posting lists for each term in parallel from Inverted Index shards
5. Intersect: find docIds present in all posting lists (AND semantics)
6. Ranking: score each candidate:
   ```
   score = TF-IDF × PageRank × freshness × click-through rate
   ```
7. Top-10 by score → Snippet Generator extracts relevant text snippet
8. Return results with title, URL, snippet

### 5.4 PageRank Calculation

Offline batch job (runs weekly or on significant web graph changes):
1. Build web graph: nodes = pages, edges = links
2. Run PageRank algorithm (iterative):
   ```
   PR(A) = (1-d) + d × Σ(PR(B) / outlinks(B))  for all B linking to A
   d = 0.85 (damping factor)
   ```
3. Converges after ~50 iterations for web-scale graph
4. Store PageRank scores in Document Store
5. Indexer uses PageRank as ranking signal

### 5.5 Autocomplete

1. User types "pyt" → Autocomplete Service
2. Prefix search on trie or Elasticsearch prefix query
3. Returns top-10 suggestions ranked by query frequency
4. Query frequency updated from search logs (Kafka consumer)
5. Cached in Redis (TTL 1hr)

## 6. Key Interview Concepts

### Inverted Index
The core data structure. Maps each term to a list of documents containing it:
```
"python" → [doc1, doc5, doc23, ...]
"scraping" → [doc5, doc12, ...]
Query "python scraping" → intersect → [doc5, ...]
```
Posting lists are sorted by relevance score — top results retrieved without scanning entire list.

### TF-IDF Scoring
- TF (Term Frequency): how often term appears in document (more = more relevant)
- IDF (Inverse Document Frequency): how rare the term is across all documents (rarer = more distinctive)
- TF-IDF = TF × IDF — balances frequency with distinctiveness
- "the" has high TF but low IDF (appears everywhere) → low score
- "photosynthesis" has moderate TF but high IDF → high score for relevant docs

### PageRank
Measures page importance based on link structure. A page linked by many important pages has high PageRank. Iterative algorithm — converges to stable scores. Used as a global quality signal independent of query.

### Crawl Freshness vs Efficiency
Popular pages (news, social media) change frequently — crawl every hour. Static pages (Wikipedia articles) change rarely — crawl weekly. URL Frontier assigns crawl priority based on historical change frequency and PageRank.

### Index Sharding
Inverted index is too large for one machine (500TB). Shard by term hash: all posting lists for terms starting with "a-f" on shard 1, "g-m" on shard 2, etc. Query fans out to all shards in parallel, results merged and ranked.

### Deduplication
Web has massive duplicate content (mirrors, scrapers). Content hash (MD5/SHA256 of extracted text) detects exact duplicates. Near-duplicate detection: SimHash — if two pages have similar SimHash, they're near-duplicates. Only index one canonical version.

## 7. Failure Scenarios

### Crawler Blocked by Website
- Detection: HTTP 429 or connection refused
- Recovery: back off, retry after delay; respect Crawl-Delay; mark domain as rate-limited
- Prevention: distributed crawlers with different IPs; respect robots.txt

### Index Shard Failure
- Impact: queries for terms on that shard return no results
- Recovery: replica shard serves reads; failed shard rebuilt from document store
- Prevention: each shard replicated to 3 nodes; reads from any replica

### Stale Index (Crawl Lag)
- Impact: search results show outdated content
- Recovery: prioritize recrawl of high-PageRank pages; real-time indexing for news/social
- Prevention: tiered crawl frequency based on page importance and change rate

### Query Spike (Trending Topic)
- Detection: query rate spikes for specific terms
- Recovery: query cache in Redis absorbs repeated identical queries; auto-scale Query Service
- Prevention: aggressive caching for trending queries; pre-warm cache for anticipated events
