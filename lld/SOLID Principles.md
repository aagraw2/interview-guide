
## Overview
SOLID are five design principles that help keep code maintainable by managing responsibilities and dependencies.

## 1. Single Responsibility Principle (SRP)

**Definition:**  
A class should have only one reason to change. Each class handles **one responsibility**.

**Implementation Tips:**

- Split functionalities into separate classes.
    
- Keep data access, business logic, and UI logic separate.
    
- Use small helper classes for repetitive tasks.
    

**Example:**

```java
// SRP: Payment processing separated from notification
public class PaymentService {
    public void processPayment(Order order) {
        // Payment processing logic
    }
}

public class NotificationService {
    public void sendNotification(User user, String message) {
        // Notification logic
    }
}
```

**Common Mistakes:**

- God classes handling multiple responsibilities.
    
- Mixing logging, validation, and business logic in a single class.
    

---

## 2. Open/Closed Principle (OCP)

**Definition:**  
Software entities should be **open for extension, closed for modification**.

**Implementation Tips:**

- Use interfaces or abstract classes to extend functionality.
    
- Apply Strategy or Decorator patterns to add behavior without modifying existing classes.
    

**Example:**

```java
public interface PaymentMethod {
    void pay(double amount);
}

public class CreditCard implements PaymentMethod {
    public void pay(double amount) {
        // Credit card payment logic
    }
}

public class PayPal implements PaymentMethod {
    public void pay(double amount) {
        // PayPal payment logic
    }
}
```

**Common Mistakes:**

- Directly modifying existing classes to add new features.
    

---

## 3. Liskov Substitution Principle (LSP)

**Definition:**  
Subtypes must be substitutable for their base types without breaking behavior.

**Implementation Tips:**

- Ensure derived classes honor contracts of the base class.
    
- Avoid throwing exceptions or changing expected behavior in overrides.
    

**Example:**

```java
public interface Shape {
    double area();
}

public class Rectangle implements Shape {
    private double width;
    private double height;

    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }

    public double area() {
        return width * height;
    }
}

public class Square implements Shape {
    private double side;

    public Square(double side) {
        this.side = side;
    }

    public double area() {
        return side * side;
    }
}
```

**Common Mistakes:**

- Overriding methods to throw exceptions or return invalid data.
    

---

## 4. Interface Segregation Principle (ISP)

**Definition:**  
Clients should not be forced to depend on interfaces they do not use.

**Implementation Tips:**

- Split large interfaces into smaller, focused ones.
    
- Each service or class should implement only what it needs.
    

**Example:**

```java
public interface Reader {
    int read(byte[] bytes);
}

public interface Writer {
    void write(byte[] bytes);
}
```

**Common Mistakes:**

- Creating large, “fat” interfaces with unused methods.
    

---

## 5. Dependency Inversion Principle (DIP)

**Definition:**  
High-level modules should not depend on low-level modules. Both should depend on **abstractions**.

**Implementation Tips:**

- Depend on interfaces or abstract classes, not concrete implementations.
    
- Use Dependency Injection (DI) to pass dependencies.
    

**Example:**

```java
public interface Notifier {
    void send(String message);
}

public class EmailNotifier implements Notifier {
    public void send(String message) {
        // Send email
    }
}

public class UserService {
    private Notifier notifier;

    public UserService(Notifier notifier) {
        this.notifier = notifier;
    }

    public void notifyUser(String message) {
        notifier.send(message);
    }
}
```

**Common Mistakes:**

- High-level classes directly creating low-level instances.
    
- Tight coupling between modules reduces testability.
    

---

## Patterns Supporting SOLID in LLD

|Principle|Patterns / Techniques|
|---|---|
|SRP|Adapter, Facade, Helper classes|
|OCP|Strategy, Decorator, Template Method|
|LSP|Polymorphism, Subclassing|
|ISP|Interface Segregation, Multiple small interfaces|
|DIP|Dependency Injection, Factory Pattern|

---

## Summary
- **SRP**: one reason to change per class.
- **OCP**: extend behavior without modifying existing code.
- **LSP**: subtypes must remain substitutable.
- **ISP**: prefer small, role-focused interfaces.
- **DIP**: depend on abstractions; inject implementations.
