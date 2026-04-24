# Design Patterns Quick Reference

Mental models, one-liners, and bare-minimum templates for quick revision before an interview.

---

## Strategy

**Mental model:** Same action, different algorithms. Swap the "how" at runtime without changing the "what".

**One-liner:** Encapsulate a family of algorithms behind an interface and make them interchangeable.

**Spot it when:** You see if-else chains choosing between different ways to do the same thing. Payment methods, sorting algorithms, pricing rules, compression formats.

**Template:**
```java
interface Strategy { void execute(); }

class ConcreteA implements Strategy { public void execute() { /* algo A */ } }
class ConcreteB implements Strategy { public void execute() { /* algo B */ } }

class Context {
    private Strategy strategy;
    public void setStrategy(Strategy s) { this.strategy = s; }
    public void run() { strategy.execute(); }
}
```

**Key pitfall:** Strategy should be stateless. If it holds mutable state, it is not thread-safe.

---

## Observer

**Mental model:** One thing changes, many things react. Like a newsletter subscription.

**One-liner:** Subject maintains a list of observers and notifies all of them when its state changes.

**Spot it when:** Multiple components need to react to a single event. Price alerts, event listeners, notification systems, stock tickers.

**Template:**
```java
interface Observer { void update(Subject s); }

interface Subject {
    void register(Observer o);
    void remove(Observer o);
    void notifyAll();
}

class ConcreteSubject implements Subject {
    private List<Observer> observers = new ArrayList<>();
    private int state;

    public void setState(int state) {
        this.state = state;
        notifyAll();
    }

    public void notifyAll() {
        for (Observer o : new ArrayList<>(observers)) o.update(this);
    }
}
```

**Key pitfall:** Iterate over a copy of the observer list to avoid `ConcurrentModificationException` if an observer removes itself during notification.

---

## Factory

**Mental model:** You ask for a product by type, the factory figures out which class to instantiate.

**One-liner:** Centralize object creation so the client never calls `new` directly on concrete classes.

**Spot it when:** Object type is determined at runtime. Database connections, notification channels, parsers, UI components.

**Template (registry-based, most flexible):**
```java
interface Product { void use(); }

class Factory {
    private Map<String, Supplier<Product>> registry = new HashMap<>();

    public void register(String type, Supplier<Product> creator) {
        registry.put(type, creator);
    }

    public Product create(String type) {
        Supplier<Product> creator = registry.get(type);
        if (creator == null) throw new IllegalArgumentException("Unknown type: " + type);
        return creator.get();
    }
}
```

**Key pitfall:** Never return null. Throw `IllegalArgumentException` for unknown types.

---

## Singleton

**Mental model:** One instance for the entire application. Everyone shares the same object.

**One-liner:** Restrict instantiation to one object and provide a global access point.

**Spot it when:** Shared resource that should only exist once. Logger, config manager, connection pool, cache.

**Template (Bill Pugh, best practice):**
```java
public class Singleton {
    private Singleton() {}

    private static class Helper {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return Helper.INSTANCE;
    }
}
```

**Double-checked locking (if you need lazy + thread-safe explicitly):**
```java
private static volatile Singleton instance;

public static Singleton getInstance() {
    if (instance == null) {
        synchronized (Singleton.class) {
            if (instance == null) instance = new Singleton();
        }
    }
    return instance;
}
```

**Key pitfall:** Reflection can break it. Add `if (instance != null) throw new RuntimeException()` in the constructor if that matters.

---

## Decorator

**Mental model:** Wrap an object to add behavior without touching the original class. Like adding toppings to a pizza.

**One-liner:** Attach additional responsibilities to an object dynamically by wrapping it.

**Spot it when:** You need flexible combinations of features. Java I/O streams, coffee shop toppings, UI component styling, middleware.

**Template:**
```java
interface Component { void operation(); }

class ConcreteComponent implements Component {
    public void operation() { System.out.println("base"); }
}

abstract class Decorator implements Component {
    protected Component wrapped;
    public Decorator(Component c) { this.wrapped = c; }
    public void operation() { wrapped.operation(); }
}

class ConcreteDecorator extends Decorator {
    public ConcreteDecorator(Component c) { super(c); }
    public void operation() {
        super.operation();
        System.out.println("extra behavior");
    }
}

// Usage
Component c = new ConcreteDecorator(new ConcreteDecorator(new ConcreteComponent()));
```

**Key pitfall:** Order of wrapping matters. `Encrypt(Compress(data))` is different from `Compress(Encrypt(data))`.

---

## Builder

**Mental model:** Construct a complex object step by step. Like filling out a form before submitting.

**One-liner:** Separate the construction of a complex object from its representation using a fluent interface.

**Spot it when:** Object has many optional parameters, or you want immutable objects with readable construction. HTTP requests, SQL queries, config objects.

**Template:**
```java
public class Product {
    private final String required;
    private final int optional;

    private Product(Builder b) {
        this.required = b.required;
        this.optional = b.optional;
    }

    public static class Builder {
        private final String required;
        private int optional = 0;

        public Builder(String required) { this.required = required; }

        public Builder optional(int val) { this.optional = val; return this; }

        public Product build() {
            // validate here
            return new Product(this);
        }
    }
}
```

**Key pitfall:** Always return `this` from setter methods. Validate in `build()`, not in individual setters.

---

## Adapter

**Mental model:** A power plug adapter. The device and the socket are incompatible, the adapter makes them work together.

**One-liner:** Convert the interface of a class into another interface the client expects.

**Spot it when:** Integrating third-party libraries or legacy code whose interface does not match what your code expects.

**Template:**
```java
interface Target { void request(); }

class Adaptee { public void specificRequest() { /* existing logic */ } }

class Adapter implements Target {
    private Adaptee adaptee;
    public Adapter(Adaptee a) { this.adaptee = a; }
    public void request() { adaptee.specificRequest(); }
}
```

**Key pitfall:** Adapter should only translate, not add business logic. Translate exceptions too if the adaptee throws different exception types.

---

## Facade

**Mental model:** A hotel concierge. You say "book me a trip" and they coordinate flights, hotel, and taxi without you touching any of it.

**One-liner:** Provide a simplified interface to a complex subsystem.

**Spot it when:** Client needs to coordinate multiple subsystems to do one thing. Order processing, home theater control, computer startup sequence.

**Template:**
```java
class SubsystemA { public void doA() {} }
class SubsystemB { public void doB() {} }

class Facade {
    private SubsystemA a;
    private SubsystemB b;

    public Facade(SubsystemA a, SubsystemB b) {
        this.a = a;
        this.b = b;
    }

    public void simpleOperation() {
        a.doA();
        b.doB();
    }
}
```

**Key pitfall:** Do not expose subsystem references through the facade. Keep them private. If the facade grows too large, split it into multiple focused facades.

---

## Command

**Mental model:** Wrap a request in an object so you can queue it, log it, or undo it later.

**One-liner:** Encapsulate a request as an object, decoupling the sender from the receiver.

**Spot it when:** You need undo/redo, task queues, macro recording, or transaction logging.

**Template:**
```java
interface Command {
    void execute();
    void undo();
}

class Receiver { public void action() {} }

class ConcreteCommand implements Command {
    private Receiver receiver;
    public ConcreteCommand(Receiver r) { this.receiver = r; }
    public void execute() { receiver.action(); }
    public void undo() { /* reverse action */ }
}

class Invoker {
    private Stack<Command> history = new Stack<>();
    public void run(Command c) { c.execute(); history.push(c); }
    public void undo() { if (!history.isEmpty()) history.pop().undo(); }
}
```

**Key pitfall:** Store previous state before executing if you need undo. Do not forget to push to history after execute.

---

## State

**Mental model:** The same action does different things depending on what state the object is in. Like a traffic light.

**One-liner:** Allow an object to change its behavior when its internal state changes by delegating to state objects.

**Spot it when:** An object has complex conditional logic that depends on its current state. Vending machines, order workflows, media players, TCP connections.

**Template:**
```java
interface State { void handle(Context ctx); }

class Context {
    private State state;
    public Context() { this.state = new InitialState(); }
    public void setState(State s) { this.state = s; }
    public void request() { state.handle(this); }
}

class InitialState implements State {
    public void handle(Context ctx) {
        System.out.println("handling in initial state");
        ctx.setState(new NextState());
    }
}
```

**Key pitfall:** State objects should decide the next state, not the context. If the context decides, you end up with the same if-else mess you were trying to avoid.

---

## Chain of Responsibility

**Mental model:** Pass a request down a line of handlers. Each one either handles it or passes it on. Like escalating a support ticket.

**One-liner:** Give multiple objects a chance to handle a request by chaining them and passing the request along.

**Spot it when:** Multiple handlers could process a request and the right one is not known upfront. Approval workflows, middleware pipelines, logging levels, ATM dispensers.

**Template:**
```java
abstract class Handler {
    protected Handler next;
    public void setNext(Handler h) { this.next = h; }

    public void handle(Request r) {
        if (canHandle(r)) process(r);
        else if (next != null) next.handle(r);
        else System.out.println("unhandled");
    }

    protected abstract boolean canHandle(Request r);
    protected abstract void process(Request r);
}
```

**Key pitfall:** Always handle the case where no handler in the chain can process the request. Do not silently drop it.

---

## Composite

**Mental model:** A folder contains files and other folders. You can call `getSize()` on any of them and it just works.

**One-liner:** Compose objects into tree structures and let clients treat individual objects and compositions uniformly.

**Spot it when:** You have a tree structure where leaves and branches need to be treated the same way. File systems, org charts, UI component trees, menus.

**Template:**
```java
interface Component {
    void operation();
}

class Leaf implements Component {
    public void operation() { System.out.println("leaf"); }
}

class Composite implements Component {
    private List<Component> children = new ArrayList<>();
    public void add(Component c) { children.add(c); }
    public void remove(Component c) { children.remove(c); }
    public void operation() { children.forEach(Component::operation); }
}
```

**Key pitfall:** Initialize the children list in the composite so you never get a NullPointerException. Watch out for circular references if you allow arbitrary additions.

---

## Pattern Cheat Sheet

| Pattern | Category | Core idea | Classic example |
|---------|----------|-----------|-----------------|
| Strategy | Behavioral | Swap algorithms | Payment methods |
| Observer | Behavioral | Notify dependents | Event listeners |
| Command | Behavioral | Encapsulate requests | Undo/redo |
| State | Behavioral | Behavior by state | Vending machine |
| Chain of Responsibility | Behavioral | Pass request down chain | Approval workflow |
| Factory | Creational | Centralize creation | DB connections |
| Singleton | Creational | One instance | Logger, config |
| Builder | Creational | Step-by-step construction | HTTP request |
| Adapter | Structural | Translate interface | Legacy integration |
| Decorator | Structural | Wrap to add behavior | Java I/O streams |
| Facade | Structural | Simplify subsystem | Order processing |
| Composite | Structural | Uniform tree operations | File system |
