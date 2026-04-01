## 1. What is a Search Index?

A search index is a data structure optimized for full-text search queries. Unlike databases that match exact values, search indexes find documents containing specific words or phrases, rank results by relevance, and support fuzzy matching.

```
Examples:
  - Elasticsearch
  - Apache Solr
  - Amazon OpenSearch
  - Algolia
  - Meilisearch
```

**Core use case:** Product search, log analysis, document search, autocomplete, fuzzy matching.

---

## 2. Inverted Index

The fundamental data structure behind search engines.

### Forward index (database)

```
Document 1: "The quick brown fox"
Document 2: "The lazy dog"
Document 3: "Quick brown dogs"

Storage:
  doc_id → content
  1 → "The quick brown fox"
  2 → "The lazy dog"
  3 → "Quick brown dogs"

Query: Find documents containing "quick"
  → Must scan all documents (slow)
```

### Inverted index (search engine)

```
Storage:
  term → [doc_ids]
  quick → [1, 3]
  brown → [1, 3]
  fox   → [1]
  lazy  → [2]
  dog   → [2]
  dogs  → [3]

Query: Find documents containing "quick"
  → Lookup "quick" in index → [1, 3] (instant)
```

The inverted index maps terms to documents, enabling O(1) lookup instead of O(n) scan.

---

## 3. Text Analysis Pipeline

Before indexing, text goes through an analysis pipeline.

### Tokenization

Split text into individual terms (tokens).

```
Input: "The quick brown fox"
Tokens: ["The", "quick", "brown", "fox"]
```

### Lowercasing

Convert to lowercase for case-insensitive search.

```
Tokens: ["the", "quick", "brown", "fox"]
```

### Stop word removal

Remove common words that don't add meaning.

```
Stop words: ["the", "a", "an", "and", "or", "but"]
Tokens: ["quick", "brown", "fox"]
```

### Stemming / Lemmatization

Reduce words to their root form.

```
Stemming:
  running → run
  runs    → run
  ran     → ran (not perfect)

Lemmatization:
  running → run
  runs    → run
  ran     → run (uses dictionary)

Tokens: ["quick", "brown", "fox"]
```

### Full pipeline

```
Input: "The Quick Brown Foxes are Running"

1. Tokenize:    ["The", "Quick", "Brown", "Foxes", "are", "Running"]
2. Lowercase:   ["the", "quick", "brown", "foxes", "are", "running"]
3. Stop words:  ["quick", "brown", "foxes", "running"]
4. Stemming:    ["quick", "brown", "fox", "run"]

Indexed terms: ["quick", "brown", "fox", "run"]
```

Now a search for "running fox" matches this document (both stemmed to "run" and "fox").

---

## 4. Relevance Scoring

When multiple documents match a query, how do you rank them?

### TF-IDF (Term Frequency - Inverse Document Frequency)

```
TF (Term Frequency):
  How often does the term appear in this document?
  More occurrences → higher score

IDF (Inverse Document Frequency):
  How rare is the term across all documents?
  Rare terms → higher score
  Common terms (like "the") → lower score

Score = TF × IDF
```

### Example

```
Documents:
  Doc 1: "elasticsearch is a search engine"
  Doc 2: "elasticsearch is fast"
  Doc 3: "search engines are useful"

Query: "elasticsearch search"

Term: "elasticsearch"
  TF in Doc 1: 1/6 = 0.17
  TF in Doc 2: 1/4 = 0.25
  IDF: log(3/2) = 0.18  (appears in 2 of 3 docs)
  Score for Doc 1: 0.17 × 0.18 = 0.03
  Score for Doc 2: 0.25 × 0.18 = 0.045

Term: "search"
  TF in Doc 1: 1/6 = 0.17
  TF in Doc 3: 1/4 = 0.25
  IDF: log(3/2) = 0.18
  Score for Doc 1: 0.17 × 0.18 = 0.03
  Score for Doc 3: 0.25 × 0.18 = 0.045

Final scores:
  Doc 1: 0.03 + 0.03 = 0.06
  Doc 2: 0.045
  Doc 3: 0.045

Ranking: Doc 1 > Doc 2 = Doc 3
```

### BM25 (Best Match 25)

Modern improvement over TF-IDF. Used by Elasticsearch by default.

```
Improvements:
  - Diminishing returns for term frequency (10 occurrences not 10x better than 1)
  - Document length normalization (longer docs don't automatically rank higher)
  - Tunable parameters (k1, b)

BM25 is the industry standard for relevance scoring
```

---

## 5. Elasticsearch Architecture

### Cluster structure

```
Cluster
  ├── Node 1 (primary shard 0, replica shard 1)
  ├── Node 2 (primary shard 1, replica shard 0)
  └── Node 3 (primary shard 2, replica shard 2)

Index: "products"
  Shards: 3 primary shards
  Replicas: 1 replica per shard
```

### Sharding

Data is split across multiple shards for horizontal scaling.

```
Index: "products" (1 million documents)
  Shard 0: documents 0-333k
  Shard 1: documents 333k-666k
  Shard 2: documents 666k-1M

Query: "laptop"
  → Search all 3 shards in parallel
  → Merge results
  → Return top 10
```

### Replication

Each shard has replicas for fault tolerance and read scaling.

```
Primary shard 0 (Node 1) → Replica shard 0 (Node 2)
Primary shard 1 (Node 2) → Replica shard 1 (Node 3)
Primary shard 2 (Node 3) → Replica shard 2 (Node 1)

Write:
  → Write to primary shard
  → Replicate to replica shard
  → Acknowledge

Read:
  → Can read from primary OR replica
  → Load balance across replicas
```

---

## 6. Write Path

### Indexing a document

```
1. Client sends document to any node (coordinating node)

2. Coordinating node:
   - Determines which shard (hash(doc_id) % num_shards)
   - Routes to primary shard

3. Primary shard:
   - Writes to in-memory buffer
   - Appends to transaction log (durability)
   - Returns success

4. Primary shard replicates to replica shards

5. (Background) Refresh:
   - Every 1 second, flush buffer to segment (immutable file)
   - Segment is now searchable

6. (Background) Merge:
   - Merge small segments into larger segments
   - Delete old segments
```

### Near real-time search

Documents are searchable within 1 second (refresh interval).

```
Write at 10:00:00.000
Refresh at 10:00:01.000 → document is now searchable

Trade-off:
  Faster refresh → more searchable, more I/O
  Slower refresh → less I/O, longer delay
```

---

## 7. Query Types

### Match query

Full-text search with analysis.

```
Query: "running shoes"
  → Analyzed: ["run", "shoe"] (stemmed)
  → Matches documents containing "run" OR "shoe"

Example:
  Doc 1: "Running sneakers" → match (run)
  Doc 2: "Shoe store" → match (shoe)
  Doc 3: "Running shoe sale" → match (both, higher score)
```

### Term query

Exact match (no analysis).

```
Query: "status": "active"
  → Matches documents where status field is exactly "active"
  → Case-sensitive, no stemming
```

### Bool query

Combine multiple queries with AND, OR, NOT.

```
{
  "bool": {
    "must": [
      {"match": {"title": "laptop"}},
      {"range": {"price": {"lte": 1000}}}
    ],
    "must_not": [
      {"term": {"status": "out_of_stock"}}
    ]
  }
}

Matches: title contains "laptop" AND price <= 1000 AND status != "out_of_stock"
```

### Fuzzy query

Tolerate typos (Levenshtein distance).

```
Query: "laptpo" (typo)
  → Fuzzy match: "laptop" (1 character difference)

Levenshtein distance = 1 → match
```

### Prefix query

Match terms starting with a prefix.

```
Query: "elast"
  → Matches: "elastic", "elasticsearch", "elasticity"
```

### Wildcard query

Pattern matching with * and ?.

```
Query: "elast*"
  → Matches: "elastic", "elasticsearch"

Query: "elast?"
  → Matches: "elasti" (exactly 1 character)
```

---

## 8. Aggregations

Elasticsearch supports SQL-like aggregations for analytics.

### Bucket aggregations

Group documents into buckets.

```
{
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          {"to": 100},
          {"from": 100, "to": 500},
          {"from": 500}
        ]
      }
    }
  }
}

Result:
  0-100: 50 products
  100-500: 200 products
  500+: 30 products
```

### Metric aggregations

Calculate statistics.

```
{
  "aggs": {
    "avg_price": {"avg": {"field": "price"}},
    "max_price": {"max": {"field": "price"}},
    "total_sales": {"sum": {"field": "sales"}}
  }
}

Result:
  avg_price: 250
  max_price: 1500
  total_sales: 50000
```

### Nested aggregations

Combine bucket and metric aggregations.

```
{
  "aggs": {
    "categories": {
      "terms": {"field": "category"},
      "aggs": {
        "avg_price": {"avg": {"field": "price"}}
      }
    }
  }
}

Result:
  Electronics: avg_price = 500
  Clothing: avg_price = 50
  Books: avg_price = 20
```

---

## 9. Elasticsearch vs Database

|Feature|Elasticsearch|SQL Database|
|---|---|---|
|Primary use case|Full-text search, analytics|Transactional queries|
|Data structure|Inverted index|B-tree index|
|Query type|Fuzzy, relevance-ranked|Exact match|
|Consistency|Eventually consistent|Strongly consistent (ACID)|
|Joins|Limited (nested docs)|Full join support|
|Aggregations|Fast (pre-computed)|Slower (computed on query)|
|Write performance|High (batch indexing)|Moderate (ACID overhead)|
|Schema|Flexible (dynamic mapping)|Strict (predefined schema)|

**Use Elasticsearch when:** You need full-text search, fuzzy matching, relevance ranking, or fast aggregations.

**Use SQL database when:** You need ACID transactions, complex joins, or strong consistency.

---

## 10. Common Use Cases

### 1. Product search (e-commerce)

```
Query: "wireless headphones under $100"

Elasticsearch:
  - Full-text search: "wireless headphones"
  - Range filter: price < 100
  - Relevance ranking: BM25 score
  - Facets: brand, color, rating
```

### 2. Log analysis (ELK stack)

```
Elasticsearch + Logstash + Kibana

Logstash:
  - Collect logs from servers
  - Parse and enrich
  - Send to Elasticsearch

Elasticsearch:
  - Index logs
  - Search: "error" AND "payment service"
  - Aggregate: error count by service

Kibana:
  - Visualize logs
  - Dashboards and alerts
```

### 3. Autocomplete

```
Query: "elast"

Elasticsearch:
  - Prefix query: "elast*"
  - Suggestions: ["elastic", "elasticsearch", "elasticity"]
  - Ranked by popularity
```

### 4. Fuzzy search

```
Query: "iphone 13 pro max" (user typo: "iphon 13 pro max")

Elasticsearch:
  - Fuzzy match: "iphon" → "iphone" (1 edit distance)
  - Returns: iPhone 13 Pro Max
```

---

## 11. Common Interview Questions + Answers

### Q: What's the difference between a database index and a search index?

> "A database index (B-tree) is optimized for exact lookups and range queries on structured data. A search index (inverted index) is optimized for full-text search, fuzzy matching, and relevance ranking. The inverted index maps terms to documents, enabling fast keyword lookups. Databases can't efficiently handle queries like 'find documents containing words similar to laptop' — that's where search indexes excel."

### Q: How does Elasticsearch achieve near real-time search?

> "Elasticsearch writes documents to an in-memory buffer and a transaction log for durability. Every 1 second (refresh interval), the buffer is flushed to an immutable segment file that becomes searchable. This means documents are searchable within 1 second of being indexed. The trade-off is that more frequent refreshes increase I/O, so you can tune the refresh interval based on your latency requirements."

### Q: What's the difference between a match query and a term query?

> "A match query is for full-text search — it analyzes the query text (tokenization, lowercasing, stemming) and matches analyzed terms in the inverted index. A term query is for exact matching — it doesn't analyze the query and matches the exact value. Use match for text fields like product descriptions, and term for keyword fields like status or category."

### Q: How does Elasticsearch rank search results?

> "Elasticsearch uses BM25 by default, which is an improvement over TF-IDF. It scores documents based on term frequency (how often the term appears in the document) and inverse document frequency (how rare the term is across all documents). BM25 adds diminishing returns for term frequency and document length normalization, so longer documents don't automatically rank higher. You can also boost specific fields or use custom scoring functions."

---

## 12. Interview Tricks & Pitfalls

### ✅ Trick 1: Explain the inverted index

The inverted index is the core data structure. Always mention it when discussing search engines — it shows you understand the fundamentals.

### ✅ Trick 2: Know the analysis pipeline

Tokenization, lowercasing, stop words, stemming — these are common interview topics. Mentioning them shows depth.

### ✅ Trick 3: Discuss eventual consistency

Elasticsearch is eventually consistent (1-second refresh interval). Mentioning this shows you understand the trade-offs vs SQL databases.

### ❌ Pitfall 1: Thinking Elasticsearch replaces databases

Elasticsearch is for search and analytics, not transactional workloads. You still need a database as the source of truth. Elasticsearch is typically used alongside a database.

### ❌ Pitfall 2: Confusing match and term queries

Match queries are analyzed (full-text search), term queries are not (exact match). Don't mix them up.

### ❌ Pitfall 3: Forgetting about sharding

Elasticsearch scales horizontally via sharding. Mentioning sharding shows you understand how it handles large datasets.

---

## 13. Quick Reference

```
What is a search index?
  Data structure optimized for full-text search
  Inverted index: term → [doc_ids]
  Examples: Elasticsearch, Solr, OpenSearch

Inverted index:
  Forward index: doc_id → content (database)
  Inverted index: term → [doc_ids] (search engine)
  Enables O(1) term lookup instead of O(n) scan

Analysis pipeline:
  1. Tokenization: Split text into terms
  2. Lowercasing: Case-insensitive search
  3. Stop words: Remove common words
  4. Stemming: Reduce to root form (running → run)

Relevance scoring:
  TF-IDF: Term frequency × Inverse document frequency
  BM25: Modern improvement (diminishing returns, length normalization)

Elasticsearch architecture:
  Cluster → Nodes → Indices → Shards → Segments
  Sharding: Horizontal scaling (split data across shards)
  Replication: Fault tolerance (replicas of each shard)

Write path:
  1. Write to in-memory buffer + transaction log
  2. Refresh (1 sec): Flush buffer to segment (searchable)
  3. Merge: Combine small segments into large segments

Query types:
  Match: Full-text search (analyzed)
  Term: Exact match (not analyzed)
  Bool: Combine with AND, OR, NOT
  Fuzzy: Tolerate typos (Levenshtein distance)
  Prefix: Match terms starting with prefix
  Wildcard: Pattern matching with * and ?

Aggregations:
  Bucket: Group documents (range, terms)
  Metric: Calculate statistics (avg, max, sum)
  Nested: Combine bucket + metric

Elasticsearch vs SQL:
  Elasticsearch → Full-text search, fuzzy matching, analytics
  SQL → ACID transactions, complex joins, strong consistency

Use cases:
  Product search (e-commerce)
  Log analysis (ELK stack)
  Autocomplete
  Fuzzy search

Near real-time:
  Documents searchable within 1 second (refresh interval)
  Trade-off: Faster refresh → more I/O

Eventual consistency:
  Elasticsearch is eventually consistent (not ACID)
  Use database as source of truth, Elasticsearch for search
```
