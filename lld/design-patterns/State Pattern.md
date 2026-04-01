## 1. What is State Pattern?

State pattern allows an object to alter its behavior when its internal state changes. The object will appear to change its class.

```
Without State:
  if (state == RED) { /* red logic */ }
  else if (state == GREEN) { /* green logic */ }
  else if (state == YELLOW) { /* yellow logic */ }
  // Complex conditionals everywhere

With State:
  state.handle(context);
  // Each state encapsulates its behavior
```

**Key principle:** Encapsulate state-specific behavior in separate state classes.

---

## 2. Implementation Guide

### Step 1: Define State Interface

```java
public interface TrafficLightState {
    void handle(TrafficLight context);
}
```

### Step 2: Create Context Class

```java
public class TrafficLight {
    private TrafficLightState state;
    
    public TrafficLight() {
        this.state = new RedState();  // Initial state
    }
    
    public void setState(TrafficLightState state) {
        this.state = state;
    }
    
    public void change() {
        state.handle(this);
    }
}
```

### Step 3: Implement Concrete States

```java
public class RedState implements TrafficLightState {
    @Override
    public void handle(TrafficLight context) {
        System.out.println("Red Light - STOP");
        context.setState(new GreenState());
    }
}

public class GreenState implements TrafficLightState {
    @Override
    public void handle(TrafficLight context) {
        System.out.println("Green Light - GO");
        context.setState(new YellowState());
    }
}

public class YellowState implements TrafficLightState {
    @Override
    public void handle(TrafficLight context) {
        System.out.println("Yellow Light - SLOW DOWN");
        context.setState(new RedState());
    }
}
```

### Step 4: Usage

```java
TrafficLight light = new TrafficLight();
light.change();  // Red -> Green
light.change();  // Green -> Yellow
light.change();  // Yellow -> Red
light.change();  // Red -> Green
```

---

## 3. Common Interview Questions

### Q1: Vending Machine

```java
public interface VendingMachineState {
    void insertCoin(VendingMachine machine);
    void selectProduct(VendingMachine machine);
    void dispense(VendingMachine machine);
}

public class VendingMachine {
    private VendingMachineState state;
    private int coinCount = 0;
    
    public VendingMachine() {
        this.state = new NoCoinState();
    }
    
    public void setState(VendingMachineState state) {
        this.state = state;
    }
    
    public void insertCoin() {
        state.insertCoin(this);
    }
    
    public void selectProduct() {
        state.selectProduct(this);
    }
    
    public void dispense() {
        state.dispense(this);
    }
    
    public void addCoin() {
        coinCount++;
    }
    
    public void resetCoins() {
        coinCount = 0;
    }
    
    public int getCoinCount() {
        return coinCount;
    }
}

public class NoCoinState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("Coin inserted");
        machine.addCoin();
        machine.setState(new HasCoinState());
    }
    
    @Override
    public void selectProduct(VendingMachine machine) {
        System.out.println("Please insert coin first");
    }
    
    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Please insert coin first");
    }
}

public class HasCoinState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("Coin already inserted");
        machine.addCoin();
    }
    
    @Override
    public void selectProduct(VendingMachine machine) {
        System.out.println("Product selected");
        machine.setState(new DispensingState());
    }
    
    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Please select product first");
    }
}

public class DispensingState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("Please wait, dispensing product");
    }
    
    @Override
    public void selectProduct(VendingMachine machine) {
        System.out.println("Already dispensing");
    }
    
    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Dispensing product...");
        machine.resetCoins();
        machine.setState(new NoCoinState());
    }
}

// Usage
VendingMachine machine = new VendingMachine();
machine.selectProduct();  // Error: no coin
machine.insertCoin();     // Coin inserted
machine.selectProduct();  // Product selected
machine.dispense();       // Dispensing
```

### Q2: Document Workflow

```java
public interface DocumentState {
    void edit(Document doc);
    void review(Document doc);
    void publish(Document doc);
}

public class Document {
    private DocumentState state;
    private String content;
    
    public Document() {
        this.state = new DraftState();
    }
    
    public void setState(DocumentState state) {
        this.state = state;
    }
    
    public void setContent(String content) {
        this.content = content;
    }
    
    public String getContent() {
        return content;
    }
    
    public void edit() {
        state.edit(this);
    }
    
    public void review() {
        state.review(this);
    }
    
    public void publish() {
        state.publish(this);
    }
}

public class DraftState implements DocumentState {
    @Override
    public void edit(Document doc) {
        System.out.println("Editing draft");
        doc.setContent("Updated content");
    }
    
    @Override
    public void review(Document doc) {
        System.out.println("Sending for review");
        doc.setState(new ModerationState());
    }
    
    @Override
    public void publish(Document doc) {
        System.out.println("Cannot publish draft directly");
    }
}

public class ModerationState implements DocumentState {
    @Override
    public void edit(Document doc) {
        System.out.println("Sending back to draft");
        doc.setState(new DraftState());
    }
    
    @Override
    public void review(Document doc) {
        System.out.println("Already in review");
    }
    
    @Override
    public void publish(Document doc) {
        System.out.println("Publishing document");
        doc.setState(new PublishedState());
    }
}

public class PublishedState implements DocumentState {
    @Override
    public void edit(Document doc) {
        System.out.println("Cannot edit published document");
    }
    
    @Override
    public void review(Document doc) {
        System.out.println("Already published");
    }
    
    @Override
    public void publish(Document doc) {
        System.out.println("Already published");
    }
}
```

---

## 4. Interview Tricks & Pitfalls

### Pitfall 1: State Transition Logic in Context

```java
// WRONG: Context decides next state
public void change() {
    if (state instanceof RedState) {
        state = new GreenState();
    }
}

// CORRECT: State decides next state
public void change() {
    state.handle(this);  // State changes itself
}
```

### Pitfall 2: Shared State Objects

```java
// WRONG: Reusing state instances with mutable state
private static final RedState RED = new RedState();

// CORRECT: Create new instances or make stateless
public void handle(Context context) {
    context.setState(new GreenState());
}
```

### Pitfall 3: Too Many States

```java
// PROBLEM: State explosion
// SOLUTION: Combine with other patterns or use state parameters
```

---

## 5. State vs Strategy

| Aspect | State | Strategy |
|--------|-------|----------|
| Purpose | Change behavior based on state | Choose algorithm |
| Who decides | Object changes state internally | Client sets strategy |
| Transitions | States know about each other | Strategies independent |
| Example | Traffic light | Sorting algorithms |

---

## 6. When to Use

✅ **Use when:**
- Object behavior depends on state
- Many conditional statements based on state
- State transitions are complex
- States have different behaviors

❌ **Don't use when:**
- Simple state (use enum)
- Few states with simple logic
- States don't transition

---

## 7. Real-World Examples

- **TCP Connection**: Closed, Listen, Established states
- **Order Processing**: Pending, Confirmed, Shipped, Delivered
- **Media Player**: Playing, Paused, Stopped
- **Game Character**: Idle, Walking, Running, Jumping

---

## 8. Quick Checklist

1. ☐ Define state interface
2. ☐ Create context class holding current state
3. ☐ Implement concrete state classes
4. ☐ Each state handles its own transitions
5. ☐ Context delegates to current state
6. ☐ Demonstrate state transitions
