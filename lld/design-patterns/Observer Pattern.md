## 1. What is Observer Pattern?

The Observer pattern defines a one-to-many dependency between objects so that when one object (Subject) changes state, all its dependents (Observers) are notified and updated automatically.

```
Without Observer:
  Object A changes → Manually notify B, C, D
  Tight coupling, hard to maintain

With Observer:
  Object A changes → Automatically notifies all registered observers
  Loose coupling, easy to add/remove observers
```

**Also known as:** Publish-Subscribe, Event-Listener, Dependents

---

## 2. Key Components

### Subject (Observable/Publisher)

The object being observed. Maintains list of observers and notifies them of changes.

```
Responsibilities:
  - Maintain list of observers
  - Provide methods to register/unregister observers
  - Notify all observers when state changes
```

### Observer (Subscriber/Listener)

The object that wants to be notified of changes.

```
Responsibilities:
  - Implement update method
  - Register with subject
  - React to notifications
```

### Concrete Subject

Specific implementation of Subject with actual state.

```
Responsibilities:
  - Store state
  - Trigger notifications when state changes
  - Provide getter methods for state
```

### Concrete Observer

Specific implementation of Observer with actual behavior.

```
Responsibilities:
  - Implement specific update logic
  - Maintain reference to subject (optional)
  - Update own state based on subject's state
```

---

## 3. UML Structure

```
┌─────────────────┐
│    Subject      │
├─────────────────┤
│ +attach(obs)    │
│ +detach(obs)    │
│ +notify()       │
└────────┬────────┘
         │
         │ notifies
         ▼
┌─────────────────┐
│    Observer     │
├─────────────────┤
│ +update()       │
└─────────────────┘
         △
         │ implements
         │
┌────────┴────────┐
│ ConcreteObserver│
├─────────────────┤
│ +update()       │
│ -state          │
└─────────────────┘
```

---

## 4. Step-by-Step Implementation Guide

### Step 1: Define Observer Interface

```java
// Observer.java
public interface Observer {
    /**
     * Called when subject's state changes
     * @param subject The subject that changed
     */
    void update(Subject subject);
}
```

**Alternative with data parameter:**
```java
public interface Observer<T> {
    void update(T data);
}
```

### Step 2: Define Subject Interface

```java
// Subject.java
public interface Subject {
    /**
     * Register an observer
     */
    void registerObserver(Observer observer);
    
    /**
     * Unregister an observer
     */
    void removeObserver(Observer observer);
    
    /**
     * Notify all registered observers
     */
    void notifyObservers();
}
```

### Step 3: Implement Concrete Subject

```java
// Product.java
import java.util.*;

public class Product implements Subject {
    
    // IMPORTANT: Use List to maintain order, ArrayList for performance
    private List<Observer> observers = new ArrayList<>();
    
    // Subject's state
    private String name;
    private double price;
    private boolean inStock;
    
    public Product(String name, double price) {
        this.name = name;
        this.price = price;
        this.inStock = true;
    }
    
    // State modification methods should trigger notification
    public void setPrice(double price) {
        if (this.price != price) {  // Only notify if actually changed
            this.price = price;
            notifyObservers();
        }
    }
    
    public void setInStock(boolean inStock) {
        if (this.inStock != inStock) {
            this.inStock = inStock;
            notifyObservers();
        }
    }
    
    // Getters for observers to access state
    public String getName() {
        return name;
    }
    
    public double getPrice() {
        return price;
    }
    
    public boolean isInStock() {
        return inStock;
    }
    
    @Override
    public void registerObserver(Observer observer) {
        if (observer != null && !observers.contains(observer)) {
            observers.add(observer);
        }
    }
    
    @Override
    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }
    
    @Override
    public void notifyObservers() {
        // IMPORTANT: Iterate over copy to avoid ConcurrentModificationException
        for (Observer observer : new ArrayList<>(observers)) {
            observer.update(this);
        }
    }
}
```

### Step 4: Implement Concrete Observers

```java
// EmailNotificationObserver.java
public class EmailNotificationObserver implements Observer {
    
    private String email;
    
    public EmailNotificationObserver(String email) {
        this.email = email;
    }
    
    @Override
    public void update(Subject subject) {
        if (subject instanceof Product) {
            Product product = (Product) subject;
            sendEmail(email, 
                "Price Alert: " + product.getName() + 
                " is now $" + product.getPrice());
        }
    }
    
    private void sendEmail(String to, String message) {
        System.out.println("Email to " + to + ": " + message);
    }
}

// SMSNotificationObserver.java
public class SMSNotificationObserver implements Observer {
    
    private String phoneNumber;
    
    public SMSNotificationObserver(String phoneNumber) {
        this.phoneNumber = phoneNumber;
    }
    
    @Override
    public void update(Subject subject) {
        if (subject instanceof Product) {
            Product product = (Product) subject;
            sendSMS(phoneNumber, 
                product.getName() + " price: $" + product.getPrice());
        }
    }
    
    private void sendSMS(String phone, String message) {
        System.out.println("SMS to " + phone + ": " + message);
    }
}

// InventoryObserver.java
public class InventoryObserver implements Observer {
    
    private String warehouseId;
    
    public InventoryObserver(String warehouseId) {
        this.warehouseId = warehouseId;
    }
    
    @Override
    public void update(Subject subject) {
        if (subject instanceof Product) {
            Product product = (Product) subject;
            if (!product.isInStock()) {
                reorderStock(product);
            }
        }
    }
    
    private void reorderStock(Product product) {
        System.out.println("Warehouse " + warehouseId + 
            ": Reordering " + product.getName());
    }
}
```

### Step 5: Usage Example

```java
// Main.java
public class Main {
    
    public static void main(String[] args) {
        
        // Create subject
        Product iphone = new Product("iPhone 15", 999.99);
        
        // Create observers
        Observer emailObserver = new EmailNotificationObserver("user@example.com");
        Observer smsObserver = new SMSNotificationObserver("+1234567890");
        Observer inventoryObserver = new InventoryObserver("WH-001");
        
        // Register observers
        iphone.registerObserver(emailObserver);
        iphone.registerObserver(smsObserver);
        iphone.registerObserver(inventoryObserver);
        
        // Change state - all observers notified automatically
        System.out.println("=== Price Change ===");
        iphone.setPrice(899.99);
        
        System.out.println("\n=== Stock Change ===");
        iphone.setInStock(false);
        
        // Unregister observer
        System.out.println("\n=== After Removing SMS Observer ===");
        iphone.removeObserver(smsObserver);
        iphone.setPrice(849.99);
    }
}
```

**Output:**
```
=== Price Change ===
Email to user@example.com: Price Alert: iPhone 15 is now $899.99
SMS to +1234567890: iPhone 15 price: $899.99

=== Stock Change ===
Email to user@example.com: Price Alert: iPhone 15 is now $899.99
SMS to +1234567890: iPhone 15 price: $899.99
Warehouse WH-001: Reordering iPhone 15

=== After Removing SMS Observer ===
Email to user@example.com: Price Alert: iPhone 15 is now $849.99
```

---

## 5. Implementation Variations

### Push Model (Current Implementation)

Subject pushes all data to observers.

```java
public interface Observer {
    void update(Subject subject);  // Subject passed
}

// Observer pulls what it needs
public void update(Subject subject) {
    Product product = (Product) subject;
    double price = product.getPrice();  // Pull specific data
}
```

**Pros:** Observers can choose what data to use
**Cons:** Observers need to know subject's interface

### Pull Model

Subject only notifies, observers pull data they need.

```java
public interface Observer {
    void update();  // No parameters
}

public class ConcreteObserver implements Observer {
    private Product product;  // Reference to subject
    
    public ConcreteObserver(Product product) {
        this.product = product;
        product.registerObserver(this);
    }
    
    @Override
    public void update() {
        double price = product.getPrice();  // Pull data
        // React to change
    }
}
```

**Pros:** More flexible, observers control what they need
**Cons:** Observers need reference to subject

### Event-Based (With Event Object)

Pass specific event data instead of entire subject.

```java
// Event class
public class PriceChangeEvent {
    private String productName;
    private double oldPrice;
    private double newPrice;
    
    public PriceChangeEvent(String productName, double oldPrice, double newPrice) {
        this.productName = productName;
        this.oldPrice = oldPrice;
        this.newPrice = newPrice;
    }
    
    // Getters...
}

// Observer interface
public interface Observer {
    void update(PriceChangeEvent event);
}

// Subject notifies with event
public void setPrice(double newPrice) {
    double oldPrice = this.price;
    this.price = newPrice;
    
    PriceChangeEvent event = new PriceChangeEvent(name, oldPrice, newPrice);
    notifyObservers(event);
}
```

**Pros:** Type-safe, clear what changed
**Cons:** Need event class for each type of change

---

## 6. Where to Put What

### Subject (Observable) Location

```
Place in: Domain/Model layer
Why: Represents business entity that changes

Example structure:
src/
  model/
    Product.java          ← Subject implementation
    Stock.java            ← Another subject
  observer/
    Observer.java         ← Observer interface
    Subject.java          ← Subject interface
  notification/
    EmailObserver.java    ← Concrete observer
    SMSObserver.java      ← Concrete observer
```

### Observer Location

```
Place in: Service/Handler layer
Why: Represents side effects and reactions

Example:
src/
  service/
    NotificationService.java  ← Contains observers
    InventoryService.java     ← Contains observers
```

### Registration Logic

```
Place in: Application/Controller layer or Factory
Why: Wiring/configuration concern

Example:
public class ProductFactory {
    public static Product createProduct(String name, double price) {
        Product product = new Product(name, price);
        
        // Register default observers
        product.registerObserver(new EmailObserver());
        product.registerObserver(new InventoryObserver());
        
        return product;
    }
}
```

---

## 7. Common Interview Questions & Implementations

### Q1: Implement Weather Station

```java
// WeatherData.java (Subject)
public class WeatherData implements Subject {
    private List<Observer> observers = new ArrayList<>();
    private float temperature;
    private float humidity;
    private float pressure;
    
    public void setMeasurements(float temp, float humidity, float pressure) {
        this.temperature = temp;
        this.humidity = humidity;
        this.pressure = pressure;
        notifyObservers();
    }
    
    public float getTemperature() { return temperature; }
    public float getHumidity() { return humidity; }
    public float getPressure() { return pressure; }
    
    // Subject methods...
}

// CurrentConditionsDisplay.java (Observer)
public class CurrentConditionsDisplay implements Observer {
    private float temperature;
    private float humidity;
    
    @Override
    public void update(Subject subject) {
        if (subject instanceof WeatherData) {
            WeatherData weatherData = (WeatherData) subject;
            this.temperature = weatherData.getTemperature();
            this.humidity = weatherData.getHumidity();
            display();
        }
    }
    
    public void display() {
        System.out.println("Current: " + temperature + "F, " + humidity + "% humidity");
    }
}

// Usage
WeatherData weatherData = new WeatherData();
Observer currentDisplay = new CurrentConditionsDisplay();
Observer statisticsDisplay = new StatisticsDisplay();

weatherData.registerObserver(currentDisplay);
weatherData.registerObserver(statisticsDisplay);

weatherData.setMeasurements(80, 65, 30.4f);
```

### Q2: Implement Stock Market Ticker

```java
// Stock.java (Subject)
public class Stock implements Subject {
    private String symbol;
    private double price;
    private List<Observer> observers = new ArrayList<>();
    
    public Stock(String symbol, double price) {
        this.symbol = symbol;
        this.price = price;
    }
    
    public void setPrice(double price) {
        this.price = price;
        notifyObservers();
    }
    
    public String getSymbol() { return symbol; }
    public double getPrice() { return price; }
    
    // Subject methods...
}

// Investor.java (Observer)
public class Investor implements Observer {
    private String name;
    private double buyThreshold;
    private double sellThreshold;
    
    public Investor(String name, double buyThreshold, double sellThreshold) {
        this.name = name;
        this.buyThreshold = buyThreshold;
        this.sellThreshold = sellThreshold;
    }
    
    @Override
    public void update(Subject subject) {
        if (subject instanceof Stock) {
            Stock stock = (Stock) subject;
            double price = stock.getPrice();
            
            if (price <= buyThreshold) {
                System.out.println(name + ": BUY " + stock.getSymbol() + " at $" + price);
            } else if (price >= sellThreshold) {
                System.out.println(name + ": SELL " + stock.getSymbol() + " at $" + price);
            }
        }
    }
}

// Usage
Stock apple = new Stock("AAPL", 150.0);
Observer investor1 = new Investor("Alice", 145.0, 160.0);
Observer investor2 = new Investor("Bob", 140.0, 165.0);

apple.registerObserver(investor1);
apple.registerObserver(investor2);

apple.setPrice(144.0);  // Alice buys
apple.setPrice(161.0);  // Alice sells
```

### Q3: Implement Auction System

```java
// Auction.java (Subject)
public class Auction implements Subject {
    private String itemName;
    private double currentBid;
    private String currentBidder;
    private List<Observer> observers = new ArrayList<>();
    
    public void placeBid(String bidder, double amount) {
        if (amount > currentBid) {
            currentBid = amount;
            currentBidder = bidder;
            notifyObservers();
        }
    }
    
    // Getters and Subject methods...
}

// Bidder.java (Observer)
public class Bidder implements Observer {
    private String name;
    private double maxBid;
    
    @Override
    public void update(Subject subject) {
        if (subject instanceof Auction) {
            Auction auction = (Auction) subject;
            if (!auction.getCurrentBidder().equals(name) && 
                auction.getCurrentBid() < maxBid) {
                System.out.println(name + " notified: Current bid is $" + 
                    auction.getCurrentBid());
            }
        }
    }
}
```

---

## 8. Interview Tricks & Pitfalls

### Pitfall 1: ConcurrentModificationException

```java
// WRONG: Modifying list while iterating
public void notifyObservers() {
    for (Observer observer : observers) {
        observer.update(this);
        // If update() calls removeObserver(), exception thrown!
    }
}

// CORRECT: Iterate over copy
public void notifyObservers() {
    for (Observer observer : new ArrayList<>(observers)) {
        observer.update(this);
    }
}
```

### Pitfall 2: Memory Leaks

```java
// PROBLEM: Observer holds reference to subject, subject holds reference to observer
// If observer is no longer needed but not unregistered, memory leak!

// SOLUTION 1: Always unregister
public void cleanup() {
    subject.removeObserver(this);
}

// SOLUTION 2: Use WeakReference
private List<WeakReference<Observer>> observers = new ArrayList<>();

public void registerObserver(Observer observer) {
    observers.add(new WeakReference<>(observer));
}

public void notifyObservers() {
    Iterator<WeakReference<Observer>> it = observers.iterator();
    while (it.hasNext()) {
        Observer observer = it.next().get();
        if (observer == null) {
            it.remove();  // Remove garbage collected observers
        } else {
            observer.update(this);
        }
    }
}
```

### Pitfall 3: Notification Order

```java
// PROBLEM: Observers may depend on notification order
// SOLUTION: Document that order is not guaranteed, or use priority queue

public class PrioritySubject implements Subject {
    private PriorityQueue<PriorityObserver> observers = 
        new PriorityQueue<>(Comparator.comparingInt(PriorityObserver::getPriority));
    
    // Notify in priority order
}
```

### Pitfall 4: Infinite Loops

```java
// PROBLEM: Observer A updates Subject B, which notifies Observer C,
// which updates Subject A, creating infinite loop

// SOLUTION: Add flag to prevent re-entry
private boolean notifying = false;

public void notifyObservers() {
    if (notifying) return;  // Prevent re-entry
    
    notifying = true;
    try {
        for (Observer observer : new ArrayList<>(observers)) {
            observer.update(this);
        }
    } finally {
        notifying = false;
    }
}
```

### Pitfall 5: Exception Handling

```java
// PROBLEM: If one observer throws exception, others don't get notified

// SOLUTION: Catch exceptions and continue
public void notifyObservers() {
    for (Observer observer : new ArrayList<>(observers)) {
        try {
            observer.update(this);
        } catch (Exception e) {
            // Log error but continue notifying others
            System.err.println("Observer failed: " + e.getMessage());
        }
    }
}
```

---

## 9. Things to Remember

### Design Decisions

1. **Push vs Pull**: Choose based on how much data observers need
   - Push: Subject sends all data (simpler for observers)
   - Pull: Observers request what they need (more flexible)

2. **Update Granularity**: Notify on every change or batch changes?
   ```java
   // Option 1: Notify on every change
   public void setPrice(double price) {
       this.price = price;
       notifyObservers();
   }
   
   // Option 2: Batch changes
   public void beginUpdate() { updating = true; }
   public void endUpdate() { 
       updating = false; 
       notifyObservers(); 
   }
   ```

3. **Observer Registration**: Who registers observers?
   - Subject registers itself (auto-registration)
   - Client code registers (explicit control)
   - Factory/Builder registers (centralized configuration)

### Thread Safety

```java
// Make observer list thread-safe
private List<Observer> observers = 
    Collections.synchronizedList(new ArrayList<>());

// Or use CopyOnWriteArrayList (better for read-heavy)
private List<Observer> observers = new CopyOnWriteArrayList<>();

// Synchronize notification
public synchronized void notifyObservers() {
    for (Observer observer : observers) {
        observer.update(this);
    }
}
```

### Performance Considerations

```java
// For many observers, consider async notification
private ExecutorService executor = Executors.newFixedThreadPool(10);

public void notifyObservers() {
    for (Observer observer : new ArrayList<>(observers)) {
        executor.submit(() -> observer.update(this));
    }
}
```

---

## 10. When to Use Observer Pattern

### Use When:
✅ One object changing should trigger updates in multiple objects
✅ You want loose coupling between subject and observers
✅ Number of observers can change at runtime
✅ You need to broadcast changes to unknown number of objects

### Don't Use When:
❌ Simple one-to-one relationships (use direct method call)
❌ Observers need to be notified in specific order (use Chain of Responsibility)
❌ Performance is critical and you have many observers (consider event bus)
❌ You need guaranteed delivery (use message queue)

---

## 11. Real-World Examples

- **Java**: `java.util.Observer` and `java.util.Observable` (deprecated in Java 9)
- **JavaScript**: Event listeners (`addEventListener`)
- **Android**: `LiveData`, `Observable`
- **Spring**: `ApplicationEvent` and `ApplicationListener`
- **RxJava**: `Observable` and `Observer`
- **GUI Frameworks**: Button click listeners, model-view binding

---

## 12. Quick Implementation Checklist

When implementing Observer pattern in an interview:

1. ☐ Define `Observer` interface with `update()` method
2. ☐ Define `Subject` interface with `register()`, `remove()`, `notify()` methods
3. ☐ Implement concrete subject with:
   - ☐ List of observers
   - ☐ State variables
   - ☐ State modification methods that call `notifyObservers()`
4. ☐ Implement concrete observers with:
   - ☐ `update()` method that reacts to changes
   - ☐ Logic to extract needed data from subject
5. ☐ Handle edge cases:
   - ☐ Null checks when registering
   - ☐ Duplicate registration prevention
   - ☐ Copy list before iterating (avoid ConcurrentModificationException)
   - ☐ Exception handling in notification loop
6. ☐ Demonstrate usage:
   - ☐ Create subject
   - ☐ Create and register observers
   - ☐ Change subject state
   - ☐ Show observers being notified