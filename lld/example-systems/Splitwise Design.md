
## 1. Functional Requirements

- User can create groups
    
- Add users to group
    
- Add expense (equal, exact, percentage split)
    
- Track balances between users
    
- Settle up balances
    
- View individual and group balances
    

---

## 2. Non-Functional Requirements

- Low latency for expense addition and balance updates
    
- High consistency in balance calculations
    
- Scalable for large groups
    
- Extensible for new split types
    
- Thread-safe operations
    

---

## 3. Core Entities

|Entity|Responsibility|
|---|---|
|User|Represents a participant|
|Group|Collection of users|
|Expense|Represents an expense|
|ExpenseSplit|Represents split details per user|
|BalanceSheet|Maintains net balances between users|
|SplitStrategy|Defines how expense is split|
|ExpenseService|Handles expense creation and settlement|

---

## 4. Flows (Use Cases)

### Add Expense (Main Flow)

1. User selects group and enters expense details
    
2. Select split type (equal/exact/percentage)
    
3. SplitStrategy calculates per-user share
    
4. Create Expense and ExpenseSplits
    
5. Update BalanceSheet:
    
    - Payer gets credit
        
    - Others get debit
        

---

### View Balances (Main Flow)

1. Fetch BalanceSheet for user/group
    
2. Display who owes whom and how much
    

---

### Settle Up (Main Flow)

1. User chooses to settle with another user
    
2. Transfer amount (logical)
    
3. Update BalanceSheet to reduce debt
    

---

### Split Calculation (Supporting Flow)

1. Based on split type:
    
    - Equal → divide equally
        
    - Exact → use provided amounts
        
    - Percentage → compute share
        

---

## 5. Code

```java
import java.util.*;

// MODELS

class User {
    String id;
    String name;

    public User(String id, String name) {
        this.id = id;
        this.name = name;
    }
}

class Group {
    String id;
    List<User> users;

    public Group(String id, List<User> users) {
        this.id = id;
        this.users = users;
    }
}

class Expense {
    String id;
    User paidBy;
    double amount;
    List<ExpenseSplit> splits;

    public Expense(String id, User paidBy, double amount, List<ExpenseSplit> splits) {
        this.id = id;
        this.paidBy = paidBy;
        this.amount = amount;
        this.splits = splits;
    }
}

class ExpenseSplit {
    User user;
    double amount;

    public ExpenseSplit(User user, double amount) {
        this.user = user;
        this.amount = amount;
    }
}

// Maintains balances: who owes whom
class BalanceSheet {
    // userA -> (userB -> amount) => userA owes userB
    Map<User, Map<User, Double>> balances = new HashMap<>();

    public void addTransaction(User from, User to, double amount) {
        balances.putIfAbsent(from, new HashMap<>());
        Map<User, Double> inner = balances.get(from);

        inner.put(to, inner.getOrDefault(to, 0.0) + amount);
    }

    public void settle(User from, User to, double amount) {
        if (!balances.containsKey(from)) return;

        Map<User, Double> inner = balances.get(from);
        double current = inner.getOrDefault(to, 0.0);

        inner.put(to, Math.max(0, current - amount));
    }
}

// Strategy Pattern for split logic
interface SplitStrategy {
    List<ExpenseSplit> split(User paidBy, double amount, List<User> users, List<Double> values);
}

// Equal Split
class EqualSplitStrategy implements SplitStrategy {
    @Override
    public List<ExpenseSplit> split(User paidBy, double amount, List<User> users, List<Double> values) {
        List<ExpenseSplit> splits = new ArrayList<>();
        double share = amount / users.size();

        for (User user : users) {
            if (!user.equals(paidBy)) {
                splits.add(new ExpenseSplit(user, share));
            }
        }
        return splits;
    }
}

// Exact Split
class ExactSplitStrategy implements SplitStrategy {
    @Override
    public List<ExpenseSplit> split(User paidBy, double amount, List<User> users, List<Double> values) {
        List<ExpenseSplit> splits = new ArrayList<>();

        for (int i = 0; i < users.size(); i++) {
            User user = users.get(i);
            double val = values.get(i);

            if (!user.equals(paidBy)) {
                splits.add(new ExpenseSplit(user, val));
            }
        }
        return splits;
    }
}

// Percentage Split
class PercentageSplitStrategy implements SplitStrategy {
    @Override
    public List<ExpenseSplit> split(User paidBy, double amount, List<User> users, List<Double> values) {
        List<ExpenseSplit> splits = new ArrayList<>();

        for (int i = 0; i < users.size(); i++) {
            User user = users.get(i);
            double percent = values.get(i);

            if (!user.equals(paidBy)) {
                splits.add(new ExpenseSplit(user, amount * percent / 100));
            }
        }
        return splits;
    }
}

// Factory Pattern for strategy selection
class SplitStrategyFactory {
    public static SplitStrategy getStrategy(String type) {
        switch (type) {
            case "EQUAL":
                return new EqualSplitStrategy();
            case "EXACT":
                return new ExactSplitStrategy();
            case "PERCENTAGE":
                return new PercentageSplitStrategy();
            default:
                throw new IllegalArgumentException("Invalid split type");
        }
    }
}

// SERVICES

// Singleton Expense Service
class ExpenseService {
    private static ExpenseService instance;

    private BalanceSheet balanceSheet = new BalanceSheet();

    private ExpenseService() {}

    public static ExpenseService getInstance() {
        if (instance == null) {
            synchronized (ExpenseService.class) {
                if (instance == null) {
                    instance = new ExpenseService();
                }
            }
        }
        return instance;
    }

    public void addExpense(String type, User paidBy, double amount, List<User> users, List<Double> values) {
        SplitStrategy strategy = SplitStrategyFactory.getStrategy(type);

        List<ExpenseSplit> splits = strategy.split(paidBy, amount, users, values);

        for (ExpenseSplit split : splits) {
            balanceSheet.addTransaction(split.user, paidBy, split.amount);
        }
    }

    public void settle(User from, User to, double amount) {
        balanceSheet.settle(from, to, amount);
    }
}
```

---

## 6. Complexity Analysis

### Add Expense

- Split calculation → O(N)
    
- Balance update → O(N)
    

**Overall: O(N)**

---

### View Balances

- Iterate balances → O(N²) worst case
    

**Overall: O(N²)**

---

### Settle Up

- Update balance → O(1)
    

**Overall: O(1)**

---

### Split Calculation

- Iterate users → O(N)
    

**Overall: O(N)**

---

## 7. Key Design Decisions

- Strategy Pattern cleanly separates split logic
    
- Factory Pattern avoids conditional logic in service layer
    
- Singleton ensures single balance manager
    
- BalanceSheet uses nested map for quick lookup
    
- Separation of expense creation and balance tracking improves clarity
    
- Easily extensible for new split types or settlement optimizations