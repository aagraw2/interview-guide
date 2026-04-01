## 1. What is a Distributed Scheduler?

A distributed scheduler executes scheduled tasks (cron jobs) across multiple servers in a coordinated way, ensuring tasks run exactly once even with multiple workers.

```
Single-server cron:
  Server runs task at 2:00 AM daily
  Problem: If server fails, task doesn't run

Distributed scheduler:
  Multiple servers, one executes task
  If one fails, another takes over
  Task runs exactly once
```

**Use cases:** Scheduled reports, data cleanup, batch processing, periodic sync, reminder emails.

---

## 2. Challenges

### 1. Exactly-once execution

```
Problem:
  3 servers, all run same cron job at 2:00 AM
  → Task executes 3 times (duplicate work)

Solution:
  Leader election or distributed locking
  → Only one server executes
```

### 2. Failure handling

```
Problem:
  Server starts task, crashes mid-execution
  → Task incomplete, no retry

Solution:
  Task state tracking
  → Detect failure, retry on another server
```

### 3. Task distribution

```
Problem:
  100 tasks, 10 servers
  → How to distribute evenly?

Solution:
  Work queue or consistent hashing
  → Balance load across servers
```

---

## 3. Approaches

### Leader election

One server is elected leader and executes all tasks.

```
3 servers:
  Server 1: Leader (executes tasks)
  Server 2: Follower (standby)
  Server 3: Follower (standby)

Leader fails:
  → Server 2 elected new leader
  → Takes over task execution

Implementation: ZooKeeper, etcd, Consul
```

**Pros:**
- Simple (one server executes)
- No duplicate execution

**Cons:**
- Single point of execution (not distributed)
- Leader can be bottleneck

### Distributed locking

Each server tries to acquire lock before executing task.

```
Task: daily_report at 2:00 AM

Server 1: Acquire lock "daily_report" → Success → Execute
Server 2: Acquire lock "daily_report" → Fail (locked) → Skip
Server 3: Acquire lock "daily_report" → Fail (locked) → Skip

Only Server 1 executes
```

**Pros:**
- True distributed execution
- No single point of failure

**Cons:**
- Requires distributed lock service
- Lock expiry issues

### Work queue

Tasks pushed to queue, workers pull and execute.

```
Scheduler → Queue (Kafka, RabbitMQ, SQS)
Workers pull tasks from queue

Task: daily_report at 2:00 AM
  → Scheduler pushes to queue at 2:00 AM
  → Worker 1 pulls and executes
  → Other workers see queue empty

Consumer groups ensure exactly-once
```

**Pros:**
- Scalable (add more workers)
- Load balancing (queue distributes)

**Cons:**
- Requires message queue
- More complex

---

## 4. Implementation Examples

### Celery Beat (Python)

```python
from celery import Celery
from celery.schedules import crontab

app = Celery('tasks', broker='redis://localhost:6379')

# Define task
@app.task
def daily_report():
    generate_report()
    send_email()

# Schedule task
app.conf.beat_schedule = {
    'daily-report': {
        'task': 'tasks.daily_report',
        'schedule': crontab(hour=2, minute=0),  # 2:00 AM daily
    },
}

# Run beat scheduler (one instance only)
# celery -A tasks beat

# Run workers (multiple instances)
# celery -A tasks worker
```

**How it works:**
- Celery Beat: Scheduler (one instance)
- Celery Workers: Execute tasks (multiple instances)
- Redis/RabbitMQ: Message broker

### Quartz (Java)

```java
import org.quartz.*;
import org.quartz.impl.StdSchedulerFactory;

public class DailyReportJob implements Job {
    public void execute(JobExecutionContext context) {
        generateReport();
        sendEmail();
    }
}

// Schedule job
JobDetail job = JobBuilder.newJob(DailyReportJob.class)
    .withIdentity("dailyReport", "reports")
    .build();

Trigger trigger = TriggerBuilder.newTrigger()
    .withIdentity("dailyReportTrigger", "reports")
    .withSchedule(CronScheduleBuilder.dailyAtHourAndMinute(2, 0))
    .build();

Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
scheduler.scheduleJob(job, trigger);
scheduler.start();
```

**Clustering:**
- Quartz with JDBC JobStore
- Multiple instances share database
- Distributed locking via database

### Kubernetes CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
spec:
  schedule: "0 2 * * *"  # 2:00 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: report-generator
            image: my-app:latest
            command: ["python", "generate_report.py"]
          restartPolicy: OnFailure
```

**How it works:**
- Kubernetes creates Job at scheduled time
- Job creates Pod to execute task
- Pod runs to completion
- Automatic retry on failure

---

## 5. Distributed Locking Pattern

```python
import redis
import time

redis_client = redis.Redis()

def execute_scheduled_task(task_name, task_func):
    lock_key = f"lock:{task_name}"
    lock_value = f"{socket.gethostname()}:{os.getpid()}"
    lock_ttl = 300  # 5 minutes
    
    # Try to acquire lock
    acquired = redis_client.set(
        lock_key,
        lock_value,
        nx=True,  # Only if not exists
        ex=lock_ttl  # Expire after 5 minutes
    )
    
    if acquired:
        try:
            # Execute task
            task_func()
        finally:
            # Release lock (check ownership)
            script = """
            if redis.call("get", KEYS[1]) == ARGV[1] then
                return redis.call("del", KEYS[1])
            else
                return 0
            end
            """
            redis_client.eval(script, 1, lock_key, lock_value)
    else:
        print(f"Task {task_name} already running on another server")

# Schedule with APScheduler
from apscheduler.schedulers.blocking import BlockingScheduler

scheduler = BlockingScheduler()

@scheduler.scheduled_job('cron', hour=2, minute=0)
def daily_report_job():
    execute_scheduled_task('daily_report', generate_report)

scheduler.start()
```

---

## 6. Task State Tracking

Track task execution state to handle failures.

```python
# Task states
PENDING = 'pending'
RUNNING = 'running'
COMPLETED = 'completed'
FAILED = 'failed'

def execute_task_with_tracking(task_id, task_func):
    # Mark as running
    db.update_task_state(task_id, RUNNING, started_at=now())
    
    try:
        # Execute task
        result = task_func()
        
        # Mark as completed
        db.update_task_state(task_id, COMPLETED, completed_at=now(), result=result)
    except Exception as e:
        # Mark as failed
        db.update_task_state(task_id, FAILED, failed_at=now(), error=str(e))
        raise

# Detect stuck tasks (running > 1 hour)
def detect_stuck_tasks():
    stuck_tasks = db.query("""
        SELECT * FROM tasks
        WHERE state = 'running'
        AND started_at < NOW() - INTERVAL '1 hour'
    """)
    
    for task in stuck_tasks:
        # Mark as failed
        db.update_task_state(task.id, FAILED, error='Timeout')
        
        # Retry on another server
        schedule_retry(task)
```

---

## 7. Idempotency

Tasks must be idempotent (safe to retry).

```python
# Non-idempotent (bad)
def send_daily_email():
    users = get_all_users()
    for user in users:
        send_email(user)  # Sends duplicate if retried

# Idempotent (good)
def send_daily_email():
    # Check if already sent today
    if email_sent_today():
        return
    
    users = get_all_users()
    for user in users:
        send_email(user)
    
    # Mark as sent
    mark_email_sent_today()

# Or use idempotency key
def send_daily_email():
    idempotency_key = f"daily_email_{date.today()}"
    
    if redis.exists(idempotency_key):
        return  # Already sent
    
    users = get_all_users()
    for user in users:
        send_email(user)
    
    # Store idempotency key (expires in 2 days)
    redis.setex(idempotency_key, 172800, '1')
```

---

## 8. Monitoring

### Metrics

```
Task execution count:
  daily_report: 1 execution/day
  Alert if 0 (task didn't run) or > 1 (duplicate)

Task duration:
  daily_report: avg 5 minutes
  Alert if > 30 minutes (stuck or slow)

Task failure rate:
  daily_report: 0% failures
  Alert if > 5%

Last execution time:
  daily_report: 2024-01-01 02:00:00
  Alert if > 25 hours ago (missed execution)
```

### Alerting

```
Alert: daily_report didn't run
  - Check scheduler is running
  - Check lock service (Redis/ZooKeeper)
  - Check task queue
  - Check worker health

Alert: daily_report ran twice
  - Check distributed locking
  - Check for clock skew
  - Check for duplicate schedulers
```

---

## 9. Common Interview Questions + Answers

### Q: How do you ensure a scheduled task runs exactly once in a distributed system?

> "Use distributed locking with Redis or ZooKeeper. Before executing the task, each server tries to acquire a lock with the task name. Only one server succeeds and executes the task. The lock has a TTL to prevent deadlocks if the server crashes. After execution, the server releases the lock. Alternatively, use leader election where only the leader executes tasks, or use a work queue where the scheduler pushes tasks and workers pull them with consumer groups ensuring exactly-once delivery."

### Q: What happens if a server crashes while executing a scheduled task?

> "Track task state in a database with states like PENDING, RUNNING, COMPLETED, FAILED. When a task starts, mark it as RUNNING with a timestamp. Have a separate monitor process that detects stuck tasks — if a task is RUNNING for more than the expected duration (e.g., 1 hour), mark it as FAILED and schedule a retry. The lock TTL also ensures the lock is released automatically, allowing another server to retry. Make tasks idempotent so retries are safe."

### Q: How would you implement a distributed scheduler with Kubernetes?

> "Use Kubernetes CronJobs which create Jobs at scheduled times. Each Job creates a Pod that executes the task and terminates. Kubernetes ensures only one Job runs at a time for each CronJob. Set concurrencyPolicy to Forbid to prevent overlapping executions. For more complex scheduling, use a dedicated scheduler like Celery Beat running as a single-replica Deployment, with Celery workers as a multi-replica Deployment pulling tasks from Redis or RabbitMQ."

### Q: How do you handle time zones in distributed schedulers?

> "Store all schedules in UTC and convert to local time zones when needed. For example, 'send email at 9 AM user local time' becomes multiple scheduled tasks — one for each time zone. Use libraries like pytz or moment-timezone for conversions. For Kubernetes CronJobs, the schedule is in the cluster's time zone (usually UTC). For user-facing schedules, store the user's time zone and calculate the UTC time for execution."

---

## 10. Quick Reference

```
What is a distributed scheduler?
  Execute scheduled tasks across multiple servers
  Ensure exactly-once execution
  Handle failures automatically

Challenges:
  1. Exactly-once execution (no duplicates)
  2. Failure handling (retry on crash)
  3. Task distribution (load balancing)

Approaches:
  Leader election: One leader executes all tasks
  Distributed locking: Lock before execution
  Work queue: Scheduler pushes, workers pull

Implementation:
  Celery Beat: Python, Redis/RabbitMQ
  Quartz: Java, JDBC clustering
  Kubernetes CronJob: Container-based

Distributed locking:
  Redis: SET lock_key value NX EX ttl
  Only one server acquires lock
  Execute task, release lock

Task state tracking:
  PENDING → RUNNING → COMPLETED/FAILED
  Detect stuck tasks (RUNNING > 1 hour)
  Retry failed tasks

Idempotency:
  Tasks must be safe to retry
  Use idempotency keys
  Check if already executed today

Monitoring:
  Execution count (alert if 0 or > 1)
  Duration (alert if too long)
  Failure rate (alert if > 5%)
  Last execution time (alert if missed)

Best practices:
  - Use distributed locking or leader election
  - Track task state in database
  - Make tasks idempotent
  - Set appropriate lock TTL
  - Monitor execution metrics
  - Handle time zones properly (use UTC)
```
