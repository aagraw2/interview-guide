# Prefix Sums Patterns

Prefix sums enable O(1) range queries after O(n) preprocessing. Essential for subarray sum problems.

---

## Pattern 1: Basic Prefix Sum Array

### Core Idea
Build an array where prefix[i] = sum of elements from index 0 to i-1. Range sum = prefix[right+1] - prefix[left].

### When to Use
- Multiple range sum queries
- Subarray sum problems
- Need O(1) range queries

### Mental Model
```
prefix[0] = 0
prefix[i] = prefix[i-1] + nums[i-1]

sum(left, right) = prefix[right+1] - prefix[left]
```

### Generic Template
```java
public class PrefixSum {
    private int[] prefix;

    public PrefixSum(int[] nums) {
        prefix = new int[nums.length + 1];
        for (int i = 0; i < nums.length; i++) {
            prefix[i + 1] = prefix[i] + nums[i];
        }
    }

    public int rangeSum(int left, int right) {
        return prefix[right + 1] - prefix[left];
    }
}
```

### Example: Range Sum Query - Immutable
```java
class NumArray {
    private int[] prefix;

    public NumArray(int[] nums) {
        prefix = new int[nums.length + 1];
        for (int i = 0; i < nums.length; i++) {
            prefix[i + 1] = prefix[i] + nums[i];
        }
    }

    public int sumRange(int left, int right) {
        return prefix[right + 1] - prefix[left];
    }
}
```

---

## Pattern 2: Prefix Sum + HashMap (Subarray Sum Equals K)

### Core Idea
Store prefix sums in a map. If currentSum - k exists in map, there's a subarray with sum k.

### When to Use
- Count subarrays with exact sum
- Find subarray with target sum
- Need O(n) time

### Mental Model
```
map stores: prefixSum → count of occurrences

for each position:
    if (currentSum - target) exists in map:
        found subarrays ending here
    add currentSum to map
```

### Generic Template
```java
public int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> prefixCount = new HashMap<>();
    prefixCount.put(0, 1);  // Empty prefix

    int currentSum = 0;
    int count = 0;

    for (int num : nums) {
        currentSum += num;

        // If (currentSum - k) seen before, those are valid subarrays
        count += prefixCount.getOrDefault(currentSum - k, 0);

        prefixCount.put(currentSum, prefixCount.getOrDefault(currentSum, 0) + 1);
    }

    return count;
}
```

### Example: Subarray Sum Equals K
```java
public int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> prefixCount = new HashMap<>();
    prefixCount.put(0, 1);

    int sum = 0;
    int count = 0;

    for (int num : nums) {
        sum += num;
        count += prefixCount.getOrDefault(sum - k, 0);
        prefixCount.put(sum, prefixCount.getOrDefault(sum, 0) + 1);
    }

    return count;
}
```

---

## Pattern 3: Prefix Product (Except Self)

### Core Idea
Build left products and right products separately, then multiply them.

### When to Use
- Product of array except self
- Can't use division
- O(n) time, O(1) extra space

### Mental Model
```
result[i] = leftProduct[i] * rightProduct[i]

Pass 1: Build left products
Pass 2: Multiply by right products (backwards)
```

### Example: Product of Array Except Self
```java
public int[] productExceptSelf(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];

    // Left products
    result[0] = 1;
    for (int i = 1; i < n; i++) {
        result[i] = result[i - 1] * nums[i - 1];
    }

    // Multiply by right products
    int rightProduct = 1;
    for (int i = n - 1; i >= 0; i--) {
        result[i] *= rightProduct;
        rightProduct *= nums[i];
    }

    return result;
}
```

---

## Pattern 4: Difference Array (Range Updates)

### Core Idea
For range updates, use a difference array. diff[i] = nums[i] - nums[i-1]. Range add becomes two point updates.

### When to Use
- Multiple range increment operations
- Need to apply updates efficiently
- Query final array after all updates

### Mental Model
```
To add val to range [left, right]:
    diff[left] += val
    diff[right + 1] -= val

Final array = prefix sum of diff array
```

### Generic Template
```java
public int[] rangeAddQueries(int length, int[][] updates) {
    int[] diff = new int[length + 1];

    for (int[] update : updates) {
        int left = update[0];
        int right = update[1];
        int val = update[2];

        diff[left] += val;
        diff[right + 1] -= val;
    }

    // Build result from prefix sum
    int[] result = new int[length];
    result[0] = diff[0];
    for (int i = 1; i < length; i++) {
        result[i] = result[i - 1] + diff[i];
    }

    return result;
}
```

---

## Pattern 5: 2D Prefix Sum

### Core Idea
Extend prefix sums to 2D for rectangle sum queries.

### When to Use
- Sum of submatrix queries
- 2D range queries

### Mental Model
```
prefix[i][j] = sum of rectangle from (0,0) to (i-1,j-1)

Build:
prefix[i][j] = prefix[i-1][j] + prefix[i][j-1] - prefix[i-1][j-1] + matrix[i-1][j-1]

Query (r1,c1) to (r2,c2):
sum = prefix[r2+1][c2+1] - prefix[r1][c2+1] - prefix[r2+1][c1] + prefix[r1][c1]
```

### Example: Range Sum Query 2D - Immutable
```java
class NumMatrix {
    private int[][] prefix;

    public NumMatrix(int[][] matrix) {
        int m = matrix.length, n = matrix[0].length;
        prefix = new int[m + 1][n + 1];

        for (int i = 1; i <= m; i++) {
            for (int j = 1; j <= n; j++) {
                prefix[i][j] = prefix[i-1][j] + prefix[i][j-1]
                             - prefix[i-1][j-1] + matrix[i-1][j-1];
            }
        }
    }

    public int sumRegion(int r1, int c1, int r2, int c2) {
        return prefix[r2+1][c2+1] - prefix[r1][c2+1]
             - prefix[r2+1][c1] + prefix[r1][c1];
    }
}
```

---

## Quick Reference

| Pattern | Time Build | Time Query | Space | Key Insight |
|---------|------------|------------|-------|-------------|
| Basic Prefix Sum | O(n) | O(1) | O(n) | sum(l,r) = prefix[r+1] - prefix[l] |
| Prefix + HashMap | O(n) | - | O(n) | Check if (sum - k) seen before |
| Prefix Product | O(n) | - | O(1) | Left pass + right pass |
| Difference Array | O(n + q) | O(n) | O(n) | Range add = 2 point updates |
| 2D Prefix Sum | O(mn) | O(1) | O(mn) | Inclusion-exclusion principle |
