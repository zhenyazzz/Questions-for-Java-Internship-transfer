# Module 3 — Spring (Core + Hibernate) — Questions 30–42

> Questions 30–42 from *Questions for Java Internship transfer.pdf* (Spring IoC/DI, bean lifecycle, AOP and proxies, transactions, JPA/Hibernate, caching, advanced cycles and prototype). See also: `other/spring-questions-30-52.md` (questions 43+ for Spring MVC).

## Table of contents

30. [What principles Spring based on?](#q30)
31. [What are the types of DI?](#q31)
32. [What are Bean scopes in Spring?](#q32)
33. [What is the Bean lifecycle?](#q33)
34. [What is AOP?](#q34)
35. [What types of Proxies do we have in Spring?](#q35)
36. [What does Transactional annotation do?](#q36)
37. [What is ORM?](#q37)
38. [How to connect the class to the table?](#q38)
39. [What is the N+1 problem in Hibernate?](#q39)
40. [What Cache levels do you know in Hibernate?](#q40)
41. [How to resolve cyclic bean dependencies? *](#q41)
42. [How to inject a Prototype bean to a Singleton? *](#q42)

**Additional topics (JPA/Hibernate):**

- [Entity lifecycle](#entity-lifecycle)
- [Dirty checking and flush](#dirty-checking-flush)

---

<a id="q30"></a>

## 30. What principles Spring based on?

### Question Restatement
What **design principles** does the Spring Framework emphasize, and how do they show up in everyday applications?

### 1. Definition
Spring is not a single “pattern”; it is a **modular framework** built around **Inversion of Control (IoC)** and **Dependency Injection (DI)**, with strong support for **AOP**, **portable abstractions** (transactions, JDBC, messaging), and **convention-over-configuration** (especially in Spring Boot).

### 2. Core principles

| Idea | What it means in practice |
|------|---------------------------|
| **IoC** | Object graphs are assembled by the **container**, not by `new` in domain code. |
| **DI** | Dependencies are **injected** (constructor preferred), improving testability and loose coupling. |
| **Programming to interfaces** | Easier to swap implementations (repos, clients, strategies). |
| **Separation of concerns** | Web, service, persistence, security are **layers** with clear boundaries. |
| **AOP** | Cross-cutting behavior (transactions, security, logging) stays **out of** business methods. |
| **Non-invasive** | POJOs + annotations/XML; minimal inheritance from framework types. |

### 3. Trade-offs
- **Pros**: maintainable enterprise structure, testability, ecosystem integration.
- **Cons**: “magic” (proxies, auto-config) can hide behavior until you read how the container works.

### 4. Common pitfalls
- Treating Spring as “just annotations” without understanding **proxy-based** AOP and **bean scopes**.

### 5. Summary (Short speaking answer)
Spring centers on **IoC/DI** so components stay loosely coupled and testable; it adds **AOP** for cross-cutting concerns and **modular infrastructure** (transactions, data access, etc.). Spring Boot layers **sensible defaults** on top while remaining override-friendly.

In Spring, Inversion of Control (IoC) means that the framework is responsible for creating, configuring, and managing objects (beans), instead of the developer doing it manually.

This is achieved through Dependency Injection (DI), where dependencies are provided to a class by the container, typically via constructor injection.

IoC helps reduce tight coupling between components, improves testability, and allows better control over the application lifecycle.


**SHORT (30 sec answer)**  
Spring is built on **IoC** and **DI** (container-managed wiring), encourages **interfaces and layered architecture**, uses **AOP** for cross-cutting concerns, and favors **explicit configuration with strong defaults** in Boot.

---

<a id="q31"></a>

## 31. What are the types of DI?

### Question Restatement
Which **injection styles** does Spring support, and which should you prefer in production code?

### 1. Definition
**Dependency Injection** means dependencies are supplied by the framework rather than created inside the class.

### 2. Types in Spring

| Style | Mechanism | Typical use |
|-------|-----------|-------------|
| **Constructor injection** | `final` fields via constructor | **Default best practice** — required deps, immutability, clear graph. |
| **Setter injection** | `@Autowired` on setters | Optional / mutable collaborators; easier to misconfigure partial state. |
| **Field injection** | `@Autowired` on fields | Quick to write; **worst for tests** and clarity (hidden dependencies). |

### 3. Why constructor injection wins
- Dependencies are **explicit** in the signature.
- Works naturally with **`final`** fields.
- Fails fast if the graph is incomplete.
- No reflection-heavy test setup compared to field injection.

### 4. Common pitfalls
- **Circular dependencies** with **constructor-only** injection: Spring cannot instantiate either bean (see Q41).
- Over-using **optional** setter injection where a dependency is actually required.

### 5. Summary (Short speaking answer)
Spring supports **constructor, setter, and field** injection. Prefer **constructor injection** for required dependencies; use **setter** sparingly for optional ones; avoid **field injection** in serious code because it harms testability and hides coupling.

**SHORT (30 sec answer)**  
**Constructor** (preferred), **setter** (optional deps), **field** (discouraged). Constructor makes dependencies explicit and enables immutable components.

---

<a id="q32"></a>

## 32. What are Bean scopes in Spring?

### Question Restatement
What **scopes** exist for Spring beans, and what is the lifecycle/visibility of each?

### 1. Definition
**Scope** controls **how many instances** of a bean definition exist in the container and **how long** each instance lives.

### 2. Scopes

| Scope | Approximate lifetime | Notes |
|-------|----------------------|-------|
| **singleton** (default) | One instance **per IoC container** | Not a JVM global singleton — one per context. |
| **prototype** | **New instance** every time bean is requested | Container does **not** manage full destruction lifecycle like singletons. |
| **request** | One per **HTTP request** (web) | Requires web context. |
| **session** | One per **HTTP session** (web) | |
| **application** | One per **ServletContext** | |
| **websocket** | One per **WebSocket session** | |

### 3. Important nuance
**Singleton** + mutable state = **thread-safety** concerns under concurrent HTTP traffic; prefer stateless services or narrow-scoped beans.

### 4. Common pitfalls
- Confusing **Spring singleton** with **Gang of Four singleton pattern**.
- Injecting **prototype** into **singleton** without `ObjectProvider` / `@Lookup` → you still get **one** prototype instance (see Q42).

### 5. Summary (Short speaking answer)

In Spring, bean scopes define the lifecycle and visibility of a bean instance within the container.

The default scope is **singleton**, where a single instance of the bean is created per Spring container and reused everywhere. This is the most commonly used scope and is suitable for stateless services.

The **prototype** scope creates a new instance every time the bean is requested from the container. Spring does not manage the full lifecycle of prototype beans (e.g., destruction callbacks are not handled).

In web applications, there are additional scopes:

* **request** — a new bean instance is created for each HTTP request
* **session** — one bean instance per HTTP session
* **application** — one bean per ServletContext (shared across the whole application)
* **websocket** — one bean per WebSocket session

Key difference: singleton is shared and managed for the entire lifecycle, while prototype and web scopes create multiple instances tied to specific contexts.

In practice, most beans are singleton, and other scopes are used only when you need state tied to a request, session, or per-use instance.

---

<a id="q33"></a>

## 33. What is the Bean lifecycle?

### Question Restatement
What **phases** does a Spring-managed bean go through from creation to destruction?

### 1. Definition
The **bean lifecycle** has two layers that interviews often conflate:

1. **Container / definition phase** — metadata is loaded and **bean definitions** can still be changed **before any** regular bean instance exists.
2. **Bean instance phase** — a specific bean is **instantiated**, **injected**, **initialized**, optionally **wrapped** (e.g. AOP proxy), then later **destroyed** (for singletons).

If you only describe `BeanPostProcessor`, you are describing **instance** hooks. **`BeanFactoryPostProcessor`** belongs to the **first** layer.

---

### 2. Phase A — Definitions (no bean instance yet)

| Mechanism | What it operates on | What it is for |
|-----------|---------------------|----------------|
| **`BeanDefinitionRegistryPostProcessor`** | `BeanDefinitionRegistry` | Register or remove **bean definitions** early (e.g. `@Configuration` parsing via `ConfigurationClassPostProcessor`). |
| **`BeanFactoryPostProcessor`** | `ConfigurableListableBeanFactory` | **Mutate bean definitions** or factory metadata **before** instantiation: placeholders (`${…}`), custom scopes, `BeanDefinition` properties, etc. |

**Order matters:** `BeanDefinitionRegistryPostProcessor` beans run before other `BeanFactoryPostProcessor` beans. These processors are themselves beans, but they are **created and invoked in a dedicated bootstrap path** so they can prepare the factory for everyone else.

**Why it matters:** Without this phase, there is nothing coherent to say about “lifecycle” in Spring Boot / `@Configuration` — most of your graph is assembled here as **definitions**, not as live objects.

---

### 3. Phase B — One bean instance (`doCreateBean` — conceptual order)

After definitions are ready, the container creates each bean (singletons are pre-instantiated during `refresh` unless lazy). Typical pipeline:

1. **Instantiation** — constructor or `@Bean` factory method; `Supplier` / instance supplier if used.
2. **Merged definition tweaks** — `MergedBeanDefinitionPostProcessor` can post-process the **merged** `BeanDefinition` for that bean (metadata used for injection and init).
3. **Populate / dependency injection** — properties, `@Autowired`, `@Value` resolution (after placeholders were wired in Phase A).
4. **Aware callbacks** (if implemented) — e.g. `BeanNameAware`, `BeanClassLoaderAware`, `BeanFactoryAware`; in an `ApplicationContext`, `ApplicationContextAware` is handled via `ApplicationContextAwareProcessor`.
5. **`BeanPostProcessor#postProcessBeforeInitialization`** — includes infrastructure like **`@PostConstruct`** (via `InitDestroyAnnotationBeanPostProcessor` / `CommonAnnotationBeanPostProcessor`), not only “custom” BPPs.
6. **Initialization contracts** (after `@PostConstruct` in the usual setup):
   - `InitializingBean#afterPropertiesSet`
   - custom `init-method` from XML or `@Bean(initMethod = …)`
7. **`BeanPostProcessor#postProcessAfterInitialization`** — **wrapping**: **AOP proxies**, `@Transactional`, `@Async`, `@Cacheable`, etc. The **final** bean reference the context exposes is often the **proxy** returned here.
8. **In use** — business calls go through the proxy where applicable.

**Advanced (optional but fair):** `InstantiationAwareBeanPostProcessor` can **short-circuit** instantiation (`postProcessBeforeInstantiation` returning a non-null bean) or run logic **after** instantiation and **before** property population — used by frameworks, rarely by application code.

---

### 4. Phase C — Shutdown (singletons)

On context close, **singleton** beans receive destruction in reverse dependency order where possible:

- `@PreDestroy`
- `DisposableBean#destroy`
- custom `destroy-method`

Implemented via `DisposableBeanAdapter` and related infrastructure.

---

### 5. Important caveats

- **Prototype** scope: Spring **does not** run the full **destruction** protocol for instances it hands out — **you** own cleanup (close resources, pools, etc.).
- **`BeanPostProcessor` and `BeanFactoryPostProcessor` beans** are **special**: they must be registered early; ordering uses `Ordered` / `PriorityOrdered` — wrong order breaks other beans.
- **Circular dependencies** with **constructor** injection are **not** solved by lifecycle callbacks; that is a **graph** problem (`@Lazy`, redesign).

---

### 6. Interview hooks

- **“Where do placeholders get resolved?”** — `BeanFactoryPostProcessor` (e.g. property sources), not `BeanPostProcessor`.
- **“Where do proxies appear?”** — almost always **`postProcessAfterInitialization`**, not at raw construction.
- **“What runs first — BFP or BPP?”** — **All relevant `BeanFactoryPostProcessor` invocations complete before** normal beans are instantiated; **`BeanPostProcessor`** runs **per bean instance** during initialization.

---

### 7. Summary (Short speaking answer)

**Definitions first:** `BeanDefinitionRegistryPostProcessor` and **`BeanFactoryPostProcessor`** adjust **bean definitions** and factory setup **before** instances exist (configuration parsing, placeholders, definition tweaks). **Then** each bean is **created → injected → Aware → `BeanPostProcessor` before init** (including **`@PostConstruct`**) → **`InitializingBean` / custom init** → **`BeanPostProcessor` after init** (proxies). On shutdown, **singletons** get **`@PreDestroy` / `DisposableBean` / destroy-method**; **prototypes** are not fully managed on destruction.

**SHORT (30 sec answer)**  
**BFP / registry post-processors** mutate **definitions** before beans exist → then per bean: **instantiate → inject → Aware → BPP before init (`@PostConstruct` here) → `afterPropertiesSet` / init-method → BPP after init (proxies)** → **destroy** callbacks for **singletons** on context close; **prototypes** are not destroyed by the container.

---

<a id="q34"></a>

## 34. What is AOP?

### Question Restatement
What is **Aspect-Oriented Programming**, and how does Spring implement it?

### 1. Definition
**AOP** modularizes **cross-cutting concerns** (logging, security, transactions, metrics) that would otherwise be **duplicated** across many methods.

### 2. Core vocabulary

| Term | Meaning |
|------|---------|
| **Aspect** | Module encapsulating cross-cutting behavior. |
| **Join point** | A point in program execution (in Spring AOP: typically **method execution**). |
| **Pointcut** | Predicate selecting join points (e.g. “all service methods”). |
| **Advice** | Action at a join point: `@Before`, `@After`, `@Around`, etc. |
| **Weaving** | Applying aspects — Spring uses **runtime** proxies. |

### 3. Spring AOP limitations (typical interview honesty)
- Method-level on **Spring beans** through **proxies** — not a full bytecode weaving solution like AspectJ compile-time for all `new` calls.

### 4. Common pitfalls
- **Self-invocation** bypasses proxy → `@Transactional`/`@Cacheable` may not apply (call through another bean or refactor).

### 5. Summary (Short speaking answer)
AOP separates **cross-cutting** behavior into **aspects** applied via **proxies** around Spring beans. You define **pointcuts** + **advice**. Spring AOP is **proxy-based** and mostly **method interception**.

**SHORT (30 sec answer)**  
AOP factors out cross-cutting logic into **aspects** with **pointcuts** and **advice**. Spring applies it at **runtime** via **proxies** around `@Component` beans.

---

<a id="q35"></a>

## 35. What types of Proxies do we have in Spring?

### Question Restatement
When does Spring create a **JDK dynamic proxy** vs **CGLIB**, and what are the implications?

### 1. Definition
Spring wraps beans with **proxies** to implement **AOP** (transactions, security, async, etc.).

### 2. Two mechanisms

| Proxy type | How it works | When used |
|------------|--------------|-----------|
| **JDK dynamic proxy** | Implements **interfaces** at runtime | Target implements interfaces; proxy is **interface-only**. |
| **CGLIB** | Subclasses **concrete class** (code generation) | No interface proxying needed, or class-based proxying selected / required. |

### 3. Practical implications
- **JDK proxies**: cannot cast to concrete class — inject **interface**.
- **CGLIB**: cannot proxy **final** classes/methods; subclassing limitations.
- **Self-invocation**: internal `this.method()` does not go through proxy.

### 4. Spring Boot default
Historically moved toward **class-based proxies** in many setups (`spring.aop.proxy-target-class=true` is common) — **verify** in your version/config.

### 5. Summary (Short speaking answer)
Spring uses **JDK dynamic proxies** for interface-based targets and **CGLIB** subclass proxies for classes. Limitations: **final** methods, **self-invocation**, and **casting** expectations in JDK proxy mode.

**SHORT (30 sec answer)**  
**JDK dynamic proxies** for interfaces; **CGLIB** subclasses concrete classes. Framework features like `@Transactional` rely on these proxies — so **external calls** through the bean matter.

---

<a id="q36"></a>

## 36. What does Transactional annotation do?

### Question Restatement
What happens when a method is annotated with `@Transactional`, and what are the default rollback rules and common traps?

### 1. Definition
`@Transactional` declares that the method (or class) should run inside a **database transaction** boundary managed by Spring’s **transaction infrastructure** (typically via **AOP proxy**).

### 2. Behavior (typical)
- **Begin** transaction before method (or join existing one depending on **propagation**).
- **Commit** on successful completion.
- **Rollback** on **unchecked** exceptions (`RuntimeException`, `Error`) **by default** — **checked** exceptions do **not** roll back unless configured.

### 3. Important attributes

| Attribute | Purpose |
|-----------|---------|
| `propagation` | e.g. `REQUIRED`, `REQUIRES_NEW`, `NESTED` |
| `isolation` | DB isolation level mapping |
| `readOnly` | Hint for optimization / driver |
| `rollbackFor` / `noRollbackFor` | Exception-based rollback tuning |
| `timeout` | Limit transaction duration |

### 4. Common pitfalls
- **Self-invocation** inside same class → annotation ignored (no proxy).
- **Wrong exception type** → commit on failure (checked exception).
- **Transaction boundaries too large** → long locks, poor throughput.

### 5. Summary (Short speaking answer)
`@Transactional` wraps calls in a **transactional proxy**: commit on success, rollback on **runtime errors** by default. Configure **propagation**, **isolation**, and **rollback rules** explicitly for real systems; watch **proxy** boundaries.

**SHORT (30 sec answer)**  
Spring opens/commits/rolls back a DB transaction around the method via **AOP**. Default rollback on **unchecked** exceptions. Propagation/isolation tune how the method participates in existing work.

---

<a id="q37"></a>

## 37. What is ORM?

### Question Restatement
What is **Object-Relational Mapping**, and why do we use it (and when does it hurt)?

### 1. Definition
**ORM** maps **object graphs** to **relational tables** so application code can work with **entities** instead of hand-written SQL for every CRUD path.

### 2. Benefits
- Less boilerplate for simple persistence paths.
- Navigable **associations** in code (`@ManyToOne`, etc.).
- **Unit of Work** / dirty checking in Hibernate-style providers.

### 3. Costs / risks
- **Leaky abstraction**: SQL still matters for performance.
- **N+1** queries, **excessive fetch** of graphs, **locking** surprises.
- Schema evolution still needs discipline (migrations — Module 2).

### 4. Java ecosystem
**JPA** is the API; **Hibernate** is a common implementation — interviewers often conflate “JPA vs Hibernate”; clarify **API vs provider**.

### 5. Summary (Short speaking answer)
ORM bridges **objects** and **relational** storage, reducing repetitive SQL and modeling associations. Trade-offs: productivity vs **visibility of generated SQL** and **performance tuning** needs.

**SHORT (30 sec answer)**  
ORM maps objects to tables. In Java: **JPA** API, often **Hibernate** underneath. Good for productivity; you must still understand SQL and fetch behavior.

---

<a id="q38"></a>

## 38. How to connect the class to the table?

### Question Restatement
How do you map a **Java class** to a **database table** in JPA/Hibernate?

### 1. Definition
Mark the class as an **entity** and map fields/columns/relations with **annotations** (or XML mapping files).

### 2. Minimal mapping toolkit
- `@Entity` — managed type.
- `@Table(name = "...")` — table name (optional if default matches).
- `@Id`, `@GeneratedValue` — primary key strategy.
- `@Column` — column name, nullability, length, etc.
- Relationship mappings: `@ManyToOne`, `@OneToMany`, `@JoinColumn`, etc.

### 3. Code example
```java
@Entity
@Table(name = "orders")
public class OrderEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "status", nullable = false)
    private String status;
}
```

### 4. Common pitfalls
- **Equals/hashCode** for entities with associations — naive implementations cause subtle bugs in collections and L2 cache.
- **Lazy loading** outside a session → `LazyInitializationException`.

### 5. Summary (Short speaking answer)
Use `@Entity` + `@Table` + field mappings (`@Column`, `@Id`, relations). Prefer explicit naming where defaults are unclear; understand **fetch type** and **transaction boundaries** for lazy loading.

**SHORT (30 sec answer)**  
Annotate the class with `@Entity`, map the table with `@Table`, PK with `@Id`/`@GeneratedValue`, columns with `@Column`, and relations with JPA relationship annotations.

---

<a id="entity-lifecycle"></a>

## Entity lifecycle

Hibernate (and JPA) do **not** treat entities as “plain objects” — they track them in a **persistence context**. In JPA this is the **`EntityManager`** (in Spring you typically inject it or obtain it via `@PersistenceContext` / `EntityManagerFactory`); in “plain” Hibernate the same role is played by **`Session`**. That context **is** the **first-level cache (L1)** and the place where **dirty checking**, **flush**, and **entity state** tracking happen.

### Four states (JPA terminology and Hibernate)

| State | Summary |
|-------|---------|
| **Transient (new)** | Object created with `new`, **not** associated with a persistence context and **not** managed by the ORM. No identifier yet (or not assigned by the DB). No `INSERT` on commit until you call `persist()` / `entityManager.persist()`. |
| **Managed / persistent** | Entity is **registered** in the persistence context: it has an **ID** (or gets one after flush). Hibernate/JPA **track** field changes. **Dirty checking** produces `UPDATE` on `flush`/commit. You do **not** need `save()` for every change to an already-loaded managed entity — mutating fields inside the transaction is enough. |
| **Detached** | Was managed, but the context **closed** (`close()`), or you called **`detach()`**, **`clear()`**, **`evict()`**, etc. Field changes are **not** tracked and will **not** reach the DB until **`merge()`** (or another valid `persist`/`merge` flow per JPA rules). |
| **Removed** | Marked for deletion via **`remove()`** / `session.remove()`. The row is **physically** deleted on **`flush`/commit** (or at the synchronization point your settings imply). |

### Typical transitions (simplified)

- **Transient → Managed:** `persist(entity)` (or `merge`, depending on scenario and id).
- **Managed → Detached:** context/session closed, or `detach`, `clear`, `evict`.
- **Detached → Managed:** `merge(detached)` — returns a **managed** instance/state; the context reconciles with the DB per merge semantics.
- **Managed → Removed:** `remove(managed)`.
- **Removed:** after flush/commit the row is gone from the DB; the in-memory object may still exist, but the entity is deleted in the context.

### Interview points

- The **same id** in one persistence context → **one** instance (identity map / L1).
- **Lazy** associations outside an active session → **`LazyInitializationException`** (classic “detached / no session”).
- Do **not** confuse **entity state** with **Spring bean lifecycle** (question 33) — different abstraction levels.

---

<a id="q39"></a>

## 39. What is the N+1 problem in Hibernate?

### Question Restatement
What does **N+1 queries** mean in ORM, why is it bad, and how do you fix it?

### 1. Definition
**N+1** means: **1** query loads a **set of N root rows** (parents or “main” entities), then **up to N extra round-trips** to load **related** data — most often because a **lazy** association is touched **once per row** (in a loop, during serialization, or inside templating). The classic textbook case is **1 + N** queries; in worse cases you get **1 + N × M** (nested collections) or multiple independent “+1” chains in one request.

The problem is not “joins are bad” — it is **unplanned lazy loading**: the ORM does exactly what you told it (lazy by default on many mappings), and **each access** may become **one SQL statement**.

---

### 2. Concrete pattern (mental model)

**Example:** load 100 `Order` rows, then for each order access `order.getCustomer()` (or a lazy `@ManyToOne`).

1. `SELECT * FROM orders …` → 100 rows. **(1 query)**
2. First time you read `customer` for order #1 → Hibernate loads that customer. **(+1)**
3. Repeat for orders #2…#100 → up to **100 more queries** (often similar `SELECT … FROM customers WHERE id = ?`).

Total: **1 + N** (here **101** queries). With two lazy levels (e.g. order → lines → product), the pattern multiplies.

---

### 3. Why it hurts

- **Latency:** each query is a **network round-trip** to the DB; even fast queries add up (100 × 1 ms ≫ one 5 ms join).
- **CPU / pool:** more statements, more **connection pool** churn and DB parser work.
- **Looks fine in dev:** small tables and local DB hide the issue; **production** traffic and larger `N` expose it sharply.
- **Hard to see in code:** the bug is often **implicit** (getter called from JSON mapper, GraphQL field resolver, or Thymeleaf), not an obvious `for` loop with `getSession().get()`.

---

### 4. Typical triggers (watch these in reviews)

- **Lazy `@ManyToOne` / `@OneToMany`** loaded **per entity** when the association is first read.
- **HTTP APIs:** Jackson / serialization touching lazy fields **after** the service method returned (session closed) — you may get `LazyInitializationException` **or**, if Open Session In View is on, **N+1 in the view layer** (very expensive).
- **GraphQL:** one resolver per field per parent → classic N+1 unless you use **DataLoader**, **batch loaders**, or fetch joins at the root query.
- **`@Transactional` boundary too narrow:** loading parents in one method and accessing children in another without a fetch plan.
- **Query cache without L2** (see question 40): cached query returns **IDs**, then Hibernate loads entities **one by one** → N+1-style behavior.

---

### 5. How to detect

- **SQL logging** in a lower environment: `spring.jpa.show-sql=true` (noisy) or **logging binders** (e.g. P6Spy, datasource-proxy) to **count** statements per request.
- **Hibernate statistics** (`hibernate.generate_statistics`, `SessionFactory.getStatistics()`) — query count per use case.
- **Integration tests** asserting **max number of queries** (libraries that wrap the datasource and count statements).
- **APM / traces** (Jaeger, etc.) showing many similar `SELECT` by primary key in one trace span.

---

### 6. Fixes (toolbox) — in more detail

| Approach | What it does | Trade-off |
|----------|----------------|-----------|
| **`JOIN FETCH` in JPQL/HQL** | Loads root + association in **one** (or fewer) queries with a join. | **Duplicate rows** if you fetch **multiple bags/collections** in one query; cartesian growth. Often pair **one** collection fetch per query or split queries. |
| **`@EntityGraph` / named entity graph** | Declares which attributes to **fetch** for a repository method or query. | Same join semantics as fetch join — mind **cardinality** and duplicates. |
| **`@BatchSize` / `hibernate.default_batch_fetch_size`** | Does **not** remove N+1 logically but collapses **N** statements into **⌈N / batchSize⌉** round-trips (e.g. `WHERE id IN (?,?,?)`). | Still more round-trips than one join; usually **much** better than 1-by-1. |
| **`@Fetch(SUBSELECT)`** (Hibernate) | Uses a **subselect** to load a collection in fewer queries. | Hibernate-specific; understand SQL shape. |
| **DTO / projection (`SELECT new …`, interfaces, Blaze-Persistence, jOOQ)** | Reads **only** columns you need for the screen/report; no lazy graph. | No entity graph for writes; best for **read models**. |
| **Two-step fetch** | Load IDs first, then `WHERE id IN (:ids)` for children — explicit control. | More code; clear and predictable. |
| **`fetch` API in Spring Data JPA** | `JOIN FETCH` via `@Query` or custom methods. | Same as JPQL fetch rules. |

**Pagination warning:** `JOIN FETCH` + **`LIMIT`/`Pageable` on a collection** is tricky: duplicates and wrong totals. Common patterns: fetch **without** collection join + **batch** load children, or **@BatchSize**, or separate queries for page of IDs then batch-load children.

---

### 7. Eager loading is not “the fix” by default

- **`FetchType.EAGER`** on many associations → **unexpected** loads, **cartesian products**, and **harder** N+1 reasoning. Prefer **explicit fetch plans** per use case (`JOIN FETCH`, entity graph, batch size) over **global** eager.

---

### 8. Summary (Short speaking answer)

N+1 is **one query for many roots** plus **extra queries proportional to how often you touch lazy associations** (often **N** times). It is a **round-trip** and **statement count** problem. Fix by **controlling fetches**: **join fetch / entity graphs**, **batch fetching** to group primary-key loads, **DTOs/projections** for reads, and by **avoiding** lazy access in serializers/resolvers without a plan. **Detect** with SQL logging, statistics, or query-count tests.

**SHORT (30 sec answer)**  
Load **N** parents in one query, then **lazy** relations trigger **one query per row** (or per access) → **1+N** (or worse). Fix: **`JOIN FETCH` / `@EntityGraph`**, **`@BatchSize` / default batch fetch size**, **DTO queries**, or **subselect** fetch; watch **pagination + fetch join** and **serialization** touching lazy fields.

---

<a id="q40"></a>

## 40. What Cache levels do you know in Hibernate?

### Question Restatement
What is **first-level** vs **second-level** cache (and **query cache**) in Hibernate?

### 1. Definition
The ORM provides a **multi-level** caching model to reduce database load. Levels **do not** replace each other: each has a different **scope**, **consistency semantics**, and **risks**.

---

### 2. L1 cache (first-level / persistence context)

- **Always on** — in JPA the persistence context is **not** an optional “switch off”: it **is** how entity work is scoped.
- **Lives for one `EntityManager` / `Session`** — in a typical Spring transactional flow, one persistence context per transaction / unit of work with `EntityManager`.
- **Stores entities by identifier** (identity map): a second `find` by the same `id` in the same context **should not** issue another `SELECT` for that entity if it is already loaded and not evicted.
- **Tied to dirty checking:** the **snapshot** used for comparison at flush lives here too.

**Interview angle:** L1 is not “optimization for speed” first — it is a **model invariant**: one id → one managed instance in the context.

---

### 3. L2 cache (second-level)

- **Off by default** — you must enable it and configure a provider (regions, entities, access strategies).
- **Scope:** **`SessionFactory` / `EntityManagerFactory`** — shared across **all** sessions/threads in the application (unlike L1).
- **Providers:** Ehcache, Infinispan, Hazelcast, **Redis** (via JCache/adapters), Caffeine, etc., depending on version and setup.
- **What is cached:** usually **not** shared mutable live heap objects but **disassembled state** (value arrays) to avoid **one mutable instance** visible across threads and to **reassemble** the entity when loading into a new persistence context.

**Risks:** inconsistency when other nodes/processes update data without **invalidation** / TTL / **concurrency** strategy (read-write, nonstrict-read-write, etc.).

---

### 4. Query cache

- Caches not “raw table rows” but a **reusable representation of JPQL/HQL/Criteria results** — often a **list of entity identifiers** from the query.
- **Why L2 pairing matters:** if the query cache holds only **IDs** and entities are **not** in L2, the next access may hit the DB **once per entity** → a harsh **N+1** instead of a win. Query cache pays off **together with L2** (or when you fully control where rows are loaded from).
- Any data change that affects the query result must **invalidate** the relevant query-cache regions — otherwise you get **stale** results.

**Takeaway:** query cache **speeds up repeated queries**; it does **not** replace a sound fetch plan.

---

### 5. Summary table

| Level | Scope | Enabled? | Purpose |
|-------|-------|----------|---------|
| **L1** | `Session` / persistence context | **Always** (context is mandatory) | Identity map, managed entities, dirty checking. |
| **L2** | `SessionFactory` / shared | **Optional** | Cached state by id across sessions; needs provider and consistency policy. |
| **Query cache** | Shared | **Optional** | Cached query results; often IDs; needs alignment with L2 and invalidation. |

### 6. Consistency warning
**L2/query cache** can show **stale** data if invalidation/TTL strategy is wrong — not “free performance.”

### 7. Summary (Short speaking answer)
**L1** is the mandatory session-scoped context: **one id → one managed instance**; repeat reads should not hit the DB unnecessarily. **L2** is shared **state** across sessions; you need a provider and consistency policy. **Query cache** stores **query results** (often **IDs**); without **L2** and invalidation you easily get **N+1** and stale reads.

**SHORT (30 sec answer)**  
**L1** — always, per `EntityManager`/session. **L2** — optional, factory-wide, provider (Ehcache, Redis, …), **disassembled state**. **Query cache** — optional for JPQL/HQL; often stores **IDs** → **pair with L2**, otherwise **N+1**; needs **invalidation**.

---

<a id="dirty-checking-flush"></a>

## Dirty checking and flush

These two mechanisms complete the story of **managed** entities and **L1**: they explain **when** and **why** `INSERT` / `UPDATE` / `DELETE` reach the database.

### Dirty checking

- Applies only to **managed** entities inside an active persistence context.
- Hibernate keeps an **original snapshot** of state; on **flush** (or at end of transaction depending on mode) it compares **current fields** to that snapshot.
- If fields changed, the entity is **dirty** and an **`UPDATE`** is generated when needed.
- You **do not** need `save()` / `merge()` for a **simple change** to an already-loaded **managed** entity — mutate fields and complete the transaction with a flush (or rely on flush per your `FlushMode`).

**Caveat:** for **detached** instances dirty checking does **not** apply — use **`merge()`** or reload.

### Flush (sync persistence context to the database)

**Flush** is **executing SQL** from the context to the DB: synchronizing **in-memory** persistence context state with the database. Do **not** confuse with **commit**: commit ends the transaction; flush **pushes** SQL **within** that transaction.

**Default `FlushMode.AUTO`:**

- Before running a **query** against the DB (so JPQL/Criteria/native queries **see** changes made earlier in the same session).
- Before **commit** of the transaction.

Other modes (`COMMIT`, `MANUAL`) change **when** flush runs — useful for batches and benchmarks, risky if you do not understand visibility ordering.

### Relation to caching

- **L1** holds managed entities and **snapshots** for dirty checking.
- After flush, **SQL** hits the DB; **L2** (if enabled) updates per second-level **region** and strategy rules.

### Summary (short)

**Dirty checking** detects changes to managed entities and emits `UPDATE` on flush. **Flush** syncs the context to the DB; in **AUTO**, typically before queries and commit. For detached entities, dirty checking does **not** apply without `merge`.

---

<a id="q41"></a>

## 41. How to resolve cyclic bean dependencies? *

### Question Restatement
When **Bean A** depends on **B** and **B** depends on **A**, how does Spring handle it, and what is the **right** fix?

### 1. Definition
A **circular dependency** is a cycle in the dependency graph. **Constructor-only** cycles cannot be resolved by straightforward instantiation order.

### 2. Preferred solution (architecture)
**Remove the cycle**: extract shared logic into a **third** component, or invert dependency via **events**, **callbacks**, or **interfaces** so the graph becomes acyclic.

### 3. Framework-level mitigations (if stuck)
- **`@Lazy`** on one injection point — breaks the cycle by injecting a **lazy proxy** resolved later.
- **Setter injection** on one side (less ideal) — allows partial construction.
- **`ObjectProvider<T>`** / `ApplicationContext.getBean` — **late** lookup (use sparingly).
- Note: newer Spring Boot may **fail fast** on cycles by default — **do not** rely on cycles as a design tool.

### 4. Common pitfalls
- “Fixing” cycles with **more** indirection without addressing **domain modeling**.
- Hidden cycles via **configuration** classes and `@Bean` methods.

### 5. Summary (Short speaking answer)
Best fix: **redesign** to **eliminate** the cycle. If unavoidable, **@Lazy**, **setter** injection, or **ObjectProvider** can break instantiation order — but treat this as a **smell** to refactor away.

**SHORT (30 sec answer)**  
Prefer **refactoring** to remove the cycle. Spring tricks: **`@Lazy`**, one side **setter** injection, **`ObjectProvider`**. Boot may **disallow** cycles — cycles are fragile design.

---

<a id="q42"></a>

## 42. How to inject a Prototype bean to a Singleton? *

### Question Restatement
Why does injecting a **prototype-scoped** bean into a **singleton** give you **only one** instance, and how do you get a **new** prototype per use?

### 1. Definition
A **singleton** is created **once**. Its dependencies are injected **once** at creation — so a **direct** prototype dependency is resolved **once** and reused for the singleton’s lifetime.

### 2. Correct patterns (new instance per call)

| Approach | Idea |
|----------|------|
| **`ObjectProvider<Proto>`** | Call `getObject()` when you need a fresh instance. |
| **`Provider<Proto>`** (JSR-330) | Same delayed resolution pattern. |
| **`@Lookup` method** | Abstract method Spring implements to return new prototype instance. |
| **Scoped proxy** | Inject a proxy that routes to new prototype (advanced; understand semantics). |

### 3. Example (ObjectProvider)
```java
@Component
class SingletonService {
    private final ObjectProvider<PrototypeBean> prototypeProvider;

    SingletonService(ObjectProvider<PrototypeBean> prototypeProvider) {
        this.prototypeProvider = prototypeProvider;
    }

    void handle() {
        PrototypeBean fresh = prototypeProvider.getObject();
        // use fresh instance
    }
}
```

### 4. Common pitfalls
- Expecting **new prototype per HTTP request** without **request scope** or explicit resolution — wrong scope choice.
- Holding prototype with **state** inside singleton fields without synchronization → **race conditions**.

### 5. Summary (Short speaking answer)
Direct injection freezes **one** prototype into the singleton. Use **`ObjectProvider`/`Provider`**, **`@Lookup`**, or **scoped proxies** to obtain **new** prototype instances when needed.

**SHORT (30 sec answer)**  
Singleton wires dependencies once — prototype injected directly is still **one** instance. Use **`ObjectProvider.getObject()`**, **`Provider`**, or **`@Lookup`** to create a **new** prototype per use.
