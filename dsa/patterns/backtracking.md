# Backtracking Patterns

Backtracking explores all possibilities by building candidates incrementally and abandoning invalid paths.

---

## Pattern 1: Subsets (Include/Exclude)

### Core Idea
For each element, decide to include or exclude it.

### When to Use
- Generate all subsets
- Power set
- Combination problems

### Mental Model
```
at each position:
    include element → recurse
    exclude element → recurse
```

### Generic Template
```java
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), result);
    return result;
}

private void backtrack(int[] nums, int start, List<Integer> current, List<List<Integer>> result) {
    result.add(new ArrayList<>(current));  // Add current subset

    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);           // Include
        backtrack(nums, i + 1, current, result);
        current.remove(current.size() - 1);  // Exclude (backtrack)
    }
}
```

### Example: Subsets
```java
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), result);
    return result;
}

private void backtrack(int[] nums, int start, List<Integer> path, List<List<Integer>> result) {
    result.add(new ArrayList<>(path));

    for (int i = start; i < nums.length; i++) {
        path.add(nums[i]);
        backtrack(nums, i + 1, path, result);
        path.remove(path.size() - 1);
    }
}
```

---

## Pattern 2: Subsets with Duplicates

### Core Idea
Sort first. Skip duplicates at the same decision level.

### When to Use
- Subsets with duplicate elements
- Combinations with duplicates

### Mental Model
```
sort array
at each level:
    skip if same as previous at same level
```

### Example: Subsets II
```java
public List<List<Integer>> subsetsWithDup(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(nums);  // Sort to group duplicates
    backtrack(nums, 0, new ArrayList<>(), result);
    return result;
}

private void backtrack(int[] nums, int start, List<Integer> path, List<List<Integer>> result) {
    result.add(new ArrayList<>(path));

    for (int i = start; i < nums.length; i++) {
        // Skip duplicates at same level
        if (i > start && nums[i] == nums[i - 1]) continue;

        path.add(nums[i]);
        backtrack(nums, i + 1, path, result);
        path.remove(path.size() - 1);
    }
}
```

---

## Pattern 3: Permutations

### Core Idea
Use each element exactly once. Track used elements.

### When to Use
- Generate all orderings
- Arrange n elements

### Mental Model
```
for each position:
    try all unused elements
    mark as used → recurse
    unmark (backtrack)
```

### Example: Permutations
```java
public List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    boolean[] used = new boolean[nums.length];
    backtrack(nums, used, new ArrayList<>(), result);
    return result;
}

private void backtrack(int[] nums, boolean[] used, List<Integer> path, List<List<Integer>> result) {
    if (path.size() == nums.length) {
        result.add(new ArrayList<>(path));
        return;
    }

    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;

        used[i] = true;
        path.add(nums[i]);
        backtrack(nums, used, path, result);
        path.remove(path.size() - 1);
        used[i] = false;
    }
}
```

### Permutations with Duplicates
```java
public List<List<Integer>> permuteUnique(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(nums);
    boolean[] used = new boolean[nums.length];
    backtrack(nums, used, new ArrayList<>(), result);
    return result;
}

private void backtrack(int[] nums, boolean[] used, List<Integer> path, List<List<Integer>> result) {
    if (path.size() == nums.length) {
        result.add(new ArrayList<>(path));
        return;
    }

    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;
        // Skip duplicate: same value and previous wasn't used
        if (i > 0 && nums[i] == nums[i - 1] && !used[i - 1]) continue;

        used[i] = true;
        path.add(nums[i]);
        backtrack(nums, used, path, result);
        path.remove(path.size() - 1);
        used[i] = false;
    }
}
```

---

## Pattern 4: Combinations (Choose K)

### Core Idea
Select exactly k elements from n elements.

### When to Use
- Choose k from n
- C(n, k) combinations

### Mental Model
```
base case: path.size() == k → add to result
for each remaining element:
    include → recurse
    exclude (backtrack)
```

### Example: Combinations
```java
public List<List<Integer>> combine(int n, int k) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(n, k, 1, new ArrayList<>(), result);
    return result;
}

private void backtrack(int n, int k, int start, List<Integer> path, List<List<Integer>> result) {
    if (path.size() == k) {
        result.add(new ArrayList<>(path));
        return;
    }

    // Optimization: need (k - path.size()) more elements
    for (int i = start; i <= n - (k - path.size()) + 1; i++) {
        path.add(i);
        backtrack(n, k, i + 1, path, result);
        path.remove(path.size() - 1);
    }
}
```

---

## Pattern 5: Combination Sum (With Repetition)

### Core Idea
Elements can be used multiple times. Stay at same index after including.

### When to Use
- Sum to target with unlimited use
- Coin change combinations

### Mental Model
```
include element → recurse with same index
exclude element → move to next index
```

### Example: Combination Sum
```java
public List<List<Integer>> combinationSum(int[] candidates, int target) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(candidates, target, 0, new ArrayList<>(), result);
    return result;
}

private void backtrack(int[] nums, int target, int start, List<Integer> path, List<List<Integer>> result) {
    if (target == 0) {
        result.add(new ArrayList<>(path));
        return;
    }
    if (target < 0) return;

    for (int i = start; i < nums.length; i++) {
        path.add(nums[i]);
        backtrack(nums, target - nums[i], i, path, result);  // Same index (can reuse)
        path.remove(path.size() - 1);
    }
}
```

### Combination Sum II (Each Element Once)
```java
public List<List<Integer>> combinationSum2(int[] candidates, int target) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(candidates);
    backtrack(candidates, target, 0, new ArrayList<>(), result);
    return result;
}

private void backtrack(int[] nums, int target, int start, List<Integer> path, List<List<Integer>> result) {
    if (target == 0) {
        result.add(new ArrayList<>(path));
        return;
    }

    for (int i = start; i < nums.length; i++) {
        if (nums[i] > target) break;  // Pruning
        if (i > start && nums[i] == nums[i - 1]) continue;  // Skip duplicates

        path.add(nums[i]);
        backtrack(nums, target - nums[i], i + 1, path, result);  // Next index
        path.remove(path.size() - 1);
    }
}
```

---

## Pattern 6: Palindrome Partitioning

### Core Idea
Partition string into all palindromic substrings.

### When to Use
- Partition with constraints
- All valid splits

### Example: Palindrome Partitioning
```java
public List<List<String>> partition(String s) {
    List<List<String>> result = new ArrayList<>();
    backtrack(s, 0, new ArrayList<>(), result);
    return result;
}

private void backtrack(String s, int start, List<String> path, List<List<String>> result) {
    if (start == s.length()) {
        result.add(new ArrayList<>(path));
        return;
    }

    for (int end = start + 1; end <= s.length(); end++) {
        String substring = s.substring(start, end);
        if (isPalindrome(substring)) {
            path.add(substring);
            backtrack(s, end, path, result);
            path.remove(path.size() - 1);
        }
    }
}

private boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        if (s.charAt(left++) != s.charAt(right--)) return false;
    }
    return true;
}
```

---

## Pattern 7: Word Search (Grid DFS)

### Core Idea
DFS on grid with backtracking. Mark visited, explore 4 directions, unmark.

### When to Use
- Find word in grid
- Path in matrix

### Example: Word Search
```java
public boolean exist(char[][] board, String word) {
    int m = board.length, n = board[0].length;

    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (dfs(board, word, i, j, 0)) {
                return true;
            }
        }
    }

    return false;
}

private boolean dfs(char[][] board, String word, int i, int j, int index) {
    if (index == word.length()) return true;

    if (i < 0 || i >= board.length || j < 0 || j >= board[0].length
        || board[i][j] != word.charAt(index)) {
        return false;
    }

    char temp = board[i][j];
    board[i][j] = '#';  // Mark visited

    boolean found = dfs(board, word, i + 1, j, index + 1)
                 || dfs(board, word, i - 1, j, index + 1)
                 || dfs(board, word, i, j + 1, index + 1)
                 || dfs(board, word, i, j - 1, index + 1);

    board[i][j] = temp;  // Unmark (backtrack)

    return found;
}
```

---

## Pattern 8: N-Queens

### Core Idea
Place queens row by row, checking column and diagonal conflicts.

### When to Use
- Constraint satisfaction
- Placement problems

### Example: N-Queens
```java
public List<List<String>> solveNQueens(int n) {
    List<List<String>> result = new ArrayList<>();
    char[][] board = new char[n][n];
    for (char[] row : board) Arrays.fill(row, '.');

    backtrack(board, 0, result);
    return result;
}

private void backtrack(char[][] board, int row, List<List<String>> result) {
    if (row == board.length) {
        result.add(construct(board));
        return;
    }

    for (int col = 0; col < board.length; col++) {
        if (isValid(board, row, col)) {
            board[row][col] = 'Q';
            backtrack(board, row + 1, result);
            board[row][col] = '.';
        }
    }
}

private boolean isValid(char[][] board, int row, int col) {
    // Check column
    for (int i = 0; i < row; i++) {
        if (board[i][col] == 'Q') return false;
    }
    // Check upper-left diagonal
    for (int i = row - 1, j = col - 1; i >= 0 && j >= 0; i--, j--) {
        if (board[i][j] == 'Q') return false;
    }
    // Check upper-right diagonal
    for (int i = row - 1, j = col + 1; i >= 0 && j < board.length; i--, j++) {
        if (board[i][j] == 'Q') return false;
    }
    return true;
}

private List<String> construct(char[][] board) {
    List<String> result = new ArrayList<>();
    for (char[] row : board) {
        result.add(new String(row));
    }
    return result;
}
```

---

## Quick Reference

| Pattern | Start Index | Can Reuse | Key Condition |
|---------|-------------|-----------|---------------|
| Subsets | i + 1 | No | Add at every node |
| Subsets II | i + 1 | No | Skip if same as prev |
| Permutations | 0 | No (used[]) | used[] tracking |
| Combinations | i + 1 | No | path.size() == k |
| Combo Sum | i (same) | Yes | target == 0 |
| Combo Sum II | i + 1 | No | Skip duplicates |
| Palindrome | end | - | isPalindrome check |
| Word Search | - | No (mark grid) | index == word.length |
| N-Queens | col 0..n | - | isValid check |

## Backtracking Template
```java
void backtrack(state, choices) {
    if (goal reached) {
        add to result
        return
    }

    for each choice in choices:
        if (valid choice):
            make choice
            backtrack(new state, remaining choices)
            undo choice  // BACKTRACK
}
```
