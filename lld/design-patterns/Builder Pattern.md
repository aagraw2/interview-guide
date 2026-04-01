## 1. What is Builder Pattern?

Builder pattern constructs complex objects step by step. It separates object construction from representation, allowing same construction process to create different representations.

```
Without Builder:
  Computer c = new Computer("Dell", 16, 512, 8, 6);  // What do these numbers mean?
  
With Builder:
  Computer c = new Computer.Builder("Dell")
      .setRAM(16)
      .setSSD(512)
      .setCPU(8)
      .setGPU(6)
      .build();  // Clear and readable
```

**Key principle:** Build complex objects step by step with a fluent interface.

---

## 2. Implementation Guide

### Step 1: Create Product Class with Private Constructor

```java
public class Computer {
    // Required parameters
    private final String brand;
    
    // Optional parameters
    private final int ram;
    private final int ssd;
    private final int cpu;
    private final int gpu;
    private final boolean hasWifi;
    private final boolean hasBluetooth;
    
    // Private constructor - only Builder can create
    private Computer(Builder builder) {
        this.brand = builder.brand;
        this.ram = builder.ram;
        this.ssd = builder.ssd;
        this.cpu = builder.cpu;
        this.gpu = builder.gpu;
        this.hasWifi = builder.hasWifi;
        this.hasBluetooth = builder.hasBluetooth;
    }
    
    // Getters only (immutable object)
    public String getBrand() { return brand; }
    public int getRam() { return ram; }
    public int getSsd() { return ssd; }
    public int getCpu() { return cpu; }
    public int getGpu() { return gpu; }
    public boolean hasWifi() { return hasWifi; }
    public boolean hasBluetooth() { return hasBluetooth; }
    
    @Override
    public String toString() {
        return "Computer{brand='" + brand + "', ram=" + ram + "GB, ssd=" + ssd + 
               "GB, cpu=" + cpu + " cores, gpu=" + gpu + "GB}";
    }
    
    // Static nested Builder class
    public static class Builder {
        // Required parameters
        private final String brand;
        
        // Optional parameters - initialized to default values
        private int ram = 8;
        private int ssd = 256;
        private int cpu = 4;
        private int gpu = 2;
        private boolean hasWifi = true;
        private boolean hasBluetooth = false;
        
        // Constructor with required parameters
        public Builder(String brand) {
            if (brand == null || brand.isEmpty()) {
                throw new IllegalArgumentException("Brand is required");
            }
            this.brand = brand;
        }
        
        // Setter methods return Builder for chaining
        public Builder setRAM(int ram) {
            if (ram <= 0) {
                throw new IllegalArgumentException("RAM must be positive");
            }
            this.ram = ram;
            return this;
        }
        
        public Builder setSSD(int ssd) {
            if (ssd <= 0) {
                throw new IllegalArgumentException("SSD must be positive");
            }
            this.ssd = ssd;
            return this;
        }
        
        public Builder setCPU(int cpu) {
            if (cpu <= 0) {
                throw new IllegalArgumentException("CPU cores must be positive");
            }
            this.cpu = cpu;
            return this;
        }
        
        public Builder setGPU(int gpu) {
            if (gpu < 0) {
                throw new IllegalArgumentException("GPU cannot be negative");
            }
            this.gpu = gpu;
            return this;
        }
        
        public Builder setWifi(boolean hasWifi) {
            this.hasWifi = hasWifi;
            return this;
        }
        
        public Builder setBluetooth(boolean hasBluetooth) {
            this.hasBluetooth = hasBluetooth;
            return this;
        }
        
        // Build method creates the Computer
        public Computer build() {
            // Validation before building
            if (ram < 4) {
                throw new IllegalStateException("RAM must be at least 4GB");
            }
            return new Computer(this);
        }
    }
}
```

### Step 2: Usage

```java
// Create computer with all options
Computer gamingPC = new Computer.Builder("Alienware")
    .setRAM(32)
    .setSSD(1024)
    .setCPU(16)
    .setGPU(12)
    .setWifi(true)
    .setBluetooth(true)
    .build();

// Create computer with minimal options (uses defaults)
Computer officePC = new Computer.Builder("Dell")
    .setRAM(8)
    .setSSD(256)
    .build();

// Create computer with some options
Computer laptop = new Computer.Builder("HP")
    .setRAM(16)
    .setSSD(512)
    .setCPU(8)
    .build();

System.out.println(gamingPC);
System.out.println(officePC);
```

---

## 3. Common Interview Questions

### Q1: Build HTTP Request

```java
public class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final Map<String, String> params;
    private final String body;
    private final int timeout;
    
    private HttpRequest(Builder builder) {
        this.url = builder.url;
        this.method = builder.method;
        this.headers = builder.headers;
        this.params = builder.params;
        this.body = builder.body;
        this.timeout = builder.timeout;
    }
    
    public static class Builder {
        private final String url;
        private String method = "GET";
        private Map<String, String> headers = new HashMap<>();
        private Map<String, String> params = new HashMap<>();
        private String body = "";
        private int timeout = 30000;
        
        public Builder(String url) {
            this.url = url;
        }
        
        public Builder method(String method) {
            this.method = method;
            return this;
        }
        
        public Builder addHeader(String key, String value) {
            this.headers.put(key, value);
            return this;
        }
        
        public Builder addParam(String key, String value) {
            this.params.put(key, value);
            return this;
        }
        
        public Builder body(String body) {
            this.body = body;
            return this;
        }
        
        public Builder timeout(int timeout) {
            this.timeout = timeout;
            return this;
        }
        
        public HttpRequest build() {
            return new HttpRequest(this);
        }
    }
    
    public void send() {
        System.out.println("Sending " + method + " request to " + url);
        System.out.println("Headers: " + headers);
        System.out.println("Params: " + params);
        System.out.println("Body: " + body);
    }
}

// Usage
HttpRequest request = new HttpRequest.Builder("https://api.example.com/users")
    .method("POST")
    .addHeader("Content-Type", "application/json")
    .addHeader("Authorization", "Bearer token123")
    .body("{\"name\":\"John\",\"age\":30}")
    .timeout(5000)
    .build();

request.send();
```

### Q2: Build SQL Query

```java
public class SqlQuery {
    private final String table;
    private final List<String> columns;
    private final String whereClause;
    private final String orderBy;
    private final Integer limit;
    private final Integer offset;
    
    private SqlQuery(Builder builder) {
        this.table = builder.table;
        this.columns = builder.columns;
        this.whereClause = builder.whereClause;
        this.orderBy = builder.orderBy;
        this.limit = builder.limit;
        this.offset = builder.offset;
    }
    
    public String toSql() {
        StringBuilder sql = new StringBuilder("SELECT ");
        
        if (columns.isEmpty()) {
            sql.append("*");
        } else {
            sql.append(String.join(", ", columns));
        }
        
        sql.append(" FROM ").append(table);
        
        if (whereClause != null) {
            sql.append(" WHERE ").append(whereClause);
        }
        
        if (orderBy != null) {
            sql.append(" ORDER BY ").append(orderBy);
        }
        
        if (limit != null) {
            sql.append(" LIMIT ").append(limit);
        }
        
        if (offset != null) {
            sql.append(" OFFSET ").append(offset);
        }
        
        return sql.toString();
    }
    
    public static class Builder {
        private final String table;
        private List<String> columns = new ArrayList<>();
        private String whereClause;
        private String orderBy;
        private Integer limit;
        private Integer offset;
        
        public Builder(String table) {
            this.table = table;
        }
        
        public Builder select(String... columns) {
            this.columns = Arrays.asList(columns);
            return this;
        }
        
        public Builder where(String condition) {
            this.whereClause = condition;
            return this;
        }
        
        public Builder orderBy(String column) {
            this.orderBy = column;
            return this;
        }
        
        public Builder limit(int limit) {
            this.limit = limit;
            return this;
        }
        
        public Builder offset(int offset) {
            this.offset = offset;
            return this;
        }
        
        public SqlQuery build() {
            return new SqlQuery(this);
        }
    }
}

// Usage
SqlQuery query = new SqlQuery.Builder("users")
    .select("id", "name", "email")
    .where("age > 18")
    .orderBy("name ASC")
    .limit(10)
    .offset(20)
    .build();

System.out.println(query.toSql());
// Output: SELECT id, name, email FROM users WHERE age > 18 ORDER BY name ASC LIMIT 10 OFFSET 20
```

### Q3: Build Pizza

```java
public class Pizza {
    private final String size;
    private final boolean cheese;
    private final boolean pepperoni;
    private final boolean mushrooms;
    private final boolean olives;
    private final boolean bacon;
    
    private Pizza(Builder builder) {
        this.size = builder.size;
        this.cheese = builder.cheese;
        this.pepperoni = builder.pepperoni;
        this.mushrooms = builder.mushrooms;
        this.olives = builder.olives;
        this.bacon = builder.bacon;
    }
    
    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder(size + " pizza with: ");
        List<String> toppings = new ArrayList<>();
        if (cheese) toppings.add("cheese");
        if (pepperoni) toppings.add("pepperoni");
        if (mushrooms) toppings.add("mushrooms");
        if (olives) toppings.add("olives");
        if (bacon) toppings.add("bacon");
        sb.append(String.join(", ", toppings));
        return sb.toString();
    }
    
    public static class Builder {
        private String size = "Medium";
        private boolean cheese = false;
        private boolean pepperoni = false;
        private boolean mushrooms = false;
        private boolean olives = false;
        private boolean bacon = false;
        
        public Builder size(String size) {
            this.size = size;
            return this;
        }
        
        public Builder addCheese() {
            this.cheese = true;
            return this;
        }
        
        public Builder addPepperoni() {
            this.pepperoni = true;
            return this;
        }
        
        public Builder addMushrooms() {
            this.mushrooms = true;
            return this;
        }
        
        public Builder addOlives() {
            this.olives = true;
            return this;
        }
        
        public Builder addBacon() {
            this.bacon = true;
            return this;
        }
        
        public Pizza build() {
            return new Pizza(this);
        }
    }
}

// Usage
Pizza pizza = new Pizza.Builder()
    .size("Large")
    .addCheese()
    .addPepperoni()
    .addMushrooms()
    .build();

System.out.println(pizza);
```

---

## 4. Interview Tricks & Pitfalls

### Pitfall 1: Forgetting to Return 'this'

```java
// WRONG: Can't chain methods
public Builder setRAM(int ram) {
    this.ram = ram;
    // Missing return this;
}

// CORRECT: Return this for chaining
public Builder setRAM(int ram) {
    this.ram = ram;
    return this;
}
```

### Pitfall 2: Mutable Objects

```java
// WRONG: Object can be modified after creation
public class Computer {
    private int ram;
    
    public void setRam(int ram) {  // Setter allows modification
        this.ram = ram;
    }
}

// CORRECT: Make object immutable
public class Computer {
    private final int ram;  // final
    
    public int getRam() {  // Only getter, no setter
        return ram;
    }
}
```

### Pitfall 3: Not Validating in build()

```java
// WRONG: No validation
public Computer build() {
    return new Computer(this);
}

// CORRECT: Validate before building
public Computer build() {
    if (ram < 4) {
        throw new IllegalStateException("RAM must be at least 4GB");
    }
    if (ssd < 128) {
        throw new IllegalStateException("SSD must be at least 128GB");
    }
    return new Computer(this);
}
```

### Pitfall 4: Required vs Optional Parameters

```java
// WRONG: All parameters in builder constructor
public Builder(String brand, int ram, int ssd) {
    // Too many required parameters
}

// CORRECT: Only truly required parameters in constructor
public Builder(String brand) {
    this.brand = brand;
    // Optional parameters have defaults
}
```

---

## 5. Builder vs Constructor

| Aspect | Constructor | Builder |
|--------|-------------|---------|
| Readability | `new Computer(16, 512, 8, 6)` unclear | `builder.setRAM(16).setSSD(512)` clear |
| Optional params | Need multiple constructors | Single builder |
| Immutability | Easy | Easy |
| Validation | In constructor | In build() method |
| When to use | Few parameters | Many parameters (>3-4) |

---

## 6. When to Use

✅ **Use when:**
- Many constructor parameters (>3-4)
- Many optional parameters
- Need immutable objects
- Want readable object creation
- Complex object construction

❌ **Don't use when:**
- Simple objects with few parameters
- Object is mutable by design
- No optional parameters

---

## 7. Real-World Examples

- **Java**: `StringBuilder`, `StringBuffer`
- **Java 8**: `Stream.Builder`
- **HTTP Clients**: OkHttp, Apache HttpClient
- **Testing**: Mockito builders
- **Android**: `AlertDialog.Builder`, `Notification.Builder`

---

## 8. Quick Implementation Checklist

1. ☐ Make product class with final fields
2. ☐ Create private constructor taking Builder
3. ☐ Create static nested Builder class
4. ☐ Builder constructor takes required parameters
5. ☐ Builder setter methods return `this`
6. ☐ Builder has `build()` method
7. ☐ Validate in `build()` method
8. ☐ Product class has only getters (immutable)

**Template:**
```java
public class Product {
    private final String required;
    private final int optional;
    
    private Product(Builder builder) {
        this.required = builder.required;
        this.optional = builder.optional;
    }
    
    public static class Builder {
        private final String required;
        private int optional = 0;
        
        public Builder(String required) {
            this.required = required;
        }
        
        public Builder setOptional(int optional) {
            this.optional = optional;
            return this;
        }
        
        public Product build() {
            // Validate
            return new Product(this);
        }
    }
}
```
