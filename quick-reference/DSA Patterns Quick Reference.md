# DSA Patterns Quick Reference

Mental models, recognition triggers, and bare-minimum templates for quick revision before an interview.

---

## Arrays & Hashing

**Mental model:** Use a hashmap to remember what you have seen so you can look it up in O(1) later.

**Spot it when:** "Find pair/triplet that sums to X", "check if duplicate exists", "group by property", "count frequency".

**Core template:**
```java
Map<Integer, Integer> map = new HashMap<>();
for (int num : arr) {
    if (map.containsKey(target - num)) return true;  // Two Sum
    map.put(num, map.getOrDefault(num, 0) + 1);      // Frequency
}
```

**Key tricks:**
- Two Sum: `map.containsKey(target - num)`
- Anagram: Sort string or use frequency array as key
- Subarray sum = K: Prefix sum + hashmap `map.get(prefixSum - k)`

---

## Sliding Window

**Mental model:** A window that slides across the array. Expand right to include more, shrink left when the window violates the constraint.

**Spot it when:** "Longest/shortest subarray/substring with constraint", "max/min in window", "all subarrays satisfying X".

**1. Fixed-size window:**
```java
int sum = 0;
for (int i = 0; i < k; i++) sum += arr[i];
int maxSum = sum;

for (int i = k; i < arr.length; i++) {
    sum += arr[i] - arr[i - k];  // Add right, remove left
    maxSum = Math.max(maxSum, sum);
}
```

**2. Variable window — longest with constraint (shrink when invalid):**
```java
// Longest substring without repeating characters
int left = 0, maxLen = 0;
Set<Character> seen = new HashSet<>();

for (int right = 0; right < s.length(); right++) {
    while (seen.contains(s.charAt(right)))
        seen.remove(s.charAt(left++));   // Shrink until valid
    seen.add(s.charAt(right));
    maxLen = Math.max(maxLen, right - left + 1);
}
```

**3. Variable window — shortest with constraint (shrink when valid):**
```java
// Minimum size subarray sum >= target
int left = 0, sum = 0, minLen = Integer.MAX_VALUE;

for (int right = 0; right < arr.length; right++) {
    sum += arr[right];
    while (sum >= target) {              // Shrink while still valid
        minLen = Math.min(minLen, right - left + 1);
        sum -= arr[left++];
    }
}
return minLen == Integer.MAX_VALUE ? 0 : minLen;
```

**4. Window with frequency map — minimum window substring:**
```java
Map<Character, Integer> need = new HashMap<>();
for (char c : t.toCharArray()) need.merge(c, 1, Integer::sum);

int left = 0, matched = 0, minLen = Integer.MAX_VALUE, start = 0;
Map<Character, Integer> window = new HashMap<>();

for (int right = 0; right < s.length(); right++) {
    char c = s.charAt(right);
    window.merge(c, 1, Integer::sum);
    if (need.containsKey(c) && window.get(c).equals(need.get(c))) matched++;

    while (matched == need.size()) {     // Window is valid, try to shrink
        if (right - left + 1 < minLen) { minLen = right - left + 1; start = left; }
        char leftChar = s.charAt(left++);
        window.merge(leftChar, -1, Integer::sum);
        if (need.containsKey(leftChar) && window.get(leftChar) < need.get(leftChar)) matched--;
    }
}
return minLen == Integer.MAX_VALUE ? "" : s.substring(start, start + minLen);
```

**5. Longest repeating character replacement:**
```java
// Key insight: window is valid if (windowSize - maxFreq) <= k
int left = 0, maxFreq = 0, maxLen = 0;
int[] count = new int[26];

for (int right = 0; right < s.length(); right++) {
    maxFreq = Math.max(maxFreq, ++count[s.charAt(right) - 'A']);
    while ((right - left + 1) - maxFreq > k)
        count[s.charAt(left++) - 'A']--;
    maxLen = Math.max(maxLen, right - left + 1);
}
```

**6. Sliding window maximum — monotonic deque:**
```java
Deque<Integer> deque = new ArrayDeque<>();  // Stores indices, front = max
int[] result = new int[nums.length - k + 1];

for (int i = 0; i < nums.length; i++) {
    while (!deque.isEmpty() && deque.peekFirst() < i - k + 1)
        deque.pollFirst();                   // Remove out-of-window indices
    while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i])
        deque.pollLast();                    // Remove smaller elements
    deque.offerLast(i);
    if (i >= k - 1) result[i - k + 1] = nums[deque.peekFirst()];
}
```

---

## Two Pointers

**Mental model:** One pointer from each end moving toward each other, or both moving in the same direction at different speeds.

**Spot it when:** Sorted array, "find pair", "remove duplicates in-place", "partition", "palindrome check".

**1. Opposite ends — find pair with target sum:**
```java
int left = 0, right = arr.length - 1;
while (left < right) {
    int sum = arr[left] + arr[right];
    if (sum == target) return true;
    else if (sum < target) left++;
    else right--;
}
```

**2. Same direction — remove duplicates in-place:**
```java
int slow = 0;
for (int fast = 1; fast < arr.length; fast++) {
    if (arr[fast] != arr[slow]) {
        arr[++slow] = arr[fast];  // Overwrite next position
    }
}
return slow + 1;  // New length
```

**3. Same direction — move zeroes / partition:**
```java
int slow = 0;
for (int fast = 0; fast < arr.length; fast++) {
    if (arr[fast] != 0) {
        arr[slow++] = arr[fast];  // Move non-zero to front
    }
}
while (slow < arr.length) arr[slow++] = 0;  // Fill rest with zeros
```

**4. 3Sum — fix one, two pointers on rest:**
```java
Arrays.sort(nums);
for (int i = 0; i < nums.length - 2; i++) {
    if (i > 0 && nums[i] == nums[i-1]) continue;  // Skip duplicate fixed element
    int left = i + 1, right = nums.length - 1;
    while (left < right) {
        int sum = nums[i] + nums[left] + nums[right];
        if (sum == 0) {
            result.add(Arrays.asList(nums[i], nums[left], nums[right]));
            while (left < right && nums[left] == nums[left+1]) left++;   // Skip dups
            while (left < right && nums[right] == nums[right-1]) right--;
            left++; right--;
        } else if (sum < 0) left++;
        else right--;
    }
}
```

**5. Container with most water:**
```java
int left = 0, right = height.length - 1, maxWater = 0;
while (left < right) {
    maxWater = Math.max(maxWater, Math.min(height[left], height[right]) * (right - left));
    if (height[left] < height[right]) left++;   // Move the shorter side
    else right--;
}
```

**6. Trapping rain water:**
```java
int left = 0, right = arr.length - 1;
int leftMax = 0, rightMax = 0, water = 0;
while (left < right) {
    if (arr[left] < arr[right]) {
        if (arr[left] >= leftMax) leftMax = arr[left];
        else water += leftMax - arr[left];
        left++;
    } else {
        if (arr[right] >= rightMax) rightMax = arr[right];
        else water += rightMax - arr[right];
        right--;
    }
}
```

---

## Binary Search

**Mental model:** Cut the search space in half every step. Works on sorted data or when you can check "is this answer too big or too small".

**Spot it when:** Sorted array, "find target", "find first/last occurrence", "search in rotated array", "minimize/maximize something".

**1. Standard — find exact target:**
```java
int left = 0, right = arr.length - 1;
while (left <= right) {
    int mid = left + (right - left) / 2;
    if (arr[mid] == target) return mid;
    else if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
}
return -1;
```

**2. First occurrence (leftmost index):**
```java
int left = 0, right = arr.length - 1, result = -1;
while (left <= right) {
    int mid = left + (right - left) / 2;
    if (arr[mid] == target) {
        result = mid;       // Record, but keep searching left
        right = mid - 1;
    } else if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
}
return result;
```

**3. Last occurrence (rightmost index):**
```java
int left = 0, right = arr.length - 1, result = -1;
while (left <= right) {
    int mid = left + (right - left) / 2;
    if (arr[mid] == target) {
        result = mid;       // Record, but keep searching right
        left = mid + 1;
    } else if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
}
return result;
```

**4. Lower bound — first index where arr[i] >= target:**
```java
int left = 0, right = arr.length;   // right = length, not length-1
while (left < right) {              // strict <
    int mid = left + (right - left) / 2;
    if (arr[mid] < target) left = mid + 1;
    else right = mid;               // mid could be the answer
}
return left;  // insertion point
```

**5. Upper bound — first index where arr[i] > target:**
```java
int left = 0, right = arr.length;
while (left < right) {
    int mid = left + (right - left) / 2;
    if (arr[mid] <= target) left = mid + 1;
    else right = mid;
}
return left;
```

**6. Search in rotated sorted array:**
```java
int left = 0, right = arr.length - 1;
while (left <= right) {
    int mid = left + (right - left) / 2;
    if (arr[mid] == target) return mid;

    if (arr[left] <= arr[mid]) {          // Left half is sorted
        if (arr[left] <= target && target < arr[mid])
            right = mid - 1;             // Target in left half
        else left = mid + 1;
    } else {                              // Right half is sorted
        if (arr[mid] < target && target <= arr[right])
            left = mid + 1;              // Target in right half
        else right = mid - 1;
    }
}
return -1;
```

**7. Find peak element:**
```java
int left = 0, right = arr.length - 1;
while (left < right) {
    int mid = left + (right - left) / 2;
    if (arr[mid] > arr[mid + 1]) right = mid;   // Peak is on left side
    else left = mid + 1;                         // Peak is on right side
}
return left;  // left == right == peak index
```

**8. Binary search on answer (minimize/maximize):**
```java
// "What is the minimum X such that condition(X) is true?"
int left = minPossible, right = maxPossible;
while (left < right) {
    int mid = left + (right - left) / 2;
    if (condition(mid)) right = mid;   // mid works, try smaller
    else left = mid + 1;               // mid too small, go bigger
}
return left;
// Examples: Koko eating bananas, capacity to ship packages, split array
```

**When to use `left <= right` vs `left < right`:**
- `left <= right` with `return mid`: when you know the exact target exists or want -1
- `left < right` with `return left`: when searching for a boundary (lower/upper bound, answer search)

---

## Stack

**Mental model:** Last in, first out. Use it when you need to remember things in reverse order, match pairs, or find the next greater/smaller element.

**Spot it when:** "Valid parentheses", "next greater element", "evaluate expression", "largest rectangle", "undo operations".

**1. Matching brackets:**
```java
Stack<Character> stack = new Stack<>();
for (char c : s.toCharArray()) {
    if (c == '(' || c == '[' || c == '{') stack.push(c);
    else {
        if (stack.isEmpty()) return false;
        char top = stack.pop();
        if (c == ')' && top != '(') return false;
        if (c == ']' && top != '[') return false;
        if (c == '}' && top != '{') return false;
    }
}
return stack.isEmpty();
```

**2. Monotonic stack — next greater element (from right):**
```java
int[] result = new int[arr.length];
Stack<Integer> stack = new Stack<>();  // Stores indices

for (int i = arr.length - 1; i >= 0; i--) {
    while (!stack.isEmpty() && arr[stack.peek()] <= arr[i])
        stack.pop();                          // Remove smaller elements
    result[i] = stack.isEmpty() ? -1 : arr[stack.peek()];
    stack.push(i);
}
// Use for: daily temperatures, next greater element
```

**3. Monotonic stack — next greater element (from left, online):**
```java
Stack<Integer> stack = new Stack<>();
for (int i = 0; i < arr.length; i++) {
    while (!stack.isEmpty() && arr[stack.peek()] < arr[i]) {
        int idx = stack.pop();
        result[idx] = arr[i];  // arr[i] is the next greater for idx
    }
    stack.push(i);
}
// Remaining in stack have no next greater → result stays -1
```

**4. Monotonic stack — largest rectangle in histogram:**
```java
Stack<Integer> stack = new Stack<>();
int maxArea = 0;
for (int i = 0; i <= heights.length; i++) {
    int h = (i == heights.length) ? 0 : heights[i];
    while (!stack.isEmpty() && heights[stack.peek()] > h) {
        int height = heights[stack.pop()];
        int width = stack.isEmpty() ? i : i - stack.peek() - 1;
        maxArea = Math.max(maxArea, height * width);
    }
    stack.push(i);
}
```

**5. Min stack — O(1) getMin:**
```java
Stack<Integer> stack = new Stack<>();
Stack<Integer> minStack = new Stack<>();  // Tracks minimums

void push(int val) {
    stack.push(val);
    if (minStack.isEmpty() || val <= minStack.peek())
        minStack.push(val);
}

void pop() {
    if (stack.pop().equals(minStack.peek()))
        minStack.pop();
}

int getMin() { return minStack.peek(); }
```

**6. Evaluate reverse polish notation:**
```java
Stack<Integer> stack = new Stack<>();
for (String token : tokens) {
    if ("+-*/".contains(token)) {
        int b = stack.pop(), a = stack.pop();
        switch (token) {
            case "+": stack.push(a + b); break;
            case "-": stack.push(a - b); break;
            case "*": stack.push(a * b); break;
            case "/": stack.push(a / b); break;
        }
    } else stack.push(Integer.parseInt(token));
}
return stack.pop();
```

---

## Linked List

**Mental model:** Nodes connected by pointers. Use dummy head to simplify edge cases. Use fast/slow pointers for cycle detection and finding middle.

**Spot it when:** "Reverse list", "detect cycle", "find middle", "merge lists", "remove node".

**1. Reverse — iterative:**
```java
ListNode prev = null, curr = head;
while (curr != null) {
    ListNode next = curr.next;
    curr.next = prev;
    prev = curr;
    curr = next;
}
return prev;
```

**2. Reverse — recursive:**
```java
ListNode reverse(ListNode head) {
    if (head == null || head.next == null) return head;
    ListNode newHead = reverse(head.next);
    head.next.next = head;   // Node after head points back to head
    head.next = null;        // Head's next becomes null (it's now the tail)
    return newHead;
}
```

**3. Fast/slow — find middle:**
```java
ListNode slow = head, fast = head;
while (fast != null && fast.next != null) {
    slow = slow.next;
    fast = fast.next.next;
}
return slow;  // Middle (for even length: second middle)
```

**4. Fast/slow — detect cycle:**
```java
ListNode slow = head, fast = head;
while (fast != null && fast.next != null) {
    slow = slow.next;
    fast = fast.next.next;
    if (slow == fast) return true;
}
return false;
```

**5. Fast/slow — find cycle start:**
```java
// After detecting meeting point, reset one pointer to head
// Both move one step at a time — they meet at cycle start
ListNode slow = head, fast = head;
while (fast != null && fast.next != null) {
    slow = slow.next; fast = fast.next.next;
    if (slow == fast) {
        slow = head;                    // Reset to head
        while (slow != fast) { slow = slow.next; fast = fast.next; }
        return slow;                    // Cycle start
    }
}
return null;
```

**6. Remove nth from end:**
```java
ListNode dummy = new ListNode(0);
dummy.next = head;
ListNode fast = dummy, slow = dummy;

for (int i = 0; i <= n; i++) fast = fast.next;  // Advance fast by n+1

while (fast != null) { slow = slow.next; fast = fast.next; }

slow.next = slow.next.next;  // Remove the node
return dummy.next;
```

**7. Merge two sorted lists:**
```java
ListNode dummy = new ListNode(0), curr = dummy;
while (l1 != null && l2 != null) {
    if (l1.val <= l2.val) { curr.next = l1; l1 = l1.next; }
    else { curr.next = l2; l2 = l2.next; }
    curr = curr.next;
}
curr.next = (l1 != null) ? l1 : l2;
return dummy.next;
```

**8. Reverse K groups:**
```java
ListNode reverseKGroup(ListNode head, int k) {
    ListNode curr = head;
    int count = 0;
    while (curr != null && count < k) { curr = curr.next; count++; }
    if (count < k) return head;          // Less than k nodes left, don't reverse

    ListNode prev = null, node = head;
    for (int i = 0; i < k; i++) {
        ListNode next = node.next;
        node.next = prev;
        prev = node;
        node = next;
    }
    head.next = reverseKGroup(node, k);  // head is now tail of reversed group
    return prev;                          // prev is new head
}
```

**Key tricks:**
- Always use dummy head when the head node might be removed or changed
- Intersection of two lists: advance the longer list by the length difference, then walk together

---

## Greedy

**Mental model:** Make the locally optimal choice at each step and hope it leads to a global optimum. Works when local choices do not affect future choices.

**Spot it when:** "Maximize/minimize something", "intervals", "scheduling", "jump game".

**Interval template:**
```java
Arrays.sort(intervals, (a, b) -> a[1] - b[1]);  // Sort by end time
int count = 0, lastEnd = Integer.MIN_VALUE;

for (int[] interval : intervals) {
    if (interval[0] >= lastEnd) {  // No overlap
        count++;
        lastEnd = interval[1];
    }
}
```

**Key tricks:**
- Jump game: Track max reachable index
- Gas station: If total gas >= total cost, solution exists. Start from where surplus begins.
- Meeting rooms: Sort by start, check if any overlap

---

## Intervals

**Mental model:** Sort by start time, then merge or check overlaps.

**Spot it when:** "Merge intervals", "find free slots", "meeting rooms", "insert interval".

**Merge template:**
```java
Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
List<int[]> merged = new ArrayList<>();
int[] current = intervals[0];

for (int i = 1; i < intervals.length; i++) {
    if (intervals[i][0] <= current[1]) {  // Overlap
        current[1] = Math.max(current[1], intervals[i][1]);  // Extend
    } else {
        merged.add(current);
        current = intervals[i];
    }
}
merged.add(current);
```

**Key tricks:**
- Meeting rooms II: Min heap of end times, count max overlaps
- Insert interval: Add before, merge overlapping, add after

---

## Backtracking

**Mental model:** Try all possibilities recursively. If a path does not work, backtrack and try another.

**Spot it when:** "Generate all permutations/combinations/subsets", "solve puzzle", "word search in grid".

**Template:**
```java
void backtrack(List<Integer> current, int start) {
    if (isComplete(current)) {
        result.add(new ArrayList<>(current));
        return;
    }
    
    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);           // Choose
        backtrack(current, i + 1);      // Explore
        current.remove(current.size() - 1);  // Unchoose
    }
}
```

**Key tricks:**
- Permutations: Use a `used` array or swap elements
- Combinations: Pass `start` index to avoid duplicates
- Subsets: Include or exclude each element
- Avoid duplicates: Sort first, skip `if (i > start && nums[i] == nums[i-1])`

---

## Dynamic Programming

**Mental model:** If you are solving the same subproblem multiple times, store the answer. Bottom-up or top-down with memoization.

**Spot it when:** "Maximize/minimize", "count ways", "longest/shortest", "can you reach", overlapping subproblems, optimal substructure.

**Top-down (memoization) — easier to think, recursive:**
```java
Map<Integer, Integer> memo = new HashMap<>();

int solve(int i) {
    if (i == base) return baseValue;
    if (memo.containsKey(i)) return memo.get(i);
    int result = /* recurrence using solve(i-1), solve(i-2), etc. */;
    memo.put(i, result);
    return result;
}
```

**Bottom-up (tabulation) — faster, no recursion stack:**
```java
int[] dp = new int[n + 1];
dp[0] = base0;
dp[1] = base1;
for (int i = 2; i <= n; i++) {
    dp[i] = /* recurrence using dp[i-1], dp[i-2], etc. */;
}
return dp[n];
```

**Sub-pattern 1 — Linear DP (Fibonacci-like):**
```java
// Climbing stairs, min cost stairs, decode ways
dp[i] = dp[i-1] + dp[i-2];                          // Count ways
dp[i] = Math.min(dp[i-1], dp[i-2]) + cost[i];       // Min cost
```

**Sub-pattern 2 — Rob or skip:**
```java
// House robber, max sum non-adjacent
dp[i] = Math.max(dp[i-1], dp[i-2] + nums[i]);
// dp[i-1] = skip current, dp[i-2] + nums[i] = take current
```

**Sub-pattern 3 — 0/1 Knapsack:**
```java
// For each item, choose to take or skip
int[][] dp = new int[n+1][W+1];
for (int i = 1; i <= n; i++) {
    for (int w = 0; w <= W; w++) {
        dp[i][w] = dp[i-1][w];                              // Skip item i
        if (w >= weight[i])
            dp[i][w] = Math.max(dp[i][w], dp[i-1][w-weight[i]] + value[i]); // Take
    }
}
```

**Sub-pattern 4 — Unbounded Knapsack:**
```java
// Coin change, rod cutting — can reuse items
int[] dp = new int[amount + 1];
Arrays.fill(dp, Integer.MAX_VALUE);
dp[0] = 0;
for (int i = 1; i <= amount; i++) {
    for (int coin : coins) {
        if (i >= coin && dp[i - coin] != Integer.MAX_VALUE)
            dp[i] = Math.min(dp[i], dp[i - coin] + 1);
    }
}
```

**Sub-pattern 5 — Two sequences (LCS family):**
```java
// LCS, edit distance, longest common substring
int[][] dp = new int[m+1][n+1];
for (int i = 1; i <= m; i++) {
    for (int j = 1; j <= n; j++) {
        if (s1[i-1] == s2[j-1])
            dp[i][j] = dp[i-1][j-1] + 1;           // Match
        else
            dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]); // Skip one
    }
}
// Edit distance: dp[i][j] = min(dp[i-1][j]+1, dp[i][j-1]+1, dp[i-1][j-1] + (s1[i]!=s2[j]?1:0))
```

**Sub-pattern 6 — Grid DP:**
```java
// Unique paths, min path sum, dungeon game
int[][] dp = new int[m][n];
dp[0][0] = grid[0][0];
for (int i = 1; i < m; i++) dp[i][0] = dp[i-1][0] + grid[i][0];
for (int j = 1; j < n; j++) dp[0][j] = dp[0][j-1] + grid[0][j];
for (int i = 1; i < m; i++)
    for (int j = 1; j < n; j++)
        dp[i][j] = Math.min(dp[i-1][j], dp[i][j-1]) + grid[i][j];
```

**Sub-pattern 7 — Interval DP:**
```java
// Burst balloons, matrix chain multiplication, palindrome partitioning
for (int len = 2; len <= n; len++) {
    for (int i = 0; i <= n - len; i++) {
        int j = i + len - 1;
        for (int k = i; k < j; k++) {
            dp[i][j] = Math.max(dp[i][j], dp[i][k] + dp[k+1][j] + cost(i,k,j));
        }
    }
}
```

**Sub-pattern 8 — State machine DP (stock problems):**
```java
// Best time to buy/sell stock with cooldown
int hold = -prices[0], notHold = 0, cooldown = 0;
for (int i = 1; i < prices.length; i++) {
    int newHold = Math.max(hold, notHold - prices[i]);
    int newNotHold = Math.max(notHold, cooldown + prices[i]);
    int newCooldown = notHold;
    hold = newHold; notHold = newNotHold; cooldown = newCooldown;
}
return Math.max(notHold, cooldown);
```

**Sub-pattern 9 — LIS (Longest Increasing Subsequence):**
```java
// O(n^2)
int[] dp = new int[n];
Arrays.fill(dp, 1);
for (int i = 1; i < n; i++)
    for (int j = 0; j < i; j++)
        if (nums[j] < nums[i]) dp[i] = Math.max(dp[i], dp[j] + 1);

// O(n log n) patience sort
List<Integer> tails = new ArrayList<>();
for (int num : nums) {
    int pos = Collections.binarySearch(tails, num);
    if (pos < 0) pos = -(pos + 1);
    if (pos == tails.size()) tails.add(num);
    else tails.set(pos, num);
}
return tails.size();
```

**Sub-pattern 10 — Kadane's (max subarray):**
```java
int maxSoFar = nums[0], maxEndingHere = nums[0];
for (int i = 1; i < nums.length; i++) {
    maxEndingHere = Math.max(nums[i], maxEndingHere + nums[i]);
    maxSoFar = Math.max(maxSoFar, maxEndingHere);
}
// For max product: track both min and max (negative × negative = positive)

---

## Trees

**Mental model:** Recursion is your friend. Most tree problems are DFS. The key is knowing where to put the logic — before, between, or after the recursive calls.

**Spot it when:** "Traverse tree", "find path", "check property", "construct tree", "BST operations".

**The three DFS orderings — where you put the logic changes everything:**

**Preorder — process node BEFORE children (root → left → right):**
```java
void preorder(TreeNode node) {
    if (node == null) return;
    process(node);          // ← logic HERE: see node before subtrees
    preorder(node.left);
    preorder(node.right);
}
// Use for: serialize tree, copy tree, print directory structure
```

**Inorder — process node BETWEEN children (left → root → right):**
```java
void inorder(TreeNode node) {
    if (node == null) return;
    inorder(node.left);
    process(node);          // ← logic HERE: left subtree already done
    inorder(node.right);
}
// Use for: BST gives sorted order, kth smallest, validate BST
```

**Postorder — process node AFTER children (left → right → root):**
```java
void postorder(TreeNode node) {
    if (node == null) return;
    postorder(node.left);
    postorder(node.right);
    process(node);          // ← logic HERE: both subtrees already done
}
// Use for: delete tree, compute height, evaluate expression tree,
//          any problem where you need subtree results before current node
```

**The "return value up" pattern (postorder thinking):**
```java
// When you need info from subtrees to compute current node's answer
int height(TreeNode node) {
    if (node == null) return 0;
    int left = height(node.left);    // Get left subtree result
    int right = height(node.right);  // Get right subtree result
    return 1 + Math.max(left, right); // Combine and return up
}
// Same pattern for: diameter, max path sum, check balanced
```

**The "pass value down" pattern (preorder thinking):**
```java
// When you need info from ancestors to process current node
void validate(TreeNode node, long min, long max) {
    if (node == null) return;
    if (node.val <= min || node.val >= max) { valid = false; return; }
    validate(node.left, min, node.val);   // Pass constraint down
    validate(node.right, node.val, max);
}
// Same pattern for: path sum, level tracking, root-to-leaf paths
```

**BFS — level order traversal:**
```java
Queue<TreeNode> queue = new LinkedList<>();
queue.offer(root);

while (!queue.isEmpty()) {
    int size = queue.size();           // Snapshot of current level size
    for (int i = 0; i < size; i++) {
        TreeNode node = queue.poll();
        process(node);
        if (node.left != null) queue.offer(node.left);
        if (node.right != null) queue.offer(node.right);
    }
    // After inner loop: one full level processed
}
// Use for: level order, right side view, zigzag traversal, min depth
```

**BST-specific patterns:**

Inorder successor (next larger node):
```java
// If node has right child: go right, then all the way left
// If no right child: go up until you make a left turn
TreeNode successor(TreeNode root, TreeNode target) {
    TreeNode result = null;
    while (root != null) {
        if (target.val < root.val) {
            result = root;          // root could be successor
            root = root.left;
        } else {
            root = root.right;
        }
    }
    return result;
}
```

LCA in BST:
```java
TreeNode lca(TreeNode root, TreeNode p, TreeNode q) {
    if (p.val < root.val && q.val < root.val) return lca(root.left, p, q);
    if (p.val > root.val && q.val > root.val) return lca(root.right, p, q);
    return root;  // They split here, or one equals root
}
```

LCA in binary tree (not BST):
```java
TreeNode lca(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) return root;
    TreeNode left = lca(root.left, p, q);
    TreeNode right = lca(root.right, p, q);
    if (left != null && right != null) return root;  // p and q on different sides
    return left != null ? left : right;
}
```

**Key tricks:**
- Max depth: postorder — `1 + max(left, right)`
- Diameter: postorder — `max(diameter, leftHeight + rightHeight)` with global max
- Max path sum: postorder — return max single path up, track global max including both sides
- Balanced tree: postorder — return -1 if unbalanced, height otherwise
- Construct from preorder + inorder: preorder[0] is root, find it in inorder to split left/right

---

## Graphs

**Mental model:** Nodes and edges. DFS for exploring paths, BFS for shortest path, Union-Find for connectivity, topological sort for dependencies.

**Spot it when:** "Connected components", "shortest path", "detect cycle", "topological sort", "islands".

**1. DFS — explore all reachable nodes:**
```java
boolean[] visited = new boolean[n];

void dfs(int node, List<List<Integer>> graph) {
    visited[node] = true;
    for (int neighbor : graph.get(node)) {
        if (!visited[neighbor]) dfs(neighbor, graph);
    }
}
// Use for: connected components, path existence, flood fill
```

**2. BFS — shortest path in unweighted graph:**
```java
int[] dist = new int[n];
Arrays.fill(dist, -1);
Queue<Integer> queue = new LinkedList<>();
queue.offer(start);
dist[start] = 0;

while (!queue.isEmpty()) {
    int node = queue.poll();
    for (int neighbor : graph.get(node)) {
        if (dist[neighbor] == -1) {
            dist[neighbor] = dist[node] + 1;
            queue.offer(neighbor);
        }
    }
}
// Use for: shortest path, word ladder, rotting oranges
```

**3. Cycle detection in undirected graph (DFS with parent):**
```java
boolean hasCycle(int node, int parent, boolean[] visited, List<List<Integer>> graph) {
    visited[node] = true;
    for (int neighbor : graph.get(node)) {
        if (!visited[neighbor]) {
            if (hasCycle(neighbor, node, visited, graph)) return true;
        } else if (neighbor != parent) {  // Visited and not parent = cycle
            return true;
        }
    }
    return false;
}
```

**4. Cycle detection in directed graph (DFS with recursion stack):**
```java
boolean[] visited = new boolean[n];
boolean[] inStack = new boolean[n];  // Currently in DFS path

boolean hasCycle(int node, List<List<Integer>> graph) {
    visited[node] = true;
    inStack[node] = true;
    for (int neighbor : graph.get(node)) {
        if (!visited[neighbor] && hasCycle(neighbor, graph)) return true;
        if (inStack[neighbor]) return true;  // Back edge = cycle
    }
    inStack[node] = false;  // Remove from current path
    return false;
}
```

**5. Topological sort — Kahn's algorithm (BFS):**
```java
int[] indegree = new int[n];
for (int u = 0; u < n; u++)
    for (int v : graph.get(u)) indegree[v]++;

Queue<Integer> queue = new LinkedList<>();
for (int i = 0; i < n; i++)
    if (indegree[i] == 0) queue.offer(i);  // Start with nodes with no dependencies

List<Integer> order = new ArrayList<>();
while (!queue.isEmpty()) {
    int node = queue.poll();
    order.add(node);
    for (int neighbor : graph.get(node)) {
        if (--indegree[neighbor] == 0) queue.offer(neighbor);
    }
}
// If order.size() != n, there's a cycle
// Use for: course schedule, build order, task dependencies
```

**6. Topological sort — DFS:**
```java
boolean[] visited = new boolean[n];
Stack<Integer> stack = new Stack<>();

void dfs(int node, List<List<Integer>> graph) {
    visited[node] = true;
    for (int neighbor : graph.get(node))
        if (!visited[neighbor]) dfs(neighbor, graph);
    stack.push(node);  // Push AFTER all neighbors processed
}

// Call dfs for all unvisited nodes, then pop stack for order
```

**7. Dijkstra — shortest path in weighted graph:**
```java
int[] dist = new int[n];
Arrays.fill(dist, Integer.MAX_VALUE);
dist[start] = 0;

PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
pq.offer(new int[]{0, start});  // {distance, node}

while (!pq.isEmpty()) {
    int[] curr = pq.poll();
    int d = curr[0], node = curr[1];
    if (d > dist[node]) continue;  // Stale entry, skip

    for (int[] edge : graph.get(node)) {  // {neighbor, weight}
        int neighbor = edge[0], weight = edge[1];
        if (dist[node] + weight < dist[neighbor]) {
            dist[neighbor] = dist[node] + weight;
            pq.offer(new int[]{dist[neighbor], neighbor});
        }
    }
}
// Use for: network delay time, cheapest flights (with K stops variant)
```

**8. Union-Find — connectivity and cycle detection:**
```java
int[] parent = new int[n];
int[] rank = new int[n];
for (int i = 0; i < n; i++) parent[i] = i;

int find(int x) {
    if (parent[x] != x) parent[x] = find(parent[x]);  // Path compression
    return parent[x];
}

boolean union(int x, int y) {
    int px = find(x), py = find(y);
    if (px == py) return false;  // Already connected = cycle
    if (rank[px] < rank[py]) parent[px] = py;
    else if (rank[px] > rank[py]) parent[py] = px;
    else { parent[py] = px; rank[px]++; }
    return true;
}
// Use for: redundant connection, accounts merge, number of provinces
```

**9. Multi-source BFS:**
```java
// Start BFS from multiple sources simultaneously
Queue<int[]> queue = new LinkedList<>();
for (each source cell) {
    queue.offer(new int[]{row, col});
    visited[row][col] = true;
}
// Then standard BFS — all sources expand at same time
// Use for: rotting oranges, 01 matrix (distance to nearest 0)
```

**Key tricks:**
- Number of islands: DFS/BFS on each unvisited `'1'` cell, mark all connected cells visited
- Word ladder: BFS where each word is a node, edges connect words differing by one letter
- Pacific Atlantic: DFS from both borders inward, find cells reachable from both

---

## Backtracking

**Mental model:** Try all possibilities recursively. If a path does not work, undo the last choice and try another. The key is: choose, explore, unchoose.

**Spot it when:** "Generate all permutations/combinations/subsets", "solve puzzle", "word search in grid".

**The universal template:**
```java
void backtrack(state, choices) {
    if (isComplete(state)) { result.add(copy of state); return; }
    for (each choice) {
        if (shouldPrune(choice)) continue;
        makeChoice(state, choice);      // Choose
        backtrack(state, nextChoices);  // Explore
        undoChoice(state, choice);      // Unchoose
    }
}
```

**1. Subsets — include or exclude each element:**
```java
void subsets(int[] nums, int start, List<Integer> current) {
    result.add(new ArrayList<>(current));  // Every state is a valid subset
    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);
        subsets(nums, i + 1, current);
        current.remove(current.size() - 1);
    }
}
```

**2. Combinations — pick exactly k elements:**
```java
void combine(int[] nums, int start, int k, List<Integer> current) {
    if (current.size() == k) { result.add(new ArrayList<>(current)); return; }
    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);
        combine(nums, i + 1, k, current);
        current.remove(current.size() - 1);
    }
}
```

**3. Permutations — all orderings:**
```java
void permute(int[] nums, boolean[] used, List<Integer> current) {
    if (current.size() == nums.length) { result.add(new ArrayList<>(current)); return; }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;
        used[i] = true;
        current.add(nums[i]);
        permute(nums, used, current);
        current.remove(current.size() - 1);
        used[i] = false;
    }
}
```

**4. Combination sum — reuse elements:**
```java
void combinationSum(int[] candidates, int start, int remaining, List<Integer> current) {
    if (remaining == 0) { result.add(new ArrayList<>(current)); return; }
    for (int i = start; i < candidates.length; i++) {
        if (candidates[i] > remaining) break;  // Pruning (sort first)
        current.add(candidates[i]);
        combinationSum(candidates, i, remaining - candidates[i], current);  // i not i+1 (reuse)
        current.remove(current.size() - 1);
    }
}
```

**5. Avoid duplicates — sort + skip same value at same level:**
```java
Arrays.sort(nums);
void backtrack(int start, List<Integer> current) {
    result.add(new ArrayList<>(current));
    for (int i = start; i < nums.length; i++) {
        if (i > start && nums[i] == nums[i-1]) continue;  // Skip duplicate at same level
        current.add(nums[i]);
        backtrack(i + 1, current);
        current.remove(current.size() - 1);
    }
}
```

**6. Word search in grid:**
```java
boolean dfs(char[][] board, String word, int i, int j, int k) {
    if (k == word.length()) return true;
    if (i < 0 || i >= m || j < 0 || j >= n || board[i][j] != word.charAt(k)) return false;

    char temp = board[i][j];
    board[i][j] = '#';  // Mark visited

    boolean found = dfs(board, word, i+1, j, k+1) || dfs(board, word, i-1, j, k+1)
                 || dfs(board, word, i, j+1, k+1) || dfs(board, word, i, j-1, k+1);

    board[i][j] = temp;  // Unmark
    return found;
}

---

## Dynamic Programming

**Mental model:** If you are solving the same subproblem multiple times, store the answer. Bottom-up or top-down with memoization.

**Spot it when:** "Maximize/minimize", "count ways", "longest/shortest", "can you reach", overlapping subproblems, optimal substructure.

**Top-down (memoization) — easier to think, recursive:**
```java
Map<Integer, Integer> memo = new HashMap<>();

int solve(int i) {
    if (i == base) return baseValue;
    if (memo.containsKey(i)) return memo.get(i);
    int result = /* recurrence using solve(i-1), solve(i-2), etc. */;
    memo.put(i, result);
    return result;
}
```

**Bottom-up (tabulation) — faster, no recursion stack:**
```java
int[] dp = new int[n + 1];
dp[0] = base0;
dp[1] = base1;
for (int i = 2; i <= n; i++) {
    dp[i] = /* recurrence using dp[i-1], dp[i-2], etc. */;
}
return dp[n];
```

**Sub-pattern 1 — Linear DP (Fibonacci-like):**
```java
// Climbing stairs, min cost stairs, decode ways
dp[i] = dp[i-1] + dp[i-2];                          // Count ways
dp[i] = Math.min(dp[i-1], dp[i-2]) + cost[i];       // Min cost
```

**Sub-pattern 2 — Rob or skip:**
```java
// House robber, max sum non-adjacent
dp[i] = Math.max(dp[i-1], dp[i-2] + nums[i]);
// dp[i-1] = skip current, dp[i-2] + nums[i] = take current
```

**Sub-pattern 3 — 0/1 Knapsack:**
```java
// For each item, choose to take or skip
int[][] dp = new int[n+1][W+1];
for (int i = 1; i <= n; i++) {
    for (int w = 0; w <= W; w++) {
        dp[i][w] = dp[i-1][w];                              // Skip item i
        if (w >= weight[i])
            dp[i][w] = Math.max(dp[i][w], dp[i-1][w-weight[i]] + value[i]); // Take
    }
}
```

**Sub-pattern 4 — Unbounded Knapsack:**
```java
// Coin change, rod cutting — can reuse items
int[] dp = new int[amount + 1];
Arrays.fill(dp, Integer.MAX_VALUE);
dp[0] = 0;
for (int i = 1; i <= amount; i++) {
    for (int coin : coins) {
        if (i >= coin && dp[i - coin] != Integer.MAX_VALUE)
            dp[i] = Math.min(dp[i], dp[i - coin] + 1);
    }
}
```

**Sub-pattern 5 — Two sequences (LCS family):**
```java
// LCS, edit distance, longest common substring
int[][] dp = new int[m+1][n+1];
for (int i = 1; i <= m; i++) {
    for (int j = 1; j <= n; j++) {
        if (s1[i-1] == s2[j-1])
            dp[i][j] = dp[i-1][j-1] + 1;           // Match
        else
            dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]); // Skip one
    }
}
// Edit distance: dp[i][j] = min(dp[i-1][j]+1, dp[i][j-1]+1, dp[i-1][j-1] + (s1[i]!=s2[j]?1:0))
```

**Sub-pattern 6 — Grid DP:**
```java
// Unique paths, min path sum, dungeon game
int[][] dp = new int[m][n];
dp[0][0] = grid[0][0];
// Fill first row and column
for (int i = 1; i < m; i++) dp[i][0] = dp[i-1][0] + grid[i][0];
for (int j = 1; j < n; j++) dp[0][j] = dp[0][j-1] + grid[0][j];
// Fill rest
for (int i = 1; i < m; i++)
    for (int j = 1; j < n; j++)
        dp[i][j] = Math.min(dp[i-1][j], dp[i][j-1]) + grid[i][j];
```

**Sub-pattern 7 — Interval DP:**
```java
// Burst balloons, matrix chain multiplication, palindrome partitioning
// Fill by increasing length of interval
for (int len = 2; len <= n; len++) {
    for (int i = 0; i <= n - len; i++) {
        int j = i + len - 1;
        for (int k = i; k < j; k++) {
            dp[i][j] = Math.max(dp[i][j], dp[i][k] + dp[k+1][j] + cost(i,k,j));
        }
    }
}
```

**Sub-pattern 8 — State machine DP (stock problems):**
```java
// Best time to buy/sell stock with cooldown
int hold = -prices[0], notHold = 0, cooldown = 0;
for (int i = 1; i < prices.length; i++) {
    int newHold = Math.max(hold, notHold - prices[i]);  // Buy today or keep holding
    int newNotHold = Math.max(notHold, cooldown + prices[i]); // Sell today or stay
    int newCooldown = notHold;                           // Was not holding yesterday
    hold = newHold; notHold = newNotHold; cooldown = newCooldown;
}
return Math.max(notHold, cooldown);
```

**Sub-pattern 9 — LIS (Longest Increasing Subsequence):**
```java
// O(n^2) DP
int[] dp = new int[n];
Arrays.fill(dp, 1);
for (int i = 1; i < n; i++)
    for (int j = 0; j < i; j++)
        if (nums[j] < nums[i]) dp[i] = Math.max(dp[i], dp[j] + 1);

// O(n log n) patience sort
List<Integer> tails = new ArrayList<>();
for (int num : nums) {
    int pos = Collections.binarySearch(tails, num);
    if (pos < 0) pos = -(pos + 1);
    if (pos == tails.size()) tails.add(num);
    else tails.set(pos, num);
}
return tails.size();
```

**Sub-pattern 10 — Kadane's (max subarray):**
```java
int maxSoFar = nums[0], maxEndingHere = nums[0];
for (int i = 1; i < nums.length; i++) {
    maxEndingHere = Math.max(nums[i], maxEndingHere + nums[i]);
    maxSoFar = Math.max(maxSoFar, maxEndingHere);
}
// For max product: track both min and max (negative × negative = positive)
```

---

## Heap (Priority Queue)

**Mental model:** Always gives you the min or max element in O(log n). Use it when you need "top K", "Kth largest/smallest", or "always process the smallest/largest next".

**Spot it when:** "Top K elements", "Kth largest", "merge K sorted", "median from stream", "always pick minimum cost next".

**1. Min heap — Kth largest (keep K largest, peek is Kth):**
```java
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
for (int num : arr) {
    minHeap.offer(num);
    if (minHeap.size() > k) minHeap.poll();  // Evict smallest
}
return minHeap.peek();  // Smallest of the top K = Kth largest
```

**2. Max heap — Kth smallest (keep K smallest, peek is Kth):**
```java
PriorityQueue<Integer> maxHeap = new PriorityQueue<>((a, b) -> b - a);
for (int num : arr) {
    maxHeap.offer(num);
    if (maxHeap.size() > k) maxHeap.poll();  // Evict largest
}
return maxHeap.peek();  // Largest of the bottom K = Kth smallest
```

**3. Top K frequent elements:**
```java
Map<Integer, Integer> freq = new HashMap<>();
for (int num : nums) freq.merge(num, 1, Integer::sum);

PriorityQueue<Integer> minHeap = new PriorityQueue<>(Comparator.comparingInt(freq::get));
for (int num : freq.keySet()) {
    minHeap.offer(num);
    if (minHeap.size() > k) minHeap.poll();
}
return minHeap.stream().mapToInt(i -> i).toArray();
```

**4. Merge K sorted lists:**
```java
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
// Add first element of each list: {value, listIndex, elementIndex}
for (int i = 0; i < lists.length; i++)
    if (lists[i] != null) pq.offer(new int[]{lists[i].val, i, 0});

ListNode dummy = new ListNode(0), curr = dummy;
while (!pq.isEmpty()) {
    int[] top = pq.poll();
    curr.next = new ListNode(top[0]);
    curr = curr.next;
    // Add next element from same list
    ListNode next = lists[top[1]].next;
    if (next != null) pq.offer(new int[]{next.val, top[1], top[2]+1});
}
```

**5. Median from data stream — two heaps:**
```java
PriorityQueue<Integer> lower = new PriorityQueue<>((a, b) -> b - a);  // Max heap (lower half)
PriorityQueue<Integer> upper = new PriorityQueue<>();                   // Min heap (upper half)

void addNum(int num) {
    lower.offer(num);
    upper.offer(lower.poll());          // Balance: push max of lower to upper
    if (lower.size() < upper.size())
        lower.offer(upper.poll());      // Keep lower >= upper in size
}

double findMedian() {
    if (lower.size() > upper.size()) return lower.peek();
    return (lower.peek() + upper.peek()) / 2.0;
}
```

**6. K closest points to origin:**
```java
// Max heap of size K — evict farthest
PriorityQueue<int[]> maxHeap = new PriorityQueue<>(
    (a, b) -> (b[0]*b[0]+b[1]*b[1]) - (a[0]*a[0]+a[1]*a[1])
);
for (int[] point : points) {
    maxHeap.offer(point);
    if (maxHeap.size() > k) maxHeap.poll();
}
return maxHeap.toArray(new int[0][]);
```

---

## Trie (Prefix Tree)

**Mental model:** A tree where each path from root to node represents a string. Use it for prefix matching.

**Spot it when:** "Autocomplete", "word search with prefix", "longest common prefix", "dictionary operations".

**Template:**
```java
class TrieNode {
    Map<Character, TrieNode> children = new HashMap<>();
    boolean isEnd = false;
}

class Trie {
    private TrieNode root = new TrieNode();
    
    public void insert(String word) {
        TrieNode node = root;
        for (char c : word.toCharArray()) {
            node.children.putIfAbsent(c, new TrieNode());
            node = node.children.get(c);
        }
        node.isEnd = true;
    }
    
    public boolean search(String word) {
        TrieNode node = root;
        for (char c : word.toCharArray()) {
            if (!node.children.containsKey(c)) return false;
            node = node.children.get(c);
        }
        return node.isEnd;
    }
    
    public boolean startsWith(String prefix) {
        TrieNode node = root;
        for (char c : prefix.toCharArray()) {
            if (!node.children.containsKey(c)) return false;
            node = node.children.get(c);
        }
        return true;
    }
}
```

**Key tricks:**
- Word search II: Build trie of words, DFS on grid pruning with trie
- Longest common prefix: Traverse trie until branching

---

## Bit Manipulation

**Mental model:** Work directly with binary representation. XOR cancels duplicates, AND checks bits, shifts multiply/divide by 2.

**Spot it when:** "Find single number", "count set bits", "power of two", "XOR tricks".

**Common operations:**
```java
n & (n - 1)           // Remove rightmost set bit
n & 1                 // Check if odd
n ^ n                 // = 0 (XOR cancels)
a ^ b ^ a             // = b (XOR is commutative)
1 << k                // 2^k
n >> 1                // n / 2
n << 1                // n * 2
```

**Key tricks:**
- Single number: XOR all elements, duplicates cancel out
- Power of two: `n > 0 && (n & (n-1)) == 0`
- Count set bits: Loop `n = n & (n-1)` until n is 0
- Sum without +: Use XOR for sum, AND for carry, shift carry left, repeat

---

## Prefix Sum

**Mental model:** Precompute cumulative sums so you can answer range queries in O(1).

**Spot it when:** "Sum of subarray [i, j]", "range queries", "subarray sum equals K".

**Template:**
```java
int[] prefix = new int[arr.length + 1];
for (int i = 0; i < arr.length; i++) {
    prefix[i + 1] = prefix[i] + arr[i];
}

// Sum from i to j (inclusive)
int sum = prefix[j + 1] - prefix[i];
```

**Key tricks:**
- Subarray sum = K: `map.get(prefixSum - k)` counts how many subarrays end here with sum K
- 2D prefix sum: `prefix[i][j] = prefix[i-1][j] + prefix[i][j-1] - prefix[i-1][j-1] + matrix[i-1][j-1]`

---

## Matrix

**Mental model:** Treat as a 2D array. Use direction arrays for neighbors. Mark visited cells.

**Spot it when:** "Traverse matrix", "spiral order", "rotate", "set zeroes", "search in matrix".

**Direction arrays:**
```java
int[][] dirs = {{0,1}, {1,0}, {0,-1}, {-1,0}};  // Right, Down, Left, Up
for (int[] dir : dirs) {
    int newRow = row + dir[0];
    int newCol = col + dir[1];
    if (isValid(newRow, newCol)) {
        // Process neighbor
    }
}
```

**Key tricks:**
- Spiral matrix: Four boundary pointers (top, bottom, left, right), move inward
- Rotate 90 degrees: Transpose then reverse each row
- Set matrix zeroes: Use first row/col as markers

---

## Pattern Recognition Cheat Sheet

| If problem says... | Think... |
|-------------------|----------|
| "Find pair that sums to X" | Hashmap (Two Sum) |
| "Longest substring with constraint" | Sliding window |
| "Sorted array, find X" | Binary search |
| "Next greater element" | Monotonic stack |
| "Detect cycle in linked list" | Fast/slow pointers |
| "Reverse linked list" | Three pointers |
| "Merge K sorted" | Min heap |
| "Generate all permutations" | Backtracking |
| "Count ways to reach" | DP |
| "Maximize/minimize with choices" | DP or Greedy |
| "Shortest path unweighted" | BFS |
| "Connected components" | DFS or Union-Find |
| "Topological sort" | Kahn's algorithm (BFS) or DFS |
| "Intervals overlap" | Sort by start/end |
| "Prefix matching" | Trie |
| "Single number in duplicates" | XOR |

---

## Time Complexity Quick Reference

| Operation | Complexity | When to use |
|-----------|------------|-------------|
| HashMap get/put | O(1) | Fast lookup |
| HashSet add/contains | O(1) | Check existence |
| Sorting | O(n log n) | Preprocess for binary search or two pointers |
| Binary search | O(log n) | Sorted data |
| Heap push/pop | O(log n) | Top K, priority |
| DFS/BFS | O(V + E) | Graph traversal |
| Union-Find | O(α(n)) ≈ O(1) | Connectivity |
| Trie insert/search | O(m) | m = word length |

---

## Common Pitfalls

- **Off-by-one errors:** Use `<=` vs `<` carefully. Draw out small examples.
- **Integer overflow:** Use `long` for sums or products. Check `mid = left + (right - left) / 2` in binary search.
- **Modifying while iterating:** Copy the list or use indices.
- **Forgetting base cases:** Always handle empty input, single element, null.
- **Not handling duplicates:** Sort first, skip duplicates explicitly.
- **Stack/queue empty:** Always check `!stack.isEmpty()` before `pop()` or `peek()`.
- **Graph visited array:** Initialize correctly, mark visited before adding to queue (BFS).

---

## Interview Strategy

1. **Clarify first:** Ask about input size, constraints, edge cases (empty, duplicates, negatives).
2. **Brute force first:** State the naive O(n^2) or O(n^3) solution even if you know better. Shows you understand the problem.
3. **Optimize:** Identify bottleneck. Can you use a hashmap? Binary search? DP?
4. **Code:** Write clean code with good variable names. Handle edge cases.
5. **Test:** Walk through a small example. Check edge cases (empty, single element, all same).
6. **Complexity:** State time and space complexity clearly.
