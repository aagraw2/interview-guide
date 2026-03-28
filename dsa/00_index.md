# 📘 DSA Practice Sheet (Complete - Full List)

## Legend
- P1 → Must Do  
- P2 → Important  

---

## Arrays & Hashing

| Name | Leetcode Link | Concept | Priority | Difficulty |
|------|--------------|--------|----------|------------|
| Two Sum | https://leetcode.com/problems/two-sum/ | Hashmap is used to store complements for constant time lookup | P1 | Easy |
| Contains Duplicate | https://leetcode.com/problems/contains-duplicate/ | HashSet is used to detect duplicates efficiently | P1 | Easy |
| Subarray Sum Equals K | https://leetcode.com/problems/subarray-sum-equals-k/ | Prefix sum (running sum) with hashmap to track frequencies | P1 | Medium |
| Single Number | https://leetcode.com/problems/single-number/ | XOR (bitwise operation that cancels duplicates) | P1 | Easy |
| Product of Array Except Self | https://leetcode.com/problems/product-of-array-except-self/ | Prefix and suffix multiplication without division | P1 | Medium |
| Best Time to Buy and Sell Stock | https://leetcode.com/problems/best-time-to-buy-and-sell-stock/ | Track minimum price seen so far | P1 | Easy |
| Maximum Subarray | https://leetcode.com/problems/maximum-subarray/ | Kadane’s algorithm (maximum subarray sum using running total) | P1 | Medium |
| Rotate Array | https://leetcode.com/problems/rotate-array/ | Reverse parts of array to rotate | P2 | Medium |
| Sort Colors | https://leetcode.com/problems/sort-colors/ | Dutch National Flag algorithm (three pointer partitioning) | P1 | Medium |
| Next Permutation | https://leetcode.com/problems/next-permutation/ | Find pivot and rearrange suffix in lexicographical order | P2 | Medium |
| Longest Consecutive Sequence | https://leetcode.com/problems/longest-consecutive-sequence/ | HashSet is used to expand sequences efficiently | P1 | Medium |
| First Missing Positive | https://leetcode.com/problems/first-missing-positive/ | Index marking (placing numbers at correct indices) | P2 | Hard |
| Counting Bits | https://leetcode.com/problems/counting-bits/ | Dynamic programming using previously computed results | P2 | Easy |
| Maximum XOR of Two Numbers | https://leetcode.com/problems/maximum-xor-of-two-numbers-in-an-array/ | Trie (prefix tree) for bitwise comparison | P2 | Medium |

---

## Sliding Window

| Name | Leetcode Link | Concept | Priority | Difficulty |
|------|--------------|--------|----------|------------|
| Longest Substring Without Repeating Characters | https://leetcode.com/problems/longest-substring-without-repeating-characters/ | Sliding window with set to maintain uniqueness | P1 | Medium |
| Minimum Window Substring | https://leetcode.com/problems/minimum-window-substring/ | Sliding window with frequency map and shrinking logic | P1 | Hard |
| Fruit Into Baskets | https://leetcode.com/problems/fruit-into-baskets/ | Sliding window with at most two distinct elements | P1 | Medium |
| Longest Repeating Character Replacement | https://leetcode.com/problems/longest-repeating-character-replacement/ | Window with max frequency tracking | P1 | Medium |
| Sliding Window Maximum | https://leetcode.com/problems/sliding-window-maximum/ | Monotonic deque (double-ended queue maintaining decreasing order) | P1 | Hard |

---

## Two Pointers

| Name | Leetcode Link | Concept | Priority | Difficulty |
|------|--------------|--------|----------|------------|
| Valid Palindrome | https://leetcode.com/problems/valid-palindrome/ | Two pointers ignoring non-alphanumeric characters | P1 | Easy |
| 3Sum | https://leetcode.com/problems/3sum/ | Sorting and using two pointers to find triplets | P1 | Medium |
| Container With Most Water | https://leetcode.com/problems/container-with-most-water/ | Move pointer with smaller height to maximize area | P1 | Medium |
| Trapping Rain Water | https://leetcode.com/problems/trapping-rain-water/ | Two pointers with prefix max tracking | P1 | Hard |

---

## Binary Search

| Name | Leetcode Link | Concept | Priority | Difficulty |
|------|--------------|--------|----------|------------|
| Search Insert Position | https://leetcode.com/problems/search-insert-position/ | Binary search for lower bound index | P1 | Easy |
| Find Peak Element | https://leetcode.com/problems/find-peak-element/ | Binary search using slope comparison | P1 | Medium |
| Search in Rotated Sorted Array | https://leetcode.com/problems/search-in-rotated-sorted-array/ | Identify sorted half in rotated array | P1 | Medium |
| Find Minimum in Rotated Sorted Array | https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/ | Binary search to find pivot element | P1 | Medium |
| Median of Two Sorted Arrays | https://leetcode.com/problems/median-of-two-sorted-arrays/ | Binary search partitioning technique | P2 | Hard |
| Koko Eating Bananas | https://leetcode.com/problems/koko-eating-bananas/ | Binary search on answer space | P1 | Medium |
| Capacity to Ship Packages Within D Days | https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/ | Binary search on feasible capacity | P1 | Medium |
| Split Array Largest Sum | https://leetcode.com/problems/split-array-largest-sum/ | Binary search to minimize largest subarray sum | P2 | Hard |
| Nth Root of Integer | N/A | Binary search to approximate root value | P2 | Medium |

---

## Stack

| Name | Leetcode Link | Concept | Priority | Difficulty |
|------|--------------|--------|----------|------------|
| Valid Parentheses | https://leetcode.com/problems/valid-parentheses/ | Stack is used to match opening and closing brackets | P1 | Easy |
| Daily Temperatures | https://leetcode.com/problems/daily-temperatures/ | Monotonic stack to find next greater element | P1 | Medium |
| Largest Rectangle in Histogram | https://leetcode.com/problems/largest-rectangle-in-histogram/ | Stack to calculate width and height efficiently | P1 | Hard |
| Asteroid Collision | https://leetcode.com/problems/asteroid-collision/ | Stack simulation of collisions | P2 | Medium |
| Min Stack | https://leetcode.com/problems/min-stack/ | Maintain minimum element along with stack | P1 | Medium |
| Evaluate Reverse Polish Notation | https://leetcode.com/problems/evaluate-reverse-polish-notation/ | Stack evaluation of postfix expression | P1 | Medium |
| Decode String | https://leetcode.com/problems/decode-string/ | Stack to decode nested patterns | P1 | Medium |
| Remove K Digits | https://leetcode.com/problems/remove-k-digits/ | Monotonic stack to remove larger digits | P2 | Medium |
| Car Fleet | https://leetcode.com/problems/car-fleet/ | Stack-like processing of arrival times | P2 | Medium |

---

## Linked List

| Name | Leetcode Link | Concept | Priority | Difficulty |
|------|--------------|--------|----------|------------|
| Reverse Linked List | https://leetcode.com/problems/reverse-linked-list/ | Iterative pointer reversal | P1 | Easy |
| Linked List Cycle | https://leetcode.com/problems/linked-list-cycle/ | Fast and slow pointers (Floyd algorithm) | P1 | Easy |
| Linked List Cycle II | https://leetcode.com/problems/linked-list-cycle-ii/ | Find cycle start using Floyd algorithm | P1 | Medium |
| Merge K Sorted Lists | https://leetcode.com/problems/merge-k-sorted-lists/ | Min heap to merge lists | P1 | Hard |
| LRU Cache | https://leetcode.com/problems/lru-cache/ | Doubly linked list and hashmap for O(1) operations | P1 | Medium |
| Middle of the Linked List | https://leetcode.com/problems/middle-of-the-linked-list/ | Slow and fast pointers | P1 | Easy |
| Add Two Numbers | https://leetcode.com/problems/add-two-numbers/ | Simulate addition with carry | P1 | Medium |
| Intersection of Two Linked Lists | https://leetcode.com/problems/intersection-of-two-linked-lists/ | Two pointer switching technique | P1 | Easy |
| Odd Even Linked List | https://leetcode.com/problems/odd-even-linked-list/ | Separate odd and even indexed nodes | P2 | Medium |
| Reverse Nodes in K Group | https://leetcode.com/problems/reverse-nodes-in-k-group/ | Reverse nodes in chunks | P2 | Hard |

---

## Greedy

| Name | Leetcode Link | Concept | Priority | Difficulty |
|------|--------------|--------|----------|------------|
| Assign Cookies | https://leetcode.com/problems/assign-cookies/ | Greedy matching smallest valid pair | P1 | Easy |
| Gas Station | https://leetcode.com/problems/gas-station/ | Greedy traversal to find valid start | P1 | Medium |
| Jump Game | https://leetcode.com/problems/jump-game/ | Track maximum reachable index | P1 | Medium |
| Jump Game II | https://leetcode.com/problems/jump-game-ii/ | Greedy level-based jump count | P1 | Medium |
| Candy | https://leetcode.com/problems/candy/ | Two-pass greedy distribution | P2 | Hard |
| Minimum Number of Arrows to Burst Balloons | https://leetcode.com/problems/minimum-number-of-arrows-to-burst-balloons/ | Interval overlap greedy | P1 | Medium |
| Non Overlapping Intervals | https://leetcode.com/problems/non-overlapping-intervals/ | Sort by end time and remove overlaps | P1 | Medium |

---

## Backtracking

| Name | Leetcode Link | Concept | Priority | Difficulty |
|------|--------------|--------|----------|------------|
| Permutations | https://leetcode.com/problems/permutations/ | Generate all arrangements recursively | P1 | Medium |
| Combination Sum | https://leetcode.com/problems/combination-sum/ | Explore combinations with repetition | P1 | Medium |
| N Queens | https://leetcode.com/problems/n-queens/ | Place queens with constraints checking | P2 | Hard |
| Sudoku Solver | https://leetcode.com/problems/sudoku-solver/ | Fill grid using backtracking | P2 | Hard |
| Subsets | https://leetcode.com/problems/subsets/ | Generate all subsets using recursion | P1 | Medium |
| Subsets II | https://leetcode.com/problems/subsets-ii/ | Handle duplicates in subset generation | P2 | Medium |
| Combination Sum II | https://leetcode.com/problems/combination-sum-ii/ | Avoid duplicate combinations | P2 | Medium |
| Palindrome Partitioning | https://leetcode.com/problems/palindrome-partitioning/ | Partition string into palindromes | P1 | Medium |
| Word Search | https://leetcode.com/problems/word-search/ | DFS search in grid | P1 | Medium |
| Word Search II | https://leetcode.com/problems/word-search-ii/ | Trie with DFS optimization | P2 | Hard |

---

## Dynamic Programming

| Name | Leetcode Link | Concept | Priority | Difficulty |
|------|--------------|--------|----------|------------|
| Decode Ways | https://leetcode.com/problems/decode-ways/ | DP on string using previous valid states | P1 | Medium |
| Palindrome Partitioning II | https://leetcode.com/problems/palindrome-partitioning-ii/ | DP + palindrome precomputation | P2 | Hard |
| Rod Cutting Problem | N/A | Unbounded knapsack (choose cuts for max profit) | P2 | Medium |
| Unique Paths | https://leetcode.com/problems/unique-paths/ | Grid DP counting paths | P1 | Medium |
| Unique Paths II | https://leetcode.com/problems/unique-paths-ii/ | DP with obstacles | P1 | Medium |
| Minimum Path Sum | https://leetcode.com/problems/minimum-path-sum/ | DP minimizing path cost | P1 | Medium |
| Target Sum | https://leetcode.com/problems/target-sum/ | Convert to subset sum problem | P2 | Medium |
| House Robber II | https://leetcode.com/problems/house-robber-ii/ | Circular house constraint | P1 | Medium |
| Min Cost Climbing Stairs | https://leetcode.com/problems/min-cost-climbing-stairs/ | DP with min cost transitions | P1 | Easy |
| Word Break II | https://leetcode.com/problems/word-break-ii/ | DP + backtracking to generate sentences | P2 | Hard |
| Longest Palindromic Subsequence | https://leetcode.com/problems/longest-palindromic-subsequence/ | DP on substring ranges | P2 | Medium |
| Distinct Subsequences | https://leetcode.com/problems/distinct-subsequences/ | DP counting subsequences | P2 | Hard |
| Minimum Falling Path Sum | https://leetcode.com/problems/minimum-falling-path-sum/ | DP on matrix | P2 | Medium |
| Dungeon Game | https://leetcode.com/problems/dungeon-game/ | Reverse DP from destination | P2 | Hard |
| Cherry Pickup | https://leetcode.com/problems/cherry-pickup/ | 3D DP / two paths simultaneously | P2 | Hard |
| Matrix Chain Multiplication | N/A | DP splitting intervals optimally | P2 | Hard |
| Burst Balloons | https://leetcode.com/problems/burst-balloons/ | Interval DP (choose last burst) | P2 | Hard |
| Maximum Profit with K Transactions | N/A | DP with transaction states | P2 | Hard |
| Stock Buy Sell with Cooldown | https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/ | DP with cooldown state | P2 | Medium |
| Stock Buy Sell with Transaction Fee | https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-transaction-fee/ | DP with cost adjustment | P2 | Medium |
| Knapsack (0/1) | N/A | Choose or skip items | P1 | Medium |
| Unbounded Knapsack | N/A | Infinite supply variant | P1 | Medium |

---

## Trees

| Name | Leetcode Link | Concept | Priority | Difficulty |
|------|--------------|--------|----------|------------|
| Maximum Depth of Binary Tree | https://leetcode.com/problems/maximum-depth-of-binary-tree/ | DFS height calculation | P1 | Easy |
| Balanced Binary Tree | https://leetcode.com/problems/balanced-binary-tree/ | Check height difference | P1 | Easy |
| Same Tree | https://leetcode.com/problems/same-tree/ | Recursive comparison | P1 | Easy |
| Binary Tree Right Side View | https://leetcode.com/problems/binary-tree-right-side-view/ | Level order traversal (BFS) | P1 | Medium |
| Serialize and Deserialize Binary Tree | https://leetcode.com/problems/serialize-and-deserialize-binary-tree/ | Encode tree using DFS/BFS | P1 | Hard |
| Construct Binary Tree from Preorder and Inorder | https://leetcode.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/ | Tree reconstruction using recursion | P1 | Medium |
| Search in BST | https://leetcode.com/problems/search-in-a-binary-search-tree/ | BST traversal property | P1 | Easy |
| Convert Sorted Array to BST | https://leetcode.com/problems/convert-sorted-array-to-binary-search-tree/ | Build balanced BST | P1 | Easy |
| Binary Tree Zigzag Level Order Traversal | https://leetcode.com/problems/binary-tree-zigzag-level-order-traversal/ | BFS alternating directions | P2 | Medium |
| Binary Tree Maximum Path Sum | https://leetcode.com/problems/binary-tree-maximum-path-sum/ | DFS tracking global max | P2 | Hard |
| Boundary Traversal of Binary Tree | N/A | Traverse boundary nodes in order | P2 | Medium |
| Sum Root to Leaf Numbers | https://leetcode.com/problems/sum-root-to-leaf-numbers/ | DFS forming numbers | P2 | Medium |
| Path Sum | https://leetcode.com/problems/path-sum/ | DFS checking sum | P1 | Easy |
| Path Sum III | https://leetcode.com/problems/path-sum-iii/ | Prefix sum in tree | P2 | Medium |
| Lowest Common Ancestor (Binary Tree) | https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/ | Recursive search | P1 | Medium |
| Lowest Common Ancestor (BST) | https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree/ | BST property traversal | P1 | Easy |
| Convert Sorted List to BST | https://leetcode.com/problems/convert-sorted-list-to-binary-search-tree/ | Inorder simulation | P2 | Medium |
| Flatten Binary Tree to Linked List | https://leetcode.com/problems/flatten-binary-tree-to-linked-list/ | Modify tree to linked list | P2 | Medium |
| Recover Binary Search Tree | https://leetcode.com/problems/recover-binary-search-tree/ | Fix swapped nodes using inorder | P2 | Hard |
| Inorder Successor in BST | https://leetcode.com/problems/inorder-successor-in-bst/ | Next greater node in BST | P2 | Medium |

---

## Graphs

| Name | Leetcode Link | Concept | Priority | Difficulty |
|------|--------------|--------|----------|------------|
| Number of Islands | https://leetcode.com/problems/number-of-islands/ | DFS/BFS traversal on grid | P1 | Medium |
| Clone Graph | https://leetcode.com/problems/clone-graph/ | Graph copy using DFS/BFS | P1 | Medium |
| Rotting Oranges | https://leetcode.com/problems/rotting-oranges/ | Multi-source BFS | P1 | Medium |
| Pacific Atlantic Water Flow | https://leetcode.com/problems/pacific-atlantic-water-flow/ | DFS from borders | P2 | Medium |
| Course Schedule | https://leetcode.com/problems/course-schedule/ | Topological sort using BFS/DFS | P1 | Medium |
| Course Schedule II | https://leetcode.com/problems/course-schedule-ii/ | Return ordering from topo sort | P1 | Medium |
| Network Delay Time | https://leetcode.com/problems/network-delay-time/ | Dijkstra shortest path | P1 | Medium |
| Redundant Connection | https://leetcode.com/problems/redundant-connection/ | Union-Find cycle detection | P1 | Medium |
| Redundant Connection II | https://leetcode.com/problems/redundant-connection-ii/ | Directed graph cycle detection | P2 | Hard |
| Detect Cycle in Undirected Graph | N/A | DFS/Union-Find cycle detection | P1 | Medium |
| Detect Cycle in Directed Graph | N/A | DFS with recursion stack | P1 | Medium |
| Number of Provinces | https://leetcode.com/problems/number-of-provinces/ | Connected components | P1 | Medium |
| Accounts Merge | https://leetcode.com/problems/accounts-merge/ | Union-Find grouping | P1 | Medium |
| Word Ladder | https://leetcode.com/problems/word-ladder/ | BFS shortest transformation | P1 | Hard |
| Alien Dictionary | https://leetcode.com/problems/alien-dictionary/ | Topological sort on characters | P2 | Hard |
| Cheapest Flights Within K Stops | https://leetcode.com/problems/cheapest-flights-within-k-stops/ | Modified BFS/Dijkstra | P1 | Medium |
| Swim in Rising Water | https://leetcode.com/problems/swim-in-rising-water/ | Dijkstra / binary search + BFS | P2 | Hard |
| Shortest Path in Binary Matrix | https://leetcode.com/problems/shortest-path-in-binary-matrix/ | BFS shortest path | P1 | Medium |
| Flood Fill | https://leetcode.com/problems/flood-fill/ | DFS fill algorithm | P1 | Easy |
| Number of Distinct Islands | https://leetcode.com/problems/number-of-distinct-islands/ | Shape hashing | P2 | Medium |
| Eventual Safe States | https://leetcode.com/problems/find-eventual-safe-states/ | Reverse graph topo sort | P2 | Medium |
| Reconstruct Itinerary | https://leetcode.com/problems/reconstruct-itinerary/ | Eulerian path (graph traversal) | P2 | Hard |
| Minimum Effort Path | https://leetcode.com/problems/path-with-minimum-effort/ | Dijkstra / binary search | P2 | Medium |
| Strongly Connected Components | N/A | Kosaraju / Tarjan algorithm | P2 | Hard |
| Bellman Ford Algorithm | N/A | Shortest path with negative edges | P2 | Medium |
| Floyd Warshall Algorithm | N/A | All-pairs shortest path | P2 | Hard |
| Graph Valid Tree | https://leetcode.com/problems/graph-valid-tree/ | Check cycle + connectivity | P1 | Medium |

---

## Heap / Trie / Misc

| Name | Leetcode Link | Concept | Priority | Difficulty |
|------|--------------|--------|----------|------------|
| Top K Frequent Elements | https://leetcode.com/problems/top-k-frequent-elements/ | Min heap for top K | P1 | Medium |
| Most Frequent IDs | https://leetcode.com/problems/most-frequent-ids/ | Heap + hashmap frequency tracking | P2 | Hard |
| Kth Largest Element in Array | https://leetcode.com/problems/kth-largest-element-in-an-array/ | Min heap of size K | P1 | Medium |
| Find Median from Data Stream | https://leetcode.com/problems/find-median-from-data-stream/ | Two heaps (max + min) | P1 | Hard |
| Last Stone Weight | https://leetcode.com/problems/last-stone-weight/ | Max heap simulation | P1 | Easy |
| Merge K Sorted Arrays | N/A | Heap merge technique | P2 | Medium |
| K Closest Points to Origin | https://leetcode.com/problems/k-closest-points-to-origin/ | Heap based selection | P1 | Medium |
| Implement Trie | https://leetcode.com/problems/implement-trie-prefix-tree/ | Prefix tree for string search | P1 | Medium |
| Add and Search Word | https://leetcode.com/problems/design-add-and-search-words-data-structure/ | Trie with wildcard support | P2 | Medium |
| Encode and Decode Strings | https://leetcode.com/problems/encode-and-decode-strings/ | String serialization | P2 | Medium |

---

## Math / Bit / Misc

| Name | Leetcode Link | Concept | Priority | Difficulty |
|------|--------------|--------|----------|------------|
| Power of Two | https://leetcode.com/problems/power-of-two/ | Bit manipulation (single bit check) | P1 | Easy |
| Hamming Distance | https://leetcode.com/problems/hamming-distance/ | Count differing bits | P1 | Easy |
| Reverse Bits | https://leetcode.com/problems/reverse-bits/ | Bit reversal | P2 | Easy |
| Happy Number | https://leetcode.com/problems/happy-number/ | Cycle detection using set | P1 | Easy |
| Plus One | https://leetcode.com/problems/plus-one/ | Digit carry simulation | P1 | Easy |
| Excel Sheet Column Number | https://leetcode.com/problems/excel-sheet-column-number/ | Base conversion | P1 | Easy |
| Insert Delete GetRandom O(1) | https://leetcode.com/problems/insert-delete-getrandom-o1/ | Array + hashmap for O(1) ops | P1 | Medium |
| Max Points on a Line | https://leetcode.com/problems/max-points-on-a-line/ | Slope hashing | P2 | Hard |

---
