# Heap & Trie Patterns

Heaps efficiently track min/max. Tries enable fast prefix operations.

---

## HEAP PATTERNS

## Pattern 1: Top K Elements (Min-Heap of Size K)

### Core Idea
Use min-heap of size k. After processing, heap contains k largest elements.

### When to Use
- K largest/most frequent elements
- K closest points

### Mental Model
```
for each element:
    add to min-heap
    if heap.size > k:
        remove smallest
heap contains k largest
```

### Example: Kth Largest Element in Array
```java
public int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();

    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) {
            minHeap.poll();
        }
    }

    return minHeap.peek();
}
```

### Example: Top K Frequent Elements
```java
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int num : nums) {
        freq.put(num, freq.getOrDefault(num, 0) + 1);
    }

    // Min-heap by frequency
    PriorityQueue<Integer> heap = new PriorityQueue<>(
        (a, b) -> freq.get(a) - freq.get(b)
    );

    for (int num : freq.keySet()) {
        heap.offer(num);
        if (heap.size() > k) {
            heap.poll();
        }
    }

    int[] result = new int[k];
    for (int i = 0; i < k; i++) {
        result[i] = heap.poll();
    }
    return result;
}
```

### Example: K Closest Points to Origin
```java
public int[][] kClosest(int[][] points, int k) {
    // Max-heap by distance
    PriorityQueue<int[]> heap = new PriorityQueue<>(
        (a, b) -> (b[0]*b[0] + b[1]*b[1]) - (a[0]*a[0] + a[1]*a[1])
    );

    for (int[] point : points) {
        heap.offer(point);
        if (heap.size() > k) {
            heap.poll();
        }
    }

    return heap.toArray(new int[k][]);
}
```

---

## Pattern 2: Two Heaps (Find Median)

### Core Idea
Use max-heap for lower half, min-heap for upper half. Median is at the tops.

### When to Use
- Running median
- Sliding window median

### Mental Model
```
maxHeap: lower half (max at top)
minHeap: upper half (min at top)

balance: |maxHeap| == |minHeap| or |maxHeap| == |minHeap| + 1

median = maxHeap.peek() or average of both tops
```

### Example: Find Median from Data Stream
```java
class MedianFinder {
    private PriorityQueue<Integer> maxHeap;  // Lower half
    private PriorityQueue<Integer> minHeap;  // Upper half

    public MedianFinder() {
        maxHeap = new PriorityQueue<>(Collections.reverseOrder());
        minHeap = new PriorityQueue<>();
    }

    public void addNum(int num) {
        maxHeap.offer(num);
        minHeap.offer(maxHeap.poll());

        if (minHeap.size() > maxHeap.size()) {
            maxHeap.offer(minHeap.poll());
        }
    }

    public double findMedian() {
        if (maxHeap.size() > minHeap.size()) {
            return maxHeap.peek();
        }
        return (maxHeap.peek() + minHeap.peek()) / 2.0;
    }
}
```

---

## Pattern 3: Merge K Sorted (Min-Heap)

### Core Idea
Min-heap with one element from each list. Pop smallest, push its successor.

### When to Use
- Merge k sorted lists/arrays
- K-way merge

### Example: Merge K Sorted Lists
```java
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> heap = new PriorityQueue<>(
        (a, b) -> a.val - b.val
    );

    for (ListNode node : lists) {
        if (node != null) {
            heap.offer(node);
        }
    }

    ListNode dummy = new ListNode(0);
    ListNode curr = dummy;

    while (!heap.isEmpty()) {
        ListNode smallest = heap.poll();
        curr.next = smallest;
        curr = curr.next;

        if (smallest.next != null) {
            heap.offer(smallest.next);
        }
    }

    return dummy.next;
}
```

---

## Pattern 4: Heap for Scheduling

### Core Idea
Use heap to track resources. Pop when resource is free.

### When to Use
- Meeting rooms
- CPU scheduling

### Example: Meeting Rooms II
```java
public int minMeetingRooms(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]);

    PriorityQueue<Integer> endTimes = new PriorityQueue<>();

    for (int[] interval : intervals) {
        if (!endTimes.isEmpty() && interval[0] >= endTimes.peek()) {
            endTimes.poll();  // Reuse room
        }
        endTimes.offer(interval[1]);
    }

    return endTimes.size();
}
```

---

## TRIE PATTERNS

## Pattern 5: Basic Trie (Prefix Tree)

### Core Idea
Tree where each node represents a character. Path from root = prefix.

### When to Use
- Autocomplete
- Spell checker
- Word search

### Template
```java
class Trie {
    private TrieNode root;

    class TrieNode {
        TrieNode[] children = new TrieNode[26];
        boolean isEnd = false;
    }

    public Trie() {
        root = new TrieNode();
    }

    public void insert(String word) {
        TrieNode node = root;
        for (char c : word.toCharArray()) {
            int idx = c - 'a';
            if (node.children[idx] == null) {
                node.children[idx] = new TrieNode();
            }
            node = node.children[idx];
        }
        node.isEnd = true;
    }

    public boolean search(String word) {
        TrieNode node = searchPrefix(word);
        return node != null && node.isEnd;
    }

    public boolean startsWith(String prefix) {
        return searchPrefix(prefix) != null;
    }

    private TrieNode searchPrefix(String prefix) {
        TrieNode node = root;
        for (char c : prefix.toCharArray()) {
            int idx = c - 'a';
            if (node.children[idx] == null) {
                return null;
            }
            node = node.children[idx];
        }
        return node;
    }
}
```

---

## Pattern 6: Trie with Wildcard Search

### Core Idea
DFS when encountering wildcard (.), try all children.

### When to Use
- Word dictionary with wildcards
- Pattern matching

### Example: Add and Search Word
```java
class WordDictionary {
    private TrieNode root;

    class TrieNode {
        TrieNode[] children = new TrieNode[26];
        boolean isEnd = false;
    }

    public WordDictionary() {
        root = new TrieNode();
    }

    public void addWord(String word) {
        TrieNode node = root;
        for (char c : word.toCharArray()) {
            int idx = c - 'a';
            if (node.children[idx] == null) {
                node.children[idx] = new TrieNode();
            }
            node = node.children[idx];
        }
        node.isEnd = true;
    }

    public boolean search(String word) {
        return dfs(word, 0, root);
    }

    private boolean dfs(String word, int index, TrieNode node) {
        if (index == word.length()) {
            return node.isEnd;
        }

        char c = word.charAt(index);

        if (c == '.') {
            for (TrieNode child : node.children) {
                if (child != null && dfs(word, index + 1, child)) {
                    return true;
                }
            }
            return false;
        } else {
            TrieNode child = node.children[c - 'a'];
            return child != null && dfs(word, index + 1, child);
        }
    }
}
```

---

## Pattern 7: Trie for Word Search II

### Core Idea
Build trie from words, DFS on board, prune trie branches.

### When to Use
- Find multiple words in grid
- Optimize multiple word searches

### Example: Word Search II
```java
class TrieNode {
    TrieNode[] children = new TrieNode[26];
    String word = null;
}

public List<String> findWords(char[][] board, String[] words) {
    TrieNode root = buildTrie(words);
    List<String> result = new ArrayList<>();

    for (int i = 0; i < board.length; i++) {
        for (int j = 0; j < board[0].length; j++) {
            dfs(board, i, j, root, result);
        }
    }

    return result;
}

private void dfs(char[][] board, int i, int j, TrieNode node, List<String> result) {
    if (i < 0 || i >= board.length || j < 0 || j >= board[0].length) return;

    char c = board[i][j];
    if (c == '#' || node.children[c - 'a'] == null) return;

    node = node.children[c - 'a'];
    if (node.word != null) {
        result.add(node.word);
        node.word = null;  // Avoid duplicates
    }

    board[i][j] = '#';  // Mark visited
    dfs(board, i + 1, j, node, result);
    dfs(board, i - 1, j, node, result);
    dfs(board, i, j + 1, node, result);
    dfs(board, i, j - 1, node, result);
    board[i][j] = c;    // Restore
}

private TrieNode buildTrie(String[] words) {
    TrieNode root = new TrieNode();
    for (String word : words) {
        TrieNode node = root;
        for (char c : word.toCharArray()) {
            int idx = c - 'a';
            if (node.children[idx] == null) {
                node.children[idx] = new TrieNode();
            }
            node = node.children[idx];
        }
        node.word = word;
    }
    return root;
}
```

---

## Quick Reference

### Heap
| Pattern | Heap Type | Size | Use Case |
|---------|-----------|------|----------|
| Top K | Min-heap | K | K largest elements |
| Bottom K | Max-heap | K | K smallest elements |
| Two Heaps | Max + Min | n/2 each | Running median |
| K-way Merge | Min-heap | K | Merge sorted lists |
| Scheduling | Min-heap | Variable | Resource allocation |

### Trie
| Pattern | Technique | Use Case |
|---------|-----------|----------|
| Basic Trie | insert/search/startsWith | Prefix operations |
| Wildcard | DFS for '.' | Pattern matching |
| Word Search | Trie + grid DFS | Multiple word search |

### Heap Operations
```java
// Min-heap (default)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();

// Max-heap
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());

// Custom comparator
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
```
