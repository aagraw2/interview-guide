# Two Pointers Patterns

Two pointers reduce O(n²) to O(n) by intelligently moving pointers based on current state.

---

## Pattern 1: Opposite Direction (Two Ends)

### Core Idea
Start pointers at both ends, move inward based on comparison.

### When to Use
- Sorted array
- Find pair with target sum
- Palindrome checking
- Container problems (maximize area)

### Mental Model
```
left = 0, right = n-1
while left < right:
    if condition met → found answer
    if need larger → left++
    if need smaller → right--
```

### Generic Template
```java
public int[] twoPointerOpposite(int[] nums, int target) {
    int left = 0, right = nums.length - 1;

    while (left < right) {
        int sum = nums[left] + nums[right];

        if (sum == target) {
            return new int[] { left, right };
        } else if (sum < target) {
            left++;  // Need larger sum
        } else {
            right--;  // Need smaller sum
        }
    }

    return new int[] {};
}
```

### Example: Valid Palindrome
```java
public boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;

    while (left < right) {
        // Skip non-alphanumeric
        while (left < right && !Character.isLetterOrDigit(s.charAt(left))) {
            left++;
        }
        while (left < right && !Character.isLetterOrDigit(s.charAt(right))) {
            right--;
        }

        if (Character.toLowerCase(s.charAt(left)) !=
            Character.toLowerCase(s.charAt(right))) {
            return false;
        }

        left++;
        right--;
    }

    return true;
}
```

### Example: Container With Most Water
```java
public int maxArea(int[] height) {
    int left = 0, right = height.length - 1;
    int maxWater = 0;

    while (left < right) {
        int h = Math.min(height[left], height[right]);
        int w = right - left;
        maxWater = Math.max(maxWater, h * w);

        // Move the shorter line (it's the bottleneck)
        if (height[left] < height[right]) {
            left++;
        } else {
            right--;
        }
    }

    return maxWater;
}
```

---

## Pattern 2: Same Direction (Fast & Slow)

### Core Idea
Both pointers move in same direction at different speeds or conditions.

### When to Use
- Remove duplicates in-place
- Partition array
- Move elements to specific positions
- Cycle detection

### Mental Model
```
slow = position where next valid element goes
fast = scans through array

for each element at fast:
    if valid → place at slow, slow++
```

### Generic Template
```java
public int removeElements(int[] nums, int condition) {
    int slow = 0;

    for (int fast = 0; fast < nums.length; fast++) {
        if (/* nums[fast] is valid */) {
            nums[slow] = nums[fast];
            slow++;
        }
    }

    return slow;  // New length
}
```

### Example: Move Zeroes
```java
public void moveZeroes(int[] nums) {
    int slow = 0;  // Position for next non-zero

    for (int fast = 0; fast < nums.length; fast++) {
        if (nums[fast] != 0) {
            // Swap to maintain relative order
            int temp = nums[slow];
            nums[slow] = nums[fast];
            nums[fast] = temp;
            slow++;
        }
    }
}
```

### Example: Remove Duplicates from Sorted Array
```java
public int removeDuplicates(int[] nums) {
    if (nums.length == 0) return 0;

    int slow = 1;  // First element always kept

    for (int fast = 1; fast < nums.length; fast++) {
        if (nums[fast] != nums[fast - 1]) {
            nums[slow] = nums[fast];
            slow++;
        }
    }

    return slow;
}
```

---

## Pattern 3: Three Pointers (Dutch National Flag)

### Core Idea
Partition array into three sections using three pointers.

### When to Use
- Sort array with 3 distinct values
- Partition into low/mid/high sections
- Three-way partitioning

### Mental Model
```
low = boundary for 0s (everything before is 0)
mid = current element being examined
high = boundary for 2s (everything after is 2)

while mid <= high:
    if nums[mid] == 0 → swap with low, low++, mid++
    if nums[mid] == 1 → mid++
    if nums[mid] == 2 → swap with high, high--
```

### Example: Sort Colors
```java
public void sortColors(int[] nums) {
    int low = 0, mid = 0, high = nums.length - 1;

    while (mid <= high) {
        if (nums[mid] == 0) {
            swap(nums, low, mid);
            low++;
            mid++;
        } else if (nums[mid] == 1) {
            mid++;
        } else {  // nums[mid] == 2
            swap(nums, mid, high);
            high--;
            // Don't increment mid - need to check swapped element
        }
    }
}

private void swap(int[] nums, int i, int j) {
    int temp = nums[i];
    nums[i] = nums[j];
    nums[j] = temp;
}
```

---

## Pattern 4: K-Sum (Fix One + Two Pointers)

### Core Idea
Fix one element, use two pointers on remaining sorted array.

### When to Use
- 3Sum, 4Sum problems
- Find triplets/quadruplets with target sum
- Need to avoid duplicates

### Mental Model
```
sort array
for each element i:
    use two pointers on remaining array [i+1, n-1]
    skip duplicates
```

### Example: 3Sum
```java
public List<List<Integer>> threeSum(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(nums);

    for (int i = 0; i < nums.length - 2; i++) {
        // Skip duplicates for first element
        if (i > 0 && nums[i] == nums[i - 1]) continue;

        int left = i + 1, right = nums.length - 1;
        int target = -nums[i];

        while (left < right) {
            int sum = nums[left] + nums[right];

            if (sum == target) {
                result.add(Arrays.asList(nums[i], nums[left], nums[right]));

                // Skip duplicates
                while (left < right && nums[left] == nums[left + 1]) left++;
                while (left < right && nums[right] == nums[right - 1]) right--;

                left++;
                right--;
            } else if (sum < target) {
                left++;
            } else {
                right--;
            }
        }
    }

    return result;
}
```

---

## Pattern 5: Merge Two Sorted Arrays

### Core Idea
Compare elements from two arrays, pick smaller/larger and advance that pointer.

### When to Use
- Merge sorted arrays
- Squares of sorted array
- Find intersection

### Mental Model
```
while both pointers valid:
    compare elements
    pick appropriate one
    advance that pointer
handle remaining elements
```

### Example: Squares of a Sorted Array
```java
public int[] sortedSquares(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    int left = 0, right = n - 1;
    int pos = n - 1;  // Fill from end (largest first)

    while (left <= right) {
        int leftSq = nums[left] * nums[left];
        int rightSq = nums[right] * nums[right];

        if (leftSq > rightSq) {
            result[pos] = leftSq;
            left++;
        } else {
            result[pos] = rightSq;
            right--;
        }
        pos--;
    }

    return result;
}
```

---

## Pattern 6: Two Sequences (Subsequence Check)

### Core Idea
Use one pointer per sequence, advance based on matches.

### When to Use
- Check if string is subsequence
- Backspace string compare
- Compare two sequences

### Mental Model
```
i for sequence 1, j for sequence 2
while both valid:
    if match → advance both
    else → advance only one (based on problem)
```

### Example: Is Subsequence
```java
public boolean isSubsequence(String s, String t) {
    int i = 0, j = 0;

    while (i < s.length() && j < t.length()) {
        if (s.charAt(i) == t.charAt(j)) {
            i++;  // Matched, move to next char in s
        }
        j++;  // Always move through t
    }

    return i == s.length();  // All chars in s matched
}
```

### Example: Backspace String Compare
```java
public boolean backspaceCompare(String s, String t) {
    int i = s.length() - 1;
    int j = t.length() - 1;

    while (i >= 0 || j >= 0) {
        i = getNextValidIndex(s, i);
        j = getNextValidIndex(t, j);

        if (i >= 0 && j >= 0) {
            if (s.charAt(i) != t.charAt(j)) return false;
        } else if (i >= 0 || j >= 0) {
            return false;  // One string exhausted
        }

        i--;
        j--;
    }

    return true;
}

private int getNextValidIndex(String s, int index) {
    int skip = 0;
    while (index >= 0) {
        if (s.charAt(index) == '#') {
            skip++;
            index--;
        } else if (skip > 0) {
            skip--;
            index--;
        } else {
            break;
        }
    }
    return index;
}
```

---

## Pattern 7: Trapping Rain Water

### Core Idea
Water at position = min(maxLeft, maxRight) - height. Use two pointers with running max.

### When to Use
- Trapping rain water
- Need left max and right max at each position

### Mental Model
```
track leftMax and rightMax
process from smaller side (guaranteed water level)
```

### Example: Trapping Rain Water
```java
public int trap(int[] height) {
    int left = 0, right = height.length - 1;
    int leftMax = 0, rightMax = 0;
    int water = 0;

    while (left < right) {
        if (height[left] < height[right]) {
            if (height[left] >= leftMax) {
                leftMax = height[left];
            } else {
                water += leftMax - height[left];
            }
            left++;
        } else {
            if (height[right] >= rightMax) {
                rightMax = height[right];
            } else {
                water += rightMax - height[right];
            }
            right--;
        }
    }

    return water;
}
```

---

## Quick Reference

| Pattern | Pointers | Direction | Key Insight |
|---------|----------|-----------|-------------|
| Opposite Ends | 2 | Inward | Sorted array, target sum |
| Fast & Slow | 2 | Same | In-place modification |
| Dutch Flag | 3 | Both | Three-way partition |
| K-Sum | 1 fixed + 2 | Inward | Sort + fix + two pointers |
| Merge Arrays | 2 | Same/opposite | Compare and pick |
| Subsequence | 2 (1 per seq) | Same | Match advances both |
| Rain Water | 2 | Inward | Process smaller side |

## When to Use Two Pointers

- Sorted array
- Pairs/triplets with sum condition
- Palindrome checking
- In-place array modification
- Partitioning
- Merging sorted sequences
