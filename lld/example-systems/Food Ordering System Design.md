
# 1. Functional Requirements

- User can browse restaurants
    
- View menu items
    
- Add items to cart
    
- Place order
    
- Apply coupons
    
- Make payment
    
- Track order status (PLACED → PREPARING → OUT_FOR_DELIVERY → DELIVERED → CANCELLED)
    
- Restaurant can accept/reject order
    
- Maintain order history
    

---

# 2. Non-Functional Requirements

- Low latency
    
- High availability
    
- Scalable system
    
- Fault-tolerant
    
- Consistent order state
    
- Thread-safe operations
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|User|Customer|
|Restaurant|Food provider|
|MenuItem|Item in menu|
|Cart|Holds selected items|
|CartItem|Item + quantity|
|Order|Represents order|
|OrderStatus|Order lifecycle|
|Payment|Payment details|
|Coupon|Discount logic|
|RestaurantRepository|Stores restaurants|
|OrderRepository|Stores orders|
|PaymentService|Handles payment|
|OrderService|Orchestrates order flow|
|PricingStrategy|Calculates total price|

---

# 4. Flows

---

## Browse Restaurants

1. Fetch list of restaurants
    
2. Display menus
    

---

## Add to Cart

1. User selects item
    
2. Add/update CartItem
    

---

## Place Order

1. Validate cart
    
2. Calculate total price
    
3. Apply coupon
    
4. Process payment
    
5. Create order (PLACED)
    
6. Send to restaurant
    

---

## Restaurant Accept/Reject

1. Restaurant updates status
    
2. ACCEPT → PREPARING
    
3. REJECT → CANCELLED
    

---

## Delivery Flow

1. PREPARING → OUT_FOR_DELIVERY
    
2. → DELIVERED
    

---

# 5. Code

```java
import java.util.*;
import java.util.concurrent.*;

// ENUM
enum OrderStatus {
    PLACED, PREPARING, OUT_FOR_DELIVERY, DELIVERED, CANCELLED
}

// MODELS
class User {
    String id;
}

class MenuItem {
    String id;
    String name;
    double price;
}

class Restaurant {
    String id;
    List<MenuItem> menu = new ArrayList<>();
}

class CartItem {
    MenuItem item;
    int quantity;

    public CartItem(MenuItem item, int quantity) {
        this.item = item;
        this.quantity = quantity;
    }
}

class Cart {
    String userId;
    List<CartItem> items = new ArrayList<>();
}

class Order {
    String id;
    String userId;
    String restaurantId;
    List<CartItem> items;
    double totalAmount;
    OrderStatus status;

    public Order(String id, String userId, String restaurantId, List<CartItem> items, double total) {
        this.id = id;
        this.userId = userId;
        this.restaurantId = restaurantId;
        this.items = items;
        this.totalAmount = total;
        this.status = OrderStatus.PLACED;
    }
}

class Payment {
    String id;
    String orderId;
    double amount;
    boolean success;
}

// REPOSITORIES
class RestaurantRepository {
    private Map<String, Restaurant> store = new ConcurrentHashMap<>();

    public Restaurant get(String id) {
        return store.get(id);
    }
}

class OrderRepository {
    private Map<String, Order> store = new ConcurrentHashMap<>();

    public void save(Order order) {
        store.put(order.id, order);
    }

    public Order get(String id) {
        return store.get(id);
    }
}

// STRATEGY: Pricing
interface PricingStrategy {
    double calculateTotal(List<CartItem> items);
}

class DefaultPricingStrategy implements PricingStrategy {
    public double calculateTotal(List<CartItem> items) {
        double total = 0;
        for (CartItem ci : items) {
            total += ci.item.price * ci.quantity;
        }
        return total;
    }
}

// STRATEGY: Coupon
interface CouponStrategy {
    double applyDiscount(double amount);
}

class PercentageCoupon implements CouponStrategy {
    private double percent;

    public PercentageCoupon(double percent) {
        this.percent = percent;
    }

    public double applyDiscount(double amount) {
        return amount - (amount * percent / 100);
    }
}

// PAYMENT SERVICE
class PaymentService {
    public Payment pay(String orderId, double amount) {
        Payment p = new Payment();
        p.id = UUID.randomUUID().toString();
        p.orderId = orderId;
        p.amount = amount;
        p.success = true; // mock
        return p;
    }
}

// ORDER SERVICE
class OrderService {
    private OrderRepository orderRepo;
    private PricingStrategy pricingStrategy;
    private PaymentService paymentService;

    public OrderService(OrderRepository repo,
                        PricingStrategy pricing,
                        PaymentService paymentService) {
        this.orderRepo = repo;
        this.pricingStrategy = pricing;
        this.paymentService = paymentService;
    }

    public Order placeOrder(String userId,
                            String restaurantId,
                            List<CartItem> items,
                            CouponStrategy coupon) {

        double total = pricingStrategy.calculateTotal(items);

        if (coupon != null) {
            total = coupon.applyDiscount(total);
        }

        Order order = new Order(
            UUID.randomUUID().toString(),
            userId,
            restaurantId,
            items,
            total
        );

        Payment payment = paymentService.pay(order.id, total);

        if (!payment.success) {
            throw new RuntimeException("Payment failed");
        }

        orderRepo.save(order);
        return order;
    }

    public void updateStatus(String orderId, OrderStatus status) {
        orderRepo.get(orderId).status = status;
    }
}
```

---

# 6. Complexity Analysis

- Add to Cart → O(1)
    
- Calculate Total → O(N)
    
- Place Order → O(N)
    
- Update Status → O(1)
    

---

# 7. Design Patterns Used

- Strategy Pattern → Pricing, Coupon
    
- Repository Pattern → Data access
    
- Dependency Injection → Loose coupling
    

---

# 8. Key Design Decisions

- Pricing and coupon logic made extensible
    
- Payment decoupled from order logic
    
- Order lifecycle clearly defined
    
- Thread-safe repositories used
    
- System extensible for multiple payment methods
    
- Coupon system pluggable
    
- Supports future features like surge pricing, delivery fees, taxes
    

---

# 9. Possible Extensions

- Add Delivery Partner assignment (Strategy Pattern)
    
- Use State Pattern for order lifecycle
    
- Add inventory management
    
- Add real-time tracking (WebSockets)
    
- Introduce event-driven architecture (Kafka)
    
- Add ratings & reviews system
    
- Support scheduled orders