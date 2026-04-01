# Spring & Hibernate — Questions 30–52

This file contains interview-ready answers in English for quick preparation.

## Table of Contents

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
41. [How to resolve cyclic bean dependencies?](#q41)
42. [How to inject a Prototype bean to a Singleton?](#q42)
43. [What is the MVC pattern? Which components MVC consists of?](#q43)
44. [What is the Dispatcher Servlet?](#q44)
45. [What is ModelAndView?](#q45)
46. [How is request routing done in Spring MVC?](#q46)
47. [What is the difference between Controller and RestController annotations?](#q47)
48. [What is the difference between RequestParam and PathVariable annotations?](#q48)
49. [How to handle exceptions in Spring MVC?](#q49)
50. [What is a ViewResolver?](#q50)
51. [How to upload files using Spring MVC?](#q51)
52. [What is Thymeleaf?](#q52)

---

<a id="q30"></a>
## 30. What principles Spring based on?

Spring is built on a few core ideas:
- **IoC (Inversion of Control)**: object creation and wiring are managed by the container, not by your business code.
- **DI (Dependency Injection)**: dependencies are provided from outside, which reduces coupling and improves testability.
- **Programming to interfaces**: encourages clean architecture and replaceable implementations.
- **Separation of concerns**: web, service, persistence, security are split into layers.
- **AOP**: cross-cutting concerns (transactions, logging, security) are applied without polluting business logic.
- **Convention over configuration** (especially in Spring Boot): sensible defaults with override capability.

In interview wording: Spring helps build maintainable enterprise apps by combining IoC/DI + modular abstractions + declarative infrastructure.

---

<a id="q31"></a>
## 31. What are the types of DI?

Three common types:
1. **Constructor injection** (recommended)
   - Dependencies are required at object creation.
   - Works best with immutable fields (`final`), easier testing.
2. **Setter injection**
   - Useful for optional dependencies.
   - Object can exist in partially initialized state if misused.
3. **Field injection** (generally discouraged)
   - Easy to write but poor for tests, immutability, and explicit design.

Best practice: use constructor injection by default, setter only for optional collaborators.

---

<a id="q32"></a>
## 32. What are Bean scopes in Spring?

- **singleton** (default): one bean instance per Spring container.
- **prototype**: new instance each time requested.
- **request** (web): one instance per HTTP request.
- **session** (web): one per HTTP session.
- **application** (web): one per ServletContext.
- **websocket**: one per WebSocket session.

Key interview point: singleton scope means one Spring-managed instance, not JVM-wide singleton.

---

<a id="q33"></a>
## 33. What is the Bean lifecycle?

Typical lifecycle:
1. Bean definition loaded.
2. Bean instantiated (constructor/factory method).
3. Dependencies injected.
4. Aware callbacks (`BeanNameAware`, `ApplicationContextAware`, etc.).
5. `BeanPostProcessor#postProcessBeforeInitialization`.
6. Init callbacks:
   - `@PostConstruct`
   - `InitializingBean#afterPropertiesSet`
   - custom `initMethod`
7. `BeanPostProcessor#postProcessAfterInitialization` (often where proxies are created).
8. Bean ready for use.
9. On context shutdown:
   - `@PreDestroy`
   - `DisposableBean#destroy`
   - custom `destroyMethod`

Important: prototype beans usually do not get destroy callback automatically.

---

<a id="q34"></a>
## 34. What is AOP?

**AOP (Aspect-Oriented Programming)** is a way to modularize cross-cutting concerns:
- logging
- transactions
- security checks
- metrics

Core terms:
- **Aspect**: module with cross-cutting logic.
- **Advice**: code to run (`@Before`, `@After`, `@Around`, etc.).
- **Pointcut**: expression that selects join points.
- **Join point**: method execution point where advice applies.

In Spring AOP, interception is mostly method-level via proxies.

---

<a id="q35"></a>
## 35. What types of Proxies do we have in Spring?

Spring uses:
1. **JDK dynamic proxies**
   - Works when target implements an interface.
   - Proxy type is interface-based.
2. **CGLIB proxies** (class-based subclassing)
   - Used when no interface is available or forced.
   - Cannot proxy final classes/methods well.

In modern Spring Boot, class-based proxying is common in many setups.

Interview trap: self-invocation (`this.someTransactionalMethod()`) bypasses proxy, so advice may not run.

---

<a id="q36"></a>
## 36. What does Transactional annotation do?

`@Transactional` declares transaction boundaries around method execution.

What it does:
- opens transaction before method (via proxy/AOP),
- commits on success,
- rolls back on runtime exceptions by default.

Important options:
- `propagation` (REQUIRED, REQUIRES_NEW, etc.),
- `isolation`,
- `readOnly`,
- `timeout`,
- `rollbackFor`.

Important caveats:
- works on proxied method calls (external call through bean proxy),
- default rollback is for unchecked exceptions (`RuntimeException`, `Error`).

---

<a id="q37"></a>
## 37. What is ORM?

**ORM (Object-Relational Mapping)** maps object model to relational tables so developers work with objects rather than raw SQL everywhere.

Benefits:
- less boilerplate CRUD,
- mapping associations (`@OneToMany`, etc.),
- dirty checking and unit-of-work patterns.

Costs:
- abstraction leaks,
- hidden SQL/performance pitfalls,
- need to understand generated queries.

In Java, common stack: JPA API + Hibernate implementation.

---

<a id="q38"></a>
## 38. How to connect the class to the table?

Using JPA annotations:
- `@Entity` on class,
- `@Table(name = "...")` optional,
- `@Id` for PK,
- `@GeneratedValue` for ID strategy,
- `@Column` for column mapping,
- relation annotations (`@OneToMany`, `@ManyToOne`, `@JoinColumn`, etc.).

Example:
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

---

<a id="q39"></a>
## 39. What is the N+1 problem in Hibernate?

N+1 means:
1 query loads parent list (N rows), then
N extra queries load child relation for each row.

Example: fetch 100 orders, then lazy-load customer for each order -> 101 queries total.

Why bad:
- many DB round-trips,
- latency explosion.

How to fix:
- `JOIN FETCH` in JPQL,
- `@EntityGraph`,
- batch fetching (`hibernate.default_batch_fetch_size`),
- DTO projections for read models.

---

<a id="q40"></a>
## 40. What Cache levels do you know in Hibernate?

1. **First-level cache (L1)**
   - bound to Session/EntityManager,
   - always enabled,
   - identity map for managed entities.

2. **Second-level cache (L2)**
   - shared across sessions,
   - optional, provider-backed (Ehcache, Caffeine/JCache, Infinispan, etc.).

3. **Query cache** (optional)
   - caches query result IDs/scalars,
   - often used with L2 for entities.

Key point: L1 is mandatory per persistence context; L2/query are optional and need tuning.

---

<a id="q41"></a>
## 41. How to resolve cyclic bean dependencies?

Preferred solution: **remove cycle by design**.
- Extract shared logic into third service.
- Depend on interface + event/callback pattern.

If unavoidable:
- Use `@Lazy` on one dependency.
- Use setter injection on one side (constructor cycle cannot be resolved cleanly without lazy/proxy tricks).
- Use `ObjectProvider<T>` for delayed lookup.

Note: Spring Boot disallows circular references by default in newer versions unless explicitly enabled.

---

<a id="q42"></a>
## 42. How to inject a Prototype bean to a Singleton?

If you inject prototype directly into singleton, it is created once at singleton construction.

To get fresh instance each time:
- inject `ObjectProvider<PrototypeBean>` and call `getObject()`,
- or use `javax.inject.Provider<PrototypeBean>`,
- or `@Lookup` method injection.

Example:
```java
@Component
class SingletonService {
    private final ObjectProvider<TaskContext> provider;
    SingletonService(ObjectProvider<TaskContext> provider) { this.provider = provider; }

    void process() {
        TaskContext ctx = provider.getObject(); // new prototype instance
    }
}
```

---

<a id="q43"></a>
## 43. What is the MVC pattern? Which components MVC consists of?

MVC separates concerns:
- **Model**: business data/state,
- **View**: rendering/UI,
- **Controller**: request handling and orchestration.

In Spring MVC:
- Controller receives HTTP request,
- interacts with service/model,
- returns view name or response body,
- View renders output (HTML/JSON depending on controller style).

---

<a id="q44"></a>
## 44. What is the Dispatcher Servlet?

`DispatcherServlet` is the **front controller** in Spring MVC.

Responsibilities:
- receives incoming HTTP requests,
- finds matching handler (controller method),
- delegates parameter binding/validation/conversion,
- invokes handler,
- resolves view or writes response body,
- applies interceptors and exception resolvers.

It centralizes web request workflow.

---

<a id="q45"></a>
## 45. What is ModelAndView?

`ModelAndView` is a holder of:
- **model** (attributes for rendering),
- **view** (view name or View object).

Used in classic MVC controllers that render HTML templates.

Example:
```java
return new ModelAndView("orders/list")
        .addObject("orders", orders);
```

In REST APIs, usually not used; we return DTO/body directly.

---

<a id="q46"></a>
## 46. How is request routing done in Spring MVC?

Routing is annotation-based:
- class-level `@RequestMapping`,
- method-level `@GetMapping`, `@PostMapping`, etc.

Spring builds handler mappings at startup.
At runtime:
1. DispatcherServlet gets request.
2. Finds matching mapping by path + HTTP method + params/headers/consumes/produces.
3. Invokes selected controller method.

If no match -> 404 (or method mismatch -> 405).

---

<a id="q47"></a>
## 47. What is the difference between Controller and RestController annotations?

- `@Controller`:
  - used for MVC views (templates),
  - methods usually return view names unless `@ResponseBody` is added.

- `@RestController`:
  - shorthand for `@Controller + @ResponseBody`,
  - return values are serialized to HTTP body (JSON/XML), not treated as view names.

Use `@RestController` for REST APIs.

---

<a id="q48"></a>
## 48. What is the difference between RequestParam and PathVariable annotations?

- `@RequestParam`:
  - gets value from query string or form params,
  - example: `/orders?status=NEW`.

- `@PathVariable`:
  - gets value from URI path template,
  - example: `/orders/{id}` -> `/orders/42`.

Rule of thumb:
- resource identity -> path variable,
- filtering/sorting/paging options -> request params.

---

<a id="q49"></a>
## 49. How to handle exceptions in Spring MVC?

Main options:
1. `@ExceptionHandler` in controller (local).
2. `@ControllerAdvice` / `@RestControllerAdvice` (global).
3. Throw `ResponseStatusException`.
4. Custom exception + `@ResponseStatus`.
5. Override `ResponseEntityExceptionHandler` for framework exceptions.

Best practice for REST:
- centralize handling in `@RestControllerAdvice`,
- return consistent error DTO (code, message, path, timestamp, traceId).

---

<a id="q50"></a>
## 50. What is a ViewResolver?

`ViewResolver` maps a logical view name to a concrete View implementation/template.

Example:
- controller returns `"orders/list"`,
- resolver maps it to `/templates/orders/list.html` (ThymeleafViewResolver) or JSP path.

In REST-only applications, ViewResolver is usually irrelevant because response body is written directly.

---

<a id="q51"></a>
## 51. How to upload files using Spring MVC?

Use `MultipartFile` with multipart/form-data.

Steps:
1. Enable multipart support (Spring Boot usually auto-configures with servlet multipart).
2. Create endpoint:
```java
@PostMapping("/upload")
public ResponseEntity<String> upload(@RequestParam("file") MultipartFile file) {
    // validate size/type, scan if needed, store safely
    return ResponseEntity.ok("uploaded");
}
```
3. Validate:
   - max size,
   - MIME/type whitelist,
   - file name sanitization.
4. Store in safe location/object storage (S3, MinIO, etc.), not arbitrary filesystem path from user input.

Security notes:
- never trust original filename,
- protect against path traversal,
- consider antivirus scanning for untrusted uploads.

---

<a id="q52"></a>
## 52. What is Thymeleaf?

**Thymeleaf** is a server-side Java template engine (commonly used with Spring MVC) for generating HTML.

Features:
- natural templates (HTML works in browser and as template),
- Spring integration (`th:each`, `th:text`, `th:if`, form binding),
- layout dialect support.

Use cases:
- server-rendered web pages (admin panels, internal tools, MVC apps).

Not needed for pure JSON REST APIs.

