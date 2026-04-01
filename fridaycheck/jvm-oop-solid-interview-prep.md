# JVM, JRE, JDK, OOP & SOLID — Middle Java Interview Prep

Practical guide to Java platform fundamentals and design principles.

---

## 1) JVM, JRE, JDK

### What is JVM (Java Virtual Machine)

The **runtime engine** that executes Java bytecode. It provides:

- Platform independence ("write once, run anywhere")
- Memory management (garbage collection)
- Security (sandbox, bytecode verification)
- Performance optimizations (JIT compilation)

### What is JRE (Java Runtime Environment)

Everything needed to **run** Java applications:

- JVM
- Core libraries (`java.lang`, `java.util`, etc.)
- Configuration files

**Does NOT include**: compiler, debugger, development tools.

### What is JDK (Java Development Kit)

Complete development toolkit:

- Everything in JRE
- **Compiler** (`javac`) — converts `.java` to `.class`
- **Debugger** (`jdb`)
- **Tools**: `javadoc`, `jar`, `jconsole`, `jps`, etc.

### Execution flow

```
YourCode.java → [javac] → YourCode.class → [JVM] → Execution
     ↑                                              ↓
  Developer                                    End User
     │                                              │
    JDK                                            JRE
```

**Interview tip**: Production servers need JRE only; development machines need JDK.

---

## 2) JVM Basics (High Level)

### ClassLoader

Responsible for loading `.class` files into JVM. Three main levels:

1. **Bootstrap** — loads core Java classes (`java.lang.`*)
2. **Extension/Platform** — loads platform extensions
3. **Application** — loads your application classes from classpath

**Why it matters**: understanding helps debug `ClassNotFoundException` and `NoClassDefFoundError`.

### Runtime Memory (briefly)

- **Heap**: objects live here, garbage collected
- **Stack**: method frames, local variables (per thread)
- **Metaspace**: class metadata (replaced PermGen in Java 8+)
- **PC Register**: current instruction per thread

### Execution Engine

- **Interpreter**: executes bytecode line by line (slow)
- **JIT Compiler** (Just-In-Time): hot code compiled to native machine code for speed
- **Garbage Collector**: automatic memory cleanup

---

## 3) OOP Principles

### Encapsulation

**Idea**: Bundle data and methods that operate on it, hide internal details.

```java
public class BankAccount {
    private double balance; // Hidden
    
    public void deposit(double amount) { // Controlled access
        if (amount > 0) {
            this.balance += amount;
        }
    }
    
    public double getBalance() { // Read-only access
        return balance;
    }
}
```

**Why**: Prevents invalid states, allows internal changes without breaking callers.

### Inheritance

**Idea**: Create new classes from existing ones, reusing and extending behavior.

```java
public class Animal {
    public void eat() { System.out.println("Eating..."); }
}

public class Dog extends Animal {
    public void bark() { System.out.println("Woof!"); }
}

Dog d = new Dog();
d.eat();  // Inherited from Animal
d.bark(); // Dog's own method
```

**Why**: Code reuse, hierarchical organization.

**Common mistake**: Overuse ("favor composition over inheritance").

### Polymorphism

**Idea**: Same interface, different implementations.

```java
public interface PaymentProcessor {
    void processPayment(BigDecimal amount);
}

public class StripeProcessor implements PaymentProcessor { ... }
public class PayPalProcessor implements PaymentProcessor { ... }

// Polymorphic usage:
PaymentProcessor processor = getProcessor(); // Could be Stripe or PayPal
processor.processPayment(amount); // Different behavior, same interface
```

**Why**: Flexibility, testability (mocking), extensibility.

### Abstraction

**Idea**: Hide complexity, show only essential features.

```java
public interface OrderService {
    Order createOrder(CreateOrderRequest request); // What, not how
}

// Implementation details hidden behind interface
```

**Why**: Manage complexity, change implementation without affecting users.

---

## 4) SOLID Principles

### S — Single Responsibility Principle

**Idea**: A class should have only one reason to change (one job).

```java
// BAD: Multiple responsibilities
public class OrderManager {
    void createOrder() { ... }
    void sendEmail() { ... }      // Email responsibility
    void generateReport() { ... } // Reporting responsibility
    void saveToDatabase() { ... } // Persistence responsibility
}

// GOOD: Separated concerns
public class OrderService { void createOrder() { ... } }
public class EmailService { void sendOrderConfirmation() { ... } }
public class ReportGenerator { void generateOrderReport() { ... } }
public class OrderRepository { void save(Order order) { ... } }
```

**Real example**: Controller that handles HTTP, business logic, and database calls — violates SRP.

### O — Open/Closed Principle

**Idea**: Open for extension, closed for modification.

```java
// BAD: Modify existing code for new payment type
public class PaymentProcessor {
    void process(String type, BigDecimal amount) {
        if (type.equals("stripe")) { ... }
        else if (type.equals("paypal")) { ... }
        // Adding new type requires modifying this class!
    }
}

// GOOD: Extend without modifying
public interface PaymentProcessor {
    void process(BigDecimal amount);
}

public class StripeProcessor implements PaymentProcessor { ... }
public class PayPalProcessor implements PaymentProcessor { ... }
// New type? Just add new class!
```

**Typical error**: `if-else` chains checking types — every new feature breaks existing code.

### L — Liskov Substitution Principle

**Idea**: Subtypes must be substitutable for their base types without altering correctness.

```java
// BAD VIOLATION:
public class Rectangle {
    void setWidth(int w) { ... }
    void setHeight(int h) { ... }
}

public class Square extends Rectangle {
    // Violates LSP: setting width also changes height!
    void setWidth(int w) {
        super.setWidth(w);
        super.setHeight(w); // Unexpected side effect!
    }
}

// Client code breaks:
Rectangle r = new Square();
r.setWidth(5);
r.setHeight(10);
// Expected area: 50, but Square gives 100!
```

**Rule**: Inheritance should model "is-a" relationships where subclass doesn't break parent contract.

### I — Interface Segregation Principle

**Idea**: Clients shouldn't depend on interfaces they don't use.

```java
// BAD: Fat interface
public interface Worker {
    void work();
    void eat();
    void sleep();
}

// Robot must implement eat() and sleep() — nonsense!

// GOOD: Segregated interfaces
public interface Workable { void work(); }
public interface Feedable { void eat(); }
public interface Sleepable { void sleep(); }

public class Human implements Workable, Feedable, Sleepable { ... }
public class Robot implements Workable { ... } // Only what it needs
```

**Typical error**: `UserService` interface with 20 methods when clients only need 2-3.

### D — Dependency Inversion Principle

**Idea**: Depend on abstractions, not concretions. High-level modules shouldn't depend on low-level details.

```java
// BAD: High-level depends on low-level
public class OrderService {
    private MySQLDatabase db = new MySQLDatabase(); // Concrete dependency!
    
    void saveOrder(Order order) {
        db.insert(order); // Tightly coupled to MySQL
    }
}

// GOOD: Both depend on abstraction
public interface OrderRepository {
    void save(Order order);
}

public class OrderService {
    private final OrderRepository repository; // Abstraction!
    
    public OrderService(OrderRepository repository) {
        this.repository = repository; // Dependency injection
    }
    
    void saveOrder(Order order) {
        repository.save(order); // Works with any implementation
    }
}
```

**Why**: Testability (inject mocks), flexibility (switch PostgreSQL → MongoDB), loose coupling.

---

## 5) Common Developer Mistakes

### SOLID violations

- **God Class**: 2000-line class doing everything (violates SRP)
- **Copy-paste inheritance**: Using inheritance just to reuse code (violates LSP)
- **Concrete dependencies everywhere**: Hard to test, hard to change (violates DIP)
- **Fat interfaces**: Forcing clients to implement methods they don't need (violates ISP)
- `if-else` type checking instead of polymorphism (violates OCP)

### Wrong inheritance

```java
// BAD: Square extends Rectangle (breaks LSP)
// BAD: Stack extends ArrayList (Stack is not an ArrayList!)
```

### Tight coupling

```java
public class OrderProcessor {
    private StripePayment stripe = new StripePayment(); // Direct instantiation!
    private EmailSender email = new EmailSender();
    private MySQLDatabase db = new MySQLDatabase();
    // Changing any requires modifying this class
}
```

---

## 6) Practical Advice

### Writing cleaner code

1. **Start with SRP**: If class does too much, split it
2. **Prefer composition**: `class Car { private Engine engine; }` vs `class Car extends Engine`
3. **Program to interfaces**: `List<String> list = new ArrayList<>()` not `ArrayList<String>`
4. **Dependency injection**: Pass dependencies via constructor, don't create inside
5. **Use polymorphism**: Replace switch/if-else on types with strategy pattern

### Applying SOLID without fanaticism

- Don't create interface for every single class
- Don't split until there's actual pain
- YAGNI: if only one implementation exists, concrete class is fine
- Balance: perfect OOP design vs shipping working code

---

## 7) Short Interview Summary

The Java platform consists of JDK (development tools + JRE), JRE (runtime libraries + JVM), and JVM (execution engine with classloader, memory management, and JIT compiler). OOP principles — encapsulation, inheritance, polymorphism, abstraction — organize code for maintainability and reuse. SOLID provides design guidelines: Single Responsibility (one reason to change), Open/Closed (extend without modifying), Liskov Substitution (subtypes must not break parent contracts), Interface Segregation (small focused interfaces), and Dependency Inversion (depend on abstractions). Common mistakes include God classes, wrong inheritance hierarchies, tight coupling through concrete dependencies, and type-checking instead of polymorphism. Apply OOP and SOLD pragmatically: favor composition over inheritance, program to interfaces, use dependency injection, but avoid over-engineering simple cases.

---

## 8) Typical Interview Questions with Answers

### Q1: What's the difference between JDK, JRE, and JVM?

**A**: JVM executes bytecode; JRE includes JVM + libraries to run Java; JDK includes JRE + development tools (compiler, debugger). JDK for development, JRE for running apps, JVM is the actual runtime engine.

### Q2: Explain the compilation and execution process.

**A**: `.java` files compiled by `javac` into `.class` bytecode. ClassLoader loads classes into JVM. Bytecode is verified, then interpreted or JIT-compiled to native code and executed.

### Q3: What's polymorphism and give a real example.

**A**: Same interface, different behaviors. Example: `PaymentProcessor` interface with `StripeProcessor` and `PayPalProcessor` implementations. Client code calls `process()` without knowing which processor is used — extensible and testable.

### Q4: Why favor composition over inheritance?

**A**: Inheritance creates tight coupling and rigid hierarchies (breaks if parent changes). Composition is more flexible (can change behavior at runtime), avoids LSP violations, and better models "has-a" vs "is-a" relationships.

### Q5: Explain SRP with a bad and good example.

**A**: Bad: `OrderManager` that creates orders, sends emails, generates reports, and saves to DB. Good: Separate `OrderService`, `EmailService`, `ReportGenerator`, `OrderRepository` — each with single responsibility.

### Q6: What's wrong with `if (obj instanceof Type) { ((Type)obj).doSomething(); }`?

**A**: Violates OCP (must modify code for new types) and misses polymorphism. Better: use polymorphic method dispatch or Strategy pattern so new types work without changes.

### Q7: Explain DIP and why it matters for testing.

**A**: Depend on abstractions (interfaces) not concretions. Allows injecting mocks in tests. Without DIP, you can't test `OrderService` without hitting real database.

### Q8: LSP violation example?

**A**: `Square extends Rectangle` — setting width also sets height in Square, breaking Rectangle contract. Client expecting independent width/height gets unexpected behavior.

### Q9: When to use inheritance vs composition?

**A**: Inheritance for true "is-a" relationships (Dog is an Animal) where all parent behaviors make sense. Composition for "has-a" or "uses-a" relationships, or when behavior needs to change dynamically.

### Q10: Practical OOP in Spring?

**A**: Dependency injection implements DIP. Service layer uses interfaces (PaymentService). Controllers depend on service interfaces not implementations. Repositories abstract persistence. This enables mocking in unit tests and swapping implementations.