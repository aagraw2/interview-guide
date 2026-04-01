## 1. What is a Vector Database?

A vector database stores and queries high-dimensional vectors (embeddings) for similarity search, commonly used in AI/ML applications.

```
Traditional Database:
  Store: {id: 1, name: "cat", category: "animal"}
  Query: WHERE category = "animal"
  Exact match

Vector Database:
  Store: {id: 1, vector: [0.2, 0.8, 0.1, ...], metadata: {name: "cat"}}
  Query: Find similar to [0.3, 0.7, 0.2, ...]
  Similarity search (semantic)
```

**Key principle:** Find similar items based on vector distance, not exact matches.

---

## 2. Core Concepts

### Vector (Embedding)

A numerical representation of data in high-dimensional space.

```
Text embedding (OpenAI):
  "cat" → [0.2, 0.8, 0.1, 0.4, ..., 0.3]  (1536 dimensions)
  "dog" → [0.3, 0.7, 0.2, 0.5, ..., 0.4]  (1536 dimensions)
  
Image embedding (ResNet):
  cat.jpg → [0.1, 0.9, 0.2, ..., 0.5]  (2048 dimensions)
  
Similar items have similar vectors
```

### Similarity Search

Find vectors closest to a query vector.

```
Query: "kitten" → [0.25, 0.75, 0.15, ...]

Results (by distance):
  1. "cat" → [0.2, 0.8, 0.1, ...]  (distance: 0.05)
  2. "dog" → [0.3, 0.7, 0.2, ...]  (distance: 0.12)
  3. "bird" → [0.5, 0.3, 0.4, ...]  (distance: 0.45)

Closer vectors = more similar
```

### Distance Metrics

Measure similarity between vectors.

```
Cosine Similarity:
  Measures angle between vectors
  Range: -1 (opposite) to 1 (identical)
  Good for: Text embeddings
  
  cos(A, B) = (A · B) / (||A|| * ||B||)

Euclidean Distance:
  Straight-line distance
  Range: 0 (identical) to ∞
  Good for: Image embeddings
  
  dist(A, B) = sqrt(Σ(Ai - Bi)²)

Dot Product:
  Inner product of vectors
  Range: -∞ to ∞
  Good for: Normalized vectors
  
  dot(A, B) = Σ(Ai * Bi)
```

### Approximate Nearest Neighbor (ANN)

Fast approximate search instead of exact search.

```
Exact search:
  Compare query to all vectors
  O(n) time
  Slow for large datasets

ANN search:
  Use index structure (HNSW, IVF)
  O(log n) time
  Fast, slightly less accurate

Trade-off: Speed vs accuracy
```

---

## 3. How It Works

### Indexing

```
1. Generate embeddings
   text → embedding model → vector

2. Build index
   vectors → ANN index (HNSW, IVF)

3. Store in vector database
   {id, vector, metadata}

Example:
  documents = ["cat", "dog", "bird"]
  embeddings = model.encode(documents)
  
  for doc, emb in zip(documents, embeddings):
      db.insert(id=doc, vector=emb, metadata={"text": doc})
```

### Querying

```
1. Generate query embedding
   query → embedding model → query_vector

2. Search index
   query_vector → ANN search → top_k results

3. Return results with metadata

Example:
  query = "kitten"
  query_vector = model.encode(query)
  
  results = db.search(
      vector=query_vector,
      top_k=10,
      metric="cosine"
  )
  
  for result in results:
      print(result.metadata["text"], result.score)
```

---

## 4. Use Cases

### Semantic Search

```
Traditional search:
  Query: "cat"
  Results: Documents containing "cat" (keyword match)

Semantic search:
  Query: "cat"
  Results: Documents about cats, kittens, felines (meaning match)

Example:
  Query: "how to train a puppy"
  Results:
    - "dog training tips"
    - "puppy obedience guide"
    - "teaching your new dog"
  
  No exact keyword match, but semantically similar
```

### Recommendation Systems

```
Find similar items:
  User likes: "The Matrix"
  Embedding: [0.2, 0.8, 0.1, ...]
  
  Similar movies:
    - "Inception" (sci-fi, mind-bending)
    - "Blade Runner" (dystopian, AI)
    - "The Terminator" (AI, action)

Content-based filtering using embeddings
```

### RAG (Retrieval-Augmented Generation)

```
1. User asks question
2. Generate query embedding
3. Search vector database for relevant documents
4. Pass documents to LLM as context
5. LLM generates answer

Example:
  Question: "What is the refund policy?"
  
  Vector search:
    - "Refund Policy" document
    - "Returns and Exchanges" document
  
  LLM: "Based on the refund policy, you can return items within 30 days..."
```

### Image Search

```
Query by image:
  Upload image → generate embedding → find similar images

Example:
  Upload: cat.jpg
  Results:
    - cat2.jpg (similar cat)
    - kitten.jpg (similar animal)
    - dog.jpg (similar pet)

Reverse image search
```

---

## 5. Implementation Example (Pinecone)

### Setup

```python
import pinecone
from sentence_transformers import SentenceTransformer

# Initialize Pinecone
pinecone.init(api_key="your-api-key", environment="us-west1-gcp")

# Create index
pinecone.create_index(
    name="documents",
    dimension=384,  # Embedding dimension
    metric="cosine"
)

index = pinecone.Index("documents")

# Load embedding model
model = SentenceTransformer('all-MiniLM-L6-v2')
```

### Insert Data

```python
documents = [
    {"id": "1", "text": "The cat sat on the mat"},
    {"id": "2", "text": "The dog played in the park"},
    {"id": "3", "text": "Birds fly in the sky"}
]

# Generate embeddings
embeddings = model.encode([doc["text"] for doc in documents])

# Insert into Pinecone
vectors = [
    (doc["id"], emb.tolist(), {"text": doc["text"]})
    for doc, emb in zip(documents, embeddings)
]

index.upsert(vectors=vectors)
```

### Query Data

```python
# Generate query embedding
query = "kitten on a rug"
query_vector = model.encode(query).tolist()

# Search
results = index.query(
    vector=query_vector,
    top_k=3,
    include_metadata=True
)

# Print results
for match in results['matches']:
    print(f"Score: {match['score']:.4f}")
    print(f"Text: {match['metadata']['text']}")
    print()
```

---

## 6. Vector Database Options

### Pinecone

```
Managed vector database
Serverless, auto-scaling
Good for: Production applications

Pros:
  ✅ Fully managed (no ops)
  ✅ Fast (optimized for ANN)
  ✅ Easy to use
  
Cons:
  ❌ Expensive
  ❌ Vendor lock-in
```

### Weaviate

```
Open-source vector database
Supports multiple embedding models
Good for: Self-hosted, flexible

Pros:
  ✅ Open-source
  ✅ Flexible (multiple models)
  ✅ GraphQL API
  
Cons:
  ❌ Self-hosted (ops overhead)
  ❌ Less mature than Pinecone
```

### Milvus

```
Open-source, distributed
High performance
Good for: Large-scale deployments

Pros:
  ✅ Open-source
  ✅ Scalable (distributed)
  ✅ High performance
  
Cons:
  ❌ Complex setup
  ❌ Steep learning curve
```

### Qdrant

```
Open-source, Rust-based
Fast and efficient
Good for: Performance-critical applications

Pros:
  ✅ Fast (Rust)
  ✅ Easy to use
  ✅ Good documentation
  
Cons:
  ❌ Smaller community
```

### pgvector (PostgreSQL Extension)

```
Vector extension for PostgreSQL
Good for: Existing PostgreSQL users

Pros:
  ✅ Use existing PostgreSQL
  ✅ ACID transactions
  ✅ Familiar SQL
  
Cons:
  ❌ Slower than specialized vector DBs
  ❌ Limited scalability
```

---

## 7. Indexing Algorithms

### HNSW (Hierarchical Navigable Small World)

```
Graph-based index
Fast search, high accuracy
Most popular algorithm

How it works:
  - Build multi-layer graph
  - Each layer is a skip list
  - Navigate from top to bottom
  
Pros:
  ✅ Fast search
  ✅ High accuracy
  ✅ Good for high-dimensional data
  
Cons:
  ❌ Slow indexing
  ❌ High memory usage
```

### IVF (Inverted File Index)

```
Clustering-based index
Partition vectors into clusters
Search only relevant clusters

How it works:
  - Cluster vectors (k-means)
  - Query finds nearest clusters
  - Search within clusters
  
Pros:
  ✅ Fast indexing
  ✅ Low memory usage
  
Cons:
  ❌ Lower accuracy
  ❌ Requires tuning (number of clusters)
```

### LSH (Locality-Sensitive Hashing)

```
Hash-based index
Similar vectors hash to same bucket

How it works:
  - Hash vectors to buckets
  - Query hashes to bucket
  - Search within bucket
  
Pros:
  ✅ Very fast
  ✅ Low memory
  
Cons:
  ❌ Lower accuracy
  ❌ Requires tuning (hash functions)
```

---

## 8. Performance Considerations

### Embedding Dimension

```
Higher dimension:
  ✅ More information
  ✅ Better accuracy
  ❌ Slower search
  ❌ More storage

Lower dimension:
  ✅ Faster search
  ✅ Less storage
  ❌ Less information
  ❌ Lower accuracy

Common dimensions:
  - OpenAI: 1536
  - Sentence-BERT: 384, 768
  - ResNet: 2048
```

### Index Parameters

```
HNSW parameters:
  - M: Number of connections per node (higher = more accurate, slower)
  - ef_construction: Search depth during indexing (higher = better index)
  - ef_search: Search depth during query (higher = more accurate, slower)

Example:
  index = HNSWIndex(
      dimension=384,
      M=16,  # Default: 16
      ef_construction=200,  # Default: 200
      ef_search=50  # Default: 50
  )
```

### Filtering

```
Metadata filtering:
  Query: Find similar documents where category = "tech"
  
  1. Filter by metadata (category = "tech")
  2. Search within filtered vectors
  
  Can be slow if filter is selective
  
Solution:
  - Pre-filter (filter before search)
  - Post-filter (search then filter)
  - Hybrid (filter + search together)
```

---

## 9. Challenges

### Cold start

```
Problem:
  New items have no embeddings
  Can't search until embedded

Solution:
  - Batch embedding generation
  - Async embedding (queue)
  - Pre-compute embeddings
```

### Embedding drift

```
Problem:
  Embedding model updated
  Old embeddings incompatible with new model

Solution:
  - Re-embed all data (expensive)
  - Versioned embeddings
  - Gradual migration
```

### Cost

```
Problem:
  Embedding generation is expensive (API calls)
  Vector storage is expensive (high-dimensional)

Solution:
  - Cache embeddings
  - Use smaller models (lower dimension)
  - Batch processing
```

---

## 10. Common Interview Questions + Answers

### Q: What is a vector database and when would you use it?

> "A vector database stores high-dimensional vectors (embeddings) and enables fast similarity search. Unlike traditional databases that do exact matches, vector databases find semantically similar items based on vector distance. You'd use it for semantic search where you want to find documents by meaning not keywords, recommendation systems to find similar items, or RAG applications where you need to retrieve relevant context for LLMs."

### Q: How does similarity search work in vector databases?

> "First, you generate embeddings for your data using a model like OpenAI or Sentence-BERT. These embeddings are stored in the vector database with an index structure like HNSW. When querying, you generate an embedding for the query and use approximate nearest neighbor search to find the closest vectors. The database returns the top-k most similar items based on a distance metric like cosine similarity. It's approximate rather than exact to achieve fast search on large datasets."

### Q: What's the difference between exact and approximate nearest neighbor search?

> "Exact search compares the query vector to every vector in the database, which is O(n) and slow for large datasets. Approximate nearest neighbor search uses index structures like HNSW or IVF to narrow the search space, achieving O(log n) time. It trades a small amount of accuracy for much faster search. In practice, ANN is accurate enough for most applications and necessary for real-time search on millions of vectors."

### Q: How do you choose between different vector databases?

> "Consider your requirements: For production with minimal ops, use managed services like Pinecone. For self-hosted with flexibility, use Weaviate or Qdrant. For large-scale distributed deployments, use Milvus. If you already use PostgreSQL and have moderate scale, pgvector is convenient. Evaluate based on performance needs, scale, budget, and whether you want managed or self-hosted. Also consider the indexing algorithm — HNSW is generally best for accuracy, IVF for speed."

---

## 11. Quick Reference

```
What is a Vector Database?
  Stores high-dimensional vectors (embeddings)
  Enables fast similarity search
  Used for semantic search, recommendations, RAG

Core concepts:
  Vector: Numerical representation of data
  Similarity: Distance between vectors (cosine, euclidean)
  ANN: Approximate nearest neighbor search
  Embedding: Vector representation from ML model

How it works:
  1. Generate embeddings (text/image → vector)
  2. Store in vector database with index
  3. Query: Generate query embedding → ANN search → results

Use cases:
  - Semantic search (meaning-based)
  - Recommendations (similar items)
  - RAG (retrieve context for LLMs)
  - Image search (reverse image search)

Distance metrics:
  - Cosine: Angle between vectors (text)
  - Euclidean: Straight-line distance (images)
  - Dot product: Inner product (normalized)

Popular options:
  - Pinecone (managed, easy)
  - Weaviate (open-source, flexible)
  - Milvus (distributed, scalable)
  - Qdrant (fast, Rust)
  - pgvector (PostgreSQL extension)

Indexing algorithms:
  - HNSW: Fast, accurate, high memory
  - IVF: Fast indexing, lower accuracy
  - LSH: Very fast, lowest accuracy

Performance:
  - Higher dimension = more accurate, slower
  - HNSW parameters: M, ef_construction, ef_search
  - Filtering: Pre-filter vs post-filter

Challenges:
  - Cold start (new items need embeddings)
  - Embedding drift (model updates)
  - Cost (embedding generation, storage)

Best practices:
  - Cache embeddings
  - Batch processing
  - Monitor search latency
  - Tune index parameters
  - Use appropriate distance metric
```
