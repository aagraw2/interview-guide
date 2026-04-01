## 1. What is a Graph Database?

A graph database stores data as nodes (entities) and edges (relationships), optimized for traversing connections between data.

```
Relational Database:
  users table: {id, name}
  friendships table: {user_id, friend_id}
  
  Query: Find friends of friends
  SELECT * FROM users WHERE id IN (
    SELECT friend_id FROM friendships WHERE user_id IN (
      SELECT friend_id FROM friendships WHERE user_id = 1
    )
  )
  Multiple joins, slow

Graph Database:
  Nodes: User(id, name)
  Edges: FRIENDS_WITH
  
  Query: Find friends of friends
  MATCH (user:User {id: 1})-[:FRIENDS_WITH*2]-(friend)
  RETURN friend
  
  Direct traversal, fast
```

**Key principle:** Relationships are first-class citizens, not foreign keys.

---

## 2. Core Concepts

### Node

An entity with properties.

```
Node: User
  Properties:
    id: 1
    name: "Alice"
    email: "alice@example.com"
    age: 30

Node: Post
  Properties:
    id: 101
    title: "Hello World"
    content: "..."
    created_at: "2024-01-01"

Nodes are like rows in a table
```

### Edge (Relationship)

A connection between nodes with optional properties.

```
Edge: FRIENDS_WITH
  From: User(id: 1)
  To: User(id: 2)
  Properties:
    since: "2020-01-01"
    strength: 0.8

Edge: POSTED
  From: User(id: 1)
  To: Post(id: 101)
  Properties:
    timestamp: "2024-01-01T10:00:00Z"

Edges are directed (have direction)
Can have properties (metadata)
```

### Label

A type or category for nodes.

```
Labels:
  User, Post, Comment, Product, Order

Node with multiple labels:
  Node: (id: 1)
  Labels: User, Admin
  Properties: {name: "Alice"}

Labels are like table names
```

### Property

Key-value pair on nodes or edges.

```
Node properties:
  User: {id, name, email, age}
  
Edge properties:
  FRIENDS_WITH: {since, strength}

Properties are like columns
```

---

## 3. Graph Database vs Relational Database

### Relational Database

```
Schema:
  users: {id, name}
  posts: {id, user_id, title}
  comments: {id, post_id, user_id, content}
  likes: {user_id, post_id}

Query: Find users who liked posts by Alice's friends
  SELECT DISTINCT u.name
  FROM users u
  JOIN likes l ON u.id = l.user_id
  JOIN posts p ON l.post_id = p.id
  JOIN users author ON p.user_id = author.id
  JOIN friendships f ON author.id = f.friend_id
  WHERE f.user_id = (SELECT id FROM users WHERE name = 'Alice')

Multiple joins, slow for deep relationships
```

### Graph Database

```
Schema:
  (User)-[:FRIENDS_WITH]->(User)
  (User)-[:POSTED]->(Post)
  (User)-[:LIKED]->(Post)

Query: Find users who liked posts by Alice's friends
  MATCH (alice:User {name: 'Alice'})-[:FRIENDS_WITH]->(friend)
        -[:POSTED]->(post)<-[:LIKED]-(liker)
  RETURN DISTINCT liker.name

Direct traversal, fast for deep relationships
```

---

## 4. Query Language (Cypher)

Cypher is the query language for Neo4j (most popular graph database).

### Create Nodes

```cypher
// Create user
CREATE (u:User {id: 1, name: 'Alice', email: 'alice@example.com'})

// Create multiple nodes
CREATE (u1:User {id: 1, name: 'Alice'})
CREATE (u2:User {id: 2, name: 'Bob'})
CREATE (p:Post {id: 101, title: 'Hello World'})
```

### Create Relationships

```cypher
// Create friendship
MATCH (u1:User {id: 1}), (u2:User {id: 2})
CREATE (u1)-[:FRIENDS_WITH {since: '2020-01-01'}]->(u2)

// Create post relationship
MATCH (u:User {id: 1}), (p:Post {id: 101})
CREATE (u)-[:POSTED {timestamp: '2024-01-01'}]->(p)
```

### Query Patterns

```cypher
// Find Alice's friends
MATCH (alice:User {name: 'Alice'})-[:FRIENDS_WITH]->(friend)
RETURN friend.name

// Find friends of friends
MATCH (alice:User {name: 'Alice'})-[:FRIENDS_WITH*2]-(fof)
RETURN DISTINCT fof.name

// Find shortest path
MATCH path = shortestPath(
  (alice:User {name: 'Alice'})-[:FRIENDS_WITH*]-(bob:User {name: 'Bob'})
)
RETURN path

// Find mutual friends
MATCH (alice:User {name: 'Alice'})-[:FRIENDS_WITH]->(mutual)
      <-[:FRIENDS_WITH]-(bob:User {name: 'Bob'})
RETURN mutual.name
```

### Aggregation

```cypher
// Count friends
MATCH (alice:User {name: 'Alice'})-[:FRIENDS_WITH]->(friend)
RETURN COUNT(friend) AS friend_count

// Find most popular posts
MATCH (p:Post)<-[:LIKED]-(u:User)
RETURN p.title, COUNT(u) AS likes
ORDER BY likes DESC
LIMIT 10
```

---

## 5. Use Cases

### Social Networks

```
Nodes: User, Post, Comment
Edges: FRIENDS_WITH, FOLLOWS, POSTED, COMMENTED, LIKED

Queries:
  - Find friends of friends
  - Recommend friends (mutual friends)
  - Find influencers (most followers)
  - Detect communities (clustering)

Example:
  MATCH (user:User {id: 1})-[:FRIENDS_WITH]->(friend)
        -[:FRIENDS_WITH]->(fof)
  WHERE NOT (user)-[:FRIENDS_WITH]->(fof) AND user <> fof
  RETURN fof.name, COUNT(*) AS mutual_friends
  ORDER BY mutual_friends DESC
  LIMIT 10
```

### Recommendation Engines

```
Nodes: User, Product, Category
Edges: PURCHASED, VIEWED, RATED, BELONGS_TO

Queries:
  - Recommend products (collaborative filtering)
  - Find similar products
  - Personalized recommendations

Example:
  // Users who bought this also bought
  MATCH (p:Product {id: 123})<-[:PURCHASED]-(u:User)
        -[:PURCHASED]->(rec:Product)
  WHERE rec <> p
  RETURN rec.name, COUNT(*) AS score
  ORDER BY score DESC
  LIMIT 10
```

### Fraud Detection

```
Nodes: User, Account, Transaction, Device, IP
Edges: OWNS, SENT, RECEIVED, USED, FROM

Queries:
  - Find suspicious patterns (rings, cycles)
  - Detect money laundering (complex paths)
  - Identify fake accounts (shared devices/IPs)

Example:
  // Find accounts sharing devices
  MATCH (a1:Account)-[:USED]->(d:Device)<-[:USED]-(a2:Account)
  WHERE a1 <> a2
  RETURN a1.id, a2.id, d.id
```

### Knowledge Graphs

```
Nodes: Entity, Concept, Document
Edges: RELATED_TO, IS_A, PART_OF, MENTIONS

Queries:
  - Find related concepts
  - Semantic search
  - Entity extraction

Example:
  // Find related topics
  MATCH (topic:Topic {name: 'Machine Learning'})
        -[:RELATED_TO*1..2]-(related:Topic)
  RETURN related.name, related.description
```

---

## 6. Implementation Example (Neo4j)

### Setup

```python
from neo4j import GraphDatabase

class GraphDB:
    def __init__(self, uri, user, password):
        self.driver = GraphDatabase.driver(uri, auth=(user, password))
    
    def close(self):
        self.driver.close()
    
    def execute(self, query, parameters=None):
        with self.driver.session() as session:
            result = session.run(query, parameters)
            return [record.data() for record in result]

db = GraphDB("bolt://localhost:7687", "neo4j", "password")
```

### Create Data

```python
# Create users
db.execute("""
    CREATE (u1:User {id: 1, name: 'Alice'})
    CREATE (u2:User {id: 2, name: 'Bob'})
    CREATE (u3:User {id: 3, name: 'Charlie'})
""")

# Create friendships
db.execute("""
    MATCH (u1:User {id: 1}), (u2:User {id: 2})
    CREATE (u1)-[:FRIENDS_WITH {since: '2020-01-01'}]->(u2)
""")

db.execute("""
    MATCH (u2:User {id: 2}), (u3:User {id: 3})
    CREATE (u2)-[:FRIENDS_WITH {since: '2021-01-01'}]->(u3)
""")
```

### Query Data

```python
# Find Alice's friends
friends = db.execute("""
    MATCH (alice:User {name: 'Alice'})-[:FRIENDS_WITH]->(friend)
    RETURN friend.name AS name
""")
print(friends)  # [{'name': 'Bob'}]

# Find friends of friends
fof = db.execute("""
    MATCH (alice:User {name: 'Alice'})-[:FRIENDS_WITH*2]-(fof)
    WHERE fof <> alice
    RETURN DISTINCT fof.name AS name
""")
print(fof)  # [{'name': 'Charlie'}]

# Find shortest path
path = db.execute("""
    MATCH path = shortestPath(
        (alice:User {name: 'Alice'})-[:FRIENDS_WITH*]-(charlie:User {name: 'Charlie'})
    )
    RETURN [node IN nodes(path) | node.name] AS path
""")
print(path)  # [{'path': ['Alice', 'Bob', 'Charlie']}]
```

---

## 7. Performance Considerations

### Indexing

```cypher
// Create index on User.id
CREATE INDEX user_id_index FOR (u:User) ON (u.id)

// Create index on User.name
CREATE INDEX user_name_index FOR (u:User) ON (u.name)

// Create composite index
CREATE INDEX user_email_age_index FOR (u:User) ON (u.email, u.age)

Indexes speed up node lookups
Use for frequently queried properties
```

### Query Optimization

```cypher
// Bad: Scan all users
MATCH (u:User)
WHERE u.name = 'Alice'
RETURN u

// Good: Use index
MATCH (u:User {name: 'Alice'})
RETURN u

// Bad: Cartesian product
MATCH (u:User), (p:Post)
WHERE u.id = p.user_id
RETURN u, p

// Good: Direct relationship
MATCH (u:User)-[:POSTED]->(p:Post)
RETURN u, p
```

### Limiting Traversal Depth

```cypher
// Bad: Unbounded traversal (can be slow)
MATCH (alice:User {name: 'Alice'})-[:FRIENDS_WITH*]-(friend)
RETURN friend

// Good: Limit depth
MATCH (alice:User {name: 'Alice'})-[:FRIENDS_WITH*1..3]-(friend)
RETURN friend

// Good: Limit results
MATCH (alice:User {name: 'Alice'})-[:FRIENDS_WITH*1..3]-(friend)
RETURN friend
LIMIT 100
```

---

## 8. Graph Database Options

### Neo4j

```
Most popular graph database
Cypher query language
ACID transactions
Clustering support
Good for: General-purpose graph queries

Pros:
  ✅ Mature, well-documented
  ✅ Rich query language (Cypher)
  ✅ Good tooling
  
Cons:
  ❌ Expensive (enterprise features)
  ❌ JVM-based (memory overhead)
```

### Amazon Neptune

```
Managed graph database (AWS)
Supports Gremlin and SPARQL
Serverless option available
Good for: AWS-native applications

Pros:
  ✅ Managed (no ops)
  ✅ Scales automatically
  ✅ Integrates with AWS
  
Cons:
  ❌ Vendor lock-in
  ❌ Less mature than Neo4j
```

### ArangoDB

```
Multi-model database (graph, document, key-value)
AQL query language
Good for: Mixed workloads

Pros:
  ✅ Multi-model (flexible)
  ✅ Good performance
  
Cons:
  ❌ Less popular (smaller community)
```

---

## 9. When to Use Graph Databases

### Good fit

```
✅ Relationship-heavy data (social networks)
✅ Deep traversals (friends of friends)
✅ Pattern matching (fraud detection)
✅ Recommendation engines
✅ Knowledge graphs

Examples:
  - Social networks (Facebook, LinkedIn)
  - Fraud detection (PayPal)
  - Recommendations (Amazon, Netflix)
  - Knowledge graphs (Google)
```

### Poor fit

```
❌ Simple CRUD operations
❌ Aggregations on large datasets
❌ Time-series data
❌ Document storage

Examples:
  - Analytics dashboards (use data warehouse)
  - Logs and metrics (use time-series DB)
  - Content management (use document DB)
```

---

## 10. Common Interview Questions + Answers

### Q: What is a graph database and when would you use it?

> "A graph database stores data as nodes and edges, where relationships are first-class citizens. Unlike relational databases that use foreign keys and joins, graph databases allow direct traversal of relationships, making them much faster for queries involving multiple hops. You'd use a graph database for relationship-heavy data like social networks, recommendation engines, or fraud detection where you need to traverse connections efficiently."

### Q: How does a graph database differ from a relational database?

> "In a relational database, relationships are represented by foreign keys and require joins to traverse. In a graph database, relationships are edges that can be traversed directly without joins. This makes graph databases much faster for queries involving multiple levels of relationships, like finding friends of friends. However, relational databases are better for simple CRUD operations and aggregations. Graph databases excel when the relationships between data are as important as the data itself."

### Q: What are some use cases for graph databases?

> "Social networks for finding friends of friends and recommending connections. Fraud detection for identifying suspicious patterns like accounts sharing devices or circular money transfers. Recommendation engines for collaborative filtering — users who bought this also bought that. Knowledge graphs for semantic search and entity relationships. Network topology for understanding infrastructure dependencies. Any domain where you need to traverse complex relationships efficiently."

### Q: How do you optimize graph database queries?

> "First, create indexes on frequently queried node properties. Second, limit traversal depth to avoid scanning the entire graph. Third, use specific relationship types and directions to narrow the search space. Fourth, use LIMIT to cap result sizes. Fifth, avoid Cartesian products by using direct relationships instead of WHERE clauses. Finally, use EXPLAIN to analyze query plans and identify bottlenecks."

---

## 11. Quick Reference

```
What is a Graph Database?
  Stores data as nodes (entities) and edges (relationships)
  Optimized for traversing connections
  Relationships are first-class citizens

Core concepts:
  Node: Entity with properties (User, Post)
  Edge: Relationship with properties (FRIENDS_WITH)
  Label: Type/category for nodes (User, Admin)
  Property: Key-value pair on nodes/edges

Query language (Cypher):
  MATCH: Find patterns
  CREATE: Create nodes/edges
  WHERE: Filter results
  RETURN: Return results

Use cases:
  - Social networks (friends, followers)
  - Recommendation engines (collaborative filtering)
  - Fraud detection (suspicious patterns)
  - Knowledge graphs (entity relationships)

Graph DB vs Relational DB:
  Graph: Fast for deep traversals, relationship-heavy
  Relational: Fast for simple queries, aggregations

Popular options:
  - Neo4j (most popular, Cypher)
  - Amazon Neptune (managed, AWS)
  - ArangoDB (multi-model)

When to use:
  ✅ Relationship-heavy data
  ✅ Deep traversals
  ✅ Pattern matching
  ✅ Recommendations
  
  ❌ Simple CRUD
  ❌ Aggregations
  ❌ Time-series data

Performance:
  - Create indexes on frequently queried properties
  - Limit traversal depth
  - Use specific relationship types
  - Avoid Cartesian products
  - Use LIMIT to cap results

Best practices:
  - Model relationships explicitly
  - Use meaningful relationship types
  - Index frequently queried properties
  - Limit traversal depth
  - Monitor query performance
```
