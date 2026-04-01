# Java Concurrency & Locks — Middle Java Interview Prep

Practical guide to multithreading for interviews without deep JVM internals.

---

## 1) Thread vs Runnable vs Callable

### Thread
- A `Thread` is the actual execution unit managed by the OS/JVM.
- Extending `Thread` class is generally discouraged (locks you into inheritance, harder to test).

### Runnable
- Represents a task to run, no return value.
- `void run()` — cannot throw checked exceptions, cannot return result.
- Preferred over extending Thread.

### Callable
- Like Runnable but **returns a value** and **can throw checked exceptions**.
- `V call() throws Exception`
- Used with `ExecutorService.submit()` which returns a `Future`.

### When to use what
- **Runnable**: fire-and-forget tasks, no result needed.
- **Callable**: need result back or exception handling.
- **ExecutorService + Callable/Runnable**: modern preferred approach (don't create raw Threads).

```java
// Runnable example
executor.execute(() -> System.out.println("Task done"));

// Callable example
Future<Integer> future = executor.submit(() -> {
    return 42; // returns value
});
```

---

## 2) Race Conditions

### What is a race condition
When the outcome depends on the **relative timing/interleaving** of threads. Multiple threads access shared data concurrently, and at least one modifies it.

### Simple bug example
```java
public class Counter {
    private int count = 0;
    
    public void increment() {
        count++; // Not atomic! Read -> modify -> write
    }
}

// Two threads call increment()
// Thread A reads 0, Thread B reads 0
// Both increment to 1, both write 1
// Expected 2, got 1 — race condition!
```

### Why they happen
The `count++` operation is actually 3 steps:
1. Read current value
2. Add 1
3. Write back

If threads interleave between these steps, updates are lost.

---

## 3) synchronized

### What it does
Creates a **mutual exclusion** block — only one thread can execute synchronized code on the same monitor at a time.

### What is a monitor
Every Java object has an intrinsic lock (monitor). When you enter `synchronized`, you acquire this lock; when you exit, you release it.

### Guarantees
- **Atomicity**: only one thread in the block for given lock
- **Visibility**: changes made before unlock are visible to next thread acquiring the lock

### Example usage
```java
public class SafeCounter {
    private int count = 0;
    private final Object lock = new Object();
    
    public void increment() {
        synchronized (lock) {
            count++;
        }
    }
    
    public int get() {
        synchronized (lock) {
            return count;
        }
    }
}
```

Synchronized methods use `this` as the lock; static synchronized uses the class object.

---

## 4) volatile

### What it guarantees
**Visibility**: when one thread writes to a volatile variable, other threads immediately see the new value.

### What it does NOT guarantee
**Atomicity**: compound operations (read-modify-write) are not atomic.

### Where it works
Simple flag communication between threads:
```java
public class Worker {
    private volatile boolean running = true;
    
    public void shutdown() {
        running = false;
    }
    
    public void work() {
        while (running) {
            // do work
        }
    }
}
```

### Where it fails
Counter increment — still has race condition:
```java
public class BadCounter {
    private volatile int count = 0;
    
    public void increment() {
        count++; // Still read-modify-write, not atomic!
    }
}
```

**Rule**: `volatile` is for visibility, not for compound operations. Use `synchronized` or `AtomicInteger` for counters.

---

## 5) ExecutorService

### Why it exists
Creating raw `Thread` objects is expensive and hard to manage. ExecutorService provides:
- Thread **pool** (reuse threads)
- Task **queuing**
- Lifecycle management
- Controlled **concurrency level**

### Thread pool idea
Instead of creating a new thread per task, you have a fixed pool of workers that pick up tasks from a queue.

### Example usage
```java
// Fixed pool of 4 threads
ExecutorService executor = Executors.newFixedThreadPool(4);

// Submit tasks
Future<Integer> result = executor.submit(() -> {
    return computeSomething();
});

// Get result (blocking)
Integer value = result.get();

// Shutdown gracefully
executor.shutdown();
```

### Common implementations
- `newFixedThreadPool(n)`: fixed size, good for CPU-bound work
- `newCachedThreadPool()`: grows as needed, risky under load
- `newSingleThreadExecutor()`: sequential execution

---

## 6) ReentrantLock

### What it is
An explicit lock object (`java.util.concurrent.locks.ReentrantLock`) that provides more flexibility than `synchronized`.

### Key features
- **lock() / unlock()**: explicit control
- **tryLock()**: attempt to acquire without blocking indefinitely
- **tryLock(timeout)**: wait with timeout
- **lockInterruptibly()**: can be interrupted while waiting
- **Multiple Condition variables**: can have different wait-sets

### Example
```java
private final ReentrantLock lock = new ReentrantLock();

public void doWork() {
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock(); // Must release in finally!
    }
}

public boolean tryDoWork() {
    if (lock.tryLock()) {
        try {
            // got lock, do work
            return true;
        } finally {
            lock.unlock();
        }
    }
    return false; // couldn't get lock
}
```

---

## 7) ReentrantLock vs synchronized

| Aspect | synchronized | ReentrantLock |
|--------|--------------|---------------|
| **Simplicity** | Built-in, automatic | Manual lock/unlock required |
| **Flexibility** | Limited | tryLock, timeouts, interruptible |
| **Conditions** | One implicit | Multiple Condition objects |
| **Performance** | Similar in modern JVM | Similar, slightly more overhead |
| **Deadlock risk** | Lower (automatic release) | Higher (must remember unlock) |

### When to use synchronized
- Simple locking needs
- When you don't need timeouts or try-lock
- When you want automatic release (less error-prone)

### When to use ReentrantLock
- Need tryLock (non-blocking attempt)
- Need timeout on lock acquisition
- Need lockInterruptibly
- Need multiple wait conditions
- Complex locking scenarios

**General rule**: Start with `synchronized`, move to `ReentrantLock` only when you need its extra features.

---

## 8) Multithreading Problems

### Deadlock
**What**: Two or more threads blocked forever, each waiting for resources held by the other.

**Example**:
```java
// Thread A: lock X, then tries to lock Y
// Thread B: lock Y, then tries to lock X
// Both hold one lock, wait for other — deadlock!

synchronized(resourceX) {
    synchronized(resourceY) { // Thread A waiting for Y held by B
        // work
    }
}
```

**Prevention**:
- Always acquire locks in the same order globally
- Use timeout-based tryLock()
- Keep locking sections small
- Avoid nested locks when possible

### Livelock (briefly)
Threads are not blocked but keep changing state in response to each other without making progress. Like two people trying to pass each other in a hallway and both stepping side to side forever.

### Starvation (briefly)
One thread is perpetually denied access to a resource because other threads always get it first. Can happen with unfair locks or improper priority settings.

---

## 9) Typical Developer Mistakes

### Forgot unlock()
```java
lock.lock();
// do work — exception here!
lock.unlock(); // Never reached!
```
**Fix**: Always use `try-finally`.

### Using volatile instead of synchronization
```java
private volatile int counter = 0;
public void increment() { counter++; } // Still broken!
```

### Creating too many threads
```java
for (int i = 0; i < 10000; i++) {
    new Thread(() -> doWork()).start(); // Exhausts resources!
}
```
**Fix**: Use thread pools.

### Wrong shared state handling
```java
// SimpleDateFormat is not thread-safe!
private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
// Multiple threads using sdf — corruption!
```
**Fix**: Use ThreadLocal or DateTimeFormatter (thread-safe).

### Locking on mutable field
```java
private Object lock = new Object();
public void changeLock() { lock = new Object(); } // Breaks synchronization!
```
**Fix**: Lock on `final` fields.

---

## 10) Practical Advice

### When to use synchronized
- Simple counter/flag protection
- When code is straightforward and fits in one method
- When you want automatic release (less error-prone)

### When to use ReentrantLock
- Non-blocking lock attempts needed
- Timeout requirements
- Multiple wait conditions
- Complex lock handover scenarios

### When to use ExecutorService
- Always prefer over raw Thread creation
- For any concurrent task execution
- For async operations with result handling (Future/CompletableFuture)

### Additional tips
- Prefer immutable objects where possible (no synchronization needed)
- Use concurrent collections (`ConcurrentHashMap`, `CopyOnWriteArrayList`) instead of synchronizing standard collections
- Keep critical sections as small as possible
- Always document thread-safety of your classes

---

## 11) Short Interview Summary (5-7 sentences)

Java provides multiple ways to handle concurrency: `Thread`/`Runnable`/`Callable` for task definition, but modern code should use `ExecutorService` with thread pools for efficiency. Race conditions occur when threads access shared mutable state without proper synchronization, leading to lost updates or corrupted data. Use `synchronized` for simple locking needs—it provides atomicity and visibility through object monitors. Use `volatile` only for visibility of single variables, not for compound operations like increment. `ReentrantLock` offers more control than synchronized with features like tryLock and timeouts, but requires careful unlock in finally blocks. Common pitfalls include deadlocks from inconsistent lock ordering, forgotten unlocks, and using volatile where atomicity is needed. For production code, prefer thread pools, concurrent collections, and immutable objects over low-level synchronization.

---

## 12) Typical Interview Questions with Answers

### Q1: What's the difference between `start()` and `run()`?
**A**: `start()` creates a new thread and executes `run()` on that thread. Calling `run()` directly just executes it as a normal method on the current thread—no new thread is created.

### Q2: Why should we prefer `ExecutorService` over creating `new Thread()`?
**A**: Thread creation is expensive. Thread pools reuse threads, control concurrency level, provide task queuing, and offer lifecycle management (shutdown, graceful termination).

### Q3: Can `volatile` replace `synchronized`?
**A**: No. `volatile` only guarantees visibility, not atomicity. It works for simple flags but fails for compound operations like `count++` which need atomic read-modify-write.

### Q4: How do you handle exceptions in threads?
**A**: Uncaught exceptions terminate the thread. Use `UncaughtExceptionHandler` on Thread or handle within Runnable/Callable. With ExecutorService, exceptions are captured in the Future and thrown when calling `get()`.

### Q5: What's a deadlock and how to prevent it?
**A**: Deadlock is when threads block forever waiting for each other's locks. Prevent by: acquiring locks in consistent global order, using tryLock with timeouts, keeping lock sections small, and avoiding nested locks.

### Q6: What's the difference between `wait()`/`notify()` and `ReentrantLock` with `Condition`?
**A**: Both allow threads to wait for conditions. `wait()`/`notify()` use object monitor and are harder to use correctly (must hold lock, can miss signals). `Condition` with ReentrantLock is more flexible—multiple conditions per lock, can interrupt, can check status.

### Q7: Is `ConcurrentHashMap` completely thread-safe?
**A**: Individual operations are thread-safe, but compound operations (check-then-put) are not atomic. Use methods like `putIfAbsent()`, `compute()`, or external synchronization for multi-step logic.

### Q8: What's thread starvation?
**A**: A thread is perpetually denied access to a resource because other threads always acquire it first. Can happen with unfair locks or improper thread priorities.

### Q9: What is `ThreadLocal` and when to use it?
**A**: `ThreadLocal` provides thread-local variables—each thread has its own independent copy. Useful for non-thread-safe objects (SimpleDateFormat), user context in web requests, or database connections per thread.

### Q10: How does `synchronized` guarantee visibility?
**A**: JMM defines that all changes made in a synchronized block are visible to any thread that subsequently acquires the same lock. Unlock happens-before subsequent lock on the same monitor.
