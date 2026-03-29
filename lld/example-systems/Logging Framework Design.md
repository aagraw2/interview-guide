
# 1. Functional Requirements

- Log messages at different levels (DEBUG, INFO, WARN, ERROR)
    
- Support multiple appenders (Console, File, DB, etc.)
    
- Allow configurable log format
    
- Enable/disable logging based on level
    
- Support asynchronous logging
    
- Allow multiple loggers
    
- Support configurable logging via builder
    

---

# 2. Non-Functional Requirements

- High performance (minimal latency)
    
- Thread-safe
    
- Scalable
    
- Extensible (new appenders, formats)
    
- Non-blocking logging
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|Logger|Entry point for logging|
|LogLevel|Defines severity levels|
|LogMessage|Encapsulates log data|
|Appender|Output destination|
|Formatter|Formats log messages|
|LogConfig|Holds logger configuration|
|LogConfigBuilder|Builds configuration|
|AsyncDispatcher|Handles async logging|

---

# 4. Design Approach

- Use **Builder Pattern** for flexible logger configuration
    
- Use **Strategy Pattern** for Formatter
    
- Use **Factory Pattern** for Logger creation
    
- Use **Observer-like pattern** for multiple appenders
    
- Use **Chain of Responsibility** for log level filtering
    

---

# 5. Flows

---

## Log Message Flow

1. Client builds LogConfig using builder
    
2. Logger initialized with config
    
3. Client logs message
    
4. Logger checks level
    
5. Create LogMessage
    
6. Send to appenders
    
7. Formatter formats message
    
8. Appender writes output
    

---

## Async Flow

1. Message pushed to queue
    
2. Dispatcher thread consumes
    
3. Sends to appenders
    

---

# 6. Code

```java
import java.util.*;
import java.util.concurrent.*;

// ENUM
enum LogLevel {
    DEBUG, INFO, WARN, ERROR
}

// MODEL
class LogMessage {
    String message;
    LogLevel level;
    long timestamp;

    public LogMessage(String message, LogLevel level) {
        this.message = message;
        this.level = level;
        this.timestamp = System.currentTimeMillis();
    }
}

// STRATEGY: Formatter
interface Formatter {
    String format(LogMessage msg);
}

class SimpleFormatter implements Formatter {
    public String format(LogMessage msg) {
        return "[" + msg.level + "] " + msg.timestamp + " - " + msg.message;
    }
}

// APPENDER
interface Appender {
    void append(LogMessage message);
}

// CONSOLE APPENDER
class ConsoleAppender implements Appender {
    private Formatter formatter;

    public ConsoleAppender(Formatter formatter) {
        this.formatter = formatter;
    }

    public void append(LogMessage message) {
        System.out.println(formatter.format(message));
    }
}

// FILE APPENDER
class FileAppender implements Appender {
    private Formatter formatter;

    public FileAppender(Formatter formatter) {
        this.formatter = formatter;
    }

    public void append(LogMessage message) {
        System.out.println("Writing to file: " + formatter.format(message));
    }
}

// CONFIG (BUILT VIA BUILDER)
class LogConfig {
    LogLevel level;
    List<Appender> appenders;
    boolean isAsync;

    public LogConfig(LogLevel level, List<Appender> appenders, boolean isAsync) {
        this.level = level;
        this.appenders = appenders;
        this.isAsync = isAsync;
    }
}

// BUILDER PATTERN
class LogConfigBuilder {
    private LogLevel level = LogLevel.INFO;
    private List<Appender> appenders = new ArrayList<>();
    private boolean isAsync = false;

    public LogConfigBuilder setLevel(LogLevel level) {
        this.level = level;
        return this;
    }

    public LogConfigBuilder addAppender(Appender appender) {
        this.appenders.add(appender);
        return this;
    }

    public LogConfigBuilder setAsync(boolean async) {
        this.isAsync = async;
        return this;
    }

    public LogConfig build() {
        if (appenders.isEmpty()) {
            appenders.add(new ConsoleAppender(new SimpleFormatter()));
        }
        return new LogConfig(level, appenders, isAsync);
    }
}

// LOGGER
class Logger {
    private LogLevel level;
    private List<Appender> appenders;

    public Logger(LogConfig config) {
        this.level = config.level;
        this.appenders = config.appenders;
    }

    protected void log(LogMessage message) {
        if (message.level.ordinal() < level.ordinal()) return;

        for (Appender appender : appenders) {
            appender.append(message);
        }
    }

    public void debug(String msg) { log(new LogMessage(msg, LogLevel.DEBUG)); }
    public void info(String msg)  { log(new LogMessage(msg, LogLevel.INFO)); }
    public void warn(String msg)  { log(new LogMessage(msg, LogLevel.WARN)); }
    public void error(String msg) { log(new LogMessage(msg, LogLevel.ERROR)); }
}

// ASYNC LOGGER
class AsyncLogger extends Logger {
    private BlockingQueue<LogMessage> queue = new LinkedBlockingQueue<>();

    public AsyncLogger(LogConfig config) {
        super(config);
        start();
    }

    @Override
    protected void log(LogMessage message) {
        queue.offer(message);
    }

    private void start() {
        new Thread(() -> {
            while (true) {
                try {
                    LogMessage msg = queue.take();
                    super.log(msg);
                } catch (Exception ignored) {}
            }
        }).start();
    }
}

// FACTORY
class LoggerFactory {
    public static Logger getLogger(LogConfig config) {
        if (config.isAsync) {
            return new AsyncLogger(config);
        }
        return new Logger(config);
    }
}
```

---

# 7. Complexity Analysis

- Logging → O(N) (N = appenders)
    
- Async enqueue → O(1)
    
- Async processing → O(N)
    

---

# 8. Design Patterns Used

- Builder Pattern → LogConfig creation
    
- Strategy Pattern → Formatter
    
- Factory Pattern → Logger creation
    
- Observer Pattern → Appenders
    
- Chain of Responsibility → Log level filtering
    

---

# 9. Key Design Decisions

- Builder used for flexible configuration
    
- Default fallback appender added
    
- Async vs Sync handled via factory
    
- Formatter decoupled from appender
    
- Log level filtering centralized
    
- Thread-safe queue for async logging
    

---

# 10. Edge Cases Handled

- No appender configured
    
- High-frequency logging
    
- Concurrent logging
    
- Async queue overflow (extendable)
    

---

# 11. Possible Extensions

- File rotation
    
- Structured logging (JSON)
    
- Distributed logging (Kafka, ELK)
    
- Dynamic config reload
    
- Log batching
    
- Backpressure handling
    
- Context propagation (traceId)