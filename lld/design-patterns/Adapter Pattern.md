## 1. What is Adapter Pattern?

Adapter pattern allows incompatible interfaces to work together. It acts as a bridge between two incompatible interfaces by wrapping an existing class with a new interface.

```
Without Adapter:
  Client expects: processPayment(amount)
  PayPal provides: makePayment(amount)
  ❌ Incompatible!

With Adapter:
  Client → Adapter → PayPal
  Adapter translates processPayment() to makePayment()
  ✅ Compatible!
```

**Key principle:** Convert interface of a class into another interface clients expect.

---

## 2. Implementation Guide

### Step 1: Define Target Interface (What Client Expects)

```java
public interface PaymentProcessor {
    void processPayment(double amount);
    boolean refund(String transactionId, double amount);
}
```

### Step 2: Existing Class (Adaptee - Incompatible Interface)

```java
public class PayPalPayment {
    public void makePayment(double amount) {
        System.out.println("Processing PayPal payment of $" + amount);
    }
    
    public void sendRefund(String txnId, double amount) {
        System.out.println("Refunding $" + amount + " to transaction " + txnId);
    }
}
```

### Step 3: Create Adapter

```java
public class PayPalAdapter implements PaymentProcessor {
    private PayPalPayment payPalPayment;
    
    public PayPalAdapter(PayPalPayment payPalPayment) {
        this.payPalPayment = payPalPayment;
    }
    
    @Override
    public void processPayment(double amount) {
        // Adapt: translate processPayment to makePayment
        payPalPayment.makePayment(amount);
    }
    
    @Override
    public boolean refund(String transactionId, double amount) {
        // Adapt: translate refund to sendRefund
        payPalPayment.sendRefund(transactionId, amount);
        return true;
    }
}
```

### Step 4: Usage

```java
// Client code expects PaymentProcessor interface
public class PaymentService {
    public void makePayment(PaymentProcessor processor, double amount) {
        processor.processPayment(amount);
    }
}

// Using adapter
PayPalPayment payPal = new PayPalPayment();
PaymentProcessor processor = new PayPalAdapter(payPal);

PaymentService service = new PaymentService();
service.makePayment(processor, 100.0);
```

---

## 3. Object Adapter vs Class Adapter

### Object Adapter (Composition - Recommended)

```java
public class Adapter implements Target {
    private Adaptee adaptee;  // Composition
    
    public Adapter(Adaptee adaptee) {
        this.adaptee = adaptee;
    }
    
    @Override
    public void request() {
        adaptee.specificRequest();
    }
}
```

**Pros:** More flexible, can adapt multiple adaptees

### Class Adapter (Inheritance)

```java
public class Adapter extends Adaptee implements Target {
    @Override
    public void request() {
        specificRequest();  // Inherited method
    }
}
```

**Pros:** Less code
**Cons:** Single inheritance limitation, less flexible

---

## 4. Common Interview Questions

### Q1: Media Player

```java
// Target interface
public interface MediaPlayer {
    void play(String audioType, String fileName);
}

// Adaptee 1
public class MP3Player {
    public void playMP3(String fileName) {
        System.out.println("Playing MP3 file: " + fileName);
    }
}

// Adaptee 2
public class MP4Player {
    public void playMP4(String fileName) {
        System.out.println("Playing MP4 file: " + fileName);
    }
}

// Adaptee 3
public class VLCPlayer {
    public void playVLC(String fileName) {
        System.out.println("Playing VLC file: " + fileName);
    }
}

// Adapter
public class MediaAdapter implements MediaPlayer {
    private MP3Player mp3Player;
    private MP4Player mp4Player;
    private VLCPlayer vlcPlayer;
    
    public MediaAdapter(String audioType) {
        if (audioType.equalsIgnoreCase("mp3")) {
            mp3Player = new MP3Player();
        } else if (audioType.equalsIgnoreCase("mp4")) {
            mp4Player = new MP4Player();
        } else if (audioType.equalsIgnoreCase("vlc")) {
            vlcPlayer = new VLCPlayer();
        }
    }
    
    @Override
    public void play(String audioType, String fileName) {
        if (audioType.equalsIgnoreCase("mp3")) {
            mp3Player.playMP3(fileName);
        } else if (audioType.equalsIgnoreCase("mp4")) {
            mp4Player.playMP4(fileName);
        } else if (audioType.equalsIgnoreCase("vlc")) {
            vlcPlayer.playVLC(fileName);
        }
    }
}

// Concrete player using adapter
public class AudioPlayer implements MediaPlayer {
    private MediaAdapter mediaAdapter;
    
    @Override
    public void play(String audioType, String fileName) {
        if (audioType.equalsIgnoreCase("mp3")) {
            System.out.println("Playing MP3 file: " + fileName);
        } else if (audioType.equalsIgnoreCase("mp4") || 
                   audioType.equalsIgnoreCase("vlc")) {
            mediaAdapter = new MediaAdapter(audioType);
            mediaAdapter.play(audioType, fileName);
        } else {
            System.out.println("Invalid format: " + audioType);
        }
    }
}

// Usage
MediaPlayer player = new AudioPlayer();
player.play("mp3", "song.mp3");
player.play("mp4", "video.mp4");
player.play("vlc", "movie.vlc");
```

### Q2: Temperature Sensor

```java
// Target interface (what client expects)
public interface TemperatureSensor {
    double getTemperatureInCelsius();
}

// Adaptee (existing class with different interface)
public class FahrenheitSensor {
    public double getTemperatureInFahrenheit() {
        return 98.6;  // Body temperature in Fahrenheit
    }
}

// Adapter
public class FahrenheitToCelsiusAdapter implements TemperatureSensor {
    private FahrenheitSensor sensor;
    
    public FahrenheitToCelsiusAdapter(FahrenheitSensor sensor) {
        this.sensor = sensor;
    }
    
    @Override
    public double getTemperatureInCelsius() {
        double fahrenheit = sensor.getTemperatureInFahrenheit();
        return (fahrenheit - 32) * 5 / 9;
    }
}

// Usage
FahrenheitSensor fSensor = new FahrenheitSensor();
TemperatureSensor cSensor = new FahrenheitToCelsiusAdapter(fSensor);

System.out.println("Temperature: " + cSensor.getTemperatureInCelsius() + "°C");
```

### Q3: Database Adapter

```java
// Target interface
public interface Database {
    void connect(String url);
    void executeQuery(String query);
    void disconnect();
}

// Adaptee 1: Legacy MySQL
public class LegacyMySQL {
    public void mysqlConnect(String host, int port, String db) {
        System.out.println("Connecting to MySQL: " + host + ":" + port + "/" + db);
    }
    
    public void runQuery(String sql) {
        System.out.println("MySQL executing: " + sql);
    }
    
    public void close() {
        System.out.println("MySQL connection closed");
    }
}

// Adapter for MySQL
public class MySQLAdapter implements Database {
    private LegacyMySQL mysql;
    
    public MySQLAdapter(LegacyMySQL mysql) {
        this.mysql = mysql;
    }
    
    @Override
    public void connect(String url) {
        // Parse URL: mysql://host:port/database
        String[] parts = url.replace("mysql://", "").split("[:/]");
        String host = parts[0];
        int port = Integer.parseInt(parts[1]);
        String db = parts[2];
        mysql.mysqlConnect(host, port, db);
    }
    
    @Override
    public void executeQuery(String query) {
        mysql.runQuery(query);
    }
    
    @Override
    public void disconnect() {
        mysql.close();
    }
}

// Adaptee 2: MongoDB
public class MongoDBClient {
    public void open(String connectionString) {
        System.out.println("Opening MongoDB: " + connectionString);
    }
    
    public void find(String collection, String filter) {
        System.out.println("MongoDB find in " + collection + " where " + filter);
    }
    
    public void shutdown() {
        System.out.println("MongoDB shutdown");
    }
}

// Adapter for MongoDB
public class MongoDBAdapter implements Database {
    private MongoDBClient mongo;
    
    public MongoDBAdapter(MongoDBClient mongo) {
        this.mongo = mongo;
    }
    
    @Override
    public void connect(String url) {
        mongo.open(url);
    }
    
    @Override
    public void executeQuery(String query) {
        // Convert SQL-like query to MongoDB query
        mongo.find("collection", query);
    }
    
    @Override
    public void disconnect() {
        mongo.shutdown();
    }
}

// Usage
Database db1 = new MySQLAdapter(new LegacyMySQL());
db1.connect("mysql://localhost:3306/mydb");
db1.executeQuery("SELECT * FROM users");
db1.disconnect();

Database db2 = new MongoDBAdapter(new MongoDBClient());
db2.connect("mongodb://localhost:27017/mydb");
db2.executeQuery("users");
db2.disconnect();
```

---

## 5. Interview Tricks & Pitfalls

### Pitfall 1: Adapter Does Too Much

```java
// WRONG: Adapter contains business logic
public class Adapter implements Target {
    public void request() {
        // Business logic here (wrong!)
        if (condition) { /* ... */ }
        adaptee.specificRequest();
    }
}

// CORRECT: Adapter only translates
public class Adapter implements Target {
    public void request() {
        // Only translation, no business logic
        adaptee.specificRequest();
    }
}
```

### Pitfall 2: Not Handling Exceptions

```java
// WRONG: Exceptions not handled
public void processPayment(double amount) {
    paypal.makePayment(amount);  // May throw PayPalException
}

// CORRECT: Translate exceptions
public void processPayment(double amount) {
    try {
        paypal.makePayment(amount);
    } catch (PayPalException e) {
        throw new PaymentException("Payment failed", e);
    }
}
```

### Pitfall 3: Two-Way Adapter

```java
// PROBLEM: Adapter tries to adapt both ways
// SOLUTION: Create separate adapters for each direction
```

---

## 6. Adapter vs Other Patterns

| Pattern | Purpose | When to Use |
|---------|---------|-------------|
| Adapter | Make incompatible interfaces work | Integrating legacy/third-party code |
| Decorator | Add behavior | Extending functionality |
| Proxy | Control access | Lazy loading, access control |
| Facade | Simplify interface | Simplifying complex subsystem |

---

## 7. When to Use

✅ **Use when:**
- Integrating third-party libraries
- Working with legacy code
- Interface mismatch
- Want to reuse existing class

❌ **Don't use when:**
- Can modify existing class
- Interfaces already compatible
- Simple wrapper suffices

---

## 8. Real-World Examples

- **Java**: `InputStreamReader` (adapts `InputStream` to `Reader`)
- **Java**: `Arrays.asList()` (adapts array to `List`)
- **JDBC**: `DriverManager` (adapts different database drivers)
- **Collections**: `Collections.enumeration()` (adapts `Iterator` to `Enumeration`)

---

## 9. Quick Checklist

1. ☐ Identify target interface (what client expects)
2. ☐ Identify adaptee (existing incompatible class)
3. ☐ Create adapter implementing target interface
4. ☐ Adapter holds reference to adaptee
5. ☐ Adapter methods delegate to adaptee
6. ☐ Handle any necessary data transformation
7. ☐ Handle exception translation if needed

**Template:**
```java
// Target interface
public interface Target {
    void request();
}

// Adaptee (existing class)
public class Adaptee {
    public void specificRequest() {
        // Existing functionality
    }
}

// Adapter
public class Adapter implements Target {
    private Adaptee adaptee;
    
    public Adapter(Adaptee adaptee) {
        this.adaptee = adaptee;
    }
    
    @Override
    public void request() {
        // Translate and delegate
        adaptee.specificRequest();
    }
}
```
