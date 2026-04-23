# Binary Search Patterns

Binary search halves the search space each iteration for O(log n) time. Master the boundary conditions.

---

## Pattern 1: Classic Binary Search (Find Exact)

### Core Idea
Search for exact target in sorted array. Return index or -1.

### When to Use
- Sorted array
- Find exact element
- O(log n) required

### Mental Model
```
while left <= right:
    mid = left + (right - left) / 2
    if nums[mid] == target → found
    if nums[mid] < target → search right half
    if nums[mid] > target → search left half
```

### Generic Template
```java
public int binarySearch(int[] nums, int target) {
    int left = 0, right = nums.length - 1;

    while (left <= right) {
        int mid = left + (right - left) / 2;  // Avoid overflow

        if (nums[mid] == target) {
            return mid;
        } else if (nums[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }

    return -1;  // Not found
}
```

---

## Pattern 2: Find Left Boundary (First Occurrence)

### Core Idea
Find leftmost position where target could be inserted (lower bound).

### When to Use
- Find first occurrence of target
- Find insertion position
- Count elements less than target

### Mental Model
```
when found, don't return - continue searching left
result is 'left' after loop
```

### Generic Template
```java
public int findLeftBoundary(int[] nums, int target) {
    int left = 0, right = nums.length;  // Note: right = length

    while (left < right) {
        int mid = left + (right - left) / 2;

        if (nums[mid] < target) {
            left = mid + 1;
        } else {
            right = mid;  // Could be answer, keep searching left
        }
    }

    return left;  // First position >= target
}
```

### Example: Search Insert Position
```java
public int searchInsert(int[] nums, int target) {
    int left = 0, right = nums.length;

    while (left < right) {
        int mid = left + (right - left) / 2;

        if (nums[mid] < target) {
            left = mid + 1;
        } else {
            right = mid;
        }
    }

    return left;
}
```

---

## Pattern 3: Find Right Boundary (Last Occurrence)

### Core Idea
Find rightmost position of target.

### When to Use
- Find last occurrence of target
- Count elements <= target

### Mental Model
```
when found, continue searching right
result is 'right' after loop (or left - 1)
```

### Generic Template
```java
public int findRightBoundary(int[] nums, int target) {
    int left = 0, right = nums.length;

    while (left < right) {
        int mid = left + (right - left) / 2;

        if (nums[mid] <= target) {
            left = mid + 1;  // Could be answer, keep searching right
        } else {
            right = mid;
        }
    }

    return left - 1;  // Last position <= target
}
```

---

## Pattern 4: Search in Rotated Sorted Array

### Core Idea
One half is always sorted. Determine which half and check if target is in sorted half.

### When to Use
- Array rotated at some pivot
- Find target or minimum

### Mental Model
```
find mid
determine which half is sorted (compare with left/right)
check if target is in sorted half → search there
else → search other half
```

### Example: Search in Rotated Sorted Array
```java
public int search(int[] nums, int target) {
    int left = 0, right = nums.length - 1;

    while (left <= right) {
        int mid = left + (right - left) / 2;

        if (nums[mid] == target) {
            return mid;
        }

        // Left half is sorted
        if (nums[left] <= nums[mid]) {
            if (target >= nums[left] && target < nums[mid]) {
                right = mid - 1;  // Target in left half
            } else {
                left = mid + 1;   // Target in right half
            }
        }
        // Right half is sorted
        else {
            if (target > nums[mid] && target <= nums[right]) {
                left = mid + 1;   // Target in right half
            } else {
                right = mid - 1;  // Target in left half
            }
        }
    }

    return -1;
}
```

### Example: Find Minimum in Rotated Sorted Array
```java
public int findMin(int[] nums) {
    int left = 0, right = nums.length - 1;

    while (left < right) {
        int mid = left + (right - left) / 2;

        if (nums[mid] > nums[right]) {
            // Minimum is in right half
            left = mid + 1;
        } else {
            // Minimum is in left half (including mid)
            right = mid;
        }
    }

    return nums[left];
}
```

---

## Pattern 5: Search on Answer Space

### Core Idea
Binary search on possible answer values, not array indices. Check if answer is feasible.

### When to Use
- "Minimum maximum" or "Maximum minimum" problems
- Answer has monotonic feasibility
- Can verify if answer works in O(n)

### Mental Model
```
left = minimum possible answer
right = maximum possible answer

while left < right:
    mid = candidate answer
    if canAchieve(mid):
        right = mid  // Try smaller (or larger)
    else:
        left = mid + 1
```

### Example: Koko Eating Bananas
```java
public int minEatingSpeed(int[] piles, int h) {
    int left = 1;
    int right = Arrays.stream(piles).max().getAsInt();

    while (left < right) {
        int mid = left + (right - left) / 2;

        if (canFinish(piles, h, mid)) {
            right = mid;  // Try eating slower
        } else {
            left = mid + 1;  // Need to eat faster
        }
    }

    return left;
}

private boolean canFinish(int[] piles, int h, int speed) {
    int hours = 0;
    for (int pile : piles) {
        hours += (pile + speed - 1) / speed;  // Ceiling division
    }
    return hours <= h;
}
```

### Example: Capacity to Ship Packages
```java
public int shipWithinDays(int[] weights, int days) {
    int left = Arrays.stream(weights).max().getAsInt();
    int right = Arrays.stream(weights).sum();

    while (left < right) {
        int mid = left + (right - left) / 2;

        if (canShip(weights, days, mid)) {
            right = mid;
        } else {
            left = mid + 1;
        }
    }

    return left;
}

private boolean canShip(int[] weights, int days, int capacity) {
    int daysNeeded = 1;
    int currentLoad = 0;

    for (int weight : weights) {
        if (currentLoad + weight > capacity) {
            daysNeeded++;
            currentLoad = 0;
        }
        currentLoad += weight;
    }

    return daysNeeded <= days;
}
```

---

## Pattern 6: Find Peak Element

### Core Idea
Move toward the side with a larger neighbor. A peak must exist on that side.

### When to Use
- Find local maximum/minimum
- Mountain array problems

### Mental Model
```
if nums[mid] < nums[mid + 1]:
    peak is on right → left = mid + 1
else:
    peak is on left (including mid) → right = mid
```

### Example: Find Peak Element
```java
public int findPeakElement(int[] nums) {
    int left = 0, right = nums.length - 1;

    while (left < right) {
        int mid = left + (right - left) / 2;

        if (nums[mid] < nums[mid + 1]) {
            left = mid + 1;  // Peak is to the right
        } else {
            right = mid;     // Peak is to the left (or this is peak)
        }
    }

    return left;
}
```

---

## Pattern 7: Search in 2D Matrix

### Core Idea
Treat 2D matrix as 1D sorted array. Convert index: row = idx / cols, col = idx % cols.

### When to Use
- Sorted 2D matrix (row-major sorted)
- Each row starts greater than previous row ends

### Example: Search a 2D Matrix
```java
public boolean searchMatrix(int[][] matrix, int target) {
    int m = matrix.length, n = matrix[0].length;
    int left = 0, right = m * n - 1;

    while (left <= right) {
        int mid = left + (right - left) / 2;
        int row = mid / n;
        int col = mid % n;
        int value = matrix[row][col];

        if (value == target) {
            return true;
        } else if (value < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }

    return false;
}
```

---

## Common Pitfalls

### 1. Overflow in mid calculation
```java
// Wrong - can overflow
int mid = (left + right) / 2;

// Correct
int mid = left + (right - left) / 2;
```

### 2. Infinite loop with wrong boundary update
```java
// When using while (left < right):
// If left = mid, can infinite loop when left + 1 == right
// Use: mid = left + (right - left + 1) / 2 for this case
```

### 3. Off-by-one errors
```java
// left <= right: searches all elements, returns -1 if not found
// left < right: left == right is the answer
```

---

## Quick Reference

| Pattern | Condition | Left Update | Right Update | Return |
|---------|-----------|-------------|--------------|--------|
| Exact Match | left <= right | mid + 1 | mid - 1 | mid or -1 |
| Left Boundary | left < right | mid + 1 | mid | left |
| Right Boundary | left < right | mid + 1 | mid | left - 1 |
| Rotated Search | left <= right | Depends on sorted half | Depends | mid or -1 |
| Answer Space | left < right | mid + 1 | mid | left |
| Peak Element | left < right | mid + 1 | mid | left |

## When to Use Binary Search

- Sorted array or rotated sorted array
- Answer has monotonic property
- O(log n) time required
- "Minimum maximum" or "Maximum minimum"
- Can verify feasibility in O(n)
