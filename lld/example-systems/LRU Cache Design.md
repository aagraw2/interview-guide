
## 1. Functional Requirements

- Put a key-value pair into cache
    
- Get value by key
    
- Evict least recently used (LRU) item when capacity is exceeded
    
- Update value if key already exists
    
- Maintain access order
    

---

## 2. Non-Functional Requirements

- O(1) time complexity for get and put
    
- Thread-safe operations (basic consideration)
    
- Memory efficient
    
- Scalable for high read/write operations
    
- Extensible for eviction policy changes
    

---

## 3. Core Entities

|Entity|Responsibility|
|---|---|
|Node|Represents a cache entry (key, value)|
|DoublyLinkedList|Maintains usage order (LRU → MRU)|
|Cache|Stores key → Node mapping|
|EvictionPolicy|Defines eviction logic (LRU)|
|CacheService|Exposes get/put operations|

---

## 4. Flows (Use Cases)

### Get Key (Main Flow)

1. Check if key exists in cache map
    
2. If not found → return -1
    
3. If found:
    
    - Move node to head (most recently used)
        
    - Return value
        

---

### Put Key (Main Flow)

1. Check if key exists:
    
    - If yes:
        
        - Update value
            
        - Move node to head
            
2. If not:
    
    - Create new node
        
    - Add node to head
        
    - Add to map
        
3. If capacity exceeded:
    
    - Remove tail node (LRU)
        
    - Remove from map
        

---

### Eviction (Supporting Flow)

1. Identify least recently used node (tail.prev)
    
2. Remove node from DLL
    
3. Remove from map
    

---

## 5. Code

```java
import java.util.*;

// MODELS

class Node {
    int key, value;
    Node prev, next;

    public Node(int key, int value) {
        this.key = key;
        this.value = value;
    }
}

// Doubly Linked List to maintain LRU order
class DoublyLinkedList {
    Node head, tail;

    public DoublyLinkedList() {
        head = new Node(-1, -1);
        tail = new Node(-1, -1);
        head.next = tail;
        tail.prev = head;
    }

    // Add node right after head (MRU position)
    public void addFirst(Node node) {
        node.next = head.next;
        node.prev = head;

        head.next.prev = node;
        head.next = node;
    }

    // Remove any node
    public void remove(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    // Remove LRU node (last node before tail)
    public Node removeLast() {
        if (tail.prev == head) return null;
        Node lru = tail.prev;
        remove(lru);
        return lru;
    }
}

// Strategy Pattern for eviction policy
interface EvictionPolicy {
    void keyAccessed(Node node);
    Node evict();
}

// LRU Eviction Policy Implementation
class LRUEvictionPolicy implements EvictionPolicy {
    private DoublyLinkedList dll;

    public LRUEvictionPolicy(DoublyLinkedList dll) {
        this.dll = dll;
    }

    @Override
    public void keyAccessed(Node node) {
        dll.remove(node);
        dll.addFirst(node);
    }

    @Override
    public Node evict() {
        return dll.removeLast();
    }
}

// SERVICES

// Singleton Cache Service
class LRUCache {
    private static LRUCache instance;

    private final int capacity;
    private Map<Integer, Node> cache;
    private DoublyLinkedList dll;
    private EvictionPolicy evictionPolicy;

    // Private constructor for Singleton
    private LRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new HashMap<>();
        this.dll = new DoublyLinkedList();
        this.evictionPolicy = new LRUEvictionPolicy(dll);
    }

    // Singleton instance getter
    public static LRUCache getInstance(int capacity) {
        if (instance == null) {
            synchronized (LRUCache.class) {
                if (instance == null) {
                    instance = new LRUCache(capacity);
                }
            }
        }
        return instance;
    }

    // Get operation
    public int get(int key) {
        if (!cache.containsKey(key)) return -1;

        Node node = cache.get(key);

        // Move to MRU position
        evictionPolicy.keyAccessed(node);

        return node.value;
    }

    // Put operation
    public void put(int key, int value) {
        if (cache.containsKey(key)) {
            Node node = cache.get(key);
            node.value = value;

            // Move to MRU
            evictionPolicy.keyAccessed(node);
        } else {
            Node node = new Node(key, value);
            cache.put(key, node);
            dll.addFirst(node);

            if (cache.size() > capacity) {
                Node evicted = evictionPolicy.evict();
                if (evicted != null) {
                    cache.remove(evicted.key);
                }
            }
        }
    }
}
```

---

## 6. Complexity Analysis

### Get Operation

- HashMap lookup → O(1)
    
- DLL remove + insert → O(1)
    

**Overall: O(1)**

---

### Put Operation

- HashMap lookup → O(1)
    
- DLL insert/remove → O(1)
    
- Eviction (if needed) → O(1)
    

**Overall: O(1)**

---

### Eviction

- Remove last node → O(1)
    
- Map removal → O(1)
    

**Overall: O(1)**

---

## 7. Key Design Decisions

- HashMap + DoublyLinkedList ensures strict O(1) operations
    
- Strategy Pattern allows easy extension to LFU or custom eviction
    
- Singleton ensures single cache instance (common in infra systems)
    
- Separation of DLL and eviction policy improves modularity
    
- Clean separation between data structure and service logic