# Dynamic Programming Patterns

DP solves problems by breaking them into overlapping subproblems with optimal substructure.

---

## Pattern 1: Fibonacci Pattern (Linear DP)

### Core Idea
Current state depends on previous 1-2 states.

### When to Use
- Climbing stairs
- House robber
- Min cost climbing

### Mental Model
```
dp[i] = f(dp[i-1], dp[i-2])
can optimize to O(1) space with 2 variables
```

### Example: Climbing Stairs
```java
public int climbStairs(int n) {
    if (n <= 2) return n;

    int prev2 = 1, prev1 = 2;

    for (int i = 3; i <= n; i++) {
        int curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }

    return prev1;
}
```

### Example: House Robber
```java
public int rob(int[] nums) {
    if (nums.length == 1) return nums[0];

    int prev2 = nums[0];
    int prev1 = Math.max(nums[0], nums[1]);

    for (int i = 2; i < nums.length; i++) {
        int curr = Math.max(prev1, prev2 + nums[i]);
        prev2 = prev1;
        prev1 = curr;
    }

    return prev1;
}
```

### Example: House Robber II (Circular)
```java
public int rob(int[] nums) {
    if (nums.length == 1) return nums[0];

    // Either rob 0 to n-2 OR rob 1 to n-1
    return Math.max(
        robRange(nums, 0, nums.length - 2),
        robRange(nums, 1, nums.length - 1)
    );
}

private int robRange(int[] nums, int start, int end) {
    int prev2 = 0, prev1 = 0;

    for (int i = start; i <= end; i++) {
        int curr = Math.max(prev1, prev2 + nums[i]);
        prev2 = prev1;
        prev1 = curr;
    }

    return prev1;
}
```

---

## Pattern 2: Grid DP (2D Path)

### Core Idea
dp[i][j] depends on dp[i-1][j] and dp[i][j-1].

### When to Use
- Unique paths
- Minimum path sum
- Grid traversal

### Mental Model
```
dp[i][j] = dp[i-1][j] + dp[i][j-1]  (paths)
dp[i][j] = min(dp[i-1][j], dp[i][j-1]) + grid[i][j]  (min sum)
```

### Example: Unique Paths
```java
public int uniquePaths(int m, int n) {
    int[] dp = new int[n];
    Arrays.fill(dp, 1);

    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[j] += dp[j - 1];
        }
    }

    return dp[n - 1];
}
```

### Example: Minimum Path Sum
```java
public int minPathSum(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[] dp = new int[n];

    dp[0] = grid[0][0];
    for (int j = 1; j < n; j++) {
        dp[j] = dp[j - 1] + grid[0][j];
    }

    for (int i = 1; i < m; i++) {
        dp[0] += grid[i][0];
        for (int j = 1; j < n; j++) {
            dp[j] = Math.min(dp[j], dp[j - 1]) + grid[i][j];
        }
    }

    return dp[n - 1];
}
```

---

## Pattern 3: String DP (Two Strings)

### Core Idea
Compare characters from two strings. dp[i][j] represents subproblem for s1[0..i] and s2[0..j].

### When to Use
- Longest common subsequence
- Edit distance
- String matching

### Mental Model
```
if s1[i] == s2[j]:
    dp[i][j] = dp[i-1][j-1] + 1
else:
    dp[i][j] = max(dp[i-1][j], dp[i][j-1])
```

### Example: Longest Common Subsequence
```java
public int longestCommonSubsequence(String text1, String text2) {
    int m = text1.length(), n = text2.length();
    int[][] dp = new int[m + 1][n + 1];

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i - 1) == text2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }

    return dp[m][n];
}
```

---

## Pattern 4: Palindrome DP

### Core Idea
dp[i][j] = true if s[i..j] is palindrome.

### When to Use
- Longest palindromic substring
- Count palindromic substrings
- Palindrome partitioning

### Mental Model
```
dp[i][j] = (s[i] == s[j]) && dp[i+1][j-1]
expand from shorter to longer substrings
```

### Example: Longest Palindromic Substring
```java
public String longestPalindrome(String s) {
    int n = s.length();
    int start = 0, maxLen = 1;

    // Expand around center
    for (int i = 0; i < n; i++) {
        // Odd length
        int len1 = expandAroundCenter(s, i, i);
        // Even length
        int len2 = expandAroundCenter(s, i, i + 1);

        int len = Math.max(len1, len2);
        if (len > maxLen) {
            maxLen = len;
            start = i - (len - 1) / 2;
        }
    }

    return s.substring(start, start + maxLen);
}

private int expandAroundCenter(String s, int left, int right) {
    while (left >= 0 && right < s.length() && s.charAt(left) == s.charAt(right)) {
        left--;
        right++;
    }
    return right - left - 1;
}
```

---

## Pattern 5: Knapsack (0/1)

### Core Idea
For each item, choose to include or exclude. Can't reuse items.

### When to Use
- Subset sum
- Target sum
- Partition equal subset

### Mental Model
```
dp[i][w] = max(dp[i-1][w], dp[i-1][w-weight[i]] + value[i])
include item i OR exclude it
```

### Example: 0/1 Knapsack
```java
public int knapsack(int[] weights, int[] values, int capacity) {
    int n = weights.length;
    int[] dp = new int[capacity + 1];

    for (int i = 0; i < n; i++) {
        // Traverse backwards to avoid using same item twice
        for (int w = capacity; w >= weights[i]; w--) {
            dp[w] = Math.max(dp[w], dp[w - weights[i]] + values[i]);
        }
    }

    return dp[capacity];
}
```

### Example: Partition Equal Subset Sum
```java
public boolean canPartition(int[] nums) {
    int sum = Arrays.stream(nums).sum();
    if (sum % 2 != 0) return false;

    int target = sum / 2;
    boolean[] dp = new boolean[target + 1];
    dp[0] = true;

    for (int num : nums) {
        for (int j = target; j >= num; j--) {
            dp[j] = dp[j] || dp[j - num];
        }
    }

    return dp[target];
}
```

---

## Pattern 6: Unbounded Knapsack

### Core Idea
Can use items unlimited times.

### When to Use
- Coin change
- Rod cutting
- Combination sum

### Mental Model
```
dp[w] = max/min(dp[w], dp[w-weight[i]] + value[i])
traverse forwards (can reuse)
```

### Example: Coin Change
```java
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);
    dp[0] = 0;

    for (int coin : coins) {
        for (int j = coin; j <= amount; j++) {  // Forward (can reuse)
            dp[j] = Math.min(dp[j], dp[j - coin] + 1);
        }
    }

    return dp[amount] > amount ? -1 : dp[amount];
}
```

---

## Pattern 7: Longest Increasing Subsequence (LIS)

### Core Idea
For each element, find longest LIS ending at that element.

### When to Use
- Longest increasing subsequence
- Patience sorting

### Mental Model
```
O(n²): dp[i] = max(dp[j] + 1) for all j < i where nums[j] < nums[i]
O(n log n): binary search on tails array
```

### Example: LIS (O(n²))
```java
public int lengthOfLIS(int[] nums) {
    int[] dp = new int[nums.length];
    Arrays.fill(dp, 1);
    int maxLen = 1;

    for (int i = 1; i < nums.length; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
        maxLen = Math.max(maxLen, dp[i]);
    }

    return maxLen;
}
```

### Example: LIS (O(n log n))
```java
public int lengthOfLIS(int[] nums) {
    List<Integer> tails = new ArrayList<>();

    for (int num : nums) {
        int pos = Collections.binarySearch(tails, num);
        if (pos < 0) pos = -(pos + 1);

        if (pos == tails.size()) {
            tails.add(num);
        } else {
            tails.set(pos, num);
        }
    }

    return tails.size();
}
```

---

## Pattern 8: State Machine DP (Stock Problems)

### Core Idea
Track different states (holding, not holding, cooldown).

### When to Use
- Buy/sell stock with constraints
- State transitions

### Mental Model
```
hold[i] = max(hold[i-1], notHold[i-1] - price)
notHold[i] = max(notHold[i-1], hold[i-1] + price)
```

### Example: Buy/Sell Stock with Cooldown
```java
public int maxProfit(int[] prices) {
    int n = prices.length;
    if (n < 2) return 0;

    int hold = -prices[0];
    int notHold = 0;
    int cooldown = 0;

    for (int i = 1; i < n; i++) {
        int prevHold = hold;
        hold = Math.max(hold, cooldown - prices[i]);
        cooldown = notHold;
        notHold = Math.max(notHold, prevHold + prices[i]);
    }

    return notHold;
}
```

---

## Pattern 9: Word Break

### Core Idea
dp[i] = true if s[0..i-1] can be segmented.

### When to Use
- Word segmentation
- Dictionary problems

### Example: Word Break
```java
public boolean wordBreak(String s, List<String> wordDict) {
    Set<String> dict = new HashSet<>(wordDict);
    boolean[] dp = new boolean[s.length() + 1];
    dp[0] = true;

    for (int i = 1; i <= s.length(); i++) {
        for (int j = 0; j < i; j++) {
            if (dp[j] && dict.contains(s.substring(j, i))) {
                dp[i] = true;
                break;
            }
        }
    }

    return dp[s.length()];
}
```

---

## Pattern 10: Maximum Product Subarray

### Core Idea
Track both max and min at each position (negatives can flip).

### When to Use
- Product problems with negatives
- Need to track both extremes

### Example: Maximum Product Subarray
```java
public int maxProduct(int[] nums) {
    int maxProd = nums[0];
    int minProd = nums[0];
    int result = nums[0];

    for (int i = 1; i < nums.length; i++) {
        int temp = maxProd;
        maxProd = Math.max(nums[i], Math.max(maxProd * nums[i], minProd * nums[i]));
        minProd = Math.min(nums[i], Math.min(temp * nums[i], minProd * nums[i]));
        result = Math.max(result, maxProd);
    }

    return result;
}
```

---

## Quick Reference

| Pattern | State | Transition |
|---------|-------|------------|
| Fibonacci | dp[i] | dp[i-1] + dp[i-2] |
| Grid | dp[i][j] | dp[i-1][j] + dp[i][j-1] |
| Two Strings | dp[i][j] | Based on char match |
| Palindrome | dp[i][j] | dp[i+1][j-1] if match |
| 0/1 Knapsack | dp[w] | max(exclude, include) backward |
| Unbounded | dp[w] | max(exclude, include) forward |
| LIS | dp[i] | max(dp[j] + 1) for j < i |
| State Machine | hold, notHold | State transitions |

## DP Approach

1. **Define state**: What does dp[i] or dp[i][j] represent?
2. **Base cases**: What are the initial values?
3. **Recurrence relation**: How to compute dp[i] from smaller subproblems?
4. **Order of computation**: Which subproblems must be solved first?
5. **Space optimization**: Can we reduce from 2D to 1D?
