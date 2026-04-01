## 1. What Is an LRU Cache?

LRU (Least Recently Used) Cache is a data structure that stores a limited number of items and evicts the least recently used item when capacity is reached.

```
Cache capacity: 3

Operations:
  put(1, "a")  → Cache: [1]
  put(2, "b")  → Cache: [1, 2]
  put(3, "c")  → Cache: [1, 2, 3]
  get(1)       → Cache: [2, 3, 1]  (1 moved to end, most recent)
  put(4, "d")  → Cache: [3, 1, 4]  (2 evicted, least recent)
```

**Key property:** O(1) time complexity for both get and put operations.

---

## 2. Why LRU?

### Temporal locality

```
Recently accessed items are likely to be accessed again soon

Example: Web browser cache
  - User visits homepage → cache it
  - User likely to visit homepage again soon
  - Keep recently visited pages in cache
```

---

### Cache eviction policies comparison

```
FIFO (First In First Out):
  Evict oldest item (by insertion time)
  Ignores access patterns
  Simple but not optimal

LRU (Least Recently Used):
  Evict least recently accessed item
  Considers access patterns
  Better hit rate than FIFO

LFU (Least Frequently Used):
  Evict least frequently accessed item
  Tracks access count
  Good for long-term patterns, but complex
```

**LRU is the sweet spot:** Better than FIFO, simpler than LFU.

---

## 3. Implementation

### Data structures needed

```
1. Hash Map: O(1) lookup by key
2. Doubly Linked List: O(1) insertion/deletion, maintain order

Why doubly linked list?
  - Need to move nodes to end (most recent)
  - Need to remove nodes from middle
  - Need to remove from head (least recent)
  - All O(1) with doubly linked list
```

---

### Structure

```
Hash Map:
  key → Node (in linked list)

Doubly Linked List:
  [Head] ↔ [Node 1] ↔ [Node 2] ↔ [Node 3] ↔ [Tail]
   ↑                                          ↑
  Least recent                           Most recent

Node:
  key: int
  value: int
  prev: Node
  next: Node
```

---

### Python implementation

```python
class Node:
    def __init__(self, key, value):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}  # key -> Node
        
        # Dummy head and tail for easier operations
        self.head = Node(0, 0)
        self.tail = Node(0, 0)
        self.head.next = self.tail
        self.tail.prev = self.head
    
    def get(self, key):
        if key not in self.cache:
            return -1
        
        # Move to end (most recent)
        node = self.cache[key]
        self._remove(node)
        self._add(node)
        
        return node.value
    
    def put(self, key, value):
        if key in self.cache:
            # Update existing
            self._remove(self.cache[key])
        
        # Add new node
        node = Node(key, value)
        self._add(node)
        self.cache[key] = node
        
        # Check capacity
        if len(self.cache) > self.capacity:
            # Remove least recent (head.next)
            lru = self.head.next
            self._remove(lru)
            del self.cache[lru.key]
    
    def _remove(self, node):
        """Remove node from linked list"""
        prev_node = node.prev
        next_node = node.next
        prev_node.next = next_node
        next_node.prev = prev_node
    
    def _add(self, node):
        """Add node to end (most recent)"""
        prev_node = self.tail.prev
        prev_node.next = node
        node.prev = prev_node
        node.next = self.tail
        self.tail.prev = node
```

---

### Time complexity

```
get(key):  O(1)
  - Hash map lookup: O(1)
  - Remove from list: O(1)
  - Add to end: O(1)

put(key, value):  O(1)
  - Hash map lookup: O(1)
  - Remove from list: O(1)
  - Add to end: O(1)
  - Evict LRU: O(1)

Space complexity: O(capacity)
```

---

## 4. Operations Walkthrough

### Example: Capacity = 3

```
Initial: []

put(1, "a"):
  Cache: {1: "a"}
  List: [1]

put(2, "b"):
  Cache: {1: "a", 2: "b"}
  List: [1, 2]

put(3, "c"):
  Cache: {1: "a", 2: "b", 3: "c"}
  List: [1, 2, 3]

get(1):
  Return "a"
  Move 1 to end (most recent)
  List: [2, 3, 1]

put(4, "d"):
  Capacity exceeded
  Evict 2 (least recent)
  Cache: {1: "a", 3: "c", 4: "d"}
  List: [3, 1, 4]

get(3):
  Return "c"
  Move 3 to end
  List: [1, 4, 3]

put(5, "e"):
  Evict 1 (least recent)
  Cache: {3: "c", 4: "d", 5: "e"}
  List: [4, 3, 5]
```

---

## 5. Variants

### LRU-K

Track last K accesses, evict based on Kth most recent access.

```
LRU-2:
  Track last 2 accesses
  Evict item with oldest 2nd-to-last access
  
  Better for: Filtering one-time accesses
  Example: Scan through data once → doesn't pollute cache
```

---

### 2Q (Two Queues)

```
Two queues:
  A1 (FIFO): First-time accesses
  Am (LRU): Frequently accessed items

New item → A1
If accessed again → move to Am
Evict from A1 first, then Am

Prevents: One-time accesses from evicting hot items
```

---

### ARC (Adaptive Replacement Cache)

```
Balances between:
  - Recently used items
  - Frequently used items

Adapts based on workload
More complex, but better hit rate
```

---

## 6. Real-World Usage

### Operating system page cache

```
Linux page cache uses LRU
  - Recently accessed pages stay in memory
  - Least recent pages evicted when memory full
  - Improves file I/O performance
```

---

### Database buffer pool

```
PostgreSQL, MySQL use LRU-like algorithms
  - Cache frequently accessed pages
  - Evict least recent when buffer full
  - Reduces disk I/O
```

---

### CDN caching

```
CDN edge servers use LRU
  - Cache popular content
  - Evict least recent when storage full
  - Improves content delivery speed
```

---

### Application-level caching

```
Redis, Memcached support LRU eviction
  - maxmemory-policy: allkeys-lru
  - When memory full, evict LRU keys
  - Keeps hot data in cache
```

---

## 7. Interview Problem Variations

### Variation 1: LRU with TTL

```
Each item has expiration time
Evict expired items before LRU

get(key):
  if key expired:
    remove key
    return -1
  else:
    move to end
    return value
```

---

### Variation 2: LRU with priority

```
Items have priority levels
Evict lowest priority first, then LRU within priority

Structure:
  Multiple LRU caches, one per priority level
  Evict from lowest priority cache first
```

---

### Variation 3: Thread-safe LRU

```
Add locking for concurrent access

class ThreadSafeLRUCache:
    def __init__(self, capacity):
        self.cache = LRUCache(capacity)
        self.lock = threading.Lock()
    
    def get(self, key):
        with self.lock:
            return self.cache.get(key)
    
    def put(self, key, value):
        with self.lock:
            self.cache.put(key, value)
```

---

### Variation 4: LFU (Least Frequently Used)

```
Track access frequency, evict least frequent

Structure:
  - Hash map: key → (value, frequency)
  - Min heap: (frequency, key) for finding LFU

get(key):  O(log n) due to heap operations
put(key, value):  O(log n)

More complex than LRU, but better for some workloads
```

---

## 8. Common Mistakes

### Mistake 1: Using single linked list

```
Single linked list:
  - Can't remove node in O(1) (need previous node)
  - Must traverse from head to find previous
  - O(n) removal

Doubly linked list:
  - Have prev pointer
  - O(1) removal
```

---

### Mistake 2: Not using dummy head/tail

```
Without dummy nodes:
  - Special cases for empty list
  - Special cases for single node
  - More complex code

With dummy nodes:
  - No special cases
  - Cleaner code
  - Easier to reason about
```

---

### Mistake 3: Forgetting to update hash map

```
When removing node from list:
  - Must also remove from hash map
  - Otherwise, hash map points to invalid node
  - Memory leak
```

---

## 9. Interview Tips

### Tip 1: Start with requirements

```
"Let me clarify the requirements:
  - What operations: get, put?
  - What should get return if key not found?
  - What's the capacity?
  - Thread-safe or single-threaded?
  - Any other operations like delete, clear?"
```

---

### Tip 2: Explain the approach

```
"I'll use a hash map for O(1) lookup and a doubly linked list
to maintain order. The hash map maps keys to nodes in the list.
The list keeps most recent items at the tail and least recent
at the head. When we access an item, we move it to the tail.
When capacity is exceeded, we remove from the head."
```

---

### Tip 3: Draw a diagram

```
Draw the structure:
  Hash Map: {1 → Node1, 2 → Node2, 3 → Node3}
  
  List: [Head] ↔ [Node1] ↔ [Node2] ↔ [Node3] ↔ [Tail]
         ↑                                      ↑
        LRU                                    MRU

Show operations visually
```

---

### Tip 4: Test with examples

```
Walk through operations:
  put(1, "a") → show state
  put(2, "b") → show state
  get(1) → show how 1 moves to end
  put(3, "c") → show eviction

Catch bugs early
```

---

## 10. Common Interview Questions + Answers

### Q: Why use a doubly linked list instead of a single linked list?

> "A doubly linked list allows O(1) removal of a node from anywhere in the list because we have a pointer to the previous node. With a single linked list, to remove a node, we'd need to traverse from the head to find the previous node, which is O(n). Since we need to move accessed nodes to the end (most recent) and remove the least recent node, we need O(1) removal, which requires a doubly linked list."

### Q: What's the space complexity of an LRU cache?

> "The space complexity is O(capacity). We store at most 'capacity' items in the hash map and the linked list. Each item requires a node in the list (with key, value, prev, next pointers) and an entry in the hash map. So the space is proportional to the capacity, which is O(capacity)."

### Q: How would you implement an LRU cache in a distributed system?

> "In a distributed system, I'd use Redis with its built-in LRU eviction policy. Configure maxmemory and maxmemory-policy allkeys-lru. Redis handles the LRU logic internally. For a custom implementation, I'd use a distributed cache like Redis or Memcached as the backing store, with a centralized service managing the LRU logic. The challenge is maintaining the linked list order across multiple servers, which requires coordination. Alternatively, use consistent hashing to partition the cache across servers, with each server maintaining its own LRU cache for its partition."

### Q: What's the difference between LRU and LFU?

> "LRU evicts the least recently used item — the item that hasn't been accessed for the longest time. LFU evicts the least frequently used item — the item with the lowest access count. LRU is simpler to implement (O(1) operations with hash map + doubly linked list) and works well for temporal locality. LFU is more complex (requires tracking frequencies, typically O(log n) operations with a heap) but can be better for workloads where some items are consistently popular over time. LRU is more commonly used because it's simpler and works well for most caching scenarios."

---

## 11. Quick Reference

```
LRU Cache = evict least recently used item when capacity reached

Implementation:
  - Hash map: key → Node (O(1) lookup)
  - Doubly linked list: maintain order (O(1) insert/delete)
  - Head: least recent
  - Tail: most recent

Operations:
  get(key):
    1. Lookup in hash map
    2. If found: move to tail (most recent), return value
    3. If not found: return -1
    Time: O(1)
  
  put(key, value):
    1. If exists: remove old node
    2. Create new node, add to tail
    3. Add to hash map
    4. If capacity exceeded: remove head (LRU)
    Time: O(1)

Node structure:
  key, value, prev, next

Why doubly linked list?
  - O(1) removal from anywhere (need prev pointer)
  - O(1) insertion at end
  - O(1) removal from head

Dummy head/tail:
  - Simplifies edge cases
  - No special handling for empty list

Variants:
  LRU-K:  Track last K accesses
  2Q:     Two queues (FIFO + LRU)
  ARC:    Adaptive (balances recent vs frequent)
  LFU:    Least frequently used (track access count)

Real-world usage:
  - OS page cache
  - Database buffer pool
  - CDN caching
  - Redis/Memcached eviction

Interview tips:
  ✅ Clarify requirements
  ✅ Explain approach (hash map + doubly linked list)
  ✅ Draw diagram
  ✅ Walk through example
  ✅ Test edge cases (empty, capacity 1, eviction)

Common mistakes:
  ❌ Single linked list (O(n) removal)
  ❌ No dummy nodes (complex edge cases)
  ❌ Forgetting to update hash map on removal

Key insight: O(1) get and put requires hash map + doubly linked list
```
