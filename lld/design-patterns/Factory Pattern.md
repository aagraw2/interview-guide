## 1. What is Factory Pattern?

The Factory pattern provides an interface for creating objects without specifying their exact classes. It delegates object creation to factory methods, promoting loose coupling.

```
Without Factory:
  if (type.equals("CAR")) {
      vehicle = new Car();
  } else if (type.equals("BIKE")) {
      vehicle = new Bike();
  }
  // Client knows all concrete classes

With Factory:
  vehicle = VehicleFactory.create(type);
  // Client doesn't know concrete classes
```

**Key principle:** Encapsulate object creation logic.

---

## 2. Types of Factory Patterns

### Simple Factory (Not a GoF Pattern)

Single factory class with creation method.

### Factory Method Pattern

Subclasses decide which class to instantiate.

### Abstract Factory Pattern

Factory of factories for related object families.

---

## 3. Simple Factory Implementation

### Step 1: Define Product Interface

```java
// Vehicle.java
public interface Vehicle {
    String getName();
    int getPrice();
    void start();
}
```

### Step 2: Create Concrete Products

```java
// Car.java
public class Car implements Vehicle {
    @Override
    public String getName() {
        return "Car";
    }
    
    @Override
    public int getPrice() {
        return 1500000;
    }
    
    @Override
    public void start() {
        System.out.println("Car engine started");
    }
}

// Bike.java
public class Bike implements Vehicle {
    @Override
    public String getName() {
        return "Bike";
    }
    
    @Override
    public int getPrice() {
        return 200000;
    }
    
    @Override
    public void start() {
        System.out.println("Bike engine started");
    }
}

// Truck.java
public class Truck implements Vehicle {
    @Override
    public String getName() {
        return "Truck";
    }
    
    @Override
    public int getPrice() {
        return 3000000;
    }
    
    @Override
    public void start() {
        System.out.println("Truck engine started");
    }
}
```

### Step 3: Create Factory Class

```java
// VehicleFactory.java
public class VehicleFactory {
    
    // Simple factory method
    public static Vehicle createVehicle(String type) {
        if (type == null || type.isEmpty()) {
            throw new IllegalArgumentException("Vehicle type cannot be null");
        }
        
        switch (type.toUpperCase()) {
            case "CAR":
                return new Car();
            case "BIKE":
                return new Bike();
            case "TRUCK":
                return new Truck();
            default:
                throw new IllegalArgumentException("Unknown vehicle type: " + type);
        }
    }
}

// Usage
Vehicle car = VehicleFactory.createVehicle("CAR");
car.start();
System.out.println(car.getName() + " costs " + car.getPrice());
```

**Pros:**
- ✅ Centralizes object creation
- ✅ Client doesn't know concrete classes

**Cons:**
- ❌ Violates Open/Closed Principle (need to modify factory for new types)
- ❌ Factory class can become large

---

## 4. Factory Method Pattern (GoF)

### Step 1: Define Creator Interface

```java
// VehicleFactory.java (abstract)
public abstract class VehicleFactory {
    
    // Factory method - subclasses implement
    public abstract Vehicle createVehicle();
    
    // Template method using factory method
    public Vehicle orderVehicle() {
        Vehicle vehicle = createVehicle();
        
        System.out.println("Processing order for " + vehicle.getName());
        vehicle.start();
        System.out.println("Vehicle ready for delivery");
        
        return vehicle;
    }
}
```

### Step 2: Create Concrete Factories

```java
// CarFactory.java
public class CarFactory extends VehicleFactory {
    @Override
    public Vehicle createVehicle() {
        return new Car();
    }
}

// BikeFactory.java
public class BikeFactory extends VehicleFactory {
    @Override
    public Vehicle createVehicle() {
        return new Bike();
    }
}

// TruckFactory.java
public class TruckFactory extends VehicleFactory {
    @Override
    public Vehicle createVehicle() {
        return new Truck();
    }
}

// Usage
VehicleFactory factory = new CarFactory();
Vehicle car = factory.orderVehicle();

factory = new BikeFactory();
Vehicle bike = factory.orderVehicle();
```

**Pros:**
- ✅ Follows Open/Closed Principle (add new factories without modifying existing code)
- ✅ Single Responsibility (each factory creates one type)

**Cons:**
- ❌ More classes to maintain
- ❌ Can be overkill for simple scenarios

---

## 5. Registry-Based Factory (Modern Approach)

Flexible factory using registration.

```java
// VehicleFactory.java
import java.util.*;
import java.util.function.Supplier;

public class VehicleFactory {
    
    // Registry maps type to creator function
    private final Map<String, Supplier<Vehicle>> registry = new HashMap<>();
    
    // Register a vehicle type
    public void register(String type, Supplier<Vehicle> creator) {
        registry.put(type.toUpperCase(), creator);
    }
    
    // Create vehicle by type
    public Vehicle createVehicle(String type) {
        Supplier<Vehicle> creator = registry.get(type.toUpperCase());
        
        if (creator == null) {
            throw new IllegalArgumentException("Unknown vehicle type: " + type);
        }
        
        return creator.get();
    }
    
    // Get all registered types
    public Set<String> getAvailableTypes() {
        return registry.keySet();
    }
}

// Usage
VehicleFactory factory = new VehicleFactory();

// Register vehicle types
factory.register("CAR", Car::new);
factory.register("BIKE", Bike::new);
factory.register("TRUCK", Truck::new);

// Create vehicles
Vehicle car = factory.createVehicle("CAR");
Vehicle bike = factory.createVehicle("BIKE");

// Can register new types at runtime
factory.register("SCOOTER", Scooter::new);
Vehicle scooter = factory.createVehicle("SCOOTER");
```

**Pros:**
- ✅ Highly flexible (register types at runtime)
- ✅ No need to modify factory for new types
- ✅ Clean and maintainable

**Cons:**
- ❌ Requires Java 8+ (Supplier interface)
- ❌ Type safety only at runtime

---

## 6. Factory with Parameters

Pass parameters to factory for customization.

```java
// Vehicle.java (with properties)
public interface Vehicle {
    String getName();
    int getPrice();
    String getColor();
    void start();
}

// Car.java (with constructor parameters)
public class Car implements Vehicle {
    private String color;
    private String model;
    
    public Car(String color, String model) {
        this.color = color;
        this.model = model;
    }
    
    @Override
    public String getName() {
        return model + " Car";
    }
    
    @Override
    public String getColor() {
        return color;
    }
    
    @Override
    public int getPrice() {
        return 1500000;
    }
    
    @Override
    public void start() {
        System.out.println(color + " " + model + " car started");
    }
}

// VehicleFactory.java (with parameters)
public class VehicleFactory {
    
    public static Vehicle createVehicle(String type, String color, String model) {
        switch (type.toUpperCase()) {
            case "CAR":
                return new Car(color, model);
            case "BIKE":
                return new Bike(color, model);
            case "TRUCK":
                return new Truck(color, model);
            default:
                throw new IllegalArgumentException("Unknown type: " + type);
        }
    }
}

// Usage
Vehicle redSedan = VehicleFactory.createVehicle("CAR", "Red", "Sedan");
Vehicle blueSUV = VehicleFactory.createVehicle("CAR", "Blue", "SUV");
```

---

## 7. Common Interview Questions & Implementations

### Q1: Implement Shape Factory

```java
// Shape.java
public interface Shape {
    void draw();
    double area();
}

// Circle.java
public class Circle implements Shape {
    private double radius;
    
    public Circle(double radius) {
        this.radius = radius;
    }
    
    @Override
    public void draw() {
        System.out.println("Drawing Circle with radius " + radius);
    }
    
    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

// Rectangle.java
public class Rectangle implements Shape {
    private double width;
    private double height;
    
    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }
    
    @Override
    public void draw() {
        System.out.println("Drawing Rectangle " + width + "x" + height);
    }
    
    @Override
    public double area() {
        return width * height;
    }
}

// ShapeFactory.java
public class ShapeFactory {
    
    public static Shape createShape(String type, double... dimensions) {
        switch (type.toUpperCase()) {
            case "CIRCLE":
                if (dimensions.length < 1) {
                    throw new IllegalArgumentException("Circle needs radius");
                }
                return new Circle(dimensions[0]);
                
            case "RECTANGLE":
                if (dimensions.length < 2) {
                    throw new IllegalArgumentException("Rectangle needs width and height");
                }
                return new Rectangle(dimensions[0], dimensions[1]);
                
            default:
                throw new IllegalArgumentException("Unknown shape: " + type);
        }
    }
}

// Usage
Shape circle = ShapeFactory.createShape("CIRCLE", 5.0);
circle.draw();
System.out.println("Area: " + circle.area());

Shape rectangle = ShapeFactory.createShape("RECTANGLE", 4.0, 6.0);
rectangle.draw();
System.out.println("Area: " + rectangle.area());
```

### Q2: Implement Notification Factory

```java
// Notification.java
public interface Notification {
    void send(String recipient, String message);
}

// EmailNotification.java
public class EmailNotification implements Notification {
    @Override
    public void send(String recipient, String message) {
        System.out.println("Sending email to " + recipient);
        System.out.println("Message: " + message);
    }
}

// SMSNotification.java
public class SMSNotification implements Notification {
    @Override
    public void send(String recipient, String message) {
        System.out.println("Sending SMS to " + recipient);
        System.out.println("Message: " + message);
    }
}

// PushNotification.java
public class PushNotification implements Notification {
    @Override
    public void send(String recipient, String message) {
        System.out.println("Sending push notification to " + recipient);
        System.out.println("Message: " + message);
    }
}

// NotificationFactory.java
public class NotificationFactory {
    
    private static final Map<String, Supplier<Notification>> registry = new HashMap<>();
    
    static {
        // Register notification types
        registry.put("EMAIL", EmailNotification::new);
        registry.put("SMS", SMSNotification::new);
        registry.put("PUSH", PushNotification::new);
    }
    
    public static Notification createNotification(String type) {
        Supplier<Notification> creator = registry.get(type.toUpperCase());
        
        if (creator == null) {
            throw new IllegalArgumentException("Unknown notification type: " + type);
        }
        
        return creator.get();
    }
    
    // Send notification (convenience method)
    public static void sendNotification(String type, String recipient, String message) {
        Notification notification = createNotification(type);
        notification.send(recipient, message);
    }
}

// Usage
NotificationFactory.sendNotification("EMAIL", "user@example.com", "Hello!");
NotificationFactory.sendNotification("SMS", "+1234567890", "Your OTP is 123456");
NotificationFactory.sendNotification("PUSH", "user123", "New message received");
```

### Q3: Implement Database Connection Factory

```java
// DatabaseConnection.java
public interface DatabaseConnection {
    void connect();
    void disconnect();
    void executeQuery(String query);
}

// MySQLConnection.java
public class MySQLConnection implements DatabaseConnection {
    private String host;
    private int port;
    
    public MySQLConnection(String host, int port) {
        this.host = host;
        this.port = port;
    }
    
    @Override
    public void connect() {
        System.out.println("Connecting to MySQL at " + host + ":" + port);
    }
    
    @Override
    public void disconnect() {
        System.out.println("Disconnecting from MySQL");
    }
    
    @Override
    public void executeQuery(String query) {
        System.out.println("MySQL executing: " + query);
    }
}

// PostgreSQLConnection.java
public class PostgreSQLConnection implements DatabaseConnection {
    private String host;
    private int port;
    
    public PostgreSQLConnection(String host, int port) {
        this.host = host;
        this.port = port;
    }
    
    @Override
    public void connect() {
        System.out.println("Connecting to PostgreSQL at " + host + ":" + port);
    }
    
    @Override
    public void disconnect() {
        System.out.println("Disconnecting from PostgreSQL");
    }
    
    @Override
    public void executeQuery(String query) {
        System.out.println("PostgreSQL executing: " + query);
    }
}

// DatabaseConnectionFactory.java
public class DatabaseConnectionFactory {
    
    public static DatabaseConnection createConnection(String dbType, String host, int port) {
        switch (dbType.toUpperCase()) {
            case "MYSQL":
                return new MySQLConnection(host, port);
            case "POSTGRESQL":
                return new PostgreSQLConnection(host, port);
            case "MONGODB":
                return new MongoDBConnection(host, port);
            default:
                throw new IllegalArgumentException("Unsupported database: " + dbType);
        }
    }
    
    // Convenience method with default port
    public static DatabaseConnection createConnection(String dbType, String host) {
        int defaultPort = getDefaultPort(dbType);
        return createConnection(dbType, host, defaultPort);
    }
    
    private static int getDefaultPort(String dbType) {
        switch (dbType.toUpperCase()) {
            case "MYSQL": return 3306;
            case "POSTGRESQL": return 5432;
            case "MONGODB": return 27017;
            default: return 0;
        }
    }
}

// Usage
DatabaseConnection mysql = DatabaseConnectionFactory.createConnection("MYSQL", "localhost");
mysql.connect();
mysql.executeQuery("SELECT * FROM users");
mysql.disconnect();
```

---

## 8. Interview Tricks & Pitfalls

### Pitfall 1: Tight Coupling with Switch/If-Else

```java
// PROBLEM: Need to modify factory for every new type
public static Vehicle createVehicle(String type) {
    switch (type) {
        case "CAR": return new Car();
        case "BIKE": return new Bike();
        // Adding new type requires modifying this method
    }
}

// SOLUTION: Use registry-based factory
private Map<String, Supplier<Vehicle>> registry = new HashMap<>();

public void register(String type, Supplier<Vehicle> creator) {
    registry.put(type, creator);
}

// Add new types without modifying factory
factory.register("SCOOTER", Scooter::new);
```

### Pitfall 2: Not Handling Invalid Types

```java
// WRONG: Returns null
public static Vehicle createVehicle(String type) {
    if (type.equals("CAR")) return new Car();
    return null;  // Causes NullPointerException later
}

// CORRECT: Throw exception
public static Vehicle createVehicle(String type) {
    if (type.equals("CAR")) return new Car();
    throw new IllegalArgumentException("Unknown type: " + type);
}
```

### Pitfall 3: Not Validating Parameters

```java
// WRONG: No validation
public static Shape createShape(String type, double... dimensions) {
    return new Circle(dimensions[0]);  // ArrayIndexOutOfBoundsException if empty
}

// CORRECT: Validate parameters
public static Shape createShape(String type, double... dimensions) {
    if (dimensions == null || dimensions.length == 0) {
        throw new IllegalArgumentException("Dimensions required");
    }
    if (dimensions[0] <= 0) {
        throw new IllegalArgumentException("Dimensions must be positive");
    }
    return new Circle(dimensions[0]);
}
```

### Pitfall 4: Creating Expensive Objects Repeatedly

```java
// PROBLEM: Creating new instance every time
public static Logger createLogger() {
    return new Logger();  // Expensive initialization
}

// SOLUTION: Cache instances (combine with Singleton or Flyweight)
private static final Map<String, Logger> loggerCache = new HashMap<>();

public static Logger createLogger(String name) {
    return loggerCache.computeIfAbsent(name, Logger::new);
}
```

---

## 9. Things to Remember

### When to Use Factory Pattern

✅ **Use when:**
- Object creation is complex
- Type determined at runtime
- Want to decouple client from concrete classes
- Need centralized creation logic
- Multiple related products

❌ **Don't use when:**
- Simple object creation (just use `new`)
- Only one product type
- No variation in creation logic

### Design Decisions

1. **Simple Factory vs Factory Method:**
   - Simple: One factory class (easier, less flexible)
   - Factory Method: Multiple factory classes (more flexible, follows OCP)

2. **Static vs Instance Methods:**
   - Static: Simpler, no state needed
   - Instance: Can maintain state, more testable

3. **Registry-Based:**
   - Most flexible
   - Can register types at runtime
   - Modern approach

### Where to Put Factory

```
Project Structure:
src/
  model/
    Vehicle.java          ← Product interface
    Car.java              ← Concrete product
    Bike.java             ← Concrete product
  factory/
    VehicleFactory.java   ← Factory class
  service/
    VehicleService.java   ← Uses factory
```

---

## 10. Real-World Examples

- **Java**: `Calendar.getInstance()`, `NumberFormat.getInstance()`
- **JDBC**: `DriverManager.getConnection()`
- **Spring**: `BeanFactory`, `ApplicationContext`
- **Collections**: `Collections.synchronizedList()`
- **Logging**: `LoggerFactory.getLogger()` (SLF4J)

---

## 11. Quick Implementation Checklist

When implementing Factory in an interview:

1. ☐ Define product interface
2. ☐ Create concrete product classes
3. ☐ Create factory class with creation method
4. ☐ Handle invalid types (throw exception)
5. ☐ Validate parameters
6. ☐ Consider using registry for flexibility
7. ☐ Add convenience methods if needed
8. ☐ Demonstrate usage with multiple product types

**Recommended template (Registry-Based):**
```java
public class Factory {
    private Map<String, Supplier<Product>> registry = new HashMap<>();
    
    public void register(String type, Supplier<Product> creator) {
        registry.put(type, creator);
    }
    
    public Product create(String type) {
        Supplier<Product> creator = registry.get(type);
        if (creator == null) {
            throw new IllegalArgumentException("Unknown type: " + type);
        }
        return creator.get();
    }
}
```
