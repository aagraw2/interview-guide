# Trees Patterns

Trees combine recursion with data structures. Master DFS, BFS, and BST properties.

---

## Pattern 1: DFS - Recursive Traversal

### Core Idea
Process node, then recursively process children. Order determines traversal type.

### When to Use
- Tree traversal
- Path problems
- Tree properties (height, diameter)

### Mental Model
```
Preorder:  process → left → right
Inorder:   left → process → right  (BST gives sorted)
Postorder: left → right → process
```

### Generic Template
```java
// Return value from subtrees (height, count, etc.)
public int dfs(TreeNode node) {
    if (node == null) return baseCase;

    int left = dfs(node.left);
    int right = dfs(node.right);

    // Process current node using left and right results
    return combine(left, right, node.val);
}
```

### Example: Maximum Depth
```java
public int maxDepth(TreeNode root) {
    if (root == null) return 0;

    return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}
```

### Example: Invert Binary Tree
```java
public TreeNode invertTree(TreeNode root) {
    if (root == null) return null;

    TreeNode left = invertTree(root.left);
    TreeNode right = invertTree(root.right);

    root.left = right;
    root.right = left;

    return root;
}
```

---

## Pattern 2: DFS with Global State

### Core Idea
Track global variable (max sum, diameter) while traversing.

### When to Use
- Problems requiring comparison across subtrees
- Diameter, max path sum

### Mental Model
```
global variable for result
dfs returns local info
update global during traversal
```

### Example: Diameter of Binary Tree
```java
private int diameter = 0;

public int diameterOfBinaryTree(TreeNode root) {
    height(root);
    return diameter;
}

private int height(TreeNode node) {
    if (node == null) return 0;

    int left = height(node.left);
    int right = height(node.right);

    diameter = Math.max(diameter, left + right);

    return 1 + Math.max(left, right);
}
```

### Example: Binary Tree Maximum Path Sum
```java
private int maxSum = Integer.MIN_VALUE;

public int maxPathSum(TreeNode root) {
    maxGain(root);
    return maxSum;
}

private int maxGain(TreeNode node) {
    if (node == null) return 0;

    int left = Math.max(maxGain(node.left), 0);
    int right = Math.max(maxGain(node.right), 0);

    // Path through this node
    maxSum = Math.max(maxSum, left + right + node.val);

    // Return max gain if continuing from this node
    return node.val + Math.max(left, right);
}
```

---

## Pattern 3: BFS - Level Order Traversal

### Core Idea
Process nodes level by level using a queue.

### When to Use
- Level order traversal
- Shortest path in tree
- Level-based operations

### Mental Model
```
queue.add(root)
while queue not empty:
    levelSize = queue.size()
    process levelSize nodes
    add their children to queue
```

### Example: Binary Tree Level Order Traversal
```java
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;

    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);

    while (!queue.isEmpty()) {
        int levelSize = queue.size();
        List<Integer> level = new ArrayList<>();

        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();
            level.add(node.val);

            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }

        result.add(level);
    }

    return result;
}
```

### Example: Right Side View
```java
public List<Integer> rightSideView(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    if (root == null) return result;

    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);

    while (!queue.isEmpty()) {
        int levelSize = queue.size();

        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();

            if (i == levelSize - 1) {
                result.add(node.val);  // Last node in level
            }

            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
    }

    return result;
}
```

---

## Pattern 4: BST Property (Inorder = Sorted)

### Core Idea
BST inorder traversal gives sorted order. Use for kth smallest, validation.

### When to Use
- Validate BST
- Kth smallest/largest
- BST operations

### Mental Model
```
left subtree < node < right subtree
inorder gives sorted order
```

### Example: Validate Binary Search Tree
```java
public boolean isValidBST(TreeNode root) {
    return validate(root, null, null);
}

private boolean validate(TreeNode node, Integer min, Integer max) {
    if (node == null) return true;

    if ((min != null && node.val <= min) ||
        (max != null && node.val >= max)) {
        return false;
    }

    return validate(node.left, min, node.val) &&
           validate(node.right, node.val, max);
}
```

### Example: Kth Smallest Element in BST
```java
public int kthSmallest(TreeNode root, int k) {
    Stack<TreeNode> stack = new Stack<>();
    TreeNode curr = root;

    while (curr != null || !stack.isEmpty()) {
        while (curr != null) {
            stack.push(curr);
            curr = curr.left;
        }

        curr = stack.pop();
        k--;
        if (k == 0) return curr.val;

        curr = curr.right;
    }

    return -1;
}
```

---

## Pattern 5: LCA (Lowest Common Ancestor)

### Core Idea
Find the deepest node that is ancestor of both nodes.

### When to Use
- Find common ancestor
- Path between nodes

### Example: LCA of BST
```java
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    while (root != null) {
        if (p.val < root.val && q.val < root.val) {
            root = root.left;
        } else if (p.val > root.val && q.val > root.val) {
            root = root.right;
        } else {
            return root;
        }
    }
    return null;
}
```

### Example: LCA of Binary Tree
```java
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) {
        return root;
    }

    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);

    if (left != null && right != null) {
        return root;  // p and q are in different subtrees
    }

    return left != null ? left : right;
}
```

---

## Pattern 6: Path Sum / Path Problems

### Core Idea
Track sum/path while traversing from root to leaf.

### When to Use
- Path sum problems
- Root to leaf paths

### Example: Path Sum
```java
public boolean hasPathSum(TreeNode root, int targetSum) {
    if (root == null) return false;

    if (root.left == null && root.right == null) {
        return root.val == targetSum;
    }

    return hasPathSum(root.left, targetSum - root.val) ||
           hasPathSum(root.right, targetSum - root.val);
}
```

### Example: Sum Root to Leaf Numbers
```java
public int sumNumbers(TreeNode root) {
    return dfs(root, 0);
}

private int dfs(TreeNode node, int currentSum) {
    if (node == null) return 0;

    currentSum = currentSum * 10 + node.val;

    if (node.left == null && node.right == null) {
        return currentSum;
    }

    return dfs(node.left, currentSum) + dfs(node.right, currentSum);
}
```

---

## Pattern 7: Tree Construction

### Core Idea
Use traversal orders to reconstruct tree.

### When to Use
- Construct from preorder + inorder
- Construct from sorted array

### Example: Construct from Preorder and Inorder
```java
private int preIndex = 0;
private Map<Integer, Integer> inorderMap;

public TreeNode buildTree(int[] preorder, int[] inorder) {
    inorderMap = new HashMap<>();
    for (int i = 0; i < inorder.length; i++) {
        inorderMap.put(inorder[i], i);
    }
    return build(preorder, 0, inorder.length - 1);
}

private TreeNode build(int[] preorder, int left, int right) {
    if (left > right) return null;

    int rootVal = preorder[preIndex++];
    TreeNode root = new TreeNode(rootVal);

    int inIndex = inorderMap.get(rootVal);

    root.left = build(preorder, left, inIndex - 1);
    root.right = build(preorder, inIndex + 1, right);

    return root;
}
```

### Example: Convert Sorted Array to BST
```java
public TreeNode sortedArrayToBST(int[] nums) {
    return build(nums, 0, nums.length - 1);
}

private TreeNode build(int[] nums, int left, int right) {
    if (left > right) return null;

    int mid = left + (right - left) / 2;
    TreeNode node = new TreeNode(nums[mid]);

    node.left = build(nums, left, mid - 1);
    node.right = build(nums, mid + 1, right);

    return node;
}
```

---

## Pattern 8: Serialize / Deserialize

### Core Idea
Convert tree to string and back.

### When to Use
- Save/load tree
- Network transmission

### Example: Serialize and Deserialize
```java
public String serialize(TreeNode root) {
    if (root == null) return "null";

    return root.val + "," +
           serialize(root.left) + "," +
           serialize(root.right);
}

public TreeNode deserialize(String data) {
    Queue<String> queue = new LinkedList<>(Arrays.asList(data.split(",")));
    return buildTree(queue);
}

private TreeNode buildTree(Queue<String> queue) {
    String val = queue.poll();
    if (val.equals("null")) return null;

    TreeNode node = new TreeNode(Integer.parseInt(val));
    node.left = buildTree(queue);
    node.right = buildTree(queue);

    return node;
}
```

---

## Quick Reference

| Pattern | Technique | Use Case |
|---------|-----------|----------|
| DFS Recursive | Return value from subtrees | Height, count, same tree |
| DFS + Global | Track max while traversing | Diameter, max path sum |
| BFS | Level-order queue | Level traversal, right view |
| BST Property | Inorder = sorted | Validate, kth smallest |
| LCA | Find in both subtrees | Common ancestor |
| Path Sum | Track sum down path | Path problems |
| Construction | Build from traversals | Rebuild tree |
| Serialize | String conversion | Save/load |

## Tree Recursion Template
```java
public Result solve(TreeNode node) {
    // Base case
    if (node == null) return baseResult;

    // Recursive calls
    Result left = solve(node.left);
    Result right = solve(node.right);

    // Combine results
    return combine(left, right, node);
}
```
