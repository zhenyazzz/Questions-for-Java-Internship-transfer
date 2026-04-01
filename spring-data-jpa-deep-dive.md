# Spring Data JPA — Interview Cheat Sheet (Middle/Senior)

Production-grade insights, edge cases, and interview-ready answers.

---

## 1) Architecture: How It Actually Works

### Repository Proxy Generation

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByStatus(OrderStatus status);  // Who implements this?
}
```

**Runtime mechanics:**
1. Spring creates JDK dynamic proxy implementing your interface
2. `SimpleJpaRepository` provides CRUD methods
3. Query methods parsed by `PartTreeJpaQuery` → JPQL generated from method name
4. `EntityManager` injected into repository implementation

**Stack trace:**
```
Your Code
  → Spring Data Proxy (AOP)
  → SimpleJpaRepository / PartTreeJpaQuery
  → EntityManager (JPA API)
  → Hibernate Session
  → JDBC
  → Database
```

**Interview Answer:**
Spring Data generates a JDK dynamic proxy at runtime that intercepts method calls. Standard CRUD comes from `SimpleJpaRepository`, while query methods are parsed by `PartTreeJpaQuery` which converts the method name into JPQL. The actual database interaction goes through Hibernate's `Session` (EntityManager implementation).

---

## 2) Persistence Context Deep Dive

### What It Really Is

The Persistence Context (PC) is Hibernate's **first-level cache** — a Map<EntityType, Map<ID, Entity>> scoped to the transaction.

**Identity guarantee (critical):**
```java
@Transactional
public void demo() {
    Order o1 = orderRepository.findById(1L).get();
    Order o2 = orderRepository.findById(1L).get();
    System.out.println(o1 == o2);  // true — same reference!
}
```

**Interview Answer:**
Within a transaction, Persistence Context guarantees that loading the same entity twice returns the exact same Java reference. This prevents inconsistencies where two copies of the same database row could diverge in memory.

### Entity Lifecycle States

| State | In PC? | DB Row? | Changes Tracked? |
|-------|--------|---------|------------------|
| **Transient** | No | No | N/A |
| **Managed** | Yes | Yes | Yes (dirty checking) |
| **Detached** | No | Yes | No |
| **Removed** | Yes (temporarily) | Yes (until flush) | N/A |

**State transitions:**
```java
// Transient → Managed
Order order = new Order();
orderRepository.save(order);  // Now managed, has ID after flush

// Managed → Detached
orderService.findOrder(1L);  // Returns detached after transaction ends

// Detached → Managed (merge)
orderRepository.save(detachedOrder);  // Hibernate merges state

// Managed → Removed
orderRepository.delete(order);  // Marked, deleted on flush
```

---

## 3) Dirty Checking: The Hidden Cost

### How It Works

Hibernate stores a **snapshot** of entity state when loaded from DB. On flush, it compares current state vs snapshot field by field.

**Cost:** Reflection-based field comparison on every entity in PC at flush time.

**Optimization:**
```java
@Transactional(readOnly = true)  // Disables dirty checking entirely
public List<Order> findOrders() {
    return orderRepository.findAll();  // No snapshots taken
}
```

**Common Trap:**
Loading 10,000 entities in a read-only transaction without `@Transactional(readOnly = true)` causes 10,000 snapshot creations and comparisons at flush — major CPU overhead.

---

## 4) save() vs persist() vs merge() — Clarified

### The Real Differences

| Method | JPA Standard? | Input State | Output | SQL Timing |
|--------|--------------|-------------|--------|------------|
| `persist()` | Yes | Transient only | Managed entity | On flush |
| `save()` | No (Spring Data) | Any | Managed entity | Immediately generates ID if needed |
| `merge()` | Yes | Detached | New managed copy | On flush |

**Critical distinction:**
```java
// persist() — only for new entities, ID assigned on flush
Order order = new Order();
entityManager.persist(order);
System.out.println(order.getId());  // null! (before flush)

// save() — generates ID immediately (calls persist + flush if needed for ID)
Order order = new Order();
Order saved = orderRepository.save(order);
System.out.println(saved.getId());  // Has ID immediately
```

**merge() behavior (tricky):**
```java
Order detached = orderRepository.findById(1L).get();  // Load
// transaction ends, detached is now detached

detached.setTotal(BigDecimal.TEN);  // Change
Order managed = orderRepository.save(detached);  // Actually calls merge()

// 'managed' is a NEW instance! 'detached' stays detached
System.out.println(detached == managed);  // false!
```

**Interview Answer:**
`persist()` is pure JPA for new entities only and doesn't return an ID until flush. `save()` is Spring Data convenience that works with any state and ensures ID is populated immediately by triggering flush if using ID generation strategy like IDENTITY. `merge()` creates a new managed copy from a detached entity — the returned entity is a different instance than what you passed in.

**Common Trap:**
Modifying the original entity after `merge()` and expecting changes to be tracked — they won't be, because the returned copy is the managed one, not your original.

---

## 5) Lazy Loading Internals

### How Proxies Work

When you load an entity with `@ManyToOne(fetch = LAZY)`:

```java
Order order = orderRepository.findById(1L).get();
// order.getCustomer() returns a proxy — a subclass with method interception
```

**Hibernate creates a runtime proxy** (subclass of Customer) where:
- `getId()` returns the foreign key value (already known, no DB hit)
- Any other method (`getName()`, `getEmail()`) triggers the actual SELECT

**When query fires:**
```java
Customer customer = order.getCustomer();     // No query — returns proxy
String name = customer.getName();             // NOW: SELECT ... FROM customers WHERE id = ?
```

**Interview Answer:**
Hibernate creates a bytecode-generated proxy subclass. The proxy holds only the foreign key ID initially. When any method other than ID getter is invoked, Hibernate executes the SQL query to load the full entity state, populates the proxy, and delegates the method call.

---

## 6) N+1: Deep Dive & Trade-offs

### The Problem

```java
List<Order> orders = orderRepository.findAll();  // 1 query
for (Order o : orders) {
    o.getCustomer().getName();  // N additional queries
}
```

### Solutions Compared

| Solution | How It Works | Trade-offs |
|----------|--------------|------------|
| **JOIN FETCH** | `JOIN FETCH o.customer` — single query with join | Returns duplicates for `@OneToMany`, breaks pagination |
| **EntityGraph** | `@EntityGraph(attributePaths = "customer")` — declarative fetch | Same join behavior as JOIN FETCH |
| **Batch Fetching** | `@BatchSize(size = 50)` — loads 50 at once | N+1 becomes N/50+1, no pagination issues |
| **Subselect** | `@Fetch(FetchMode.SUBSELECT)` — second query with WHERE IN | Two queries total, efficient for `@OneToMany` |

**JOIN FETCH pagination problem explained:**
```java
// WRONG:
@Query("SELECT o FROM Order o JOIN FETCH o.items")
Page<Order> findAll(Pageable pageable);

// Hibernate cannot apply LIMIT/OFFSET correctly because 
// join creates multiple rows per Order — page would be based on row count, not entity count
```

**The correct pagination fix:**
```java
// Step 1: Get IDs only (no join, can paginate correctly)
@Query("SELECT o.id FROM Order o WHERE ...")
Page<Long> findOrderIds(Pageable pageable);

// Step 2: Fetch full data with join for specific IDs
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id IN :ids")
List<Order> findWithItemsByIds(@Param("ids") List<Long> ids);
```

**Interview Answer:**
JOIN FETCH solves N+1 with a single query but creates duplicate rows for `@OneToMany` collections and breaks pagination since LIMIT applies to rows, not parent entities. For pagination scenarios, use two queries: first get IDs only, then fetch full data with join for those specific IDs. Alternatively, use `@BatchSize` to load associations in batches without join complications.

**Common Trap:**
Using JOIN FETCH with `@OneToMany` and `Pageable` — you get wrong result counts and potentially corrupted page content because the SQL row count doesn't match the entity count.

---

## 7) Flush vs Commit: Exact Mechanics

### What Flush Does

Flush synchronizes the Persistence Context state to the database by executing SQL, but **does NOT commit the transaction**.

**Flush triggers:**
1. Explicit: `entityManager.flush()`
2. Before transaction commit (always)
3. Before JPQL query execution (to ensure query sees current state)
4. Before native query execution

**Why it matters:**
```java
@Transactional
public void complexOperation() {
    orderRepository.save(order);  // Scheduled for insert
    
    // Native query won't see the new order unless flushed!
    jdbcTemplate.queryForObject("SELECT ...");  // No auto-flush for native!
    
    entityManager.flush();  // Force insert before native query
    jdbcTemplate.queryForObject("SELECT ...");  // Now sees it
}
```

**Interview Answer:**
Flush executes pending SQL (INSERT/UPDATE/DELETE) within the transaction but doesn't make changes permanent. Commit actually ends the transaction and makes changes durable. Hibernate flushes automatically before queries to ensure consistency, but native queries don't trigger auto-flush — you must flush manually if the native query depends on pending Hibernate changes.

---

## 8) @Transactional: Proxies & Self-Invocation

### How Spring Transactional Works

1. Spring creates a proxy around your bean (JDK dynamic proxy or CGLIB)
2. Proxy intercepts method calls
3. On entry: starts transaction, binds EntityManager to thread
4. On exit: commits or rolls back

### The Self-Invocation Problem

```java
@Service
public class OrderService {
    
    public void processOrder(Long id) {
        // This internal call bypasses the proxy!
        updateStatus(id, OrderStatus.PROCESSING);  // No transaction!
    }
    
    @Transactional
    public void updateStatus(Long id, OrderStatus status) {
        Order order = orderRepository.findById(id).get();
        order.setStatus(status);
    }
}
```

**Why it fails:** `this.updateStatus()` is a direct method call, not through the Spring proxy. The `@Transactional` interceptor never fires.

**Solutions:**

1. **Self-inject (pragmatic):**
```java
@Service
public class OrderService {
    @Autowired
    private OrderService self;  // Self-reference through Spring
    
    public void processOrder(Long id) {
        self.updateStatus(id, OrderStatus.PROCESSING);  // Goes through proxy
    }
    
    @Transactional
    public void updateStatus(Long id, OrderStatus status) { ... }
}
```

2. **Refactor to separate service (cleaner):**
```java
@Service
public class OrderProcessor {
    private final OrderUpdater updater;  // Different bean
    
    public void process(Long id) {
        updater.updateStatus(id, OrderStatus.PROCESSING);  // Through proxy
    }
}
```

**Interview Answer:**
Spring `@Transactional` works through proxies. When a method calls another method in the same class, it's a direct `this` reference that bypasses the proxy, so the transactional interceptor never executes. Solutions include self-injection or refactoring the transactional method to a different bean so the call goes through Spring's proxy.

**Common Trap:**
Thinking `@Transactional` works on private methods or internal calls — it doesn't, because proxies can't intercept them.

---

## 9) Pagination: Why Standard Approaches Fail

### The Problem with JOIN FETCH + Pageable

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items")
Page<Order> findAll(Pageable pageable);  // WRONG!
```

**Why it breaks:**
- `JOIN FETCH o.items` creates multiple rows per Order (one per item)
- `LIMIT 20` applies to SQL rows, not Order entities
- Hibernate's `PageImpl` gets wrong total count

### The Working Solution

```java
// Repository
@Query("SELECT o.id FROM Order o WHERE o.status = :status")
Page<Long> findOrderIdsByStatus(@Param("status") OrderStatus status, Pageable pageable);

@Query("SELECT o FROM Order o LEFT JOIN FETCH o.items WHERE o.id IN :ids")
List<Order> findWithItemsByIds(@Param("ids") List<Long> ids);

// Service
public Page<Order> getOrders(OrderStatus status, Pageable pageable) {
    Page<Long> idPage = orderRepository.findOrderIdsByStatus(status, pageable);
    
    if (idPage.hasContent()) {
        List<Order> orders = orderRepository.findWithItemsByIds(idPage.getContent());
        return new PageImpl<>(orders, pageable, idPage.getTotalElements());
    }
    return Page.empty();
}
```

**Interview Answer:**
Standard JOIN FETCH with pagination fails because the SQL join creates multiple rows per entity, and LIMIT applies to rows rather than distinct entities. The reliable two-step approach is: first paginate IDs only (single row per entity), then fetch full entities with join fetch for those specific IDs. This preserves correct pagination while avoiding N+1.

---

## 10) equals() and hashCode(): Production Disasters

### The Changing ID Problem

```java
@Entity
public class Order {
    @Id
    @GeneratedValue
    private Long id;
    
    @Override
    public boolean equals(Object o) {
        return id != null && id.equals(((Order)o).id);
    }
    
    @Override
    public int hashCode() {
        return id != null ? id.hashCode() : 1;
    }
}
```

**The disaster scenario:**
```java
Set<Order> set = new HashSet<>();
Order order = new Order();  // id is null
set.add(order);  // hashCode = 1

orderRepository.save(order);  // id is now 42!
// hashCode changed from 1 to 42!

set.contains(order);  // FALSE! hashCode changed while in set!
```

**The Real Bug: HashSet Corruption**
```java
Set<Order> processingQueue = new HashSet<>();

// Add transient orders
orders.forEach(processingQueue::add);

// Persist (IDs assigned)
orderRepository.saveAll(orders);  // IDs changed!

// Now the set is corrupted
processingQueue.contains(orders.get(0));  // Returns false!
```

### The Fix: Business Key

Use a field that exists from object creation and never changes:

```java
@Entity
public class Order {
    @Id
    @GeneratedValue
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String orderNumber;  // Assigned at creation, never changes
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order)) return false;
        return orderNumber != null && 
               orderNumber.equals(((Order) o).orderNumber);
    }
    
    @Override
    public int hashCode() {
        return orderNumber != null ? orderNumber.hashCode() : getClass().hashCode();
    }
}
```

**Interview Answer:**
Using auto-generated ID in equals/hashCode causes critical bugs: the hashCode changes when the entity is persisted and gets an ID. If the entity is in a HashMap or HashSet during this transition, the collection becomes corrupted and can't find the entity anymore. Always use a business key (like orderNumber, email, UUID) that's assigned at object creation and never changes.

**Common Trap:**
Using Lombok `@EqualsAndHashCode` with ID field on JPA entities — this breaks collections when entities transition from transient to managed.

---

## 11) DTO Projections vs Entities

### When Entities Hurt

Loading full entities for read-only operations wastes memory and CPU:
- Hibernate creates snapshots for dirty checking (unnecessary for reads)
- All fields loaded, even if you only need 3
- Lazy loading triggers additional queries

### DTO Projection Options

**1. Interface-based (Spring Data)**
```java
public interface OrderSummary {
    Long getId();
    BigDecimal getTotal();
    String getCustomerName();  // From @Value("#{target.customer.name}")
}

@Query("SELECT o.id as id, o.total as total, o.customer.name as customerName FROM Order o")
List<OrderSummary> findSummaries();
```

**2. Class-based with constructor expression (preferred)**
```java
@Query("SELECT new com.example.OrderDTO(o.id, o.total, o.customer.name) FROM Order o")
List<OrderDTO> findOrderDTOs();
```

**3. Native query with ResultTransformer (complex cases)**

**Benefits:**
- No dirty checking overhead
- Only required fields fetched
- No lazy loading issues
- Can aggregate in SQL

**When to use:**
- Reporting/listing endpoints
- Export functionality
- Complex read queries
- High-throughput read paths

**When NOT to use:**
- CRUD operations needing updates
- When you need to modify and save back

**Interview Answer:**
DTO projections bypass Hibernate's dirty checking and lazy loading overhead by selecting only needed fields directly into immutable DTOs. Use constructor expressions in JPQL for type-safe results. Projections are ideal for read-only list endpoints and reporting, while full entities should be used when you need to modify and persist changes.

---

## 12) When NOT to Use JPA

### Use Native SQL When

1. **Complex reporting with aggregations, window functions**
   ```java
   @Query(value = "SELECT date_trunc('month', created_at) as month, sum(total) " +
                  "FROM orders GROUP BY month ORDER BY month", nativeQuery = true)
   List<Object[]> monthlyRevenue();
   ```

2. **Recursive queries (CTEs)**
   ```sql
   WITH RECURSIVE subordinates AS (
     SELECT id, manager_id FROM employees WHERE id = ?
     UNION ALL
     SELECT e.id, e.manager_id FROM employees e
     INNER JOIN subordinates s ON s.id = e.manager_id
   )
   ```

3. **Complex joins across many tables with specific optimization needs**

4. **Bulk operations affecting millions of rows**
   - JPA `deleteAll()` loads entities then deletes one by one
   - Native `DELETE WHERE` is single statement

### Use JdbcTemplate When

- Batch operations with extreme performance requirements
- Working with legacy schemas that don't map well to entities
- Reading into non-JPA DTOs without the overhead

---

## 13) Open Session In View: The Hidden Killer

### What It Does

```yaml
spring:
  jpa:
    open-in-view: true  # DEFAULT in Spring Boot
```

Extends Persistence Context (and database connection) for the **entire HTTP request**, including view rendering.

### Why It Destroys Production

```java
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable Long id) {
    // Transaction ends here, but PC and connection stay open!
    return orderService.findOrder(id);  // Returns lazy entity
}

// Thymeleaf/Freemarker renders view:
// order.getCustomer().getName()  -- DB call during rendering!
// order.getItems().size()      -- Another DB call!
// Slow template = connection held for seconds
```

**Impact:**
- Connection pool exhaustion during high load
- Database connections held while doing CPU work (rendering)
- Unexpected N+1 in view layer (hard to debug)

**The Fix:**
```yaml
spring:
  jpa:
    open-in-view: false
```

**Then handle explicitly:**
```java
@GetMapping("/orders/{id}")
public OrderDTO getOrder(@PathVariable Long id) {
    // Eagerly fetch everything needed for DTO
    return orderService.findOrderWithDetails(id);  // JOIN FETCH inside
}
```

**Interview Answer:**
Open Session in View keeps the database connection open for the entire HTTP request including view rendering. If the view triggers lazy loading (which it often does), you get database calls during template rendering. This destroys connection pools under load because connections are held while doing CPU-intensive rendering work. Always disable it in production and use eager fetching or DTOs for read endpoints.

**Common Trap:**
N+1 problems appearing "randomly" in production — often caused by view layer triggering lazy loading with OSIV enabled.

---

## 14) Quick Reference: Interview Answers

**Q: Why does Hibernate generate UPDATE I didn't ask for?**
> Dirty checking. Entity loaded and modified in transaction, Hibernate auto-flushes changes. Use `@Transactional(readOnly = true)` to disable dirty checking for queries.

**Q: N+1 solutions and trade-offs?**
> JOIN FETCH: single query but breaks pagination and creates duplicates for collections. Entity Graph: same as JOIN FETCH. Batch fetching: N+1 becomes N/batch+1, works with pagination. Subselect: two queries total. For paginated lists with associations, use two-step: paginate IDs, then fetch full data for those IDs.

**Q: save() vs persist() vs merge()?**
> `persist()`: JPA standard, new entities only, ID after flush. `save()`: Spring Data, works with any state, ID immediately (may flush for IDENTITY). `merge()`: takes detached, returns NEW managed copy (different instance than input).

**Q: Self-invocation with @Transactional?**
> Internal `this.method()` calls bypass Spring's proxy, so @Transactional interceptor never fires. Solutions: self-inject the service, or move transactional method to a different bean.

**Q: Why does entity disappear from HashSet after save()?**
> Auto-generated ID used in hashCode changes during persist, corrupting the HashSet. Always use immutable business key (orderNumber, UUID) for equals/hashCode, never database ID.

**Q: OSIV — good or bad?**
> Bad for production. Holds DB connection through view rendering, causes pool exhaustion and hides N+1 in view layer. Disable it and use explicit JOIN FETCH or DTOs.

**Q: Flush vs Commit?**
> Flush executes SQL within transaction (INSERT/UPDATE/DELETE) but doesn't commit. Happens before queries for consistency and at transaction end. Commit makes changes permanent. Native queries don't trigger auto-flush.

**Q: When to use DTO projections over entities?**
> For read-only list endpoints and reporting — avoids dirty checking overhead, lazy loading issues, and reduces data transfer. Don't use when you need to modify and persist changes.

**Q: Why LazyInitializationException outside transaction?**
> Entity loaded in one transaction, proxy not initialized. Transaction ends, PC closed. Accessing unloaded association tries to load but no PC/connection available. Fix: @Transactional on caller, JOIN FETCH to load eagerly, or initialize in transaction.

**Q: Pagination with JOIN FETCH on @OneToMany?**
> Doesn't work correctly — LIMIT applies to joined rows, not parent entities. Use two queries: first paginate IDs only, then fetch full data with join for those IDs.

---

## Summary

Spring Data JPA generates runtime proxy implementations that delegate to Hibernate. Persistence Context (first-level cache) scopes to the transaction and guarantees identity (same ID = same reference). Dirty checking compares entity state to snapshots — disable with `@Transactional(readOnly = true)` for queries. `save()` ensures ID is immediately available, `persist()` waits for flush, `merge()` creates a new managed copy. Lazy loading uses bytecode proxies that fire SQL on first non-ID access. N+1 is solved by JOIN FETCH (breaks pagination), Entity Graph (same), batch fetching (best for pagination), or subselect. Use two-step pagination: IDs first, then full data. Never use auto-generated ID in equals/hashCode — use immutable business key. Disable OSIV in production — it destroys connection pools. Use DTO projections for read-heavy endpoints to avoid dirty checking overhead. Flush executes SQL but doesn't commit; commit makes changes permanent. Self-invocation bypasses @Transactional proxy — use self-injection or separate beans.

---

## Interview Speaking Mode

How to sound like a strong mid/senior engineer in a real interview.

---

### Spoken Answers by Topic

#### Persistence Context

**Short answer:**
Persistence Context is basically a first-level cache that's scoped to your transaction. When you load an entity, Hibernate keeps it in memory, and if you load the same entity again in the same transaction, you get the exact same Java instance — not a copy, the same object. It also tracks changes you make, so when the transaction commits, Hibernate knows what to update without you calling save explicitly.

**If interviewer goes deeper:**
- "The PC uses snapshots — it saves the state when the entity is loaded and compares on flush. That's why read-only transactions are faster, no snapshots taken."
- "Entities become detached when the transaction ends. If you try to lazy-load something on a detached entity, you get LazyInitializationException because there's no session to fetch the data."

---

#### Lazy Loading

**Short answer:**
Lazy loading means Hibernate doesn't fetch related data until you actually need it. When you load an Order, the Customer field gets a proxy object — looks like a Customer but has no data yet. The first time you call anything other than getId(), Hibernate fires a SQL query to load the real data. This is great for performance, but if you access it outside a transaction, you're in trouble.

**If interviewer goes deeper:**
- "The proxy is a bytecode-generated subclass. It intercepts method calls and checks if data is loaded. If not, it hits the database."
- "That's why equals() can trigger lazy loading — if you compare a proxy to a real object, calling equals() loads the proxy."

---

#### N+1 Problem

**Short answer:**
N+1 happens when you load a list of entities — that's one query — then loop through them accessing lazy associations, causing a separate query for each item. So 100 orders become 101 database hits. You fix it by telling Hibernate to fetch the associations upfront with a join, or by loading them in batches instead of one by one.

**If interviewer goes deeper:**
- "JOIN FETCH works but breaks pagination because you're joining to a one-to-many — you get multiple rows per parent, so LIMIT applies to rows, not entities."
- "I usually do two queries: first get the IDs with pagination, then load full data with joins for just those IDs."

---

#### @Transactional Mechanics

**Short answer:**
Spring wraps your bean in a proxy. When a method marked @Transactional is called through that proxy, Spring starts a transaction, binds an EntityManager to the thread, executes your method, then commits or rolls back. The key thing people miss is that if you call another transactional method internally using `this`, you bypass the proxy entirely — the transaction doesn't propagate because Spring never intercepts the call.

**If interviewer goes deeper:**
- "Solutions are either injecting the service into itself or moving the method to a different bean so the call goes through Spring's proxy."
- "For transactions that must run independently, I use REQUIRES_NEW — but you have to understand the parent transaction gets suspended."

---

#### Flush vs Commit

**Short answer:**
Flush is Hibernate sending your pending changes to the database as SQL, but the transaction is still open. Commit is when you actually end the transaction and make those changes permanent. Hibernate auto-flushes before queries so the database sees current state, and definitely before commit. But if you're mixing JPA with native SQL, you might need to flush manually — native queries don't trigger auto-flush.

**If interviewer goes deeper:**
- "I use manual flush when I need to run a native query that depends on changes I've made through JPA in the same transaction."
- "Flush can fail with constraint violations — that's still before commit, so you can catch and handle it."

---

#### save() vs persist() vs merge()

**Short answer:**
`persist()` is pure JPA for new entities only — the ID is assigned on flush, not immediately. `save()` is Spring Data convenience that gives you the ID right away, even if it has to flush early. `merge()` is for detached entities — you pass in a detached object, Hibernate creates a new managed copy, merges your changes, and returns that new instance. The original stays detached.

**If interviewer goes deeper:**
- "The merge behavior trips people up — they think they're modifying the returned entity, but they modified the original detached one, which isn't tracked."
- "I prefer explicit persist/merge in complex transactional code because it makes intent clearer."

---

#### equals() and hashCode()

**Short answer:**
Never use the auto-generated database ID in equals or hashCode. The ID is null when the entity is created, then gets assigned when you save. If you put that entity in a HashSet and then save it, the hashCode changes while the object is in the set — the set breaks and can't find your entity anymore. Use a business key that's assigned at creation and never changes, like an order number or UUID.

**If interviewer goes deeper:**
- "The business key approach works because it exists from object creation. I use UUIDs generated in the constructor or natural keys like email for User entities."
- "For entities without natural keys, I sometimes use a version field plus a manually assigned UUID."

---

#### Open Session In View

**Short answer:**
OSIV keeps your database connection open for the entire HTTP request — including while the view is rendering. If your template triggers lazy loading, you're making database calls during view rendering. That's bad because rendering is CPU work, not IO work, so you're holding a connection from the pool while doing CPU-intensive work. I've seen this exhaust connection pools under load. I disable it and fetch what I need in the service layer.

**If interviewer goes deeper:**
- "The hidden N+1 is the worst part — you think your queries are fine, but the view is firing queries you didn't write."
- "In microservices, this is even worse because you're holding DB resources while potentially making HTTP calls to other services."

---

### Bad Answer vs Good Answer

| Topic | Weak Candidate | Strong Candidate |
|-------|----------------|------------------|
| **Lazy Loading** | "It loads data when you need it" | "Hibernate creates a proxy subclass that intercepts method calls. Accessing non-ID fields triggers a SQL query. Works great until you access it outside a transaction — then you get LazyInitializationException." |
| **N+1** | "It's when you have too many queries" | "You load N entities with one query, then accessing lazy associations triggers N more queries. I fix it with JOIN FETCH for small lists or batch fetching for paginated data — each has trade-offs with memory and complexity." |
| **@Transactional** | "It makes methods transactional" | "Spring uses a proxy. When called through the proxy, it starts a transaction. But internal `this` calls bypass the proxy — that's the self-invocation problem. I either self-inject or refactor to another bean." |
| **equals/hashCode** | "Use the ID field" | "Never use auto-generated ID — it changes during persist and corrupts HashMaps. I use an immutable business key assigned at creation, like orderNumber or a UUID." |
| **Flush** | "It saves to the database" | "Flush executes SQL within the transaction but doesn't commit. Happens auto before queries and at commit. Native queries don't trigger it, so I flush manually when mixing JPA and JDBC." |

---

### Common Interviewer Follow-ups

After hearing your answer, expect these:

**On Lazy Loading:**
- "What's in the proxy object initially?"
- "How do you fix LazyInitializationException in a web app?"

**On N+1:**
- "Why does JOIN FETCH break pagination?"
- "What's the memory impact of JOIN FETCH on large collections?"

**On Transactions:**
- "What's the difference between REQUIRED and REQUIRES_NEW?"
- "What happens if a REQUIRES_NEW method fails?"

**On Persistence Context:**
- "Why is readOnly = true faster?"
- "What's the identity guarantee in PC?"

**On equals/hashCode:**
- "What if I have no business key?"
- "What happens to a HashMap if hashCode changes?"

**On OSIV:**
- "How do you handle lazy loading in REST APIs without OSIV?"
- "What's the connection pool impact?"

---

### 1-Minute Summary Answers

**When asked "Explain Spring Data JPA":**
> Spring Data generates proxy implementations at runtime — SimpleJpaRepository for CRUD, custom query methods get parsed into JPQL. Hibernate handles the actual SQL and manages the Persistence Context, which is a per-transaction cache that tracks entity changes. Lazy loading uses proxies to defer queries until data is accessed. The main pitfalls are N+1 from unoptimized lazy loading, self-invocation breaking transactions, and wrong equals/hashCode implementations corrupting collections.

**When asked "N+1 and how to fix":**
> N+1 is one query for the list, then N queries for lazy associations. Fixes: JOIN FETCH loads in one query but breaks pagination and creates duplicates for collections. Batch loading with @BatchSize reduces to N/batch queries and works with pagination. Best for paginated APIs: two queries — paginate IDs first, then fetch full data for just those IDs.

**When asked "@Transactional internals":**
> Spring creates a proxy around your bean. Calls through the proxy start a transaction, bind an EntityManager, run your code, then commit. Internal `this` calls bypass the proxy entirely, so @Transactional doesn't work. Fix by self-injecting or moving the method to another bean. REQUIRES_NEW suspends the current transaction and starts a new one — useful when you need something to commit regardless of parent failure.

**When asked "Lazy loading":**
> Hibernate creates a bytecode proxy that looks like your entity but has no data. When you call any method other than getId(), it executes the SQL to load data. Great for performance, but accessing it outside a transaction throws LazyInitializationException. In REST APIs, I disable OSIV and use JOIN FETCH or DTO projections to load what I need upfront.

---

### Quick Confidence Builders

Phrases that show experience without bragging:

- "In production, I've seen this cause..."
- "The trade-off is..."
- "The fix depends on your access pattern..."
- "I prefer X over Y because..."
- "The edge case people miss is..."
- "In my current project, we handle this by..."

Things that signal weakness (avoid):

- "I think it's..."
- "Maybe..."
- "I'm not sure but..."
- Long pauses on basic concepts
- "You can just..."
