
Concurrency means making progress on multiple tasks in overlapping time. It improves responsiveness and throughput, but it also introduces shared-state risks.

---

## 1) Concurrency vs Parallelism

- **Concurrency**: many tasks are in progress (interleaved), even on one CPU core.
- **Parallelism**: many tasks run at the exact same time (multiple cores).

Interview line:
- "Concurrency is about structure and coordination; parallelism is about hardware execution."

---

## 2) Process vs Thread

- **Process**: isolated memory space, heavier context switch, stronger fault isolation.
- **Thread**: lightweight execution unit within a process, shares heap memory with sibling threads.

Why threads are tricky:
- Shared memory enables speed, but unsafe access causes race conditions.

---

## 3) Common Concurrency Problems

### Race condition

Two or more threads update/read shared state without proper coordination, causing nondeterministic bugs.

```java
class Counter {
    int value = 0;
    void increment() { value++; } // not atomic
}
```

`value++` is read-modify-write; interleavings can lose updates.

### Visibility issue

One thread updates a value, another thread may keep reading stale cached data if no memory visibility guarantee exists.

### Atomicity issue

An operation that should be "all-or-nothing" is observed partially by other threads.

---

## 4) Synchronization

Synchronization coordinates access to shared resources to ensure correctness.

Goals:
- **Mutual exclusion**: only one thread enters critical section at a time.
- **Ordering/visibility**: writes by one thread become visible to others predictably.

### `synchronized` (Java monitor lock)

```java
class Counter {
    private int value = 0;

    public synchronized void increment() {
        value++;
    }

    public synchronized int get() {
        return value;
    }
}
```

What it gives:
- Mutual exclusion on the monitor.
- Happens-before guarantees on lock acquire/release (visibility).

When to use:
- Simple critical sections on one object.

---

## 5) Thread Safety

A class is **thread-safe** if it behaves correctly when used by multiple threads without external coordination.

### Thread safety strategies

1. **Immutability** (best when possible)
   - Objects cannot change after creation.
2. **Thread confinement**
   - Data is used by only one thread (e.g., local variables).
3. **Synchronization/locks**
   - Protect shared mutable state.
4. **Atomic classes**
   - Use lock-free primitives for simple counters/flags.
5. **Concurrent collections**
   - `ConcurrentHashMap`, `BlockingQueue`, etc.

### Example: atomic counter

```java
import java.util.concurrent.atomic.AtomicInteger;

class SafeCounter {
    private final AtomicInteger value = new AtomicInteger(0);

    void increment() {
        value.incrementAndGet();
    }

    int get() {
        return value.get();
    }
}
```

---

## 6) Locks

## Mutex (mutual exclusion lock)

- Only one thread can hold the lock at a time.
- In Java, `synchronized` and `ReentrantLock` are mutex-style locks.

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

class Wallet {
    private int balance = 0;
    private final Lock lock = new ReentrantLock();

    void deposit(int amount) {
        lock.lock();
        try {
            balance += amount;
        } finally {
            lock.unlock();
        }
    }
}
```

Why `try/finally`:
- Guarantees unlock even if exception occurs.

### Read-Write Lock

- Multiple readers can proceed concurrently.
- Writer gets exclusive access.
- Useful when reads are frequent and writes are rare.

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

class ConfigStore {
    private String config = "v1";
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();

    String read() {
        rwLock.readLock().lock();
        try {
            return config;
        } finally {
            rwLock.readLock().unlock();
        }
    }

    void update(String newConfig) {
        rwLock.writeLock().lock();
        try {
            config = newConfig;
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

Trade-off:
- Better read throughput, but more overhead than a simple mutex.

---

## 7) Deadlocks

Deadlock happens when threads wait forever due to cyclic lock dependency.

### Classic example

```java
// Thread 1: lock(A) -> lock(B)
// Thread 2: lock(B) -> lock(A)
// Both can wait forever.
```

### Coffman conditions (all 4 required)

1. Mutual exclusion
2. Hold and wait
3. No preemption
4. Circular wait

Break any one condition to prevent deadlock.

---

## 8) Deadlock Prevention Techniques

### 1. Global lock ordering (most practical)

Always acquire locks in the same order system-wide.

```java
void transfer(Account from, Account to, int amount) {
    Account first = from.id() < to.id() ? from : to;
    Account second = from.id() < to.id() ? to : from;

    synchronized (first) {
        synchronized (second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}
```

### 2. Use timeouts (`tryLock`)

If lock is unavailable, back off/retry instead of waiting forever.

### 3. Keep critical sections small

- Hold locks for minimum time.
- Avoid I/O or network calls while holding locks.

### 4. Avoid nested locks where possible

- Prefer lock-free or single-lock designs if feasible.

### 5. Use higher-level concurrency utilities

- `BlockingQueue`, executors, concurrent collections, actor/event models.

---

## 9) Interview-Ready Patterns and Best Practices

- Protect **shared mutable state**, not methods blindly.
- Document lock ownership: "which lock guards which data".
- Prefer **immutable DTOs/value objects**.
- Use **composition** of small thread-safe components.
- Validate under load with stress tests and race detectors (where available).
- Design for **idempotency** in distributed workflows (retries can happen).

---

## 10) Quick Interview Q&A

### Q: Is `volatile` enough for thread safety?
- Only for visibility/order of single variable updates, not for compound operations like `count++`.

### Q: `synchronized` vs `ReentrantLock`?
- `synchronized` is simpler.
- `ReentrantLock` gives extra control (timeouts, fairness option, interruptible lock).

### Q: When to use Read-Write lock?
- Read-heavy workloads with infrequent writes and measurable contention.

### Q: Deadlock vs starvation?
- **Deadlock**: cyclic waiting, no progress.
- **Starvation**: a thread may wait indefinitely because others keep getting preference.

---

## Summary

- Concurrency improves throughput/responsiveness but adds correctness challenges.
- Synchronization ensures mutual exclusion + visibility for shared state.
- Thread safety can be achieved via immutability, confinement, locks, atomics, and concurrent collections.
- Mutex is general-purpose; read-write lock helps read-heavy paths.
- Deadlocks are preventable with lock ordering, timeouts, and small critical sections.
