# Core Java Essentials — Middle Java Interview Prep

Immutable objects, equals/hashCode, String, Optional, Exceptions, and Performance.

---

## 1) Immutable Objects

### What is immutability
An object is **immutable** if its state cannot be changed after creation. All fields are final, set only via constructor.

### Why they are useful
- **Thread-safety**: no synchronization needed, multiple threads can safely share
- **Security**: cannot be modified after passing around
- **Caching**: safe to reuse (like String pool)
- **Predictability**: no side effects, easier to reason about

### How to create properly
```java
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;
    
    public Money(BigDecimal amount, Currency currency) {
        // Defensive copy if mutable input
        this.amount = amount != null ? new BigDecimal(amount.toString()) : null;
        this.currency = currency;
    }
    
    // No setters!
    public BigDecimal getAmount() { 
        // Return defensive copy if mutable
        return new BigDecimal(amount.toString()); 
    }
    
    // Create new instance instead of modifying
    public Money add(Money other) {
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

### Common mistakes
- **Leaking mutable state**: returning reference to mutable internal object
- **Shallow copy**: having mutable fields (like Date, ArrayList) without defensive copies
- **Subclassing**: not making class final (subclass can add mutability)

**Bug example**:
```java
public final class Period {
    private final Date start; // Date is mutable!
    
    public Period(Date start) {
        this.start = start; // Bug: external code can mutate!
    }
    
    public Date getStart() {
        return start; // Bug: caller can modify internal state!
    }
}

// Attacker code:
Period p = new Period(new Date());
p.getStart().setTime(0); // Pwned! Internal state changed!
```

**Fix**: Use `java.time.Instant/LocalDate` (immutable) or defensive copies.

---

## 2) equals and hashCode

### The contract
1. **Reflexive**: `x.equals(x)` is true
2. **Symmetric**: `x.equals(y)` == `y.equals(x)`
3. **Transitive**: if `x.equals(y)` and `y.equals(z)`, then `x.equals(z)`
4. **Consistent`: same result if no changes
5. **hashCode**: equal objects MUST have equal hash codes

### Why they matter
- **HashMap/HashSet**: rely on both for storage and lookup
- **Collections framework**: contracts assumed everywhere

### Impact on collections
Wrong implementation causes:
- Objects "disappear" from HashMap (can't find after put)
- Duplicates appear in HashSet
- Performance degrades to O(n) with bad hashCode

### Typical mistakes
**Mistake 1**: Only override `equals()`, forget `hashCode()`
```java
@Override
public boolean equals(Object o) { ... }
// No hashCode() override!
```
Result: Equal objects go to different buckets in HashMap.

**Mistake 2**: Mutable fields in hashCode
```java
public class Person {
    private String name; // Not final!
    
    @Override public int hashCode() {
        return Objects.hash(name);
    }
}

Person p = new Person("Alice");
map.put(p, "data");
p.setName("Bob"); // hashCode changed!
map.get(p); // Returns null! Object lost!
```

**Mistake 3**: Using different fields in equals vs hashCode

### Correct implementation
```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Person)) return false;
    Person other = (Person) o;
    return Objects.equals(id, other.id) && 
           Objects.equals(email, other.email);
}

@Override
public int hashCode() {
    return Objects.hash(id, email); // Same fields as equals!
}
```

---

## 3) String

### Why immutable
- **String pool**: can safely share same instance
- **Security**: can't change after passing to method
- **HashCode caching**: computed once, reused
- **Thread-safety**: no locks needed

### String pool idea
When you write `String s = "hello"`, JVM may reuse existing "hello" from pool instead of creating new object.

### String vs StringBuilder vs StringBuffer

| Class | Mutable | Thread-safe | When to use |
|-------|---------|-------------|-------------|
| **String** | No | Yes (by immutability) | Constants, concatenation in one line |
| **StringBuilder** | Yes | No | String building in single thread (loops) |
| **StringBuffer** | Yes | Yes | Legacy code, rare multi-thread string building |

### When to use what
- **Simple constant**: `String`
- **Building in loop**: `StringBuilder`
- **Never**: `StringBuffer` in new code (use StringBuilder + local scope)

### Performance example
```java
// BAD: O(n²) - creates n intermediate String objects
String result = "";
for (String s : list) {
    result += s + ","; // Creates new String every iteration!
}

// GOOD: O(n)
StringBuilder sb = new StringBuilder();
for (String s : list) {
    sb.append(s).append(",");
}
String result = sb.toString();
```

---

## 4) Optional

### Purpose
`Optional<T>` is a container that may or may not contain a non-null value. Forces caller to handle absence explicitly.

### When to use
- **Return values** from methods that might not find something
- Optional fields in domain objects (with care)

```java
public Optional<User> findById(Long id) {
    User user = repository.find(id);
    return Optional.ofNullable(user);
}

// Caller must handle:
User user = findById(1L)
    .orElseThrow(() -> new NotFoundException("User not found"));
```

### When it's an anti-pattern
- **Method parameters**: `void process(Optional<User> user)` — just overload or use null
- **Fields**: complicates serialization, adds overhead
- **Collections**: `List<Optional<User>>` — just use empty list
- **Creating empty Optional instead of null checks everywhere**

### Common mistakes
**Mistake 1**: `isPresent()` + `get()` instead of functional methods
```java
// Verbose
if (opt.isPresent()) {
    return opt.get().getName();
} else {
    return "Unknown";
}

// Better
return opt.map(User::getName).orElse("Unknown");
```

**Mistake 2**: Returning null from method returning Optional
```java
// WRONG
public Optional<User> findUser() {
    return null; // Defeats the purpose!
}

// CORRECT
return Optional.empty();
```

---

## 5) Exceptions

### Checked vs Unchecked

| Type | Extends | Handling | When to use |
|------|---------|----------|-------------|
| **Checked** | `Exception` | Must catch or declare | Expected, recoverable conditions (IOException) |
| **Unchecked** | `RuntimeException` | Optional | Programming errors, bugs (NPE, IllegalArgument) |

### When to use what
- **Checked**: Client can reasonably recover (file not found, network timeout)
- **Unchecked**: Programmer mistake, precondition violation, null pointer

### Best practices

**Don't catch broad Exception**
```java
// BAD: Swallows everything, hard to debug
try {
    process();
} catch (Exception e) { // Too broad!
    log.error("Error");
}
```

**Don't swallow errors**
```java
// BAD: Exception lost!
try {
    saveToDatabase();
} catch (SQLException e) {
    // Empty catch - transaction might fail silently!
}
```

**Don't use exceptions for flow control**
```java
// BAD: Abusing exceptions
public User findUser(Long id) {
    try {
        return repository.findById(id);
    } catch (NotFoundException e) {
        return null;
    }
}

// BETTER: Return Optional or null directly
public Optional<User> findUser(Long id) {
    return Optional.ofNullable(repository.findById(id));
}
```

**Proper handling example**
```java
try {
    processOrder(order);
} catch (ValidationException e) {
    // Expected, can inform user
    return ResponseEntity.badRequest().body(e.getMessage());
} catch (PaymentException e) {
    // Expected business failure
    return ResponseEntity.status(402).body("Payment failed");
} catch (Exception e) {
    // Unexpected - log and fail
    log.error("Unexpected error processing order {}", order.getId(), e);
    throw new InternalErrorException("Processing failed", e);
}
```

---

## 6) Performance

### Boxing/unboxing cost
Converting between primitive and wrapper (`int` ↔ `Integer`) creates objects and CPU overhead.

```java
// BAD: Autoboxing in tight loop
Integer sum = 0;
for (int i = 0; i < 1000000; i++) {
    sum += i; // Creates Integer objects!
}

// GOOD: Use primitive
int sum = 0;
for (int i = 0; i < 1000000; i++) {
    sum += i; // Fast, no allocation
}
```

**Prefer**: `Stream` primitives (`IntStream`, `LongStream`) over `Stream<Integer>`.

### Collection complexity
Know your Big-O:
- `ArrayList.get(index)`: O(1)
- `LinkedList.get(index)`: O(n)
- `HashMap.get/put`: O(1) average
- `TreeMap.get/put`: O(log n)

### Common performance killers

**Creating unnecessary objects**
```java
// BAD: New formatter every call
public String format(Date date) {
    return new SimpleDateFormat("yyyy-MM-dd").format(date);
}

// GOOD: Reuse or use thread-safe DateTimeFormatter
private static final DateTimeFormatter FORMATTER = 
    DateTimeFormatter.ofPattern("yyyy-MM-dd");
```

**Strings in loops** (see String section)

**Wrong collections**
```java
// BAD: Using LinkedList for random access
List<User> users = new LinkedList<>();
for (int i = 0; i < users.size(); i++) {
    User u = users.get(i); // O(n) each time - O(n²) total!
}

// GOOD: ArrayList
List<User> users = new ArrayList<>(); // get(i) is O(1)
```

**Excessive synchronization**
```java
// BAD: Synchronized on every method
public synchronized void process() { ... }

// BETTER: Use concurrent collections or lock only critical sections
```

---

## 7) Hidden Pitfalls

### Memory leaks
- **Static collections** that grow forever
- **Listeners/observers** never unregistered
- **ThreadLocal** values not cleaned up in thread pools
- **Cache** without eviction policy

### Thread issues
- Sharing SimpleDateFormat (not thread-safe)
- Modifying collections while iterating (ConcurrentModificationException)
- Forgotten synchronization on compound operations

### equals/hashCode bugs
- Mutable objects as HashMap keys
- Different fields used in equals vs hashCode
- Inheritance breaking symmetry

### Optional misuse
- Optional fields in entities
- Optional parameters
- Null instead of Optional.empty()

---

## 8) Practical Advice

### Safer code
1. **Prefer immutability**: make fields final, use defensive copies
2. **Proper equals/hashCode**: use IDE generation, same fields in both
3. **Handle nulls**: use Optional for returns, Objects.requireNonNull for parameters
4. **Choose right collection**: ArrayList default, HashMap for lookup, know complexity
5. **String building**: use StringBuilder in loops
6. **Exceptions**: checked for recoverable, unchecked for bugs, never swallow

### Production awareness
- Monitor heap growth and GC frequency
- Profile before optimizing (don't guess)
- Document thread-safety of your classes
- Use static analysis (SpotBugs, Sonar) to catch common bugs

---

## 9) Short Interview Summary

Immutable objects provide thread-safety and predictability by preventing state changes after creation; use final fields and defensive copies. Always implement `equals()` and `hashCode()` together with the same fields to avoid breaking HashMap/HashSet. String is immutable for security and sharing; use StringBuilder for loop concatenation, never StringBuffer in new code. Use Optional for return values to force explicit null handling, but avoid it for fields and parameters. Prefer checked exceptions for recoverable conditions and unchecked for programming errors; never catch broad Exception or swallow errors. Performance hot spots include boxing in tight loops, O(n²) collection misuse, and unnecessary object creation; profile before optimizing. Watch for memory leaks in static collections and unregistered listeners.

---

## 10) Typical Interview Questions with Answers

### Q1: Why is String immutable?
**A**: For security (can't be modified after passing), string pool reuse, thread-safety without locks, and hashCode caching. If String were mutable, a passed password could be changed by another reference holder.

### Q2: What happens if two equal objects have different hashCodes?
**A**: HashMap breaks. Objects may be "lost" (can't find after put) or duplicates appear in HashSet. The equals/hashCode contract requires equal objects to have equal hashCodes.

### Q3: String vs StringBuilder vs StringBuffer?
**A**: String is immutable. StringBuilder is mutable, fast, not thread-safe (preferred). StringBuffer is mutable and synchronized (slower, legacy, rarely needed now).

### Q4: When should I use Optional?
**A**: For return values that might be absent. Anti-patterns: fields, method parameters, collection elements, or creating just to immediately unwrap with isPresent()+get().

### Q5: Checked vs Unchecked exceptions?
**A**: Checked (extends Exception) must be caught/declared; use for expected, recoverable conditions. Unchecked (RuntimeException) for programming errors caller can't reasonably recover from.

### Q6: Common memory leak causes in Java?
**A**: Static collections growing forever, listeners not unregistered, ThreadLocal values in thread pools, caches without eviction, and holding references to large object graphs longer than needed.

### Q7: Why avoid autoboxing in tight loops?
**A**: It creates wrapper objects (Integer instead of int) causing GC pressure and CPU overhead. Use primitive streams (IntStream) and variables.

### Q8: What's wrong with `catch(Exception e) { }`?
**A**: Swallows all errors including programming bugs, making debugging impossible. Catch specific exceptions you can handle, let others propagate or wrap appropriately.

### Q9: How to make a class immutable?
**A**: Declare class final, all fields final and private, no setters, defensive copies in constructor and getters for mutable fields, return new instances for modifications.

### Q10: HashMap get() is O(1) — when is it not?
**A**: When hashCode is poor (all keys collide), list/tree in bucket grows, degrading to O(log n) with Java 8+ tree or O(n) in worst case. Also when you need to traverse the found bucket chain.
