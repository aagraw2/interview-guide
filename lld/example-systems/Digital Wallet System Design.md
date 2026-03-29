
# 1. Functional Requirements

- User can create a wallet
    
- Add money to wallet
    
- Transfer money between wallets
    
- Check wallet balance
    
- Maintain transaction history
    
- Support debit and credit operations
    
- Ensure idempotent transactions
    

---

# 2. Non-Functional Requirements

- Strong consistency for transactions
    
- High availability
    
- Low latency for operations
    
- Thread-safe operations
    
- Fault-tolerant system
    
- Scalable for large number of users
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|User|Represents wallet owner|
|Wallet|Holds balance|
|Transaction|Represents money movement|
|TransactionType|CREDIT / DEBIT / TRANSFER|
|WalletRepository|Stores wallet data|
|TransactionRepository|Stores transaction history|
|WalletService|Handles wallet operations|
|TransactionService|Manages transaction lifecycle|
|LockManager|Handles concurrency control|

---

# 4. Flows (Use Cases)

---

## Create Wallet

1. User requests wallet creation
    
2. WalletService creates wallet with zero balance
    
3. Store in repository
    

---

## Add Money

1. User requests to add money
    
2. WalletService updates balance
    
3. TransactionService records CREDIT transaction
    

---

## Transfer Money

1. User initiates transfer (from → to)
    
2. Acquire locks on both wallets
    
3. Check sufficient balance
    
4. Debit sender wallet
    
5. Credit receiver wallet
    
6. Record transaction
    
7. Release locks
    

---

## Check Balance

1. Fetch wallet from repository
    
2. Return current balance
    

---

## Transaction History

1. Fetch transactions for wallet
    
2. Return list
    

---

# 5. Code

```java
import java.util.*;
import java.util.concurrent.*;

// ENUM
enum TransactionType {
    CREDIT, DEBIT, TRANSFER
}

// MODELS
class User {
    String id;

    public User(String id) {
        this.id = id;
    }
}

class Wallet {
    String id;
    String userId;
    double balance;

    public Wallet(String id, String userId) {
        this.id = id;
        this.userId = userId;
        this.balance = 0.0;
    }
}

class Transaction {
    String id;
    String fromWallet;
    String toWallet;
    double amount;
    TransactionType type;

    public Transaction(String id, String from, String to, double amount, TransactionType type) {
        this.id = id;
        this.fromWallet = from;
        this.toWallet = to;
        this.amount = amount;
        this.type = type;
    }
}

// REPOSITORIES
class WalletRepository {
    private Map<String, Wallet> store = new ConcurrentHashMap<>();

    public void save(Wallet wallet) {
        store.put(wallet.id, wallet);
    }

    public Wallet get(String id) {
        return store.get(id);
    }
}

class TransactionRepository {
    private List<Transaction> transactions = new CopyOnWriteArrayList<>();

    public void save(Transaction txn) {
        transactions.add(txn);
    }

    public List<Transaction> getByWallet(String walletId) {
        List<Transaction> result = new ArrayList<>();
        for (Transaction t : transactions) {
            if (walletId.equals(t.fromWallet) || walletId.equals(t.toWallet)) {
                result.add(t);
            }
        }
        return result;
    }
}

// LOCK MANAGER (Singleton)
class LockManager {
    private static LockManager instance;
    private Map<String, Object> locks = new ConcurrentHashMap<>();

    private LockManager() {}

    public static LockManager getInstance() {
        if (instance == null) {
            synchronized (LockManager.class) {
                if (instance == null) {
                    instance = new LockManager();
                }
            }
        }
        return instance;
    }

    public Object getLock(String key) {
        locks.putIfAbsent(key, new Object());
        return locks.get(key);
    }
}

// SERVICES
class TransactionService {
    private TransactionRepository txnRepo;

    public TransactionService(TransactionRepository txnRepo) {
        this.txnRepo = txnRepo;
    }

    public void record(Transaction txn) {
        txnRepo.save(txn);
    }
}

class WalletService {
    private WalletRepository walletRepo;
    private TransactionService txnService;
    private LockManager lockManager = LockManager.getInstance();

    public WalletService(WalletRepository walletRepo, TransactionService txnService) {
        this.walletRepo = walletRepo;
        this.txnService = txnService;
    }

    public Wallet createWallet(String userId) {
        Wallet wallet = new Wallet(UUID.randomUUID().toString(), userId);
        walletRepo.save(wallet);
        return wallet;
    }

    public void addMoney(String walletId, double amount) {
        Wallet wallet = walletRepo.get(walletId);

        synchronized (lockManager.getLock(walletId)) {
            wallet.balance += amount;

            Transaction txn = new Transaction(
                UUID.randomUUID().toString(),
                null,
                walletId,
                amount,
                TransactionType.CREDIT
            );
            txnService.record(txn);
        }
    }

    public void transfer(String fromId, String toId, double amount) {
        Object lock1 = lockManager.getLock(fromId);
        Object lock2 = lockManager.getLock(toId);

        // Avoid deadlock using ordering
        Object first = fromId.compareTo(toId) < 0 ? lock1 : lock2;
        Object second = fromId.compareTo(toId) < 0 ? lock2 : lock1;

        synchronized (first) {
            synchronized (second) {
                Wallet from = walletRepo.get(fromId);
                Wallet to = walletRepo.get(toId);

                if (from.balance < amount) {
                    throw new RuntimeException("Insufficient balance");
                }

                from.balance -= amount;
                to.balance += amount;

                Transaction txn = new Transaction(
                    UUID.randomUUID().toString(),
                    fromId,
                    toId,
                    amount,
                    TransactionType.TRANSFER
                );

                txnService.record(txn);
            }
        }
    }

    public double getBalance(String walletId) {
        return walletRepo.get(walletId).balance;
    }

    public List<Transaction> getTransactions(String walletId, TransactionRepository repo) {
        return repo.getByWallet(walletId);
    }
}
```

---

# 6. Complexity Analysis

### Add Money

- Lookup wallet → O(1)
    
- Update balance → O(1)
    

---

### Transfer Money

- Lock acquisition → O(1)
    
- Balance update → O(1)
    

---

### Fetch Transactions

- Iterate list → O(N)
    

---

# 7. Key Design Decisions

- Separate WalletService and TransactionService for clear responsibilities
    
- LockManager used to ensure thread-safe operations
    
- Ordered locking to prevent deadlocks during transfers
    
- Transaction recording separated from business logic
    
- Thread-safe repositories for concurrent access
    
- Singleton Pattern used for centralized lock management
    
- Dependency Injection used for decoupling services
    
- Strong consistency maintained during balance updates