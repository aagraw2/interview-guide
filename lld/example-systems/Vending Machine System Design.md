
# 1. Functional Requirements

- Display available items
    
- Select item
    
- Insert money (coins/notes)
    
- Validate payment
    
- Dispense item
    
- Return change
    
- Handle insufficient funds
    
- Handle out-of-stock items
    

---

# 2. Non-Functional Requirements

- Low latency operations
    
- High reliability
    
- Thread-safe
    
- Extensible (new item types, payment modes)
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|VendingMachine|Main controller|
|State|Represents machine state|
|Item|Product entity|
|Inventory|Manages stock|
|Payment|Handles money insertion|
|DispenseService|Dispenses item|
|ChangeService|Returns change|

---

# 4. Design Approach

- Use **State Pattern** for machine states
    
- Use **Strategy Pattern** for payment handling
    
- Use **Factory Pattern** for item creation
    

---

# 5. States

- IdleState
    
- HasMoneyState
    
- DispensingState
    
- OutOfStockState
    

---

# 6. Flows

---

## Purchase Flow

1. User selects item
    
2. Check availability
    
3. Move to HasMoneyState
    
4. Insert money
    
5. Validate payment
    
6. Dispense item
    
7. Return change
    
8. Move to IdleState
    

---

## Insufficient Funds

1. User inserts less money
    
2. Prompt for more money
    
3. Stay in HasMoneyState
    

---

## Out of Stock

1. User selects item
    
2. If stock = 0 → move to OutOfStockState
    

---

# 7. Code

```java
import java.util.*;

// ITEM
class Item {
    String name;
    int price;

    public Item(String name, int price) {
        this.name = name;
        this.price = price;
    }
}

// INVENTORY
class Inventory {
    private Map<Item, Integer> stock = new HashMap<>();

    public void addItem(Item item, int qty) {
        stock.put(item, stock.getOrDefault(item, 0) + qty);
    }

    public boolean isAvailable(Item item) {
        return stock.getOrDefault(item, 0) > 0;
    }

    public void reduce(Item item) {
        stock.put(item, stock.get(item) - 1);
    }
}

// STATE INTERFACE
interface State {
    void selectItem(Item item);
    void insertMoney(int amount);
    void dispense();
}

// VENDING MACHINE
class VendingMachine {
    State idleState;
    State hasMoneyState;
    State dispensingState;
    State outOfStockState;

    State currentState;

    Inventory inventory = new Inventory();
    Item selectedItem;
    int balance;

    public VendingMachine() {
        idleState = new IdleState(this);
        hasMoneyState = new HasMoneyState(this);
        dispensingState = new DispensingState(this);
        outOfStockState = new OutOfStockState(this);

        currentState = idleState;
    }

    public void setState(State state) {
        currentState = state;
    }

    public void selectItem(Item item) {
        currentState.selectItem(item);
    }

    public void insertMoney(int amount) {
        currentState.insertMoney(amount);
    }

    public void dispense() {
        currentState.dispense();
    }
}

// IDLE STATE
class IdleState implements State {
    private VendingMachine vm;

    public IdleState(VendingMachine vm) {
        this.vm = vm;
    }

    public void selectItem(Item item) {
        if (!vm.inventory.isAvailable(item)) {
            vm.setState(vm.outOfStockState);
            return;
        }
        vm.selectedItem = item;
        vm.setState(vm.hasMoneyState);
    }

    public void insertMoney(int amount) {
        System.out.println("Select item first");
    }

    public void dispense() {
        System.out.println("No item selected");
    }
}

// HAS MONEY STATE
class HasMoneyState implements State {
    private VendingMachine vm;

    public HasMoneyState(VendingMachine vm) {
        this.vm = vm;
    }

    public void selectItem(Item item) {
        System.out.println("Item already selected");
    }

    public void insertMoney(int amount) {
        vm.balance += amount;

        if (vm.balance >= vm.selectedItem.price) {
            vm.setState(vm.dispensingState);
        }
    }

    public void dispense() {
        System.out.println("Insert money first");
    }
}

// DISPENSING STATE
class DispensingState implements State {
    private VendingMachine vm;

    public DispensingState(VendingMachine vm) {
        this.vm = vm;
    }

    public void selectItem(Item item) {
        System.out.println("Processing...");
    }

    public void insertMoney(int amount) {
        System.out.println("Processing...");
    }

    public void dispense() {
        vm.inventory.reduce(vm.selectedItem);

        int change = vm.balance - vm.selectedItem.price;

        System.out.println("Dispensed: " + vm.selectedItem.name);
        System.out.println("Change returned: " + change);

        vm.balance = 0;
        vm.selectedItem = null;

        vm.setState(vm.idleState);
    }
}

// OUT OF STOCK STATE
class OutOfStockState implements State {
    private VendingMachine vm;

    public OutOfStockState(VendingMachine vm) {
        this.vm = vm;
    }

    public void selectItem(Item item) {
        System.out.println("Item out of stock");
        vm.setState(vm.idleState);
    }

    public void insertMoney(int amount) {
        System.out.println("Cannot accept money");
    }

    public void dispense() {
        System.out.println("No item to dispense");
    }
}
```

---

# 8. Complexity Analysis

- Select item → O(1)
    
- Insert money → O(1)
    
- Dispense → O(1)
    

---

# 9. Design Patterns Used

- State Pattern → Machine behavior
    
- Strategy Pattern → Payment (extendable)
    
- Factory Pattern → Item creation (extendable)
    

---

# 10. Key Design Decisions

- State pattern simplifies complex state transitions
    
- Inventory separated for modularity
    
- Balance tracked centrally
    
- Clear separation of concerns
    
- Easy extensibility for new states/payments
    

---

# 11. Edge Cases Handled

- Insufficient funds
    
- Out-of-stock items
    
- Invalid operation in wrong state
    
- Multiple money insertions
    

---

# 12. Possible Extensions

- Support UPI/card payments
    
- Add refund feature
    
- Add display panel
    
- Add item categories
    
- Add concurrency handling
    
- Add audit logs
    
- Add dynamic pricing
    
- Add maintenance mode