# NoSQL Databases Interview Preparation for Java Backend Engineers

> Target: Java backend engineer, Junior+/Middle final interview level.  
> Goal: explain NoSQL systems like an engineer who has seen production traffic, incidents, scaling limits, and consistency bugs.

---

# Table of Contents

- [Roadmap](#roadmap)
- [NoSQL Fundamentals](#nosql-fundamentals)
- [Types of NoSQL Databases](#types-of-nosql-databases)
- [MongoDB Architecture](#mongodb-architecture)
- [Data Modeling in MongoDB](#data-modeling-in-mongodb)
- [Indexing](#indexing)
- [Replication and High Availability](#replication-and-high-availability)
- [Sharding](#sharding)
- [Transactions and Consistency](#transactions-and-consistency)
- [Redis](#redis)
- [Cassandra Basics](#cassandra-basics)
- [Storage of Binary Data](#storage-of-binary-data)
- [Scaling and Performance](#scaling-and-performance)
- [NoSQL in Microservices](#nosql-in-microservices)
- [Monitoring and Production Operations](#monitoring-and-production-operations)
- [Common Production Problems](#common-production-problems)
- [CAP Theorem](#cap-theorem)
- [PACELC Theorem](#pacelc-theorem)
- [Interview Q&A](#interview-qa)
- [Final Revision Checklist](#final-revision-checklist)

---

# Roadmap

Think about NoSQL in this order:

1. **Data shape**: key-value, document, wide-column, graph.
2. **Access pattern**: which queries must be fast, predictable, and cheap?
3. **Distribution model**: replication for availability, sharding for capacity.
4. **Consistency contract**: which reads must see the latest write?
5. **Operational behavior**: indexes, hot keys, lag, memory, backups, failover.
6. **Application responsibility**: retries, idempotency, schema evolution, cache invalidation.

A strong interview answer does not say “NoSQL is faster.” It says: **NoSQL moves some responsibility from the database schema and optimizer to the application design and operations team.**

---

# NoSQL Fundamentals

## What it is
NoSQL databases are non-relational or not-only-relational databases optimized for specific data models, horizontal scaling, flexible schemas, high throughput, low latency, or distributed availability.

Examples:

- **MongoDB**: document database for JSON-like domain objects.
- **Redis**: in-memory key-value/data-structure store for cache, sessions, counters, locks.
- **Cassandra**: wide-column distributed store for massive write throughput and multi-node availability.
- **Neo4j**: graph database for relationship-heavy traversal queries.

## Why it exists
Relational databases are excellent for normalized data, joins, constraints, and strong transactions. Problems appear when systems need:

- Very high write volume.
- Flexible or frequently changing data structures.
- Low-latency lookup at large scale.
- Geo-distributed or always-on behavior.
- Horizontal scaling beyond one powerful machine.
- Data models where joins are too expensive or unnatural.

In a Spring Boot system, SQL may be perfect for orders and payments, while MongoDB stores product catalog documents, Redis caches sessions, Kafka moves domain events, and Cassandra stores high-volume time-series events.

## How it works internally
Most production NoSQL systems combine:

- **Partitioning**: split data across nodes by key range, hash, token, or shard key.
- **Replication**: keep multiple copies for failover, durability, and sometimes read scaling.
- **Local storage engines**: B-tree, LSM-tree, memory-first structures, append logs, compression.
- **Client-side or router-side routing**: decide which node owns the key or partition.
- **Consistency knobs**: choose how many replicas must acknowledge writes or reads.

### Horizontal vs vertical scaling

- **Vertical scaling**: buy a bigger machine. Simpler, but has hardware and cost limits.
- **Horizontal scaling**: add more machines. More capacity, but introduces partitions, replication lag, routing, rebalancing, and operational complexity.

### CAP intuition
In a distributed system, a network partition means nodes cannot reliably communicate. During that partition, the system must choose:

- **Consistency**: reject or delay operations that cannot be safely coordinated.
- **Availability**: continue accepting operations, possibly returning stale or conflicting data.

Partition tolerance is not optional once the database runs on multiple machines. Networks fail.

### Eventual consistency intuition
Eventual consistency means a write may not be visible everywhere immediately, but if no new writes happen and replication succeeds, replicas converge to the same value.

Example: after updating a user profile in MongoDB primary, a read from a lagging secondary may briefly return the old profile. In a product catalog this may be fine. In payment authorization it may not be.

## Production usage
Teams use NoSQL when they can design around explicit access patterns:

- Store an entire product page document in MongoDB to avoid joins on every catalog read.
- Store `user:{id}:session` in Redis with TTL for fast session lookup.
- Store IoT/device metrics in Cassandra partitioned by device and time bucket.
- Use Kafka events to update read models asynchronously.

The production mindset is: **model the query first, then model the data.**

## Common production problems

- Choosing NoSQL only because it is trendy.
- Treating MongoDB like SQL with many cross-document joins.
- Assuming eventual consistency is harmless.
- Missing indexes and causing full collection scans.
- Bad partition/shard keys causing one node to do most work.
- Forgetting that distributed retries create duplicate writes unless operations are idempotent.
- Ignoring backups because “we have replicas.” Replication copies corruption too.

## Tradeoffs

| Benefit | Cost |
|---|---|
| Flexible schema | Application must validate and migrate data carefully |
| Horizontal scale | More distributed systems complexity |
| Fast access for known patterns | Poor performance for unplanned ad-hoc queries |
| Denormalized reads | Duplicate data and consistency maintenance |
| Tunable consistency | More decisions and more failure modes |

## Interview traps

- Saying NoSQL means “no schema.” Production NoSQL still needs a schema contract, often enforced in code.
- Saying NoSQL is always faster. It is faster only for workloads it is modeled for.
- Confusing replication with sharding.
- Treating CAP as a vendor label instead of a failure-mode discussion.
- Ignoring operational issues: indexes, lag, memory, backups, and hot partitions.

## Key points to remember

- NoSQL is about **fit for data shape, access pattern, scale, and consistency**.
- Distribution solves capacity but creates consistency and operational problems.
- Flexible schema does not remove the need for data governance.
- Replication improves availability; sharding improves capacity.
- Strong NoSQL design starts from production queries, not entity diagrams.

---

# Types of NoSQL Databases

## What it is
NoSQL databases are usually grouped by the primary data model they optimize:

- **Key-value**: lookup by key.
- **Document**: JSON/BSON-like documents.
- **Column-family / wide-column**: rows with dynamic columns grouped by partition.
- **Graph**: nodes and edges optimized for relationship traversal.

## Why it exists
Different workloads need different query primitives. A single database model cannot be optimal for cache lookups, nested product documents, time-series writes, and graph traversal at the same time.

## How it works internally

| Type | Internal idea | Best access pattern |
|---|---|---|
| Key-value | Hash/map from key to value, often memory-first | `get(key)`, `set(key)`, counters, TTL |
| Document | Indexed documents with nested fields | Fetch/update aggregate documents |
| Wide-column | Partition key maps to sorted rows/columns | Huge write throughput by partition |
| Graph | Adjacent nodes and edges | Multi-hop relationship queries |

## Production usage

- **Redis**: cache product by ID, session tokens, rate-limit counters, distributed locks with caution.
- **MongoDB**: catalog, CMS, user profiles, order read models, feature flags.
- **Cassandra**: event logs, metrics, time-series data, high-volume append workloads.
- **Neo4j / graph stores**: fraud rings, social graphs, dependency graphs, recommendation paths.

## Common production problems

- Using Redis as a durable source of truth without persistence and recovery planning.
- Using MongoDB for deep graph traversal.
- Using Cassandra for flexible ad-hoc querying.
- Using a document database but modeling everything as normalized tables.
- Choosing a database before knowing the read/write patterns.

## Tradeoffs

- Key-value is extremely fast but query-poor.
- Document databases balance flexibility and queryability but need careful indexing.
- Wide-column systems scale writes well but require query-first table design.
- Graph databases are powerful for relationships but not general-purpose OLTP replacements.

## Interview traps

- Calling Cassandra “columnar” in the analytics sense. It is a wide-column operational database, not the same as Parquet/columnar warehouse storage.
- Saying MongoDB has no relationships. It supports embedding and references; it just does not encourage join-heavy OLTP design.
- Treating Redis Pub/Sub as a durable message broker. Plain Pub/Sub does not persist messages for offline consumers.

## Key points to remember

- Pick NoSQL type by **dominant access pattern**.
- Wrong database type creates unnatural queries, operational pain, or consistency bugs.
- Polyglot persistence is normal: one service ecosystem may use SQL, MongoDB, Redis, Kafka, and object storage together.

---

# MongoDB Architecture

## What it is
MongoDB is a document database that stores BSON documents in collections, supports rich indexes and aggregation, and provides high availability through replica sets and horizontal scale through sharding.

## Why it exists
MongoDB fits applications where domain objects are naturally document-shaped and reads often need an aggregate object at once: product catalog, content pages, user preferences, order summaries, notification templates.

## How it works internally

### BSON documents
MongoDB stores documents as **BSON**, a binary JSON-like format with additional types such as `ObjectId`, dates, decimals, and binary data.

### Collections
A collection groups documents. Unlike a SQL table, documents in one collection do not need identical fields, but production systems usually enforce expected structure through application validation, JSON schema validation, or both.

### Dynamic schema
Dynamic schema allows incremental changes. It does not mean random structure is acceptable. A Java service using Spring Data MongoDB still needs DTOs, validation, compatibility handling, and migration strategy.

### ObjectId
`ObjectId` is MongoDB's common `_id` type. It is globally unique enough for distributed generation and includes a timestamp component. Teams may still use business IDs when they need external stable identifiers.

### Embedded documents and denormalization
MongoDB commonly embeds related data inside a parent document. Example: a product document embeds localized descriptions, attributes, and image metadata. This makes reads fast because one query returns the aggregate.

### Reference-based relations and lack of JOIN-first design
MongoDB supports references by storing IDs and supports `$lookup` in aggregation, but production OLTP design avoids turning every request into many joins. If the page always needs product and current category summary together, embed or duplicate the needed fields.

### Aggregation pipeline basics
The aggregation pipeline processes documents through stages such as `$match`, `$project`, `$group`, `$sort`, `$lookup`, and `$unwind`. It is useful for reporting, transformations, and read models, but heavy aggregation on hot paths must be measured carefully.

### Read/write flow
A typical write goes to the primary node, updates the storage engine, records journal/operation log information, and replicates to secondaries. Reads may go to primary or secondaries depending on read preference and consistency needs.

### WiredTiger basics
WiredTiger is MongoDB's default storage engine. It provides document-level concurrency, caching, compression, and durable storage behavior. The practical point: MongoDB performance depends heavily on working set size, indexes, memory, and disk latency.

### Journaling basics
Journaling helps MongoDB recover after crashes by recording durable write information before data files are fully flushed. It improves crash safety but does not replace backups or majority replication.

## Production usage

A Spring Boot catalog service might use:

```java
@Document("products")
class ProductDocument {
    @Id String id;
    String sku;
    String title;
    List<Attribute> attributes;
    PriceSnapshot price;
    List<ImageMetadata> images;
    Instant updatedAt;
}
```

This avoids joining `products`, `attributes`, `prices`, and `images` on every product page read. Price source-of-truth may still live elsewhere; MongoDB may store a read-optimized snapshot updated by Kafka events.

## Common production problems

- Huge documents close to the BSON document limit.
- Arrays growing forever, such as storing all user events inside one user document.
- Updating the same document too frequently, creating a hot document.
- Relying on `$lookup` for every request and losing MongoDB's main advantage.
- Letting schemas drift without versioning.
- Misunderstanding secondary reads and reading stale data.

## Tradeoffs

| Advantage | Disadvantage |
|---|---|
| Natural JSON-like model for Java APIs | Denormalization can duplicate data |
| Flexible schema | Schema drift and migration complexity |
| Powerful indexes and aggregation | Bad indexes cause severe latency |
| Replica sets and sharding | Operational complexity increases quickly |
| Single-document atomicity | Multi-document consistency requires more care |

## Interview traps

- “MongoDB has no schema” is an amateur answer. It has flexible schema; production code still needs schema discipline.
- “MongoDB has no joins” is incomplete. `$lookup` exists, but join-heavy modeling is usually a smell.
- “Denormalization is always good” is wrong. It improves reads but complicates updates and consistency.

## Key points to remember

- MongoDB works best when documents represent aggregates used by application requests.
- Embedding optimizes read locality; referencing controls duplication and document growth.
- Storage engine, indexes, and working set determine real latency.
- MongoDB design is access-pattern-first, not normalization-first.

---

# Data Modeling in MongoDB

## What it is
MongoDB data modeling is the practice of designing documents, references, indexes, and update flows around real application access patterns.

## Why it exists
SQL modeling often starts with normalized entities. MongoDB modeling starts with: **what does the service need to read and write atomically and frequently?**

## How it works internally

### Access-pattern-first design
Before designing collections, list queries:

- `GET /products/{id}` by ID.
- `GET /products?category=phone&brand=...` with filters.
- `PATCH /products/{id}/attributes` by admin users.
- `GET /users/{id}/orders?cursor=...`.

Then design documents and indexes to support these paths.

### Embedding vs referencing

Embed when:

- Child data is read with parent.
- Child lifecycle belongs to parent.
- Child count is bounded.
- Atomic update with parent matters.

Reference when:

- Child data is large or unbounded.
- Child is shared by many parents.
- Child changes frequently and duplication would be expensive.
- Independent querying is required.

### One-to-many modeling

- **One-to-few**: embed addresses inside user.
- **One-to-many bounded**: embed recent order items in order.
- **One-to-many unbounded**: reference comments/events in separate collection and paginate.

### Many-to-many modeling
Use references or duplicated summary fields. Example: products and categories can store category IDs in products plus category summary in read models. Keep update propagation explicit, often through Kafka events.

### Large document and hot document problems
MongoDB documents have a maximum size, and large documents increase memory and network cost. A single document updated by thousands of requests per second becomes a contention hotspot.

Bad example: one `dailyStats` document with a counter for every event update. Better: bucket by time, shard key, or use Redis/Cassandra for counters/events.

### Schema evolution and versioning
Strategies:

- Add optional fields with safe defaults.
- Use `schemaVersion` for documents requiring migration logic.
- Backfill asynchronously with batch jobs.
- Support old and new shapes during rolling deployment.
- Use application validation to prevent invalid new documents.

## Production usage

In a Spring Boot microservice, MongoDB documents should align with service boundaries. If `catalog-service` owns product documents, `order-service` should not directly mutate them. It should consume product events or call an API.

## Common production problems

- Copying relational schema into MongoDB collection-per-table design.
- Embedding unbounded arrays such as all login events in user document.
- Duplicating data without an update strategy.
- Designing only for writes and forgetting read filters.
- Designing only for current version and breaking old documents.

## Tradeoffs

- Embedding: faster reads, better locality, simpler atomic updates, but duplication and growth risk.
- Referencing: smaller documents and less duplication, but more round trips, `$lookup`, or application joins.
- Denormalization: production performance tool, not a free lunch.

## Interview traps

- Interviewers often ask “embed or reference?” They want your reasoning, not a fixed rule.
- They may check whether you know MongoDB single-document atomicity makes embedded aggregates attractive.
- They may ask about large arrays to see if you understand unbounded growth.

## Key points to remember

- Model MongoDB around **queries, atomicity, and growth boundaries**.
- Embed bounded, owned, frequently-read-together data.
- Reference large, shared, frequently-changing, or unbounded data.
- Plan schema evolution before production data accumulates.

---

# Indexing

## What it is
An index is an auxiliary data structure that lets MongoDB find matching documents without scanning the entire collection.

## Why it exists
Without indexes, queries become full collection scans. At small data size that may look fine; at production size it becomes high latency, CPU burn, lock pressure, and incident tickets.

## How it works internally
MongoDB indexes are B-tree-like structures. The database can traverse ordered index entries to locate matching documents, support sort order, enforce uniqueness, and sometimes answer queries directly from the index.

### Main index types

- **Single field index**: supports queries like `{ email: "a@x.com" }`.
- **Compound index**: supports multi-field filters and sorts, such as `{ tenantId: 1, status: 1, createdAt: -1 }`.
- **Multikey index**: indexes array values.
- **Text index**: supports text search, though dedicated search engines may be better for advanced search.
- **TTL index**: expires documents automatically after time.
- **Unique index**: enforces uniqueness, such as email per tenant.
- **Sparse index**: includes only documents where the indexed field exists.
- **Partial index**: includes documents matching a filter, such as active users only.

### Cardinality
Cardinality means how selective a field is. High-cardinality fields like `userId` or `email` usually filter well. Low-cardinality fields like `status` alone often do not.

### Covered queries
A covered query can be answered fully from the index without reading documents. This is powerful for list endpoints that return only indexed fields.

### Explain plans
`explain()` shows whether MongoDB uses an index, how many documents were examined, and whether the query spills or sorts inefficiently.

## Production usage

A common Spring Boot endpoint:

```text
GET /orders?tenantId=t1&status=PAID&cursor=2026-01-01T10:00:00Z
```

A good index may be:

```javascript
db.orders.createIndex({ tenantId: 1, status: 1, createdAt: -1, _id: -1 })
```

This supports tenant filtering, status filtering, stable sort, and cursor-based pagination.

## Common production problems

- Missing index on a high-traffic endpoint.
- Wrong compound index field order.
- Too many indexes slowing writes and consuming memory/disk.
- Indexing low-cardinality fields alone.
- Sorting on a field not supported by the index.
- Creating indexes in production without understanding build impact.
- Relying on offset pagination with large skips.

## Tradeoffs

| Index benefit | Index cost |
|---|---|
| Faster reads | Slower writes because indexes must be updated |
| Efficient sorting | More disk and memory usage |
| Uniqueness enforcement | More operational care during migrations |
| TTL cleanup | Expiration is not exact real-time deletion |

## Interview traps

- “Add indexes to everything” is wrong. Indexes are not free.
- Compound index order matters: equality fields usually first, then range/sort fields.
- A query can have an index and still be slow if selectivity is poor.
- TTL indexes are background cleanup, not precise schedulers.

## Key points to remember

- Indexes are production-critical for MongoDB latency.
- Design compound indexes from real query filters and sort order.
- Use `explain()` before and after changes.
- Too many indexes hurt writes, memory, disk, and deployments.

---

# Replication and High Availability

## What it is
Replication stores copies of data on multiple nodes so the database can survive node failure and provide failover.

## Why it exists
Production systems need durability and availability. A single database node is a single point of failure.

## How it works internally

### Replica sets
A MongoDB replica set usually has one **primary** and multiple **secondary** nodes.

- Writes go to the primary.
- Secondaries replicate operations from the primary.
- If primary fails, eligible members elect a new primary.

### Elections and failover
When the primary becomes unavailable, nodes vote. A node needs majority support to become primary. This prevents split brain where two primaries accept conflicting writes.

### Read preference
Read preference controls where reads go:

- `primary`: freshest typical reads.
- `secondary`: can reduce primary load but may be stale.
- `primaryPreferred`, `secondaryPreferred`, `nearest`: useful but require consistency awareness.

### Write concern
Write concern defines how many nodes must acknowledge a write. `majority` improves durability and failover safety but increases latency.

### Read concern
Read concern controls visibility guarantees, such as local, majority-committed, or snapshot reads in transactions.

### Replication lag
Secondaries can fall behind the primary due to network, disk, CPU, index builds, or high write volume. Reads from secondaries may be stale.

## Production usage

For an order placement flow:

- Use primary reads and majority writes for state that affects customer-visible correctness.
- Avoid reading from secondaries immediately after a write if the user expects read-your-writes behavior.
- Monitor replication lag and election events.

For analytics dashboards:

- Secondary reads may be acceptable if data can lag by seconds.

## Common production problems

- Reading stale data from secondaries after writes.
- Using weak write concern and losing acknowledged writes during failover.
- Assuming replicas replace backups.
- Elections causing temporary write unavailability.
- Hidden lag causing inconsistent user experience.

## Tradeoffs

- More replicas improve fault tolerance but increase replication cost.
- Majority write concern improves safety but adds latency.
- Secondary reads improve read scale but can break consistency expectations.
- Fast failover still means applications must handle transient errors and retry safely.

## Interview traps

- Replication is not sharding. Replication copies data; sharding splits data.
- Failover is not invisible to applications. Drivers retry some operations, but services need idempotency.
- “Eventual consistency” is not a bug if the system explicitly accepts it.

## Key points to remember

- Replica sets provide high availability and durability.
- Primary/secondary architecture has consistency implications.
- Majority acknowledgement prevents many failover data-loss scenarios.
- Replication lag must be monitored and designed around.

---

# Sharding

## What it is
Sharding is horizontal partitioning: splitting a collection across multiple shards so the cluster can handle more data and throughput than one replica set.

## Why it exists
A single replica set has limits: disk, CPU, memory, write throughput, and working set size. Sharding adds capacity by distributing data and load.

## How it works internally

### Shard key
The shard key determines how documents are distributed. It must exist in sharded documents and should match high-volume query patterns.

### Mongos router
Applications connect to `mongos`, which routes queries to the relevant shard or broadcasts to many shards if it cannot target one.

### Config servers
Config servers store cluster metadata: shard mappings, chunks, and routing information.

### Chunk migration and balancing
MongoDB splits data into chunks and migrates chunks between shards to balance data distribution. Migration consumes resources and can affect latency.

### Distributed queries
Queries containing the shard key can target one shard. Queries without shard key may scatter-gather across all shards, increasing latency and load.

## Production usage

Good shard key example for multi-tenant order data:

```javascript
{ tenantId: 1, orderId: 1 }
```

If most queries include `tenantId`, this targets relevant partitions. For very large tenants, this may still create hot tenants, so hashed or compound strategies may be needed.

Bad shard key examples:

- Monotonically increasing `createdAt`: new writes hit the same shard.
- Low-cardinality `status`: only a few chunks and poor distribution.
- Field not used in queries: causes scatter-gather reads.

## Common production problems

- Hot shard due to poor shard key.
- Resharding large collections being expensive and risky.
- Scatter-gather queries saturating the cluster.
- Assuming sharding automatically fixes bad queries.
- Choosing shard key before understanding traffic distribution.

## Tradeoffs

| Benefit | Cost |
|---|---|
| More storage and write capacity | More operational complexity |
| Horizontal scale | Shard key is hard to change |
| Parallelism | Distributed queries can be slower |
| Isolation by tenant/key | Hot tenants can still overload one shard |

## Interview traps

- Sharding is not a first step. First fix schema, indexes, query patterns, and hardware sizing.
- A unique index in a sharded collection has constraints involving shard key design.
- A shard key must be evaluated for cardinality, write distribution, read targeting, and future growth.

## Key points to remember

- Sharding solves capacity, not bad modeling.
- The shard key is one of the most important design decisions.
- Avoid hot shards and scatter-gather queries.
- Resharding is possible but operationally serious.

---

# Transactions and Consistency

## What it is
Transactions group multiple operations into an all-or-nothing unit. MongoDB supports ACID semantics for single-document operations and multi-document transactions.

## Why it exists
Some business operations require multiple writes to succeed or fail together, such as creating an order and reserving inventory in the same database boundary.

## How it works internally
MongoDB provides:

- **Single-document atomicity**: updates to one document are atomic.
- **Multi-document transactions**: available across replica sets and sharded clusters, with additional coordination.
- **Snapshot-like reads in transactions**: consistent view during the transaction.

Distributed transactions require coordination, locks/resources, transaction logs, and retries. They are more expensive than single-document updates.

## Production usage

Use transactions when correctness requires them and the write set is small. Prefer document modeling that keeps atomic invariants inside one document.

Example:

- Good single-document design: order document contains order lines and status transitions.
- Transaction candidate: transfer ownership between two documents where partial update is unacceptable.
- Avoid: wrapping every repository call in a transaction “just in case.”

For cross-service workflows, use events, outbox pattern, sagas, idempotency, and compensating actions rather than distributed transactions across service databases.

## Common production problems

- Long-running transactions increasing memory and lock pressure.
- Retrying non-idempotent writes and creating duplicates.
- Expecting MongoDB transactions to make poor data modeling cheap.
- Mixing transactions with high-throughput sharded workloads without measuring cost.
- Ignoring transient transaction errors and retry requirements.

## Tradeoffs

- Transactions simplify correctness but reduce throughput and increase latency.
- Single-document atomicity is cheap; distributed transactions are expensive.
- Eventual consistency improves availability and decoupling but requires business-level reconciliation.

## Interview traps

- “MongoDB has no transactions” is outdated.
- “Use transactions everywhere” is also wrong.
- Interviewers check whether you know idempotency and retry logic are part of distributed correctness.

## Key points to remember

- Use single-document atomicity as much as possible.
- Multi-document transactions exist but should be minimized.
- Distributed systems require idempotent retries.
- Cross-service consistency usually uses events and sagas, not shared DB transactions.

---

# Redis

## What it is
Redis is an in-memory data structure store commonly used as a cache, session store, rate limiter, counter system, lightweight queue, Pub/Sub mechanism, and coordination helper.

## Why it exists
Many backend systems need sub-millisecond or low-millisecond access to frequently used data. Disk-based databases are not always appropriate for hot, temporary, or derived data.

## How it works internally
Redis keeps data primarily in memory and supports data structures:

- Strings.
- Hashes.
- Lists.
- Sets.
- Sorted sets.
- Streams.
- Bitmaps and HyperLogLog.

Redis is fast because operations are memory-based, data structures are efficient, protocol overhead is low, and the execution model avoids many locking costs.

### TTL and eviction
TTL expires keys after a configured time. Eviction policies decide what happens when memory is full, such as evicting least recently used keys or refusing writes.

### Pub/Sub basics
Redis Pub/Sub broadcasts messages to currently subscribed clients. It is not a durable event log. Redis Streams provide more durable stream-like behavior.

### Persistence modes

- **RDB snapshots**: periodic point-in-time snapshots.
- **AOF**: append-only file for better durability at write cost.
- Some deployments use both.

### Distributed locks
Redis can implement locks using `SET key value NX PX ttl`, but correctness requires unique tokens, expiration, safe release, retry policy, and awareness of network partitions. For critical financial correctness, prefer stronger coordination or database constraints.

## Production usage

Common Spring Boot patterns:

- Cache product details: `product:{id}` with TTL.
- Store sessions: `session:{token}` with TTL.
- Rate limit: `rate:user:{id}:minute` with increment and expiry.
- Prevent cache stampede: short lock, request coalescing, stale-while-revalidate.

## Common production problems

- Cache stampede after many hot keys expire simultaneously.
- Hot key overload on a single Redis node.
- Big keys causing latency spikes.
- Memory exhaustion and unexpected eviction.
- Treating cache as source of truth.
- Distributed lock bugs causing duplicate processing.
- No timeout on Redis calls causing thread pool exhaustion in Java services.

## Tradeoffs

| Benefit | Cost |
|---|---|
| Extremely low latency | Memory is expensive |
| TTL-based temporary data | Expiration can surprise business logic |
| Simple counters/rate limits | Hot keys can bottleneck |
| Cache reduces database load | Cache invalidation is hard |
| Persistence options exist | Durability is not the same as a primary database by default |

## Interview traps

- Redis is not just a cache, but cache is its most common role.
- TTL is not exact business scheduling.
- Pub/Sub is not Kafka.
- Distributed locks are easy to misuse.

## Key points to remember

- Redis is fast because it is memory-first and data-structure-oriented.
- Always design cache invalidation, TTL, fallback, and failure behavior.
- Protect Java services with timeouts, circuit breakers, and bounded connection pools.
- Use Redis for derived/temporary data unless durability is explicitly designed.

---

# Cassandra Basics

## What it is
Cassandra is a distributed wide-column database designed for high write throughput, large-scale partitioned data, multi-node availability, and tunable consistency.

## Why it exists
Some workloads produce huge volumes of append-heavy data where one primary node would be a bottleneck: metrics, clickstreams, IoT events, audit logs, and time-series-like data.

## How it works internally

- Data is distributed by **partition key** across nodes.
- Writes are optimized with append-oriented structures and later compaction.
- Replication stores data on multiple nodes.
- Consistency is tunable per operation, such as `ONE`, `QUORUM`, or `ALL`.
- Data modeling is query-first; tables are often designed for one query pattern.

Cassandra is commonly described as AP-oriented because it can continue accepting reads/writes during some partitions depending on consistency level, then reconcile later.

## Production usage

A metrics table might use:

```text
partition key: device_id + day_bucket
clustering key: event_time
```

This avoids one infinite partition per device and supports efficient range reads for a device/day.

## Common production problems

- Bad partition keys creating hot or huge partitions.
- Trying to query by fields not designed into the table.
- Tombstone buildup from deletes/TTL.
- Misconfigured consistency level causing stale reads or high latency.
- Underestimating compaction and disk needs.

## Tradeoffs

- Excellent write scalability but limited ad-hoc querying.
- High availability but eventual consistency concerns.
- Tunable consistency but more application responsibility.
- Great for time-series patterns if partitioning is correct.

## Interview traps

- Cassandra is not a relational database with different syntax.
- You do not design Cassandra by normalizing entities; you design by query.
- AP-oriented does not mean inconsistent chaos; consistency level and reconciliation matter.

## Key points to remember

- Cassandra is built for distributed write-heavy workloads.
- Partition key design is everything.
- Tunable consistency lets you trade latency, availability, and freshness.
- Avoid unbounded partitions and uncontrolled tombstones.

---

# Storage of Binary Data

## What it is
Binary storage means handling files such as images, videos, PDFs, and large attachments in or near the database.

## Why it exists
Applications need file metadata, access control, lifecycle management, and sometimes file streaming. The question is whether the database should store the bytes or only metadata.

## How it works internally

### GridFS
GridFS stores large files in MongoDB by splitting them into chunks and storing metadata separately. It is useful when files must live inside MongoDB operational boundaries.

### Object storage pattern
Most production systems store actual bytes in object storage such as S3-compatible storage and keep metadata in MongoDB:

```json
{
  "_id": "img_123",
  "ownerId": "user_42",
  "bucket": "product-images",
  "objectKey": "products/p100/main.jpg",
  "contentType": "image/jpeg",
  "size": 384211,
  "checksum": "...",
  "createdAt": "2026-01-10T12:00:00Z"
}
```

## Production usage

For a Spring Boot product service:

1. Upload image to object storage.
2. Store metadata and object key in MongoDB product document or separate image collection.
3. Serve through CDN or signed URLs.
4. Use async cleanup for orphaned files.

## Common production problems

- Storing large images directly in documents and bloating working set.
- Backups becoming huge and slow.
- Serving files through application servers unnecessarily.
- Orphaned objects after failed DB writes.
- Inconsistent metadata and object storage state.

## Tradeoffs

- GridFS keeps file chunks in MongoDB but increases database storage/load.
- Object storage is cheaper and scalable but introduces cross-system consistency handling.
- Metadata in DB plus bytes in object storage is usually the production default.

## Interview traps

- Do not say “store images as Base64 in MongoDB” as the default answer.
- Mention document size, backup cost, CDN, object storage, and metadata.
- Explain failure handling between DB transaction and object upload.

## Key points to remember

- Store large binaries in object storage by default.
- Store metadata, ownership, and references in MongoDB.
- Use GridFS only when it fits operational requirements.
- Plan cleanup and consistency between storage and metadata.

---

# Scaling and Performance

## What it is
Scaling and performance are about keeping latency, throughput, cost, and reliability acceptable as data and traffic grow.

## Why it exists
Most database incidents are not caused by exotic algorithms. They are caused by missing indexes, hot keys, unbounded growth, memory pressure, bad pagination, and unclear ownership.

## How it works internally

### Read scaling
- Add indexes.
- Use read replicas carefully.
- Cache hot derived data in Redis.
- Use projections to avoid loading huge documents.
- Build read models from Kafka events.

### Write scaling
- Batch writes where safe.
- Avoid hot documents/partitions.
- Use sharding when one replica set is not enough.
- Keep indexes minimal on write-heavy collections.
- Use async processing for non-critical updates.

### Query optimization
Use `explain()`, slow query logs, index stats, and real data volume. Optimize the endpoint's actual filter, sort, projection, and pagination.

### Connection pooling
Java services should use bounded MongoDB/Redis connection pools, timeouts, and backpressure. Unlimited concurrency creates cascading failure.

### Pagination
Offset pagination with large `skip` becomes expensive. Cursor-based pagination using stable sort fields such as `createdAt` and `_id` is preferred.

### Memory pressure and OOM
Databases need working set memory. Redis must fit memory plus overhead. MongoDB needs hot indexes/documents in cache. Java services need controlled result sizes to avoid heap pressure.

### Cache invalidation
Strategies:

- TTL only.
- Write-through.
- Cache-aside with invalidation on write.
- Event-driven invalidation via Kafka.
- Stale-while-revalidate for hot keys.

## Production usage

A high-traffic product endpoint may use:

- MongoDB as source/read model.
- Redis cache with short TTL.
- Kafka event to invalidate cache after product update.
- Cursor pagination for search/list pages.
- Dashboards watching p95 latency, slow queries, cache hit rate, and DB CPU.

## Common production problems

- Loading entire collections into Java memory.
- Returning huge MongoDB documents when only three fields are needed.
- Cache stampede under peak traffic.
- Redis OOM due to no max memory policy.
- MongoDB working set larger than RAM causing disk thrashing.
- Batch jobs competing with online traffic.

## Tradeoffs

- Caching reduces read load but creates staleness and invalidation complexity.
- Batching improves throughput but increases latency and retry complexity.
- More indexes speed reads but slow writes.
- Sharding adds capacity but complicates queries and operations.

## Interview traps

- Performance is not only “add cache.” First understand query plan and data model.
- Cursor pagination must use stable ordering.
- Read replicas can return stale data.
- Cache failures must not bring down the whole service.

## Key points to remember

- Measure before optimizing.
- Use indexes, projections, cursor pagination, and bounded results.
- Avoid hot documents, hot keys, and unbounded arrays.
- Caches need failure and invalidation strategy.

---

# NoSQL in Microservices

## What it is
NoSQL in microservices means using specialized storage per service boundary and accepting that cross-service consistency is usually asynchronous.

## Why it exists
Each service owns its data and chooses a database fitting its workload. Sharing one database schema across services creates coupling and deployment risk.

## How it works internally

### Database per service
Each microservice owns its database. Other services access data through APIs or events, not direct table/collection writes.

### Polyglot persistence
Different services may use different storage:

- Catalog service: MongoDB.
- Session service: Redis.
- Analytics ingestion: Cassandra.
- Payments: relational DB.
- Event backbone: Kafka.

### Kafka interaction patterns
- Publish domain events after successful state change.
- Use outbox pattern to avoid losing events between DB commit and Kafka publish.
- Build read models in MongoDB from events.
- Use idempotent consumers because messages may be duplicated.

### Eventual consistency between services
If order service depends on catalog price snapshots, it may not query catalog synchronously for every checkout. It may consume price events and store snapshots. The system must define acceptable staleness and reconciliation rules.

## Production usage

Example flow:

1. Admin updates product in catalog MongoDB.
2. Catalog service writes product and outbox event.
3. Outbox publisher sends `ProductUpdated` to Kafka.
4. Search service updates Elasticsearch.
5. Redis cache key is invalidated.
6. Order service updates product snapshot if needed.

## Common production problems

- Services directly reading another service's database.
- No idempotency in Kafka consumers.
- Duplicate events causing duplicate side effects.
- Assuming all services see changes instantly.
- No reconciliation job for missed or failed updates.

## Tradeoffs

- Database per service improves autonomy but adds eventual consistency.
- Events decouple services but require schema evolution and retry handling.
- Duplicated read models improve performance but need synchronization.

## Interview traps

- “Microservices solve consistency” is wrong. They make consistency explicit and harder.
- Kafka does not remove the need for idempotency.
- Polyglot persistence should be justified by workload, not fashion.

## Key points to remember

- Service data ownership matters more than database brand.
- Cross-service consistency is often eventual.
- Use outbox, idempotent consumers, and reconciliation.
- Pick MongoDB/Redis/Cassandra based on service access patterns.

---

# Monitoring and Production Operations

## What it is
Monitoring and operations are the practices that keep NoSQL systems reliable after deployment: observing query performance, replication, disk, memory, backups, and failure behavior.

## Why it exists
A correct data model can still fail in production if nobody watches slow queries, replication lag, disk growth, memory, or backup restore ability.

## How it works internally

### What to monitor

- **Slow queries**: endpoint latency, query duration, full scans.
- **Query profiling**: identify inefficient filters, sorts, and indexes.
- **Index monitoring**: unused indexes, missing indexes, index size.
- **Replication lag**: stale reads and failover risk.
- **Disk usage**: data, journal, oplog, compaction, snapshots.
- **Memory usage**: working set, WiredTiger cache, Redis max memory.
- **Connections**: pool saturation, connection spikes, timeouts.
- **CPU and I/O**: sustained saturation, disk latency.
- **Backup status**: successful backups and tested restores.
- **Alerting**: actionable thresholds, not noise.

### Backup and restore strategies

- Scheduled backups with retention.
- Point-in-time recovery where required.
- Restore drills in staging.
- Separate backup credentials and storage.
- Document RPO and RTO.

## Production usage

A practical MongoDB alert set:

- p95 query latency above threshold.
- Replication lag > acceptable seconds.
- Disk > 80% and growth rate high.
- Connection pool near exhaustion.
- Oplog window too small.
- Failed backup or untested restore.
- Full collection scan on hot collection.

## Common production problems

- Backups exist but restores were never tested.
- Alerts fire too late or too often.
- Slow query logs ignored until incident.
- Indexes grow beyond memory.
- Replication lag hidden because users mostly read from primary until failover.

## Tradeoffs

- More observability costs money but prevents blind outages.
- Aggressive profiling can add overhead.
- Longer backup retention costs storage but improves recovery options.
- Strict alerts reduce risk but can cause alert fatigue if poorly tuned.

## Interview traps

- Replication is not backup.
- Monitoring is not only CPU and memory.
- A production engineer talks about restore testing, not just backup creation.

## Key points to remember

- Watch slow queries, indexes, lag, disk, memory, connections, and backups.
- Test restores regularly.
- Alert on user impact and leading indicators.
- Operations are part of database design.

---

# Common Production Problems

## What it is
These are recurring NoSQL failure patterns seen in real backend systems.

## Why it exists
NoSQL gives flexibility and scale, but that flexibility lets teams create data growth, consistency, and operational problems that SQL constraints might have prevented earlier.

## How it works internally

### Frequent failures

- **Unbounded document growth**: arrays grow forever.
- **Missing indexes**: full collection scans under traffic.
- **Bad shard keys**: hot shards and scatter-gather queries.
- **Replication lag**: stale reads and failover surprises.
- **Cache stampede**: many requests rebuild same expired cache key.
- **TTL surprises**: data disappears earlier/later than business expects.
- **Transaction abuse**: high latency and lock/resource pressure.
- **Memory exhaustion**: Redis eviction or MongoDB disk thrashing.
- **Distributed consistency bugs**: duplicate events, out-of-order updates, missing idempotency.

## Production usage

Incident-style example:

1. Marketing campaign increases traffic to product page.
2. Redis hot key expires.
3. Thousands of requests hit MongoDB simultaneously.
4. Query uses sort without correct compound index.
5. MongoDB CPU spikes; Java thread pools fill; API latency cascades.
6. Fix requires cache stampede protection, index correction, timeout tuning, and alerting.

## Common production problems

The biggest mistake is treating each symptom separately. Cache failure, missing indexes, unbounded concurrency, and no backpressure often combine into one outage.

## Tradeoffs

- Fixing incidents often means trading freshness for stability, such as serving stale cache briefly.
- Reducing write pressure may require fewer indexes or async read models.
- Preventing hot partitions may require more complex keys.

## Interview traps

- Interviewers like failure scenarios because they reveal practical depth.
- Do not answer only with definitions. Explain detection, mitigation, and prevention.
- Mention timeouts, retries, idempotency, and observability.

## Key points to remember

- Most NoSQL incidents are predictable design/operations failures.
- Protect databases from unbounded application concurrency.
- Design for growth limits, not just happy-path documents.
- Every distributed retry must be safe or idempotent.

---

# CAP Theorem

## What it is
CAP says that during a network partition, a distributed system cannot provide all three: strong consistency, availability, and partition tolerance.

## Why it exists
Once data lives on multiple nodes, network failures create moments when nodes disagree or cannot coordinate. The database must decide whether to reject operations or allow potentially stale/conflicting operations.

## How it works internally

### CAP properties

- **Consistency**: every read sees the latest successful write, or a consistent view.
- **Availability**: every request to a non-failed node receives a response.
- **Partition tolerance**: system continues operating despite network communication failure between nodes.

### CP vs AP

- **CP**: preserve consistency during partitions, possibly rejecting requests.
- **AP**: preserve availability during partitions, possibly allowing stale/conflicting data.

### Why CA is unrealistic
In a real distributed system, partitions can happen. If partition tolerance is required, CA is not a meaningful choice under failure. A single-node database can be CA-like only because it avoids distribution.

### MongoDB CAP behavior
MongoDB replica sets with primary elections and majority write concern lean CP for writes: if a majority cannot be reached, the system avoids unsafe primary writes. Secondary reads can introduce stale-read behavior depending on read preference/read concern.

### Cassandra CAP behavior
Cassandra is commonly AP-oriented with tunable consistency. It can accept operations on reachable replicas depending on consistency level and reconcile later.

### Redis CAP tradeoffs
A single Redis instance is not a distributed CAP system. Redis with replication or clustering has tradeoffs around failover, asynchronous replication, possible data loss, and availability during partitions depending on configuration.

## Production usage

Use CAP to reason about failure mode:

- Payment authorization: prefer consistency; reject if uncertain.
- Product recommendations: prefer availability; stale data is acceptable.
- User profile update: depends on business requirement; maybe primary read after write.

## Common production problems

- Memorizing database labels without explaining configuration.
- Ignoring read preference and write concern.
- Treating stale reads as impossible.
- Designing workflows that require global strong consistency across services but deploying AP-style components.

## Tradeoffs

- CP systems protect correctness but can be unavailable during partitions.
- AP systems stay responsive but require conflict handling and convergence.
- Many databases are configurable, so real behavior depends on settings and client choices.

## Interview traps

- CAP only forces the C vs A choice during a partition.
- Availability in CAP has a strict meaning, not “high uptime.”
- Do not claim a distributed system can guarantee CA under partitions.

## Key points to remember

- In distributed systems, partitions are unavoidable.
- During partitions, choose consistency or availability.
- MongoDB behavior depends on replica set, read/write concerns, and read preference.
- Cassandra exposes consistency/availability tradeoffs directly.

---

# PACELC Theorem

## What it is
PACELC extends CAP: **if Partition happens, choose Availability or Consistency; Else, when the system is healthy, choose Latency or Consistency.**

## Why it exists
CAP focuses on partition failures, but most production time is normal operation. Even without failures, databases trade stronger consistency for higher latency because coordination takes time.

## How it works internally

- Stronger consistency often requires waiting for replicas, quorums, or coordination.
- Lower latency often means reading local/nearby replicas or accepting stale data.
- Geo-distributed systems make this more obvious because cross-region coordination is slow.

## Production usage

Examples:

- MongoDB `writeConcern: majority` improves durability/consistency but adds latency.
- Reading from nearest secondary reduces latency but may return stale data.
- Cassandra `QUORUM` is more consistent than `ONE` but slower and less available.
- Redis cache is fast but may serve stale derived data.

## Common production problems

- Assuming no partition means no consistency tradeoff.
- Using low-latency reads where read-your-writes is required.
- Applying global consistency to data that could tolerate staleness.

## Tradeoffs

- Consistency costs latency.
- Low latency often costs freshness.
- Global coordination costs more than local coordination.
- The right choice depends on business risk.

## Interview traps

- PACELC is not a replacement for CAP; it extends the conversation.
- The important interview point is not acronym memorization but explaining latency vs consistency in normal operation.

## Key points to remember

- CAP: what happens during partition?
- PACELC: what do you trade when there is no partition?
- Real systems tune consistency differently per endpoint.
- Business criticality decides the acceptable tradeoff.

---

# Interview Q&A

## 143. What are NoSQL databases and how do they differ from relational databases?

**Production-oriented answer:**  
NoSQL databases are databases optimized for non-relational data models such as documents, key-value pairs, wide-column rows, or graphs. They often prioritize flexible schema, horizontal scaling, high throughput, and specialized access patterns. Relational databases prioritize structured tables, SQL, joins, constraints, and strong transactional consistency.

In production I would not say NoSQL is simply “faster.” MongoDB is faster than SQL only when the data is modeled around document reads and indexes. Redis is faster because it is in-memory and key-based. Cassandra scales writes because it distributes partitions across nodes. SQL may still be the better choice for payments, accounting, complex joins, and strict relational constraints.

**Real-world usage:**

- SQL for financial records and transactional order state.
- MongoDB for product catalog documents.
- Redis for cache/session/rate limit.
- Cassandra for large append-only event/time-series data.

**Interview traps:**

- Do not say NoSQL means no schema. It means flexible schema; production still requires validation.
- Do not say NoSQL replaces SQL. It complements SQL.
- Do not ignore consistency and operations.

**Common mistakes:**

- Choosing MongoDB because migrations are annoying.
- Modeling MongoDB like normalized SQL.
- Using Redis as the only durable database without persistence planning.

**Tradeoffs:**

- Flexibility and scale versus weaker constraints and more application responsibility.
- Denormalized fast reads versus duplicated data and consistency work.

## 144. What are the main types of NoSQL databases? (Document, key-value, graph, columnar)

**Production-oriented answer:**  
The main NoSQL types are document, key-value, wide-column, and graph databases.

- **Document**: MongoDB stores JSON/BSON-like documents. Good for aggregate objects like products, profiles, content.
- **Key-value**: Redis stores values by key. Good for cache, session, counters, rate limits.
- **Wide-column**: Cassandra stores partitioned rows with flexible columns. Good for massive write-heavy workloads.
- **Graph**: Neo4j stores nodes and relationships. Good for traversals like fraud networks or recommendations.

**Real-world usage:**  
A Spring Boot e-commerce platform might use MongoDB for catalog, Redis for cache, PostgreSQL for payments, Kafka for events, and Cassandra for clickstream analytics.

**Interview traps:**

- Cassandra is wide-column, not the same as analytical columnar storage.
- Redis has data structures, but access is still primarily key-based.
- Graph databases are not needed just because entities have relationships; they are useful when traversal is the core query.

**Common mistakes:**

- Using a key-value store when filtering by many fields is required.
- Using MongoDB for graph traversal.
- Using Cassandra for ad-hoc queries.

**Tradeoffs:**

- Specialized databases are powerful for the right access pattern and painful for the wrong one.

## 145. In what cases is it better to use NoSQL instead of SQL?

**Production-oriented answer:**  
Use NoSQL when the workload benefits from flexible schema, document-shaped aggregates, massive horizontal write scaling, low-latency key lookup, or specialized data relationships. The decision should be based on access pattern and consistency requirements, not fashion.

Good cases:

- Product catalog with variable attributes by category.
- User preferences or profile documents that change shape over time.
- Session/cache/rate-limit data with TTL in Redis.
- High-volume event/time-series ingestion in Cassandra.
- Read models updated asynchronously from Kafka.

Avoid NoSQL as default when:

- Strong relational constraints are central.
- Complex joins are frequent.
- Multi-row transactions are the normal case.
- Reporting requires flexible ad-hoc SQL.

**Interview traps:**

- “When data is big” is incomplete. SQL databases can handle large data too.
- “When we do not need schema” sounds immature.

**Common mistakes:**

- Moving to NoSQL to avoid modeling discipline.
- Ignoring query patterns and indexes.
- Underestimating eventual consistency.

**Tradeoffs:**

- NoSQL can simplify reads and scaling, but often shifts correctness and validation into application code.

## 146. What popular NoSQL databases do you know? (MongoDB, Cassandra, Redis, etc.)

**Production-oriented answer:**  
Popular NoSQL databases include MongoDB, Redis, Cassandra, DynamoDB, Couchbase, Neo4j, Elasticsearch/OpenSearch, and HBase. They solve different problems.

- **MongoDB**: document database for flexible JSON-like aggregates.
- **Redis**: in-memory key-value/data structure store.
- **Cassandra**: distributed wide-column store for write-heavy workloads.
- **DynamoDB**: managed key-value/document store with partitioned scaling.
- **Neo4j**: graph traversal.
- **Elasticsearch/OpenSearch**: search and analytics-oriented indexing, not a general transactional database.

**Real-world usage:**  
For Java microservices, Redis and MongoDB are common because Spring has strong integrations. Cassandra appears when write volume and availability requirements justify its modeling constraints.

**Interview traps:**

- Do not list names only. Explain what each is good at.
- Elasticsearch is often grouped with NoSQL, but it is usually not the source of truth for transactional data.

**Common mistakes:**

- Choosing based on popularity instead of workload.

## 147. What is a collection in MongoDB and how does it differ from a table in SQL?

**Production-oriented answer:**  
A MongoDB collection is a group of BSON documents. It is similar to a SQL table in that it groups related records, but it does not require every document to have identical columns. A SQL table has a fixed schema, rows, columns, constraints, and normalized relationships. A MongoDB collection has flexible documents, nested fields, arrays, and indexes.

**Real-world usage:**  
A `products` collection may store products with different attribute sets for phones, books, and clothes. In SQL, that often requires multiple related tables or JSON columns.

**Interview traps:**

- Flexible schema does not mean no schema.
- Collections still need indexes and validation.

**Common mistakes:**

- Putting unrelated document types into one collection just because MongoDB allows it.
- Not versioning schema changes.

**Tradeoffs:**

- Collections allow flexible evolution but require application-level discipline.

## 148. What is a document in MongoDB?

**Production-oriented answer:**  
A document is MongoDB's basic record unit. It is stored as BSON and can contain nested objects, arrays, dates, numbers, strings, ObjectIds, and other types. Every document has an `_id` field that uniquely identifies it in the collection.

**Real-world usage:**  
A product document may contain title, brand, category, attributes, price snapshot, image metadata, and audit fields. This lets a product page load with one indexed query instead of many joins.

**Interview traps:**

- A document is not just a JSON string. BSON has typed binary representation.
- Large or unbounded documents are dangerous.

**Common mistakes:**

- Embedding unlimited arrays.
- Storing huge binary content directly in the document.
- Updating one hot document from many concurrent requests.

**Tradeoffs:**

- Rich document structure improves locality but can create growth and update problems.

## 149. How are relationships between documents defined in MongoDB? Is there JOIN?

**Production-oriented answer:**  
MongoDB models relationships mainly through embedding or references. Embedding stores related data inside the parent document. Referencing stores another document's ID and resolves it with a second query, application-side join, or aggregation `$lookup`.

MongoDB has `$lookup`, which is join-like inside aggregation, but MongoDB is not designed around join-heavy OLTP queries like a relational database. In production, if data is always read together and bounded, embedding is usually preferred. If data is large, shared, or changes independently, referencing is better.

**Real-world usage:**

- Embed order items inside an order.
- Reference user ID from orders.
- Duplicate product name/price snapshot in an order to preserve historical checkout state.

**Interview traps:**

- Saying “MongoDB has no joins” is incomplete.
- Saying “always embed” is wrong.

**Common mistakes:**

- `$lookup` on hot endpoints without measuring.
- Duplicating data without event-based update strategy.
- Referencing everything like SQL foreign keys.

**Tradeoffs:**

- Embedding improves read locality and atomicity.
- Referencing reduces duplication and document growth.

## 150. How is an index implemented in MongoDB and why is it needed?

**Production-oriented answer:**  
MongoDB indexes are B-tree-like structures that map indexed field values to document locations. They let the database find and sort documents without scanning the whole collection. Indexes are required for predictable production latency.

For example, if an endpoint queries orders by `tenantId`, `status`, and `createdAt`, a compound index like `{ tenantId: 1, status: 1, createdAt: -1 }` may avoid scanning millions of orders.

**Real-world usage:**  
Before deploying a Spring Boot list endpoint, check `explain()` to confirm the query uses the intended index and does not scan too many documents.

**Interview traps:**

- Indexes are not free. They slow writes and consume memory/disk.
- Compound index order matters.
- Low-cardinality indexes may not help much.

**Common mistakes:**

- Adding indexes after production incident instead of during endpoint design.
- Indexing every field.
- Sorting without matching index support.

**Tradeoffs:**

- Faster reads and sorting versus slower writes and more storage.

## 151. What are sharding and replication in NoSQL?

**Production-oriented answer:**  
Replication means keeping copies of the same data on multiple nodes for high availability, durability, and sometimes read scaling. Sharding means splitting data across multiple nodes so each node owns only part of the dataset, increasing storage and throughput capacity.

**Real-world usage:**  
In MongoDB, a replica set protects against node failure. A sharded cluster distributes a large collection across shards using a shard key.

**Interview traps:**

- Replication and sharding solve different problems.
- Replication does not reduce total data size per replica set.
- Sharding does not automatically improve every query.

**Common mistakes:**

- Sharding before fixing indexes.
- Thinking replicas are backups.
- Choosing a shard key with low cardinality or monotonically increasing values.

**Tradeoffs:**

- Replication improves availability but adds lag and failover behavior.
- Sharding improves capacity but adds routing, balancing, and distributed query complexity.

## 152. How does replication differ from sharding?

**Production-oriented answer:**  
Replication copies the same data to multiple nodes. Sharding partitions different data across nodes. They are orthogonal and often used together: each shard can be a replica set.

**Real-world usage:**  
A MongoDB cluster may have three shards, and each shard may have three replicas. That gives both horizontal capacity and failover.

**Interview traps:**

- Do not say replicas store different parts of data. That is sharding.
- Do not say sharding is for backup. It is for capacity and distribution.

**Common mistakes:**

- Reading from secondary replicas when strong freshness is required.
- Designing shard key without query targeting.

**Tradeoffs:**

- Replication cost is extra storage for copies.
- Sharding cost is operational and query complexity.

## 153. What is eventual consistency

**Production-oriented answer:**  
Eventual consistency means updates do not have to be visible everywhere immediately, but replicas or services converge to the same state if no new changes occur and replication succeeds.

**Real-world usage:**  
After a product update, MongoDB primary may have the new value, Redis cache may still contain the old value, search index may update after a Kafka event, and another service may receive the update seconds later. That is acceptable if the business allows temporary staleness.

**Interview traps:**

- Eventual consistency does not mean random or unreliable.
- It requires defined convergence, retries, idempotency, and monitoring.

**Common mistakes:**

- Using eventual consistency for money movement without reconciliation.
- No idempotency in consumers.
- No business rule for conflict resolution.

**Tradeoffs:**

- Better availability, decoupling, and latency versus temporary stale reads and more application logic.

## 154. How to store binary data (e.g., images) in MongoDB?

**Production-oriented answer:**  
Usually store large binary files in object storage and store metadata plus object references in MongoDB. MongoDB can store binary data, and GridFS can split large files into chunks, but storing images directly in documents often bloats the database and hurts backups, memory, and performance.

**Real-world usage:**  
Upload image to S3-compatible storage, store `objectKey`, `contentType`, `size`, checksum, owner, and timestamps in MongoDB, then serve through CDN or signed URL.

**Interview traps:**

- Do not recommend Base64 in documents as default.
- Mention document size, backup size, CDN, and metadata consistency.

**Common mistakes:**

- Keeping large images in product documents.
- No cleanup for object storage if DB write fails.
- No checksum or content validation.

**Tradeoffs:**

- Object storage scales cheaply but requires cross-system consistency handling.
- GridFS centralizes storage in MongoDB but increases DB load.

## 155. What are TTL indexes in MongoDB?

**Production-oriented answer:**  
TTL indexes automatically remove documents after a specified time based on a date field. They are useful for temporary data such as sessions, verification tokens, temporary locks, audit staging data, or short-lived events.

Example:

```javascript
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 })
```

**Real-world usage:**  
A Spring Boot authentication service can store password reset tokens with expiration. MongoDB cleans them up eventually.

**Interview traps:**

- TTL deletion is not exact to the millisecond; it is background cleanup.
- TTL should not be used as a precise scheduler.

**Common mistakes:**

- Accidentally applying TTL to business-critical data.
- Expecting immediate deletion.
- Forgetting TTL index in lower environments and finding stale behavior differences.

**Tradeoffs:**

- Automatic cleanup versus less precise deletion timing.

## 156. Can transactions be used in MongoDB? If so, how?

**Production-oriented answer:**  
Yes. MongoDB supports single-document atomic operations and multi-document ACID transactions. In production, prefer designing documents so critical invariants fit inside one document. Use multi-document transactions only when needed and keep them short.

In Spring, this can be managed with MongoDB transaction support and sessions when using a properly configured replica set or sharded cluster.

**Real-world usage:**  
A multi-document transaction may be used when updating two related documents must be atomic. But cross-service operations should usually use outbox, Kafka, sagas, and idempotency rather than distributed transactions across databases.

**Interview traps:**

- “MongoDB has no transactions” is outdated.
- “Use transactions like SQL everywhere” is bad production advice.

**Common mistakes:**

- Long-running transactions.
- Retrying non-idempotent operations.
- Using transactions to hide poor document modeling.

**Tradeoffs:**

- Correctness and simpler code in some cases versus latency, resource usage, and reduced throughput.

## 157. How is fault tolerance ensured in NoSQL databases?

**Production-oriented answer:**  
Fault tolerance is achieved through replication, failover, partitioning, quorum or majority acknowledgements, backups, monitoring, client retries, idempotency, and deployment across failure domains.

In MongoDB, replica sets elect a new primary if the old primary fails. In Cassandra, data is replicated across nodes and requests can use tunable consistency. In Redis, Sentinel or Cluster can provide failover, but asynchronous replication can still lose recent writes.

**Real-world usage:**  
A Java service must set timeouts, retry transient errors safely, and use idempotency keys for operations that may be retried after failover.

**Interview traps:**

- Replication is not backup.
- Fault tolerance is not only a database feature; clients must handle failures too.

**Common mistakes:**

- No restore testing.
- Infinite retries causing overload.
- Weak write concern for critical data.

**Tradeoffs:**

- More replicas and stronger acknowledgements improve safety but increase cost and latency.

## 158. What is the CAP theorem? What is the extension of the CAP theorem?

**Production-oriented answer:**  
CAP says that during a network partition, a distributed system must choose between consistency and availability while tolerating the partition. The common extension is PACELC: if there is a Partition, choose Availability or Consistency; Else, choose Latency or Consistency.

**Real-world usage:**  
MongoDB with majority writes may reject writes without majority to protect consistency. Cassandra may accept writes on available replicas depending on consistency level and reconcile later. Even without failure, reading from a nearby replica lowers latency but can reduce freshness.

**Interview traps:**

- CAP is about behavior during partitions, not normal operation only.
- PACELC highlights that latency versus consistency exists even without partitions.

**Common mistakes:**

- Saying a distributed system is CA under partitions.
- Memorizing CP/AP labels without explaining settings.

**Tradeoffs:**

- Strong consistency can cost availability during partition and latency during normal operation.

## 159. Why is it impossible to ensure all three properties simultaneously?

**Production-oriented answer:**  
During a network partition, nodes cannot communicate reliably. If a client writes to one side and another client reads or writes on the other side, the system has two choices: reject some requests to preserve consistency, or respond to all reachable clients and risk stale/conflicting data. It cannot guarantee both latest-value consistency and availability to all non-failed nodes while communication is broken.

**Real-world usage:**  
If a MongoDB primary cannot reach majority, it should not keep accepting writes as primary because another majority side might elect a new primary. Rejecting writes hurts availability but prevents split brain.

**Interview traps:**

- Do not describe partition tolerance as optional in real distributed systems.
- Availability in CAP is not the same as “five nines uptime.”

**Common mistakes:**

- Assuming retries solve partitions. Retries help after recovery but cannot restore communication during partition.

**Tradeoffs:**

- Business critical systems often choose consistency; less critical read models may choose availability/staleness.

## 160. Give examples of CP and AP databases.

**Production-oriented answer:**  
MongoDB replica sets configured with primary writes and majority write concern are commonly described as CP-leaning because they preserve consistency by requiring majority and may become unavailable for writes during partitions. Cassandra is commonly AP-leaning because it is designed for availability and partition tolerance with tunable consistency, allowing operations at lower consistency levels and reconciling later.

Other examples:

- CP-style: ZooKeeper, etcd, Consul for coordination.
- AP-style: Cassandra, Dynamo-style systems with eventual consistency options.

**Real-world usage:**  
Use CP systems for coordination, locks, leader election, and critical metadata. Use AP systems for workloads that tolerate stale reads and need high availability across failures.

**Interview traps:**

- Most databases are configurable. Explain the mode/settings.
- Redis standalone is not a good CAP example; Redis Cluster/Sentinel introduces specific tradeoffs.

**Common mistakes:**

- Saying MongoDB is always CP without discussing read preference and write concern.
- Saying Cassandra is always inconsistent; consistency level matters.

**Tradeoffs:**

- CP: safer correctness, possible unavailability.
- AP: higher availability, conflict/staleness handling.

## 161. How does the CAP theorem manifest in MongoDB?

**Production-oriented answer:**  
MongoDB replica sets use a primary-secondary model with elections. With majority write concern, writes are considered durable after majority acknowledgement. If a network partition prevents a node from reaching majority, MongoDB avoids unsafe primary behavior and may stop accepting writes until a majority primary exists. This is CP-leaning behavior for writes.

However, MongoDB behavior depends on configuration. If the application reads from secondaries, it may see stale data. If write concern is weak, acknowledged writes may be less safe during failover. So the practical answer is: MongoDB can provide strong consistency patterns when using primary reads and majority concerns, but client settings matter.

**Real-world usage:**  
For checkout/order flows, use primary reads and majority writes. For dashboards, secondary reads may be fine if staleness is acceptable.

**Interview traps:**

- Avoid a one-word label. Discuss read preference, read concern, write concern, and elections.
- Mention split brain prevention through majority election.

**Common mistakes:**

- Reading from secondary immediately after writing and expecting read-your-writes.
- Using default settings blindly for critical data.

**Tradeoffs:**

- Stronger consistency and durability increase latency and can reduce availability during partitions.

## 162. Advanced: How do you design a compound index in MongoDB for a production API?

**Production-oriented answer:**  
Start from the exact query shape: equality filters, range filters, sort fields, and projection. Usually put high-selectivity equality fields first, then range/sort fields. Validate with `explain()` on production-like data.

Example query:

```javascript
{ tenantId: "t1", status: "PAID", createdAt: { $lt: cursor } }
```

Sort:

```javascript
{ createdAt: -1, _id: -1 }
```

Candidate index:

```javascript
{ tenantId: 1, status: 1, createdAt: -1, _id: -1 }
```

**Real-world usage:**  
This supports a Spring Boot endpoint with cursor pagination and avoids expensive skip.

**Interview traps:**

- Field order matters.
- Sort must align with index order.
- Indexes with low selectivity may not help.

**Common mistakes:**

- Adding separate single-field indexes and expecting MongoDB to always combine them efficiently.
- Not testing with realistic data size.

**Tradeoffs:**

- Great read performance versus write overhead and index storage.

## 163. Advanced: What makes a good MongoDB shard key?

**Production-oriented answer:**  
A good shard key has high cardinality, distributes writes evenly, supports common query targeting, avoids monotonic hotspots, and is stable. It should match how the application reads and writes data.

**Real-world usage:**  
For multi-tenant orders, `{ tenantId, orderId }` may target tenant queries, but very large tenants can become hot. A hashed component or bucketing may be needed.

**Interview traps:**

- High cardinality alone is not enough if all queries scatter across shards.
- Monotonic `createdAt` can create hot shards.

**Common mistakes:**

- Choosing shard key after data is huge.
- Using low-cardinality fields like `status`.
- Ignoring future top tenants.

**Tradeoffs:**

- Query targeting versus write distribution.
- Tenant isolation versus hot tenant risk.

## 164. Advanced: How do you prevent Redis cache stampede?

**Production-oriented answer:**  
Cache stampede happens when many requests miss the same key and all rebuild it simultaneously. Prevent it with TTL jitter, request coalescing, locks with short expiry, stale-while-revalidate, prewarming, and backpressure.

**Real-world usage:**  
For a hot product page, serve stale data for a short window while one request refreshes the cache from MongoDB. Add random TTL jitter so many keys do not expire at once.

**Interview traps:**

- “Just increase TTL” may create stale data and does not solve synchronized expiry fully.
- Redis locks need safe release and timeouts.

**Common mistakes:**

- No fallback when Redis is down.
- Infinite wait on lock.
- Letting all Java threads block on cache rebuild.

**Tradeoffs:**

- Stale-while-revalidate improves stability but accepts temporary staleness.

## 165. Advanced: How do you handle eventual consistency between MongoDB, Redis, and Kafka?

**Production-oriented answer:**  
Define the source of truth, acceptable staleness, event flow, retry behavior, and reconciliation. Use MongoDB for durable state, Kafka for change propagation, Redis for derived cache. Use outbox pattern so database updates and event publishing do not get lost between systems. Consumers must be idempotent.

**Real-world usage:**  
When product price changes, update MongoDB and write an outbox event. Publisher sends `ProductPriceChanged` to Kafka. Consumers update Redis/search/read models. If Redis invalidation fails, TTL and retry/reconciliation repair it.

**Interview traps:**

- Do not promise instant consistency across systems.
- Kafka exactly-once does not remove the need for idempotent side effects.

**Common mistakes:**

- Publishing Kafka event before DB commit.
- No replay/rebuild strategy for read models.
- No version field to handle out-of-order updates.

**Tradeoffs:**

- Decoupling and scalability versus temporary inconsistency and operational complexity.

## 166. Advanced: What distributed systems failures should a Java backend service expect from NoSQL databases?

**Production-oriented answer:**  
Expect timeouts, partial failures, retries, duplicate operations, stale reads, failover interruptions, connection pool exhaustion, network partitions, replication lag, and overloaded nodes. The service must use bounded timeouts, circuit breakers, retry budgets, idempotency keys, and clear fallback behavior.

**Real-world usage:**  
During MongoDB primary election, writes may fail briefly. The Java service should retry safe operations and return controlled errors for unsafe ones. During Redis outage, non-critical cache reads should fall back to MongoDB with rate limiting.

**Interview traps:**

- Do not assume the driver hides all failures.
- Retrying everything can make outages worse.

**Common mistakes:**

- No timeout configuration.
- Unbounded thread pools.
- Non-idempotent retries creating duplicate orders/events.

**Tradeoffs:**

- Aggressive retries improve transient recovery but can amplify overload.

## 167. Advanced: What should you monitor in MongoDB production?

**Production-oriented answer:**  
Monitor slow queries, query plans, index usage, full collection scans, replication lag, election events, oplog window, disk usage, memory/working set, connection counts, lock/resource pressure, CPU, I/O latency, backup success, and restore testing.

**Real-world usage:**  
An alert on replication lag protects secondary-read freshness and failover safety. Slow query profiling catches missing indexes before traffic peaks.

**Interview traps:**

- CPU/memory only is not enough.
- Backups are not proven until restored.

**Common mistakes:**

- No alerts for disk growth.
- Ignoring unused indexes.
- No dashboard linking API latency to DB latency.

**Tradeoffs:**

- Detailed monitoring costs resources and money, but lack of monitoring costs outages.

## 168. Advanced: How do you optimize a slow MongoDB query?

**Production-oriented answer:**  
First reproduce the query shape and inspect `explain()`: documents examined, index used, sort stage, projection, and execution time. Then verify whether the data model, index, filter selectivity, sort, and pagination are appropriate.

Steps:

1. Identify endpoint and exact query.
2. Run `explain()` on production-like data.
3. Add or adjust compound index.
4. Reduce returned fields with projection.
5. Replace offset pagination with cursor pagination.
6. Avoid `$lookup`/aggregation on hot paths if possible.
7. Re-test and monitor after deployment.

**Interview traps:**

- Do not jump directly to caching.
- Adding an index may slow writes; evaluate workload.

**Common mistakes:**

- Optimizing with tiny local data.
- Ignoring sort.
- Returning huge documents.

**Tradeoffs:**

- Query-specific indexes improve latency but increase storage/write cost.

## 169. Advanced: How do you deal with replication lag?

**Production-oriented answer:**  
First detect lag through monitoring. Then decide whether the endpoint can tolerate stale reads. For read-your-writes flows, read from primary or use stronger read concern/session semantics. Reduce lag by fixing slow disks, overloaded secondaries, large index builds, network issues, or excessive write volume.

**Real-world usage:**  
After updating user settings, redirecting the user to a page that reads from secondary may show old settings. Use primary reads for that flow.

**Interview traps:**

- Secondary reads are not free scaling if freshness matters.
- Lag can become dangerous during failover.

**Common mistakes:**

- No lag alert.
- Using secondary reads globally.
- Running heavy analytics on secondaries that must be failover candidates.

**Tradeoffs:**

- Reading from primary improves freshness but increases primary load.

## 170. Advanced: What are common MongoDB data modeling mistakes?

**Production-oriented answer:**  
Common mistakes include modeling collections like SQL tables, embedding unbounded arrays, creating hot documents, duplicating data without synchronization, using `$lookup` as a default, ignoring schema versioning, and designing without query patterns.

**Real-world usage:**  
Storing all user notifications inside one user document eventually creates huge documents and slow updates. Better: separate notifications collection partitioned/indexed by `userId` and `createdAt`, with cursor pagination.

**Interview traps:**

- MongoDB flexibility makes bad modeling easy.
- Normalization rules from SQL do not directly apply.

**Common mistakes:**

- No document growth estimate.
- No index plan.
- No migration strategy.

**Tradeoffs:**

- Embedding improves locality but must be bounded.

## 171. Advanced: When should Redis not be used?

**Production-oriented answer:**  
Do not use Redis as the default source of truth for critical durable data unless persistence, replication, backups, and data-loss semantics are explicitly acceptable. Avoid Redis for large unbounded data, complex querying, or workloads where memory cost is too high.

**Real-world usage:**  
Use Redis for login session TTLs, but not as the only ledger for payments. Use it to cache product pages, but keep product source data in MongoDB or SQL.

**Interview traps:**

- Redis persistence exists, but Redis is still memory-first and operationally different from a durable OLTP database.

**Common mistakes:**

- No max memory policy.
- Big keys.
- Treating eviction as harmless.

**Tradeoffs:**

- Low latency versus memory cost and durability constraints.

## 172. Advanced: How do you choose between MongoDB and Cassandra?

**Production-oriented answer:**  
Choose MongoDB for document-shaped aggregates, rich secondary indexes, flexible querying, and application-facing read models. Choose Cassandra for huge distributed write-heavy workloads with predictable query patterns and high availability across nodes or regions.

**Real-world usage:**  
Product catalog: MongoDB. Device metrics at millions of writes per second: Cassandra. User sessions: Redis. Payments: SQL.

**Interview traps:**

- Cassandra is not better MongoDB. It has a stricter query-first model.
- MongoDB can scale, but shard key and query behavior matter.

**Common mistakes:**

- Using Cassandra then expecting ad-hoc filters.
- Using MongoDB for massive time-series writes without considering partitioning and retention.

**Tradeoffs:**

- MongoDB gives more flexible document querying.
- Cassandra gives stronger horizontal write scaling for predictable access patterns.

---

# Final Revision Checklist

Before the interview, make sure you can explain:

- Why NoSQL exists without saying “because it is faster.”
- Difference between replication and sharding.
- Embedding vs referencing in MongoDB with examples.
- How compound indexes are designed and why too many indexes hurt.
- What read preference, write concern, and replication lag mean in practice.
- Why shard key choice is difficult and important.
- When MongoDB transactions are useful and when they are a smell.
- Redis cache failure patterns: stampede, hot keys, TTL, eviction, locks.
- Cassandra partition key and tunable consistency basics.
- Why binary files usually belong in object storage, not documents.
- CAP and PACELC as production failure/latency reasoning tools.
- How Kafka, MongoDB, and Redis interact in eventually consistent microservices.
- What you would monitor before trusting a NoSQL system in production.
