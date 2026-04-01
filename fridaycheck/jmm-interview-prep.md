# Java Memory Model (JMM) — Middle Java Interview Prep

## 1) Heap vs Stack

In Java, **Heap** is shared memory where objects are stored. **Stack** is per-thread memory that stores method call frames, local primitive values, and references to objects.  
This is important for concurrency: each thread has its own stack (private view of local variables), but threads can point to the same object in heap.  
So bugs usually happen not because local variables are shared, but because multiple threads read/write the same heap object without proper coordination.

---

## 2) Multithreading Problems

### Visibility
One thread updates a value, but another thread still sees the old value.  
Why? Because without synchronization rules, there is no guarantee when updates become visible across threads.

### Race conditions (short)
A race condition is when final result depends on timing/interleaving of threads.  
Example: two threads increment the same counter and one increment is lost.

### Why these happen
Because operations that look like one line of Java code are often multiple steps internally (read, modify, write), and because shared state is accessed concurrently without a clear ordering rule.

---

## 3) `volatile`

### What it guarantees
- **Visibility**: write to a volatile field by one thread is visible to other threads reading that field.
- **Ordering around that variable**: helps prevent some reordering problems in communication patterns.

### What it does NOT guarantee
- It does **not** make compound actions atomic (`count++`, check-then-act).
- It does **not** replace locking for multi-step invariants.

### Example where `volatile` is needed
```java
class Worker {
    private volatile boolean running = true;

    public void stop() { running = false; }

    public void run() {
        while (running) {
            // do work
        }
    }
}
```
Without `volatile`, the loop may not stop reliably.

### Example where `volatile` does not save you
```java
class Counter {
    private volatile int count = 0;

    public void increment() {
        count++; // NOT atomic
    }
}
```
Multiple threads can still lose updates.

---

## 4) `synchronized`

### What it does
`synchronized` uses an object monitor (intrinsic lock).  
When one thread enters a synchronized block on a given lock, others must wait for that lock.

### Guarantees
- **Atomicity** for the guarded critical section (only one thread at a time for same lock).
- **Visibility**: changes made before unlock become visible to thread that later locks the same monitor.

### Simple example
```java
class SafeCounter {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }

    public synchronized int get() {
        return count;
    }
}
```

---

## 5) Happens-before (simple explanation)

`happens-before` means: if operation A happens-before operation B, then B is guaranteed to see A's effects.

Most practical cases:
- **Volatile write -> volatile read** of same field.
- **Unlock (`synchronized` exit) -> later lock (`synchronized` enter)** on same monitor.
- **`Thread.start()`**: actions before start in parent happen-before actions in started thread.
- **Thread actions -> `Thread.join()` return** in waiting thread.

Why this matters: it gives a reliable mental model for when shared data is safely published/visible.

---

## 6) Ordering (reordering)

### What it is
JVM/compiler/runtime can reorder some operations as long as single-threaded behavior stays correct.  
In multithreading, such reordering can break communication logic if no synchronization is used.

### Simple broken logic example
```java
class Holder {
    int data = 0;
    boolean ready = false;

    void writer() {
        data = 42;
        ready = true;
    }

    void reader() {
        if (ready) {
            System.out.println(data); // can print 0 without proper synchronization
        }
    }
}
```
Without `volatile`/`synchronized`, reader may observe `ready == true` but stale `data`.

---

## 7) Typical Developer Mistakes

- Using `volatile` where a lock/atomic class is needed for compound updates.
- Thinking `count++` is atomic because it is one line of code.
- Sharing mutable state without clear ownership/synchronization strategy.
- Mixing multiple locks inconsistently and creating deadlocks or partial protection.
- Reading/writing shared state from different threads "because it worked locally".

Practical rule: if state is shared and mutable, define one synchronization strategy and apply it consistently.

---

## 8) Short Interview Summary (5-7 sentences)

The Java Memory Model defines how threads interact through shared memory and when updates become visible across threads. Stack is thread-local, while heap is shared, so most concurrency bugs happen on shared heap objects. The two key practical issues are visibility and race conditions: one thread may not see another thread's update, or updates may be lost due to non-atomic operations. `volatile` is useful for visibility and simple state flags, but it does not make compound operations like increment atomic. `synchronized` provides both mutual exclusion and visibility guarantees by locking a monitor. The happens-before rules explain when one thread is guaranteed to observe another thread's changes (`volatile`, `synchronized`, `start/join`). In real projects, correctness depends on choosing and consistently applying the right synchronization strategy for shared mutable state.

---

# Memory Management in Java — Interview Prep

## 1) How Objects Are Created in Java

When you write `new User()`, Java does the following:
- Allocates memory on the **Heap** for the object.
- Initializes fields to default values (0, false, null).
- Runs the constructor to set actual values.
- Returns a **reference** (like a pointer) that you store in a variable.

**Where the reference lives**: if it's a local variable, the reference is on the **Stack**; if it's a static field, it's in a special static area. The **object itself** always lives on the Heap.

**Practical point**: creating too many short-lived objects increases GC pressure, so in hot paths we try to reuse objects or use primitives instead of boxed types.

---

## 2) References and GC Roots

### What is a reference
A reference is a way to reach an object in the Heap. As long as at least one reference exists, the object is considered "alive".

### Types of references (brief overview)
- **Strong**: normal reference (`User u = new User()`). Prevents GC.
- **Soft**: for caches. Cleared when memory is low. Good for "recreatable" data.
- **Weak**: for canonical mappings (e.g., `WeakHashMap`). Cleared at next GC cycle.
- **Phantom**: for cleanup actions after object is already unreachable.

**For most code, you only need strong references.** Soft and weak are used in specific caching or resource management scenarios.

### GC Roots
GC Roots are starting points from which the Garbage Collector traces reachability. Examples:
- Local variables on thread stacks
- Static fields of loaded classes
- Active thread objects
- JNI references

**How GC decides to delete**: if no path exists from any GC Root to an object, that object is unreachable and can be collected.

---

## 3) How Garbage Collector Works

### Core idea: reachability
GC doesn't count references; it traces reachability from GC Roots. If an object cannot be reached by following references from any root, it's eligible for collection.

### What happens during collection
- **Stop-the-world pause**: most GC events pause application threads briefly (some modern collectors minimize this).
- **Mark**: trace reachable objects from roots.
- **Sweep/Compact**: reclaim unreachable memory, possibly defragmenting.

### Generations (simplified)
Java uses a **generational** approach based on the observation that "most objects die young".
- **Young Generation** (Eden + Survivor): new objects start here. Quick, frequent collections (Minor GC).
- **Old Generation** (Tenured): long-lived objects promoted here. Larger, less frequent collections (Major/Full GC).

**Practical takeaway**: objects that live long enough get promoted to Old Gen. If you accidentally keep references to short-lived objects, they get promoted and increase Old Gen pressure.

---

## 4) Types of GC (practical understanding)

### G1 GC (Garbage First)
- **Default** in modern Java.
- Divides heap into regions, collects regions with most garbage first.
- Targets predictable pause times (configurable `MaxGCPauseMillis`).
- **When to know about it**: it's what runs by default; tune region size and pause targets if you have large heaps or latency requirements.

### ZGC
- Designed for **very low pause times** (sub-millisecond target).
- Uses colored pointers and load barriers; mostly concurrent.
- **When it matters**: massive heaps (hundreds of GB) where even G1 pauses are too long.

**Interview level**: know that G1 is default and balances throughput vs latency; ZGC is the ultra-low-latency option for demanding applications.

---

## 5) Typical Memory Problems

### Memory leaks in Java
Even with GC, leaks happen when you **hold references longer than intended**.

Common patterns:
- **Static collections**: `static List<Object> cache = new ArrayList<>()` that grows forever.
- **Listeners/observers**: adding callbacks but never removing them.
- **Caches without eviction**: unbounded maps holding cached data.
- **ThreadLocal misuse**: values not cleaned up after thread pool reuse.
- **Long-lived objects holding references to large graphs**: one forgotten reference keeps entire object tree alive.

**Example of a leak**:
```java
public class OrderService {
    private static final List<Order> ALL_ORDERS = new ArrayList<>();
    
    public void process(Order order) {
        ALL_ORDERS.add(order); // grows forever, never cleaned
    }
}
```

---

## 6) Practical Advice

### How to avoid leaks
- Be careful with `static` collections; use bounded caches (Caffeine, Guava Cache) with expiration.
- Always unregister listeners/callbacks when components are destroyed.
- Clear large objects explicitly when done (set to null) if they have unusually long-lived containers.
- Use `try-with-resources` to ensure streams and connections close promptly.
- Monitor heap growth and GC frequency in production; investigate if Old Gen keeps growing.

### Where developers typically break things
- Caching "just in case" without limits or eviction.
- Holding references in long-lived objects "temporarily" that become permanent.
- Not understanding that thread pools reuse threads, so ThreadLocal values persist across tasks.

---

## 7) Short Interview Summary (Memory Management)

Objects are created on the Heap, while references live on the Stack or in static storage. Garbage Collector traces reachability from GC Roots (stacks, statics, active threads); unreachable objects are collected. Modern Java uses G1 GC by default, which balances throughput and pause times; ZGC offers even lower pauses for massive heaps. Memory leaks happen not because GC fails, but because the program accidentally keeps references alive—typically in static collections, unbounded caches, or forgotten listeners. To avoid leaks, use bounded data structures with expiration, properly manage listeners and resources, and monitor heap metrics in production.

