## 1. What is Decorator Pattern?

Decorator pattern attaches additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.

```
Without Decorator:
  Need classes: TextBox, BorderedTextBox, ScrollableTextBox, 
                BorderedScrollableTextBox, ShadowedBorderedScrollableTextBox...
  Class explosion!

With Decorator:
  textBox = new ShadowDecorator(
              new BorderDecorator(
                new ScrollDecorator(
                  new TextBox())));
  Flexible combinations!
```

**Key principle:** Wrap objects to add behavior without modifying original class.

---

## 2. Implementation Guide

### Step 1: Define Component Interface

```java
public interface UIComponent {
    void render();
    int getCost();
}
```

### Step 2: Create Concrete Component

```java
public class TextBox implements UIComponent {
    @Override
    public void render() {
        System.out.println("Rendering TextBox");
    }
    
    @Override
    public int getCost() {
        return 10;
    }
}
```

### Step 3: Create Abstract Decorator

```java
public abstract class ComponentDecorator implements UIComponent {
    protected UIComponent component;
    
    public ComponentDecorator(UIComponent component) {
        this.component = component;
    }
    
    @Override
    public void render() {
        component.render();
    }
    
    @Override
    public int getCost() {
        return component.getCost();
    }
}
```

### Step 4: Create Concrete Decorators

```java
public class BorderDecorator extends ComponentDecorator {
    public BorderDecorator(UIComponent component) {
        super(component);
    }
    
    @Override
    public void render() {
        super.render();
        addBorder();
    }
    
    @Override
    public int getCost() {
        return super.getCost() + 5;
    }
    
    private void addBorder() {
        System.out.println("Adding Border");
    }
}

public class ScrollDecorator extends ComponentDecorator {
    public ScrollDecorator(UIComponent component) {
        super(component);
    }
    
    @Override
    public void render() {
        super.render();
        addScrollbar();
    }
    
    @Override
    public int getCost() {
        return super.getCost() + 3;
    }
    
    private void addScrollbar() {
        System.out.println("Adding Scrollbar");
    }
}

public class ShadowDecorator extends ComponentDecorator {
    public ShadowDecorator(UIComponent component) {
        super(component);
    }
    
    @Override
    public void render() {
        super.render();
        addShadow();
    }
    
    @Override
    public int getCost() {
        return super.getCost() + 2;
    }
    
    private void addShadow() {
        System.out.println("Adding Shadow");
    }
}
```

### Step 5: Usage

```java
// Simple text box
UIComponent simple = new TextBox();
simple.render();
System.out.println("Cost: " + simple.getCost());

// Text box with border
UIComponent bordered = new BorderDecorator(new TextBox());
bordered.render();
System.out.println("Cost: " + bordered.getCost());

// Text box with border and scroll
UIComponent complex = new ScrollDecorator(
                        new BorderDecorator(
                          new TextBox()));
complex.render();
System.out.println("Cost: " + complex.getCost());

// Text box with all decorations
UIComponent full = new ShadowDecorator(
                     new ScrollDecorator(
                       new BorderDecorator(
                         new TextBox())));
full.render();
System.out.println("Cost: " + full.getCost());
```

---

## 3. Common Interview Questions

### Q1: Coffee Shop

```java
public interface Coffee {
    String getDescription();
    double getCost();
}

public class SimpleCoffee implements Coffee {
    @Override
    public String getDescription() {
        return "Simple Coffee";
    }
    
    @Override
    public double getCost() {
        return 2.0;
    }
}

public abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription();
    }
    
    @Override
    public double getCost() {
        return coffee.getCost();
    }
}

public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription() + ", Milk";
    }
    
    @Override
    public double getCost() {
        return coffee.getCost() + 0.5;
    }
}

public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription() + ", Sugar";
    }
    
    @Override
    public double getCost() {
        return coffee.getCost() + 0.2;
    }
}

public class WhipDecorator extends CoffeeDecorator {
    public WhipDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription() + ", Whip";
    }
    
    @Override
    public double getCost() {
        return coffee.getCost() + 0.7;
    }
}

// Usage
Coffee coffee = new SimpleCoffee();
System.out.println(coffee.getDescription() + " $" + coffee.getCost());

coffee = new MilkDecorator(coffee);
System.out.println(coffee.getDescription() + " $" + coffee.getCost());

coffee = new SugarDecorator(coffee);
System.out.println(coffee.getDescription() + " $" + coffee.getCost());

coffee = new WhipDecorator(coffee);
System.out.println(coffee.getDescription() + " $" + coffee.getCost());
```

### Q2: Data Stream

```java
public interface DataSource {
    void writeData(String data);
    String readData();
}

public class FileDataSource implements DataSource {
    private String filename;
    private String data;
    
    public FileDataSource(String filename) {
        this.filename = filename;
    }
    
    @Override
    public void writeData(String data) {
        System.out.println("Writing to file: " + data);
        this.data = data;
    }
    
    @Override
    public String readData() {
        System.out.println("Reading from file");
        return data;
    }
}

public abstract class DataSourceDecorator implements DataSource {
    protected DataSource wrappee;
    
    public DataSourceDecorator(DataSource source) {
        this.wrappee = source;
    }
    
    @Override
    public void writeData(String data) {
        wrappee.writeData(data);
    }
    
    @Override
    public String readData() {
        return wrappee.readData();
    }
}

public class EncryptionDecorator extends DataSourceDecorator {
    public EncryptionDecorator(DataSource source) {
        super(source);
    }
    
    @Override
    public void writeData(String data) {
        String encrypted = encrypt(data);
        super.writeData(encrypted);
    }
    
    @Override
    public String readData() {
        String data = super.readData();
        return decrypt(data);
    }
    
    private String encrypt(String data) {
        System.out.println("Encrypting data");
        return "encrypted(" + data + ")";
    }
    
    private String decrypt(String data) {
        System.out.println("Decrypting data");
        return data.replace("encrypted(", "").replace(")", "");
    }
}

public class CompressionDecorator extends DataSourceDecorator {
    public CompressionDecorator(DataSource source) {
        super(source);
    }
    
    @Override
    public void writeData(String data) {
        String compressed = compress(data);
        super.writeData(compressed);
    }
    
    @Override
    public String readData() {
        String data = super.readData();
        return decompress(data);
    }
    
    private String compress(String data) {
        System.out.println("Compressing data");
        return "compressed(" + data + ")";
    }
    
    private String decompress(String data) {
        System.out.println("Decompressing data");
        return data.replace("compressed(", "").replace(")", "");
    }
}

// Usage
DataSource source = new FileDataSource("data.txt");
source = new EncryptionDecorator(source);
source = new CompressionDecorator(source);

source.writeData("Hello World");
String data = source.readData();
System.out.println("Final data: " + data);
```

---

## 4. Interview Tricks & Pitfalls

### Pitfall 1: Breaking Interface

```java
// WRONG: Decorator adds methods not in interface
public class SpecialDecorator extends Decorator {
    public void specialMethod() { }  // Not in interface!
}

// Client can't call specialMethod through interface
Component c = new SpecialDecorator(new ConcreteComponent());
// c.specialMethod();  // Compile error!

// CORRECT: Keep same interface or cast when needed
```

### Pitfall 2: Order Matters

```java
// Different order = different result
Component c1 = new EncryptionDecorator(new CompressionDecorator(source));
// Compress then encrypt

Component c2 = new CompressionDecorator(new EncryptionDecorator(source));
// Encrypt then compress

// Results are different!
```

### Pitfall 3: Too Many Decorators

```java
// PROBLEM: Deep nesting hard to read
Component c = new D1(new D2(new D3(new D4(new D5(new Base())))));

// SOLUTION: Use builder or factory
public class ComponentBuilder {
    private Component component;
    
    public ComponentBuilder(Component base) {
        this.component = base;
    }
    
    public ComponentBuilder addBorder() {
        component = new BorderDecorator(component);
        return this;
    }
    
    public ComponentBuilder addScroll() {
        component = new ScrollDecorator(component);
        return this;
    }
    
    public Component build() {
        return component;
    }
}

// Usage
Component c = new ComponentBuilder(new TextBox())
    .addBorder()
    .addScroll()
    .build();
```

---

## 5. When to Use

✅ **Use when:**
- Add responsibilities dynamically
- Avoid subclass explosion
- Combine behaviors flexibly
- Extend sealed/final classes

❌ **Don't use when:**
- Simple inheritance suffices
- Order of decorators matters (confusing)
- Need to remove decorations

---

## 6. Real-World Examples

- **Java I/O**: `BufferedReader(new FileReader())`
- **Java I/O**: `DataInputStream(new BufferedInputStream())`
- **Collections**: `Collections.synchronizedList(list)`
- **Collections**: `Collections.unmodifiableList(list)`

---

## 7. Quick Checklist

1. ☐ Define component interface
2. ☐ Create concrete component
3. ☐ Create abstract decorator implementing interface
4. ☐ Decorator holds component reference
5. ☐ Create concrete decorators extending abstract decorator
6. ☐ Decorators call super then add behavior
7. ☐ Demonstrate wrapping multiple decorators
