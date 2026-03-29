
# 1. Functional Requirements

- Send notifications to users
    
- Support multiple notification channels (Email, SMS, Push)
    
- Users can subscribe/unsubscribe to notification types
    
- Support event-based notifications
    
- Retry failed notifications
    
- Support user preferences (channel selection, opt-in/out)
    

---

# 2. Non-Functional Requirements

- High scalability
    
- Low latency for notification triggering
    
- Reliable delivery with retries
    
- Fault-tolerant system
    
- Extensible for new notification channels
    
- Thread-safe operations
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|Notification|Represents a notification message|
|User|Represents recipient|
|Channel|Delivery medium (Email/SMS/Push)|
|NotificationRequest|Encapsulates notification trigger|
|UserPreference|Stores user channel preferences|
|NotificationService|Orchestrates notification flow|
|ChannelHandler|Sends notification via specific channel|
|RetryPolicy|Defines retry behavior|
|DeliveryQueue|Queue for async processing|

---

# 4. Flows (Use Cases)

---

## Send Notification

1. Event triggers notification request
    
2. NotificationService fetches user preferences
    
3. Select applicable channels
    
4. Create notification tasks
    
5. Push tasks to DeliveryQueue
    

---

## Process Notification

1. Worker consumes task from queue
    
2. ChannelHandler sends notification
    
3. If success → mark delivered
    
4. If failure → retry based on policy
    

---

## Retry Flow

1. Delivery fails
    
2. Increment retry count
    
3. Check retry policy
    
4. Requeue task if allowed
    
5. Else mark failed
    

---

## Update User Preferences

1. User updates preferred channels
    
2. System stores preferences
    

---

# 5. Code

```java
import java.util.*;
import java.util.concurrent.*;

// ENUMS
enum ChannelType {
    EMAIL, SMS, PUSH
}

// MODELS
class Notification {
    String userId;
    String message;
    ChannelType channel;

    public Notification(String userId, String message, ChannelType channel) {
        this.userId = userId;
        this.message = message;
        this.channel = channel;
    }
}

class NotificationRequest {
    String userId;
    String message;

    public NotificationRequest(String userId, String message) {
        this.userId = userId;
        this.message = message;
    }
}

class UserPreference {
    String userId;
    Set<ChannelType> channels;

    public UserPreference(String userId, Set<ChannelType> channels) {
        this.userId = userId;
        this.channels = channels;
    }
}

// CHANNEL HANDLER (Strategy Pattern)
interface ChannelHandler {
    void send(Notification notification);
}

class EmailHandler implements ChannelHandler {
    public void send(Notification notification) {
        System.out.println("Email sent: " + notification.message);
    }
}

class SMSHandler implements ChannelHandler {
    public void send(Notification notification) {
        System.out.println("SMS sent: " + notification.message);
    }
}

class PushHandler implements ChannelHandler {
    public void send(Notification notification) {
        System.out.println("Push sent: " + notification.message);
    }
}

// FACTORY
class ChannelFactory {
    public static ChannelHandler getHandler(ChannelType type) {
        switch (type) {
            case EMAIL: return new EmailHandler();
            case SMS: return new SMSHandler();
            case PUSH: return new PushHandler();
            default: throw new IllegalArgumentException();
        }
    }
}

// RETRY POLICY (Strategy Pattern)
interface RetryPolicy {
    boolean shouldRetry(int retryCount);
}

class FixedRetryPolicy implements RetryPolicy {
    private int maxRetries;

    public FixedRetryPolicy(int maxRetries) {
        this.maxRetries = maxRetries;
    }

    public boolean shouldRetry(int retryCount) {
        return retryCount < maxRetries;
    }
}

// TASK
class NotificationTask {
    Notification notification;
    int retryCount;

    public NotificationTask(Notification notification) {
        this.notification = notification;
        this.retryCount = 0;
    }
}

// QUEUE (Producer-Consumer)
class DeliveryQueue {
    private BlockingQueue<NotificationTask> queue = new LinkedBlockingQueue<>();

    public void enqueue(NotificationTask task) {
        queue.offer(task);
    }

    public NotificationTask dequeue() throws InterruptedException {
        return queue.take();
    }
}

// SERVICE
class NotificationService {
    private Map<String, UserPreference> preferences = new ConcurrentHashMap<>();
    private DeliveryQueue queue;

    public NotificationService(DeliveryQueue queue) {
        this.queue = queue;
    }

    public void updatePreference(UserPreference pref) {
        preferences.put(pref.userId, pref);
    }

    public void sendNotification(NotificationRequest request) {
        UserPreference pref = preferences.get(request.userId);

        if (pref == null) return;

        for (ChannelType channel : pref.channels) {
            Notification notification = new Notification(
                request.userId,
                request.message,
                channel
            );
            queue.enqueue(new NotificationTask(notification));
        }
    }
}

// WORKER
class NotificationWorker {
    private DeliveryQueue queue;
    private RetryPolicy retryPolicy;
    private ExecutorService executor = Executors.newFixedThreadPool(5);

    public NotificationWorker(DeliveryQueue queue, RetryPolicy retryPolicy) {
        this.queue = queue;
        this.retryPolicy = retryPolicy;
        start();
    }

    private void start() {
        for (int i = 0; i < 5; i++) {
            executor.submit(() -> {
                while (true) {
                    try {
                        NotificationTask task = queue.dequeue();
                        process(task);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
            });
        }
    }

    private void process(NotificationTask task) {
        try {
            ChannelHandler handler = ChannelFactory.getHandler(task.notification.channel);
            handler.send(task.notification);
        } catch (Exception e) {
            task.retryCount++;
            if (retryPolicy.shouldRetry(task.retryCount)) {
                queue.enqueue(task);
            } else {
                System.out.println("Failed notification: " + task.notification.message);
            }
        }
    }
}
```

---

# 6. Complexity Analysis

### Send Notification

- Fetch preferences → O(1)
    
- Create tasks → O(C)
    

**Overall: O(C)** (C = number of channels)

---

### Processing Task

- Send notification → O(1)
    

---

# 7. Key Design Decisions

- Strategy Pattern used for channel-specific delivery logic
    
- Factory Pattern used for creating channel handlers
    
- Strategy Pattern used for retry mechanism
    
- Producer-Consumer Pattern used for asynchronous processing
    
- Queue-based design for scalability and decoupling
    
- Thread pool used for parallel notification processing
    
- User preferences used to control delivery channels
    
- Thread-safe data structures used for concurrency
    
- Dependency Injection used for better testability and flexibility