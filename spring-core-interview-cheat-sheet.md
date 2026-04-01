# Spring Core Interview Cheat Sheet (Middle/Senior)

Deep dive into IoC container, dependency injection, bean lifecycle, and internals.

---

## 1) What is Spring Core

### IoC Container Concept

**Inversion of Control (IoC)** means the framework controls object creation and wiring, not your code. Instead of:

```java
// Without IoC — manual object creation hell
OrderService orderService = new OrderService();
orderService.setPaymentService(new PayPalPaymentService());
orderService.setOrderRepository(new JpaOrderRepository());
// What if I need Stripe instead of PayPal? Change all instantiations!
```

Spring takes over:
```java
// With IoC — declare what you need
@Service
public class OrderService {
    private final PaymentService paymentService;
    
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;  // Spring provides this
    }
}
```

**The Container:** Spring's `ApplicationContext` is a sophisticated factory that:
- Creates objects (beans)
- Wires dependencies
- Manages lifecycle
- Provides hooks for customization

### Why It Exists

**Problem Spring solves:**
1. **Tight coupling** — classes hardcoded their dependencies
2. **Difficult testing** — can't mock dependencies easily
3. **Configuration hell** — changing implementations required code changes
4. **Lifecycle management** — who disposes resources, handles transactions?

**Spring's solution:**
- Loose coupling through interfaces
- Easy testing via dependency injection
- Externalized configuration (annotations/XML)
- Automated lifecycle management

### High-Level Architecture

```
ApplicationContext (IoC Container)
    ├── BeanFactory (core bean creation)
    ├── ResourceLoader (load config files)
    ├── EventPublisher (application events)
    └── MessageSource (i18n)

Your Code
    ← depends on ← ApplicationContext
         creates & wires
    ←────── Beans (your services, repositories)
```

**Interview Answer:**
Spring Core is an IoC container that takes control of object creation and dependency wiring. Instead of classes creating their own dependencies, Spring instantiates beans and injects what they need. This decouples components, makes testing easier with mocks, and externalizes configuration so you can change implementations without changing code.

---

## 2) IoC (Inversion of Control)

### How Spring Takes Control

**Traditional control flow:**
```
Your code → creates objects → calls methods
```

**IoC control flow:**
```
Spring config → ApplicationContext → creates beans → wires dependencies
                                    ↓
                              Your code gets ready-to-use beans
```

**Mechanism:**
1. You define **what** beans should exist (@Component, @Bean)
2. You define **how** they relate (@Autowired, constructor args)
3. Spring decides **when** to create and **how** to wire

### ApplicationContext vs BeanFactory

| Feature | BeanFactory | ApplicationContext |
|---------|-------------|---------------------|
| **What it is** | Basic IoC container | Advanced container with enterprise features |
| **Bean instantiation** | Lazy (on demand) | Eager (pre-instantiates singletons) |
| **Internationalization** | No | Yes (MessageSource) |
| **Event propagation** | No | Yes (ApplicationEvent) |
| **AOP integration** | Manual | Built-in |
| **Message resources** | No | Yes |
| **Bean post-processing** | Manual registration | Automatic |

**Code difference:**
```java
// BeanFactory — low level, rarely used directly
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions("beans.xml");
MyService service = factory.getBean(MyService.class);  // Bean created NOW

// ApplicationContext — what everyone uses
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
// All singletons already created and wired!
MyService service = context.getBean(MyService.class);
```

**Deep Insight:**
ApplicationContext extends BeanFactory and adds:
- Automatic BeanPostProcessor registration
- Automatic BeanFactoryPostProcessor registration
- Resource loading (classpath, file, URL)
- Event publishing
- Message resolution (i18n)

**Interview Answer:**
BeanFactory is the basic IoC container that creates beans on demand. ApplicationContext is a full-featured container that pre-instantiates singletons at startup, handles internationalization, publishes events, and automatically registers post-processors. In practice, everyone uses ApplicationContext or its implementations like AnnotationConfigApplicationContext.

---

## 3) Dependency Injection (DI)

### Constructor Injection (Preferred)

```java
@Service
public class OrderService {
    private final PaymentService paymentService;
    private final OrderRepository orderRepository;
    
    // Dependencies injected at creation — immutable, testable
    public OrderService(PaymentService paymentService, 
                        OrderRepository orderRepository) {
        this.paymentService = paymentService;
        this.orderRepository = orderRepository;
    }
}
```

**Why preferred:**
1. **Immutable dependencies** — fields can be final
2. **Required dependencies** — object can't be created without them
3. **Testable** — easy to pass mocks in constructor
4. **Clean code** — dependencies are explicit

### Field Injection (Why It's Bad)

```java
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService;  // Hidden dependency!
    
    @Autowired
    private OrderRepository orderRepository;  // Can't be final
}
```

**Problems:**
1. **Hidden dependencies** — not visible in API
2. **Can't be immutable** — no final fields
3. **Hard to test** — need reflection or Spring to inject
4. **Violates OOP principles** — object created in incomplete state
5. **Framework coupling** — can't instantiate without Spring

**Common Trap:**
Using `@Autowired` on fields because it's "easier" — leads to untestable, poorly-designed classes.

### Setter Injection

```java
@Service
public class OrderService {
    private OptionalService optionalService;
    
    // For truly optional dependencies
    @Autowired(required = false)
    public void setOptionalService(OptionalService service) {
        this.optionalService = service;
    }
}
```

**When to use:** Optional dependencies that have a reasonable default (e.g., a Cache that defaults to no-op if not provided).

### How Spring Injects

**Resolution process:**
1. Spring identifies the type needed (constructor parameter type)
2. Looks for beans of that type in the container
3. If one match → injects it
4. If multiple matches → looks for @Primary, or matches by name
5. If zero matches → fails (unless @Autowired(required=false))

**By name matching:**
```java
public OrderService(PaymentService stripePaymentService) {
    // Spring looks for bean NAMED "stripePaymentService"
}
```

**Interview Answer:**
I use constructor injection exclusively because it makes dependencies explicit and allows immutable fields. Field injection hides dependencies and makes unit testing harder — you need reflection or the Spring context to test. Setter injection is only for truly optional dependencies with sensible defaults.

---

## 4) Bean Definition & Creation

### What is a BeanDefinition

Before Spring creates a bean, it creates a **BeanDefinition** — a recipe for creating the bean:

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
    String getBeanClassName();           // What class to instantiate
    String getScope();                    // singleton, prototype, etc.
    ConstructorArgumentValues getConstructorArgumentValues();  // Constructor args
    MutablePropertyValues getPropertyValues();  // Properties to set
    String getInitMethodName();          // @PostConstruct
    String getDestroyMethodName();       // @PreDestroy
    boolean isLazyInit();               // Create on demand?
}
```

**BeanDefinition sources:**
- `@Component` classes → `AnnotatedGenericBeanDefinition`
- `@Bean` methods → `ConfigurationClassBeanDefinition`
- XML `<bean>` → `GenericBeanDefinition`
- Auto-configuration → various implementations

### How Spring Scans Classes

**@ComponentScan mechanics:**

```java
@ComponentScan(basePackages = "com.example")
public class AppConfig { }
```

**Step-by-step:**
1. `ClassPathBeanDefinitionScanner` scans classpath
2. Reads resources matching `com/example/**/*.class`
3. For each class, checks for `@Component` stereotype annotations
4. If found, creates `ScannedGenericBeanDefinition`
5. Registers in `BeanDefinitionRegistry`

**Stereotype annotations (all include @Component):**
- `@Component` — generic bean
- `@Service` — business logic layer (no special handling, semantic)
- `@Repository` — data access layer (enables exception translation)
- `@Controller` / `@RestController` — web layer

**Deep Insight:**
@Repository triggers `PersistenceExceptionTranslationPostProcessor` which automatically translates database-specific exceptions (like SQLException) into Spring's DataAccessException hierarchy.

---

## 5) Bean Lifecycle (Critical, Step-by-Step)

### Complete Lifecycle Flow

```
1. BEAN DEFINITION CREATION
   ├─ Component scan finds @Component class
   ├─ Or @Bean method processed
   └─ BeanDefinition registered in BeanFactory

2. BEAN INSTANTIATION
   ├─ BeanFactory calls constructor
   └─ Object created (raw, not configured)

3. DEPENDENCY INJECTION
   ├─ Constructor args resolved and injected
   ├─ @Autowired fields populated
   └─ @Value expressions resolved

4. AWARE INTERFACES
   ├─ BeanNameAware.setBeanName() — if implements
   ├─ BeanFactoryAware.setBeanFactory() — if implements
   ├─ ApplicationContextAware.setApplicationContext() — if implements
   └─ Other Aware interfaces...

5. BEANPOSTPROCESSOR PRE-INIT
   ├─ postProcessBeforeInitialization() called
   └─ @PostConstruct methods invoked

6. INITIALIZINGBEAN
   └─ afterPropertiesSet() called — if implements

7. CUSTOM INIT METHOD
   └─ init-method from XML or @Bean(initMethod)

8. BEANPOSTPROCESSOR POST-INIT
   └─ postProcessAfterInitialization() called
      └─ HERE: AOP proxies created (e.g., @Transactional)

9. BEAN READY
   └─ Fully configured bean in application context

10. DESTRUCTION (on context close)
    ├─ @PreDestroy methods invoked
    ├─ DisposableBean.destroy() — if implements
    └─ Custom destroy method
```

### Where Proxies Are Created

**Critical timing:** Proxies are created in `postProcessAfterInitialization()`:

```java
// Your bean after step 7 (initialization)
OrderService originalBean = new OrderService(paymentService);
originalBean.afterPropertiesSet();  // InitializingBean

// BeanPostProcessor wraps it
OrderService proxy = (OrderService) proxyCreator.postProcessAfterInitialization(
    originalBean, "orderService"
);

// Returned proxy intercepts method calls
```

**Why it matters:**
```java
@Service
public class OrderService {
    @Transactional
    public void createOrder() { ... }
    
    public void process() {
        this.createOrder();  // PROXY BYPASSED! No transaction!
    }
}
```

Inside `process()`, `this` refers to the **real object**, not the proxy. The proxy's transaction interceptor never fires.

**Deep Insight:**
Spring creates a JDK dynamic proxy or CGLIB subclass that wraps your bean. The proxy implements the same interfaces (or extends the class) and intercepts method calls to apply cross-cutting concerns (transactions, security, caching) before delegating to your actual bean.

### How AOP Integrates

`AnnotationAwareAspectJAutoProxyCreator` is a BeanPostProcessor that:
1. In `postProcessAfterInitialization()`, checks if bean matches any pointcuts
2. If yes, creates proxy wrapping the bean
3. Proxy contains interceptors (advice) for matched aspects

```
Method call → Proxy → Interceptor chain (transaction, security) → Your bean
```

**Interview Answer:**
Spring beans go through a defined lifecycle: definition registered, instantiated via constructor, dependencies injected, Aware interfaces called, @PostConstruct executed, InitializingBean callback, then BeanPostProcessors run. AOP proxies are created in postProcessAfterInitialization — that's why self-invocation bypasses aspects. Finally, @PreDestroy runs on shutdown.

---

## 6) Bean Scopes

### Singleton (Default)

```
Container creates ONE instance → reuses it for all injections
```

```java
@Service  // Default scope is singleton
public class OrderService { }
```

**Characteristics:**
- Created at context startup (or first use if lazy)
- Shared across entire application
- Must be thread-safe (concurrent access)

### Prototype

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class ReportBuilder { }
```

**Characteristics:**
- New instance created for EACH injection
- Spring does NOT manage full lifecycle (no @PreDestroy called)
- Use for stateful, non-thread-safe objects

### Request / Session (Web)

```java
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class ShoppingCart { }
```

**Characteristics:**
- Request: one instance per HTTP request
- Session: one instance per HTTP session
- Requires proxy (injected into longer-lived singletons)

**Proxy mechanics:**
```java
@Service
public class OrderService {
    @Autowired
    private ShoppingCart cart;  // Injected proxy!
    
    public void addItem(Item item) {
        cart.add(item);  // Proxy resolves to current request's instance
    }
}
```

**Deep Insight:**
ScopedProxyMode.TARGET_CLASS creates a CGLIB proxy that delegates to the correct scoped instance based on current ThreadLocal context (RequestContextHolder).

---

## 7) BeanPostProcessor (Very Important)

### What It Is

A BeanPostProcessor is a **factory hook** that intercepts bean creation:

```java
public interface BeanPostProcessor {
    // Called BEFORE init methods (@PostConstruct, afterPropertiesSet)
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }
    
    // Called AFTER init methods — HERE proxies are created
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

### How Spring Uses It Internally

**Key internal BeanPostProcessors:**

| Processor | Purpose |
|-----------|---------|
| **AutowiredAnnotationBeanPostProcessor** | Processes @Autowired, @Value |
| **CommonAnnotationBeanPostProcessor** | Processes @PostConstruct, @PreDestroy, @Resource |
| **ApplicationContextAwareProcessor** | Injects ApplicationContext, Environment |
| **AsyncAnnotationBeanPostProcessor** | Wraps @Async methods |
| **TransactionProxyFactoryBean** (internal) | Creates @Transactional proxies |
| **AsyncAnnotationBeanPostProcessor** | Creates @Async proxies |

**Example: Custom BeanPostProcessor**

```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof OrderService) {
            // Wrap in logging proxy
            return Proxy.newProxyInstance(
                bean.getClass().getClassLoader(),
                bean.getClass().getInterfaces(),
                (proxy, method, args) -> {
                    System.out.println("Calling " + method.getName());
                    return method.invoke(bean, args);
                }
            );
        }
        return bean;
    }
}
```

### AOP Proxy Creation Example

```java
// How @Transactional works via BeanPostProcessor

public Object postProcessAfterInitialization(Object bean, String beanName) {
    // Check if class has @Transactional methods
    if (hasTransactionalMethods(bean.getClass())) {
        // Create proxy
        ProxyFactory proxyFactory = new ProxyFactory(bean);
        
        // Add transaction interceptor
        proxyFactory.addAdvice(new TransactionInterceptor(
            transactionManager, 
            new AnnotationTransactionAttributeSource()
        ));
        
        return proxyFactory.getProxy();
    }
    return bean;
}
```

**Interview Answer:**
BeanPostProcessor is a hook into bean creation that runs before and after initialization. Spring uses it heavily internally — AutowiredAnnotationBeanPostProcessor handles @Autowired, CommonAnnotationBeanPostProcessor handles @PostConstruct, and AnnotationAwareAspectJAutoProxyCreator creates AOP proxies after initialization. This is where @Transactional and @Cacheable proxies are actually created.

---

## 8) AOP Basics (as Part of Core)

### Why Proxies Exist

**Cross-cutting concerns problem:**
```java
public class OrderService {
    public void createOrder() {
        // Transaction begin
        try {
            // Business logic
            dao.save(order);
            // Transaction commit
        } catch (Exception e) {
            // Transaction rollback
        }
    }
    
    public void updateOrder() {
        // Same transaction code copied!
        try { ... } catch { ... }
    }
}
```

**Proxy solution:**
```java
// Your clean code
@Service
public class OrderService {
    @Transactional
    public void createOrder() {
        // Just business logic
        dao.save(order);
    }
}

// Proxy generated by Spring wraps it:
public class OrderServiceProxy extends OrderService {
    public void createOrder() {
        TransactionStatus status = tm.getTransaction(definition);
        try {
            super.createOrder();  // Call real method
            tm.commit(status);
        } catch (Exception e) {
            tm.rollback(status);
        }
    }
}
```

### JDK vs CGLIB Proxies

| Aspect | JDK Dynamic Proxy | CGLIB |
|--------|-------------------|-------|
| **Requirement** | Target must implement interface | Can proxy classes |
| **Mechanism** | Implements interface | Generates bytecode subclass |
| **Final methods** | N/A | Cannot proxy (subclass can't override) |
| **Constructor** | No constraint | Needs default constructor |
| **Performance** | Slightly slower | Slightly faster |
| **Default choice** | If interface exists | If no interface, or forced |

**Configuration:**
```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(App.class);
        // Force CGLIB for all proxies
        app.setDefaultProperties(
            Collections.singletonMap("spring.aop.proxy-target-class", "true")
        );
        app.run(args);
    }
}
```

**How @Transactional Works Internally:**

1. **Detection:** `SpringTransactionAnnotationParser` finds @Transactional on methods
2. **Metadata:** Creates `RuleBasedTransactionAttribute` with propagation, isolation, timeout
3. **Proxy creation:** `InfrastructureAdvisorAutoProxyCreator` (BeanPostProcessor) wraps bean
4. **Interceptor:** `TransactionInterceptor` handles method invocation
   - Gets transaction manager
   - Creates/resumes transaction
   - Invokes target method
   - Commits or rollbacks based on exception

```
@Transactional call
    ↓
TransactionProxy (CGLIB/JDK)
    ↓
TransactionInterceptor.invoke()
    ├── Get PlatformTransactionManager
    ├── Create TransactionStatus
    ├── Invoke target method
    ├── Handle exception (rollback if RuntimeException)
    └── Commit
```

---

## 9) Circular Dependencies (Critical)

### Constructor Injection Problem

```java
@Service
public class OrderService {
    public OrderService(PaymentService paymentService) { }  // Needs Payment
}

@Service
public class PaymentService {
    public PaymentService(OrderService orderService) { }  // Needs Order
}
```

**What happens at startup:**
```
Create OrderService → needs PaymentService
    Create PaymentService → needs OrderService
        Create OrderService → needs PaymentService (LOOP!)
```

**Spring fails with:**
```
BeanCurrentlyInCreationException: 
Error creating bean with name 'orderService':
Requested bean is currently in creation: Is there an unresolvable circular reference?
```

### How Spring Resolves (With Field Injection)

With field/setter injection, Spring can break the circle using **early references**:

```java
// Step 1: Instantiate OrderService (no injection yet)
OrderService orderService = new OrderService();
beanFactory.registerSingleton("orderService", orderService);  // Early exposure!

// Step 2: Instantiate PaymentService — can get early OrderService reference
PaymentService paymentService = new PaymentService(orderService);

// Step 3: Inject PaymentService into OrderService
orderService.setPaymentService(paymentService);
```

**Mechanism:**
1. Instantiate OrderService (calls default constructor or no-arg)
2. Add to singletonsCurrentlyInCreation cache
3. Inject dependencies — when it hits PaymentService, start creating that
4. PaymentService needs OrderService → get from cache (early reference)
5. Complete PaymentService, return to OrderService
6. Finish OrderService

### Why Constructor Injection Fails

Constructor injection requires **all dependencies at instantiation time**:
```java
public OrderService(PaymentService p) {  // Must provide NOW
    this.paymentService = p;
}
```

Spring can't create an "empty" OrderService first — it needs the PaymentService immediately. But PaymentService needs OrderService. Deadlock.

### Solutions

**1. Refactor to remove cycle (best)**
```java
// Extract common dependency
@Service
public class OrderProcessor {
    private final OrderValidator validator;
    
    public void process(Order order) {
        validator.validate(order);  // Not circular
    }
}
```

**2. Use setter/field injection (works but less clean)**
```java
@Service
public class OrderService {
    @Autowired  // Setter or field injection
    private PaymentService paymentService;
}
```

**3. Use @Lazy to break immediate need**
```java
@Service
public class OrderService {
    public OrderService(@Lazy PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

@Lazy creates a proxy that delays actual injection until first use.

**4. Bean method in configuration**
```java
@Configuration
public class ServiceConfig {
    @Bean
    public OrderService orderService() {
        return new OrderService(paymentService());  // Method call
    }
    
    @Bean
    public PaymentService paymentService() {
        return new PaymentService(orderService());  // Spring handles!
    }
}
```

Spring caches @Bean method results, so the second call returns the proxy.

**Common Trap:**
Thinking "Spring will handle it" — no, with constructor injection and circular deps, Spring fails fast. This is actually good design — it forces you to fix the architectural problem.

**Interview Answer:**
Circular dependencies with constructor injection fail because Spring can't instantiate either bean without the other. With field or setter injection, Spring breaks the cycle using early references — it instantiates one bean partially, exposes it, then creates the second bean which can reference the early instance. But this is fragile; the real fix is refactoring to remove the cycle by extracting shared dependencies.

---

## 10) ApplicationContext Internals

### How Context Starts

```java
AnnotationConfigApplicationContext context = 
    new AnnotationConfigApplicationContext(AppConfig.class);
```

**Internal flow:**

```
new AnnotationConfigApplicationContext()
    ↓
this.reader = new AnnotatedBeanDefinitionReader(this)  // For @Component classes
this.scanner = new ClassPathBeanDefinitionScanner(this)  // For component scanning
    ↓
register(componentClasses)  // Register your @Configuration
    ↓
refresh()  // THE BIG ONE
    ├─ prepareRefresh()
    ├─ obtainFreshBeanFactory()
    ├─ prepareBeanFactory()  // Add standard post-processors
    ├─ postProcessBeanFactory()
    ├─ invokeBeanFactoryPostProcessors()  // Process @Bean methods
    ├─ registerBeanPostProcessors()  // Register all BPPs
    ├─ initMessageSource()
    ├─ initApplicationEventMulticaster()
    ├─ onRefresh()  // Template method for subclasses
    ├─ registerListeners()
    ├─ finishBeanFactoryInitialization()  // CREATE SINGLETONS
    │   ├─ Instantiate remaining singletons
    │   ├─ Resolve dependencies
    │   ├─ Apply BPPs
    │   └─ Call init methods
    └─ finishRefresh()
```

### Bean Loading Process

**1. Bean Definition Loading**
- @ComponentScan: Scans classpath, finds @Component classes
- @Configuration: Processes @Bean methods
- XML: Loads bean definitions from files
- Auto-configuration: Loads from META-INF

**2. BeanFactoryPostProcessors**
Modify bean definitions BEFORE beans are created:
```java
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory);
}
```

Example: `ConfigurationClassPostProcessor` processes @Configuration classes and registers @Bean definitions.

**3. Singleton Instantiation**
```java
// DefaultListableBeanFactory.preInstantiateSingletons()
for (String beanName : beanNames) {
    RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
    if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
        getBean(beanName);  // Create and wire
    }
}
```

**4. Dependency Graph Resolution**

Spring builds a dependency graph and instantiates in **topological order**:

```
A depends on B, C
B depends on D
C depends on D

Order: D → B → C → A
```

Cycles break this (hence why Spring fails on constructor circular deps).

**Deep Insight:**
The key method is `DefaultSingletonBeanRegistry.getSingleton()` which:
1. Checks if bean already exists
2. If not, marks as "currently in creation"
3. Calls ObjectFactory to create instance
4. Adds to singleton cache
5. Returns bean

This singleton cache is the heart of Spring's IoC container.

---

## 11) Common Pitfalls

### Field Injection Problems

**Already covered, but worth repeating:**
```java
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService;  // 5 problems:
    // 1. Can't make final
    // 2. Hidden dependency
    // 3. Hard to test (need ReflectionTestUtils or Spring)
    // 4. Object in incomplete state between creation and injection
    // 5. Framework coupling
}
```

### Hidden Circular Dependencies

**Not obvious in code:**
```java
// Service A
@Service
public class NotificationService {
    @Autowired
    private EmailService emailService;
}

// Service B
@Service
public class EmailService {
    @Autowired
    private TemplateService templateService;
}

// Service C — completes the circle!
@Service
public class TemplateService {
    @Autowired
    private NotificationService notificationService;  // Oops!
}
```

**Detection:**
Use `--debug` or analyze startup logs. Or run with constructor injection — fails fast.

### Proxy-Related Surprises

**1. Self-invocation bypasses proxy:**
```java
@Service
public class OrderService {
    @Transactional
    public void createOrder() { }
    
    public void batchCreate() {
        this.createOrder();  // No transaction!
    }
}
```

**2. Final methods can't be proxied:**
```java
@Service
public class OrderService {
    @Transactional
    public final void createOrder() { }  // @Transactional ignored!
}
```

CGLIB can't override final methods.

**3. Non-public methods:**
```java
@Service
public class OrderService {
    @Transactional
    private void createOrder() { }  // Ignored — can't proxy private!
}
```

### Bean Scope Misunderstandings

**Injecting prototype into singleton:**
```java
@Service  // Singleton
public class OrderService {
    @Autowired
    private ReportBuilder reportBuilder;  // Prototype
    
    public void process() {
        // Same ReportBuilder used every time!
        // Not a new instance per method call
    }
}
```

**Fix:** Use `ObjectProvider` or scoped proxy:
```java
@Autowired
private ObjectProvider<ReportBuilder> reportBuilderProvider;

public void process() {
    ReportBuilder builder = reportBuilderProvider.getObject();  // New instance!
}
```

---

## 12) Interview Questions with Strong Answers

### Q: What is IoC and how does Spring implement it?

**Strong Answer:**
> Inversion of Control means the framework controls object creation instead of your code. Spring implements IoC through its ApplicationContext container. You define beans via annotations like @Component or @Bean, and Spring instantiates them, resolves their dependencies through constructor injection, and manages their full lifecycle. This decouples your classes from their dependencies — you depend on abstractions, and Spring wires the concrete implementations.

---

### Q: How does DI work in Spring?

**Strong Answer:**
> Spring uses dependency injection to satisfy bean dependencies. During instantiation, it examines constructor parameters, @Autowired fields, or setter methods. It looks up matching beans in the container by type, then name if multiple match. For constructor injection, it resolves all dependencies first, creating beans in topological order so everything is available. The container then calls the constructor or setter with the resolved beans, populating your object with its dependencies before returning the fully configured bean.

---

### Q: Explain bean lifecycle step-by-step.

**Strong Answer:**
> First, Spring creates a BeanDefinition from @Component or @Bean. Then it instantiates the bean via constructor. Next comes dependency injection — constructor args, @Autowired fields, @Value properties populated. Then Aware interfaces are called (BeanNameAware, ApplicationContextAware if implemented). BeanPostProcessor's postProcessBeforeInitialization runs, followed by @PostConstruct and InitializingBean.afterPropertiesSet. postProcessAfterInitialization runs — this is where AOP proxies are created. Finally, @PreDestroy runs on shutdown. The key point is that proxies are created after initialization, which is why self-invocation bypasses @Transactional.

---

### Q: What is BeanPostProcessor?

**Strong Answer:**
> BeanPostProcessor is a hook into the bean creation process. It has two methods: postProcessBeforeInitialization runs before @PostConstruct and afterPropertiesSet; postProcessAfterInitialization runs after all init methods. Spring uses it internally for @Autowired injection, @PostConstruct handling, and AOP proxy creation. For example, AnnotationAwareAspectJAutoProxyCreator is a BPP that checks if a bean has @Transactional or aspect annotations, and if so, wraps it in a proxy that handles transaction management before calling the actual method.

---

### Q: How does @Transactional work internally?

**Strong Answer:**
> @Transactional is handled by Spring's AOP infrastructure. AnnotationAwareAspectJAutoProxyCreator, which is a BeanPostProcessor, detects @Transactional on your bean. In postProcessAfterInitialization, it creates a proxy — either JDK dynamic proxy if you have interfaces, or CGLIB subclass. The proxy contains a TransactionInterceptor. When you call a @Transactional method, the proxy intercepts, starts a transaction via PlatformTransactionManager, invokes your actual method, then commits or rolls back based on whether an exception was thrown. Self-invocation bypasses this because you call the target directly, not through the proxy.

---

### Q: What is the difference between BeanFactory and ApplicationContext?

**Strong Answer:**
> BeanFactory is the basic IoC container interface for creating and managing beans. ApplicationContext extends BeanFactory and adds enterprise features: automatic bean post-processor registration, internationalization support, event publishing, and resource loading. BeanFactory creates beans on demand (lazy), while ApplicationContext pre-instantiates singletons at startup. BeanFactory requires manual registration of post-processors; ApplicationContext does it automatically. In practice, everyone uses ApplicationContext implementations like AnnotationConfigApplicationContext or the web variants.

---

## 13) Interview Speaking Mode

### Short Spoken Answers

#### What is IoC?
> Inversion of Control means instead of your code creating objects and wiring them together, Spring's container does it. You declare what you need — with @Component, constructor injection — and Spring instantiates beans and injects dependencies. Your code gets ready-to-use objects without knowing how they're created.

#### How DI works
> When Spring creates a bean, it looks at constructor parameters or @Autowired fields. It finds matching beans in the container by type, or by name if there are multiple. It creates dependencies first, then injects them into your bean. For constructor injection, all dependencies must be resolvable at creation time; for setter injection, they can be set later.

#### Bean lifecycle
> Beans are instantiated, dependencies injected, then initialization callbacks run — @PostConstruct, afterPropertiesSet. BeanPostProcessors wrap the bean, often creating AOP proxies for @Transactional. Then the bean is ready. On shutdown, @PreDestroy runs. The key timing is that proxies are created after initialization, which is why calling your own @Transactional method doesn't work — you bypass the proxy.

#### BeanPostProcessor
> It's a hook into bean creation. postProcessBeforeInitialization runs before init methods; postProcessAfterInitialization runs after. Spring uses it internally for @Autowired, @PostConstruct, and creating AOP proxies. You can write custom ones to modify or wrap beans — like adding logging or validation proxies.

#### Circular dependencies
> With constructor injection, circular dependencies fail immediately because Spring can't create either bean without the other. With field or setter injection, Spring breaks the cycle by exposing an early reference — it instantiates one bean partially, puts it in a cache, then the second bean can get that reference even though the first isn't fully initialized. But this is fragile; better to refactor and remove the cycle.

---

### Bad vs Good Answers

| Question | Weak Answer | Strong Answer |
|----------|-------------|---------------|
| **IoC?** | "Spring manages objects" | "Inversion of Control delegates object creation to the container. You define beans and dependencies; Spring instantiates and wires them, giving you ready-to-use objects with loose coupling" |
| **DI mechanism?** | "It injects dependencies" | "Spring resolves dependencies by type from the container, creates them in topological order, and injects via constructor, setter, or field. Constructor injection is preferred for immutability and explicit dependencies" |
| **Bean lifecycle?** | "It creates beans" | "Instantiation → DI → Aware interfaces → BPP before init → @PostConstruct → BPP after init where AOP proxies are created → ready. Self-invocation bypasses proxy because it happens after initialization phase" |
| **BeanPostProcessor?** | "It processes beans" | "Hook into creation with before/after initialization methods. Spring uses it for @Autowired, @PostConstruct, and AOP proxy creation. Runs after bean instantiated but before it's used" |
| **@Transactional?** | "It manages transactions" | "BPP detects annotation and creates proxy in postProcessAfterInitialization. Proxy intercepts calls, starts transaction via TransactionManager, invokes target, commits/rolls back. Self-invocation bypasses because you call target, not proxy" |
| **BeanFactory vs ApplicationContext?** | "ApplicationContext is more advanced" | "ApplicationContext extends BeanFactory adding automatic BPP registration, i18n, events, resource loading. Pre-instantiates singletons vs lazy creation. Real usage is always ApplicationContext" |

---

### Common Interviewer Follow-ups

**On IoC/DI:**
- "What happens if two beans implement the same interface?"
- "How would you inject a prototype bean into a singleton?"
- "What's the difference between @Inject and @Autowired?"

**On lifecycle:**
- "Where exactly are AOP proxies created?"
- "What happens if @PostConstruct throws an exception?"
- "How would you hook into bean creation?"

**On circular deps:**
- "Why does constructor injection fail but field injection works?"
- "How would you break a circular dependency?"
- "What is early reference exposure?"

**On proxies:**
- "Why doesn't @Transactional work on private methods?"
- "What's the difference between JDK and CGLIB proxies?"
- "How do you force CGLIB proxying?"

**On scopes:**
- "How does request-scoped bean work in singleton?"
- "What's the proxy mode for scoped beans?"
- "When is prototype scope useful?"

---

### 1-Minute Summary Answers

**"What is Spring's IoC and how does it work?"**
> Spring's Inversion of Control container, ApplicationContext, manages object creation and dependency wiring. You annotate classes with @Component or define @Bean methods; Spring scans, instantiates, resolves dependencies by type or name, and injects them via constructor, setter, or field. The container manages the full lifecycle from creation through initialization callbacks to destruction, letting your code depend on abstractions rather than concrete implementations.

**"Explain Spring bean lifecycle."**
> Beans are defined from @Component or @Bean, instantiated via reflection, dependencies injected through constructor or @Autowired fields. Then Aware interfaces are called, BeanPostProcessor's beforeInit runs, @PostConstruct and InitializingBean callbacks execute, then BPP's afterInit creates AOP proxies for @Transactional. Finally @PreDestroy runs on shutdown. The critical timing is proxies created after initialization, which is why self-invocation bypasses transaction management.

**"How does @Transactional work internally?"**
> AnnotationAwareAspectJAutoProxyCreator, a BeanPostProcessor, detects @Transactional in postProcessAfterInitialization. It creates a JDK or CGLIB proxy wrapping your bean. The proxy contains TransactionInterceptor which manages transactions — starts via PlatformTransactionManager, invokes your method, then commits or rolls back based on exceptions. Self-invocation doesn't work because you call the target object directly, not the proxy.

**"Circular dependencies in Spring?"**
> Constructor circular dependencies fail because neither bean can be instantiated without the other. Field or setter injection works because Spring exposes an early bean reference — instantiates partially, caches it, then creates the second bean which can reference the early instance. But this is fragile; the proper fix is refactoring to eliminate the cycle by extracting the shared dependency into a third bean.

---

### Quick Confidence Builders

**Show you understand internals:**
- "The implementation detail that matters is the singleton cache in DefaultSingletonBeanRegistry"
- "What I've seen in production is that constructor injection fails fast on circular deps, which is actually good design"
- "The proxy is created in postProcessAfterInitialization, not at instantiation"

**Signal architectural thinking:**
- "The key insight is that Spring's design forces you toward proper dependency graphs"
- "When we refactored to constructor injection, we found hidden circular dependencies that indicated design problems"
- "I prefer explicit wiring over autowiring by name because it's clearer"

---

## Summary

Spring Core provides an IoC container (ApplicationContext) that manages bean lifecycle and dependency injection. Beans are defined via @Component stereotypes, @Bean methods, or XML. The container instantiates beans, resolves dependencies through constructor injection (preferred), field injection (avoid), or setter injection (for optional), and manages the full lifecycle. Bean lifecycle includes: BeanDefinition creation, instantiation, dependency injection, Aware interfaces, BeanPostProcessor pre-init, @PostConstruct/InitializingBean, BeanPostProcessor post-init (where AOP proxies are created), ready state, and @PreDestroy on shutdown. BeanPostProcessors hook into creation — Spring uses them for @Autowired, @PostConstruct, and AOP proxy creation. @Transactional works via proxies created by AnnotationAwareAspectJAutoProxyCreator; the proxy intercepts calls to manage transactions. Circular dependencies with constructor injection fail immediately; field/setter injection works via early reference exposure but is fragile. ApplicationContext extends BeanFactory adding automatic post-processor registration, i18n, and events. JDK proxies require interfaces; CGLIB can proxy classes but can't override final methods. Self-invocation bypasses proxies because you call the target, not the proxy.
