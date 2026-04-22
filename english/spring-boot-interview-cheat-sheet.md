# Spring Boot Interview Cheat Sheet (Middle/Senior)

Deep dive into Spring Boot internals, auto-configuration, and production mechanics.

---

## 1) What Problem Spring Boot Solves

### The Pre-Boot Pain (Classic Spring)

Before Spring Boot (Spring 3.x era), setting up a Spring application meant:

```xml
<!-- web.xml -->
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/spring-servlet.xml</param-value>
    </init-param>
</servlet>

<!-- spring-servlet.xml -->
<context:component-scan base-package="com.example" />
<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/views/" />
    <property name="suffix" value=".jsp" />
</bean>

<!-- dozens of XML files for data source, transactions, security... -->
```

**Problems:**
- Verbose XML configuration (hundreds of lines)
- Dependency version hell (Spring 4.2 needs Jackson 2.6, not 2.7)
- Manual server deployment (WAR to Tomcat)
- No standardized way to configure
- Every project reinvented the same boilerplate

### What Spring Boot Brings

**1. Convention over Configuration**
- Sensible defaults (Tomcat embedded, Jackson for JSON)
- Auto-detects what you need based on classpath
- Properties file instead of XML

**2. Standalone Applications**
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
- Embedded Tomcat starts automatically
- No WAR packaging needed
- `java -jar application.jar`

**3. Dependency Management**
- Starters aggregate compatible dependencies
- Spring Boot BOM (Bill of Materials) locks versions
- No more version conflicts

**Interview Answer:**
Spring Boot solves the configuration hell of classic Spring. Instead of writing hundreds of lines of XML to wire up components, Boot uses auto-configuration to detect what's on your classpath and configure it automatically. It also embeds the server, so you run a JAR file instead of deploying a WAR to external Tomcat.

---

## 2) Core Concept: Auto-Configuration (CRITICAL)

### What is Auto-Configuration

Auto-configuration is Spring Boot's mechanism to **automatically configure Spring beans based on:**
- What's on the classpath (classpath scanning)
- What beans already exist
- Properties you've defined

**Example:** Add `spring-boot-starter-web` to classpath → Boot auto-configures:
- DispatcherServlet
- Default JSON converter (Jackson)
- Embedded Tomcat
- Static resource handling

### How @EnableAutoConfiguration Works

**Deep Insight:**
```java
@SpringBootApplication  // Includes @EnableAutoConfiguration
public class Application { }
```

Under the hood:
```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)  // ← The magic
public @interface EnableAutoConfiguration { }
```

**AutoConfigurationImportSelector** implements `ImportSelector`:
1. Reads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
2. Each line is a fully-qualified class name of an auto-configuration
3. Checks **conditions** on each (see below)
4. Only configurations with passing conditions are applied

**The imports file** (from spring-boot-autoconfigure.jar):
```
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
... (147+ auto-configurations in Boot 3.x)
```

### Conditional Annotations (The Decision Logic)

Spring Boot uses **@Conditional** annotations to decide whether to apply a configuration:

**@ConditionalOnClass**
```java
@ConditionalOnClass({ DispatcherServlet.class, Servlet.class })
public class DispatcherServletAutoConfiguration { ... }
```
→ Only applies if DispatcherServlet is on classpath (you have spring-webmvc)

**@ConditionalOnMissingBean**
```java
@Bean
@ConditionalOnMissingBean  // ← Only if YOU didn't define this bean
public DataSource dataSource(DataSourceProperties properties) {
    return DataSourceBuilder.create().build();
}
```
→ Respects your custom beans. If you define a DataSource, Boot backs off.

**@ConditionalOnProperty**
```java
@ConditionalOnProperty(prefix = "spring.datasource", name = "url")
public class DataSourceAutoConfiguration { ... }
```
→ Only applies if `spring.datasource.url` is set in properties.

**@ConditionalOnWebApplication**
→ Only applies if running as a web app (Servlet context exists)

### How Spring Decides What Beans to Create

**Step-by-step decision tree:**

```
1. Application starts with SpringApplication.run()

2. ApplicationContext is created
   
3. AutoConfigurationImportSelector loads all auto-config classes
   from META-INF/spring/*.imports files

4. For each auto-configuration class:
   
   a. Check @ConditionalOnClass → Are required classes present?
   
   b. Check @ConditionalOnMissingBean → Did user already define this?
   
   c. Check @ConditionalOnProperty → Are required properties set?
   
   d. Check @ConditionalOnWebApplication → Is this a web app?
   
   e. If ALL conditions pass → Apply this configuration
   
5. Configuration classes define @Beans
   
6. Your @ComponentScan beans are registered
   
7. BeanFactoryPostProcessors and BeanPostProcessors run
   
8. Context refreshed → Application running
```

**Example Decision Flow for DataSource:**

```
Is HikariCP or Tomcat JDBC pool on classpath? (@ConditionalOnClass)
    ↓ YES
Is there already a DataSource bean defined? (@ConditionalOnMissingBean)
    ↓ NO (user didn't define one)
Is spring.datasource.url configured? (@ConditionalOnProperty)
    ↓ YES
→ Create HikariDataSource with connection pool configured
```

**Common Trap:**
Adding a dependency (e.g., H2) can trigger auto-configuration you didn't expect. Suddenly you have an in-memory database you didn't ask for because `DataSourceAutoConfiguration` saw H2 on classpath.

---

## 3) @SpringBootApplication Breakdown

```java
@SpringBootApplication  // One annotation = three things
```

Expands to:
```java
@Configuration              // 1. This is a configuration class
@EnableAutoConfiguration    // 2. Turn on auto-configuration
@ComponentScan(             // 3. Scan for components
    excludeFilters = {
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class)
    }
)
public @interface SpringBootApplication { }
```

### @Configuration

Marks the class as a source of bean definitions. Same as classic Spring @Configuration.

### @ComponentScan

Scans current package and sub-packages for:
- @Component, @Service, @Repository, @Controller beans
- Recursively from the package of the annotated class

**Deep Insight:**
```java
// If your main class is in com.example.Application
// Then @ComponentScan scans: com.example.*

// This is why structure matters:
com.example
├── Application.java      ← @SpringBootApplication here
├── controller/           ← Scanned ✓
├── service/              ← Scanned ✓
└── repository/           ← Scanned ✓
```

**Common Trap:**
Putting Application class in root but components outside the package tree:
```
com.example.Application  ← Scans com.example
com.other.Service        ← NOT scanned! Different package tree
```

Fix: Add explicit @ComponentScan("com") or move Application to common root.

### @EnableAutoConfiguration

As explained in section 2 — triggers the auto-configuration mechanism.

**Interview Answer:**
@SpringBootApplication is a convenience that combines three annotations: @Configuration marks this as a config source, @ComponentScan scans the current package and below for components, and @EnableAutoConfiguration triggers Boot's auto-configuration engine that decides what beans to create based on your classpath.

---

## 4) Starter Dependencies (Hidden Magic)

### What Starters Actually Do

Starters are **POM dependencies that aggregate** everything you need for a feature:

**spring-boot-starter-web** brings:
- spring-boot-starter (core)
- spring-boot-starter-json (Jackson)
- spring-boot-starter-tomcat (embedded)
- spring-web (REST support)
- spring-webmvc (MVC framework)

You add ONE dependency, you get the whole working stack.

### How It Works (POM Hierarchy)

```xml
<!-- Your pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- No version needed! Parent BOM manages it -->
</dependency>

<!-- spring-boot-starter-web pom.xml -->
<dependencies>
    <dependency>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <artifactId>spring-boot-starter-json</artifactId>
    </dependency>
    <dependency>
        <artifactId>spring-boot-starter-tomcat</artifactId>
    </dependency>
    <dependency>
        <artifactId>spring-webmvc</artifactId>
    </dependency>
</dependencies>
```

### Parent BOM (Bill of Materials)

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>
```

This parent POM defines **version properties**:
```xml
<properties>
    <spring-framework.version>6.1.0</spring-framework.version>
    <jackson.version>2.15.3</jackson.version>
    <tomcat.version>10.1.15</tomcat.version>
    ...
</properties>
```

All starters inherit these versions → **no version conflicts**.

**Deep Insight:**
Starters don't contain code — they're just dependency aggregators. The actual auto-configuration comes from `spring-boot-autoconfigure` JAR which is pulled in by `spring-boot-starter`.

---

## 5) Embedded Servlet Container

### How Boot Runs Without External Tomcat

Traditional Java web app:
```
Build WAR → Deploy to Tomcat → Tomcat starts your app
```

Spring Boot:
```
Build JAR → Run java -jar → Embedded Tomcat starts inside your app
```

**The Mechanism:**
```java
// spring-boot-starter-tomcat brings tomcat-embed-core

// EmbeddedWebServerFactoryCustomizerAutoConfiguration
// creates TomcatServletWebServerFactory

Tomcat tomcat = new Tomcat();
tomcat.setPort(8080);
Context context = tomcat.addContext("", new File(".").getAbsolutePath());

// DispatcherServlet added programmatically
Wrapper servlet = Tomcat.addServlet(context, "dispatcher", 
    new DispatcherServlet(webApplicationContext));
servlet.setLoadOnStartup(1);
servlet.addMapping("/");

tomcat.start();
```

All hidden inside Spring Boot's auto-configuration.

### Embedded Server Options

| Server | Starter | Auto-Configuration |
|--------|---------|-------------------|
| **Tomcat** (default) | spring-boot-starter-tomcat | TomcatWebServerFactoryCustomizer |
| **Jetty** | spring-boot-starter-jetty | JettyServletWebServerFactoryCustomizer |
| **Undertow** | spring-boot-starter-undertow | UndertowWebServerFactoryCustomizer |

**Switching to Jetty:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

Boot detects Jetty on classpath → auto-configures Jetty instead.

**Deep Insight:**
The embedded server starts during `ApplicationContext` refresh. Bean `ServletWebServerApplicationContext` is created, which triggers `WebServerFactory` to create and start the embedded server.

---

## 6) Application Startup Flow (STEP-BY-STEP)

### From main() to Running Application

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
        // What happens inside?
    }
}
```

**Complete startup sequence:**

```
1. SpringApplication instantiated
   - WebApplicationType determined (Servlet vs Reactive vs None)
   - ApplicationContextInitializer and ApplicationListener loaded

2. SpringApplication.run() called
   
3. SpringApplicationRunListeners started
   - Notifies listeners: "starting"
   
4. Environment prepared
   - Loads application.properties/yml
   - Applies profiles
   - Property sources configured
   
5. ApplicationContext created
   - AnnotationConfigServletWebServerApplicationContext (for web)
   - Or AnnotationConfigApplicationContext (for non-web)
   
6. ApplicationContext prepared
   - Register SpringBootExceptionReporter
   - Load sources (your @SpringBootApplication class)
   
7. Load ApplicationContextInitializers
   - Run initializers to customize context before bean loading
   
8. BeanDefinitionLoader loads bean definitions
   - From your @SpringBootApplication (configuration class)
   - @ComponentScan triggers classpath scanning
   - Finds @Component, @Service, @Repository, @Controller

9. Bean definitions registered
   - Auto-configuration classes loaded (via AutoConfigurationImportSelector)
   - Each auto-configuration checked with @Conditional annotations
   - Passing configurations register their @Bean methods
   
10. BeanFactoryPostProcessors run
    - Modify bean definitions before instantiation
    - ConfigurationClassPostProcessor processes @Configuration classes
    - AutowiredAnnotationBeanPostProcessor prepares for @Autowired

11. Bean instantiation starts
    - Singleton beans pre-instantiated
    - Constructor injection resolved
    - @Value and @Autowired populated

12. BeanPostProcessors run
    - postProcessBeforeInitialization on each bean
    - @PostConstruct methods invoked
    - postProcessAfterInitialization (proxies created here)
    
13. Embedded server started
    - ServletWebServerApplicationContext.refresh() triggers
    - WebServerFactory creates Tomcat/Jetty/Undertow
    - Server starts listening on port
    - DispatcherServlet registered

14. ApplicationRunner and CommandLineRunner beans executed

15. SpringApplicationRunListeners notified: "started"

16. Application running
```

### Visual Timeline

```
main()
   ↓
SpringApplication()
   ↓
determineWebApplicationType()  [Servlet]
   ↓
run()
   ↓
prepareEnvironment()            [application.yml loaded]
   ↓
createApplicationContext()      [AnnotationConfigServletWebServer...]
   ↓
prepareContext()
   ↓
refreshContext()
   ├── loadBeanDefinitions()    [@ComponentScan, auto-config]
   ├── BeanFactoryPostProcessors [Process @Configuration]
   ├── instantiateBeans()       [Create @Service, @Repository]
   ├── BeanPostProcessors        [@PostConstruct, proxies]
   └── onRefresh()              [Start embedded Tomcat]
       ↓
Tomcat.start()
   ↓
DispatcherServlet initialized
   ↓
Application ready
```

**Interview Answer:**
When you run SpringApplication.run(), Boot first determines if this is a web app, creates an ApplicationContext, loads your bean definitions from component scanning and auto-configuration, instantiates singleton beans running through post-processors, and then starts the embedded web server. The whole process takes the configuration from your code and auto-configuration from classpath, merges them, and wires everything together.

---

## 7) Configuration Properties System

### application.yml / application.properties

Spring Boot loads properties from multiple sources (priority order):

1. Command line arguments (`--server.port=8081`)
2. JVM system properties (`-Dserver.port=8081`)
3. OS environment variables (`SERVER_PORT=8081`)
4. `application-{profile}.properties/yml`
5. `application.properties/yml` (outside jar)
6. `application.properties/yml` (inside jar)
7. @PropertySource annotations
8. SpringApplication default properties

### @ConfigurationProperties vs @Value

**@Value (Simple, one-off):**
```java
@Value("${server.port:8080}")  // 8080 is default
private int serverPort;

@Value("${app.description:Default desc}")
private String description;
```

**Limitations:**
- No validation
- No type conversion errors handling
- No structured binding

**@ConfigurationProperties (Structured, powerful):**
```java
@Data
@ConfigurationProperties(prefix = "app")
@Component
public class AppProperties {
    
    @NotNull
    private String name;
    
    @Min(1)
    @Max(100)
    private int maxConnections = 10;
    
    private Security security = new Security();
    
    @Data
    public static class Security {
        private String username;
        private String password;
    }
}
```

```yaml
app:
  name: MyService
  max-connections: 20
  security:
    username: admin
    password: secret
```

**Benefits:**
- Type-safe configuration classes
- Validation annotations work (JSR-303)
- Relaxed binding (kebab-case ↔ camelCase)
- IDE auto-completion (with spring-boot-configuration-processor)

### Binding Mechanism

Spring Boot uses **Java Beans PropertyEditor** and **Converters**:

```yaml
app:
  timeout: 30s           → Converts to Duration
  memory: 512MB          → Converts to DataSize
  enabled: true          → Converts to boolean
  features: feature1, feature2  → Converts to List<String>
```

**Deep Insight:**
@ConfigurationProperties beans are registered as regular beans. They're processed by ConfigurationPropertiesBindingPostProcessor which binds properties using relaxed naming rules.

**Relaxed Binding Rules:**
```yaml
# All of these map to server.port property
server.port=8080
SERVER_PORT=8080
server_port=8080
serverPort=8080
```

---

## 8) Bean Lifecycle & Context

### How Boot Builds on Spring IoC

Spring Boot is a **convention layer on top of Spring Framework's IoC container**.

**Bean Creation Flow:**

```
BeanDefinition loaded (from @ComponentScan, @Bean method, or auto-config)
   ↓
BeanFactory instantiates bean (constructor called)
   ↓
Dependency injection (populate properties, @Autowired)
   ↓
BeanNameAware.setBeanName()  (if implements)
   ↓
BeanFactoryAware.setBeanFactory()  (if implements)
   ↓
ApplicationContextAware.setApplicationContext()  (if implements)
   ↓
BeanPostProcessor.postProcessBeforeInitialization()
   ↓
@PostConstruct method invoked
   ↓
InitializingBean.afterPropertiesSet()  (if implements)
   ↓
BeanPostProcessor.postProcessAfterInitialization()
   ↓
   [Bean is ready]
   ↓
... application runs ...
   ↓
@PreDestroy method invoked
   ↓
DisposableBean.destroy()  (if implements)
   ↓
Bean destroyed
```

### Bean Creation Order

**Critical concept:** Beans are created based on **dependency graph**, not declaration order.

```java
@Service
public class OrderService {  // Created 2nd
    @Autowired
    private PaymentService paymentService;  // Needs this first
}

@Service
public class PaymentService {  // Created 1st
    @Autowired
    private PaymentGateway gateway;  // Needs this before
}
```

Spring analyzes @Autowired dependencies and creates beans in **topological order**.

### BeanPostProcessors in Boot

Key processors that modify your beans:

| Processor | What It Does |
|-----------|--------------|
| **AutowiredAnnotationBeanPostProcessor** | Processes @Autowired, @Value |
| **CommonAnnotationBeanPostProcessor** | Processes @PostConstruct, @PreDestroy |
| **ConfigurationClassPostProcessor** | Processes @Configuration, @Bean |
| **AnnotationAwareAspectJAutoProxyCreator** | Creates AOP proxies (@Transactional, @Cacheable) |
| **ApplicationContextAwareProcessor** | Injects ApplicationContext if implements Aware |

**Deep Insight:**
@Transactional works via proxy created by BeanPostProcessor. The actual bean is wrapped, and the proxy intercepts method calls to start/commit transactions.

---

## 9) Profiles & Environment

### @Profile

Activate beans conditionally:

```java
@Configuration
@Profile("development")  // Only loads in "dev" profile
public class DevConfig {
    @Bean
    public DataSource h2DataSource() {
        return new EmbeddedDatabaseBuilder().setType(H2).build();
    }
}

@Configuration
@Profile("production")
public class ProdConfig {
    @Bean
    public DataSource postgresDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:postgresql://prod-db:5432/mydb")
            .build();
    }
}
```

### Activating Profiles

```bash
# application.properties
spring.profiles.active=dev,local

# Or command line
java -jar app.jar --spring.profiles.active=prod

# Or environment variable
SPRING_PROFILES_ACTIVE=prod java -jar app.jar
```

### Profile-Specific Properties

```
application.yml              # Common config
application-dev.yml          # Dev profile
application-prod.yml         # Prod profile
```

Boot loads `application.yml` first, then overlays profile-specific file.

### Environment Abstraction

```java
@Autowired
private Environment env;

public void doSomething() {
    String property = env.getProperty("server.port");
    boolean isDev = env.acceptsProfiles(Profiles.of("development"));
}
```

**Interview Answer:**
Profiles let you activate different beans and configurations based on environment. I use @Profile to separate dev and production configs — like H2 for dev, PostgreSQL for prod. Profiles can be set via application.properties, command line, or environment variables. Boot loads common application.yml first, then overlays profile-specific files.

---

## 10) Actuator (Production Essentials)

### What It Provides

Spring Boot Actuator gives **production-ready endpoints** for monitoring and management:

| Endpoint | What It Shows |
|----------|---------------|
| `/actuator/health` | Application health (UP/DOWN, disk space, DB connectivity) |
| `/actuator/info` | Build info (version, git commit) |
| `/actuator/metrics` | JVM metrics (memory, GC, threads), HTTP metrics |
| `/actuator/env` | Environment properties |
| `/actuator/loggers` | Logging levels (can change dynamically!) |
| `/actuator/beans` | All beans in ApplicationContext |
| `/actuator/threaddump` | JVM thread dump |
| `/actuator/heapdump` | JVM heap dump |

### Why It Matters

- **Health checks:** Load balancers poll `/health` to know if instance is ready
- **Metrics:** Feed to Prometheus/Grafana for dashboards
- **Log tuning:** Change log levels without restart (`POST /actuator/loggers/package`)
- **Debugging:** Thread dump when performance issues

**Security Note:**
Expose only what you need:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics  # NOT beans, env, heapdump!
  endpoint:
    health:
      show-details: when-authorized
```

---

## 11) Spring Boot vs Spring Framework (CRITICAL)

### Boot = Opinionated Layer on Spring

**What Spring Framework provides:**
- IoC/DI container
- AOP framework
- Data access abstractions
- Transaction management
- MVC framework
- Test support

**What Spring Boot ADDS:**

| Feature | What Boot Does |
|---------|---------------|
| **Auto-configuration** | Automatically configures Spring based on classpath |
| **Starter dependencies** | Curated dependency sets that work together |
| **Embedded servers** | Run apps without external Tomcat |
| **Properties externalization** | application.yml with profile support |
| **Actuator** | Production monitoring endpoints |
| **Spring Boot DevTools** | Hot reload, automatic restart |

**Analogy:**
- Spring Framework = Engine + transmission + chassis
- Spring Boot = Complete car with sensible defaults, ready to drive

**Deep Insight:**
You can use Spring Framework without Boot (still done in some legacy apps). But Boot removes all the boilerplate configuration. Boot doesn't replace Spring's core — it enhances it with smart defaults.

**Interview Answer:**
Spring Boot doesn't replace Spring Framework — it builds on top of it. Spring provides the core IoC container, MVC, and data access. Boot adds auto-configuration to detect what you need and configure it automatically, starter dependencies that bundle compatible libraries, and embedded servers so you run a JAR instead of deploying a WAR. Boot is all about convention over configuration.

---

## 12) Common Pitfalls

### Over-Reliance on Auto-Configuration

**Problem:** Not understanding what Boot configured for you.

```yaml
# application.yml is empty, everything "just works"
```

Then production issues arise:
- Default connection pool too small
- Jackson serializing lazy associations (N+1)
- Default error handling exposing stack traces

**Fix:** Explicitly configure critical settings:
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      connection-timeout: 5000
  jackson:
    serialization:
      fail-on-empty-beans: false
```

### Bean Overriding Issues

Spring Boot 2.1+ disables bean definition overriding by default:
```yaml
spring:
  main:
    allow-bean-definition-overriding: false  # Default
```

If you define a bean that conflicts with auto-configuration:
```
The bean 'dataSource', defined in class path resource [...],
could not be registered. A bean with that name has already been defined...
```

**Fix:** Name your bean differently, or exclude auto-configuration:
```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
```

### Unexpected Auto-Configurations

Adding a dependency triggers configuration you didn't expect:

```xml
<!-- Added for testing -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

But if scope is wrong or dependency leaks:
```
Boot auto-configures H2 DataSource!
Production app tries to connect to in-memory H2 instead of PostgreSQL.
```

**Fix:** Exclude if not needed:
```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    HibernateJpaAutoConfiguration.class
})
```

### Circular Dependencies

```java
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService;  // Depends on Payment
}

@Service
public class PaymentService {
    @Autowired
    private OrderService orderService;  // Depends on Order
}
```

Spring Boot 2.6+ fails on startup with clear error. Fix by refactoring or using `@Lazy`.

---

## 13) Debugging Auto-Configuration (VERY IMPORTANT)

### --debug Flag

```bash
java -jar myapp.jar --debug
# or
java -jar myapp.jar -Ddebug=true
```

Outputs **ConditionEvaluationReport** — shows why each auto-configuration matched or didn't match.

### Condition Evaluation Report

Example output:
```
============================
CONDITIONS EVALUATION REPORT
============================


Positive matches:
-----------------

   DispatcherServletAutoConfiguration matched:
      - @ConditionalOnClass found required class 'DispatcherServlet' (OnClassCondition)
      - found 'session' scope (OnWebApplicationCondition)

   DataSourceAutoConfiguration matched:
      - @ConditionalOnClass found required classes 'DataSource', 'EmbeddedDatabaseType' (OnClassCondition)
      - @ConditionalOnProperty 'spring.datasource.url' found (ConditionalOnPropertyCondition)


Negative matches:
-----------------

   ActiveMQAutoConfiguration did not match:
      - @ConditionalOnClass did not find required class 'javax.jms.ConnectionFactory' (OnClassCondition)

   RabbitAutoConfiguration did not match:
      - @ConditionalOnClass did not find required classes 'com.rabbitmq.client.Channel' (OnClassCondition)
```

**Reading the report:**
- **Positive matches:** Configurations that applied (and why)
- **Negative matches:** Configurations that didn't apply (and why)
- **Exclusions:** Explicitly excluded configurations

### Understanding Bean Creation

To see what beans were created:
```java
@Autowired
private ApplicationContext ctx;

@PostConstruct
public void printBeans() {
    Arrays.stream(ctx.getBeanDefinitionNames())
        .filter(name -> name.contains("dataSource") || name.contains("transaction"))
        .forEach(System.out::println);
}
```

Or use Actuator `/actuator/beans` endpoint.

### Fixing "Why wasn't my configuration applied?"

**Scenario:** You expected a bean to be created but it wasn't.

**Debug steps:**
1. Run with `--debug`, check if your auto-configuration class is in report
2. Check if @ConditionalOnClass requirements are met (is dependency in pom?)
3. Check if @ConditionalOnMissingBean blocked it (did you define it elsewhere?)
4. Check component scan — is your config class in scanned package?

---

## 14) Interview Questions with Strong Answers

### Q: What is Spring Boot and how is it different from Spring Framework?

**Strong Answer:**
> Spring Boot is an opinionated layer on top of Spring Framework. Spring provides the core IoC container, MVC framework, and data access abstractions. Boot adds auto-configuration that detects what's on your classpath and configures beans automatically, starter dependencies that aggregate compatible libraries, and embedded servers so you can run a JAR file instead of deploying a WAR. Boot doesn't replace Spring — it removes the boilerplate configuration.

---

### Q: How does auto-configuration work internally?

**Strong Answer:**
> Auto-configuration is triggered by @EnableAutoConfiguration, which uses AutoConfigurationImportSelector to load configuration classes from META-INF/spring/*.imports files. Each configuration is guarded by @Conditional annotations — like @ConditionalOnClass which checks if required classes are on classpath, @ConditionalOnMissingBean which backs off if you defined the bean yourself, and @ConditionalOnProperty which checks property values. Only configurations passing all conditions are applied. That's how adding spring-boot-starter-web triggers DispatcherServlet auto-configuration.

---

### Q: What happens when you run SpringApplication.run()?

**Strong Answer:**
> SpringApplication.run() creates an ApplicationContext, determines the web application type, prepares the Environment by loading application.properties, then refreshes the context. During refresh, it triggers component scanning to find your @Components, loads auto-configuration classes and checks their conditions, instantiates singleton beans in dependency order running through BeanPostProcessors, and finally starts the embedded web server. The whole sequence wires together your beans, auto-configured beans, and starts Tomcat listening on the configured port.

---

### Q: How does Spring Boot decide which beans to create?

**Strong Answer:**
> Three sources: your @ComponentScan finds beans you defined with @Service, @Repository, @Component; auto-configuration classes from starters propose beans based on classpath conditions; and @Bean methods in your @Configuration classes. For each candidate, Spring checks @Conditional annotations — @ConditionalOnClass verifies dependencies exist, @ConditionalOnMissingBean respects your custom beans, @ConditionalOnProperty checks settings. Passing conditions means the bean is registered in the context and instantiated during startup.

---

### Q: What are starters?

**Strong Answer:**
> Starters are curated dependency aggregators. They're POMs that bring in everything you need for a feature with compatible versions. Adding spring-boot-starter-web pulls in Spring MVC, Jackson for JSON, embedded Tomcat, and validation. The parent POM's BOM manages versions so you don't get conflicts. Starters don't contain code — they're just dependency management. The actual auto-configuration comes from spring-boot-autoconfigure.

---

### Q: Why does my configuration sometimes not apply?

**Strong Answer:**
> Most likely a @Conditional check is failing. Common causes: @ConditionalOnClass fails because a dependency is missing from classpath; @ConditionalOnMissingBean means you already defined a bean with that name; @ConditionalOnProperty requires a property you haven't set; or your configuration class isn't being scanned — check that it's in the component scan path or explicitly imported. Run with --debug flag to see the ConditionEvaluationReport showing exactly why each configuration was included or excluded.

---

### Q: How do you override auto-configuration?

**Strong Answer:**
> Define your own bean with the same type — @ConditionalOnMissingBean in auto-configuration will back off. For more control, exclude specific auto-configurations using @SpringBootApplication(exclude = DataSourceAutoConfiguration.class). Or use properties in application.yml to customize auto-configured beans without replacing them entirely, like spring.datasource.hikari.maximum-pool-size to configure the Hikari pool that Boot creates.

---

## 15) Interview Speaking Mode

### Short Spoken Answers

#### What is Spring Boot?
> Spring Boot is an opinionated framework that sits on top of Spring. It removes configuration boilerplate by auto-detecting what you need based on classpath and configuring it automatically. You add a starter like spring-boot-starter-web and get a working web stack with embedded Tomcat, no XML required.

#### How auto-configuration works
> Boot scans for auto-configuration classes in META-INF/spring/*.imports. Each has @Conditional annotations — checks like "is this class on classpath?" or "did user already define this bean?". If all conditions pass, the configuration applies. That's how adding a dependency can trigger beans to be created automatically.

#### What happens at startup
> SpringApplication.run creates an ApplicationContext, loads your beans from component scanning, loads auto-configurations and checks their conditions, instantiates everything in dependency order, runs post-processors for things like @PostConstruct and transaction proxies, and starts the embedded server. The whole process wires together your app based on code and properties.

#### Starters explained
> Starters are dependency bundles. They don't contain code — they just aggregate compatible libraries via Maven POMs. The parent BOM locks versions so everything works together. Actual auto-configuration comes from spring-boot-autoconfigure JAR.

#### Debugging configuration
> I use --debug flag which outputs ConditionEvaluationReport. It shows every auto-configuration and why it matched or didn't — like @ConditionalOnClass failed because a class was missing, or @ConditionalOnMissingBean skipped because I defined my own. That's how I figure out why a bean wasn't created.

---

### Bad vs Good Answers

| Question | Weak Answer | Strong Answer |
|----------|-------------|---------------|
| **What is Boot?** | "It makes Spring easier" | "It's an opinionated layer that auto-configures Spring based on classpath detection. It uses @Conditional annotations to decide what beans to create without manual XML" |
| **Auto-configuration?** | "It configures things automatically" | "AutoConfigurationImportSelector loads config classes from META-INF, checks @ConditionalOnClass, @ConditionalOnMissingBean, @ConditionalOnProperty. Only passing conditions apply the configuration" |
| **Startup flow?** | "It starts the application" | "Determines web type, creates ApplicationContext, loads component scan and auto-config beans in topological order, runs BeanPostProcessors, starts embedded server" |
| **Starters?** | "They add dependencies" | "Curated dependency aggregators with compatible versions managed by parent BOM. Auto-configuration triggered by classpath detection, not the starter itself" |
| **Override auto-config?** | "Define your own bean" | "Define bean with same name — @ConditionalOnMissingBean backs off. Or exclude specific auto-configurations, or customize via properties without full replacement" |

---

### Common Interviewer Follow-ups

**On auto-configuration:**
- "How would you create your own auto-configuration?"
- "What happens if two auto-configurations define the same bean?"
- "Can you have conditional properties in auto-configuration?"

**On startup:**
- "What's the difference between ApplicationRunner and CommandLineRunner?"
- "How would you hook into the startup process?"
- "What happens if a bean fails to initialize?"

**On profiles:**
- "How do you activate multiple profiles?"
- "Can profiles be set from environment variables?"
- "What's the difference between @Profile and conditional properties?"

**On embedded server:**
- "How would you switch from Tomcat to Jetty?"
- "When exactly does the embedded server start?"
- "Can you run multiple embedded servers in one app?"

**On debugging:**
- "How do you see what auto-configurations were applied?"
- "Why might a bean you expect not be created?"
- "How do you debug bean dependency issues?"

---

### 1-Minute Summary Answers

**"What is Spring Boot and how does it differ from Spring?"**
> Spring Boot is an opinionated convention-over-configuration layer on top of Spring Framework. Spring provides the IoC container and core frameworks. Boot adds auto-configuration that detects your classpath and creates beans automatically, starter dependencies that bundle compatible libraries, and embedded servers. You get a working application with a main method and a few annotations instead of hundreds of lines of XML configuration.

**"How does auto-configuration work?"**
> Boot's AutoConfigurationImportSelector loads configuration classes from META-INF/spring/*.imports. Each configuration has @Conditional annotations that check requirements — @ConditionalOnClass for classpath presence, @ConditionalOnMissingBean to respect user beans, @ConditionalOnProperty for settings. Configurations passing all conditions are applied, registering their beans in the context.

**"What happens when SpringApplication.run() executes?"**
> It creates an ApplicationContext, determines if this is a web app, loads the Environment with properties, triggers component scanning to find your beans, loads auto-configurations checking their conditions, instantiates beans in dependency order through post-processors, and starts the embedded server. The result is a fully wired application context ready to serve requests.

**"How do you debug auto-configuration issues?"**
> Run with --debug flag to get ConditionEvaluationReport showing exactly why each auto-configuration matched or didn't. Check if @ConditionalOnClass requirements are met, if @ConditionalOnMissingBean blocked by your own bean, or if the configuration class is outside component scan. The report shows every condition evaluation with pass/fail status.

---

### Quick Confidence Builders

**Signal deep understanding:**
- "The key insight is that auto-configuration is just @Configuration with @Conditional guards"
- "What I've seen in production is..."
- "The implementation detail that matters here is..."
- "Boot is essentially a sophisticated @Conditional system"

**Show practical experience:**
- "When we migrated from Spring to Boot, the main challenge was..."
- "I typically verify auto-configuration with --debug when..."
- "The pitfall we hit with starters was..."

---

## Summary

Spring Boot is an opinionated layer on Spring Framework that eliminates boilerplate through auto-configuration, starter dependencies, and embedded servers. Auto-configuration uses AutoConfigurationImportSelector to load configuration classes from META-INF/spring/*.imports and applies only those passing @Conditional checks — @ConditionalOnClass for classpath detection, @ConditionalOnMissingBean to respect user beans, @ConditionalOnProperty for settings. @SpringBootApplication combines @Configuration, @ComponentScan, and @EnableAutoConfiguration. Component scanning finds your beans; auto-configuration proposes defaults; conditions decide what applies. The startup process creates ApplicationContext, loads environment and properties, scans components, applies passing auto-configurations, instantiates beans in dependency order through BeanPostProcessors, and starts the embedded server. Starters aggregate compatible dependencies managed by parent BOM; they don't contain code themselves. Override auto-configuration by defining your own beans — @ConditionalOnMissingBean backs off. Debug with --debug flag showing ConditionEvaluationReport. Profiles activate beans conditionally; properties overlay from multiple sources with relaxed binding. Actuator provides production endpoints for health, metrics, and debugging. Boot doesn't replace Spring — it builds on it with smart defaults and convention over configuration.
