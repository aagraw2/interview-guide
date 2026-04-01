## 1. What is Strategy Pattern?

Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

```
Without Strategy:
  if (sortType == "merge") {
      // merge sort logic
  } else if (sortType == "quick") {
      // quick sort logic
  }
  // Hard to add new algorithms

With Strategy:
  context.setStrategy(new MergeSort());
  context.sort(arr);
  // Easy to add new algorithms
```

**Key principle:** Encapsulate algorithms and make them interchangeable.

---

## 2. Implementation Guide

### Step 1: Define Strategy Interface

```java
public interface SortStrategy {
    int[] sort(int[] arr);
}
```

### Step 2: Implement Concrete Strategies

```java
public class MergeSort implements SortStrategy {
    @Override
    public int[] sort(int[] arr) {
        System.out.println("Sorting using Merge Sort");
        // Merge sort implementation
        return mergeSort(arr, 0, arr.length - 1);
    }
    
    private int[] mergeSort(int[] arr, int left, int right) {
        if (left < right) {
            int mid = (left + right) / 2;
            mergeSort(arr, left, mid);
            mergeSort(arr, mid + 1, right);
            merge(arr, left, mid, right);
        }
        return arr;
    }
    
    private void merge(int[] arr, int left, int mid, int right) {
        // Merge logic
    }
}

public class QuickSort implements SortStrategy {
    @Override
    public int[] sort(int[] arr) {
        System.out.println("Sorting using Quick Sort");
        return quickSort(arr, 0, arr.length - 1);
    }
    
    private int[] quickSort(int[] arr, int low, int high) {
        if (low < high) {
            int pi = partition(arr, low, high);
            quickSort(arr, low, pi - 1);
            quickSort(arr, pi + 1, high);
        }
        return arr;
    }
    
    private int partition(int[] arr, int low, int high) {
        // Partition logic
        return 0;
    }
}

public class BubbleSort implements SortStrategy {
    @Override
    public int[] sort(int[] arr) {
        System.out.println("Sorting using Bubble Sort");
        for (int i = 0; i < arr.length - 1; i++) {
            for (int j = 0; j < arr.length - i - 1; j++) {
                if (arr[j] > arr[j + 1]) {
                    int temp = arr[j];
                    arr[j] = arr[j + 1];
                    arr[j + 1] = temp;
                }
            }
        }
        return arr;
    }
}
```

### Step 3: Create Context Class

```java
public class SortContext {
    private SortStrategy strategy;
    
    public SortContext(SortStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void setStrategy(SortStrategy strategy) {
        this.strategy = strategy;
    }
    
    public int[] executeStrategy(int[] arr) {
        return strategy.sort(arr);
    }
}
```

### Step 4: Usage

```java
int[] arr = {5, 2, 9, 1, 7, 6};

// Use merge sort
SortContext context = new SortContext(new MergeSort());
context.executeStrategy(arr);

// Switch to quick sort at runtime
context.setStrategy(new QuickSort());
context.executeStrategy(arr);

// Switch to bubble sort
context.setStrategy(new BubbleSort());
context.executeStrategy(arr);
```

---

## 3. Common Interview Questions

### Q1: Payment Processing System

```java
// Strategy interface
public interface PaymentStrategy {
    void pay(double amount);
}

// Concrete strategies
public class CreditCardPayment implements PaymentStrategy {
    private String cardNumber;
    private String cvv;
    
    public CreditCardPayment(String cardNumber, String cvv) {
        this.cardNumber = cardNumber;
        this.cvv = cvv;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " using Credit Card ending in " + 
            cardNumber.substring(cardNumber.length() - 4));
    }
}

public class PayPalPayment implements PaymentStrategy {
    private String email;
    
    public PayPalPayment(String email) {
        this.email = email;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " using PayPal account " + email);
    }
}

public class UPIPayment implements PaymentStrategy {
    private String upiId;
    
    public UPIPayment(String upiId) {
        this.upiId = upiId;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " using UPI ID " + upiId);
    }
}

// Context
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();
    private PaymentStrategy paymentStrategy;
    
    public void addItem(Item item) {
        items.add(item);
    }
    
    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }
    
    public void checkout() {
        double total = items.stream().mapToDouble(Item::getPrice).sum();
        paymentStrategy.pay(total);
    }
}

// Usage
ShoppingCart cart = new ShoppingCart();
cart.addItem(new Item("Laptop", 1000));
cart.addItem(new Item("Mouse", 50));

cart.setPaymentStrategy(new CreditCardPayment("1234-5678-9012-3456", "123"));
cart.checkout();

cart.setPaymentStrategy(new PayPalPayment("user@example.com"));
cart.checkout();
```

### Q2: Compression Strategy

```java
public interface CompressionStrategy {
    byte[] compress(byte[] data);
    byte[] decompress(byte[] data);
}

public class ZipCompression implements CompressionStrategy {
    @Override
    public byte[] compress(byte[] data) {
        System.out.println("Compressing using ZIP");
        // ZIP compression logic
        return data;
    }
    
    @Override
    public byte[] decompress(byte[] data) {
        System.out.println("Decompressing ZIP");
        return data;
    }
}

public class RarCompression implements CompressionStrategy {
    @Override
    public byte[] compress(byte[] data) {
        System.out.println("Compressing using RAR");
        return data;
    }
    
    @Override
    public byte[] decompress(byte[] data) {
        System.out.println("Decompressing RAR");
        return data;
    }
}

public class FileCompressor {
    private CompressionStrategy strategy;
    
    public void setStrategy(CompressionStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void compressFile(String filename) {
        byte[] data = readFile(filename);
        byte[] compressed = strategy.compress(data);
        writeFile(filename + ".compressed", compressed);
    }
}
```

### Q3: Navigation Strategy

```java
public interface RouteStrategy {
    void buildRoute(String start, String end);
}

public class CarRoute implements RouteStrategy {
    @Override
    public void buildRoute(String start, String end) {
        System.out.println("Building car route from " + start + " to " + end);
        System.out.println("Via highways, estimated time: 2 hours");
    }
}

public class WalkingRoute implements RouteStrategy {
    @Override
    public void buildRoute(String start, String end) {
        System.out.println("Building walking route from " + start + " to " + end);
        System.out.println("Via pedestrian paths, estimated time: 30 minutes");
    }
}

public class PublicTransportRoute implements RouteStrategy {
    @Override
    public void buildRoute(String start, String end) {
        System.out.println("Building public transport route from " + start + " to " + end);
        System.out.println("Via bus and metro, estimated time: 45 minutes");
    }
}

public class Navigator {
    private RouteStrategy strategy;
    
    public void setStrategy(RouteStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void navigate(String start, String end) {
        strategy.buildRoute(start, end);
    }
}
```

---

## 4. Interview Tricks & Pitfalls

### Pitfall 1: Null Strategy

```java
// WRONG: NullPointerException if strategy not set
public void execute() {
    strategy.doSomething();  // NPE if strategy is null
}

// CORRECT: Check for null or use default
public void execute() {
    if (strategy == null) {
        throw new IllegalStateException("Strategy not set");
    }
    strategy.doSomething();
}

// OR: Default strategy
public Context() {
    this.strategy = new DefaultStrategy();
}
```

### Pitfall 2: Strategy with State

```java
// PROBLEM: Strategy holds state, not thread-safe
public class StatefulStrategy implements Strategy {
    private int counter = 0;  // State
    
    public void execute() {
        counter++;  // Not thread-safe
    }
}

// SOLUTION: Make strategy stateless or thread-safe
public class StatelessStrategy implements Strategy {
    public void execute(Context context) {
        int counter = context.getCounter();  // Get state from context
        context.setCounter(counter + 1);
    }
}
```

### Pitfall 3: Too Many Strategies

```java
// PROBLEM: Creating too many strategy classes for minor variations
// SOLUTION: Use parameters or combine with other patterns

public class ConfigurableStrategy implements Strategy {
    private boolean option1;
    private boolean option2;
    
    public ConfigurableStrategy(boolean option1, boolean option2) {
        this.option1 = option1;
        this.option2 = option2;
    }
    
    public void execute() {
        if (option1) { /* ... */ }
        if (option2) { /* ... */ }
    }
}
```

---

## 5. Strategy vs State Pattern

| Aspect | Strategy | State |
|--------|----------|-------|
| Purpose | Choose algorithm | Change behavior based on state |
| Who decides | Client sets strategy | Object changes state internally |
| Transitions | Client switches strategies | States transition automatically |
| Example | Payment methods | Traffic light states |

---

## 6. When to Use

✅ **Use when:**
- Multiple algorithms for same task
- Need to switch algorithms at runtime
- Want to avoid conditional statements
- Algorithms should be independent

❌ **Don't use when:**
- Only one algorithm
- Algorithm never changes
- Simple conditional logic suffices

---

## 7. Real-World Examples

- **Java**: `Comparator` interface
- **Collections**: `Collections.sort(list, comparator)`
- **Layout Managers**: Swing/AWT layout strategies
- **Compression**: Different compression algorithms
- **Routing**: Navigation apps (car, walk, bike)

---

## 8. Quick Implementation Checklist

1. ☐ Define strategy interface with algorithm method
2. ☐ Create concrete strategy classes
3. ☐ Create context class that holds strategy reference
4. ☐ Provide method to set/change strategy
5. ☐ Context delegates to strategy
6. ☐ Handle null strategy case
7. ☐ Demonstrate switching strategies at runtime

**Template:**
```java
// Strategy interface
public interface Strategy {
    void execute();
}

// Concrete strategies
public class ConcreteStrategyA implements Strategy {
    public void execute() { /* Algorithm A */ }
}

public class ConcreteStrategyB implements Strategy {
    public void execute() { /* Algorithm B */ }
}

// Context
public class Context {
    private Strategy strategy;
    
    public void setStrategy(Strategy strategy) {
        this.strategy = strategy;
    }
    
    public void executeStrategy() {
        if (strategy == null) {
            throw new IllegalStateException("Strategy not set");
        }
        strategy.execute();
    }
}
```
