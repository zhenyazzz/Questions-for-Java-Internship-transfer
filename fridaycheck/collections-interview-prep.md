# Java Collections — Middle Java Interview Prep

Interview-focused guide covering practical aspects of Java collections without deep JDK internals.

---

## 1) How HashMap Works

### What is hash
A **hash** is an integer computed from an object's content using `hashCode()`. It should distribute values evenly across the integer range.

### How bucket (index) is calculated
HashMap uses the hash to find an **index** in an internal array (table):
- `index = (n - 1) & hash` (where n is array size, power of two)
- This is equivalent to `hash % n` but faster

### Internal structure
HashMap stores data in an **array of buckets**:
- Each bucket can hold one or more entries
- Initially: simple array slots
- When collision occurs: entries chain in a **linked list**
- When list grows large (>= 8 entries): converts to **red-black tree** for O(log n) lookup in that bucket

### What happens on collisions
Multiple keys can produce the same index (collision). They are stored in a chain/tree at that bucket. On `get()`, HashMap checks `hash` first (fast), then `equals()` to find the exact key.

### Put and Get flow
**put(key, value)**:
1. Calculate hash from key
2. Find bucket index
3. If bucket empty: create new node
4. If occupied: traverse chain/tree, check `equals()` for existing key
5. If key exists: replace value
6. If not: add new entry

**get(key)**:
1. Calculate hash
2. Go to bucket
3. Check entries by `hash` then `equals()`

---

## 2) equals and hashCode

### Why they are needed
- `hashCode()` determines which bucket to look in
- `equals()` determines if two keys are actually the same

### The contract (briefly)
1. If `a.equals(b) == true`, then `a.hashCode() == b.hashCode()` must be true
2. Equal objects must have equal hash codes
3. Unequal objects may have same hash codes (collision)

### Why correct implementation matters
If you break the contract, HashMap breaks:
- Object "disappears" (can't be found even though it was put)
- Duplicates appear (two logically equal objects treated as different)

### Real-world bug example
```java
public class Person {
    private String email;
    
    // Only equals, no hashCode override!
    @Override
    public boolean equals(Object o) {
        return o instanceof Person && 
               ((Person)o).email.equalsIgnoreCase(this.email);
    }
}

Map<Person, String> map = new HashMap<>();
Person p1 = new Person("alice@example.com");
map.put(p1, "Alice");

Person p2 = new Person("ALICE@example.com"); // logically equal to p1
// But different hashCode (Object's default - memory address)
String name = map.get(p2); // Returns null! Bug!
```

**Fix**: Always override `hashCode()` when overriding `equals()`, using the same fields.

---

## 3) Collisions

### What is a collision
When two different keys produce the **same bucket index**.

### How HashMap handles them
- Small number: linked list traversal (acceptable if list stays short)
- Large number (>= 8 entries): converts to red-black tree (O(log n) instead of O(n))

### Why bad hashCode kills performance
If `hashCode()` always returns the same value (e.g., constant), all entries go to **one bucket**:
- Instead of O(1) average, you get O(n) or O(log n) for all operations
- This defeats the purpose of HashMap entirely

**Example of terrible hashCode**:
```java
@Override
public int hashCode() {
    return 42; // All objects go to same bucket!
}
```

---

## 4) ConcurrentHashMap

### How it differs from HashMap
- **HashMap**: NOT thread-safe; concurrent modifications cause corruption (infinite loops in old JDKs, lost entries in modern ones)
- **ConcurrentHashMap**: Thread-safe without locking the entire map

### How it solves multithreading (conceptual level)
Instead of one big lock, it uses:
- **Lock striping**: different buckets can be locked independently
- **CAS operations**: for non-blocking reads and certain updates
- **Volatile reads**: for visibility without locking

Java 8+ implementation: each bucket head has its own synchronization mechanism; reads are usually lock-free.

### Why not use regular HashMap in multithreaded code
- Race conditions on internal structure
- Infinite loops during resize (pre-Java 8)
- Lost updates
- Corrupted size

### When to use ConcurrentHashMap
- Shared map accessed by multiple threads
- Cache that needs thread-safe access
- Any concurrent read/write scenario

**Alternative for simple cases**: `Collections.synchronizedMap()` - but it uses coarse locking (slower under high contention).

---

## 5) When to Use List / Set / Map

### List
- Use when: **order matters** and **duplicates allowed**
- **ArrayList**: default choice, O(1) get by index, fast iteration
- **LinkedList**: rarely needed; use only for frequent insertions/removals at ends

### Set
- Use when: **uniqueness matters**, order usually not important
- **HashSet**: backed by HashMap, O(1) operations, no ordering guarantee
- **LinkedHashSet**: maintains insertion order
- **TreeSet**: sorted order (red-black tree), O(log n)

### Map
- Use when: **key-value pairs**, fast lookup by key
- **HashMap**: default, unordered
- **LinkedHashMap**: insertion or access order
- **TreeMap**: sorted by key, O(log n)

### Quick note on HashSet internals
HashSet is actually a **wrapper around HashMap**:
- Elements are keys in the backing map
- Values are dummy objects (`PRESENT`)
- All HashMap behaviors apply (hashing, collisions, null handling)

---

## 6) Typical Developer Mistakes

### Wrong equals/hashCode
- Only overriding `equals()` and forgetting `hashCode()`
- Using mutable fields in hashCode (object "disappears" after mutation)
- Inconsistent logic between the two methods

### Using HashMap in multithreaded code
```java
// BAD: shared mutable HashMap
public class OrderCache {
    private Map<String, Order> cache = new HashMap<>(); // Dangerous!
    
    public void addOrder(Order o) {
        cache.put(o.getId(), o); // Race condition!
    }
}
```
**Fix**: Use `ConcurrentHashMap` or proper synchronization.

### Wrong collection choice
- Using LinkedList when ArrayList would be faster
- Using Vector (legacy, synchronized on every operation - slow)
- Using TreeSet/TreeMap when HashSet/HashMap is sufficient (paying log n cost unnecessarily)

### Ignoring complexity
- Assuming all Map operations are O(1) (TreeMap is O(log n))
- Not considering memory overhead of boxed types vs primitives
- Not accounting for resize cost when initial capacity is unknown

---

## 7) Practical Advice

### How to choose collections
1. **Need ordering + allow duplicates?** → List (usually ArrayList)
2. **Need uniqueness?** → Set (usually HashSet)
3. **Need key-value lookup?** → Map (usually HashMap)
4. **Need sorting?** → TreeSet/TreeMap
5. **Concurrent access?** → ConcurrentHashMap, CopyOnWriteArrayList (rare reads), or external synchronization

### Writing correct equals/hashCode
```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Person)) return false;
    Person other = (Person) o;
    return Objects.equals(email, other.email);
}

@Override
public int hashCode() {
    return Objects.hash(email); // Use same fields as equals
}
```

### Production considerations
- **Immutable keys**: never modify objects used as HashMap keys
- **Initial capacity**: if you know the size, set it to avoid resizing
- **Load factor**: default 0.75 is usually fine
- **Null handling**: HashMap allows one null key; ConcurrentHashMap allows none
- **Memory**: be aware that each HashMap entry has overhead (object headers, references)

---

## 8) Short Interview Summary (5-7 sentences)

HashMap uses an array of buckets where entries are stored based on `hashCode()`. When multiple keys collide (same bucket), they're chained in a list or tree. Correct `equals()` and `hashCode()` are essential: if they disagree, objects can "disappear" from the map or duplicates can appear. For multithreaded access, use `ConcurrentHashMap` instead of regular `HashMap` to avoid race conditions and corruption. Choose collections based on your needs: List for ordered duplicates, Set for uniqueness, Map for key-value pairs. Common mistakes include wrong equals/hashCode contracts, using HashMap concurrently, and ignoring operation complexity. Always use immutable objects as HashMap keys and prefer `ConcurrentHashMap` for shared state.
