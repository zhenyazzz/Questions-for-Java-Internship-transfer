# Java Generics — Middle Java Interview Prep

Practical guide to generics with real examples and common pitfalls.

---

## 1) What Are Generics

### Purpose
Generics are a type parameterization mechanism in Java that enables writing universal and type-safe code. They allow the same class or method to be used with different data types without the need for type casting and with compile-time checking.

### Problem they solve
**Without generics** (pre-Java 5 style):
```java
List list = new ArrayList();
list.add("hello");
list.add(123); // Compiles! But runtime disaster

String s = (String) list.get(0); // Must cast
String s2 = (String) list.get(1); // ClassCastException at runtime!
```

**With generics**:
```java
List<String> list = new ArrayList<>();
list.add("hello");
// list.add(123); // Compile error! Type safety enforced

String s = list.get(0); // No cast needed, guaranteed String
```

### Benefits
- **Type safety**: compiler catches type mismatches
- **No explicit casts**: cleaner code, no ClassCastException risk
- **Generic algorithms**: write once, use with any type

---

## 2) How Generics Work Under the Hood

### Type Erasure
Java generics use **type erasure**. After compilation, type parameters are "erased" and replaced with their bounds (or `Object` if unbounded).

```java
// Your code:
public class Box<T> {
    private T value;
    public void set(T value) { this.value = value; }
    public T get() { return value; }
}

// After compilation (conceptually):
public class Box {
    private Object value;
    public void set(Object value) { this.value = value; }
    public Object get() { return value; }
}
```

### Why you can't use primitives
Since type parameters become `Object` after erasure, and primitives aren't objects, you can't use `int`, `double`, etc. directly:

```java
List<int> list; // ERROR! int is not a reference type

// Solution: wrapper classes + autoboxing
List<Integer> list = new ArrayList<>();
list.add(42); // Autoboxed to Integer.valueOf(42)
```

Use primitive specializations like `IntStream` for performance-critical code to avoid boxing overhead.

---

## 3) Wildcards

### What is `?`
The **unknown type** — represents any type when you don't care about the specific parameter.

### `? extends` (covariance)
"Some specific subtype of T" — useful for **reading** (producing).

```java
public void printAll(List<? extends Number> list) {
    for (Number n : list) { // Can read as Number
        System.out.println(n);
    }
    // list.add(42); // ERROR! Can't add, don't know exact type
}

// Works with:
printAll(Arrays.asList(1, 2, 3));      // List<Integer>
printAll(Arrays.asList(1.5, 2.5));   // List<Double>
```

### `? super` (contravariance)
"Some specific supertype of T" — useful for **writing** (consuming).

```java
public void addIntegers(List<? super Integer> list) {
    list.add(42); // Can add Integer (guaranteed to fit)
    list.add(100);
    // Integer i = list.get(0); // ERROR! Could be Object or Number
}

// Works with:
addIntegers(new ArrayList<Integer>());  // OK
addIntegers(new ArrayList<Number>());   // OK
addIntegers(new ArrayList<Object>());   // OK
```

### PECS Rule
**Producer Extends, Consumer Super**

- **Producer** (you read from it): use `? extends`
- **Consumer** (you write to it): use `? super`

```java
// Producer: copies from src to dest
public static <T> void copy(List<? extends T> src, List<? super T> dest) {
    for (T item : src) {  // Reading from src (producer) -> extends
        dest.add(item);    // Writing to dest (consumer) -> super
    }
}

// Usage:
List<Integer> ints = Arrays.asList(1, 2, 3);
List<Number> numbers = new ArrayList<>();
copy(ints, numbers); // Integer extends Number
```

---

## 4) Limitations of Generics

### Can't instantiate type parameter
```java
public class Box<T> {
    // T t = new T(); // ERROR! Type erasure — T becomes Object
    
    // Workaround: pass Class token
    public T createInstance(Class<T> clazz) throws Exception {
        return clazz.getDeclaredConstructor().newInstance();
    }
}
```

### Can't use instanceof with parameterized types
```java
if (list instanceof ArrayList<String>) { // ERROR! Can't check at runtime
    // After erasure, all ArrayList<T> look the same
}

// Allowed: check raw type only
if (list instanceof ArrayList) { ... }
```

### Can't create generic arrays
```java
List<String>[] array = new List<String>[10]; // ERROR!
// Type erasure would make it List[] at runtime, unsafe

// Workaround: use List of Lists
List<List<String>> listOfLists = new ArrayList<>();
```

### Can't use primitives
As mentioned — use wrapper classes (`Integer`, `Double`, etc.) or specialized streams.

---

## 5) Raw Types

### What they are
Using generic type without type parameter — legacy compatibility with pre-Java 5 code.

```java
List rawList = new ArrayList();        // Raw type
List<String> typedList = new ArrayList<>(); // Generic
```

### Why they're bad
Lose all type safety, enable runtime exceptions:

```java
List rawList = new ArrayList();
rawList.add("string");
rawList.add(42); // Compiles!

// Later, mixing with generics:
List<String> strings = rawList; // Warning, but compiles
for (String s : strings) {      // ClassCastException at runtime!
    System.out.println(s);
}
```

**Never use raw types in new code.** Compiler warnings exist for a reason.

---

## 6) Generics and Collections

### Type-safe collections
```java
Map<String, List<Order>> customerOrders; // Complex generic type
```

### Type safety enforcement
```java
Set<User> users = new HashSet<>();
users.add(new User("alice"));
// users.add("bob"); // Compile error!

// Type-safe retrieval
for (User u : users) { // Guaranteed User, no cast needed
    System.out.println(u.getName());
}
```

### Why it matters
- **Compile-time errors** instead of runtime crashes
- **Self-documenting** code: Map<String, User> is clearer than Map
- **IDE support**: autocomplete knows types

---

## 7) Common Developer Mistakes

### Wrong wildcard choice
```java
// Trying to add elements with extends (covariance)
public void badAdd(List<? extends Number> list) {
    // list.add(42); // ERROR! ? extends is for reading
}

// Trying to read specific type from super (contravariance)
public Number badGet(List<? super Integer> list) {
    // return list.get(0); // ERROR! Returns Object, not Number
    return null;
}
```

### PECS confusion
```java
// Wrong: trying to consume with extends
public void fillWithIntegers(List<? extends Number> list) {
    // list.add(42); // Compile error!
}

// Correct: consume with super
public void fillWithIntegers(List<? super Integer> list) {
    list.add(42); // OK
}
```

### Using raw types (see section 5)

### ClassCastException despite generics
```java
// Unsafe cast can break type safety
List<String> strings = new ArrayList<>();
List rawList = strings;               // OK (backward compatibility)
rawList.add(42);                      // Adds Integer!

// Still compiles, but runtime crash:
String s = strings.get(0);          // ClassCastException!
```

**Lesson**: Don't mix generics with raw types. Use `@SuppressWarnings("unchecked")` sparingly and only when truly safe.

---

## 8) Practical Advice

### When to use extends vs super
- **Reading** from list → `? extends T`
- **Writing** to list → `? super T`
- **Both** (flexible method) → use PECS pattern

### Writing universal methods
```java
// Generic method that works with any Comparable
public static <T extends Comparable<T>> T max(List<T> list) {
    T max = list.get(0);
    for (T item : list) {
        if (item.compareTo(max) > 0) {
            max = item;
        }
    }
    return max;
}
```

### Avoiding ClassCastException
1. Never use raw types
2. Don't suppress unchecked warnings without understanding
3. Be careful with varargs and generics (heap pollution)
4. Use `Collections.checkedList()` when interfacing with legacy code

### Generic method tips
```java
// Bounded type parameter
public <T extends Number> double sum(List<T> numbers) {
    double sum = 0;
    for (T n : numbers) {
        sum += n.doubleValue();
    }
    return sum;
}
```

---

## 9) Short Interview Summary

Generics provide compile-time type safety by parameterizing classes and methods with type variables, eliminating the need for casts and preventing ClassCastException at runtime. Java implements generics through type erasure, replacing type parameters with Object or bounds after compilation — this prevents using primitives directly and instantiating type parameters like `new T()`. Wildcards (`? extends` for reading/producing, `? super` for writing/consuming) add flexibility following the PECS rule. Raw types compromise type safety and should never be used in new code. Common pitfalls include mixing raw types with generics, confusing `extends` with `super`, and attempting operations that violate wildcard constraints. Use bounded type parameters `<T extends SomeType>` for generic algorithms that need specific behavior.

---

## 10) Typical Interview Questions with Answers

### Q1: Why can't we use primitives with generics?
**A**: Due to type erasure, type parameters become Object at runtime. Primitives aren't objects, so you must use wrapper classes (Integer, Double) with autoboxing overhead.

### Q2: Explain the PECS rule with an example.
**A**: Producer Extends, Consumer Super. If you read from a list (producer), use `? extends`. If you write to a list (consumer), use `? super`. Example: `copy(List<? extends T> src, List<? super T> dest)` reads from src and writes to dest.

### Q3: What's the difference between `List<Object>` and raw `List`?
**A**: `List<Object>` is type-safe and accepts any object but maintains type information. Raw `List` disables generics entirely, allowing unsafe operations and compiler warnings. Never use raw types.

### Q4: Why can't we do `new T()` or `instanceof T` in generics?
**A**: Type erasure removes type parameters at compile time. At runtime, `T` doesn't exist — it's just Object. The JVM can't instantiate or check type of something that was erased.

### Q5: When would you use `? super` instead of `? extends`?
**A**: Use `? super` when you need to add elements to a collection (like adding Integers to a List of Number or Object). Use `? extends` when you're only reading elements.

### Q6: What's type erasure and why does Java use it?
**A**: Type erasure replaces type parameters with their bounds or Object after compilation. Java uses it for backward compatibility with pre-generics bytecode and to avoid creating new JVM-level types.

### Q7: Why does this code compile but fail at runtime?
```java
List<Integer> ints = new ArrayList<>();
List raw = ints;
raw.add("string");
Integer i = ints.get(0);
```
**A**: Raw type bypasses generics. Adding to raw list puts String into List<Integer>. The compiler trusts the generics, but at runtime the cast fails. Mixing raw and generic types breaks type safety.

### Q8: What's a bounded type parameter and when to use it?
**A**: `<T extends SomeType>` restricts T to SomeType or its subclasses. Use when your generic method/class needs to call methods from SomeType, like `<T extends Comparable<T>>` for sorting.

### Q9: Can you overload methods with different generic parameters?
**A**: No — after erasure they become identical. `void process(List<String>)` and `void process(List<Integer>)` both erase to `void process(List)`, causing compile error.

### Q10: What's heap pollution and how to avoid it?
**A**: Heap pollution occurs when a variable of a parameterized type refers to an object of a different parameterized type (via raw types or unchecked casts). Avoid by not mixing raw types with generics and not suppressing unchecked warnings without verification.
