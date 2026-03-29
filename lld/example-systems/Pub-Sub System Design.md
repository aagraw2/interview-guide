
# 1. Functional Requirements

- Publisher can publish messages to a topic
    
- Subscriber can subscribe to a topic
    
- Subscriber can unsubscribe from a topic
    
- Messages should be delivered to all subscribers of a topic
    
- Support multiple topics
    
- Support multiple publishers and subscribers
    
- Support synchronous and asynchronous delivery
    

---

# 2. Non-Functional Requirements

- Low latency message delivery
    
- High throughput
    
- Scalable with increasing topics and subscribers
    
- Thread-safe operations
    
- Loose coupling between publishers and subscribers
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|Message|Represents the data being sent|
|Topic|Maintains list of subscribers|
|Publisher|Publishes messages|
|Subscriber|Consumes messages|
|Broker|Routes messages to subscribers|
|DeliveryStrategy|Defines how messages are delivered|

---

# 4. Flows (Use Cases)

---

## Publish Message

1. Publisher sends message to Broker
    
2. Broker retrieves the topic
    
3. Broker fetches subscribers of the topic
    
4. Broker delivers message using delivery strategy
    

---

## Subscribe to Topic

1. Subscriber registers to a topic
    
2. Broker adds subscriber to topic
    

---

## Unsubscribe from Topic

1. Subscriber removes subscription
    
2. Broker updates topic subscriber list
    

---

# 5. Code

```java
import java.util.*;
import java.util.concurrent.*;

// MESSAGE
class Message {
    String content;

    public Message(String content) {
        this.content = content;
    }
}

// SUBSCRIBER
interface Subscriber {
    void consume(Message message);
}

class PrintSubscriber implements Subscriber {
    private String name;

    public PrintSubscriber(String name) {
        this.name = name;
    }

    @Override
    public void consume(Message message) {
        System.out.println(name + " received: " + message.content);
    }
}

// TOPIC
class Topic {
    private String name;
    private List<Subscriber> subscribers = new CopyOnWriteArrayList<>();

    public Topic(String name) {
        this.name = name;
    }

    public void addSubscriber(Subscriber sub) {
        subscribers.add(sub);
    }

    public void removeSubscriber(Subscriber sub) {
        subscribers.remove(sub);
    }

    public List<Subscriber> getSubscribers() {
        return subscribers;
    }
}

// DELIVERY STRATEGY
interface DeliveryStrategy {
    void deliver(List<Subscriber> subscribers, Message message);
}

class SyncDeliveryStrategy implements DeliveryStrategy {
    public void deliver(List<Subscriber> subscribers, Message message) {
        for (Subscriber sub : subscribers) {
            sub.consume(message);
        }
    }
}

class AsyncDeliveryStrategy implements DeliveryStrategy {
    private ExecutorService executor = Executors.newFixedThreadPool(5);

    public void deliver(List<Subscriber> subscribers, Message message) {
        for (Subscriber sub : subscribers) {
            executor.submit(() -> sub.consume(message));
        }
    }
}

// BROKER
class Broker {
    private Map<String, Topic> topics = new ConcurrentHashMap<>();
    private DeliveryStrategy deliveryStrategy;

    public Broker(DeliveryStrategy strategy) {
        this.deliveryStrategy = strategy;
    }

    public void createTopic(String topicName) {
        topics.putIfAbsent(topicName, new Topic(topicName));
    }

    public void subscribe(String topicName, Subscriber subscriber) {
        topics.get(topicName).addSubscriber(subscriber);
    }

    public void unsubscribe(String topicName, Subscriber subscriber) {
        topics.get(topicName).removeSubscriber(subscriber);
    }

    public void publish(String topicName, Message message) {
        Topic topic = topics.get(topicName);
        if (topic == null) return;

        deliveryStrategy.deliver(topic.getSubscribers(), message);
    }
}

// PUBLISHER
class Publisher {
    private Broker broker;

    public Publisher(Broker broker) {
        this.broker = broker;
    }

    public void publish(String topic, String content) {
        broker.publish(topic, new Message(content));
    }
}
```

---

# 6. Complexity Analysis

### Publish Message

- Fetch topic → O(1)
    
- Deliver to subscribers → O(N)
    

**Overall: O(N)**

---

### Subscribe / Unsubscribe

- Add/remove subscriber → O(1)
    

---

# 7. Key Design Decisions

- Broker acts as central routing component
    
- Topic maintains subscriber list for efficient lookup
    
- Strategy pattern enables flexible delivery mechanisms
    
- Thread-safe collections used for concurrent operations
    
- Decoupled publisher and subscriber via broker abstraction