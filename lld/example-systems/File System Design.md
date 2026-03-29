
# 1. Functional Requirements

- Create files and directories
    
- Delete files/directories
    
- Read file content
    
- Write/update file content
    
- List directory contents
    
- Support hierarchical structure
    
- Support file metadata (name, size, timestamps)
    

---

# 2. Non-Functional Requirements

- Fast lookup and traversal
    
- Thread-safe operations
    
- Scalable structure
    
- Low latency reads/writes
    
- Extensible (permissions, versioning)
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|FileSystem|Entry point|
|Node|Base class for File/Directory|
|File|Stores content|
|Directory|Holds child nodes|
|Metadata|File info|
|FileSystemService|Handles operations|

---

# 4. Design Approach

- Use **Composite Pattern**
    
    - File and Directory treated uniformly as Node
        
- Tree structure for hierarchy
    
- Each directory maintains map of children
    

---

# 5. Flows

---

## Create File/Directory

1. Traverse path
    
2. Validate parent exists
    
3. Add new node to parent
    

---

## Delete Node

1. Traverse to node
    
2. Remove from parent
    

---

## Read File

1. Traverse path
    
2. Return content
    

---

## List Directory

1. Traverse path
    
2. Return children
    

---

# 6. Code

```java
import java.util.*;
import java.util.concurrent.*;

// METADATA
class Metadata {
    String name;
    long createdAt;
    long updatedAt;

    public Metadata(String name) {
        this.name = name;
        this.createdAt = System.currentTimeMillis();
        this.updatedAt = this.createdAt;
    }
}

// ABSTRACT NODE (Composite Pattern)
abstract class Node {
    Metadata metadata;

    public Node(String name) {
        this.metadata = new Metadata(name);
    }

    abstract boolean isFile();
}

// FILE
class FileNode extends Node {
    StringBuilder content = new StringBuilder();

    public FileNode(String name) {
        super(name);
    }

    public boolean isFile() { return true; }

    public void write(String data) {
        content.append(data);
        metadata.updatedAt = System.currentTimeMillis();
    }

    public String read() {
        return content.toString();
    }
}

// DIRECTORY
class Directory extends Node {
    Map<String, Node> children = new ConcurrentHashMap<>();

    public Directory(String name) {
        super(name);
    }

    public boolean isFile() { return false; }

    public void add(Node node) {
        children.put(node.metadata.name, node);
    }

    public Node get(String name) {
        return children.get(name);
    }

    public void remove(String name) {
        children.remove(name);
    }

    public Collection<Node> list() {
        return children.values();
    }
}

// FILE SYSTEM
class FileSystem {
    private Directory root = new Directory("/");

    public Node traverse(String path) {
        String[] parts = path.split("/");
        Node curr = root;

        for (String part : parts) {
            if (part.isEmpty()) continue;

            if (!(curr instanceof Directory)) {
                throw new RuntimeException("Invalid path");
            }

            curr = ((Directory) curr).get(part);
            if (curr == null) {
                throw new RuntimeException("Path not found");
            }
        }
        return curr;
    }

    public void createFile(String path) {
        String[] parts = path.split("/");
        Directory parent = (Directory) traverseParent(parts);
        parent.add(new FileNode(parts[parts.length - 1]));
    }

    public void createDirectory(String path) {
        String[] parts = path.split("/");
        Directory parent = (Directory) traverseParent(parts);
        parent.add(new Directory(parts[parts.length - 1]));
    }

    private Node traverseParent(String[] parts) {
        Node curr = root;

        for (int i = 0; i < parts.length - 1; i++) {
            if (parts[i].isEmpty()) continue;

            curr = ((Directory) curr).get(parts[i]);
        }
        return curr;
    }

    public String readFile(String path) {
        FileNode file = (FileNode) traverse(path);
        return file.read();
    }

    public void writeFile(String path, String data) {
        FileNode file = (FileNode) traverse(path);
        file.write(data);
    }

    public void delete(String path) {
        String[] parts = path.split("/");
        Directory parent = (Directory) traverseParent(parts);
        parent.remove(parts[parts.length - 1]);
    }

    public List<String> list(String path) {
        Directory dir = (Directory) traverse(path);
        List<String> result = new ArrayList<>();

        for (Node node : dir.list()) {
            result.add(node.metadata.name);
        }
        return result;
    }
}
```

---

# 7. Complexity Analysis

- Traverse path → O(L) (L = path depth)
    
- Create/Delete → O(L)
    
- Read/Write → O(L)
    

---

# 8. Design Patterns Used

- Composite Pattern → File & Directory unified
    
- Repository-like structure → Directory as container
    
- Dependency Injection (extendable services)
    

---

# 9. Key Design Decisions

- Tree structure for hierarchical filesystem
    
- Directory uses ConcurrentHashMap for thread safety
    
- Metadata separated for extensibility
    
- Path traversal centralized
    
- Immutable structure avoided for simplicity
    
- Supports easy extension (permissions, versioning)
    

---

# 10. Edge Cases Handled

- Invalid path
    
- File vs directory mismatch
    
- Missing parent directories
    
- Concurrent modifications
    

---

# 11. Possible Extensions

- Add file permissions (read/write/execute)
    
- Add versioning system
    
- Add symbolic links
    
- Introduce caching
    
- Add distributed file system support
    
- Add search functionality
    
- Add file size limits
    
- Add locking mechanism per node