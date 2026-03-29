
# 1. Functional Requirements

- Clients can register webhook URLs for specific events
    
- System triggers webhook delivery on event occurrence
    
- Support multiple subscribers per event
    
- Retry failed webhook deliveries
    
- Allow unregister/remove webhook
    
- Support configurable retry policies
    
- Ensure idempotent delivery
    

---

# 2. Non-Functional Requirements

- High reliability (no message loss)
    
- Scalable for high event volume
    
- Low latency for webhook triggering
    
- Fault-tolerant with retries
    
- Thread-safe operations
    
- Observability (logging, status tracking)
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|Event|Represents a system event|
|Webhook|Subscriber endpoint (URL + metadata)|
|DeliveryTask|Represents a webhook delivery attempt|
|RetryPolicy|Defines retry behavior|
|WebhookRepository|Stores webhook subscriptions|
|DeliveryQueue|Queue for async processing|
|WebhookService|Manages webhook registration|
|DeliveryService|Handles delivery and retries|

---

# 4. Flows (Use Cases)

---

## Register Webhook

1. Client registers webhook with event type and URL
    
2. WebhookService stores it in repository
    

---

## Trigger Event

1. Event occurs in system
    
2. Fetch all webhooks for the event
    
3. Create DeliveryTasks
    
4. Push tasks to DeliveryQueue
    

---

## Deliver Webhook

1. DeliveryService consumes task from queue
    
2. Sends HTTP request to webhook URL
    
3. If success → mark completed
    
4. If failure → retry based on policy
    

---

## Retry Flow

1. Delivery fails
    
2. Increment retry count
    
3. Check retry policy
    
4. Requeue task with delay if allowed
    
5. If max retries exceeded → mark failed
    

---

## Unregister Webhook

1. Client requests removal
    
2. WebhookService deletes from repository
    

---

# 5. Code

```java
import java.util.*;
import java.util.concurrent.*;

// EVENT
class Event {
    String type;
    String payload;

    public Event(String type, String payload) {
        this.type = type;
        this.payload = payload;
    }
}

// WEBHOOK
class Webhook {
    String id;
    String url;
    String eventType;

    public Webhook(String id, String url, String eventType) {
        this.id = id;
        this.url = url;
        this.eventType = eventType;
    }
}

// DELIVERY TASK
class DeliveryTask {
    Webhook webhook;
    Event event;
    int retryCount;

    public DeliveryTask(Webhook webhook, Event event) {
        this.webhook = webhook;
        this.event = event;
        this.retryCount = 0;
    }
}

// RETRY POLICY (Strategy Pattern)
interface RetryPolicy {
    boolean shouldRetry(int retryCount);
    long nextDelay(int retryCount);
}

class ExponentialBackoffRetry implements RetryPolicy {
    private int maxRetries;

    public ExponentialBackoffRetry(int maxRetries) {
        this.maxRetries = maxRetries;
    }

    public boolean shouldRetry(int retryCount) {
        return retryCount < maxRetries;
    }

    public long nextDelay(int retryCount) {
        return (long) Math.pow(2, retryCount) * 1000;
    }
}

// REPOSITORY
class WebhookRepository {
    private Map<String, List<Webhook>> store = new ConcurrentHashMap<>();

    public void addWebhook(Webhook webhook) {
        store.putIfAbsent(webhook.eventType, new CopyOnWriteArrayList<>());
        store.get(webhook.eventType).add(webhook);
    }

    public void removeWebhook(Webhook webhook) {
        List<Webhook> list = store.get(webhook.eventType);
        if (list != null) {
            list.remove(webhook);
        }
    }

    public List<Webhook> getWebhooks(String eventType) {
        return store.getOrDefault(eventType, new ArrayList<>());
    }
}

// DELIVERY QUEUE (Producer-Consumer Pattern)
class DeliveryQueue {
    private BlockingQueue<DeliveryTask> queue = new LinkedBlockingQueue<>();

    public void enqueue(DeliveryTask task) {
        queue.offer(task);
    }

    public DeliveryTask dequeue() throws InterruptedException {
        return queue.take();
    }
}

// DELIVERY SERVICE
class DeliveryService {
    private DeliveryQueue queue;
    private RetryPolicy retryPolicy;
    private ExecutorService executor = Executors.newFixedThreadPool(5);

    public DeliveryService(DeliveryQueue queue, RetryPolicy retryPolicy) {
        this.queue = queue;
        this.retryPolicy = retryPolicy;
        startWorkers();
    }

    private void startWorkers() {
        for (int i = 0; i < 5; i++) {
            executor.submit(() -> {
                while (true) {
                    try {
                        DeliveryTask task = queue.dequeue();
                        process(task);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
            });
        }
    }

    private void process(DeliveryTask task) {
        boolean success = send(task.webhook.url, task.event.payload);

        if (!success) {
            task.retryCount++;

            if (retryPolicy.shouldRetry(task.retryCount)) {
                try {
                    Thread.sleep(retryPolicy.nextDelay(task.retryCount));
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                queue.enqueue(task);
            } else {
                System.out.println("Failed permanently: " + task.webhook.url);
            }
        }
    }

    private boolean send(String url, String payload) {
        // Simulated HTTP call
        return Math.random() > 0.3;
    }
}

// WEBHOOK SERVICE
class WebhookService {
    private WebhookRepository repo;

    public WebhookService(WebhookRepository repo) {
        this.repo = repo;
    }

    public void registerWebhook(Webhook webhook) {
        repo.addWebhook(webhook);
    }

    public void unregisterWebhook(Webhook webhook) {
        repo.removeWebhook(webhook);
    }

    public void handleEvent(Event event, DeliveryQueue queue) {
        List<Webhook> hooks = repo.getWebhooks(event.type);

        for (Webhook hook : hooks) {
            queue.enqueue(new DeliveryTask(hook, event));
        }
    }
}
```

---

# 6. Complexity Analysis

### Register / Unregister Webhook

- O(1)
    

---

### Trigger Event

- Fetch subscribers → O(N)
    
- Enqueue tasks → O(N)
    

**Overall: O(N)**

---

### Delivery Processing

- O(1) per task
    

---

# 7. Key Design Decisions

- Queue-based asynchronous processing to decouple event generation and delivery
    
- Retry mechanism with exponential backoff for handling transient failures
    
- Separate services for webhook management and delivery logic
    
- Thread pool for parallel processing of delivery tasks
    
- Thread-safe data structures for concurrent access
    
- Strategy Pattern used for flexible retry policies
    
- Producer-Consumer Pattern used for asynchronous task processing
    
- Observer Pattern used for event-to-webhook subscription model
    
- Dependency Injection used to decouple services and improve testability