
## DSA Practice Sheet
[[DSA Practice Sheet]]

---

# Interview Template

## 1. Clarify Problem & Constraints (2–3 mins)

### Problem Understanding

- Restate the problem in your own words.
    
- Ask about input types, constraints, and expected output.
    
- Clarify edge cases (empty input, negative numbers, duplicates).
    

### Constraints

- Input size → affects choice of algorithm.
    
- Time / space limits → helps decide brute force vs optimized.
    
- Language / data type restrictions.
    

### Examples

- Work through 1–2 sample inputs manually.
    
- Helps catch misinterpretation early.
    

---

## 2. Identify Patterns / Problem Type (1–2 mins)

Ask yourself:

- Is it **Array / String / HashMap / Heap / Tree / Graph / DP / Sliding Window**?
    
- Does it involve **prefix sums, two pointers, recursion, backtracking**?
    
- Can a **greedy, binary search, or divide & conquer** approach work?
    

---

## 3. Define Approach / Plan (2–4 mins)

### Brute Force (Optional)

- Start with a simple, correct solution.
    
- Mention time & space complexity.
    

### Optimized Approach

- Identify data structures needed (HashMap, Heap, Stack, etc.)
    
- Decide algorithmic strategy:
    
    - Sliding window, two pointers, recursion, DP, BFS/DFS, graph traversal
        
- Consider edge cases and constraints.
    

---

## 4. Write Pseudocode / Method Signature (1–2 mins)

- Use clear function names.
    
- Identify inputs, outputs, return type.
    
- Example:
    

```java
int[] twoSum(int[] nums, int target)
boolean isValidBST(TreeNode root)
```

- Optional: write helper functions for recursion or DFS/BFS.
    

---

## 5. Implement Core Logic (5–10 mins)

### Guidelines

- Write clean, readable code.
    
- Use meaningful variable names.
    
- Handle base cases first (empty array, null node, etc.)
    
- Comment tricky sections briefly.
    

---

## 6. Think About Edge Cases & Validation

- Empty input, single element, all duplicates.
    
- Large input (stress test).
    
- Negative numbers / zeros.
    
- Overflow / underflow.
    
- Null pointers for linked list / tree.
    

---

## 7. Optimize Further (if time)

- Reduce time complexity (O(n^2 → O(n) if possible).
    
- Reduce space complexity (use in-place / bit manipulation).
    
- Use appropriate data structures.
    

### Examples

- Sliding window → O(n) instead of nested loop.
    
- Prefix sum → O(n) for sum queries.
    
- DP memoization → avoids recomputation.
    

---

## 8. Analyze Complexity

- Time complexity → worst case.
    
- Space complexity → auxiliary space.
    
- Mention trade-offs if using extra memory for speed.
    

---

## 9. Run Through Examples (2–3 mins)

- Dry run code with 2–3 examples, including edge cases.
    
- Explain output reasoning.
    
- Verify correctness.
    

---

## 10. Golden Lines to Say

- “Let’s first clarify constraints and edge cases.”
    
- “We can use a HashMap to reduce lookup time to O(1).”
    
- “Sliding window is ideal here for contiguous subarrays.”
    
- “I’ll handle base cases upfront to avoid null pointer errors.”
    
- “We can optimize this brute force to O(n) using X.”
    
- “I’ll dry run the solution with examples to validate correctness.”
    
- “We need to consider negative numbers / duplicates / overflow.”
    
