# Graphs Patterns

Graphs extend tree concepts with cycles and multiple paths. Master DFS, BFS, and Union-Find.

---

## Pattern 1: DFS on Grid (Island Problems)

### Core Idea
Treat grid as graph. Each cell connects to 4 neighbors. DFS/BFS to explore connected regions.

### When to Use
- Number of islands
- Flood fill
- Connected components in grid

### Mental Model
```
for each cell:
    if unvisited land → start DFS/BFS
    mark all connected as visited
    count components
```

### Example: Number of Islands
```java
public int numIslands(char[][] grid) {
    int count = 0;
    int m = grid.length, n = grid[0].length;

    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == '1') {
                dfs(grid, i, j);
                count++;
            }
        }
    }

    return count;
}

private void dfs(char[][] grid, int i, int j) {
    if (i < 0 || i >= grid.length || j < 0 || j >= grid[0].length
        || grid[i][j] != '1') {
        return;
    }

    grid[i][j] = '0';  // Mark visited

    dfs(grid, i + 1, j);
    dfs(grid, i - 1, j);
    dfs(grid, i, j + 1);
    dfs(grid, i, j - 1);
}
```

---

## Pattern 2: BFS Shortest Path (Unweighted)

### Core Idea
BFS gives shortest path in unweighted graphs.

### When to Use
- Shortest path in unweighted graph
- Minimum steps
- Level-based distance

### Mental Model
```
queue with starting node
while queue not empty:
    process node
    add unvisited neighbors
first time reaching target = shortest path
```

### Example: Shortest Path in Binary Matrix
```java
public int shortestPathBinaryMatrix(int[][] grid) {
    int n = grid.length;
    if (grid[0][0] == 1 || grid[n-1][n-1] == 1) return -1;

    int[][] dirs = {{0,1},{1,0},{0,-1},{-1,0},{1,1},{1,-1},{-1,1},{-1,-1}};
    Queue<int[]> queue = new LinkedList<>();
    queue.offer(new int[] {0, 0, 1});
    grid[0][0] = 1;  // Mark visited

    while (!queue.isEmpty()) {
        int[] curr = queue.poll();
        int r = curr[0], c = curr[1], dist = curr[2];

        if (r == n - 1 && c == n - 1) return dist;

        for (int[] dir : dirs) {
            int nr = r + dir[0], nc = c + dir[1];
            if (nr >= 0 && nr < n && nc >= 0 && nc < n && grid[nr][nc] == 0) {
                grid[nr][nc] = 1;
                queue.offer(new int[] {nr, nc, dist + 1});
            }
        }
    }

    return -1;
}
```

### Example: Rotting Oranges (Multi-source BFS)
```java
public int orangesRotting(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    Queue<int[]> queue = new LinkedList<>();
    int fresh = 0;

    // Add all rotten oranges to queue
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == 2) {
                queue.offer(new int[] {i, j});
            } else if (grid[i][j] == 1) {
                fresh++;
            }
        }
    }

    if (fresh == 0) return 0;

    int[][] dirs = {{0,1},{1,0},{0,-1},{-1,0}};
    int minutes = 0;

    while (!queue.isEmpty()) {
        int size = queue.size();
        boolean rotted = false;

        for (int i = 0; i < size; i++) {
            int[] curr = queue.poll();

            for (int[] dir : dirs) {
                int nr = curr[0] + dir[0], nc = curr[1] + dir[1];
                if (nr >= 0 && nr < m && nc >= 0 && nc < n && grid[nr][nc] == 1) {
                    grid[nr][nc] = 2;
                    queue.offer(new int[] {nr, nc});
                    fresh--;
                    rotted = true;
                }
            }
        }

        if (rotted) minutes++;
    }

    return fresh == 0 ? minutes : -1;
}
```

---

## Pattern 3: Topological Sort (Course Schedule)

### Core Idea
Order nodes so all edges point forward. Detect cycles.

### When to Use
- Course prerequisites
- Build order
- Dependency resolution

### Mental Model
```
Kahn's Algorithm (BFS):
1. Count indegrees
2. Start with nodes having indegree 0
3. Process, reduce neighbors' indegree
4. Add to queue when indegree becomes 0
```

### Example: Course Schedule (Can Finish)
```java
public boolean canFinish(int numCourses, int[][] prerequisites) {
    int[] indegree = new int[numCourses];
    List<List<Integer>> graph = new ArrayList<>();

    for (int i = 0; i < numCourses; i++) {
        graph.add(new ArrayList<>());
    }

    for (int[] prereq : prerequisites) {
        graph.get(prereq[1]).add(prereq[0]);
        indegree[prereq[0]]++;
    }

    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < numCourses; i++) {
        if (indegree[i] == 0) {
            queue.offer(i);
        }
    }

    int completed = 0;
    while (!queue.isEmpty()) {
        int course = queue.poll();
        completed++;

        for (int next : graph.get(course)) {
            indegree[next]--;
            if (indegree[next] == 0) {
                queue.offer(next);
            }
        }
    }

    return completed == numCourses;
}
```

### Example: Course Schedule II (Find Order)
```java
public int[] findOrder(int numCourses, int[][] prerequisites) {
    int[] indegree = new int[numCourses];
    List<List<Integer>> graph = new ArrayList<>();

    for (int i = 0; i < numCourses; i++) {
        graph.add(new ArrayList<>());
    }

    for (int[] prereq : prerequisites) {
        graph.get(prereq[1]).add(prereq[0]);
        indegree[prereq[0]]++;
    }

    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < numCourses; i++) {
        if (indegree[i] == 0) {
            queue.offer(i);
        }
    }

    int[] order = new int[numCourses];
    int idx = 0;

    while (!queue.isEmpty()) {
        int course = queue.poll();
        order[idx++] = course;

        for (int next : graph.get(course)) {
            if (--indegree[next] == 0) {
                queue.offer(next);
            }
        }
    }

    return idx == numCourses ? order : new int[0];
}
```

---

## Pattern 4: Union-Find (Disjoint Set)

### Core Idea
Group elements into disjoint sets. Efficiently find if elements are connected.

### When to Use
- Connected components
- Cycle detection in undirected graph
- Kruskal's MST

### Template
```java
class UnionFind {
    private int[] parent;
    private int[] rank;

    public UnionFind(int n) {
        parent = new int[n];
        rank = new int[n];
        for (int i = 0; i < n; i++) {
            parent[i] = i;
        }
    }

    public int find(int x) {
        if (parent[x] != x) {
            parent[x] = find(parent[x]);  // Path compression
        }
        return parent[x];
    }

    public boolean union(int x, int y) {
        int px = find(x), py = find(y);
        if (px == py) return false;  // Already connected

        // Union by rank
        if (rank[px] < rank[py]) {
            parent[px] = py;
        } else if (rank[px] > rank[py]) {
            parent[py] = px;
        } else {
            parent[py] = px;
            rank[px]++;
        }
        return true;
    }
}
```

### Example: Number of Provinces
```java
public int findCircleNum(int[][] isConnected) {
    int n = isConnected.length;
    UnionFind uf = new UnionFind(n);
    int provinces = n;

    for (int i = 0; i < n; i++) {
        for (int j = i + 1; j < n; j++) {
            if (isConnected[i][j] == 1) {
                if (uf.union(i, j)) {
                    provinces--;
                }
            }
        }
    }

    return provinces;
}
```

---

## Pattern 5: Cycle Detection

### Core Idea
Detect cycles using DFS coloring (3 states) or Union-Find.

### When to Use
- Detect cycle in directed/undirected graph
- Validate tree

### Example: Cycle in Directed Graph (DFS)
```java
public boolean hasCycle(int n, int[][] edges) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < n; i++) graph.add(new ArrayList<>());
    for (int[] e : edges) graph.get(e[0]).add(e[1]);

    int[] state = new int[n];  // 0: unvisited, 1: visiting, 2: visited

    for (int i = 0; i < n; i++) {
        if (hasCycleDFS(graph, i, state)) return true;
    }
    return false;
}

private boolean hasCycleDFS(List<List<Integer>> graph, int node, int[] state) {
    if (state[node] == 1) return true;   // Back edge = cycle
    if (state[node] == 2) return false;  // Already processed

    state[node] = 1;  // Visiting

    for (int next : graph.get(node)) {
        if (hasCycleDFS(graph, next, state)) return true;
    }

    state[node] = 2;  // Visited
    return false;
}
```

---

## Pattern 6: Dijkstra (Weighted Shortest Path)

### Core Idea
Greedy BFS with priority queue. Always process node with smallest distance.

### When to Use
- Shortest path with positive weights
- Minimum cost

### Example: Network Delay Time
```java
public int networkDelayTime(int[][] times, int n, int k) {
    Map<Integer, List<int[]>> graph = new HashMap<>();
    for (int[] t : times) {
        graph.computeIfAbsent(t[0], x -> new ArrayList<>()).add(new int[] {t[1], t[2]});
    }

    int[] dist = new int[n + 1];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[k] = 0;

    // [node, distance]
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
    pq.offer(new int[] {k, 0});

    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int node = curr[0], d = curr[1];

        if (d > dist[node]) continue;

        if (graph.containsKey(node)) {
            for (int[] next : graph.get(node)) {
                int newDist = d + next[1];
                if (newDist < dist[next[0]]) {
                    dist[next[0]] = newDist;
                    pq.offer(new int[] {next[0], newDist});
                }
            }
        }
    }

    int maxTime = 0;
    for (int i = 1; i <= n; i++) {
        if (dist[i] == Integer.MAX_VALUE) return -1;
        maxTime = Math.max(maxTime, dist[i]);
    }
    return maxTime;
}
```

---

## Pattern 7: Clone Graph

### Core Idea
DFS/BFS with hashmap to track cloned nodes.

### When to Use
- Deep copy of graph
- Clone with references

### Example: Clone Graph
```java
public Node cloneGraph(Node node) {
    if (node == null) return null;

    Map<Node, Node> visited = new HashMap<>();
    return dfs(node, visited);
}

private Node dfs(Node node, Map<Node, Node> visited) {
    if (visited.containsKey(node)) {
        return visited.get(node);
    }

    Node clone = new Node(node.val);
    visited.put(node, clone);

    for (Node neighbor : node.neighbors) {
        clone.neighbors.add(dfs(neighbor, visited));
    }

    return clone;
}
```

---

## Pattern 8: Bipartite Check

### Core Idea
Try to 2-color the graph. If conflict, not bipartite.

### When to Use
- Check if graph is bipartite
- Two-team assignment

### Example: Is Graph Bipartite
```java
public boolean isBipartite(int[][] graph) {
    int n = graph.length;
    int[] color = new int[n];  // 0: uncolored, 1: red, -1: blue

    for (int i = 0; i < n; i++) {
        if (color[i] == 0 && !bfs(graph, i, color)) {
            return false;
        }
    }
    return true;
}

private boolean bfs(int[][] graph, int start, int[] color) {
    Queue<Integer> queue = new LinkedList<>();
    queue.offer(start);
    color[start] = 1;

    while (!queue.isEmpty()) {
        int node = queue.poll();

        for (int neighbor : graph[node]) {
            if (color[neighbor] == 0) {
                color[neighbor] = -color[node];
                queue.offer(neighbor);
            } else if (color[neighbor] == color[node]) {
                return false;
            }
        }
    }
    return true;
}
```

---

## Quick Reference

| Pattern | Algorithm | Time | Use Case |
|---------|-----------|------|----------|
| Grid DFS/BFS | DFS/BFS | O(mn) | Islands, flood fill |
| Shortest Path | BFS | O(V+E) | Unweighted graphs |
| Topological Sort | Kahn's | O(V+E) | Dependencies |
| Union-Find | DSU | O(α(n)) | Components, cycles |
| Cycle Detection | DFS 3-color | O(V+E) | Detect cycles |
| Dijkstra | Priority Queue | O(E log V) | Weighted shortest path |
| Clone | DFS + HashMap | O(V+E) | Deep copy |
| Bipartite | BFS 2-color | O(V+E) | Two-coloring |

## Graph Representation
```java
// Adjacency List
List<List<Integer>> graph = new ArrayList<>();

// Adjacency Matrix
int[][] graph = new int[n][n];

// Edge List
int[][] edges;  // [[from, to], ...]
```
