# Core Java — Questions 1–14

> Formerly `questions-0-20.md` — SQL/JDBC topics moved to `database-questions-15-29.md`.

## Оглавление

1. [Tell me please about HashMap? How does it work under the hood? What restrictions do we have for keys and values?](#q1)
2. [What about HashSet?](#q2)
3. [What about TreeMap? What additional stuff should be provided for it?](#q3)
4. [What is the difference between Terminal and Intermediate operations in Streams?](#q4)
5. [What is the difference between map() and flatMap()?](#q5)
6. [What are parallel streams? Which thread pool is used there?](#q6)
7. [How could we create a thread in Java?](#q7)
8. [What is ExecutorService? What implementations of this interface do you know?](#q8)
9. [What is the Future interface? What is the difference between our own implementation of this interface and CompletableFuture?](#q9)
10. [What is the difference between Concurrent and Synchronized collections?](#q10)
11. [What are Atomics?](#q11)
12. [What is Monitor?](#q12)
13. [What are the segments of Java Memory Model?](#q13)
14. [What is GC? What types of GC do you know? What GC is used by default?](#q14)

---

<a id="q1"></a>

## 1. Tell me please about HashMap? How does it work under the hood? What restrictions do we have for keys and values?

**Definition**  
`HashMap<K,V>` is an unordered key-value structure optimized for fast lookup (`O(1)` average for `put/get/remove`).

**Internal structure**  
- Backed by `Node<K,V>[] table` (array of buckets, capacity is power of two).  
- Entry object is `Node` (`hash`, `key`, `value`, `next`).  
- In Java 8+, long collision chains can become `TreeNode` (red-black tree).

**How it works**  
- `put`: calculate hash, index by `(n - 1) & hash`, insert into empty bucket or resolve collision in list/tree.  
- `get`: same hash/index, then compare `hash` + `equals()` in bucket.  
- `remove`: locate bucket, unlink node (or remove from tree node structure).

**Hashing and key equality**  
- `hashCode()` determines bucket; `equals()` confirms exact key match.  
- Rule: if `a.equals(b) == true`, then `a.hashCode() == b.hashCode()` must also be true.

**Collision handling (Java 8+)**  
- Small collision group: linked list.  
- If bucket size grows to `>= 8` and capacity is `>= 64`, list becomes red-black tree (better worst-case lookup).  
- If shrinks (`<= 6`), can convert back to list.

**Performance**  

| Operation | Average | Worst case |
|---|---|---|
| `put/get/remove` | `O(1)` | `O(log n)` (tree bucket) / historically `O(n)` |
| `containsValue` | `O(n)` | `O(n)` |

**Resize / rehash**  
- Default load factor: `0.75`.  
- When `size > capacity * loadFactor`, capacity doubles and entries are redistributed.

**Restrictions**  
- One `null` key is allowed; multiple `null` values are allowed.  
- Keys should be effectively immutable while inside map.  
- Broken `equals/hashCode` causes “missing” entries, duplicates, and failed lookups.

**Multithreading**  
- `HashMap` is not thread-safe.  
- Concurrent writes can corrupt state (classic interview point: resize issues in old JDKs).  
- For concurrency, prefer `ConcurrentHashMap`; `Collections.synchronizedMap` is coarse-grained and slower under contention.

**Important internals**  
- `Node` = regular bucket element.  
- `TreeNode` = red-black tree node in high-collision bucket.  
- `modCount` tracks structural changes; iterators are fail-fast (`ConcurrentModificationException`).

**Code example**
```java
Map<String, Integer> map = new HashMap<>();
map.put("Alice", 10);
map.put("Bob", 20);
int v = map.get("Alice"); // 10
```

**Bad example (mutable key)**
```java
Map<List<String>, String> m = new HashMap<>();
List<String> key = new ArrayList<>(List.of("x"));
m.put(key, "value");
key.add("y"); // key hash/equality changed -> lookup may fail
```

**Interview pitfalls**  
- Overriding `equals()` without `hashCode()`.  
- Assuming iteration order is stable.  
- Using `HashMap` in concurrent writes.

**Short speaking summary**  
`HashMap` is an array of buckets with hash-based indexing and collision resolution via list/tree. Average operations are `O(1)`, with Java 8 improving worst-case by treeifying heavy buckets. It allows one `null` key and many `null` values. Correct `equals/hashCode` is critical, and mutable keys are dangerous. It is not thread-safe; use `ConcurrentHashMap` for multithreaded access.

<a id="q2"></a>

## 2. What about HashSet?

**Definition**  
`HashSet<E>` stores unique elements with no guaranteed order.

**Internal structure**  
- Internally backed by `HashMap<E, Object>`.  
- Each set element is a map key; value is a shared dummy object (`PRESENT`).

**How it works**  
- `add(e)` -> `map.put(e, PRESENT)`  
- `contains(e)` -> `map.containsKey(e)`  
- `remove(e)` -> `map.remove(e)`

So collision handling, resize policy, and performance are inherited from `HashMap`.

**Performance**  

| Operation | Average | Worst case |
|---|---|---|
| `add/contains/remove` | `O(1)` | `O(log n)` in treeified bucket |
| Iteration | `O(n + capacity)` | `O(n + capacity)` |

**Restrictions**  
- Allows one `null` element.  
- Uniqueness depends on `equals/hashCode`.  
- Mutable elements are dangerous for the same reason as mutable map keys.

**Concurrency**  
- Not thread-safe.  
- Alternatives: `Collections.synchronizedSet(...)` or `ConcurrentHashMap.newKeySet()` for high concurrency.

**Related sets (important comparison)**  

| Set type | Ordering | Typical complexity |
|---|---|---|
| `HashSet` | no guaranteed order | `O(1)` average |
| `LinkedHashSet` | insertion order | `O(1)` average |
| `TreeSet` | sorted order | `O(log n)` |

**Code example**
```java
Set<String> set = new HashSet<>();
set.add("java");
set.add("java"); // false, duplicate ignored
boolean ok = set.contains("java"); // true
```

**Short speaking summary**  
`HashSet` is a thin wrapper around `HashMap`: element becomes key, value is a dummy marker. Because of this, it has the same hashing behavior, collision handling, and Java 8 treeification improvements. It offers fast membership checks and uniqueness, but no order guarantees and no thread safety.

<a id="q3"></a>

## 3. What about TreeMap? What additional stuff should be provided for it?

**Definition**  
`TreeMap<K,V>` is a sorted map implemented as a red-black tree.

**What additional stuff is required?**  
You must provide key ordering via one of:
- keys implementing `Comparable`, or  
- custom `Comparator` in constructor.

Without valid ordering, operations fail (`ClassCastException`).

**Internal structure**  
- No buckets/arrays.  
- Each entry is a tree node (`Entry`) with `left/right/parent/color`.

**How it works**  
- `put/get/remove` perform binary-search-tree navigation + red-black balancing.  
- Key comparison (`compareTo` / `Comparator.compare`) decides placement and equality (`compare == 0` means same key for map semantics).

**Red-black tree idea (brief)**  
- Keeps height near `O(log n)` using rotations and recoloring.  
- Prevents degenerate linear tree in typical usage.

**Performance**  

| Operation | Complexity |
|---|---|
| `put/get/remove` | `O(log n)` |
| `firstKey/lastKey/floorKey/ceilingKey` | `O(log n)` |
| Ordered iteration | `O(n)` |

**Restrictions**  
- `null` keys are not supported in normal usage.  
- `null` values are allowed.  
- Comparator should be consistent with `equals`; otherwise behavior is logically surprising.

**Concurrency**  
- Not thread-safe.  
- Concurrent sorted alternative: `ConcurrentSkipListMap`.

**TreeMap vs HashMap (core interview contrast)**  
- `HashMap`: faster average lookup, no order.  
- `TreeMap`: slower (`log n`) but always sorted and supports range queries (`subMap`, `headMap`, `tailMap`).

**Code example**
```java
NavigableMap<Integer, String> tm = new TreeMap<>();
tm.put(10, "x");
tm.put(2, "y");
System.out.println(tm.firstKey()); // 2
```

**Bad comparator example**
```java
TreeMap<String, Integer> bad = new TreeMap<>((a, b) -> 0);
bad.put("A", 1);
bad.put("B", 2); // overwrites because comparator says keys are equal
```

**Short speaking summary**  
`TreeMap` is a red-black-tree map with sorted keys and `O(log n)` operations. To use it, keys must be comparable or you provide a comparator. It gives ordered iteration and range-navigation APIs, unlike `HashMap`. It is not thread-safe and usually does not allow `null` keys.

<a id="q4"></a>

## 4. What is the difference between Terminal and Intermediate operations in Streams?

### Question Restatement
In Stream API, what is the practical and internal difference between **intermediate** and **terminal** operations, and why does this matter for performance and behavior?

### 1. Definition
- **Intermediate operations** transform a stream into another stream (`map`, `filter`, `sorted`, `distinct`, `peek`, `limit`, `skip`).
- **Terminal operations** finish the pipeline and produce a result or side effect (`collect`, `forEach`, `reduce`, `count`, `findFirst`, `anyMatch`).

### 2. Core Difference / Idea
- Intermediate = **lazy pipeline description**.
- Terminal = **execution trigger**.
- Without terminal operation, no traversal happens.

### 3. How It Works Internally
- Stream builds a chain of stages (`AbstractPipeline`).
- Source is usually a collection/array via **`Spliterator`**.
- Intermediate operations register logic (stateless/stateful), but do not pull elements immediately.
- Terminal operation starts traversal: elements flow through all stages.
- **Stateless intermediate ops**: `map`, `filter` (no full-history memory).
- **Stateful intermediate ops**: `sorted`, `distinct`, sometimes `limit/skip` in parallel (may need buffering/coordination).

### 4. Execution Model
- Computation starts only at terminal stage.
- Processing is mostly **vertical (per-element through full pipeline)**, not horizontal (whole stage then next), which helps short-circuiting.
- Short-circuit terminal ops (`anyMatch`, `findFirst`, `findAny`, `noneMatch`) can stop traversal early.
- Short-circuit intermediate ops (`limit`) can reduce work in downstream stages.

### 5. Performance

| Aspect | Notes |
|---|---|
| Time complexity | Depends on operations; `filter/map` are typically `O(n)` pass-through, `sorted` is `O(n log n)`, `distinct` about `O(n)` average with hashing |
| Memory | Stateless ops low overhead; stateful ops may allocate buffers/sets |
| Boxing cost | `Stream<Integer>` has boxing/unboxing; prefer `IntStream/LongStream/DoubleStream` (`mapToInt`) for numeric hot paths |
| Inefficiency cases | Tiny collections, heavy stateful operations, accidental repeated stream creation |

### 6. Multithreading (IMPORTANT)
- Sequential stream: one thread, deterministic stage order.
- Parallel stream: stages run over partitions; semantics depend on ordered vs unordered source.
- Side effects in intermediate ops are dangerous (race conditions, visibility issues).
- Streams are designed for **functional-style** transformations, not shared mutable state.

### 7. Code Examples
**Simple**
```java
List<String> names = List.of("Ann", "Bob", "Chris");
long cnt = names.stream()
        .filter(n -> n.length() > 3) // intermediate
        .count();                    // terminal (triggers execution)
```

**Tricky (no terminal => no work)**
```java
Stream<String> s = List.of("a", "b").stream()
        .peek(System.out::println); // nothing printed yet
// no terminal operation -> pipeline is never executed
```

**Bad**
```java
List<Integer> out = new ArrayList<>();
list.stream().forEach(out::add); // side-effect style, bad especially with parallel
```

### 8. Common Pitfalls
- Forgetting terminal operation.
- Expecting `peek` to always run.
- Reusing a stream after terminal operation (`IllegalStateException`).
- Using stateful heavy ops (`sorted/distinct`) without understanding cost.
- Using side effects instead of collectors.

### 9. Interview Deep Questions
- Why are streams lazy?
- What happens if terminal operation is absent?
- Difference between internal iteration and external iteration (`for` loop)?
- Stateful vs stateless operations with examples?
- Why stream is single-use?

### 10. Summary (Short speaking answer)
Intermediate operations build a lazy pipeline, terminal operations execute it. Internally Stream API stores stages and starts processing only when a terminal operation is called. Thanks to vertical processing and short-circuit operators, streams can avoid unnecessary work. Stateful operations like `sorted` and `distinct` are more expensive than simple `map/filter`. Streams are single-use and should avoid shared mutable side effects.

<a id="q5"></a>

## 5. What is the difference between map() and flatMap()?

### Question Restatement
What exactly is the difference between `map` and `flatMap`, both conceptually and in resulting stream shape (`Stream<T>` vs `Stream<Stream<T>>`)?

### 1. Definition
- `map(f)` applies a one-to-one transformation: element `T` -> `R`, result `Stream<R>`.
- `flatMap(f)` applies one-to-many (or zero/one/many) where function returns a stream-like container, then flattens nested streams into one stream.

### 2. Core Difference / Idea
- `map` = **transform each element**.
- `flatMap` = **transform + flatten nested structure**.
- Classic result:
  - `map`: `Stream<List<String>>`
  - `flatMap`: `Stream<String>`

### 3. How It Works Internally
- Both are intermediate and lazy.
- `map` wraps mapper stage: each source element pushes one mapped element.
- `flatMap` mapper returns inner streams; runtime concatenates their elements into downstream sink.
- Spliterator still controls source traversal; `flatMap` adds nested traversal over each produced inner stream.

### 4. Execution Model
- Nothing runs until terminal op.
- For each element:
  - `map`: apply mapper once.
  - `flatMap`: open inner stream and drain it before moving on (sequential semantics).
- Short-circuit terminal ops can stop early even with `flatMap`, but nested structure may affect how much was already traversed.

### 5. Performance

| Operation | Cost characteristics |
|---|---|
| `map` | Usually lower overhead, one function call per element |
| `flatMap` | Higher overhead: creating/draining inner streams; can be expensive with many tiny streams |
| Primitive variants | Prefer `flatMapToInt`, `mapToInt` etc. to avoid boxing |

### 6. Multithreading (IMPORTANT)
- In parallel streams, both can execute on multiple partitions.
- `flatMap` complexity rises because merging many inner streams may reduce parallel efficiency.
- Side effects in mapper/inner streams are unsafe and can become race-prone.

### 7. Code Examples
**Simple map**
```java
List<String> names = List.of("ann", "bob");
List<Integer> lengths = names.stream()
        .map(String::length)
        .toList();
```

**Nested data: map vs flatMap**
```java
List<List<String>> nested = List.of(
        List.of("a", "b"),
        List.of("c")
);

List<Stream<String>> wrongShape = nested.stream()
        .map(List::stream) // Stream<Stream<String>>
        .toList();

List<String> flat = nested.stream()
        .flatMap(List::stream) // Stream<String>
        .toList();
```

**Optional analogy**
```java
Optional<String> r1 = Optional.of("42")
        .map(v -> Optional.of("x" + v))      // Optional<Optional<String>>
        .orElse(Optional.empty());

Optional<String> r2 = Optional.of("42")
        .flatMap(v -> Optional.of("x" + v)); // Optional<String>
```

**Bad**
```java
stream.flatMap(x -> null); // NPE risk / invalid mapper contract
```

### 8. Common Pitfalls
- Using `map` and ending with nested streams/collections unintentionally.
- Returning `null` from `flatMap` mapper.
- Overusing `flatMap` for simple one-to-one mapping.
- Ignoring boxing cost in numeric flows.

### 9. Interview Deep Questions
- Why does `map(List::stream)` not flatten?
- Difference between `flatMap` and `mapMulti` (Java 16+)?
- How does `flatMap` behave with empty inner streams?
- Optional `map` vs `flatMap` analogy?

### 10. Summary (Short speaking answer)
`map` performs element-wise transformation and keeps one output per input. `flatMap` is used when each input may produce multiple outputs wrapped in a stream-like container, and then flattens them into a single stream. So `map` can produce nested structures, while `flatMap` removes one nesting level. `flatMap` is more powerful but usually more expensive due to inner stream creation and merging.

<a id="q6"></a>

## 6. What are parallel streams? Which thread pool is used there?

### Question Restatement
What are parallel streams, how are they executed internally, which thread pool is used, and when should we avoid them?

### 1. Definition
Parallel stream is a stream pipeline executed over multiple threads, typically by splitting source data and processing chunks concurrently.

### 2. Core Difference / Idea
- Sequential stream: one thread, predictable simple execution.
- Parallel stream: partition + concurrent processing + merge.
- Goal: improve throughput for CPU-bound, sufficiently large workloads.

### 3. How It Works Internally
- Source provides a **`Spliterator`**.
- Spliterator repeatedly `trySplit()` into subparts.
- Tasks are submitted to Fork/Join framework.
- Each worker processes chunk pipeline locally, then combines partial results (`reduce`/collector combiner).
- Efficiency depends heavily on split quality and merge cost.

### 4. Execution Model
- Starts only on terminal operation.
- Pipeline still lazy before terminal.
- For reductions, framework does map/filter work per chunk and then combines partial states.
- Short-circuit ops (`findAny`, `anyMatch`) may complete early, but ordering constraints (e.g., `findFirst`) can reduce parallel gains.

### 5. Performance

| Scenario | Parallel streams |
|---|---|
| Large CPU-bound pure operations | Often beneficial |
| Small collections | Usually slower (split/scheduling overhead) |
| Heavy synchronization/shared state | Often worse |
| I/O-bound/blocking tasks | Dangerous / poor fit |

Complexity stays algorithm-dependent, but constant factors (task management, splitting, merging) are higher.

### 6. Multithreading (IMPORTANT)
- Default pool: **`ForkJoinPool.commonPool()`**.
- Uses **work-stealing**: idle worker steals tasks from busy worker queues.
- Not good for blocking I/O in common pool: blocked workers can starve unrelated parallel tasks.
- Shared mutable state in lambdas leads to races and broken results.
- Prefer associative/stateless reductions (`sum`, `groupingByConcurrent` where appropriate).

**Parallel stream vs ExecutorService**
- Parallel streams: data-parallel pipeline abstraction with automatic splitting/merging.
- `ExecutorService`: explicit task orchestration/control (queueing, lifecycle, custom pool sizing).
- Use `ExecutorService` when you need strict control, blocking workloads, custom rejection policies, or request-scoped isolation.

### 7. Code Examples
**Simple**
```java
long sum = LongStream.range(1, 10_000_000)
        .parallel()
        .sum();
```

**Tricky (ordering)**
```java
List<Integer> data = IntStream.range(0, 20).boxed().toList();
data.parallelStream().forEach(System.out::println); // non-deterministic order
data.parallelStream().forEachOrdered(System.out::println); // ordered, slower
```

**Bad (shared mutable state)**
```java
List<Integer> acc = new ArrayList<>();
IntStream.range(0, 1000).parallel().forEach(acc::add); // race condition
```

### 8. Common Pitfalls
- Parallelizing small workloads.
- Using blocking I/O (HTTP/DB) in common pool tasks.
- Side effects in lambdas.
- Assuming encounter order with `forEach`.
- Using non-associative reduction (can produce inconsistent results).

### 9. Interview Deep Questions
- Which pool is used by default? (`ForkJoinPool.commonPool()`).
- What is work-stealing?
- Why can parallel be slower than sequential?
- Why `findFirst` may be slower in parallel than `findAny`?
- Why blocking calls are dangerous in common pool?
- What role does Spliterator play in parallelism quality?

### 10. Summary (Short speaking answer)
Parallel streams run stream pipelines concurrently by splitting source data with `Spliterator` and executing chunks in `ForkJoinPool.commonPool()`. Workers use work-stealing to balance load. This can speed up large CPU-bound, side-effect-free computations, but overhead can make small tasks slower. Parallel streams are risky with blocking I/O or shared mutable state. For explicit thread and lifecycle control, use `ExecutorService` instead.

<a id="q7"></a>

## 7. How could we create a thread in Java?

### Question Restatement
What are all practical ways to create and run concurrent work in Java, and which approach is recommended in production code?

### 1. Definition
A thread is the smallest unit of execution managed by the JVM/OS scheduler. In Java, you can run code concurrently by creating threads directly or, more commonly, by submitting tasks to thread pools.

### 2. Core Idea
Problem solved: run tasks concurrently without blocking the main flow, improve responsiveness, and utilize CPU resources.  
Modern idea: separate **task definition** from **thread management**.

### 3. How It Works Internally
- JVM thread maps to an OS thread (platform threads in classic model).
- Lifecycle states (simplified interview view):
  - `NEW` -> created, not started
  - `RUNNABLE` -> eligible/running on CPU
  - `BLOCKED` -> waiting for monitor lock
  - `WAITING` / `TIMED_WAITING` -> waiting on `wait/join/sleep/park`
  - `TERMINATED` -> completed
- Scheduler time-slices runnable threads; actual order is non-deterministic.
- `start()` creates a new call stack and invokes `run()` on new thread.
- Calling `run()` directly does **not** create a new thread; it runs synchronously on current thread.

### 4. Ways to Use / Implement
1. **Extend `Thread`**
   - Pros: simple for demos.
   - Cons: cannot extend another class, mixes task with thread object, poor testability/reuse.
   - Generally discouraged in real systems.
2. **Implement `Runnable`**
   - Pros: task-only abstraction, reusable, no return value.
   - Best for fire-and-forget logic.
3. **Implement `Callable<V>`**
   - Like Runnable but returns value and can throw checked exceptions.
   - Used with executors and `Future`.
4. **Use `ExecutorService` (preferred)**
   - Submit `Runnable/Callable`; pool handles reuse, queueing, throttling, lifecycle.

**Runnable vs Callable**
- `Runnable#run()` -> `void`, no checked exception.
- `Callable#call()` -> returns `V`, can throw `Exception`.

**Daemon threads**
- Daemon thread does not keep JVM alive.
- JVM exits when only daemon threads remain.
- Typical use: background maintenance, metrics reporters.

### 5. Execution Flow
Direct thread:
1. Create `Thread` with `Runnable` (or subclass).
2. Call `start()`.
3. Scheduler runs it; state transitions happen.
4. Optional coordination (`join`, interrupt, synchronization).

Executor-based:
1. Create pool.
2. Submit tasks (`execute`/`submit`).
3. Task enters queue or handed to worker.
4. Worker runs task.
5. Result available via `Future` (if `submit` used).
6. Shutdown pool (`shutdown`/`awaitTermination`).

### 6. Performance & Scalability
- Thread creation is expensive (OS resources, stack allocation).
- Too many threads -> context-switch overhead, cache misses, memory pressure.
- Thread pools amortize creation cost via worker reuse.
- CPU-bound tasks: pool size near number of CPU cores.
- I/O-bound tasks: can use more threads, but avoid unbounded growth.

### 7. Multithreading Concepts (VERY IMPORTANT)
- **Race condition:** multiple threads update/read shared state without proper coordination.
- **Synchronization:** `synchronized`, locks, atomics to protect critical sections.
- **Thread safety:** class behaves correctly under concurrent access.
- **Visibility / happens-before:** synchronization actions (lock/unlock, volatile write/read, thread start/join) create ordering/visibility guarantees.

### 8. Code Examples
**Simple (`Runnable`)**
```java
Thread t = new Thread(() -> System.out.println("Hello from " + Thread.currentThread().getName()));
t.start();
```

**Correct modern usage (`ExecutorService` + `Callable`)**
```java
ExecutorService pool = Executors.newFixedThreadPool(4);
try {
    Future<Integer> future = pool.submit(() -> 40 + 2);
    System.out.println(future.get()); // 42
} finally {
    pool.shutdown();
}
```

**Bad example (`run()` instead of `start()`)**
```java
Thread t = new Thread(() -> System.out.println("Not concurrent"));
t.run(); // runs on current thread, no new thread created
```

### 9. Common Pitfalls
- Creating a new thread per request/task.
- Using `run()` instead of `start()`.
- Shared mutable state without synchronization.
- Forgetting thread naming and exception handling.
- Using non-daemon threads for forever background loops unintentionally.

### 10. Interview Deep Questions
- What is the exact difference between `start()` and `run()`?
- Why is extending `Thread` usually discouraged?
- Which thread states exist and when transitions happen?
- When should a thread be daemon?
- How does `join()` provide happens-before guarantees?

### 11. Summary (Short speaking answer)
In Java, we can run concurrent code by extending `Thread`, implementing `Runnable`, implementing `Callable`, or preferably using `ExecutorService`. `Runnable` is for no-result tasks, `Callable` returns a value and can throw checked exceptions. `start()` creates a new thread, while `run()` is just a normal method call. Direct thread creation does not scale due to creation and context-switch cost, so thread pools are preferred in production. Correct concurrency also requires synchronization and visibility guarantees to avoid race conditions.

**SHORT (30 sec answer)**  
You can create concurrency via `Thread`, `Runnable`, `Callable`, but production code should usually use `ExecutorService`. `Runnable` has no return value, `Callable` returns a result and can throw checked exceptions. `start()` launches a new thread; `run()` does not. Daemon threads are background threads that do not keep JVM alive. Always think about thread safety, race conditions, and shutdown strategy.

**DEEP checkpoints**
- Can compare all 4 approaches with pros/cons.
- Can explain full thread lifecycle states.
- Clearly distinguishes `run()` vs `start()`.
- Explains daemon behavior and when to use.
- Mentions happens-before basics.

<a id="q8"></a>

## 8. What is ExecutorService? What implementations of this interface do you know?

### Question Restatement
What is `ExecutorService`, why do we need it instead of manual threads, and what are the key pool types and internals?

### 1. Definition
`ExecutorService` is a high-level concurrency API for asynchronous task execution and lifecycle management over a thread pool.

### 2. Core Idea
It solves thread management complexity: you submit tasks, executor decides when/where to run them, reuses workers, queues excess tasks, and supports graceful shutdown.

### 3. How It Works Internally
Most common implementation is `ThreadPoolExecutor`.

Core internals:
- **`corePoolSize`**: baseline number of worker threads kept alive.
- **`maximumPoolSize`**: hard upper bound of workers.
- **`workQueue`**: stores waiting tasks.
- **`keepAliveTime`**: idle timeout for extra workers (and optionally core workers).
- **Worker threads**: pull tasks in loop from queue and execute.

Scheduling model (ThreadPoolExecutor):
1. If workers < core -> create thread immediately.
2. Else enqueue task.
3. If queue full and workers < max -> create extra thread.
4. Else reject task (rejection policy).

Important queue types:
- **`LinkedBlockingQueue`** (optionally unbounded): stabilizes thread count but can grow memory usage.
- **`SynchronousQueue`** (no storage): direct handoff; tends to create threads quickly with high max.

### 4. Ways to Use / Implement
Common factory methods:
1. `newFixedThreadPool(n)`  
   - Fixed worker count, usually unbounded queue.
   - Good for steady CPU-bound throughput.
2. `newCachedThreadPool()`  
   - `SynchronousQueue`, potentially many threads.
   - Good for many short-lived async tasks; risky under load spikes.
3. `newSingleThreadExecutor()`  
   - One worker, tasks serialized.
   - Useful for strict ordering.
4. `newScheduledThreadPool(n)`  
   - Delayed/periodic tasks (`scheduleAtFixedRate`, `scheduleWithFixedDelay`).

Manual constructor (`ThreadPoolExecutor`) is preferred in serious systems to control queue, limits, thread factory, rejection policy.

### 5. Execution Flow
1. Create pool.
2. Submit task (`execute` or `submit`).
3. Pool accepts -> worker executes or task queued.
4. Optional result via `Future`.
5. On overload, rejection handler invoked.
6. Shutdown phase:
   - `shutdown()` -> stop accepting new tasks, finish queued/running.
   - `shutdownNow()` -> attempt interruption, returns not-started tasks.

### 6. Performance & Scalability
- Reusing threads avoids repeated creation overhead.
- Proper sizing prevents too much context switching.
- Unbounded queues can hide overload until memory pressure appears.
- Wrong pool type leads to latency spikes or OOM.
- CPU-bound: fixed-size near core count.
- I/O-bound: may require more threads but with backpressure and timeouts.

### 7. Multithreading Concepts (VERY IMPORTANT)
- Race conditions still possible inside tasks; pool does not make code thread-safe.
- Visibility rules still apply (shared data must be safely published/synchronized).
- Interrupt handling is critical for cancellation and shutdown.
- Thread pools need backpressure strategy to avoid overload collapse.

### 8. Code Examples
**Simple fixed pool**
```java
ExecutorService pool = Executors.newFixedThreadPool(4);
try {
    pool.execute(() -> System.out.println(Thread.currentThread().getName()));
} finally {
    pool.shutdown();
}
```

**Correct modern configurable usage**
```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
        4, 8,
        60, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(1000),
        Executors.defaultThreadFactory(),
        new ThreadPoolExecutor.CallerRunsPolicy()
);
```

**Bad example**
```java
ExecutorService pool = Executors.newCachedThreadPool();
while (true) {
    pool.submit(() -> doBlockingIo()); // unbounded pressure -> thread explosion risk
}
```

### 9. Common Pitfalls
- Forgetting `shutdown()`.
- Using cached pool for heavy blocking workload.
- Using unbounded queue in high-traffic service without limits/monitoring.
- Ignoring rejection policy behavior.
- Swallowing `InterruptedException`.

### 10. Interview Deep Questions
- Difference between `execute()` and `submit()`?
- Why can unbounded queues be dangerous?
- What happens when queue is full?
- How does `CallerRunsPolicy` provide backpressure?
- `shutdown()` vs `shutdownNow()` semantics?

### 11. Summary (Short speaking answer)
`ExecutorService` is a thread-pool abstraction that decouples task submission from thread management. The main implementation, `ThreadPoolExecutor`, controls behavior through core/max pool sizes, queue type, keep-alive, and rejection policy. Common pool types are fixed, cached, single-thread, and scheduled executors. Pools improve scalability by reusing threads, but wrong sizing or queue choices can cause latency, memory issues, or overload collapse. Always shut down executors and design for interruption/backpressure.

**SHORT (30 sec answer)**  
`ExecutorService` is the standard way to run async tasks in Java using reusable worker threads. Instead of creating threads manually, tasks are queued and executed by a pool. `ThreadPoolExecutor` internals are core/max size, queue, keepAlive, and rejection policy. Typical implementations are fixed, cached, single-thread, and scheduled pools. In production, configure pools explicitly and always shut them down.

**DEEP checkpoints**
- Can explain full ThreadPoolExecutor admission algorithm.
- Knows queue trade-offs (`LinkedBlockingQueue` vs `SynchronousQueue`).
- Knows rejection handlers and backpressure meaning.
- Correctly distinguishes shutdown methods.
- Can select pool by workload type.

<a id="q9"></a>

## 9. What is the Future interface? What is the difference between our own implementation of this interface and CompletableFuture?

### Question Restatement
What does `Future` give us, what are its limitations, and why is `CompletableFuture` considered a more powerful modern abstraction?

### 1. Definition
- **`Future<V>`**: handle to an asynchronous computation result.
  - check completion (`isDone`)
  - wait and get result (`get`)
  - cancel (`cancel`)
- **`CompletableFuture<T>`**: both `Future` + `CompletionStage`; supports async pipelines, callbacks, composition, and combination.

### 2. Core Idea
`Future` solves “get result later” but is mostly blocking/polling oriented.  
`CompletableFuture` solves end-to-end async orchestration without forcing blocking waits between steps.

### 3. How It Works Internally
**Future (typical with ExecutorService)**
- Task submitted to executor.
- Worker computes result.
- Future stores completion state/result/exception.
- `get()` blocks caller until done.

**CompletableFuture**
- Maintains completion state and dependent stages graph.
- Completion of one stage triggers dependent actions.
- Non-async methods may run in completing thread.
- Async methods (`thenApplyAsync`, `supplyAsync`) use executor; default is `ForkJoinPool.commonPool()` if executor not provided.

### 4. Ways to Use / Implement
1. `Future` via `ExecutorService.submit(Callable)`
   - Simple result retrieval.
   - Good for isolated tasks.
2. Custom own `Future` implementation
   - Rare and error-prone.
   - Need to correctly handle synchronization, cancellation, interruption, state transitions, spurious wakeups, visibility.
3. `CompletableFuture`
   - `supplyAsync/runAsync`
   - transform (`thenApply`)
   - compose (`thenCompose`)
   - combine (`thenCombine`, `allOf`, `anyOf`)
   - recovery (`exceptionally`, `handle`)

### 5. Execution Flow
`Future` flow:
1. submit callable
2. executor runs it
3. caller blocks on `get()` or polls `isDone`

`CompletableFuture` flow:
1. create source stage (async or manual completion)
2. chain dependent stages
3. completion propagates through graph
4. optionally block only at boundary (`join/get`) or continue non-blocking callbacks

### 6. Performance & Scalability
- Blocking `Future#get()` ties up caller threads; at scale this can reduce throughput.
- `CompletableFuture` allows non-blocking continuation style, reducing parked threads.
- But excessive tiny async stages can add overhead.
- Default common pool may be unsuitable for blocking operations; pass custom executor for I/O-heavy pipelines.

### 7. Multithreading Concepts (VERY IMPORTANT)
- Race conditions still possible if callbacks mutate shared state.
- Cancellation is cooperative; interruption handling must be coded correctly.
- Visibility is guaranteed by completion mechanics (safe publication of result), but your shared mutable objects still need thread-safe design.
- Be careful mixing blocking calls inside async chains (can deadlock/starve pools).

### 8. Code Examples
**Simple Future**
```java
ExecutorService pool = Executors.newFixedThreadPool(4);
try {
    Future<Integer> f = pool.submit(() -> 21 * 2);
    Integer result = f.get(); // blocking
} finally {
    pool.shutdown();
}
```

**Correct modern CompletableFuture**
```java
CompletableFuture<String> cf =
        CompletableFuture.supplyAsync(() -> "42")
                .thenApply(Integer::parseInt)
                .thenApply(x -> x + 1)
                .thenApply(String::valueOf);

String result = cf.join();
```

**Bad example (blocking inside common pool chain)**
```java
CompletableFuture.supplyAsync(() -> callRemote()) // blocking I/O
        .thenApply(resp -> transform(resp))
        .join(); // can hurt scalability if many requests do this on commonPool
```

### 9. Common Pitfalls
- Calling `get()` too early and serializing async workflow.
- Ignoring exceptions in async chains.
- Forgetting custom executor for blocking tasks.
- Implementing custom Future incorrectly (state races, missed notifications).
- Confusing `thenApply` vs `thenCompose` (nested futures).

### 10. Interview Deep Questions
- Why is `Future` hard to compose?
- Difference between `thenApply` and `thenCompose`?
- `join()` vs `get()`?
- Which thread executes `thenApply` callback?
- When should you avoid default common pool?

### 11. Summary (Short speaking answer)
`Future` represents a pending result and supports cancellation, but typical usage is blocking via `get()` and lacks composition/callback APIs. `CompletableFuture` extends this model with non-blocking chaining, combination, and error handling, making it suitable for complex async workflows. Async variants run on a provided executor or default `ForkJoinPool.commonPool()`. For simple one-shot background tasks, `Future` may be enough; for pipelines and orchestration, `CompletableFuture` is usually better. Custom Future implementations are rare because correctness is hard.

**SHORT (30 sec answer)**  
`Future` is a handle to async result, but it is mostly blocking and not composable. `CompletableFuture` adds callback-based, chainable async programming (`thenApply`, `thenCompose`, `allOf`) with better orchestration. By default async stages use `ForkJoinPool.commonPool()` unless custom executor is provided. Use `Future` for simple tasks and `CompletableFuture` for multi-step async flows.

**DEEP checkpoints**
- Can articulate all key Future limitations.
- Correctly compares `thenApply` vs `thenCompose`.
- Knows default pool behavior and when to override.
- Understands blocking hazards and exception propagation.
- Can explain why custom Future is rarely justified.
<a id="q10"></a>

## 10. What is the difference between Concurrent and Synchronized collections?

### Question Restatement
What is the difference between **synchronized wrappers** from `Collections` and **concurrent** collections like `ConcurrentHashMap` or `CopyOnWriteArrayList`—locking, scalability, iteration, and when to use which?

### 1. Definition
- **Synchronized collections** (`Collections.synchronizedList/Map/Set/...`): a **thread-safe facade** around a normal collection. Almost every public method is wrapped in **`synchronized`** on a **single monitor** (often `this` of the wrapper or an explicit mutex object).
- **Concurrent collections** (`java.util.concurrent`): designed for **high concurrency**—often **fine-grained** locking, **lock-free or mixed** algorithms, and iterators that do **not** throw `ConcurrentModificationException` in the classic fail-fast sense.

### 2. Core Idea
**Synchronized collections** solve correctness (“no races on individual method calls”) with **simple but coarse** locking.  
**Concurrent collections** solve **throughput and scalability** under many concurrent readers/writers by reducing lock scope, using volatile/CAS, or copying strategies.

### 3. How It Works Internally
- **`Collections.synchronizedXxx`**: one big lock serializes access to the backing collection. Internal `modCount` still exists; iterator is typically **fail-fast** if the collection is structurally modified during iteration (unless you manually synchronize on the same lock for the whole iteration).
- **`ConcurrentHashMap` (Java 8+)**: **no single map-wide lock** for reads; uses **volatile** reads, **CAS** on first bin, and **synchronized** on **bin heads** (or tree roots) for writes/conflicts. Scales better than wrapping `HashMap`.
- **`CopyOnWriteArrayList`**: **copy-on-write**—mutations copy the whole backing array under a lock; reads are **lock-free** (plain array reference). Great for **read-heavy**, rare writes.

**Memory visibility**: both styles rely on JMM rules (locks, volatile, CAS) so updates are **visible** across threads when the API contract is respected.

### 4. Execution Model
- **Synchronized**: callers **block** waiting for the one monitor; high **contention** → threads queue.
- **Concurrent**: many operations proceed with **less blocking** (e.g. concurrent reads on `ConcurrentHashMap`); writers may still block each other on the same bin/segment logic.

### 5. Performance
| Aspect | Synchronized wrappers | Concurrent collections |
|--------|------------------------|-------------------------|
| Contention | High under load (one lock) | Lower for typical read-heavy maps |
| Scalability | Poor for many writers | Better (CHM, queues, etc.) |
| Read-heavy list | Every read takes lock | `CopyOnWriteArrayList`: reads very cheap |
| Write-heavy list | OK if few threads | `CopyOnWriteArrayList`: **bad** (full copy) |

### 6. Code Examples
**Correct: compound operation on synchronized map (must lock externally)**
```java
Map<String, Integer> map = Collections.synchronizedMap(new HashMap<>());
synchronized (map) {
    if (!map.containsKey("k")) {
        map.put("k", 1); // atomic check-then-act relative to other sync users
    }
}
```

**Incorrect: “thread-safe” but still racy**
```java
Map<String, Integer> map = Collections.synchronizedMap(new HashMap<>());
if (!map.containsKey("k")) {  // race: another thread can put between these lines
    map.put("k", 1);
}
```

**ConcurrentHashMap for atomic logic**
```java
map.putIfAbsent("k", 1);
map.compute("k", (k, v) -> v == null ? 1 : v + 1);
```

### 7. Common Pitfalls
- Assuming **each method** being synchronized makes **compound operations** safe (they do not).
- Iterating synchronized collections **without** holding the lock → `ConcurrentModificationException` or inconsistent views.
- Using **`CopyOnWriteArrayList`** with **frequent writes** → memory churn and O(n) per write.
- Using **`ConcurrentHashMap`** and forgetting **`null` keys/values are not allowed**.

### 8. Interview Deep Questions
- Why is `if (!map.containsKey(k)) map.put(k,v)` unsafe with `synchronizedMap`?
- Why is `ConcurrentHashMap`’s `size()` not always O(1) cheap in theory (approximation in some versions / still manageable)?
- Fail-fast vs **weakly consistent** iteration—what can you observe during iteration?
- When is `Collections.synchronizedList` still acceptable?

### 9. Summary (Short speaking answer)
Synchronized collections wrap a classic collection and guard **every method** with **one lock**—correct for single calls but **compound operations** still need external synchronization or atomic APIs. Concurrent collections use **finer-grained** or **lock-free/CAS** designs for better scalability; iterators are often **weakly consistent** rather than fail-fast. `CopyOnWriteArrayList` optimizes **read-heavy** workloads by copying on write. Choose synchronized wrappers for low contention and simplicity; choose concurrent types for shared hot data structures under real parallelism.

**SHORT (30 sec answer)**  
Synchronized collections use **one big lock** per wrapper—simple but contends easily; compound actions like check-then-put are still racy unless you synchronize on the same monitor. Concurrent collections (`ConcurrentHashMap`, `CopyOnWriteArrayList`, queues) use **finer-grained** or **copy-on-write** strategies for scale and safer iteration semantics. Iterators: classic wrapped maps/lists tend to be **fail-fast**; many concurrent iterators are **weakly consistent**.

<a id="q11"></a>

## 11. What are Atomics?

### Question Restatement
What are **atomic** classes in `java.util.concurrent.atomic`, how do they work without traditional locks, and what are **CAS**, **ABA**, and their limits?

### 1. Definition
**Atomics** (`AtomicInteger`, `AtomicLong`, `AtomicBoolean`, `AtomicReference<V>`, and variants like `AtomicIntegerArray`, `AtomicReferenceFieldUpdater`, etc.) expose **single variables** whose updates can be done **atomically** using CPU primitives, typically **compare-and-set (CAS)**, without holding a `synchronized` monitor for every update.

### 2. Core Idea
They solve **race-free updates** to one shared counter or reference with **low overhead** compared to coarse locking, and they establish **happens-before** between successful writes and subsequent reads of that atomic variable (via volatile semantics of the underlying field).

### 3. How It Works Internally
- Under the hood, fields are **`volatile`** (visibility + ordering with respect to other volatile/lock rules).
- **`compareAndSet(expected, update)`** (CAS): in one hardware-supported sequence, if current value == expected, set to update and return true; else return false. No traditional mutex.
- **Lock-free vs wait-free**: repeated CAS in a loop (e.g. `getAndAdd`) is **lock-free** (system progresses) but a thread might **retry** under contention (live-lock style spinning in pathological cases).
- **ABA problem**: thread reads `A`, is preempted; another thread changes `A→B→A`; first thread’s CAS still succeeds with “expected A” though the value was **not** logically unchanged in between. **`AtomicStampedReference` / `AtomicMarkableReference`** add a stamp/mark to detect this when it matters.

### 4. Execution Model
- **Non-blocking** in the sense: no OS-level mutex for the atomic update path (may still spin/retry).
- Under **high contention**, CAS loops burn CPU; synchronized might sometimes be kinder to the OS scheduler (trade-off depends on workload).

### 5. Performance
- **Low contention**: atomics are often **faster** than `synchronized` for simple increments.
- **High contention**: many failed CAS retries → **CPU overhead**; may need `LongAdder` (striped accumulation) or restructuring.
- **Complex invariants** across multiple fields: atomics alone **do not** compose into one atomic transaction.

### 6. Code Examples
**Correct: safe counter**
```java
AtomicInteger count = new AtomicInteger();
count.incrementAndGet();
```

**Correct: atomic reference swap**
```java
AtomicReference<State> state = new AtomicReference<>(State.INIT);
state.compareAndSet(State.INIT, State.RUNNING);
```

**Incorrect: two atomics ≠ one atomic transaction**
```java
AtomicInteger a = new AtomicInteger(0);
AtomicInteger b = new AtomicInteger(0);
a.incrementAndGet();
b.incrementAndGet(); // another thread can see updated a but not b yet—no single atomic step
```

### 7. Common Pitfalls
- Using atomics for **multi-field** invariants without a **single** atomic object or lock.
- Ignoring **ABA** when reusing nodes in lock-free structures (advanced).
- Busy-spinning CAS loops without backoff under extreme contention.
- Assuming `get()` + `compareAndSet()` without loop is always safe (lost update—must use **loop** or higher-level API like `updateAndGet`).

### 8. Interview Deep Questions
- What is CAS at CPU level?
- Difference between `volatile`, `synchronized`, and `AtomicInteger`?
- When is `LongAdder` better than `AtomicLong`?
- Why can’t atomics replace all synchronization?

### 9. Summary (Short speaking answer)
Atomics provide **lock-free-style** updates to single variables using **CAS** on **volatile** fields, giving atomicity and visibility without a monitor. They excel for **counters, flags, and single references** under moderate contention. They do not automatically make **multi-step** logic atomic; for that you need locks, `synchronized`, or careful immutable designs. **ABA** can fool naive CAS on reused objects; stamped references exist when that matters.

**SHORT (30 sec answer)**  
Atomics (`AtomicInteger`, `AtomicReference`, …) update a variable using **CAS** (compare-and-swap) on a **volatile** field—no mutex, good for simple counters and flags. They avoid locks but don’t fix **multi-field** atomicity. Under heavy contention CAS can **retry** a lot (CPU cost). **ABA** is a classic pitfall; use stamped/markable references if reuse is an issue.

<a id="q12"></a>

## 12. What is Monitor?

### Question Restatement
What is a **monitor** in Java/JVM terms, how does it relate to **`synchronized`**, **`wait`/`notify`**, and **happens-before**?

### 1. Definition
A **monitor** is a **per-object synchronization construct** the JVM associates with every Java object. It provides **mutual exclusion** (only one thread can hold the monitor at a time for a given lock) and a **wait set** for **condition waiting** (`wait`/`notify`).

### 2. Core Idea
It solves **safe shared access**: threads coordinate so critical sections see a **consistent** view of memory, and the JMM gives **happens-before** edges between **unlock** and subsequent **lock** on the same monitor.

### 3. How It Works Internally
- **`synchronized` block/method**: **monitor enter** on the object’s monitor (instance lock or class lock for static). If busy, thread enters **`BLOCKED`** until acquired.
- **Intrinsic lock** = the monitor of that object; cannot be released manually—always paired with block/method exit (including exceptions).
- **Object header (high level)**: HotSpot stores **mark word** in the object header; it can encode **hashcode**, **GC age**, and **lightweight lock** state (stack-based lock record) or fall back to **heavyweight** monitor (OS/mutex association) under contention or complexity.
- **`wait()`**: must be called **while holding** the monitor; atomically **releases** the monitor and moves thread to **`WAITING`** on the object’s wait set until **notify/notifyAll/interrupt/timeout**.
- **`notify()` / `notifyAll()`**: wake thread(s); they must **re-acquire** monitor before continuing (`BLOCKED` then `RUNNABLE`).

**Visibility**: actions inside a `synchronized` block are ordered with respect to the **same lock**; release of lock **happens-before** acquire by another thread.

### 4. Execution Model
- **Blocking**: contended `synchronized` blocks threads.
- **Condition coordination**: `wait`/`notify` is **low-level**; production code often prefers `java.util.concurrent` locks and conditions for clearer APIs and timeouts/interrupt policies.

### 5. Performance
- Uncontended `synchronized` is often **biased** and cheap (JVM optimizations).
- High contention → **serialized** execution; threads **park**; scalability drops.
- Lock-free/CAS can win for **very hot** single-word updates but is harder to reason about for invariants.

### 6. Code Examples
**Correct: guarded state**
```java
synchronized (this) {
    while (!ready) {
        wait();
    }
    useResource();
}
```

**Incorrect: calling `wait()` without holding lock**
```java
obj.wait(); // IllegalMonitorStateException
```

### 7. Common Pitfalls
- Calling **`wait`/`notify`** outside `synchronized` → `IllegalMonitorStateException`.
- Using **`notify`** when **`notifyAll`** is safer to avoid missed signals (depends on design).
- Holding locks while doing **slow I/O** → kills scalability and risks deadlocks.
- Nested locks in inconsistent order → **deadlock**.

### 8. Interview Deep Questions
- Difference between **intrinsic lock** and `ReentrantLock`?
- `BLOCKED` vs `WAITING`?
- What **happens-before** edges does `synchronized` create?
- Biased locking / lock elision (high-level JVM optimization awareness)?

### 9. Summary (Short speaking answer)
A **monitor** is the JVM’s per-object **mutex + wait-set** used by **`synchronized`**. Entering acquires the **intrinsic lock**; exiting releases it. **`wait`/`notify`** also use the same monitor and move threads between **RUNNABLE**, **BLOCKED**, and **WAITING**. The JMM guarantees that a **unlock happens-before** later **lock** on the same monitor, giving **visibility** and **ordering** for guarded data. For complex concurrency, higher-level APIs often sit on top of these primitives.

**SHORT (30 sec answer)**  
A **monitor** is each object’s **intrinsic lock** in the JVM. **`synchronized`** does **monitor enter/exit** for mutual exclusion. **`wait`/`notify`** require holding the lock and use the monitor’s **wait set**. This gives **happens-before** between release and acquire, so threads see consistent memory. **`BLOCKED`** waits for the lock; **`WAITING`** waits on `wait()` until notified.

<a id="q13"></a>

## 13. What are the segments of Java Memory Model?

### Question Restatement
When we talk about Java memory, what areas exist in JVM runtime, which are thread-local vs shared, and how does the Java Memory Model define visibility and ordering between threads?

### 1. Definition
The **Java Memory Model (JMM)** is a formal specification that defines how threads interact through memory: what values a thread is allowed to see, when writes by one thread become visible to another, and what instruction reordering is legal.

In interviews, people also ask about “memory segments” meaning JVM runtime memory areas. Strictly speaking:
- **JMM** = visibility/ordering/atomicity rules
- **JVM runtime memory areas** = Heap, Stack, Metaspace, PC register, Native method stack

### 2. Core Idea
Without JMM rules, multithreaded Java would be unpredictable due to CPU caches, compiler/JIT optimizations, and reordering.  
JMM provides a contract so concurrent code can be reasoned about using tools like `volatile`, `synchronized`, locks, and atomics.

### 3. Internal Structure
#### Heap (shared)
- Stores objects and arrays.
- Shared by all threads.
- Main GC-managed area (young/old generations in HotSpot collectors).

#### Java Stack (thread-local)
- Each thread has its own stack.
- Contains stack frames per method call: local variables, operand stack, return info.
- Local primitive values and object references live here (object itself in heap).

#### Metaspace (shared, native memory)
- Stores class metadata: class definitions, method metadata, constant pools, etc.
- Replaced PermGen since Java 8.
- Not in Java heap; can still OOM (`OutOfMemoryError: Metaspace`) if class loading leaks.

#### PC Register (thread-local)
- Points to current bytecode instruction for each thread.
- Tiny but important for execution state and thread switching.

#### Native Method Stack (thread-local)
- Used by JNI/native methods.
- Separate from Java stack conceptually.

### 4. How It Works Internally
#### Thread stack vs shared heap
- Each thread manipulates local variables in its own stack frame.
- Shared objects are on heap; references to them may be copied into local variables.

#### Local variables vs objects
- Local primitive variable: thread-private unless published.
- Object fields: shared if object reference is shared.

#### Working memory vs main memory (JMM abstraction)
- JMM models each thread as having **working memory** (registers/cache-like abstraction) and a shared **main memory**.
- Threads may read a value into working memory and not immediately observe others’ writes unless synchronization establishes visibility.

#### Visibility, atomicity, ordering
- **Visibility**: whether one thread sees another thread’s writes.
- **Atomicity**: operation appears indivisible.
- **Ordering**: constraints on reordering of operations.

#### Happens-before (critical)
- If action A **happens-before** action B, then B must observe effects of A.
- Key happens-before edges:
  - `synchronized` unlock -> subsequent lock on same monitor
  - `volatile` write -> subsequent `volatile` read of same variable
  - `Thread.start()` -> actions in started thread
  - actions in thread -> successful `Thread.join()` return

#### `volatile` vs `synchronized` (brief)
- `volatile`: visibility + ordering for a single variable, no mutual exclusion.
- `synchronized`: mutual exclusion + visibility on lock boundaries.

### 5. Algorithms / Types (JMM-focused)
JMM itself is not a GC algorithm; it is a memory consistency model.  
Important execution concepts:
- Read/write barriers inserted by JIT for `volatile`/locks.
- Reordering allowed unless forbidden by happens-before constraints.
- Data race (conflicting unsynchronized accesses) leads to undefined interleavings from application perspective.

### 6. Execution Model
- Threads execute independently with local caches/register effects.
- On synchronization actions, JVM/CPU enforce memory barriers.
- Correct concurrent programs rely on happens-before edges to guarantee deterministic visibility.

### 7. Performance
- Synchronization primitives add overhead (locks, barriers, potential contention).
- Over-synchronization can reduce throughput.
- Under-synchronization causes rare, hard-to-reproduce bugs.
- `volatile` is lighter than full locking but not free and not a substitute for compound atomic logic.

### 8. Code / Practical Examples
**Visibility problem without happens-before**
```java
class FlagExample {
    boolean ready = false;
    int data = 0;
}
```
Thread A: `data = 42; ready = true;`  
Thread B: loops on `ready`; may never see `true` reliably without synchronization.

**Fix with volatile**
```java
class FlagExample {
    volatile boolean ready = false;
    int data = 0;
}
```
If Thread A writes `data=42` then `ready=true`, Thread B seeing `ready=true` also sees prior write to `data` due to volatile ordering guarantees.

### 9. Common Pitfalls
- Thinking “stack vs heap” alone explains thread safety (it does not).
- Believing `volatile` makes compound operations (`count++`) atomic.
- Publishing mutable objects without synchronization/safe publication.
- Ignoring classloader leaks causing Metaspace OOM.

### 10. Interview Deep Questions
- Is JMM equal to JVM memory areas? (No, different concepts.)
- Why can one thread not see another thread’s latest write?
- Difference between visibility and atomicity?
- Why does `double-checked locking` require `volatile`?
- What happens-before edge is created by `join()`?

### 11. Summary (Short speaking answer)
JMM defines how Java threads read and write shared memory: visibility, atomicity rules, and allowed reorderings. Runtime memory areas are Heap (shared), Stack/PC/Native stack (thread-local), and Metaspace (shared class metadata). Thread-local stacks hold local variables and references, while objects live in shared heap. Correct synchronization (`synchronized`, `volatile`, locks, atomics) creates happens-before relationships so writes become visible across threads. Without these guarantees, code can exhibit stale reads and race-condition behavior.

**SHORT (30 sec answer)**  
JMM is a **consistency model**, not just memory layout: it defines visibility and ordering between threads. JVM areas are Heap (shared), Stack/PC/Native stack (thread-local), and Metaspace (shared metadata). Threads can cache values, so without happens-before they may read stale data. `volatile` gives visibility/ordering for a variable, while `synchronized` adds mutual exclusion plus visibility at lock boundaries.

<a id="q14"></a>

## 14. What is GC? What types of GC do you know? What GC is used by default?

### Question Restatement
What is Java GC, how object memory moves through generations, what collectors exist (Serial, Parallel, G1, ZGC, Shenandoah), and which one is default in modern Java?

### 1. Definition
**Garbage Collection (GC)** is automatic memory management in JVM: it discovers objects no longer reachable from GC roots and reclaims their memory.

### 2. Core Idea
GC solves manual memory deallocation errors (double free, dangling pointers), but introduces runtime overhead and pause trade-offs.  
Java GC is based on **reachability**, not reference counting.

### 3. Internal Structure
Generational layout (typical HotSpot generational collectors):
- **Young Generation**
  - **Eden**: most new objects allocated here
  - **Survivor S0/S1**: objects that survive minor GC are copied between survivor spaces with age tracking
- **Old Generation (Tenured)**: long-lived objects promoted from young gen
- **Metaspace**: class metadata (native memory), collected separately via class unloading logic

### 4. How It Works Internally
#### Generational hypothesis
- Most objects die young.
- Objects that survive several collections are likely to live longer.

#### Lifecycle
1. Allocate object in **Eden** (fast bump-pointer allocation via TLAB in many cases).
2. Eden fills -> **Minor GC** (young-only collection).
3. Live objects copied to Survivor, age incremented.
4. Objects reaching age threshold (or when Survivor pressure high) promoted to **Old Gen**.
5. Old Gen pressure triggers **Major/Old collection**.
6. **Full GC** usually means broad collection (young+old+often metaspace-related work), typically more expensive.

#### Stop-The-World (STW)
- Many phases require stopping application threads.
- Modern collectors reduce STW duration by doing more work concurrently, but some pauses remain.

### 5. Algorithms / Types (VERY IMPORTANT)
#### Core algorithms
- **Mark-and-Sweep**: mark reachable, sweep unreachable (can leave fragmentation).
- **Mark-and-Compact**: mark, then compact live objects to reduce fragmentation.
- **Copying**: copy live objects from one region/space to another; fast allocation, good for young gen where most die.

#### Collector types
1. **Serial GC** (`-XX:+UseSerialGC`)
   - Single-threaded collector.
   - Simple, low overhead; good for tiny heaps/single-core environments.
2. **Parallel GC** (`-XX:+UseParallelGC`)
   - Multi-threaded STW collection focused on **throughput**.
   - Can have longer pauses than low-latency collectors.
3. **G1 GC** (`-XX:+UseG1GC`) — **VERY IMPORTANT**
   - Region-based heap (not rigid contiguous old/young only spaces conceptually).
   - Collects regions with most garbage first (“garbage-first”).
   - Aims for predictable pause targets (`-XX:MaxGCPauseMillis`).
   - Performs mixed collections (young + selected old regions).
4. **ZGC** (`-XX:+UseZGC`) (brief)
   - Ultra-low pause collector; mostly concurrent phases.
   - Uses colored pointers/load barriers.
   - Great for very large heaps and low-latency requirements.
5. **Shenandoah** (`-XX:+UseShenandoahGC`) (brief)
   - Concurrent compaction to minimize pause times.
   - Similar goal to ZGC: low-latency with large heaps.

**Default GC in modern Java:** **G1 GC** (default in modern HotSpot JDKs).

### 6. Execution Model
- App threads allocate; collector threads reclaim memory.
- Some collectors are mostly **parallel STW** (Parallel GC), others more **concurrent** (G1/ZGC/Shenandoah).
- Trade-off triangle: throughput, latency (pause time), footprint.

### 7. Performance
- Frequent young GCs may indicate high allocation rate (not always bad if pauses tiny).
- Long old/full GCs can cause latency spikes.
- Fragmentation affects allocation and pause behavior (compacting collectors mitigate).
- GC bottleneck symptoms:
  - high CPU in GC
  - long pause times
  - promotion failures / allocation stalls
  - repeated full GCs with little reclaimed memory

### 8. Code / Practical Examples
**Allocation-heavy code**
```java
for (int i = 0; i < 1_000_000; i++) {
    String s = new String("x" + i); // many short-lived objects
}
```
Most such temporary objects die in Eden and are reclaimed by Minor GC.

**Logical memory leak (GC cannot free reachable objects)**
```java
List<byte[]> cache = new ArrayList<>();
while (true) {
    cache.add(new byte[1024 * 1024]); // still strongly reachable -> heap grows
}
```

### 9. Common Pitfalls
- “Java has GC, so memory leaks are impossible” (false: logical leaks via strong references).
- Ignoring object churn (excess temporary allocations).
- Misreading minor GC frequency as always bad.
- Tuning flags blindly without metrics (`jstat`, GC logs, JFR).
- Confusing **Major** vs **Full** GC semantics (collector-dependent details).

### 10. Interview Deep Questions
- Why are objects allocated in Eden first?
- What is STW and why can’t all phases avoid it?
- Difference between Minor, Major, and Full GC?
- What happens when heap is full and GC cannot reclaim enough?
- What are Soft/Weak/Phantom references and practical use cases?
- Why can low pause collectors reduce latency but cost more CPU?

### 11. Summary (Short speaking answer)
GC is JVM’s automatic memory reclamation based on object reachability. Most collectors are generational because most objects die young: objects start in Eden, surviving ones move through Survivor to Old Gen. Minor GC handles young gen; old/full collections are heavier and can cause bigger pauses. Collector choice is a throughput-vs-latency trade-off: Serial and Parallel favor simplicity/throughput, while G1, ZGC, and Shenandoah target better pause behavior. In modern Java, default HotSpot collector is typically **G1**.

**SHORT (30 sec answer)**  
GC automatically frees unreachable objects, usually with a **generational** heap (Eden/Survivor/Old). Most objects die young, so minor collections are optimized for that pattern. Major/Full collections are heavier and may pause application threads longer. Modern Java defaults to **G1 GC**, while ZGC/Shenandoah are low-latency alternatives for large heaps. Good GC tuning balances throughput, latency, and memory footprint.

