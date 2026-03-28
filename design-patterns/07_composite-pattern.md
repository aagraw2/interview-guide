# Composite Pattern

## Intent
The Composite pattern allows treating individual objects and groups uniformly (same interface for both).

## When to Use
- Tree structures
- Hierarchical data
- Uniform operations

## Example
```java
public interface FileSystemItem {
    void showDetails();
}

public class File implements FileSystemItem {

    private String name;

    public File(String name) {
        this.name = name;
    }

    @Override
    public void showDetails() {
        System.out.println("File: " + name);
    }
}

public class Directory implements FileSystemItem {

    private String name;
    private List<FileSystemItem> children = new ArrayList<>();

    public Directory(String name) {
        this.name = name;
    }

    public void add(FileSystemItem item) {
        children.add(item);
    }

    public void remove(FileSystemItem item) {
        children.remove(item);
    }

    @Override
    public void showDetails() {
        System.out.println("Directory: " + name);

        for (FileSystemItem item : children) {
            item.showDetails();
        }
    }
}

public class Main {

    public static void main(String[] args) {

        File file1 = new File("file1.txt");
        File file2 = new File("file2.txt");

        Directory folder1 = new Directory("Documents");
        folder1.add(file1);
        folder1.add(file2);

        File file3 = new File("photo.jpg");

        Directory root = new Directory("Root");
        root.add(folder1);
        root.add(file3);

        root.showDetails();
    }
}
```