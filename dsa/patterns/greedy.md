# Greedy Patterns

Greedy algorithms make locally optimal choices at each step, hoping to find global optimum.

---

## Pattern 1: Local Maximum at Each Step

### Core Idea
At each step, make the choice that seems best at that moment.

### When to Use
- Can prove local optimal leads to global optimal
- No need to reconsider past choices
- Problem has optimal substructure

### Mental Model
```
for each step:
    choose the best option available now
    commit to that choice
    move to next step
```

### Example: Jump Game (Can Reach End)
```java
public boolean canJump(int[] nums) {
    int maxReach = 0;

    for (int i = 0; i < nums.length; i++) {
        if (i > maxReach) {
            return false;  // Can't reach this position
        }
        maxReach = Math.max(maxReach, i + nums[i]);
    }

    return true;
}
```

### Example: Jump Game II (Min Jumps)
```java
public int jump(int[] nums) {
    int jumps = 0;
    int currentEnd = 0;
    int farthest = 0;

    for (int i = 0; i < nums.length - 1; i++) {
        farthest = Math.max(farthest, i + nums[i]);

        if (i == currentEnd) {
            jumps++;
            currentEnd = farthest;
        }
    }

    return jumps;
}
```

---

## Pattern 2: Sort Then Greedy

### Core Idea
Sort input first to enable greedy processing.

### When to Use
- Interval problems
- Assignment problems
- Need to process in specific order

### Mental Model
```
sort by start/end/size/value
process in sorted order
make greedy choice based on current state
```

### Example: Assign Cookies
```java
public int findContentChildren(int[] greed, int[] cookies) {
    Arrays.sort(greed);
    Arrays.sort(cookies);

    int child = 0, cookie = 0;

    while (child < greed.length && cookie < cookies.length) {
        if (cookies[cookie] >= greed[child]) {
            child++;  // Child satisfied
        }
        cookie++;  // Try next cookie
    }

    return child;
}
```

---

## Pattern 3: Interval Scheduling (Sort by End Time)

### Core Idea
Sort by end time. Greedily pick intervals that end earliest.

### When to Use
- Maximum non-overlapping intervals
- Minimum intervals to remove
- Activity selection

### Mental Model
```
sort by end time
for each interval:
    if doesn't overlap with last selected → select it
```

### Example: Non-overlapping Intervals
```java
public int eraseOverlapIntervals(int[][] intervals) {
    if (intervals.length == 0) return 0;

    // Sort by end time
    Arrays.sort(intervals, (a, b) -> a[1] - b[1]);

    int count = 0;
    int prevEnd = intervals[0][1];

    for (int i = 1; i < intervals.length; i++) {
        if (intervals[i][0] < prevEnd) {
            count++;  // Overlaps, must remove
        } else {
            prevEnd = intervals[i][1];  // No overlap, update end
        }
    }

    return count;
}
```

### Example: Minimum Arrows to Burst Balloons
```java
public int findMinArrowShots(int[][] points) {
    if (points.length == 0) return 0;

    // Sort by end position
    Arrays.sort(points, (a, b) -> Integer.compare(a[1], b[1]));

    int arrows = 1;
    int arrowPos = points[0][1];

    for (int i = 1; i < points.length; i++) {
        if (points[i][0] > arrowPos) {
            arrows++;
            arrowPos = points[i][1];
        }
    }

    return arrows;
}
```

---

## Pattern 4: Two-Pass Greedy

### Core Idea
Make one pass in each direction, combining results.

### When to Use
- Need to consider both neighbors
- Constraints from both sides
- Candy distribution

### Mental Model
```
Pass 1: left to right, satisfy left constraint
Pass 2: right to left, satisfy right constraint
combine results
```

### Example: Candy
```java
public int candy(int[] ratings) {
    int n = ratings.length;
    int[] candies = new int[n];
    Arrays.fill(candies, 1);

    // Left to right: satisfy left neighbor constraint
    for (int i = 1; i < n; i++) {
        if (ratings[i] > ratings[i - 1]) {
            candies[i] = candies[i - 1] + 1;
        }
    }

    // Right to left: satisfy right neighbor constraint
    for (int i = n - 2; i >= 0; i--) {
        if (ratings[i] > ratings[i + 1]) {
            candies[i] = Math.max(candies[i], candies[i + 1] + 1);
        }
    }

    int total = 0;
    for (int c : candies) {
        total += c;
    }
    return total;
}
```

---

## Pattern 5: Gas Station (Circular Greedy)

### Core Idea
If total gas >= total cost, solution exists. Find starting point using greedy.

### When to Use
- Circular route problems
- Find valid starting point

### Mental Model
```
if totalGas >= totalCost → solution exists
track current tank
if tank < 0 → start from next position
```

### Example: Gas Station
```java
public int canCompleteCircuit(int[] gas, int[] cost) {
    int totalGas = 0, totalCost = 0;
    int tank = 0;
    int start = 0;

    for (int i = 0; i < gas.length; i++) {
        totalGas += gas[i];
        totalCost += cost[i];
        tank += gas[i] - cost[i];

        if (tank < 0) {
            start = i + 1;
            tank = 0;
        }
    }

    return (totalGas >= totalCost) ? start : -1;
}
```

---

## Pattern 6: Partition Labels

### Core Idea
Track last occurrence of each character. Extend partition to include all occurrences.

### When to Use
- Partition with constraints
- Each element appears in exactly one partition

### Example: Partition Labels
```java
public List<Integer> partitionLabels(String s) {
    int[] lastIndex = new int[26];

    // Record last occurrence of each character
    for (int i = 0; i < s.length(); i++) {
        lastIndex[s.charAt(i) - 'a'] = i;
    }

    List<Integer> result = new ArrayList<>();
    int start = 0, end = 0;

    for (int i = 0; i < s.length(); i++) {
        end = Math.max(end, lastIndex[s.charAt(i) - 'a']);

        if (i == end) {
            result.add(end - start + 1);
            start = i + 1;
        }
    }

    return result;
}
```

---

## Pattern 7: Task Scheduler

### Core Idea
Process most frequent tasks first with required gaps.

### When to Use
- Scheduling with cooldown
- Maximize throughput

### Example: Task Scheduler
```java
public int leastInterval(char[] tasks, int n) {
    int[] freq = new int[26];
    for (char task : tasks) {
        freq[task - 'A']++;
    }

    int maxFreq = 0;
    int maxCount = 0;
    for (int f : freq) {
        if (f > maxFreq) {
            maxFreq = f;
            maxCount = 1;
        } else if (f == maxFreq) {
            maxCount++;
        }
    }

    // Formula: (maxFreq - 1) * (n + 1) + maxCount
    int intervals = (maxFreq - 1) * (n + 1) + maxCount;

    return Math.max(intervals, tasks.length);
}
```

---

## When Greedy Works

### Proof Techniques
1. **Exchange argument**: Show swapping greedy choice with any other doesn't improve
2. **Stays ahead**: Show greedy is always at least as good at each step
3. **Structural**: Show problem has greedy choice property and optimal substructure

### Common Greedy Problems
- Activity selection → sort by end time
- Fractional knapsack → sort by value/weight ratio
- Huffman coding → always merge smallest
- Minimum spanning tree → always pick smallest edge

---

## Quick Reference

| Pattern | Sort By | Greedy Choice |
|---------|---------|---------------|
| Jump Game | - | Track max reachable |
| Assign Cookies | Size | Smallest cookie to smallest greed |
| Interval Schedule | End time | Pick earliest ending |
| Two-Pass | - | Left pass + right pass |
| Gas Station | - | Start from where tank >= 0 |
| Partition Labels | - | Extend to last occurrence |
| Task Scheduler | Frequency | Process most frequent first |

## When NOT to Use Greedy

- 0/1 Knapsack (use DP)
- Longest path in graph
- Problems where local optimal != global optimal
- When need to explore multiple possibilities
