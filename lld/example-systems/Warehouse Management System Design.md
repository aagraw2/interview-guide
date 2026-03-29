
## 1. Functional Requirements

- Add/remove inventory items
    
- Track stock quantity per item
    
- Support multiple warehouses
    
- Allocate inventory for orders
    
- Reserve, confirm, and release stock
    
- Create, confirm, and cancel orders
    
- Prevent overselling of inventory
    

---

## 2. Non-Functional Requirements

- Low latency inventory operations
    
- High concurrency for order placement
    
- Strong consistency for stock updates
    
- Scalable across warehouses and items
    
- Extensible for allocation strategies
    
- Thread-safe operations
    

---

## 3. Core Entities

|Entity|Responsibility|
|---|---|
|Item|Represents a product|
|Warehouse|Stores inventory for items|
|Inventory|Tracks available and reserved stock|
|Order|Represents customer order|
|OrderItem|Item and quantity in order|
|AllocationStrategy|Selects warehouse for allocation|
|InventoryService|Manages stock operations|
|OrderService|Manages order lifecycle|

---

## 4. Flows (Use Cases)

### Place Order (Main Flow)

1. User places order with items
    
2. OrderService creates order (PENDING)
    
3. Calls InventoryService.reserve(items)
    
4. If success → order marked RESERVED
    
5. If failure → order rejected
    

---

### Confirm Order (Main Flow)

1. Payment successful
    
2. OrderService calls InventoryService.confirm(items)
    
3. Reserved stock deducted
    
4. Order marked CONFIRMED
    

---

### Cancel Order (Main Flow)

1. User cancels order
    
2. OrderService calls InventoryService.release(items)
    
3. Reserved stock returned to available
    
4. Order marked CANCELLED
    

---

### Add Inventory (Supporting Flow)

1. Admin selects warehouse and item
    
2. InventoryService updates stock
    
3. Available quantity increased
    

---

### Allocation (Supporting Flow)

1. Iterate warehouses
    
2. Select warehouse using strategy
    
3. Check availability
    
4. Reserve stock
    

---

## 5. Code

```java
import java.util.*;

// MODELS

class Item {
    String id;
    String name;

    public Item(String id, String name) {
        this.id = id;
        this.name = name;
    }
}

class Inventory {
    Item item;
    int available;
    int reserved;

    public Inventory(Item item, int qty) {
        this.item = item;
        this.available = qty;
        this.reserved = 0;
    }

    public synchronized boolean reserve(int qty) {
        if (available >= qty) {
            available -= qty;
            reserved += qty;
            return true;
        }
        return false;
    }

    public synchronized void confirm(int qty) {
        reserved -= qty;
    }

    public synchronized void release(int qty) {
        reserved -= qty;
        available += qty;
    }

    public synchronized void addStock(int qty) {
        available += qty;
    }
}

class Warehouse {
    String id;
    Map<String, Inventory> inventoryMap = new HashMap<>();

    public Warehouse(String id) {
        this.id = id;
    }

    public void addInventory(Item item, int qty) {
        inventoryMap.putIfAbsent(item.id, new Inventory(item, 0));
        inventoryMap.get(item.id).addStock(qty);
    }

    public Inventory getInventory(String itemId) {
        return inventoryMap.get(itemId);
    }
}

class OrderItem {
    Item item;
    int quantity;

    public OrderItem(Item item, int quantity) {
        this.item = item;
        this.quantity = quantity;
    }
}

enum OrderStatus {
    PENDING, RESERVED, CONFIRMED, CANCELLED
}

class Order {
    String id;
    List<OrderItem> items;
    OrderStatus status;

    public Order(String id, List<OrderItem> items) {
        this.id = id;
        this.items = items;
        this.status = OrderStatus.PENDING;
    }
}

// Strategy Pattern for allocation
interface AllocationStrategy {
    Warehouse selectWarehouse(List<Warehouse> warehouses, OrderItem item);
}

class NearestWarehouseStrategy implements AllocationStrategy {
    @Override
    public Warehouse selectWarehouse(List<Warehouse> warehouses, OrderItem item) {
        for (Warehouse wh : warehouses) {
            Inventory inv = wh.getInventory(item.item.id);
            if (inv != null && inv.available >= item.quantity) {
                return wh;
            }
        }
        return null;
    }
}

// Factory Pattern
class AllocationStrategyFactory {
    public static AllocationStrategy getStrategy() {
        return new NearestWarehouseStrategy();
    }
}

// SERVICES

// Singleton Inventory Service
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
                if (instance == null) {
                    instance = new InventoryService();
                }
            }
        }
        return instance;
    }

    public void addWarehouse(Warehouse warehouse) {
        warehouses.add(warehouse);
    }

    public void addInventory(String warehouseId, Item item, int qty) {
        for (Warehouse wh : warehouses) {
            if (wh.id.equals(warehouseId)) {
                wh.addInventory(item, qty);
                return;
            }
        }
    }

    // Reserve stock for order
    public boolean reserve(List<OrderItem> items) {
        Map<Warehouse, List<OrderItem>> allocation = new HashMap<>();

        for (OrderItem item : items) {
            Warehouse wh = strategy.selectWarehouse(warehouses, item);
            if (wh == null) return false;

            allocation.putIfAbsent(wh, new ArrayList<>());
            allocation.get(wh).add(item);
        }

        // Reserve
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
                if (inv != null) {
                    inv.confirm(item.quantity);
                }
            }
        }
    }

    public void release(List<OrderItem> items) {
        for (Warehouse wh : warehouses) {
            for (OrderItem item : items) {
                Inventory inv = wh.getInventory(item.item.id);
                if (inv != null) {
                    inv.release(item.quantity);
                }
            }
        }
    }
}

// Order Service (separated concern)
class OrderService {
    private Map<String, Order> orders = new HashMap<>();
    private InventoryService inventoryService = InventoryService.getInstance();

    // Create order
    public Order createOrder(List<OrderItem> items) {
        Order order = new Order(UUID.randomUUID().toString(), items);

        boolean reserved = inventoryService.reserve(items);
        if (!reserved) {
            throw new RuntimeException("Stock not available");
        }

        order.status = OrderStatus.RESERVED;
        orders.put(order.id, order);

        return order;
    }

    public void confirmOrder(String orderId) {
        Order order = orders.get(orderId);

        inventoryService.confirm(order.items);
        order.status = OrderStatus.CONFIRMED;
    }

    public void cancelOrder(String orderId) {
        Order order = orders.get(orderId);

        inventoryService.release(order.items);
        order.status = OrderStatus.CANCELLED;
    }
}
```

---

## 6. Complexity Analysis

### Place Order

- Warehouse selection → O(W) per item
    
- Reserve stock → O(1) per item
    

**Overall: O(N × W)**

---

### Confirm Order

- Iterate warehouses and items → O(N × W)
    

**Overall: O(N × W)**

---

### Cancel Order

- Iterate warehouses and items → O(N × W)
    

**Overall: O(N × W)**

---

### Add Inventory

- Lookup warehouse → O(W)
    
- Update stock → O(1)
    

**Overall: O(W)**

---

## 7. Key Design Decisions

- Separate OrderService and InventoryService for clear responsibility boundaries
    
- Strategy Pattern enables flexible warehouse allocation logic
    
- Factory Pattern decouples allocation strategy creation
    
- Singleton InventoryService ensures centralized stock management
    
- Reserved vs available stock prevents overselling
    
- Synchronized inventory operations ensure thread safety
    
- Design can be extended with distributed locking and database persistence