## 1. What is Composite Pattern?

Composite pattern composes objects into tree structures to represent part-whole hierarchies. It lets clients treat individual objects and compositions uniformly.

```
Without Composite:
  if (item instanceof File) {
      file.show();
  } else if (item instanceof Directory) {
      for (child : directory.getChildren()) {
          // Recursive handling
      }
  }

With Composite:
  item.show();  // Works for both File and Directory
```

**Key principle:** Treat individual objects and compositions uniformly.

---

## 2. Implementation Guide

### Step 1: Define Component Interface

```java
public interface FileSystemItem {
    void showDetails();
    int getSize();
}
```

### Step 2: Create Leaf (Individual Object)

```java
public class File implements FileSystemItem {
    private String name;
    private int size;
    
    public File(String name, int size) {
        this.name = name;
        this.size = size;
    }
    
    @Override
    public void showDetails() {
        System.out.println("File: " + name + " (" + size + " KB)");
    }
    
    @Override
    public int getSize() {
        return size;
    }
}
```

### Step 3: Create Composite (Container)

```java
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
    
    @Override
    public int getSize() {
        int totalSize = 0;
        for (FileSystemItem item : children) {
            totalSize += item.getSize();
        }
        return totalSize;
    }
}
```

### Step 4: Usage

```java
// Create files
File file1 = new File("document.txt", 10);
File file2 = new File("image.jpg", 50);
File file3 = new File("video.mp4", 200);

// Create directories
Directory folder1 = new Directory("Documents");
folder1.add(file1);
folder1.add(file2);

Directory folder2 = new Directory("Media");
folder2.add(file3);

Directory root = new Directory("Root");
root.add(folder1);
root.add(folder2);

// Treat uniformly
root.showDetails();
System.out.println("Total size: " + root.getSize() + " KB");
```

---

## 3. Common Interview Questions

### Q1: Organization Hierarchy

```java
public interface Employee {
    void showDetails();
    double getSalary();
}

public class Developer implements Employee {
    private String name;
    private double salary;
    
    public Developer(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }
    
    @Override
    public void showDetails() {
        System.out.println("Developer: " + name + " ($" + salary + ")");
    }
    
    @Override
    public double getSalary() {
        return salary;
    }
}

public class Designer implements Employee {
    private String name;
    private double salary;
    
    public Designer(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }
    
    @Override
    public void showDetails() {
        System.out.println("Designer: " + name + " ($" + salary + ")");
    }
    
    @Override
    public double getSalary() {
        return salary;
    }
}

public class Manager implements Employee {
    private String name;
    private double salary;
    private List<Employee> team = new ArrayList<>();
    
    public Manager(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }
    
    public void addTeamMember(Employee employee) {
        team.add(employee);
    }
    
    public void removeTeamMember(Employee employee) {
        team.remove(employee);
    }
    
    @Override
    public void showDetails() {
        System.out.println("Manager: " + name + " ($" + salary + ")");
        System.out.println("Team:");
        for (Employee employee : team) {
            System.out.print("  ");
            employee.showDetails();
        }
    }
    
    @Override
    public double getSalary() {
        double totalSalary = salary;
        for (Employee employee : team) {
            totalSalary += employee.getSalary();
        }
        return totalSalary;
    }
}

// Usage
Developer dev1 = new Developer("Alice", 80000);
Developer dev2 = new Developer("Bob", 75000);
Designer designer = new Designer("Charlie", 70000);

Manager manager = new Manager("David", 100000);
manager.addTeamMember(dev1);
manager.addTeamMember(dev2);
manager.addTeamMember(designer);

manager.showDetails();
System.out.println("Total team cost: $" + manager.getSalary());
```

### Q2: Graphics System

```java
public interface Graphic {
    void draw();
    void move(int x, int y);
}

public class Circle implements Graphic {
    private int x, y, radius;
    
    public Circle(int x, int y, int radius) {
        this.x = x;
        this.y = y;
        this.radius = radius;
    }
    
    @Override
    public void draw() {
        System.out.println("Drawing circle at (" + x + "," + y + ") radius=" + radius);
    }
    
    @Override
    public void move(int dx, int dy) {
        x += dx;
        y += dy;
    }
}

public class Rectangle implements Graphic {
    private int x, y, width, height;
    
    public Rectangle(int x, int y, int width, int height) {
        this.x = x;
        this.y = y;
        this.width = width;
        this.height = height;
    }
    
    @Override
    public void draw() {
        System.out.println("Drawing rectangle at (" + x + "," + y + ") " + 
                         width + "x" + height);
    }
    
    @Override
    public void move(int dx, int dy) {
        x += dx;
        y += dy;
    }
}

public class CompositeGraphic implements Graphic {
    private List<Graphic> graphics = new ArrayList<>();
    
    public void add(Graphic graphic) {
        graphics.add(graphic);
    }
    
    public void remove(Graphic graphic) {
        graphics.remove(graphic);
    }
    
    @Override
    public void draw() {
        for (Graphic graphic : graphics) {
            graphic.draw();
        }
    }
    
    @Override
    public void move(int dx, int dy) {
        for (Graphic graphic : graphics) {
            graphic.move(dx, dy);
        }
    }
}

// Usage
Circle circle = new Circle(10, 10, 5);
Rectangle rect = new Rectangle(20, 20, 10, 15);

CompositeGraphic group1 = new CompositeGraphic();
group1.add(circle);
group1.add(rect);

Circle circle2 = new Circle(50, 50, 8);
CompositeGraphic group2 = new CompositeGraphic();
group2.add(group1);
group2.add(circle2);

group2.draw();
group2.move(5, 5);
group2.draw();
```

### Q3: Menu System

```java
public interface MenuComponent {
    void display();
    double getPrice();
}

public class MenuItem implements MenuComponent {
    private String name;
    private double price;
    
    public MenuItem(String name, double price) {
        this.name = name;
        this.price = price;
    }
    
    @Override
    public void display() {
        System.out.println("  " + name + " - $" + price);
    }
    
    @Override
    public double getPrice() {
        return price;
    }
}

public class Menu implements MenuComponent {
    private String name;
    private List<MenuComponent> items = new ArrayList<>();
    
    public Menu(String name) {
        this.name = name;
    }
    
    public void add(MenuComponent component) {
        items.add(component);
    }
    
    public void remove(MenuComponent component) {
        items.remove(component);
    }
    
    @Override
    public void display() {
        System.out.println("\n" + name + ":");
        for (MenuComponent item : items) {
            item.display();
        }
    }
    
    @Override
    public double getPrice() {
        double total = 0;
        for (MenuComponent item : items) {
            total += item.getPrice();
        }
        return total;
    }
}

// Usage
Menu mainMenu = new Menu("Main Menu");

Menu breakfast = new Menu("Breakfast");
breakfast.add(new MenuItem("Pancakes", 5.99));
breakfast.add(new MenuItem("Eggs", 3.99));

Menu lunch = new Menu("Lunch");
lunch.add(new MenuItem("Burger", 8.99));
lunch.add(new MenuItem("Salad", 6.99));

mainMenu.add(breakfast);
mainMenu.add(lunch);

mainMenu.display();
```

---

## 4. Interview Tricks & Pitfalls

### Pitfall 1: Leaf Implementing Composite Methods

```java
// WRONG: Leaf has add/remove methods
public class File implements FileSystemItem {
    public void add(FileSystemItem item) {
        throw new UnsupportedOperationException();
    }
}

// CORRECT: Only composite has add/remove
// Or use separate interfaces
```

### Pitfall 2: Not Handling Null Children

```java
// WRONG: NullPointerException
public void showDetails() {
    for (FileSystemItem item : children) {  // children might be null
        item.showDetails();
    }
}

// CORRECT: Initialize collection
private List<FileSystemItem> children = new ArrayList<>();
```

### Pitfall 3: Circular References

```java
// PROBLEM: Parent adds child, child adds parent
// SOLUTION: Check before adding
public void add(FileSystemItem item) {
    if (item == this) {
        throw new IllegalArgumentException("Cannot add to itself");
    }
    children.add(item);
}
```

---

## 5. When to Use

✅ **Use when:**
- Tree structures (files, org charts)
- Part-whole hierarchies
- Want to treat individuals and groups uniformly
- Recursive composition

❌ **Don't use when:**
- Flat structure (no hierarchy)
- Different operations for leaf and composite
- Simple list suffices

---

## 6. Real-World Examples

- **File Systems**: Files and directories
- **GUI**: Containers and components
- **Organization Charts**: Employees and managers
- **Graphics**: Shapes and groups
- **DOM**: HTML elements and containers

---

## 7. Quick Checklist

1. ☐ Define component interface
2. ☐ Create leaf class (individual)
3. ☐ Create composite class (container)
4. ☐ Composite has list of components
5. ☐ Composite has add/remove methods
6. ☐ Both implement same interface
7. ☐ Demonstrate treating uniformly
8. ☐ Show recursive operations
