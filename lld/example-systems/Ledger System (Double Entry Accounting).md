
# 1. Functional Requirements

- Create accounts (User, Merchant, System)
    
- Record transactions using double-entry accounting
    
- Each transaction must have equal debit and credit
    
- Maintain ledger entries per account
    
- Fetch account balance
    
- Fetch transaction history
    
- Ensure idempotency for transactions
    

---

# 2. Non-Functional Requirements

- Strong consistency (no imbalance allowed)
    
- Atomic transactions
    
- Thread-safe operations
    
- High reliability
    
- Scalable
    
- Audit-friendly (immutable logs)
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|Account|Represents a ledger account|
|Entry|Debit/Credit record|
|Transaction|Group of entries|
|EntryType|DEBIT / CREDIT|
|LedgerRepository|Stores entries|
|AccountRepository|Stores accounts|
|TransactionRepository|Stores transactions|
|LedgerService|Handles posting logic|
|IdempotencyStore|Prevent duplicate transactions|

---

# 4. Double Entry Principle

- Every transaction must satisfy:
    
    **Total Debit = Total Credit**
    
- Example:
    
    - User pays ₹100 to merchant
        
        - User Account → DEBIT 100
            
        - Merchant Account → CREDIT 100
            

---

# 5. Flows

---

## Create Account

1. Create account with zero balance
    
2. Store in repository
    

---

## Post Transaction (Atomic)

1. Receive transaction request with entries
    
2. Check idempotency key
    
3. Validate:
    
    - Sum(debits) == Sum(credits)
        
4. Persist transaction
    
5. Persist all entries
    
6. Update account balances
    
7. Save idempotency key
    

---

## Get Balance

1. Fetch account  
    OR
    
2. Aggregate entries
    

---

## Get Transaction History

1. Fetch entries by account
    

---

# 6. Code

```java
import java.util.*;
import java.util.concurrent.*;

// ENUM
enum EntryType {
    DEBIT, CREDIT
}

// MODELS
class Account {
    String id;
    double balance;

    public Account(String id) {
        this.id = id;
        this.balance = 0;
    }
}

class Entry {
    String accountId;
    double amount;
    EntryType type;

    public Entry(String accountId, double amount, EntryType type) {
        this.accountId = accountId;
        this.amount = amount;
        this.type = type;
    }
}

class Transaction {
    String id;
    List<Entry> entries;

    public Transaction(String id, List<Entry> entries) {
        this.id = id;
        this.entries = entries;
    }
}

// IDEMPOTENCY
class IdempotencyStore {
    private Map<String, Transaction> store = new ConcurrentHashMap<>();

    public Transaction get(String key) {
        return store.get(key);
    }

    public void save(String key, Transaction txn) {
        store.put(key, txn);
    }
}

// REPOSITORIES
class AccountRepository {
    private Map<String, Account> store = new ConcurrentHashMap<>();

    public void save(Account acc) {
        store.put(acc.id, acc);
    }

    public Account get(String id) {
        return store.get(id);
    }
}

class TransactionRepository {
    private Map<String, Transaction> store = new ConcurrentHashMap<>();

    public void save(Transaction txn) {
        store.put(txn.id, txn);
    }
}

class LedgerRepository {
    private List<Entry> entries = new CopyOnWriteArrayList<>();

    public void saveAll(List<Entry> list) {
        entries.addAll(list);
    }

    public List<Entry> getByAccount(String accountId) {
        List<Entry> result = new ArrayList<>();
        for (Entry e : entries) {
            if (e.accountId.equals(accountId)) {
                result.add(e);
            }
        }
        return result;
    }
}

// SERVICE
class LedgerService {
    private AccountRepository accRepo;
    private TransactionRepository txnRepo;
    private LedgerRepository ledgerRepo;
    private IdempotencyStore idStore;

    public LedgerService(AccountRepository accRepo,
                         TransactionRepository txnRepo,
                         LedgerRepository ledgerRepo,
                         IdempotencyStore idStore) {
        this.accRepo = accRepo;
        this.txnRepo = txnRepo;
        this.ledgerRepo = ledgerRepo;
        this.idStore = idStore;
    }

    public synchronized Transaction postTransaction(String idempotencyKey,
                                                     List<Entry> entries) {

        // IDEMPOTENCY
        Transaction existing = idStore.get(idempotencyKey);
        if (existing != null) {
            return existing;
        }

        double debit = 0, credit = 0;

        for (Entry e : entries) {
            if (e.type == EntryType.DEBIT) debit += e.amount;
            else credit += e.amount;
        }

        if (debit != credit) {
            throw new RuntimeException("Unbalanced transaction");
        }

        Transaction txn = new Transaction(UUID.randomUUID().toString(), entries);

        // UPDATE BALANCES
        for (Entry e : entries) {
            Account acc = accRepo.get(e.accountId);

            if (e.type == EntryType.DEBIT) {
                acc.balance -= e.amount;
            } else {
                acc.balance += e.amount;
            }
        }

        txnRepo.save(txn);
        ledgerRepo.saveAll(entries);
        idStore.save(idempotencyKey, txn);

        return txn;
    }

    public double getBalance(String accountId) {
        return accRepo.get(accountId).balance;
    }

    public List<Entry> getEntries(String accountId) {
        return ledgerRepo.getByAccount(accountId);
    }
}
```

---

# 7. Complexity Analysis

- Post Transaction → O(N) (N = entries)
    
- Get Balance → O(1)
    
- Get Entries → O(N)
    

---

# 8. Design Patterns Used

- Repository Pattern → Data abstraction
    
- Dependency Injection → Loose coupling
    
- Idempotency Pattern → Prevent duplicate transactions
    
- (Conceptual) Unit of Work → Atomic transaction handling
    

---

# 9. Key Design Decisions

- Enforced double-entry invariant (debit == credit)
    
- Atomic transaction using synchronized block
    
- Idempotency ensures no duplicate postings
    
- Separate transaction and entry storage
    
- Balance maintained for fast reads
    
- Immutable ledger entries for auditability
    
- Thread-safe data structures used
    

---

# 10. Edge Cases Handled

- Duplicate transaction requests
    
- Unbalanced entries
    
- Concurrent updates
    
- Partial failures prevented via atomic block
    

---

# 11. Possible Extensions

- Use database transactions (ACID) instead of in-memory sync
    
- Add account types (ASSET, LIABILITY, etc.)
    
- Support multi-currency transactions
    
- Add reconciliation system
    
- Introduce event sourcing for audit logs
    
- Add reversal transactions instead of updates
    
- Distributed locking for horizontal scaling