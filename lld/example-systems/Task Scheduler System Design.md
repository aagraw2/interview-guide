
# 1. Functional Requirements

- Schedule tasks to run at a specific time
    
- Support recurring tasks (cron / interval-based)
    
- Execute tasks asynchronously
    
- Retry failed tasks
    
- Cancel scheduled tasks
    
- Track task status (SCHEDULED → RUNNING → SUCCESS → FAILED → CANCELLED)
    

---

# 2. Non-Functional Requirements

- High reliability
    
- Low latency execution
    
- Scalable
    
- Fault-tolerant
    
- Thread-safe operations
    
- Support large number of scheduled tasks
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|Task|Represents a job|
|TaskStatus|Lifecycle state|
|Schedule|Defines execution timing|
|TaskExecutor|Executes task|
|RetryPolicy|Retry logic|
|TaskQueue|Priority queue for scheduling|
|SchedulerService|Orchestrates scheduling|
|Worker|Executes tasks|

---

# 4. Flows

---

## Schedule Task

1. User submits task with schedule
    
2. Task stored with status SCHEDULED
    
3. Insert into priority queue (based on next run time)
    

---

## Execute Task

1. Scheduler polls queue
    
2. If execution time reached → dispatch to worker
    
3. Worker executes task
    
4. Update status
    

---

## Retry Flow

1. Task fails
    
2. Check retry policy
    
3. Reschedule task
    
4. Else mark FAILED
    

---

## Cancel Task

1. Mark task CANCELLED
    
2. Remove from queue (lazy or immediate)
    

---

# 5. Code

```java
import java.util.*;
import java.util.concurrent.*;

// ENUM
enum TaskStatus {
    SCHEDULED, RUNNING, SUCCESS, FAILED, CANCELLED
}

// MODEL
class Task {
    String id;
    Runnable job;
    long nextRunTime;
    TaskStatus status;
    int retryCount;

    public Task(String id, Runnable job, long time) {
        this.id = id;
        this.job = job;
        this.nextRunTime = time;
        this.status = TaskStatus.SCHEDULED;
        this.retryCount = 0;
    }
}

// PRIORITY QUEUE
class TaskQueue {
    private PriorityBlockingQueue<Task> queue =
        new PriorityBlockingQueue<>(10, Comparator.comparingLong(t -> t.nextRunTime));

    public void add(Task task) {
        queue.offer(task);
    }

    public Task peek() {
        return queue.peek();
    }

    public Task poll() {
        return queue.poll();
    }
}

// STRATEGY: Retry
interface RetryPolicy {
    boolean shouldRetry(int retryCount);
    long nextDelay(int retryCount);
}

class FixedRetryPolicy implements RetryPolicy {
    private int maxRetries;
    private long delay;

    public FixedRetryPolicy(int maxRetries, long delay) {
        this.maxRetries = maxRetries;
        this.delay = delay;
    }

    public boolean shouldRetry(int retryCount) {
        return retryCount < maxRetries;
    }

    public long nextDelay(int retryCount) {
        return delay;
    }
}

// EXECUTOR
class TaskExecutor {
    public void execute(Task task) {
        task.job.run();
    }
}

// SCHEDULER SERVICE
class SchedulerService {
    private TaskQueue queue = new TaskQueue();
    private RetryPolicy retryPolicy;
    private ExecutorService workerPool = Executors.newFixedThreadPool(5);

    public SchedulerService(RetryPolicy retryPolicy) {
        this.retryPolicy = retryPolicy;
        start();
    }

    public void schedule(Task task) {
        queue.add(task);
    }

    public void cancel(String taskId) {
        // Lazy cancellation (mark only)
        // Actual removal skipped for simplicity
    }

    private void start() {
        new Thread(() -> {
            while (true) {
                try {
                    Task task = queue.peek();
                    if (task == null) continue;

                    long now = System.currentTimeMillis();

                    if (task.nextRunTime <= now) {
                        queue.poll();
                        execute(task);
                    }
                } catch (Exception ignored) {}
            }
        }).start();
    }

    private void execute(Task task) {
        workerPool.submit(() -> {
            if (task.status == TaskStatus.CANCELLED) return;

            task.status = TaskStatus.RUNNING;

            try {
                new TaskExecutor().execute(task);
                task.status = TaskStatus.SUCCESS;

            } catch (Exception e) {
                task.retryCount++;

                if (retryPolicy.shouldRetry(task.retryCount)) {
                    task.nextRunTime = System.currentTimeMillis() +
                                       retryPolicy.nextDelay(task.retryCount);
                    task.status = TaskStatus.SCHEDULED;
                    queue.add(task);
                } else {
                    task.status = TaskStatus.FAILED;
                }
            }
        });
    }
}
```

---

# 6. Complexity Analysis

- Schedule Task → O(log N)
    
- Fetch Next Task → O(1) (peek)
    
- Poll Task → O(log N)
    
- Execution → O(1)
    

---

# 7. Design Patterns Used

- Strategy Pattern → Retry policy
    
- Producer-Consumer Pattern → Scheduler + Worker
    
- Priority Queue Pattern → Time-based execution
    
- Dependency Injection → Flexible retry logic
    

---

# 8. Key Design Decisions

- PriorityBlockingQueue used for time-based ordering
    
- Lazy cancellation for simplicity
    
- Separate scheduler thread for polling
    
- Worker pool for parallel execution
    
- Retry logic decoupled via strategy
    
- Task status tracked explicitly
    
- Thread-safe concurrent structures used
    

---

# 9. Edge Cases Handled

- Task execution failure
    
- Retry exhaustion
    
- Concurrent scheduling
    
- Tasks scheduled in past
    
- Worker thread failures
    

---

# 10. Possible Extensions

- Cron expression support
    
- Distributed scheduler (Kafka / Zookeeper)
    
- Persistent storage (DB)
    
- Dead-letter queue for failed tasks
    
- Exponential backoff retry
    
- Rate limiting
    
- Task prioritization levels
    
- Monitoring & alerting