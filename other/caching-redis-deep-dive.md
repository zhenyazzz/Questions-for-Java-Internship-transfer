# Caching & Redis — Deep Dive for Middle Java Developer

Practical guide to caching strategies, Redis internals, and real-world pitfalls.

---

## 1) Caching Fundamentals

### Why We Cache
- **Reduce latency**: Memory access ~100ns vs database ~10ms (100x difference)
- **Decrease load**: Protect backend from repetitive expensive queries
- **Improve throughput**: Serve more requests with same resources
- **Handle bursts**: Absorb traffic spikes without crashing database

### When NOT to Use Cache
- **Frequently changing data**: Cache invalidation becomes nightmare
- **Small dataset**: If it fits in DB buffer pool, cache adds complexity without benefit
- **Write-heavy workloads**: Cache invalidation overhead exceeds gains
- **Strong consistency required**: Cache introduces eventual consistency
- **Cold data**: Data accessed once and never again wastes memory

**Real example**: Don't cache user passwords or financial transaction logs. Cache product catalog, user sessions, or computed aggregates.

---

## 2) Caching Strategies

### Cache-Aside (Lazy Loading)
Most common pattern. Application manages cache explicitly.

```java
public Product getProduct(String id) {
    // 1. Try cache first
    Product cached = redis.get("product:" + id);
    if (cached != null) {
        return cached;
    }
    
    // 2. Cache miss - load from database
    Product product = db.findById(id);
    if (product != null) {
        redis.setex("product:" + id, 3600, product); // TTL 1 hour
    }
    return product;
}
```

**Pros**: Simple, resilient (cache down → app still works)
**Cons**: Cache miss penalty, potential inconsistency window

### Write-Through
Write goes to cache and database synchronously.

```java
public void updateProduct(Product product) {
    db.save(product);        // 1. Save to DB
    redis.set(key, product);  // 2. Immediately update cache
}
```

**Pros**: Cache always consistent
**Cons**: Slower writes (two operations), unnecessary cache warming for rarely accessed data

### Write-Behind (Write-Back)
Write goes to cache first, async write to database.

```java
public void updateProduct(Product product) {
    redis.set(key, product);              // 1. Update cache immediately
    messageQueue.send(asyncWrite(product)); // 2. Async persist to DB
}
```

**Pros**: Fast writes, can batch DB updates
**Cons**: Risk of data loss if cache fails before persistence, complexity

**Use case**: Write-heavy analytics counters, high-throughput logging.

### Eviction Strategies
When cache is full, what to remove:

- **LRU** (Least Recently Used): Default in most caches. Good for general workloads.
- **LFU** (Least Frequently Used): Keeps popular items. Better for access patterns with some hot keys.
- **FIFO**: Simple, rarely optimal.
- **TTL-based**: Evict expired items first.
- **Random**: Fast, but suboptimal hit rates.

**Redis**: Supports volatile-lru, allkeys-lru, volatile-ttl, allkeys-random, noeviction.

---

## 3) Cache Invalidation (The Hard Problem)

### Why Cache Invalidation Is Hard
> "There are only two hard things in Computer Science: cache invalidation and naming things."

Cache and database can diverge. Users see stale data.

### When Cache Becomes Stale
- Database updated, cache not refreshed
- TTL expires after data change
- Race conditions between read and write

### Invalidation Strategies

**TTL (Time To Live)**
```java
redis.setex("user:" + id, 300, user); // Expires in 5 minutes
```
- Simple, automatic cleanup
- Stale data possible until TTL expires

**Explicit Invalidation**
```java
@CacheEvict(value = "products", key = "#product.id")
public void updateProduct(Product product) {
    repository.save(product);
}
```
- Immediate consistency
- Must remember to invalidate everywhere data changes

**Write-Through / Write-Behind**
- Cache always has latest (see strategy section)

### Consistency Problems

**Race Condition Example:**
```
Thread A: Read product (cache miss) → Load from DB (v1)
Thread B: Update product in DB (v2) → Invalidate cache
Thread A: Write v1 to cache (stale data!)
```

**Solutions:**
1. **Cache-aside with delete, not update**: On write, delete cache entry. Next read loads fresh data.
2. **Optimistic locking**: Include version/timestamp in cache key
3. **Short TTL**: Accept small inconsistency window

**Pattern: Delete vs Update**
```java
// Better: Delete on write
public void updateProduct(Product product) {
    db.save(product);
    redis.del("product:" + product.getId()); // Delete, don't update
}

// Next read will load fresh data
```

---

## 4) Redis Under the Hood

### In-Memory Storage
Redis stores all data in RAM (with optional persistence). This makes it extremely fast but limits dataset size to available memory.

**O(1) operations**: Most Redis commands are constant time regardless of data size.

### Data Structures

**String**
- Binary safe, max 512MB
- Use: Simple values, counters, serialized objects
```
SET user:1:name "Alice"
INCR view_count:article:123
```

**Hash**
- Field-value pairs in one key
- Use: Object properties, avoiding key explosion
```
HSET user:1 name "Alice" age 30 city "NYC"
HGET user:1 name
```

**List**
- Linked list, fast push/pop at ends
- Use: Queues, activity streams
```
LPUSH queue:jobs "job1"
RPOP queue:jobs
```

**Set**
- Unordered unique collection
- Use: Tags, followers, relationship checks
```
SADD user:1:tags "java" "redis"
SISMEMBER user:1:tags "java"
```

**Sorted Set (ZSet)**
- Set with scores, ordered by score
- Use: Leaderboards, time-series, rate limiting
```
ZADD leaderboard 1000 "player1"
ZREVRANGE leaderboard 0 9  # Top 10
```

### Storage Internals
Redis uses:
- **SDS** (Simple Dynamic String) for strings
- **Skip lists** for Sorted Sets
- **Zip lists / Quick lists** for small lists/hashes
- **Int sets** for integer-only sets

Structures adapt based on size for memory efficiency.

---

## 5) TTL and Expiration

### How TTL Works
```
EXPIRE key 300  # Key expires in 300 seconds
TTL key         # Returns remaining seconds
```

### Passive vs Active Expiration

**Passive (Lazy)**
- Check expiration only when key is accessed
- Expired key deleted on read attempt
- Memory efficient but stale keys can linger

**Active (Background)**
- Redis randomly samples keys and deletes expired ones
- Runs 10 times per second by default
- Eventually removes all expired keys

**Problem**: If many keys expire at once, memory spike before cleanup.

### TTL Problems

**Thundering Herd on Expiration**
```
1. Popular key expires
2. Thousands of requests simultaneously hit DB
3. Database overload
```

**Solution: Stale-While-Revalidate or Per-Key Randomization**
```java
// Add random jitter to TTL
int ttl = 3600 + random.nextInt(300); // 1 hour ± 5 minutes
redis.setex(key, ttl, value);
```

This prevents mass expiration at same time.

---

## 6) Redis Use Cases

### Query Caching
Already covered. Cache database query results.

### Session Storage
```java
// Store user session
redis.setex("session:" + sessionId, 1800, serialize(userSession));

// Check on every request
UserSession session = deserialize(redis.get("session:" + sessionId));
if (session == null) {
    throw new UnauthorizedException();
}
```

**Why Redis**: Fast lookup, automatic expiration, shared across app servers.

### Rate Limiting
```java
public boolean allowRequest(String userId) {
    String key = "rate_limit:" + userId;
    
    Long current = redis.incr(key);
    if (current == 1) {
        redis.expire(key, 60); // 1-minute window
    }
    
    return current <= 100; // Allow 100 requests per minute
}
```

**Sliding Window with Sorted Sets**:
```
ZADD rate_limit:user1 current_timestamp timestamp
ZREMRANGEBYSCORE rate_limit:user1 0 (current_timestamp - 60000)
ZCARD rate_limit:user1  # Count in last minute
```

### Distributed Queue
```
LPUSH queue:email "send_welcome_email:alice@example.com"
BRPOP queue:email 0  # Blocking pop, waits for items
```

**Better for production**: Use Redis Streams or dedicated message broker (RabbitMQ, Kafka).

---

## 7) Concurrency & Race Conditions

### Cache Stampede (Thundering Herd)
When cache expires, many threads simultaneously try to:
1. Check cache (miss)
2. Query database
3. Write back to cache

**Result**: Database overload from identical queries.

### Solutions

**1. Locking (Mutex)**
```java
public Product getProductWithLock(String id) {
    String cacheKey = "product:" + id;
    String lockKey = "lock:" + id;
    
    Product cached = redis.get(cacheKey);
    if (cached != null) return cached;
    
    // Try to acquire lock
    if (redis.setnx(lockKey, "1", 10)) {  // 10-second lock
        try {
            Product product = db.findById(id);
            redis.setex(cacheKey, 3600, product);
            return product;
        } finally {
            redis.del(lockKey);
        }
    } else {
        // Someone else is loading, wait and retry cache
        Thread.sleep(100);
        return redis.get(cacheKey); // Should be populated now
    }
}
```

**2. Stale-While-Revalidate**
```
1. Serve stale data from cache immediately
2. Trigger async background refresh
3. Update cache with fresh data
```

**3. Probabilistic Early Expiration**
Refresh cache before TTL expires, based on probability.

### Distributed Lock with Redis
```
SET resource:lock my_random_value NX PX 30000
# Only set if not exists, expire in 30 seconds
```

**Redisson** (Java library) provides robust distributed locks using this mechanism.

---

## 8) Distributed Caching

### Why Distributed Cache
- Multiple app instances share same cache state
- Cache survives app restart
- Centralized memory pool

### Challenges

**Cache Synchronization**
```java
// Instance 1 (Server A)
productService.updateProduct(product);
// Updates DB, clears cache

// Instance 2 (Server B)
// Has stale local cache if not using shared Redis
```

**Solution**: Use Redis (shared) not Caffeine (local only) for multi-instance apps.

**Consistency Across Regions**
- Redis replication lag between data centers
- Consider cache per region or eventual consistency acceptance

---

## 9) Persistence in Redis

### RDB (Redis Database)
- Point-in-time snapshots
- Binary file, compact
- Configured: `save 900 1` (save if 1+ changes in 15 min)

**Pros**: Fast recovery, small file, minimal performance impact
**Cons**: Data loss since last snapshot (minutes)

### AOF (Append Only File)
- Log of every write command
- Can be `always`, `everysec` (default), or `no`

**Pros**: Better durability (max 1 second loss)
**Cons**: Larger file, slower recovery, more disk I/O

### Hybrid (Redis 4.0+)
- RDB snapshot + AOF for writes since snapshot
- Best of both worlds

**For cache use cases**: Often acceptable to disable persistence (it's a cache, can be rebuilt). For session store or queue: enable AOF.

---

## 10) Performance

### Why Redis Is Fast
1. **In-memory**: No disk I/O during operations
2. **Single-threaded**: No context switching, lock contention
3. **Event loop**: Efficient I/O multiplexing (epoll/kqueue)
4. **Simple data structures**: O(1) operations
5. **Binary protocol**: RESP is efficient to parse

### When Redis Becomes Bottleneck
- **Large values**: Storing 10MB JSON objects → memory pressure, slow transfer
- **Blocking operations**: KEYS command (O(N)), unoptimized Lua scripts
- **Memory full**: Eviction kicks in, latency spikes
- **Network latency**: RTT overhead for many small requests → use pipelines

**Optimization: Pipelining**
```java
Pipeline pipe = redis.pipelined();
for (String key : keys) {
    pipe.get(key);  // Queue commands
}
List<Object> results = pipe.syncAndReturnAll();  // Execute in batch
```

---

## 11) Pitfalls

### Stale Data
**Symptom**: Users see old information after update.

**Causes**:
- TTL too long
- Missing invalidation on write path
- Race conditions (delete-then-write pattern)

**Fix**: Use delete-on-write pattern, shorter TTLs, or versioning.

### Data Loss
**Scenario**: Redis restarts without persistence, cache cold, database overwhelmed.

**Mitigation**: Enable AOF for critical data, implement circuit breaker for cache misses.

### Memory Overflow
**Symptom**: Redis OOM, evictions causing cache thrashing.

**Causes**:
- Unbounded key growth (no TTL)
- Storing large objects
- No eviction policy configured

**Fix**: Set TTL on all keys, monitor memory usage, configure `maxmemory-policy`, use smaller data structures (hash instead of multiple keys).

### Wrong Invalidation
```java
// Developer forgets to invalidate related caches
@CacheEvict("product")
public void updateProduct(Product p) { ... }

// But product list cache still has old data!
```

**Fix**: Cache tagging, broader invalidation patterns, or accept eventual consistency.

---

## 12) Spring Cache Integration

### Spring Cache Abstraction
```java
@EnableCaching
@SpringBootApplication
public class Application { }
```

### @Cacheable
```java
@Cacheable(value = "products", key = "#id")
public Product getProduct(String id) {
    return repository.findById(id);  // Only executed on cache miss
}
```

### @CacheEvict
```java
@CacheEvict(value = "products", key = "#product.id")
public void updateProduct(Product product) {
    repository.save(product);
}

@CacheEvict(value = "products", allEntries = true)
public void clearAllProducts() {
    // Clear entire cache
}
```

### @CachePut
```java
@CachePut(value = "products", key = "#product.id")
public Product updateProduct(Product product) {
    return repository.save(product);  // Updates cache with result
}
```

### Configuration
```yaml
spring:
  cache:
    type: redis
  redis:
    host: localhost
    port: 6379
    timeout: 2000ms
```

### Conditional Caching
```java
@Cacheable(value = "products", key = "#id", unless = "#result == null")
public Product getProduct(String id) { ... }
```

### Pitfalls with Spring Cache
1. **Self-invocation**: `@Cacheable` on internal method calls doesn't work (proxy issue)
2. **No TTL by default**: Must configure `spring.cache.redis.time-to-live`
3. **Serialization**: Default JDK serialization is slow; configure JSON or Kryo
4. **Cache name explosion**: Dynamic keys can cause memory issues

---

## Interview Questions

**Q: Cache-aside vs Write-Through — when to use each?**
**A**: Cache-aside for read-heavy, eventually consistent scenarios. Write-Through when strong consistency and cache freshness are critical, accepting slower writes.

**Q: How to prevent cache stampede?**
**A**: Use distributed locks (mutex) so only one thread reloads cache, or serve stale data while refreshing in background (stale-while-revalidate).

**Q: Why not just increase TTL to reduce database load?**
**A**: Longer TTL increases stale data window. For dynamic data, users see outdated information longer. Balance based on data change frequency.

**Q: Redis is single-threaded. How does it handle 10k concurrent connections?**
**A**: Event loop with I/O multiplexing (epoll). While one command executes, others are queued. For CPU-heavy operations, use Redis Cluster to shard load.

**Q: When would you choose Hash over multiple String keys?**
**A**: Hash when storing object properties together to avoid key namespace explosion and reduce memory overhead (Redis hashes are memory-efficient for small objects).

**Q: RDB vs AOF for session storage?**
**A**: AOF (everysec) for sessions to minimize data loss on restart. Sessions can be reconstructed, but user logout is poor UX.

**Q: @Cacheable not working on internal method call?**
**A**: Spring Cache uses proxies. Internal calls bypass proxy. Solution: self-injection or refactor to separate service.

---

## Summary

Caching reduces latency and database load but introduces eventual consistency challenges requiring careful invalidation strategies. Cache-aside (lazy loading) is most common, while write-through ensures consistency at write-time cost. Cache stampede occurs when many threads simultaneously reload expired data—prevent with locks or background refresh. Redis provides in-memory data structures (String, Hash, List, Set, Sorted Set) with O(1) operations and configurable TTL-based expiration. For distributed systems, Redis centralizes cache state across application instances. Spring Cache abstraction simplifies integration with @Cacheable and @CacheEvict annotations, but requires proper TTL configuration and awareness of proxy-based limitations. Common pitfalls include stale data from missed invalidations, memory overflow without eviction policies, and thundering herds on mass expiration—mitigate through jittered TTLs, bounded caches, and careful consistency design.
