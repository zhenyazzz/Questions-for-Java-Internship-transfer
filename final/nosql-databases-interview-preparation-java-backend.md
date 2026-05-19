# NoSQL Databases — Senior Engineering Notes for Java Backend Interviews

## Table of Contents

- [NoSQL Under Production Pressure](#nosql-under-production-pressure)
- [Distributed Data Mechanics](#distributed-data-mechanics)
- [NoSQL Database Types as Runtime Shapes](#nosql-database-types-as-runtime-shapes)
- [MongoDB Internal Model](#mongodb-internal-model)
- [MongoDB Data Modeling Under Load](#mongodb-data-modeling-under-load)
- [MongoDB Indexing Mechanics](#mongodb-indexing-mechanics)
- [MongoDB Replication and Elections](#mongodb-replication-and-elections)
- [MongoDB Sharding and Distributed Queries](#mongodb-sharding-and-distributed-queries)
- [Transactions, Retries, and Idempotency](#transactions-retries-and-idempotency)
- [Redis Internals and Cache Behavior](#redis-internals-and-cache-behavior)
- [Cassandra Architecture](#cassandra-architecture)
- [Binary Data Storage](#binary-data-storage)
- [NoSQL in Spring Boot Microservices](#nosql-in-spring-boot-microservices)
- [Operational Failure Modes](#operational-failure-modes)
- [CAP and PACELC in Real Systems](#cap-and-pacelc-in-real-systems)
- [Interview Q&A](#interview-qa)

---

# NoSQL Under Production Pressure

### When the relational DB becomes the bottleneck

NoSQL normally enters the architecture when the relational database stops being only a persistence layer and becomes the central runtime bottleneck. The symptom is not “SQL cannot scale.” The symptom is one database being asked to provide normalized storage, arbitrary joins, strict constraints, multi-row transactions, reporting queries, high-volume writes, low-latency reads, and cross-service integration at the same time. Under traffic those requirements conflict. Indexes needed for reads slow writes. Joins that look clean in schema design become p99 latency. Long transactions hold resources while API threads wait. Shared schemas turn independent service deployment into database-change coordination.

### Choosing storage by access pattern

The database choice is usually forced by a request shape. A product page wants a document-like aggregate. A session lookup wants one memory key. A metrics pipeline wants append-heavy writes partitioned by device and time. A recommendation service may need edge-local traversal. Once the access pattern is clear, the choice becomes mechanical: which storage engine and distribution model produce the least read amplification, write amplification, and operational surprise for this workload?

### Vertical vs horizontal scaling

Vertical scaling keeps reasoning simple. One larger PostgreSQL instance or one larger MongoDB replica set avoids distributed routing and cross-node coordination. Teams often stay there longer than architecture diagrams suggest because simplicity has real value. Horizontal scaling adds capacity, but every node boundary introduces metadata, routing, replication delay, partial failure, and retry ambiguity. After data is partitioned, a query either targets the correct partition or fans out. After data is replicated, a read either follows the leader or risks staleness. After writes cross partitions, the system pays coordination cost.

### Correctness moves to the application

NoSQL often moves correctness from database constraints into application and operational design. A relational schema can reject invalid foreign keys. A document store may accept malformed documents unless validation exists. A SQL transaction can protect a multi-table invariant. A NoSQL service may need conditional updates, idempotency keys, event versions, deduplication records, and reconciliation jobs. The work does not disappear; it moves into code paths, topology choices, monitoring, and runbooks.

### Typical incremental adoption path

A production system usually evolves toward NoSQL incrementally. First the SQL schema gets JSON columns or read replicas. Then the hottest endpoint receives Redis cache. Then a read model appears because joining at request time is too expensive. Then Kafka becomes the propagation mechanism between the write model and read models. This path looks strange from pure data modeling, but it is common because companies optimize after a concrete bottleneck becomes measurable.

---

# Distributed Data Mechanics

### Logical vs physical request path

A distributed database request has a logical path and a physical path. The logical path is “save order” or “find products by category.” The physical path includes driver server selection, connection pool checkout, topology discovery, router metadata, leader or shard targeting, storage-engine lookup, index traversal, document materialization, journal or commit-log write, replication acknowledgement, and response serialization. Latency increases when any step stops being local, indexed, memory-resident, or bounded.

### Replication and timeline problems

Replication copies data for durability and failover. It does not reduce the amount of data each replica stores, and it does not repair a bad query. Replication introduces a timeline problem: the primary can accept a write before a secondary applies it. A secondary read can observe old state. A failover before a weakly acknowledged write reaches a majority can roll that write back. Stronger acknowledgement reduces that risk by waiting for more replicas, but the request now pays latency and can become unavailable when enough replicas cannot respond.

### Partitioning and shard keys

Partitioning divides ownership. A partition key, shard key, or token decides where data lives. If a request contains the routing key, the database can target the owner. If not, the system broadcasts, waits for many nodes, merges results, and turns one query into a distributed operation. This is why shard keys are not just schema fields. They are contracts between application access patterns and physical placement. A bad key becomes visible as hot shards, scatter-gather queries, uneven disk growth, expensive balancing, and painful resharding.

### Consistency under partial failure

Consistency becomes hard because no node sees a perfect global order of events. A Spring Boot service can timeout while MongoDB later commits. Kafka can deliver the same event twice after a rebalance. A Redis invalidation can arrive before another reader observes the MongoDB write. A secondary can return the state from before the user clicked Save. These are not exotic failures; they are ordinary outcomes when several systems observe different stages of the same business change.

### Engineering eventual consistency

Eventual consistency is safe only when convergence is engineered. A catalog update that writes MongoDB and emits Kafka events must define source of truth, version ordering, idempotent consumers, retry policy, dead-letter handling, cache invalidation behavior, and reconciliation. Otherwise “eventual” becomes “eventual unless a consumer died, an event was stale, or Redis kept the old value.” Mature systems add monotonically increasing versions, outbox publishing, consumer-side deduplication, and rebuild jobs that can regenerate derived views from authoritative state.

### Ambiguity: timeouts, retries, derived state

The subtle production problem is ambiguity. A timeout is not a negative acknowledgement. A retry is not harmless. A read after write is not guaranteed if the next request hits a cache or secondary. Distributed systems become difficult over time because more derived state appears: Redis caches, search indexes, analytics stores, denormalized MongoDB read models, Kafka compacted topics, and API Gateway caches. Each derived copy improves latency or autonomy, and each copy adds a new consistency edge to operate.

---

# NoSQL Database Types as Runtime Shapes

### Document databases

Document databases optimize aggregate locality. MongoDB is effective when the API can load a document or a small indexed set of documents that already resemble the response. The risk is pretending that flexible schema removes modeling discipline. In production, document shape controls working set, index size, serialization cost, update contention, and migration complexity.

### Key-value systems

Key-value systems optimize known-key access. Redis behaves like a remote in-memory data-structure server. It is excellent for session tokens, product cache entries, rate-limit counters, temporary locks, idempotency markers, and sorted-set leaderboards. It becomes awkward when the product asks for secondary filtering. Teams that force query workloads into Redis often build manual indexes in application code, which creates correctness problems during partial failures.

### Wide-column systems

Wide-column systems optimize partition-local throughput. Cassandra works when the query is known before the table is designed: write metric by device and time bucket, read metric ranges by the same key, expire data by retention policy. It is not a general query engine. If the access pattern changes from device/day lookup to arbitrary filtering by region, firmware, and status, the system usually needs another table, another read model, or an analytical store.

### Graph databases

Graph databases optimize traversal. They make sense when the core runtime operation is walking relationships of variable depth: fraud rings, dependency graphs, authorization hierarchies, social paths. They are not justified merely because entities are related. Many ordinary relationships are cheaper as references, joins, or denormalized summaries.

### Wrong database selection

Wrong database selection leaves architectural scar tissue. MongoDB used for a strict ledger forces application code to rebuild constraints and audit guarantees. Cassandra used for ad-hoc search turns every new query into another duplicated table. Redis used as durable primary state exposes eviction and failover semantics to business correctness. SQL used for highly variable catalog attributes often degenerates into sparse columns, EAV tables, or JSON blobs without the operational advantages of a document store.

---

# MongoDB Internal Model

### BSON and the Java contract

MongoDB stores BSON documents in collections. BSON is typed binary data, not plain JSON text. That matters for Java services because driver mapping, Spring Data converters, field types, date handling, decimal precision, and missing/null semantics all become part of the contract. A field drifting from number to string is not harmless flexibility; it changes index behavior and can break deserialization.

### Collections as operational boundaries

A collection is operationally a set of documents sharing indexes, validation, storage behavior, and ownership. It can contain different shapes, but unrelated shapes in one collection make query planning, validation, index selection, and lifecycle management worse. The practical boundary is not “same Java class”; it is “same access patterns and operational lifecycle.”

### ObjectId and public identifiers

`ObjectId` allows ID generation without a central sequence. That avoids a database round trip and works well for distributed inserts. It also has timestamp characteristics that are useful for rough ordering and debugging. It should not become a hidden business contract. Public identifiers, tenant-scoped IDs, and idempotency keys deserve explicit fields, explicit uniqueness, and explicit access rules.

### Read path

A MongoDB read path usually goes through driver server selection, query planning, index traversal, document fetch from WiredTiger cache or disk, projection, BSON decoding, and network serialization. Fast reads share a pattern: selective index, small candidate set, memory-resident working set, narrow projection, bounded result. Slow reads examine far more documents than they return, sort without index support, materialize large documents, or run aggregation stages that spill.

### Write path

A MongoDB write path reaches the primary, mutates the document, updates every affected index, records durable state through journaling, and appends operations for replication. The write is not just “change this JSON.” If the collection has many indexes, each write has more physical work. If the update touches indexed array fields, write amplification can be surprisingly large. If the collection is sharded, routing and shard-key immutability also matter.

### WiredTiger cache behavior

WiredTiger cache behavior explains many incidents. When the active documents and indexes fit in memory, latency looks stable. When the working set exceeds memory, the same query starts waiting on disk and p99 expands. Teams often misdiagnose this as “MongoDB got slow” when the real issue is that data growth, index growth, or a new query pushed the hot set beyond cache. The storage engine can hide inefficient access patterns until it no longer can.

---

# MongoDB Data Modeling Under Load

### Model from the endpoint, not the ER diagram

MongoDB modeling starts from the endpoint and the write path, not from an entity diagram. For `GET /products/{id}`, embedding title, attributes, category breadcrumb, image metadata, and a price snapshot can remove several joins and stabilize latency. For `GET /users/{id}/notifications`, embedding every notification into the user document creates unbounded growth and makes pagination fight the document model. Both structures are “related data”; only one is bounded and read as an aggregate.

### Embedding vs referencing

Embedding is a locality and atomicity optimization. It keeps bounded owned data inside one read and one single-document update boundary. Referencing is a growth and independence optimization. It keeps large, shared, frequently changing, or independently queried data outside the parent. Senior MongoDB modeling usually sounds like: embed the bounded owned part, reference the unbounded or shared part, and duplicate small immutable snapshots when historical correctness matters.

### Denormalization and propagation rules

Denormalization is not laziness; it is paying write-side complexity to buy read-side predictability. An order stores product name and price because the order needs checkout-time truth. Comments may copy author display name, but the team must decide whether old comments follow profile renames. A search read model duplicates catalog fields, but Kafka events and rebuild jobs must keep it convergent. Duplicated data without propagation rules becomes silent drift.

### Aggregation pipeline cost

The aggregation pipeline is useful but has a physical cost. `$match` early with index support can be cheap. `$lookup` makes MongoDB behave more like a join engine and can be acceptable for bounded admin workflows, but it is dangerous on hot OLTP paths. `$unwind` multiplies intermediate rows. `$group` and `$sort` can hold memory or spill to disk. A pipeline that looks compact in code can become a large memory and I/O operation at production cardinality.

### Document growth and unbounded arrays

Document growth is one of the most common MongoDB scaling failures. Arrays are convenient because they match JSON APIs, but unbounded arrays eventually change the runtime profile. A few addresses inside a user document are fine. Login events, notifications, audit records, or comments belong in separate collections when they need pagination, independent retention, or independent indexing. If an endpoint asks for page 3 of a child list, the child list probably should not be embedded as an ever-growing array.

### Hot documents and write contention

Hot documents serialize throughput. One counter document updated for every request, one daily statistics document hit by all workers, or one inventory document decremented during a flash sale can become the bottleneck of an otherwise distributed system. The usual evolution is bucketed counters, append-only events, asynchronous aggregation, hash-suffixed keys, or temporary Redis counters flushed into durable summaries. The strange-looking bucketed model exists because one “clean” document could not absorb the write concurrency.

### Schema evolution in rolling deploys

Schema evolution is a deployment problem. MongoDB allows old and new documents to coexist, but Java code must survive rolling deployments. Safe migrations deploy readers that tolerate both shapes, then writers that create the new shape, then backfill, then remove compatibility. A `schemaVersion` field is often cheaper than inferring shape from nullable fields. The operational mistake is assuming flexible schema means migrations are gone; they are now application-level compatibility work.

### Single-document atomicity

Single-document atomicity is MongoDB's cheapest consistency primitive. Conditional updates can enforce state transitions without a transaction, for example updating an order from `NEW` to `PAID` only when the current status is still `NEW`. If the invariant naturally fits inside one document, model it there. If it spans multiple documents or services, treat it as coordination work and avoid pretending a document database will make it free.

---

# MongoDB Indexing Mechanics

### B-Tree indexes and collection scans

MongoDB indexes are B-Tree-like ordered structures. A selective index lets the engine walk a small ordered range and fetch matching documents. Without a useful index, the engine scans documents, applies predicates, maybe sorts in memory, and latency grows with collection size. Indexing is not a DBA cleanup task after feature delivery; it is part of the API contract.

### Compound index order

Compound index order encodes the query shape. A tenant-scoped order list filtered by `tenantId` and `status`, sorted by `createdAt`, may need `{ tenantId: 1, status: 1, createdAt: -1, _id: -1 }` to support equality, sort, and cursor pagination. Separate single-field indexes are not equivalent. The database may intersect indexes in some cases, but relying on intersection for a high-QPS endpoint usually means the physical plan is weaker than the API needs.

### Cardinality and selective indexes

Cardinality decides whether an index narrows real work. `status` alone is usually weak because it has few values. `tenantId + status + createdAt` can be strong because it scopes the search to a tenant, state, and time range. Low-cardinality fields are not useless; they need to be placed in a compound index where earlier fields reduce the candidate set.

### Multikey indexes and arrays

Multikey indexes index array elements and can quietly multiply write cost. A document with a large indexed array produces many index entries. A write that modifies the array may touch a lot more physical index state than the application expects. This is why array growth is both a modeling issue and an indexing issue.

### TTL indexes

TTL indexes are cleanup mechanisms, not schedulers. They are good for sessions, reset tokens, verification codes, temporary import records, and staging data. Deletion is asynchronous. Security-sensitive code must still validate expiration on read, because physical deletion can lag. A TTL index on the wrong field can delete business data automatically and quietly.

### Partial indexes

Partial indexes reduce operational load by indexing only documents that matter for a query or constraint. Unique active email, active subscriptions, or non-deleted users are common cases. The query must include predicates compatible with the partial filter or the planner may not use the index. Partial indexes are powerful because they encode production reality: not every document state deserves equal index cost.

### Covered queries

Covered queries avoid fetching full documents when the index contains every filtered and returned field. They are useful for high-volume list endpoints that return small projections. They become counterproductive if engineers add large or volatile fields to indexes just to force coverage. A covered query is a latency tool, not a reason to index payloads.

### explain() and physical plans

`explain()` is where assumptions meet physics. Experienced engineers look for the winning plan, index bounds, keys examined, documents examined, sort stage, and returned count. A query returning 20 documents after examining 2 million is already broken even if it still passes functional tests. The best query plan is the one whose work is bounded by the access pattern, not by total collection size.

### Index cost on writes

Every index improves some reads by taxing all relevant writes. Inserts add index entries. Updates to indexed fields rewrite index state. Deletes remove entries. Replication must carry the resulting operations. Backups and memory must store the structures. A collection with too many indexes often fails by write latency and cache pressure before anyone notices that half the indexes are unused.

---

# MongoDB Replication and Elections

### Replica set, primary, and oplog

A MongoDB replica set has one primary that accepts writes and secondaries that apply the primary's oplog. The oplog is the ordered replication stream. Replication is asynchronous unless the client waits for stronger write concern. The difference between “the primary accepted it” and “a majority has it” is a business durability decision disguised as a driver option.

### Elections and majority

Elections require majority. This prevents split brain. A primary that cannot see a majority should step down or stop safely accepting writes. During that window, Spring Boot services can see transient write failures, retryable-write behavior, or timeouts. The database driver helps with topology discovery, but it cannot make a write election invisible to business logic.

### Replication lag and read preference

Replication lag is user-visible when applications read from secondaries. A profile update followed by a secondary read can show old data. An API Gateway routing the next request to another service instance can make this appear nondeterministic. For critical flows, use primary reads, causal consistency where appropriate, or explicit cache bypass after mutation. For dashboards, lag may be acceptable if the UI and alerts know the freshness contract.

### Write concern

Write concern defines acknowledgement. `w:1` returns after the primary accepts the write. `majority` waits for a majority of voting nodes. Majority writes add latency, especially across availability zones or regions, but reduce rollback risk. Weak write concern may be acceptable for low-value telemetry. It is risky for order state, account changes, idempotency markers, and outbox records because those are exactly the records used to recover from failure.

### Read concern vs read preference

Read concern defines visibility semantics. Read preference chooses where to read; read concern chooses what kind of committed view is required. Confusing the two leads to incorrect reasoning. A secondary read with weak assumptions may be fast and stale. A majority read can be safer but slower. The correct setting depends on whether the endpoint promises freshness, monotonic reads, or merely approximate state.

### Failover and ambiguous writes

Failover creates ambiguous outcomes. A Java service sends a write, the primary commits locally, the network drops, an election happens, and the client receives a timeout. The service may not know whether the write happened. Retrying order creation without an idempotency key can create duplicates. Retrying payment capture without an external idempotency contract can charge twice. Database reliability features reduce failure frequency; they do not remove ambiguity from distributed clients.

---

# MongoDB Sharding and Distributed Queries

### How sharding stacks with replication

Sharding splits a collection across shards using a shard key. Each shard is usually a replica set, so sharding and replication stack: sharding divides ownership, replication protects each division. `mongos` uses config-server metadata to route operations. A targeted query reaches one shard; an untargeted query becomes cluster-wide work.

### Shard key trade-offs

A shard key must balance cardinality, write distribution, query targeting, and stability. These goals conflict. A monotonically increasing timestamp has high cardinality but pushes new writes to the same range. Tenant ID targets tenant queries but fails when one tenant becomes huge. A hashed key spreads writes but weakens range locality. A compound key can balance concerns, but it also locks the application into including those fields on important queries.

### Hot shards

Hot shards often surprise teams because storage distribution can look healthy while traffic distribution is not. The cluster may be balanced by bytes, yet one shard owns the top tenant, hottest product, or newest time range. Adding shards does not automatically solve this; the workload must be redistributed. Senior engineers ask which key is hot, which query targets it, and whether the model needs bucketing, tenant isolation, caching, or a different write path.

### Chunk balancing

Chunk balancing moves data between shards. It is normal maintenance, but it consumes disk, network, and cache. Balancing can make latency noisy because chunks move and caches cool. If balancing is constantly chasing a bad shard key, the cluster is paying operational tax for a modeling decision.

### Scatter-gather queries

Scatter-gather queries are the cost of missing shard-key predicates. `mongos` broadcasts to shards, waits for results, and merges. Sorting, grouping, limiting, and aggregation after scatter-gather multiply work. A query that was tolerable on one replica set can become much worse after sharding because it now runs everywhere. Sharding increases capacity for targeted workloads; it punishes unfocused access patterns.

### Resharding as migration

Resharding is an architectural migration. Even with online support, it rewrites physical placement, consumes I/O, changes routing behavior, and requires operational planning. This is why shard-key design belongs in early architecture review for collections expected to grow. The question is not only “does this key distribute today's data?” but “what happens when the largest tenant is 100x bigger and the main query changes?”

---

# Transactions, Retries, and Idempotency

### Local vs distributed write paths

NoSQL systems try to keep the common write path local: one document, one partition, one leader, or one quorum. Multi-document and distributed transactions break locality. They require transaction state, commit coordination, conflict detection, longer resource retention, and retry handling. The cost appears as latency, lower throughput, and more complicated failure behavior during elections or partitions.

### MongoDB multi-document transactions

MongoDB supports multi-document transactions, but they should protect real invariants, not compensate for relational habits. A small transaction inside one service boundary can be correct. A transaction wrapped around every repository call often means the document model is fighting the workload. Conditional updates and single-document atomicity are cheaper tools when the invariant can be modeled locally.

### Cross-service consistency: outbox

Cross-service consistency should not depend on a transaction spanning MongoDB, Redis, Kafka, and another service database. The common production pattern is local commit plus outbox. The service writes durable state and an outbox event in one local atomic boundary. A publisher sends the event to Kafka. Consumers update their own stores idempotently. If a consumer fails, replay or reconciliation repairs derived state.

### Retries and idempotency keys

Retry logic is dangerous because failures are frequently ambiguous. A timeout after sending a write does not prove the write failed. A Kafka rebalance can make the same message execute again. A Redis lock can expire while the worker still runs. Idempotency is the control surface: unique request IDs, conditional updates, processed-event records, monotonic versions, compare-and-set semantics, and deterministic side effects.

### Write and read amplification

Write amplification appears when one business change updates many physical structures: primary document, indexes, oplog, journal, replicas, outbox, Kafka topic, Redis cache, search index, audit log. Read amplification appears when one API response pulls from multiple collections, services, shards, or caches. Many production architecture changes are attempts to move amplification away from synchronous user requests and into asynchronous pipelines where it can be retried and monitored.

---

# Redis Internals and Cache Behavior

### Request path and why Redis is fast

Redis is a memory-first data-structure server with a short request path: client command, event-loop processing, in-memory lookup or mutation, optional persistence and replication work, response. This is why Redis is fast and why slow commands, big keys, network stalls, persistence pauses, or blocked clients can be felt across unrelated callers.

### Memory, eviction, and business behavior

Memory is Redis capacity. Raw value size is not enough; key overhead, allocator fragmentation, replication buffers, client output buffers, and persistence overhead also count. Once Redis approaches its memory boundary, eviction policy becomes business behavior. Evicting product-page cache is acceptable. Evicting sessions, rate-limit keys, locks, or idempotency records can change correctness and user experience.

### Cache-aside in Spring Boot

Cache-aside is common in Spring Boot: read Redis, fall back to MongoDB or SQL on miss, then store with TTL. The difficult path is not a normal miss; it is a write that commits while cache invalidation fails, a hot key that expires during peak traffic, or Redis latency causing servlet threads to pile up. Caching reduces database load only when misses, failures, and rebuilds are controlled.

### Cache stampede

Cache stampede is distributed coordination pressure. If a hot product key expires and every API instance rebuilds it, MongoDB receives a burst that normal traffic never produces. TTL jitter, request coalescing, short rebuild locks, stale-while-revalidate, and prewarming exist to keep cache miss traffic from aligning. The goal is not perfect freshness; it is protecting the source of truth from synchronized demand.

### Cache invalidation strategies

Cache invalidation is consistency design. TTL-only accepts stale data until expiry. Explicit invalidation reduces staleness but couples the write path to Redis availability. Kafka-driven invalidation decouples services but introduces delay, duplicates, and ordering problems. Mature systems often combine event invalidation with TTL because one mechanism handles freshness and the other provides eventual repair.

### Redis locks

Redis locks are useful but often overtrusted. `SET key value NX PX ttl` is only a primitive. Correct use requires unique lock tokens, compare-and-delete release, bounded work time, handling lock expiry while work continues, and idempotent protected operations. If losing the lock can corrupt money movement or inventory correctness, Redis should not be the only guard.

### Pub/Sub vs durable messaging

Redis Pub/Sub is ephemeral. Offline subscribers miss messages. It fits live notifications where loss is acceptable, not durable business events. Redis Streams or Kafka are better when consumers need replay and recovery. Treating Pub/Sub as a broker usually works in development and loses messages during deploys.

### Persistence and failover

Persistence changes Redis failure behavior but not its core identity. RDB snapshots can lose recent writes. AOF reduces loss but adds write overhead and rewrite behavior. Replication is often asynchronous, so failover can promote a replica that is missing recent writes. If Redis stores critical state, the data-loss window and recovery model must be explicit.

---

# Cassandra Architecture

### Partitioned, write-optimized design

Cassandra is built around partitioned, replicated, write-optimized storage. There is no single global primary. A coordinator receives a request, locates replicas through the token ring, and waits for the consistency level required by the operation. This architecture favors availability and write throughput, but it makes query-first modeling non-negotiable.

### Write path: commit log and SSTables

The write path is optimized for sequential work. A write is sent to replicas, recorded in the commit log, applied to an in-memory memtable, and later flushed to SSTables. Compaction merges SSTables over time. This is why writes are fast and why compaction, tombstones, and disk amplification become operational concerns.

### Read path and read amplification

The read path can be more expensive than the write path. Cassandra may consult memtables, multiple SSTables, bloom filters, partition indexes, and multiple replicas, then reconcile results. Poor partition design or too many SSTables increases read amplification. A table can accept writes beautifully and still read poorly if partitions are huge or queries do not match clustering order.

### Partition keys as architecture

Partition keys are the architecture. `device_id + day_bucket` keeps time-series partitions bounded and queryable. `device_id` alone can create a partition that grows forever. Low-cardinality keys create hot nodes. A good key must distribute writes and keep reads bounded; optimizing only one side creates the next incident.

### Tunable consistency levels

Tunable consistency controls how many replicas participate. `ONE` favors latency and availability but can read stale data. `QUORUM` uses overlapping read/write quorums for stronger consistency but costs latency and can fail when enough replicas are unavailable. `ALL` is strict but fragile. The consistency level is part of the endpoint contract, not a tuning knob to change casually.

### Tombstones, TTL, and compaction

Deletes and TTL create tombstones. Tombstones must remain long enough for replicas to learn about deletions before compaction removes them. Heavy TTL workloads can make reads slow because the database must skip large numbers of tombstones. Time-series systems using Cassandra need retention, bucket size, compaction strategy, and query shape designed together.

### When Cassandra fits

Cassandra is not a good fit for ad-hoc product filtering or relational joins. It is strong when the service knows queries upfront and needs high write throughput across nodes or regions. Many Cassandra systems deliberately maintain multiple denormalized tables for different reads and accept that consistency between those tables is application-managed.

---

# Binary Data Storage

### Why large binaries rarely belong in documents

Large binary objects rarely belong in operational database documents. Images, PDFs, and videos have different access patterns, CDN behavior, retention rules, and backup economics from metadata. Putting them inside MongoDB expands the working set, slows backup and restore, and makes ordinary queries carry bytes they do not need.

### Object storage plus metadata in MongoDB

The common production pattern is object storage for bytes and MongoDB for metadata. The document stores owner, object key, bucket, content type, size, checksum, processing status, and timestamps. The database controls authorization and lifecycle. Object storage handles large-byte durability, range reads, CDN integration, and cheaper retention.

### GridFS

GridFS stores files in MongoDB by chunking them and storing metadata. It is useful when files must be replicated and backed up with MongoDB or when operational constraints require database-managed file storage. It is not the default for high-traffic images because it moves file-serving pressure into the database tier.

### Cross-system consistency for uploads

The hard part is cross-system consistency. Uploading bytes before metadata can leave orphaned objects. Writing metadata before upload can point to missing bytes. Production systems use pending states, idempotent upload keys, checksums, background cleanup, and reconciliation jobs. File storage is rarely a pure database question; it is a lifecycle question.

---

# NoSQL in Spring Boot Microservices

### Service ownership and store boundaries

In microservices, NoSQL choices usually follow ownership boundaries. A catalog service owns product documents. An order service owns order state. A session service owns Redis keys. Other services should not write directly into those stores because direct database access bypasses invariants and turns private schema into public API.

### Polyglot persistence cost

Polyglot persistence is useful only when each database removes a real bottleneck. MongoDB can hold catalog aggregates, Redis can absorb hot reads and session lookups, Cassandra can ingest events, and SQL can protect financial state. The cost is operational: more drivers, connection pools, backups, dashboards, failure modes, deployment runbooks, and on-call expertise.

### Kafka as the consistency bridge

Kafka is the bridge between local consistency and system-wide eventual consistency. A Spring Boot service updates its local database, writes an outbox event, publishes to Kafka, and consumers update their own stores. This avoids a distributed transaction across services but accepts lag. The lag must be measurable, acceptable, and repairable.

### Event-driven failure modes

Event-driven systems fail through duplicates, reordering, poison messages, schema changes, and missed side effects. Idempotent consumers are mandatory. A consumer updating MongoDB should store event ID or version and ignore stale events. A consumer invalidating Redis should tolerate deleting an already missing key. A consumer sending email or charging an external provider needs deduplication before the side effect, not after.

### API Gateway, caching, and read-after-write

API Gateway and distributed caching add another consistency layer. If the gateway caches responses or routes requests across service instances, read-after-write behavior depends on cache invalidation, routing, and replica read preference. A user may update profile data through one instance, then read through another instance that sees Redis or a secondary. The architecture must decide whether that endpoint tolerates stale state or needs primary/cache-bypassed reads for a short window.

### Connection pools and backpressure

Connection pools are part of the architecture, not framework defaults. A MongoDB or Redis slowdown should not consume every Tomcat or Netty worker. Pools need bounds, dependency timeouts, circuit breakers, and retry budgets. Without backpressure, a slow dependency becomes API Gateway timeouts, retries, queue growth, and a wider outage.

---

# Operational Failure Modes

### Slow queries and physical plans

Slow queries are usually physical, not mysterious. The database examines too many documents, sorts without index support, spills aggregation to disk, reads cold data, or returns too much payload. The production task is to connect an API endpoint to its physical plan: keys examined, documents examined, result count, sort stage, projection, payload size, and p95/p99 latency.

### Missing indexes after scale

Missing indexes survive development because local data is small. The failure appears after a tenant import, marketing campaign, or new optional filter. Query review belongs in release review for any endpoint that filters, sorts, or paginates large collections. A functional endpoint without an index plan is unfinished.

### Memory pressure

Memory pressure changes latency shape. MongoDB slows when hot indexes and documents fall out of cache. Redis becomes dangerous near max memory because eviction policy becomes business behavior. Java services fail when result sets, serialization buffers, or blocked queues exceed heap assumptions. OOM often starts as a functionally correct but unbounded read.

### Replication lag

Replication lag is user-visible risk. It affects secondary reads, failover recovery, freshness of dashboards, and confidence in oplog windows. Lag can come from disk saturation, network problems, write bursts, index builds, or overloaded secondaries. The useful alert is not “lag exists”; it is “lag exceeded the freshness contract for this workload.”

### Hot partitions

Hot partitions create asymmetric failure. Cluster averages look fine while one shard, Cassandra node, Redis slot, or Kafka partition burns. The symptom is usually a subset of tenants, accounts, products, or time ranges timing out. Adding hardware helps only if the key distribution changes or the hot tenant is isolated.

### Disk pressure beyond “almost full”

Disk pressure is broader than “database almost full.” Journals, oplogs, SSTables, compaction, index builds, snapshots, and temporary aggregation files need headroom. Running out of disk can stop writes, break replication, or prevent recovery. Time-to-full matters more than raw percentage when growth is steep.

### Backups and restore drills

Backups are not real until restore is tested. Replication copies mistakes. A production restore plan defines RPO, RTO, backup isolation, encryption, restore process, and application-level validation. Teams that never restore in staging often discover during incidents that backups are incomplete, slow, or incompatible with current schema assumptions.

### Connection pool exhaustion cascades

Connection pool exhaustion is a classic cascade. Database latency rises, application threads wait longer, pools fill, request queues grow, gateway retries multiply traffic, and the database receives even more work. Bulkheads, timeouts, circuit breakers, and retry budgets exist to stop one dependency from consuming the whole service.

---

# CAP and PACELC in Real Systems

### CAP during network partitions

CAP is useful only when tied to partition behavior. A distributed system cannot guarantee that every reachable node answers every request and that every answer reflects the latest global state while nodes cannot communicate. If it preserves consistency, it refuses some operations. If it preserves availability, it answers with potentially stale or conflicting state and repairs later.

### Why “CA” is not a distributed target

CA is not a serious target for a distributed database because partitions are part of the environment. A single-node system can avoid partition tolerance; a multi-node system cannot. The practical question is which requests are allowed to fail during partition and which requests are allowed to return stale data.

### MongoDB: CP-leaning writes

MongoDB replica sets with majority elections are CP-leaning for writes. A primary that loses majority should stop accepting writes to avoid split brain. Majority write concern makes acknowledged writes safer across failover. Secondary reads and weaker write concerns can reduce latency or improve perceived availability, but they weaken freshness and rollback safety.

### Cassandra: AP-leaning with tunable consistency

Cassandra is commonly AP-leaning because it can continue operating with reachable replicas depending on consistency level. `ONE` favors availability and latency. `QUORUM` coordinates more replicas for stronger consistency. Cassandra is not “inconsistent by design”; it exposes the consistency-latency-availability trade to the application.

### Redis and CAP

Redis depends on topology and role. Standalone Redis is outside distributed CAP discussion. Redis with replication or cluster has asynchronous replication, failover windows, and slot availability behavior. As a cache, Redis often intentionally chooses latency over strict freshness. As a lock or primary state store, its failover semantics deserve much more scrutiny.

### PACELC in normal operation

PACELC is more useful in daily architecture reviews because most production time is not partitioned. Even when the network is healthy, the system chooses latency or consistency. Majority writes wait longer. Quorum reads wait longer. Reading a nearby replica is faster but can be stale. Redis cache reads are fast but may lag the database. The normal path already contains the tradeoff.

---

# Interview Q&A

### 143. What are NoSQL databases and how do they differ from relational databases?

NoSQL databases are storage systems optimized around runtime access patterns that do not fit normalized relational tables cleanly: document aggregates, direct key lookups, partition-local writes, or graph traversal. The production difference is the contract. A relational database gives joins, constraints, and strong transactional tools by default. A NoSQL system usually gives a more specialized read/write path and pushes more responsibility into modeling, indexes, consistency settings, idempotency, and operational discipline.

In a real Java backend, SQL may own payments because constraints and transactions are central. MongoDB may own product documents because the API reads product aggregates. Redis may absorb session and cache traffic. Cassandra may store append-heavy metrics. The mature answer is not SQL versus NoSQL; it is assigning each workload to the storage model with the least dangerous failure mode.

### 144. What are the main types of NoSQL databases? (Document, key-value, graph, columnar)

Document databases such as MongoDB are effective when API responses align with aggregate documents. Key-value databases such as Redis are optimized for known-key access, TTLs, counters, sessions, locks, and cache entries. Wide-column databases such as Cassandra organize data by partition key and clustering order for high-volume writes and predictable reads. Graph databases optimize multi-hop relationship traversal.

The production part is knowing what breaks when the type is wrong. Redis used for query-heavy data forces manual secondary indexes. Cassandra used for arbitrary filtering forces duplicated tables or unacceptable scans. MongoDB used for deep traversal creates expensive `$lookup` or application recursion. The database type should match the dominant query and failure profile.

### 145. In what cases is it better to use NoSQL instead of SQL?

Use NoSQL when the runtime shape is specialized enough to justify moving away from relational guarantees: a product aggregate read as one document, a Redis key that must be read in sub-milliseconds, a Cassandra partition receiving huge time-series writes, or a graph traversal where relationship depth is the query. It also fits systems that can tolerate engineered eventual consistency for better latency, autonomy, or write distribution.

SQL remains stronger when correctness depends on relational constraints, complex joins, multi-row transactions, ad-hoc reporting, or financial auditability. Mature architectures often use both: SQL for order/payment invariants, MongoDB for catalog read models, Redis for caching, Kafka for propagation, and object storage for binaries.

### 146. What popular NoSQL databases do you know? (MongoDB, Cassandra, Redis, etc.)

MongoDB is a document database with indexes, aggregation, replica sets, and sharding. Redis is an in-memory data-structure store used for caching, sessions, rate limits, locks, counters, and idempotency markers. Cassandra is a distributed wide-column store optimized for high write throughput and tunable consistency. DynamoDB is a managed key-value/document store with partition-based scaling. Neo4j is graph-oriented. Elasticsearch or OpenSearch is commonly used as a search read model, not as the primary transactional database for most business state.

A strong answer connects each product to read path, write path, consistency behavior, and operational cost. Listing names is less useful than explaining where each one fails.

### 147. What is a collection in MongoDB and how does it differ from a table in SQL?

A MongoDB collection is a group of BSON documents sharing indexes, validation, storage behavior, and query patterns. A SQL table enforces fixed columns and relational constraints. A collection permits flexible document shapes, nested fields, and arrays, but production systems still need schema discipline because Java deserialization, indexes, and queries assume specific field types and structures.

The difference matters during evolution. Adding a field can be a rolling change if readers tolerate missing values. Changing a field type can break DTO mapping and index usage. MongoDB flexibility speeds some changes, but compatibility work moves into application code and migration jobs.

### 148. What is a document in MongoDB?

A document is the atomic record MongoDB stores, encoded as BSON. In production it should be treated as an aggregate boundary. Data that is read together, updated together, and bounded can live together. Data that grows forever or has independent access patterns should not be embedded just because JSON makes it easy.

An order document with embedded line items is usually sensible. A user document containing every notification or login event is not. The latter grows, becomes expensive to read and update, increases cache pressure, and prevents efficient pagination.

### 149. How are relationships between documents defined in MongoDB? Is there JOIN?

Relationships are modeled through embedding, references, or controlled duplication. `$lookup` exists and behaves like a join inside aggregation, but join-heavy request paths usually mean the document model is not aligned with the access pattern. MongoDB performs best when the request targets one document or a small indexed set rather than assembling many normalized fragments at runtime.

Embedding fits bounded owned data such as order lines. Referencing fits shared or unbounded data such as customer identity or comments. Duplication fits snapshots such as product name and price inside an order. The decision is driven by read locality, write frequency, growth, and consistency requirements.

### 150. How is an index implemented in MongoDB and why is it needed?

MongoDB indexes are B-Tree-like structures that keep field values ordered and point to matching documents. They avoid full collection scans and can support range queries, sorting, uniqueness, TTL cleanup, partial indexing, and covered queries. Without the right index, latency grows with collection size rather than result size.

Indexes consume memory and disk, and every write must maintain them. A write-heavy collection with too many indexes pays write amplification. Production index design starts from endpoint shape: equality fields, range fields, sort order, projection, and cardinality. `explain()` verifies whether the engine examines a bounded set or silently scans too much.

### 151. What are sharding and replication in NoSQL?

Replication copies the same data to multiple nodes so the system can survive node failure and optionally serve reads from replicas. Sharding splits different data across nodes so the system can exceed the storage or throughput capacity of one replica set. In MongoDB, each shard is usually a replica set, so sharding and replication are layered.

Replication introduces lag, elections, read preference, and write concern decisions. Sharding introduces shard-key design, routing, balancing, scatter-gather queries, and resharding cost. They solve different bottlenecks and create different operational problems.

### 152. How does replication differ from sharding?

Replication is duplication for availability; sharding is partitioning for capacity. A three-node replica set stores copies of the same dataset. A sharded cluster divides the dataset so each shard owns only part of it. If each shard has replicas, the system has both capacity distribution and failover protection.

Confusing them leads to bad scaling decisions. Adding replicas does not reduce per-node data size or remove the primary write bottleneck. Adding shards does not fix a query without shard-key targeting; it can make that query more expensive by broadcasting it.

### 153. What is eventual consistency

Eventual consistency means replicas or derived systems can temporarily disagree after a write, but the architecture is designed to converge. The important part is the convergence design. A product update may commit to MongoDB, publish to Kafka, invalidate Redis, update search, and refresh another service's read model at different times.

This is acceptable only when the business flow tolerates the window. Product descriptions can usually lag. Inventory, authentication, and payment state need stronger controls. Engineers use versions, outbox, idempotent consumers, retries, dead-letter handling, and reconciliation so eventual consistency is bounded rather than accidental.

### 154. How to store binary data (e.g., images) in MongoDB?

Usually the bytes go to object storage and MongoDB stores metadata: owner, object key, content type, size, checksum, processing status, and timestamps. This keeps large file traffic out of the operational database working set and allows CDN delivery, cheaper storage, and independent lifecycle policies.

GridFS is available when files need to be chunked and stored inside MongoDB's replication and backup model. It is not the default for high-traffic images because it makes the database handle large-byte storage and serving pressure. The hard production problem is consistency between object storage and metadata, handled with pending states, idempotent keys, cleanup jobs, and checksums.

### 155. What are TTL indexes in MongoDB?

TTL indexes cause MongoDB to delete documents after a timestamp or age threshold. They are useful for reset tokens, sessions, verification codes, temporary locks, and short-lived staging records. Deletion is asynchronous, so TTL should not be treated as an exact scheduler.

For security-sensitive flows, application code must check expiration during reads. The TTL index is cleanup, not the only enforcement. Accidentally adding TTL to business data is dangerous because deletion becomes automatic and easy to miss during review.

### 156. Can transactions be used in MongoDB? If so, how?

MongoDB supports atomic single-document updates and multi-document transactions. The preferred production design is to keep core invariants inside one document when that matches the domain, then use conditional atomic updates. Multi-document transactions are available when several documents must change together, but they add coordination, latency, transaction state, and retry complexity.

In a sharded cluster, transaction cost is higher because coordination can cross shards. Across microservices, MongoDB transactions do not solve system-wide consistency. A service should use local atomic state change plus outbox, Kafka events, idempotent consumers, and reconciliation rather than trying to transact across service databases.

### 157. How is fault tolerance ensured in NoSQL databases?

Fault tolerance is a combination of replication, partitioning, leader election or quorum mechanics, failover, durable logs, backups, client timeouts, retries, and operational monitoring. MongoDB uses replica sets and majority elections. Cassandra replicates partitions and lets the application choose consistency level. Redis Sentinel or Cluster can fail over, but asynchronous replication means recent writes can be lost.

The application must participate. A Spring Boot service needs bounded connection pools, short dependency timeouts, retry budgets, idempotency keys, and circuit breakers. Otherwise a partial database failure becomes thread exhaustion, API Gateway retries, and a larger outage.

### 158. What is the CAP theorem? What is the extension of the CAP theorem?

CAP describes the forced choice during a network partition: preserve consistency by refusing some operations, or preserve availability by answering with potentially stale or conflicting state. Partition tolerance is not optional in a real multi-node system.

PACELC extends the discussion to normal operation: if there is a partition, choose availability or consistency; else, choose latency or consistency. Majority writes are safer but slower. Reading from a nearby replica is faster but can be stale. Redis cache reads are fast but may not reflect the latest database state.

### 159. Why is it impossible to ensure all three properties simultaneously?

During a partition, one side cannot know what happened on the other side. If it answers every request, it may answer from stale state or accept conflicting writes. If it refuses uncertain operations, it preserves consistency but sacrifices availability for those requests. Since partitions happen in distributed systems, consistency and availability cannot both be absolute under that failure.

MongoDB demonstrates this through majority elections. A primary that cannot reach majority must not keep accepting writes as if nothing happened, because another side may elect a new primary. Stopping writes hurts availability but avoids split brain.

### 160. Give examples of CP and AP databases.

ZooKeeper, etcd, and Consul are CP-style coordination systems. MongoDB replica sets with primary reads and majority write concern are CP-leaning for writes because they prefer safe leadership and majority acknowledgement over accepting writes during unsafe partitions. Cassandra and Dynamo-style systems are AP-leaning when configured for availability and eventual convergence.

The nuance is configuration. Cassandra with `QUORUM` is different from Cassandra with `ONE`. MongoDB with secondary reads and weak write concern exposes different behavior than MongoDB with majority concerns. CP/AP should be explained as failure behavior, not as a permanent product label.

### 161. How does the CAP theorem manifest in MongoDB?

MongoDB manifests CAP through replica-set leadership and majority rules. With majority write concern, writes are acknowledged only after enough replicas have them. If the primary loses majority, it cannot safely continue accepting writes, so write availability can drop during a partition or election. This is the consistency-preserving side of the tradeoff.

Reads depend on settings. Primary reads give the freshest normal view. Secondary reads can be stale because replication is asynchronous. Weak write concern can allow faster responses but increases rollback risk. In production, critical flows like checkout or account changes use primary/majority-oriented behavior, while dashboards may accept secondary reads and staleness for lower latency or reduced primary load.
