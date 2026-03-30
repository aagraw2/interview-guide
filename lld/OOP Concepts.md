## Class & Object

- **Class**: blueprint (fields + methods)
- **Object**: runtime instance of a class (has state + behavior)

```java
class Car {
    String model;
    void drive() {
        System.out.println("Driving " + model);
    }
}

Car car = new Car();
car.model = "Tesla";
car.drive();
```

---

## OOP fundamentals (core pillars)

### Encapsulation

- **What**: keep state private; expose behavior via a controlled API
- **Why**: protects invariants (valid state), reduces coupling, improves maintainability

```java
class BankAccount {
    private double balance;

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("amount");
        balance += amount;
    }

    public double getBalance() {
        return balance;
    }
}
```

### Abstraction

- **What**: expose *what* a component does, hide *how* it does it
- **How**: interfaces / abstract classes / façade APIs

```java
interface Payment {
    void pay();
}
```

### Inheritance

- **What**: reuse/extend behavior with an **is-a** relationship
- **Caution**: can increase coupling; prefer composition when behavior varies

```java
class Animal {
    void eat() { }
}

class Dog extends Animal {
    void bark() { }
}
```

### Polymorphism

- **What**: same message (method call), different runtime behavior
- **Types**:
  - **Compile-time**: method **overloading**
  - **Runtime**: method **overriding** (dynamic dispatch)

```java
interface Payment {
    void pay();
}

class CardPayment implements Payment {
    public void pay() { System.out.println("Card"); }
}

class UPIPayment implements Payment {
    public void pay() { System.out.println("UPI"); }
}
```

---

## Common OOP fundamentals interviewers expect (often missed)

### Overloading vs overriding

- **Overloading (compile-time)**: same method name, different params (same class)
- **Overriding (runtime)**: subclass changes parent behavior (same signature)

```java
class Printer {
    void print(String s) { }
    void print(int n) { }           // overloading
}

class Animal { void speak() { } }
class Dog extends Animal {
    @Override void speak() { }      // overriding
}
```

### Abstract class vs interface (Java)

- **Interface**: capability/contract; supports multiple inheritance of type
- **Abstract class**: shared base with common state/behavior; single inheritance

### Dynamic binding (runtime dispatch)

- Call resolution for overridden methods happens at runtime based on actual object type.

```java
Animal a = new Dog();
a.speak(); // calls Dog.speak() at runtime
```

### Constructors, `this`, `super`

- **Constructors**: enforce valid object creation (invariants)
- **`this`**: current object; **`super`**: parent class members/constructor

### `static`, `final` (Java)

- **static**: belongs to class, not instances
- **final**: cannot be reassigned/overridden/extended (var/method/class)

### Coupling & cohesion (design quality)

- **High cohesion**: class has one clear purpose
- **Low coupling**: minimal knowledge of other classes’ internals

---

## Access modifiers (supports encapsulation)

| Modifier | Scope |
|---|---|
| `private` | Same class only |
| *(default)* | Same package |
| `protected` | Package + subclasses |
| `public` | Everywhere |

```java
class User {
    private String password;
    public void setPassword(String password) {
        this.password = password;
    }
}
```

---

## Composition over inheritance

> Prefer **has-a** over **is-a** when behavior varies.

### Inheritance (problematic)

```java
class Bird { void fly() { } }
class Ostrich extends Bird { } // doesn't fly -> inheritance mismatch
```

### Composition (better)

```java
interface FlyBehavior { void fly(); }
class NoFly implements FlyBehavior { public void fly() { } }

class Bird {
    private FlyBehavior flyBehavior;
    Bird(FlyBehavior flyBehavior) { this.flyBehavior = flyBehavior; }
}
```

---

## Class relationships (LLD vocabulary)

### Association (uses)

```java
class Teacher {
    void teach(Student s) { }
}
```

### Aggregation (has-a, weak ownership)

- Parts can exist independently of the whole.

```java
class Department {
    List<Teacher> teachers;
}
```

### Composition (has-a, strong ownership)

- Part lifecycle depends on whole.

```java
class Engine { }
class Car {
    private Engine engine = new Engine();
}
```

### Inheritance (is-a)

```java
class Vehicle { }
class Car extends Vehicle { }
```

### Summary

| Type | Strength | Lifecycle |
|---|---|---|
| Association | Weak | Independent |
| Aggregation | Medium | Independent |
| Composition | Strong | Dependent |
| Inheritance | Strong | Dependent |

---

## Dependency Injection (DI)

> Provide dependencies from outside instead of creating them inside the class.

### Problem (tight coupling)

```java
class OrderService {
    PaymentService paymentService = new PaymentService();
}
```

### Solution (constructor DI)

```java
class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

### Types

- **Constructor injection** (preferred)
- **Setter injection**
- **Interface injection**

### Benefits

- **Loose coupling**
- **Testability** (mocking)
- **Flexibility**

---

## UML diagrams (quick interview use)

### Class diagram

- **Purpose**: static structure (classes, fields, relationships)

```
+----------------+
|   Order        |
+----------------+
| id             |
| status         |
+----------------+
| create()       |
+----------------+

        |
        | has
        v

+----------------+
| OrderItem      |
+----------------+
| itemId         |
| quantity       |
+----------------+
```

### Relationships in UML (memory cues)

- **Association**: line
- **Aggregation**: hollow diamond
- **Composition**: filled diamond
- **Inheritance**: hollow triangle arrow

### Sequence diagram

- **Purpose**: interactions over time (who calls whom, in what order)

```
User -> OrderService: createOrder()
OrderService -> InventoryService: reserve()
InventoryService -> DB: updateStock()
DB -> InventoryService: success
InventoryService -> OrderService: success
OrderService -> User: order confirmed
```

---

## Putting it together (mini example)

```java
interface Payment { void pay(); }
class CardPayment implements Payment { public void pay() { } }

class OrderService {
    private final Payment payment;
    public OrderService(Payment payment) { // DI
        this.payment = payment;
    }
    public void checkout() { payment.pay(); }
}
```

---

## Interview tips

- **Start with entities + relationships**, then refine responsibilities.
- Prefer **composition over inheritance** unless it’s a true **is-a**.
- Use **interfaces + DI** to keep dependencies swappable/testable.
- Define **encapsulation boundaries** (what’s public API vs internal).
- Use **UML** to communicate quickly (class + sequence diagrams).
- Always explain **why** you chose a relationship/pattern.

---

## Summary

- **OOP pillars**: encapsulation, abstraction, inheritance, polymorphism
- **Polymorphism**: overloading (compile-time) + overriding (runtime)
- **Design hygiene**: high cohesion, low coupling
- **Composition > inheritance** in most LLD designs
- **DI** helps reduce coupling and improve testability
- **UML** helps communicate design clearly
