## 1. What is Facade Pattern?

Facade pattern provides a simplified interface to a complex subsystem. It defines a higher-level interface that makes the subsystem easier to use.

```
Without Facade:
  tv.turnOn();
  soundSystem.turnOn();
  soundSystem.setVolume(10);
  lights.dim();
  projector.turnOn();
  // Client needs to know all subsystems

With Facade:
  homeTheater.watchMovie();
  // Single simple interface
```

**Key principle:** Simplify complex subsystems with a unified interface.

---

## 2. Implementation Guide

### Step 1: Identify Complex Subsystems

```java
public class TV {
    public void turnOn() {
        System.out.println("TV is ON");
    }
    
    public void turnOff() {
        System.out.println("TV is OFF");
    }
    
    public void setInput(String input) {
        System.out.println("TV input set to " + input);
    }
}

public class SoundSystem {
    public void turnOn() {
        System.out.println("Sound system is ON");
    }
    
    public void turnOff() {
        System.out.println("Sound system is OFF");
    }
    
    public void setVolume(int level) {
        System.out.println("Volume set to " + level);
    }
    
    public void setSurroundSound() {
        System.out.println("Surround sound enabled");
    }
}

public class Lights {
    public void dim() {
        System.out.println("Lights dimmed");
    }
    
    public void on() {
        System.out.println("Lights are ON");
    }
}

public class Projector {
    public void turnOn() {
        System.out.println("Projector is ON");
    }
    
    public void turnOff() {
        System.out.println("Projector is OFF");
    }
    
    public void setInput(String input) {
        System.out.println("Projector input set to " + input);
    }
}
```

### Step 2: Create Facade

```java
public class HomeTheaterFacade {
    private TV tv;
    private SoundSystem soundSystem;
    private Lights lights;
    private Projector projector;
    
    public HomeTheaterFacade(TV tv, SoundSystem soundSystem, 
                            Lights lights, Projector projector) {
        this.tv = tv;
        this.soundSystem = soundSystem;
        this.lights = lights;
        this.projector = projector;
    }
    
    public void watchMovie() {
        System.out.println("Get ready to watch a movie...");
        lights.dim();
        tv.turnOn();
        tv.setInput("HDMI");
        soundSystem.turnOn();
        soundSystem.setVolume(10);
        soundSystem.setSurroundSound();
        projector.turnOn();
        projector.setInput("HDMI");
        System.out.println("Movie ready!");
    }
    
    public void endMovie() {
        System.out.println("Shutting down theater...");
        projector.turnOff();
        soundSystem.turnOff();
        tv.turnOff();
        lights.on();
        System.out.println("Theater shut down!");
    }
    
    public void listenToMusic() {
        System.out.println("Setting up music...");
        lights.dim();
        soundSystem.turnOn();
        soundSystem.setVolume(8);
        System.out.println("Music ready!");
    }
}
```

### Step 3: Usage

```java
// Without facade - complex
TV tv = new TV();
SoundSystem sound = new SoundSystem();
Lights lights = new Lights();
Projector projector = new Projector();

lights.dim();
tv.turnOn();
tv.setInput("HDMI");
sound.turnOn();
sound.setVolume(10);
sound.setSurroundSound();
projector.turnOn();
projector.setInput("HDMI");

// With facade - simple
HomeTheaterFacade theater = new HomeTheaterFacade(tv, sound, lights, projector);
theater.watchMovie();
theater.endMovie();
```

---

## 3. Common Interview Questions

### Q1: Computer Startup

```java
// Subsystems
public class CPU {
    public void freeze() { System.out.println("CPU freeze"); }
    public void jump(long position) { System.out.println("CPU jump to " + position); }
    public void execute() { System.out.println("CPU execute"); }
}

public class Memory {
    public void load(long position, byte[] data) {
        System.out.println("Memory load at " + position);
    }
}

public class HardDrive {
    public byte[] read(long lba, int size) {
        System.out.println("HardDrive read " + size + " bytes from " + lba);
        return new byte[size];
    }
}

// Facade
public class ComputerFacade {
    private CPU cpu;
    private Memory memory;
    private HardDrive hardDrive;
    
    public ComputerFacade() {
        this.cpu = new CPU();
        this.memory = new Memory();
        this.hardDrive = new HardDrive();
    }
    
    public void start() {
        System.out.println("Starting computer...");
        cpu.freeze();
        memory.load(0, hardDrive.read(0, 1024));
        cpu.jump(0);
        cpu.execute();
        System.out.println("Computer started!");
    }
}

// Usage
ComputerFacade computer = new ComputerFacade();
computer.start();
```

### Q2: Order Processing

```java
// Subsystems
public class InventoryService {
    public boolean checkStock(String productId, int quantity) {
        System.out.println("Checking stock for " + productId);
        return true;
    }
    
    public void reserveStock(String productId, int quantity) {
        System.out.println("Reserving " + quantity + " units of " + productId);
    }
}

public class PaymentService {
    public boolean processPayment(String userId, double amount) {
        System.out.println("Processing payment of $" + amount + " for " + userId);
        return true;
    }
}

public class ShippingService {
    public void shipOrder(String orderId, String address) {
        System.out.println("Shipping order " + orderId + " to " + address);
    }
}

public class NotificationService {
    public void sendConfirmation(String userId, String orderId) {
        System.out.println("Sending confirmation to " + userId + " for order " + orderId);
    }
}

// Facade
public class OrderFacade {
    private InventoryService inventory;
    private PaymentService payment;
    private ShippingService shipping;
    private NotificationService notification;
    
    public OrderFacade() {
        this.inventory = new InventoryService();
        this.payment = new PaymentService();
        this.shipping = new ShippingService();
        this.notification = new NotificationService();
    }
    
    public boolean placeOrder(String userId, String productId, int quantity, 
                             double amount, String address) {
        System.out.println("Processing order...");
        
        // Check inventory
        if (!inventory.checkStock(productId, quantity)) {
            System.out.println("Out of stock");
            return false;
        }
        
        // Process payment
        if (!payment.processPayment(userId, amount)) {
            System.out.println("Payment failed");
            return false;
        }
        
        // Reserve stock
        inventory.reserveStock(productId, quantity);
        
        // Ship order
        String orderId = "ORD-" + System.currentTimeMillis();
        shipping.shipOrder(orderId, address);
        
        // Send notification
        notification.sendConfirmation(userId, orderId);
        
        System.out.println("Order placed successfully!");
        return true;
    }
}

// Usage
OrderFacade orderFacade = new OrderFacade();
orderFacade.placeOrder("user123", "prod456", 2, 99.99, "123 Main St");
```

### Q3: Hotel Booking

```java
// Subsystems
public class RoomService {
    public boolean checkAvailability(String roomType, String date) {
        System.out.println("Checking " + roomType + " availability for " + date);
        return true;
    }
    
    public void bookRoom(String roomType, String date) {
        System.out.println("Booking " + roomType + " for " + date);
    }
}

public class RestaurantService {
    public void reserveTable(int guests, String time) {
        System.out.println("Reserving table for " + guests + " at " + time);
    }
}

public class SpaService {
    public void bookAppointment(String service, String time) {
        System.out.println("Booking " + service + " at " + time);
    }
}

public class TransportService {
    public void arrangePickup(String location, String time) {
        System.out.println("Arranging pickup from " + location + " at " + time);
    }
}

// Facade
public class HotelFacade {
    private RoomService roomService;
    private RestaurantService restaurantService;
    private SpaService spaService;
    private TransportService transportService;
    
    public HotelFacade() {
        this.roomService = new RoomService();
        this.restaurantService = new RestaurantService();
        this.spaService = new SpaService();
        this.transportService = new TransportService();
    }
    
    public void bookLuxuryPackage(String date, String pickupLocation) {
        System.out.println("Booking luxury package...");
        
        roomService.checkAvailability("Suite", date);
        roomService.bookRoom("Suite", date);
        
        restaurantService.reserveTable(2, "7:00 PM");
        
        spaService.bookAppointment("Massage", "3:00 PM");
        
        transportService.arrangePickup(pickupLocation, "2:00 PM");
        
        System.out.println("Luxury package booked!");
    }
    
    public void bookBasicPackage(String date) {
        System.out.println("Booking basic package...");
        
        roomService.checkAvailability("Standard", date);
        roomService.bookRoom("Standard", date);
        
        System.out.println("Basic package booked!");
    }
}

// Usage
HotelFacade hotel = new HotelFacade();
hotel.bookLuxuryPackage("2024-12-25", "Airport");
hotel.bookBasicPackage("2024-12-26");
```

---

## 4. Interview Tricks & Pitfalls

### Pitfall 1: Facade Becomes God Object

```java
// WRONG: Facade does everything
public class Facade {
    public void doEverything() {
        // Too much responsibility
    }
}

// CORRECT: Multiple focused facades
public class OrderFacade { /* Order operations */ }
public class UserFacade { /* User operations */ }
```

### Pitfall 2: Tight Coupling

```java
// WRONG: Facade creates subsystems
public class Facade {
    private SubsystemA a = new SubsystemA();  // Tight coupling
}

// CORRECT: Dependency injection
public class Facade {
    private SubsystemA a;
    
    public Facade(SubsystemA a) {
        this.a = a;
    }
}
```

### Pitfall 3: Exposing Subsystems

```java
// WRONG: Exposing subsystems defeats purpose
public class Facade {
    public SubsystemA getSubsystemA() {
        return subsystemA;
    }
}

// CORRECT: Keep subsystems private
public class Facade {
    private SubsystemA subsystemA;
    // No getter
}
```

---

## 5. Facade vs Other Patterns

| Pattern | Purpose | Complexity |
|---------|---------|------------|
| Facade | Simplify interface | Hides complexity |
| Adapter | Make compatible | Translates interface |
| Proxy | Control access | Same interface |
| Decorator | Add behavior | Wraps object |

---

## 6. When to Use

✅ **Use when:**
- Complex subsystem with many classes
- Want to provide simple interface
- Decouple client from subsystem
- Layer your application

❌ **Don't use when:**
- Subsystem is already simple
- Need fine-grained control
- Facade becomes too complex

---

## 7. Real-World Examples

- **JDBC**: `DriverManager` simplifies database access
- **SLF4J**: Logging facade for various logging frameworks
- **Spring**: `JdbcTemplate` simplifies JDBC operations
- **Compiler**: Front-end facade for lexer, parser, semantic analyzer

---

## 8. Quick Checklist

1. ☐ Identify complex subsystems
2. ☐ Create facade class
3. ☐ Facade holds references to subsystems
4. ☐ Provide simple methods in facade
5. ☐ Each method coordinates multiple subsystems
6. ☐ Keep subsystems private
7. ☐ Demonstrate simplified usage

**Template:**
```java
// Subsystems
public class SubsystemA {
    public void operationA() { }
}

public class SubsystemB {
    public void operationB() { }
}

// Facade
public class Facade {
    private SubsystemA a;
    private SubsystemB b;
    
    public Facade(SubsystemA a, SubsystemB b) {
        this.a = a;
        this.b = b;
    }
    
    public void simpleOperation() {
        a.operationA();
        b.operationB();
    }
}
```
