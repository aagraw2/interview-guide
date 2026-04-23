# Sliding Window Patterns

Sliding window optimizes brute-force O(n²) subarray/substring problems to O(n) by maintaining a moving window.

---

## Pattern 1: Fixed Size Window

### Core Idea
Window size is constant. Slide by adding one element and removing one element.

### When to Use
- "Find max/min/average of all subarrays of size k"
- Window size given in problem
- Each window has same size

### Mental Model
```
1. Build first window of size k
2. Slide: add right element, remove left element
3. Update result at each position
```

### Generic Template
```java
public int fixedSizeWindow(int[] nums, int k) {
    int windowSum = 0;
    int result = 0;

    for (int i = 0; i < nums.length; i++) {
        // Add right element
        windowSum += nums[i];

        // Window not yet full
        if (i < k - 1) continue;

        // Update result
        result = Math.max(result, windowSum);

        // Remove left element
        windowSum -= nums[i - k + 1];
    }

    return result;
}
```

### Example: Maximum Average Subarray I
```java
public double findMaxAverage(int[] nums, int k) {
    int windowSum = 0;

    // Build first window
    for (int i = 0; i < k; i++) {
        windowSum += nums[i];
    }

    int maxSum = windowSum;

    // Slide window
    for (int i = k; i < nums.length; i++) {
        windowSum += nums[i] - nums[i - k];
        maxSum = Math.max(maxSum, windowSum);
    }

    return (double) maxSum / k;
}
```

---

## Pattern 2: Variable Size Window (Expand + Shrink)

### Core Idea
Expand right pointer, shrink left pointer when condition breaks. Maintain valid window.

### When to Use
- "Longest/shortest subarray/substring with condition"
- No fixed window size
- Condition involves uniqueness, sum, frequency

### Mental Model
```
for every right:
    include element (expand)
    while window invalid:
        remove from left (shrink)
    update answer
```

### Generic Template
```java
public int variableSizeWindow(String s) {
    int left = 0;
    int result = 0;
    Map<Character, Integer> map = new HashMap<>();

    for (int right = 0; right < s.length(); right++) {
        char r = s.charAt(right);

        // 1. Expand: add right element
        map.put(r, map.getOrDefault(r, 0) + 1);

        // 2. Shrink: while window is invalid
        while (/* window invalid condition */) {
            char l = s.charAt(left);
            map.put(l, map.get(l) - 1);
            if (map.get(l) == 0) {
                map.remove(l);
            }
            left++;
        }

        // 3. Update result
        result = Math.max(result, right - left + 1);
    }

    return result;
}
```

### Example: Longest Substring Without Repeating Characters
```java
public int lengthOfLongestSubstring(String s) {
    int left = 0;
    int maxLength = 0;
    Set<Character> set = new HashSet<>();

    for (int right = 0; right < s.length(); right++) {
        char r = s.charAt(right);

        // Shrink while duplicate exists
        while (set.contains(r)) {
            set.remove(s.charAt(left));
            left++;
        }

        // Expand
        set.add(r);

        maxLength = Math.max(maxLength, right - left + 1);
    }

    return maxLength;
}
```

---

## Pattern 3: At Most K Distinct Elements

### Core Idea
Maintain window with at most K distinct elements. Shrink when exceeding K.

### When to Use
- "Longest subarray with at most K distinct elements"
- "Fruit into baskets" (K=2)
- Counting subarrays with at most K distinct

### Mental Model
```
for every right:
    add element to frequency map
    while distinctCount > k:
        remove from left
    update answer
```

### Example: Fruit Into Baskets (At Most 2 Types)
```java
public int totalFruit(int[] fruits) {
    Map<Integer, Integer> basket = new HashMap<>();
    int left = 0;
    int maxFruits = 0;

    for (int right = 0; right < fruits.length; right++) {
        // Add fruit
        basket.put(fruits[right], basket.getOrDefault(fruits[right], 0) + 1);

        // Shrink while more than 2 types
        while (basket.size() > 2) {
            int leftFruit = fruits[left];
            basket.put(leftFruit, basket.get(leftFruit) - 1);
            if (basket.get(leftFruit) == 0) {
                basket.remove(leftFruit);
            }
            left++;
        }

        maxFruits = Math.max(maxFruits, right - left + 1);
    }

    return maxFruits;
}
```

---

## Pattern 4: Window with Max Frequency Tracking

### Core Idea
Track maximum frequency in window. Window is valid if (length - maxFreq <= k).

### When to Use
- "Longest repeating character replacement"
- Can make K modifications to achieve goal
- Need to track most frequent element

### Mental Model
```
window is valid when:
    windowSize - maxFrequency <= k
    (can replace all non-max chars)
```

### Example: Longest Repeating Character Replacement
```java
public int characterReplacement(String s, int k) {
    int[] count = new int[26];
    int left = 0;
    int maxFreq = 0;
    int maxLength = 0;

    for (int right = 0; right < s.length(); right++) {
        count[s.charAt(right) - 'A']++;
        maxFreq = Math.max(maxFreq, count[s.charAt(right) - 'A']);

        // Window size - maxFreq > k means invalid
        // We need to replace more than k characters
        while (right - left + 1 - maxFreq > k) {
            count[s.charAt(left) - 'A']--;
            left++;
        }

        maxLength = Math.max(maxLength, right - left + 1);
    }

    return maxLength;
}
```

---

## Pattern 5: Minimum Window Substring

### Core Idea
Expand until all required characters found, then shrink to find minimum valid window.

### When to Use
- "Minimum window containing all characters"
- Need all elements of target in window
- Find shortest valid window

### Mental Model
```
expand until window contains all required
while window is valid:
    update minimum
    shrink from left
```

### Example: Minimum Window Substring
```java
public String minWindow(String s, String t) {
    if (t.length() > s.length()) return "";

    Map<Character, Integer> need = new HashMap<>();
    for (char c : t.toCharArray()) {
        need.put(c, need.getOrDefault(c, 0) + 1);
    }

    Map<Character, Integer> window = new HashMap<>();
    int have = 0, required = need.size();
    int left = 0;
    int minLen = Integer.MAX_VALUE;
    int minStart = 0;

    for (int right = 0; right < s.length(); right++) {
        char r = s.charAt(right);
        window.put(r, window.getOrDefault(r, 0) + 1);

        if (need.containsKey(r) && window.get(r).equals(need.get(r))) {
            have++;
        }

        // Shrink while valid
        while (have == required) {
            if (right - left + 1 < minLen) {
                minLen = right - left + 1;
                minStart = left;
            }

            char l = s.charAt(left);
            window.put(l, window.get(l) - 1);
            if (need.containsKey(l) && window.get(l) < need.get(l)) {
                have--;
            }
            left++;
        }
    }

    return minLen == Integer.MAX_VALUE ? "" : s.substring(minStart, minStart + minLen);
}
```

---

## Pattern 6: Counting Subarrays (Product/Sum Less Than K)

### Core Idea
Each valid window ending at right contributes (right - left + 1) subarrays.

### When to Use
- "Count subarrays with product/sum < k"
- Need to count, not find length
- Each window position counts differently

### Mental Model
```
for each right:
    shrink left while condition violated
    count += (right - left + 1)
    // All subarrays ending at right starting from left to right
```

### Example: Subarray Product Less Than K
```java
public int numSubarrayProductLessThanK(int[] nums, int k) {
    if (k <= 1) return 0;

    int product = 1;
    int left = 0;
    int count = 0;

    for (int right = 0; right < nums.length; right++) {
        product *= nums[right];

        while (product >= k) {
            product /= nums[left];
            left++;
        }

        // All subarrays ending at right
        count += right - left + 1;
    }

    return count;
}
```

---

## Pattern 7: Sliding Window with Target Sum

### Core Idea
Find minimum length subarray with sum >= target.

### When to Use
- Minimum size subarray with sum condition
- Positive numbers only (for shrinking to work)

### Example: Minimum Size Subarray Sum
```java
public int minSubArrayLen(int target, int[] nums) {
    int left = 0;
    int sum = 0;
    int minLen = Integer.MAX_VALUE;

    for (int right = 0; right < nums.length; right++) {
        sum += nums[right];

        while (sum >= target) {
            minLen = Math.min(minLen, right - left + 1);
            sum -= nums[left];
            left++;
        }
    }

    return minLen == Integer.MAX_VALUE ? 0 : minLen;
}
```

---

## Quick Reference

| Pattern | Type | Key Condition | Result |
|---------|------|---------------|--------|
| Fixed Size | Fixed | Window size = k | Max/min/avg of k-sized windows |
| Variable (Expand/Shrink) | Variable | While invalid, shrink | Longest valid window |
| At Most K Distinct | Variable | map.size() <= k | Longest with <= k types |
| Max Frequency | Variable | len - maxFreq <= k | Longest after k replacements |
| Minimum Window | Variable | Contains all required | Shortest containing target |
| Count Subarrays | Variable | Condition < k | Count = sum of (r - l + 1) |
| Target Sum | Variable | sum >= target | Shortest sum >= target |

## When to Use Sliding Window

- Array or string
- Contiguous subarray/substring
- Looking for longest/shortest/count
- Can optimize O(n²) brute force to O(n)
- Monotonic condition (if valid at size k, may/may not be valid at k+1)
