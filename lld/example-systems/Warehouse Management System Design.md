
## 1. Functional Requirements

- Add/remove inventory items
    
- Track stock quantity per item
    
- Support multiple warehouses
    
- Allocate inventory for orders
    
- Reserve, confirm, and release stock
    
- Create, confirm, and cancel orders
    
- Prevent overselling of inventory
    
- Send alerts when:
    
    - Stock falls below configurable minimum threshold
        
    - Order is created / confirmed / cancelled / failed
        

---

## 2. Non-Functional Requirements

- Low latency inventory operations
    
- High concurrency for order placement
    
- Strong consistency for stock updates
    
- Scalable across warehouses and items
    
- Extensible for allocation strategies
    
- Thread-safe operations
    
- Alert threshold configurable per item or globally
    
- Extensible alerting for multiple event types
    

---

## 3. Core Entities

|Entity|Responsibility|
|---|---|
|Item|Represents a product|
|Warehouse|Stores inventory for items|
|Inventory|Tracks available and reserved stock, emits alerts if below threshold|
|Order|Represents customer order, emits lifecycle events|
|OrderItem|Item and quantity in order|
|AllocationStrategy|Selects warehouse for allocation|
|InventoryService|Manages stock operations|
|OrderService|Manages order lifecycle|
|Observer|Interface for receiving events|
|AlertService|Handles alert delivery (email, SMS, dashboard)|

---

## 4. Flows (Use Cases)

### Place Order

1. User places order with items
    
2. OrderService creates order (PENDING)
    
3. Order registers alert handlers
    
4. Calls InventoryService.reserve(items)
    
5. Reserved stock deducted
    
6. If stock < threshold → alert sent
    
7. Order moves to RESERVED → alert sent
    

---

### Confirm Order

1. Payment successful
    
2. OrderService confirms order
    
3. Reserved stock deducted (confirmed)
    
4. Order moves to CONFIRMED → alert sent
    

---

### Cancel Order

1. User cancels order
    
2. Reserved stock released back to available
    
3. If stock < threshold → alert sent
    
4. Order moves to CANCELLED → alert sent
    

---

### Add Inventory

1. Admin adds stock
    
2. Available quantity updated
    
3. No alert unless stock still below threshold
    

---

### Allocation

1. Warehouses selected using strategy
    
2. Reserve stock
    
3. If stock < threshold after reservation → alert sent
    

---

## 5. Code (Java)

```java
import java.util.*;

// MODELS
class Item {
    String id;
    String name;
    public Item(String id, String name) { this.id = id; this.name = name; }
}

class OrderItem {
    Item item;
    int quantity;
    public OrderItem(Item item, int quantity) { this.item = item; this.quantity = quantity; }
}

enum OrderStatus { PENDING, RESERVED, CONFIRMED, CANCELLED }

// EVENTS
class InventoryEvent {
    String type;
    Item item;
    int available;
    public InventoryEvent(String type, Item item, int available) {
        this.type = type; this.item = item; this.available = available;
    }
}

class OrderEvent {
    String type;
    String orderId;
    public OrderEvent(String type, String orderId) {
        this.type = type; this.orderId = orderId;
    }
}

// OBSERVER
interface Observer<T> { void update(T event); }

interface Observable<T> {  
    void addObserver(Observer<T> observer);  
    void removeObserver(Observer<T> observer);
    void notify(String type);  
}

// ALERT SERVICE
class AlertService implements Observer<InventoryEvent>, Observer<OrderEvent> {
    @Override
    public void update(InventoryEvent event) {
        System.out.println("[INVENTORY ALERT] " + event.type +
            " | Item: " + event.item.name + ", Available: " + event.available);
    }

    @Override
    public void update(OrderEvent event) {
        System.out.println("[ORDER ALERT] " + event.type +
            " | OrderId: " + event.orderId);
    }
}

// ORDER
class Order implements Observable<OrderEvent> {
    String id;
    List<OrderItem> items;
    OrderStatus status;

    List<Observer<OrderEvent>> observers = new ArrayList<>();

    public Order(String id, List<OrderItem> items) {
        this.id = id; this.items = items; this.status = OrderStatus.PENDING;
    }

    public void addObserver(Observer<OrderEvent> obs) {
        observers.add(obs);
    }

    private void notify(String type) {
        OrderEvent event = new OrderEvent(type, id);
        for (Observer<OrderEvent> obs : observers) obs.update(event);
    }

    public void markReserved() {
        status = OrderStatus.RESERVED;
        notify("ORDER_CREATED");
    }

    public void markConfirmed() {
        status = OrderStatus.CONFIRMED;
        notify("ORDER_CONFIRMED");
    }

    public void markCancelled() {
        status = OrderStatus.CANCELLED;
        notify("ORDER_CANCELLED");
    }

    public void markFailed() {
        notify("ORDER_FAILED");
    }
}

// INVENTORY
class Inventory implements Observable<InventoryEvent> {
    Item item;
    int available;
    int reserved;
    int minThreshold;

    List<Observer<InventoryEvent>> observers = new ArrayList<>();

    public Inventory(Item item, int qty, int threshold) {
        this.item = item; this.available = qty; this.reserved = 0; this.minThreshold = threshold;
    }

    public void addObserver(Observer<InventoryEvent> obs) { observers.add(obs); }

    private void notify(String type) {
        InventoryEvent event = new InventoryEvent(type, item, available);
        for (Observer<InventoryEvent> obs : observers) obs.update(event);
    }

    public synchronized boolean reserve(int qty) {
        if (available >= qty) {
            available -= qty; reserved += qty;
            if (available < minThreshold) notify("STOCK_LOW");
            return true;
        } else return false;
    }

    public synchronized void confirm(int qty) { reserved -= qty; }

    public synchronized void release(int qty) {
        reserved -= qty; available += qty;
        if (available < minThreshold) notify("STOCK_LOW");
    }

    public synchronized void addStock(int qty) {
        available += qty;
    }
}

// WAREHOUSE
class Warehouse {
    String id;
    Map<String, Inventory> inventoryMap = new HashMap<>();

    public Warehouse(String id) { this.id = id; }

    public void addInventory(Item item, int qty, int threshold, Observer<InventoryEvent> obs) {
        inventoryMap.putIfAbsent(item.id, new Inventory(item, 0, threshold));
        Inventory inv = inventoryMap.get(item.id);
        inv.addStock(qty);
        inv.addObserver(obs);
    }

    public Inventory getInventory(String itemId) { return inventoryMap.get(itemId); }
}

// STRATEGY
interface AllocationStrategy {
    Warehouse selectWarehouse(List<Warehouse> warehouses, OrderItem item);
}

class NearestWarehouseStrategy implements AllocationStrategy {
    public Warehouse selectWarehouse(List<Warehouse> warehouses, OrderItem item) {
        for (Warehouse wh : warehouses) {
            Inventory inv = wh.getInventory(item.item.id);
            if (inv != null && inv.available >= item.quantity) return wh;
        }
        return null;
    }
}

class AllocationStrategyFactory {
    public static AllocationStrategy getStrategy() {
        return new NearestWarehouseStrategy();
    }
}

// SERVICES
class InventoryService {
    private static InventoryService instance;
    private List<Warehouse> warehouses = new ArrayList<>();
    private AllocationStrategy strategy;

    private InventoryService() {
        this.strategy = AllocationStrategyFactory.getStrategy();
    }

    public static InventoryService getInstance() {
        if (instance == null) {
            synchronized (InventoryService.class) {
                if (instance == null) instance = new InventoryService();
            }
        }
        return instance;
    }

    public void addWarehouse(Warehouse wh) { warehouses.add(wh); }

    public void addInventory(String warehouseId, Item item, int qty, int threshold, Observer<InventoryEvent> obs) {
        for (Warehouse wh : warehouses) {
            if (wh.id.equals(warehouseId)) {
                wh.addInventory(item, qty, threshold, obs);
                return;
            }
        }
    }

    public boolean reserve(List<OrderItem> items) {
        Map<Warehouse, List<OrderItem>> allocation = new HashMap<>();

        for (OrderItem item : items) {
            Warehouse wh = strategy.selectWarehouse(warehouses, item);
            if (wh == null) return false;
            allocation.putIfAbsent(wh, new ArrayList<>());
            allocation.get(wh).add(item);
        }

        for (Map.Entry<Warehouse, List<OrderItem>> entry : allocation.entrySet()) {
            for (OrderItem item : entry.getValue()) {
                Inventory inv = entry.getKey().getInventory(item.item.id);
                if (!inv.reserve(item.quantity)) return false;
            }
        }
        return true;
    }

    public void confirm(List<OrderItem> items) {
        for (Warehouse wh : warehouses) {
            for (OrderItem item : items) {
                Inventory inv = wh.getInventory(item.item.id);
                if (inv != null) inv.confirm(item.quantity);
            }
        }
    }

    public void release(List<OrderItem> items) {
        for (Warehouse wh : warehouses) {
            for (OrderItem item : items) {
                Inventory inv = wh.getInventory(item.item.id);
                if (inv != null) inv.release(item.quantity);
            }
        }
    }
}

class OrderService {
    private Map<String, Order> orders = new HashMap<>();
    private InventoryService inventoryService = InventoryService.getInstance();
    private Observer<OrderEvent> orderObserver;

    public OrderService(Observer<OrderEvent> observer) {
        this.orderObserver = observer;
    }

    public Order createOrder(List<OrderItem> items) {
        Order order = new Order(UUID.randomUUID().toString(), items);
        order.addObserver(orderObserver);

        boolean reserved = inventoryService.reserve(items);
        if (!reserved) {
            order.markFailed();
            throw new RuntimeException("Stock not available");
        }

        order.markReserved();
        orders.put(order.id, order);
        return order;
    }

    public void confirmOrder(String orderId) {
        Order order = orders.get(orderId);
        inventoryService.confirm(order.items);
        order.markConfirmed();
    }

    public void cancelOrder(String orderId) {
        Order order = orders.get(orderId);
        inventoryService.release(order.items);
        order.markCancelled();
    }
}
```

---

## 6. Complexity Analysis

- **Place Order** → O(N × W)
    
- **Confirm Order** → O(N × W)
    
- **Cancel Order** → O(N × W)
    
- **Add Inventory** → O(W)
    
- **Alert notifications** → O(O × N × W)
    

---

## 7. Key Design Decisions

- Alerts triggered only when:
    
    - Stock falls below configurable threshold
        
    - Order lifecycle changes
        
- Event handling is decoupled from core business logic
    
- Inventory and Order manage their own state transitions and trigger events
    
- Services remain orchestration-only layers
    
- Strategy Pattern used for warehouse allocation
    
- Thread-safe inventory updates prevent overselling

# 8. Key Design Decision

### 👉 Where to place Observer?

| Component        | Observable? | Why                          |
| ---------------- | ----------- | ---------------------------- |
| Inventory        | ✅ Yes       | Owns stock + threshold logic |
| Order            | ✅ Yes       | Owns state transitions       |
| InventoryService | ❌ No        | Only orchestrates            |
| OrderService     | ❌ No        | Only orchestrates            |

> **Rule:** Observer lives where state changes happen