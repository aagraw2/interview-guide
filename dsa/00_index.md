# 📘 DSA Practice Sheet

## 🧠 Legend
- **P1** → Must Do  
- **P2** → Important  

---

## Hashing & Prefix Sum

### P1

| Pattern | Problem | Link | Concept | Difficulty |
|--------|--------|------|--------|------------|
| HashMap | Two Sum | https://leetcode.com/problems/two-sum/ | Store complement values in hashmap for O(1) lookup | Easy |
| HashMap | Contains Duplicate | https://leetcode.com/problems/contains-duplicate/ | Detect duplicates using hashset | Easy |
| Prefix Sum | Subarray Sum Equals K | https://leetcode.com/problems/subarray-sum-equals-k/ | Use prefix sums and hashmap to count valid subarrays | Medium |

### P2
_TBD_

---

## Sliding Window

### P1

| Pattern | Problem | Link | Concept | Difficulty |
|--------|--------|------|--------|------------|
| Sliding Window | Longest Substring Without Repeating Characters | https://leetcode.com/problems/longest-substring-without-repeating-characters/ | Expand and shrink window using set/map | Medium |
| Sliding Window | Minimum Window Substring | https://leetcode.com/problems/minimum-window-substring/ | Track required characters using frequency map | Hard |
| Sliding Window | Fruit Into Baskets | https://leetcode.com/problems/fruit-into-baskets/ | Window with at most two distinct elements | Medium |
| Sliding Window | Longest Repeating Character Replacement | https://leetcode.com/problems/longest-repeating-character-replacement/ | Window tracking most frequent character | Medium |

### P2
_TBD_

---

## Two Pointers

### P1

| Pattern | Problem | Link | Concept | Difficulty |
|--------|--------|------|--------|------------|
| Two Pointers | Valid Palindrome | https://leetcode.com/problems/valid-palindrome/ | Compare characters from both ends ignoring symbols | Easy |
| Two Pointers | 3Sum | https://leetcode.com/problems/3sum/ | Sort array then apply two pointer search | Medium |
| Two Pointers | Container With Most Water | https://leetcode.com/problems/container-with-most-water/ | Two pointers maximize area by shrinking width | Medium |
| Two Pointers | Remove Duplicates from Sorted Array | https://leetcode.com/problems/remove-duplicates-from-sorted-array/ | Use slow pointer to overwrite duplicates | Easy |

### P2
_TBD_

---

## Binary Search

### P1

| Pattern | Problem | Link | Concept | Difficulty |
|--------|--------|------|--------|------------|
| Binary Search | Find Peak Element | https://leetcode.com/problems/find-peak-element/ | Binary search based on slope change | Medium |
| Binary Search | Search in Rotated Sorted Array | https://leetcode.com/problems/search-in-rotated-sorted-array/ | Find sorted half and apply binary search | Medium |
| Binary Search | Median of Two Sorted Arrays | https://leetcode.com/problems/median-of-two-sorted-arrays/ | Partition two arrays using binary search | Hard |
| Binary Search | Koko Eating Bananas | https://leetcode.com/problems/koko-eating-bananas/ | Binary search minimum feasible speed | Medium |

### P2
_TBD_

---

## Stack & Monotonic Stack

### P1

| Pattern | Problem | Link | Concept | Difficulty |
|--------|--------|------|--------|------------|
| Stack | Valid Parentheses | https://leetcode.com/problems/valid-parentheses/ | Use stack to match opening and closing brackets | Easy |
| Monotonic Stack | Daily Temperatures | https://leetcode.com/problems/daily-temperatures/ | Monotonic stack to find next greater temperature | Medium |
| Monotonic Stack | Largest Rectangle in Histogram | https://leetcode.com/problems/largest-rectangle-in-histogram/ | Monotonic stack computing max width span | Hard |
| Stack | Asteroid Collision | https://leetcode.com/problems/asteroid-collision/ | Stack simulation of collisions | Medium |
| Monotonic Queue | Sliding Window Maximum | https://leetcode.com/problems/sliding-window-maximum/ | Monotonic deque maintaining max element | Hard |

### P2
_TBD_

---

## Linked List

### P1

| Pattern | Problem | Link | Concept | Difficulty |
|--------|--------|------|--------|------------|
| Linked List | Reverse Linked List | https://leetcode.com/problems/reverse-linked-list/ | Reverse pointers iteratively | Easy |
| Linked List | Linked List Cycle | https://leetcode.com/problems/linked-list-cycle/ | Detect cycle using fast and slow pointers | Easy |
| Floyd Cycle Detection | Linked List Cycle II | https://leetcode.com/problems/linked-list-cycle-ii/ | Detect start of cycle using Floyd algorithm | Medium |

### P2
_TBD_

---

## Heap / Priority Queue

### P1

| Pattern | Problem | Link | Concept | Difficulty |
|--------|--------|------|--------|------------|
| Heap | Merge K Sorted Lists | https://leetcode.com/problems/merge-k-sorted-lists/ | Min heap merging k sorted linked lists | Hard |
| Heap | Top K Frequent Elements | https://leetcode.com/problems/top-k-frequent-elements/ | Use heap to extract k most frequent elements | Medium |

### P2
_TBD_

---

## Greedy

### P1

| Pattern | Problem | Link | Concept | Difficulty |
|--------|--------|------|--------|------------|
| Greedy | Assign Cookies | https://leetcode.com/problems/assign-cookies/ | Greedy match smallest cookie satisfying child | Easy |
| Greedy | Gas Station | https://leetcode.com/problems/gas-station/ | Track cumulative fuel and reset start position | Medium |
| Greedy | Candy | https://leetcode.com/problems/candy/ | Two pass greedy to satisfy rating constraints | Hard |

### P2
_TBD_

---

## Dynamic Programming

### P1

| Pattern | Problem | Link | Concept | Difficulty |
|--------|--------|------|--------|------------|
| DP (1D) | Climbing Stairs | https://leetcode.com/problems/climbing-stairs/ | Fibonacci style DP relation | Easy |
| DP (1D) | House Robber | https://leetcode.com/problems/house-robber/ | DP deciding rob or skip house | Medium |
| DP (2D) | Longest Common Subsequence | https://leetcode.com/problems/longest-common-subsequence/ | 2D DP comparing characters | Medium |
| DP (2D) | Edit Distance | https://leetcode.com/problems/edit-distance/ | DP operations insert delete replace | Hard |
| Knapsack DP | Coin Change | https://leetcode.com/problems/coin-change/ | DP computing minimum number of coins | Medium |

### P2
_TBD_

---

## Trees

### P1

| Pattern | Problem | Link | Concept | Difficulty |
|--------|--------|------|--------|------------|
| Tree (DFS) | Invert Binary Tree | https://leetcode.com/problems/invert-binary-tree/ | Swap left and right recursively | Easy |
| Tree (DFS) | Diameter of Binary Tree | https://leetcode.com/problems/diameter-of-binary-tree/ | Longest path through tree nodes | Easy |
| Tree (LCA) | Lowest Common Ancestor | https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/ | Recursive ancestor discovery | Medium |

### P2
_TBD_

---

## Graphs

### P1

| Pattern | Problem | Link | Concept | Difficulty |
|--------|--------|------|--------|------------|
| Graph (DFS/BFS) | Number of Islands | https://leetcode.com/problems/number-of-islands/ | DFS flood fill connected components | Medium |
| Graph (BFS) | Rotting Oranges | https://leetcode.com/problems/rotting-oranges/ | Multi-source BFS spread simulation | Medium |
| Graph (Topo Sort) | Course Schedule | https://leetcode.com/problems/course-schedule/ | Detect cycle using indegree BFS | Medium |

### P2
_TBD_
