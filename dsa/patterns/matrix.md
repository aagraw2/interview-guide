# Matrix Patterns

Matrix problems require coordinate transformations, boundary tracking, and spatial reasoning.

---

## Pattern 1: Spiral Traversal

### Core Idea
Track four boundaries (top, bottom, left, right). Traverse and shrink boundaries.

### When to Use
- Spiral order traversal
- Layer-by-layer processing

### Mental Model
```
while boundaries valid:
    go right along top → top++
    go down along right → right--
    go left along bottom → bottom--
    go up along left → left++
```

### Example: Spiral Matrix
```java
public List<Integer> spiralOrder(int[][] matrix) {
    List<Integer> result = new ArrayList<>();
    if (matrix.length == 0) return result;

    int top = 0, bottom = matrix.length - 1;
    int left = 0, right = matrix[0].length - 1;

    while (top <= bottom && left <= right) {
        // Right
        for (int i = left; i <= right; i++) {
            result.add(matrix[top][i]);
        }
        top++;

        // Down
        for (int i = top; i <= bottom; i++) {
            result.add(matrix[i][right]);
        }
        right--;

        // Left
        if (top <= bottom) {
            for (int i = right; i >= left; i--) {
                result.add(matrix[bottom][i]);
            }
            bottom--;
        }

        // Up
        if (left <= right) {
            for (int i = bottom; i >= top; i--) {
                result.add(matrix[i][left]);
            }
            left++;
        }
    }

    return result;
}
```

---

## Pattern 2: Rotate Matrix (Transpose + Reverse)

### Core Idea
90° clockwise = transpose + reverse rows. 90° counterclockwise = transpose + reverse columns.

### When to Use
- Rotate image
- In-place rotation

### Mental Model
```
90° clockwise:
    1. Transpose: swap(m[i][j], m[j][i])
    2. Reverse each row

90° counterclockwise:
    1. Transpose
    2. Reverse each column
```

### Example: Rotate Image (90° Clockwise)
```java
public void rotate(int[][] matrix) {
    int n = matrix.length;

    // Transpose
    for (int i = 0; i < n; i++) {
        for (int j = i + 1; j < n; j++) {
            int temp = matrix[i][j];
            matrix[i][j] = matrix[j][i];
            matrix[j][i] = temp;
        }
    }

    // Reverse each row
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n / 2; j++) {
            int temp = matrix[i][j];
            matrix[i][j] = matrix[i][n - 1 - j];
            matrix[i][n - 1 - j] = temp;
        }
    }
}
```

---

## Pattern 3: Set Matrix Zeroes (In-Place Marking)

### Core Idea
Use first row/column as markers to avoid extra space.

### When to Use
- In-place matrix modification
- Mark and apply pattern

### Mental Model
```
1. Check if first row/col need zeroing
2. Use first row/col to mark other rows/cols
3. Zero based on markers
4. Handle first row/col last
```

### Example: Set Matrix Zeroes
```java
public void setZeroes(int[][] matrix) {
    int m = matrix.length, n = matrix[0].length;
    boolean firstRowZero = false, firstColZero = false;

    // Check first row and column
    for (int j = 0; j < n; j++) {
        if (matrix[0][j] == 0) firstRowZero = true;
    }
    for (int i = 0; i < m; i++) {
        if (matrix[i][0] == 0) firstColZero = true;
    }

    // Mark using first row/col
    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            if (matrix[i][j] == 0) {
                matrix[i][0] = 0;
                matrix[0][j] = 0;
            }
        }
    }

    // Zero based on markers
    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            if (matrix[i][0] == 0 || matrix[0][j] == 0) {
                matrix[i][j] = 0;
            }
        }
    }

    // Handle first row and column
    if (firstRowZero) {
        for (int j = 0; j < n; j++) matrix[0][j] = 0;
    }
    if (firstColZero) {
        for (int i = 0; i < m; i++) matrix[i][0] = 0;
    }
}
```

---

## Pattern 4: Diagonal Traversal

### Core Idea
Group cells by diagonal (i + j constant or i - j constant).

### When to Use
- Diagonal traversal
- Anti-diagonal problems

### Mental Model
```
Main diagonal: i - j is constant
Anti-diagonal: i + j is constant
```

### Example: Diagonal Traverse
```java
public int[] findDiagonalOrder(int[][] mat) {
    int m = mat.length, n = mat[0].length;
    int[] result = new int[m * n];
    int idx = 0;
    boolean up = true;

    int row = 0, col = 0;

    while (idx < m * n) {
        result[idx++] = mat[row][col];

        if (up) {
            if (col == n - 1) {
                row++;
                up = false;
            } else if (row == 0) {
                col++;
                up = false;
            } else {
                row--;
                col++;
            }
        } else {
            if (row == m - 1) {
                col++;
                up = true;
            } else if (col == 0) {
                row++;
                up = true;
            } else {
                row++;
                col--;
            }
        }
    }

    return result;
}
```

---

## Pattern 5: Search in Sorted Matrix

### Core Idea
Start from top-right (or bottom-left). Eliminate row or column based on comparison.

### When to Use
- Search in row-wise and column-wise sorted matrix
- O(m + n) search

### Mental Model
```
start at top-right (smallest in row, largest in col)
if target < current → go left
if target > current → go down
```

### Example: Search a 2D Matrix II
```java
public boolean searchMatrix(int[][] matrix, int target) {
    int m = matrix.length, n = matrix[0].length;
    int row = 0, col = n - 1;

    while (row < m && col >= 0) {
        if (matrix[row][col] == target) {
            return true;
        } else if (matrix[row][col] > target) {
            col--;
        } else {
            row++;
        }
    }

    return false;
}
```

---

## Pattern 6: Flood Fill / DFS on Grid

### Core Idea
Start from source, propagate to adjacent cells with same property.

### When to Use
- Fill connected region
- Modify connected cells

### Example: Flood Fill
```java
public int[][] floodFill(int[][] image, int sr, int sc, int color) {
    if (image[sr][sc] == color) return image;

    fill(image, sr, sc, image[sr][sc], color);
    return image;
}

private void fill(int[][] image, int r, int c, int oldColor, int newColor) {
    if (r < 0 || r >= image.length || c < 0 || c >= image[0].length
        || image[r][c] != oldColor) {
        return;
    }

    image[r][c] = newColor;

    fill(image, r + 1, c, oldColor, newColor);
    fill(image, r - 1, c, oldColor, newColor);
    fill(image, r, c + 1, oldColor, newColor);
    fill(image, r, c - 1, oldColor, newColor);
}
```

---

## Pattern 7: Game of Life (Encode States)

### Core Idea
Encode old and new state in same cell to avoid extra space.

### When to Use
- Simultaneous updates
- In-place state transition

### Mental Model
```
Use bit encoding:
    bit 0 = current state
    bit 1 = next state

After processing:
    right shift to get next state
```

### Example: Game of Life
```java
public void gameOfLife(int[][] board) {
    int m = board.length, n = board[0].length;
    int[] dirs = {-1, 0, 1};

    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            int liveNeighbors = 0;

            for (int di : dirs) {
                for (int dj : dirs) {
                    if (di == 0 && dj == 0) continue;
                    int ni = i + di, nj = j + dj;
                    if (ni >= 0 && ni < m && nj >= 0 && nj < n) {
                        liveNeighbors += board[ni][nj] & 1;
                    }
                }
            }

            // Encode next state in bit 1
            if (board[i][j] == 1 && (liveNeighbors == 2 || liveNeighbors == 3)) {
                board[i][j] = 3;  // 11 in binary
            } else if (board[i][j] == 0 && liveNeighbors == 3) {
                board[i][j] = 2;  // 10 in binary
            }
        }
    }

    // Extract next state
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            board[i][j] >>= 1;
        }
    }
}
```

---

## Pattern 8: Matrix as 1D Array (Binary Search)

### Core Idea
Treat sorted 2D matrix as 1D sorted array for binary search.

### When to Use
- Each row starts greater than previous row ends
- Binary search in matrix

### Mental Model
```
Convert 1D index to 2D:
    row = index / cols
    col = index % cols
```

### Example: Search a 2D Matrix
```java
public boolean searchMatrix(int[][] matrix, int target) {
    int m = matrix.length, n = matrix[0].length;
    int left = 0, right = m * n - 1;

    while (left <= right) {
        int mid = left + (right - left) / 2;
        int row = mid / n, col = mid % n;
        int val = matrix[row][col];

        if (val == target) return true;
        else if (val < target) left = mid + 1;
        else right = mid - 1;
    }

    return false;
}
```

---

## Quick Reference

| Pattern | Technique | Use Case |
|---------|-----------|----------|
| Spiral | 4 boundaries | Spiral traversal |
| Rotate | Transpose + reverse | 90° rotation |
| Set Zeroes | First row/col markers | In-place zeroing |
| Diagonal | i+j or i-j grouping | Diagonal traversal |
| Search | Top-right start | Row/col sorted matrix |
| Flood Fill | DFS/BFS | Fill connected region |
| Game of Life | Bit encoding | Simultaneous updates |
| 1D Mapping | row = i/n, col = i%n | Binary search |

## Direction Arrays
```java
// 4 directions
int[][] dirs4 = {{0,1}, {1,0}, {0,-1}, {-1,0}};

// 8 directions (including diagonals)
int[][] dirs8 = {{0,1}, {1,0}, {0,-1}, {-1,0},
                 {1,1}, {1,-1}, {-1,1}, {-1,-1}};

// Usage
for (int[] dir : dirs) {
    int nr = r + dir[0], nc = c + dir[1];
    if (nr >= 0 && nr < m && nc >= 0 && nc < n) {
        // valid neighbor
    }
}
```
