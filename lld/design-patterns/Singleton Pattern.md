## 1. What is Singleton Pattern?

The Singleton pattern ensures that a class has only one instance throughout the application lifetime and provides a global point of access to that instance.

```
Without Singleton:
  DatabaseConnection db1 = new DatabaseConnection();
  DatabaseConnection db2 = new DatabaseConnection();
  // Two separate instances, wasteful

With Singleton:
  DatabaseConnection db1 = DatabaseConnection.getInstance();
  DatabaseConnection db2 = DatabaseConnection.getInstance();
  // db1 == db2 (same instance)
```

**Key principle:** One instance, globally accessible.

---

## 2. Key Components

### Private Constructor

Prevents external instantiation.

```java
private Singleton() {
    // Initialization code
}
```

### Static Instance

Holds the single instance.

```java
private static Singleton instance;
```

### Public Static Method

Provides global access point.

```java
public static Singleton getInstance() {
    return instance;
}
```

---

## 3. Implementation Variations

### 3.1 Eager Initialization (Thread-Safe)

Instance created at class loading time.

```java
public class DatabaseConnection {
    
    // Instance created immediately when class loads
    private static final DatabaseConnection instance = new DatabaseConnection();
    
    private DatabaseConnection() {
        System.out.println("Database connection initialized");
    }
    
    public static DatabaseConnection getInstance() {
        return instance;
    }
    
    public void query(String sql) {
        System.out.println("Executing: " + sql);
    }
}

// Usage
DatabaseConnection db = DatabaseConnection.getInstance();
db.query("SELECT * FROM users");
```

**Pros:**
- ✅ Thread-safe (JVM guarantees)
- ✅ Simple implementation
- ✅ No synchronization overhead

**Cons:**
- ❌ Instance created even if never used (wastes memory)
- ❌ No exception handling during initialization
- ❌ Cannot pass parameters to constructor

**When to use:** When instance is lightweight and always needed.

---

### 3.2 Lazy Initialization (Not Thread-Safe)

Instance created when first requested.

```java
public class Logger {
    
    private static Logger instance;
    
    private Logger() {
        System.out.println("Logger initialized");
    }
    
    public static Logger getInstance() {
        if (instance == null) {  // Check if instance exists
            instance = new Logger();  // Create if not
        }
        return instance;
    }
    
    public void log(String message) {
        System.out.println("[LOG] " + message);
    }
}
```

**Pros:**
- ✅ Instance created only when needed (lazy loading)
- ✅ Simple implementation

**Cons:**
- ❌ NOT thread-safe (multiple threads can create multiple instances)
- ❌ Race condition in multi-threaded environment

**When to use:** Single-threaded applications only.

---

### 3.3 Synchronized Method (Thread-Safe but Slow)

```java
public class ConfigManager {
    
    private static ConfigManager instance;
    private Map<String, String> config = new HashMap<>();
    
    private ConfigManager() {
        // Load configuration
        config.put("app.name", "MyApp");
        config.put("app.version", "1.0");
    }
    
    // Synchronized method - thread-safe but slow
    public static synchronized ConfigManager getInstance() {
        if (instance == null) {
            instance = new ConfigManager();
        }
        return instance;
    }
    
    public String getConfig(String key) {
        return config.get(key);
    }
}
```

**Pros:**
- ✅ Thread-safe
- ✅ Lazy initialization

**Cons:**
- ❌ Synchronized on every call (performance overhead)
- ❌ Slow even after instance is created

**When to use:** When thread safety is critical and performance is not.

---

### 3.4 Double-Checked Locking (Recommended)

Best of both worlds - thread-safe and performant.

```java
public class CacheManager {
    
    // volatile ensures visibility across threads
    private static volatile CacheManager instance;
    private Map<String, Object> cache = new ConcurrentHashMap<>();
    
    private CacheManager() {
        System.out.println("Cache initialized");
    }
    
    public static CacheManager getInstance() {
        // First check (no locking) - fast path
        if (instance == null) {
            // Synchronize only for initialization
            synchronized (CacheManager.class) {
                // Second check (with locking) - prevents race condition
                if (instance == null) {
                    instance = new CacheManager();
                }
            }
        }
        return instance;
    }
    
    public void put(String key, Object value) {
        cache.put(key, value);
    }
    
    public Object get(String key) {
        return cache.get(key);
    }
}
```

**Pros:**
- ✅ Thread-safe
- ✅ Lazy initialization
- ✅ High performance (synchronization only during initialization)

**Cons:**
- ❌ Verbose
- ❌ Requires volatile keyword (Java 5+)

**When to use:** Multi-threaded applications where performance matters (MOST COMMON).

---

### 3.5 Bill Pugh Singleton (Best Practice)

Uses inner static helper class.

```java
public class SessionManager {
    
    private Map<String, String> sessions = new HashMap<>();
    
    private SessionManager() {
        System.out.println("Session manager initialized");
    }
    
    // Inner static class - loaded only when getInstance() is called
    private static class SingletonHelper {
        private static final SessionManager INSTANCE = new SessionManager();
    }
    
    public static SessionManager getInstance() {
        return SingletonHelper.INSTANCE;
    }
    
    public void createSession(String userId, String token) {
        sessions.put(userId, token);
    }
    
    public String getSession(String userId) {
        return sessions.get(userId);
    }
}
```

**Pros:**
- ✅ Thread-safe (JVM guarantees)
- ✅ Lazy initialization
- ✅ No synchronization overhead
- ✅ Clean and simple

**Cons:**
- ❌ None (this is the recommended approach)

**When to use:** Default choice for most scenarios (BEST PRACTICE).

---

### 3.6 Enum Singleton (Most Robust)

Protects against reflection and serialization attacks.

```java
public enum DatabasePool {
    
    INSTANCE;
    
    private Connection connection;
    
    DatabasePool() {
        // Initialize connection
        connection = createConnection();
    }
    
    public Connection getConnection() {
        return connection;
    }
    
    private Connection createConnection() {
        System.out.println("Creating database connection");
        return new Connection();  // Simplified
    }
    
    public void executeQuery(String sql) {
        System.out.println("Executing: " + sql);
    }
}

// Usage
DatabasePool.INSTANCE.executeQuery("SELECT * FROM users");
```

**Pros:**
- ✅ Thread-safe
- ✅ Serialization-safe
- ✅ Reflection-proof
- ✅ Simplest implementation

**Cons:**
- ❌ Cannot extend a class (enums can't extend)
- ❌ Eager initialization (created at class loading)

**When to use:** When you need protection against reflection/serialization.

---

## 4. Step-by-Step Implementation Guide

### Step 1: Make Constructor Private

```java
public class Singleton {
    private Singleton() {
        // Prevent instantiation
    }
}
```

### Step 2: Create Static Instance Variable

```java
public class Singleton {
    private static volatile Singleton instance;  // volatile for thread safety
    
    private Singleton() {}
}
```

### Step 3: Provide Public Static Access Method

```java
public class Singleton {
    private static volatile Singleton instance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

### Step 4: Add Business Logic

```java
public class Singleton {
    private static volatile Singleton instance;
    private int counter = 0;
    
    private Singleton() {
        System.out.println("Singleton initialized");
    }
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
    
    public void increment() {
        counter++;
    }
    
    public int getCounter() {
        return counter;
    }
}
```

---

## 5. Common Interview Questions & Implementations

### Q1: Implement Thread-Safe Logger

```java
public class Logger {
    
    private static volatile Logger instance;
    private StringBuilder logBuffer = new StringBuilder();
    
    private Logger() {
        System.out.println("Logger initialized");
    }
    
    public static Logger getInstance() {
        if (instance == null) {
            synchronized (Logger.class) {
                if (instance == null) {
                    instance = new Logger();
                }
            }
        }
        return instance;
    }
    
    public synchronized void log(String level, String message) {
        String logEntry = String.format("[%s] %s: %s%n", 
            System.currentTimeMillis(), level, message);
        logBuffer.append(logEntry);
        System.out.print(logEntry);
    }
    
    public synchronized String getLogs() {
        return logBuffer.toString();
    }
}

// Usage
Logger logger = Logger.getInstance();
logger.log("INFO", "Application started");
logger.log("ERROR", "Something went wrong");
```

### Q2: Implement Configuration Manager

```java
public class ConfigManager {
    
    private static class SingletonHelper {
        private static final ConfigManager INSTANCE = new ConfigManager();
    }
    
    private Properties properties = new Properties();
    
    private ConfigManager() {
        loadConfig();
    }
    
    private void loadConfig() {
        // Load from file or environment
        properties.setProperty("db.host", "localhost");
        properties.setProperty("db.port", "5432");
        properties.setProperty("app.name", "MyApp");
    }
    
    public static ConfigManager getInstance() {
        return SingletonHelper.INSTANCE;
    }
    
    public String get(String key) {
        return properties.getProperty(key);
    }
    
    public void set(String key, String value) {
        properties.setProperty(key, value);
    }
}

// Usage
ConfigManager config = ConfigManager.getInstance();
String dbHost = config.get("db.host");
```

### Q3: Implement Connection Pool

```java
public class ConnectionPool {
    
    private static volatile ConnectionPool instance;
    private Queue<Connection> availableConnections = new LinkedList<>();
    private Set<Connection> usedConnections = new HashSet<>();
    private static final int MAX_POOL_SIZE = 10;
    
    private ConnectionPool() {
        // Initialize pool
        for (int i = 0; i < 5; i++) {
            availableConnections.add(createConnection());
        }
    }
    
    public static ConnectionPool getInstance() {
        if (instance == null) {
            synchronized (ConnectionPool.class) {
                if (instance == null) {
                    instance = new ConnectionPool();
                }
            }
        }
        return instance;
    }
    
    public synchronized Connection getConnection() {
        if (availableConnections.isEmpty()) {
            if (usedConnections.size() < MAX_POOL_SIZE) {
                availableConnections.add(createConnection());
            } else {
                throw new RuntimeException("No connections available");
            }
        }
        
        Connection connection = availableConnections.poll();
        usedConnections.add(connection);
        return connection;
    }
    
    public synchronized void releaseConnection(Connection connection) {
        usedConnections.remove(connection);
        availableConnections.add(connection);
    }
    
    private Connection createConnection() {
        return new Connection();  // Simplified
    }
}
```

---

## 6. Interview Tricks & Pitfalls

### Pitfall 1: Breaking Singleton with Reflection

```java
// PROBLEM: Reflection can break singleton
Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
constructor.setAccessible(true);
Singleton instance1 = constructor.newInstance();  // Creates new instance!

// SOLUTION: Throw exception in constructor
private Singleton() {
    if (instance != null) {
        throw new RuntimeException("Use getInstance() method");
    }
}

// OR use Enum (reflection-proof)
public enum Singleton {
    INSTANCE;
}
```

### Pitfall 2: Breaking Singleton with Serialization

```java
// PROBLEM: Deserialization creates new instance
Singleton instance1 = Singleton.getInstance();
// Serialize and deserialize
Singleton instance2 = deserialize(serialize(instance1));
// instance1 != instance2 (different instances!)

// SOLUTION: Implement readResolve()
public class Singleton implements Serializable {
    private static volatile Singleton instance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        // ... double-checked locking
    }
    
    // Prevents new instance during deserialization
    protected Object readResolve() {
        return getInstance();
    }
}
```

### Pitfall 3: Cloning

```java
// PROBLEM: Cloning can create new instance
Singleton instance1 = Singleton.getInstance();
Singleton instance2 = (Singleton) instance1.clone();  // New instance!

// SOLUTION: Override clone() and throw exception
@Override
protected Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException("Singleton cannot be cloned");
}
```

### Pitfall 4: Multiple ClassLoaders

```java
// PROBLEM: Different classloaders can create different instances
// SOLUTION: Use same classloader or enum singleton
```

### Pitfall 5: Lazy Initialization Race Condition

```java
// WRONG: Not thread-safe
public static Singleton getInstance() {
    if (instance == null) {  // Thread 1 and 2 both see null
        instance = new Singleton();  // Both create instance!
    }
    return instance;
}

// CORRECT: Double-checked locking
public static Singleton getInstance() {
    if (instance == null) {
        synchronized (Singleton.class) {
            if (instance == null) {  // Check again inside synchronized block
                instance = new Singleton();
            }
        }
    }
    return instance;
}
```

---

## 7. Things to Remember

### When to Use Singleton

✅ **Use when:**
- Only one instance needed (database connection, logger, config)
- Global access required
- Resource is expensive to create
- Shared state across application

❌ **Don't use when:**
- You need multiple instances
- Testing is important (singletons are hard to mock)
- State should not be shared
- You're just avoiding passing parameters (use dependency injection instead)

### Thread Safety Checklist

1. ☐ Use `volatile` keyword for instance variable
2. ☐ Implement double-checked locking
3. ☐ Or use Bill Pugh (inner static class)
4. ☐ Or use enum singleton
5. ☐ Never use simple lazy initialization in multi-threaded apps

### Serialization Safety

1. ☐ Implement `readResolve()` method
2. ☐ Or use enum singleton (automatically serialization-safe)

### Reflection Safety

1. ☐ Throw exception in constructor if instance exists
2. ☐ Or use enum singleton (reflection-proof)

---

## 8. Comparison of Implementations

| Implementation | Thread-Safe | Lazy | Performance | Complexity | Recommended |
|---|---|---|---|---|---|
| Eager | ✅ | ❌ | ⭐⭐⭐ | ⭐ | For lightweight objects |
| Lazy (unsafe) | ❌ | ✅ | ⭐⭐⭐ | ⭐ | Single-threaded only |
| Synchronized | ✅ | ✅ | ⭐ | ⭐⭐ | Not recommended |
| Double-Checked | ✅ | ✅ | ⭐⭐⭐ | ⭐⭐⭐ | Good choice |
| Bill Pugh | ✅ | ✅ | ⭐⭐⭐ | ⭐⭐ | **Best practice** |
| Enum | ✅ | ❌ | ⭐⭐⭐ | ⭐ | Most robust |

---

## 9. Real-World Examples

- **Java**: `Runtime.getRuntime()`
- **Spring**: Beans with singleton scope (default)
- **Android**: `Application` class
- **Database**: Connection pools
- **Logging**: Log4j, SLF4J loggers
- **Configuration**: Application settings managers

---

## 10. Quick Implementation Checklist

When implementing Singleton in an interview:

1. ☐ Make constructor private
2. ☐ Create static instance variable (use `volatile` for thread safety)
3. ☐ Provide public static `getInstance()` method
4. ☐ Implement double-checked locking OR use Bill Pugh pattern
5. ☐ Add business logic methods
6. ☐ Consider serialization safety (`readResolve()`)
7. ☐ Consider reflection safety (exception in constructor)
8. ☐ Test with multiple threads if asked

**Recommended template:**
```java
public class Singleton {
    private static volatile Singleton instance;
    
    private Singleton() {
        if (instance != null) {
            throw new RuntimeException("Use getInstance()");
        }
    }
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**Or use Bill Pugh (simpler):**
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
