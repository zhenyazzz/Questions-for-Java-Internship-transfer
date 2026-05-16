# NoSQL Databases — Senior Engineering Notes for Java Backend Interviews

## Table of Contents

[NoSQL Under Production Pressure](#nosql-under-production-pressure) · [Distributed Data Mechanics](#distributed-data-mechanics) · [NoSQL Database Types as Runtime Shapes](#nosql-database-types-as-runtime-shapes) · [MongoDB Internal Model](#mongodb-internal-model) · [MongoDB Data Modeling Under Load](#mongodb-data-modeling-under-load) · [MongoDB Indexing Mechanics](#mongodb-indexing-mechanics) · [MongoDB Replication and Elections](#mongodb-replication-and-elections) · [MongoDB Sharding and Distributed Queries](#mongodb-sharding-and-distributed-queries) · [Transactions, Retries, and Idempotency](#transactions-retries-and-idempotency) · [Redis Internals and Cache Behavior](#redis-internals-and-cache-behavior) · [Cassandra Architecture](#cassandra-architecture) · [Binary Data Storage](#binary-data-storage) · [NoSQL in Spring Boot Microservices](#nosql-in-spring-boot-microservices) · [Operational Failure Modes](#operational-failure-modes) · [CAP and PACELC in Real Systems](#cap-and-pacelc-in-real-systems) · [Interview Q&A](#interview-qa)

---

# NoSQL Under Production Pressure

NoSQL appears when the relational database is no longer only a persistence layer but the central bottleneck in the request path. The usual symptom is not that SQL “cannot scale”; it is that one database is expected to provide normalized storage, arbitrary joins, strict constraints, multi-row transactions, reporting queries, high-volume writes, low-latency reads, and cross-service integration at the same time. Under real traffic those requirements fight each other. Indexes needed for reads slow writes. Joins that are elegant in schema design become expensive in p99 latency. Long transactions hold resources while API threads wait. Shared schemas slow independent service deployment.

A senior engineer does not choose NoSQL because it is modern. The decision is normally forced by an access pattern. A product page wants one document-like aggregate. A session lookup wants one memory key. A metrics pipeline wants append-heavy writes partitioned by device and time. A graph traversal wants edge-local navigation. Once the access pattern is clear, the database choice becomes less ideological and more mechanical: which storage engine and distribution model produce the fewest failure modes for this traffic?

Vertical scaling keeps the system understandable. One larger PostgreSQL or MongoDB replica set is easier to reason about than a distributed cluster. The tradeoff is ceiling and blast radius. Horizontal scaling adds capacity by adding nodes, but every node boundary introduces routing, replication delay, metadata movement, partial failure, and retry ambiguity. After data is partitioned, a query either targets the right partition or fans out. After data is replicated, a read either sees the leader or risks staleness. After writes cross partitions, the system pays coordination cost.

The important mental shift is that NoSQL systems often move correctness from the database into the application and operational model. A relational schema can reject invalid foreign keys. A document database may accept a malformed document unless validation exists. A single SQL transaction can maintain an invariant across tables. A NoSQL service may need conditional updates, idempotency keys, event ordering, and reconciliation. The work does not disappear; it moves to code, topology, and runbooks.

---

# Distributed Data Mechanics

A distributed database request has two paths: the logical path the developer imagines and the physical path the system actually executes. The logical path is “save order” or “find products by category.” The physical path includes driver connection selection, topology discovery, routing metadata, leader or shard targeting, storage-engine lookup, replication acknowledgement, journaling, and result materialization. Latency grows when any step stops being local, indexed, or bounded.

Replication copies data for durability and failover. It does not reduce the amount of data each replica stores, and it does not repair bad query design. Replication introduces a timeline problem: the primary may have accepted a write while a secondary has not applied it yet. If the application reads from that secondary, it may observe old state. If a failover happens before a weakly acknowledged write reaches a majority, the write can be rolled back. Stronger acknowledgement reduces that risk by waiting for more replicas, but the request pays latency and may become unavailable when a quorum is unreachable.

Partitioning divides ownership. A partition key, shard key, or token determines where data lives. If the request includes that key, the database can route directly. If not, the system broadcasts, waits for many nodes, merges results, and turns one query into a distributed operation. This is why a shard key is not just a field; it is a contract between application access patterns and physical placement. A bad key becomes visible as hot shards, scatter-gather queries, uneven disk growth, and painful resharding.

Consistency becomes difficult because nodes do not share an instant global clock or instant communication. One service times out while the database later commits. Kafka delivers an event twice. A Redis invalidation arrives before the MongoDB write is visible to another reader. A secondary returns a value from before the user clicked Save. The system is not broken; the architecture allowed multiple observers to see different stages of propagation. Production correctness depends on deciding where that is acceptable and where the service must force coordination.

Eventual consistency is only safe when convergence is engineered. A Spring Boot catalog service updating MongoDB and emitting Kafka events must define the source of truth, event version, consumer idempotency, retry policy, cache invalidation behavior, and reconciliation path. Otherwise “eventual” becomes “maybe someday unless a consumer failed.” Experienced teams add version numbers to documents, use outbox tables or collections, make consumers ignore stale events, and periodically rebuild read models from the source of truth.

---

# NoSQL Database Types as Runtime Shapes

Document databases optimize for aggregate reads and localized updates. MongoDB works best when a request can load a document or a small indexed set of documents that already look close to the API response. The storage model tolerates heterogeneous fields, nested objects, and arrays, but under load the important question is whether the document stays bounded and whether its indexes match the request path.

Key-value systems optimize for direct access. Redis is effectively a remote memory data-structure server. It is excellent when the key is known: session token, product cache entry, rate-limit counter, lock key, idempotency marker. It becomes awkward when the application wants secondary filtering because the database will not discover relationships for you. Teams that force query workloads into Redis usually start building manual indexes in application code, which becomes fragile quickly.

Wide-column systems optimize for partition-local writes and reads. Cassandra is built for large distributed write volume where the query is known before the table is designed. It is not a general query engine. If a Spring Boot service writes device metrics by `device_id` and day bucket, Cassandra can spread load and retrieve bounded time ranges efficiently. If product management later asks for arbitrary filtering by firmware version, region, and status, the model may require new tables, duplicated writes, or a different analytical system.

Graph databases optimize relationship traversal. They are justified when the core operation is moving through edges, not simply storing entities that happen to be related. Fraud rings, authorization hierarchies, social connections, and dependency graphs can become unnatural in document or relational models once queries require multiple hops with changing depth.

Wrong database selection creates architectural scar tissue. MongoDB used for a ledger forces the application to rebuild constraints and audit guarantees that SQL provides naturally. Cassandra used for ad-hoc search pushes query complexity into duplicate tables. Redis used as durable primary state creates recovery and eviction risks. SQL used for highly variable catalog attributes can become a maze of sparse columns, EAV tables, or JSON blobs without the operational benefits of a document store.

---

# MongoDB Internal Model

MongoDB stores BSON documents in collections. BSON matters because values are typed and binary encoded, not plain text JSON. Java services usually map documents through Spring Data or the MongoDB driver into domain objects, and that mapping becomes part of the schema contract. A field changing from number to string is not “flexible”; it is a production deserialization and query bug waiting for a release.

A collection is physically and operationally closer to a set of indexed documents than to a relational table. It can contain different shapes, but production systems should keep shapes compatible enough for the same indexes, validation rules, and ownership model. Mixing unrelated document types in one collection makes query plans less predictable and complicates lifecycle management.

`ObjectId` allows distributed ID generation without a central sequence. That is useful for insert throughput and offline generation. It also contains timestamp information, which can be operationally convenient, but teams should not overfit business ordering or security behavior to it. Public business identifiers often deserve separate fields with explicit uniqueness and access rules.

MongoDB reads typically flow through the driver to a selected server, then through query planning, index traversal if available, document fetch from WiredTiger cache or disk, projection, and result serialization. A fast query is usually one where the index narrows the candidate set early, the working set is memory-resident, the projection is small, and the result size is bounded. A slow query often reads far more documents than it returns, sorts without index support, materializes large documents, or executes aggregation stages that spill.

MongoDB writes reach the primary in a replica set. The storage engine updates data and indexes, durability depends on journaling and write concern, and replication records are consumed by secondaries through the oplog. The write cost is not just the document mutation. Every affected index must be updated, and in a sharded cluster the routing and shard-key placement also matter. A collection with ten indexes can have very different write behavior from the same collection with two indexes.

WiredTiger uses an internal cache and concurrency control to keep hot data and indexes available. The phrase “working set” is practical: if the hot documents and indexes fit in memory, latency is stable; if they do not, the system increasingly waits on disk. Many MongoDB incidents are working-set incidents disguised as query incidents. The query was always inefficient, but it only became visible when memory stopped hiding it.

---

# MongoDB Data Modeling Under Load

MongoDB modeling starts with request shape, not entity shape. For a `GET /products/{id}` endpoint, storing title, attributes, category breadcrumb, image metadata, and a price snapshot in one product document can remove multiple joins and produce stable latency. For `GET /users/{id}/notifications`, embedding every notification in the user document will eventually create a large hot document. The difference is boundedness and write pattern, not whether the data is conceptually related.

Embedding is a locality optimization. It keeps data that is read together and updated together inside one atomic boundary. Referencing is a growth and independence optimization. It keeps large, shared, or independently changing data outside the parent. A senior MongoDB answer usually sounds like: embed the bounded owned part, reference the unbounded or shared part, and duplicate small immutable snapshots when the historical state matters.

Denormalization is a latency trade, not a modeling shortcut. If the order document stores product name and price at checkout time, that duplication is correct because orders need historical truth. If comments store author display name, the team must decide whether old comments should update after a profile rename. If search documents duplicate catalog fields, Kafka events and rebuild jobs must keep them convergent. Duplicated data without propagation rules is data drift.

The aggregation pipeline is powerful but not free. `$match` early with an index can be cheap. `$lookup` on a hot path turns document access back into join-like behavior. `$unwind` multiplies intermediate rows. `$group` and `$sort` can hold memory or spill to disk. Pipelines used for admin reporting may be fine; pipelines inside high-QPS user requests need the same scrutiny as relational query plans.

Document growth is one of the most common MongoDB failures. Arrays are seductive because they match JSON APIs, but unbounded arrays change the write and read profile over time. A user document with addresses is fine. A user document with every login event, notification, or audit record is an outage seed. Large documents increase network transfer, BSON decoding, cache pressure, and update cost. If the application paginates the child data, the child data probably deserves its own collection.

Hot documents create concurrency and throughput limits. A single counter document updated thousands of times per second, a single inventory record hit by a flash sale, or a daily stats document receiving all events can become the serialized point in an otherwise distributed system. The usual fix is to split the write path: bucket counters, append events, aggregate asynchronously, shard by tenant or hash, or use Redis for temporary high-frequency counters and flush durable summaries later.

Schema evolution is a deployment problem. MongoDB allows old and new documents to coexist, but Java code must read both during rolling deploys. The safe sequence is to deploy readers that tolerate missing/new fields, then write the new shape, then backfill if needed, then remove old compatibility. For non-trivial transformations, a `schemaVersion` field is often cheaper than guessing document shape from nullable fields.

Single-document atomicity is MongoDB's cheapest consistency primitive. Conditional updates can enforce state transitions without a transaction: update an order from `NEW` to `PAID` only if the current status is `NEW`. This pattern scales better than wrapping broad service logic in multi-document transactions. If the invariant naturally fits in one document, model it there. If it spans multiple documents or services, be honest about the coordination cost.

---

# MongoDB Indexing Mechanics

MongoDB indexes are B-Tree-like ordered structures. A query that can use a selective index walks a small ordered structure and fetches matching documents. A query without a useful index scans documents, tests predicates, possibly sorts in memory, and becomes slower as the collection grows. Indexing is not an afterthought; it is part of API design.

Compound indexes encode field order. For a tenant-scoped order list filtered by `tenantId` and `status`, sorted by `createdAt`, an index like `{ tenantId: 1, status: 1, createdAt: -1, _id: -1 }` can support equality filters and stable cursor pagination. If the API later adds filtering by `customerId`, the index may no longer fit. One index per field is not equivalent to one compound index matching the request.

Cardinality controls selectivity. An index on `status` where values are `NEW`, `PAID`, and `CANCELLED` may not reduce much work by itself. Combined with `tenantId` and time range, it can become useful. Indexes on low-cardinality fields often look correct in code review and fail in production because they still examine too many documents.

Multikey indexes index array elements. They make tag-like searches possible but can explode index entries when arrays grow. This is why array size is an indexing concern, not only a document modeling concern. A large indexed array amplifies writes and increases index memory requirements.

TTL indexes remove documents in the background. They are good for sessions, reset tokens, temporary imports, and expiring locks if the application still checks expiration logically. They are not precise timers. If security depends on a reset token expiring at exactly 10:00:00, the read path must validate the timestamp, not rely on physical deletion.

Partial indexes are operationally useful because they reduce index size and can express real constraints, such as unique email only for non-deleted users. Sparse and partial behavior must be understood by the query writer; a query that does not include the partial filter may not use the index as expected.

Covered queries avoid fetching the full document because the index contains all fields needed by filter and projection. They are useful for high-QPS list endpoints returning small rows. They are not a reason to put large or volatile fields into indexes. Every indexed field increases write amplification.

`explain()` is the interface between application assumptions and database reality. Engineers look at whether the winning plan uses the intended index, how many keys and documents were examined, whether a sort stage exists, and whether the query returns a tiny fraction of what it scans. A query that returns 20 documents after examining 2 million is a production incident even if it looks functionally correct.

Indexes improve reads by creating extra write work. Every insert must add index entries. Every update to indexed fields must update index structures. Every delete must remove index entries. A write-heavy collection with excessive indexes pays continuous write amplification and memory pressure. Removing unused indexes is performance work, but it must be done carefully because a rarely used operational endpoint may depend on them.

---

# MongoDB Replication and Elections

A MongoDB replica set has one primary that accepts writes and secondaries that replicate the oplog. The oplog is the ordered stream of operations secondaries apply. Replication is asynchronous unless the client waits for a stronger write concern. The difference between “primary accepted it” and “majority replicated it” is the difference between low latency and stronger failover safety.

Elections require majority. This is the mechanism that prevents split brain. If the current primary cannot see a majority, it must step down or stop being able to safely accept writes. During this window, Spring Boot services see transient write failures, driver retries, or timeouts. The application cannot assume the database is continuously writable just because the cluster has replicas.

Replication lag is not only a metric; it changes what users observe. If a service writes a profile update and then reads from a secondary, the user can see old data. If an API Gateway routes the next request to a different instance with a driver configured for secondary reads, read-your-writes can break. For critical flows, read from primary or use causally consistent sessions where appropriate.

Write concern chooses acknowledgement. `w:1` returns after the primary accepts the write. `majority` waits for a majority of voting nodes. Majority writes cost latency, especially across zones or regions, but reduce rollback risk during failover. Weak write concern may be acceptable for low-value telemetry but is dangerous for order state, account changes, or idempotency markers.

Read concern chooses visibility semantics. Stronger read concern can avoid observing data that may roll back, but it can wait longer or reduce availability. Read concern and read preference are often confused. Read preference chooses where to read; read concern chooses what visibility guarantee is required.

Failover makes ambiguous outcomes common. A Java service sends a write, the primary commits locally, the network drops, an election happens, and the client sees a timeout. Did the write happen? The only safe answer is that the client may not know. This is why operations that create external effects need idempotency keys and unique constraints. Retrying blindly can create duplicate orders, duplicate emails, or duplicate payment attempts.

---

# MongoDB Sharding and Distributed Queries

Sharding distributes a collection across shards using a shard key. Each shard is usually a replica set, so sharding and replication stack: sharding splits data; replication copies each split for availability. The `mongos` router uses metadata from config servers to decide which shard should receive a request.

A good shard key has high cardinality, distributes writes, appears in common queries, and does not change. These requirements conflict. A monotonically increasing timestamp has high cardinality but sends new writes to the same range. A tenant ID targets tenant queries but fails if one tenant dominates traffic. A hashed key spreads writes but may weaken range-query locality. Shard key design is choosing which pain will be least expensive later.

Chunk balancing moves ranges of data between shards. It keeps distribution healthy but consumes disk, network, and cache. During heavy balancing, query latency can shift because chunks move and caches cool. Teams sometimes discover too late that the cluster is “balanced” by data size but not by traffic because one shard owns the hot tenant or hot key range.

Scatter-gather queries are the tax for missing shard-key predicates. If a query does not include enough shard key information, `mongos` broadcasts to multiple shards and merges results. Sorting, limiting, grouping, or joining after scatter-gather multiplies work. A query that was acceptable on one replica set may become unacceptable after sharding because it now runs everywhere.

Resharding is possible but expensive. It means rewriting physical placement for large data while the application continues to operate. Even when the database supports online resharding, the operation consumes I/O and introduces risk. Experienced teams model expected cardinality, tenant growth, and top queries before the first shard key is chosen because changing it later is an architectural migration, not a small refactor.

Unique constraints become more subtle in sharded collections. Global uniqueness is expensive unless the unique index includes the shard key or the system can coordinate globally. This is why domain identifiers and tenant boundaries should be designed together.

---

# Transactions, Retries, and Idempotency

NoSQL systems try to keep the common write path local: one document, one partition, one leader, or one quorum. Multi-document and distributed transactions break that locality. They require tracking transaction state, coordinating commit, holding resources longer, detecting conflicts, and retrying ambiguous failures. The cost shows up as latency, lower throughput, and more failure cases during elections or partitions.

MongoDB supports multi-document transactions, but they should protect real invariants, not compensate for relational modeling habits. A small transaction that updates two documents inside the same service boundary can be reasonable. A transaction wrapped around every repository call is usually a sign that the document model is wrong or the service is ignoring cheaper primitives like conditional updates.

Cross-service consistency should not depend on a transaction spanning MongoDB, Redis, Kafka, and another service database. The usual production pattern is local commit plus outbox. The service writes durable state and an outbox event in one local atomic boundary. A publisher sends the event to Kafka. Consumers update their own stores idempotently. If a consumer fails, Kafka replay or reconciliation repairs the derived state.

Retry logic is dangerous because failures are often ambiguous. A timeout after sending a write does not prove the write failed. A retry after a Kafka rebalance may process the same event again. A Redis lock may expire while the worker is still running. Idempotency is the control surface: unique request IDs, conditional updates, processed-event records, version checks, and deterministic side effects.

Write amplification appears whenever one logical change writes multiple physical structures: primary document, several indexes, oplog, journal, replicas, Kafka outbox, Redis invalidation, search document, and audit log. Read amplification appears when one API request has to fetch data from multiple collections, services, caches, or shards. Scaling work often reduces amplification by changing the data model, adding a read model, or accepting controlled staleness.

---

# Redis Internals and Cache Behavior

Redis is a memory-first data-structure server. The request path is short: client command, event loop processing, in-memory structure mutation or lookup, optional persistence/replication work, response. This is why Redis can be extremely fast and also why slow commands, big keys, network stalls, or persistence pauses are visible across clients.

Memory is the real Redis capacity boundary. Key count, value size, allocator fragmentation, replication buffers, client output buffers, and persistence overhead all matter. A cache that “fits” by raw value size can still OOM after overhead. Once memory pressure hits, eviction policy becomes business behavior. Evicting product cache is acceptable; evicting sessions or idempotency keys can break user experience or correctness.

Cache-aside is the common Spring Boot pattern. The service reads Redis first, reads MongoDB or SQL on miss, then stores the value with TTL. On writes, it invalidates or updates the cache. The failure case is not the happy path miss; it is the write that commits but cache invalidation fails, or the hot key that expires during peak traffic and causes thousands of threads to rebuild the same value.

Cache stampede is a distributed coordination problem disguised as caching. If a product page key expires and all API instances miss simultaneously, the database receives a burst it was never sized for. TTL jitter, request coalescing, short rebuild locks, stale-while-revalidate, and prewarming all exist to keep cache misses from aligning.

Cache invalidation is consistency design. TTL-only accepts stale data until expiry. Explicit invalidation reduces staleness but couples the write path to Redis availability. Kafka-driven invalidation decouples services but introduces delay, duplicate events, and ordering issues. A mature system often combines event invalidation with TTL as a safety net.

Redis distributed locks are useful but frequently overtrusted. `SET key value NX PX ttl` is only the start. The lock owner needs a unique token, release must compare token before delete, TTL must cover expected work without blocking forever, and the protected operation must tolerate duplicate execution. If losing the lock creates financial inconsistency, Redis should not be the only guard.

Redis Pub/Sub is ephemeral. Subscribers that are offline miss messages. It fits live notifications where loss is acceptable. Kafka or Redis Streams fit durable consumer recovery better. Treating Pub/Sub like a durable broker leads to invisible data loss during deploys or reconnects.

Persistence changes Redis failure behavior but not its basic identity. RDB snapshots can lose recent writes. AOF reduces loss but adds write overhead and rewrite behavior. Replication is often asynchronous, so failover can promote a replica missing recent writes. If Redis stores critical state, the team must explicitly accept the data-loss window and recovery model.

---

# Cassandra Architecture

Cassandra is built around partitioned, replicated, write-optimized storage. There is no single primary for the whole database. The partition key is hashed into a token space, replicas are placed across nodes, and coordinators route reads and writes to the appropriate replicas. This architecture favors availability and write throughput but requires query-first modeling.

The write path is optimized for appends. A write reaches a coordinator, is sent to replicas, recorded in commit log, applied to an in-memory memtable, and later flushed to SSTables. Compaction merges SSTables over time. This is why writes are fast and why compaction, tombstones, and disk amplification matter operationally.

The read path can be more expensive than the write path. Cassandra may check memtables, multiple SSTables, bloom filters, partition indexes, and reconcile replicas depending on consistency level. Poor partition design or many SSTables increase read amplification. A table that writes beautifully can read poorly if partitions are huge or queries do not match clustering order.

Partition keys are the architecture. A key like `device_id + day_bucket` keeps time-series partitions bounded and queryable. A key like `device_id` alone may create a partition that grows forever. A key with low cardinality creates hot nodes. The model must distribute writes and keep reads bounded at the same time.

Tunable consistency controls how many replicas must participate. `ONE` reduces latency and improves availability but can read stale data. `QUORUM` requires overlapping read/write quorums for stronger consistency but costs latency and can fail when too many replicas are unavailable. `ALL` maximizes acknowledgement but is fragile. The consistency level is part of the endpoint contract, not just a database setting.

Deletes and TTL create tombstones. Tombstones must remain long enough for replicas to learn about deletions, then compaction can remove them. Heavy TTL workloads can create tombstone pressure, making reads slower and compaction heavier. Time-series systems using Cassandra must design retention, bucket size, and compaction strategy together.

Cassandra is a poor fit for ad-hoc product filtering or relational joins. It is a strong fit when the service knows queries upfront and needs high write throughput across nodes or regions. Experienced teams often write multiple denormalized tables for different queries and accept that consistency between them is application-managed.

---

# Binary Data Storage

Large binary objects rarely belong in operational database documents. Images, PDFs, and videos have different access patterns, CDN behavior, retention rules, and backup economics from metadata. Storing them inside MongoDB documents expands the working set, slows backup/restore, and makes ordinary queries carry storage they do not need.

The common production pattern is object storage for bytes and MongoDB for metadata. The document stores owner, object key, bucket, content type, size, checksum, processing status, and timestamps. The file is served through CDN or signed URL. The database controls authorization and lifecycle; object storage handles large-byte durability and delivery.

GridFS stores files in MongoDB by chunking them and storing metadata. It is useful when files must be replicated and backed up exactly with MongoDB or when operational constraints require database-managed file storage. It is not the default answer for product images in a high-traffic web application because it pushes file-serving load into the database tier.

The difficult part is cross-system consistency. If the service uploads to object storage and then fails before writing metadata, an orphaned object remains. If it writes metadata first and upload fails, the database points to missing bytes. Systems solve this with pending states, checksums, background cleanup, idempotent upload keys, and reconciliation jobs.

---

# NoSQL in Spring Boot Microservices

In microservices, NoSQL is usually tied to data ownership. A catalog service owns product documents. An order service owns order state. A session service owns Redis session keys. Other services should not write directly into those stores because direct database access bypasses invariants and turns private schema into public API.

Polyglot persistence is useful only when each database removes a real bottleneck. MongoDB can hold catalog aggregates, Redis can absorb hot reads and session lookups, Cassandra can ingest events, and SQL can protect financial state. The cost is operational: more drivers, connection pools, backups, dashboards, failure modes, and team knowledge.

Kafka is the bridge between local consistency and system-wide eventual consistency. A Spring Boot service updates its local database, writes an outbox event, publishes to Kafka, and consumers update their own stores. This avoids a distributed transaction across services but accepts that other services lag. The lag must be visible and acceptable.

Event-driven systems fail through duplicates, reordering, poison messages, and missed side effects. Idempotent consumers are mandatory. A consumer updating MongoDB should store event ID or version, apply only newer state, and make repeated processing harmless. A consumer invalidating Redis should tolerate deleting a key that is already gone. A consumer sending email should deduplicate before external side effects.

API Gateway and distributed caching add another consistency layer. If the gateway caches responses or routes requests across service instances, read-after-write behavior depends on cache invalidation, routing, and replica read preferences. A user updating profile data may hit one service instance that writes MongoDB, then another instance that reads Redis or a secondary and returns old data. The architecture must decide whether that is acceptable or force primary/cache-bypassed reads for that workflow.

Connection pools are part of service stability. A MongoDB or Redis outage should not consume every Tomcat/Netty thread. Pools need bounds, timeouts, and circuit breakers. Retries need budgets and jitter. Without backpressure, a slow database turns into gateway timeouts, then retry storms, then wider outage.

---

# Operational Failure Modes

Slow queries are usually not mysterious. The database examines too many documents, sorts without index support, spills aggregation to disk, reads cold data, or returns too much payload. The production task is to connect an API endpoint to its physical query plan and measure documents examined, keys examined, result size, and p95/p99 latency.

Missing indexes often survive development because local data is small. The failure appears after a product launch, tenant import, or marketing campaign. A new optional filter on an endpoint can force collection scans. Query review should be part of release review for any endpoint that filters, sorts, or paginates large collections.

Memory pressure changes latency shape. MongoDB gets slower when indexes and hot documents fall out of cache. Redis becomes dangerous when it nears max memory and starts evicting or rejecting writes. Java services fail when result sets, serialization buffers, or blocked request queues grow beyond heap assumptions. OOM is frequently caused by unbounded reads that were functionally correct.

Replication lag must be treated as user-visible risk. It affects secondary reads, failover recovery, and confidence in backups or oplog windows. Lag can be caused by disk saturation, network problems, write bursts, index builds, or overloaded secondaries. Dashboards should show lag against business tolerance, not only cluster health.

Hot partitions create asymmetric failure. Overall cluster CPU may look fine while one shard, Cassandra node, Redis slot, or Kafka partition is overloaded. The symptom is a narrow set of tenants, keys, or endpoints timing out. Fixing it usually requires changing key distribution, bucketing, caching, or isolating large tenants, not just adding random hardware.

Disk pressure is not just “database is almost full.” Journals, oplogs, SSTables, compaction, index builds, snapshots, and temporary aggregation files all need headroom. Running out of disk can stop writes, break replication, or prevent recovery. Time-to-full is more useful than percentage-full when growth is steep.

Backups are an operational feature only after restore is tested. Replication is not backup because it replicates mistakes. A real restore plan defines RPO, RTO, backup isolation, encryption, restore procedure, and application-level validation. Teams that never restore in staging discover during incidents that backups are incomplete, too slow, or incompatible with current deployment assumptions.

Connection pool exhaustion is a classic cascading failure. Database latency rises, application threads wait longer, pools fill, request queues grow, gateway retries multiply traffic, and the database receives even more work. Bulkheads, timeouts, circuit breakers, and retry budgets are not optional around NoSQL dependencies.

---

# CAP and PACELC in Real Systems

CAP is only useful when described as behavior during partition. A distributed system cannot guarantee that every reachable node answers every request and that every answer reflects the latest global state while nodes cannot communicate. If it preserves consistency, it refuses some operations. If it preserves availability, it accepts stale or conflicting operations and repairs later.

CA is not a serious target for a distributed database because partitions are part of the environment. A single-node system can avoid partition tolerance, but a multi-node database cannot. The practical question is which requests are allowed to fail during partition and which are allowed to return stale data.

MongoDB replica sets with majority elections are CP-leaning for writes. A primary that loses majority should stop accepting writes to avoid split brain. With majority write concern, acknowledged writes are safer across failover. With secondary reads or weak write concern, the client can observe more availability or lower latency but weaker freshness and rollback safety.

Cassandra is commonly AP-leaning because it can continue operating with reachable replicas depending on consistency level. `ONE` favors availability and latency. `QUORUM` coordinates more replicas for stronger consistency. Cassandra is not simply “inconsistent”; it exposes the consistency-latency-availability trade to the application.

Redis depends on topology. A standalone Redis instance is outside distributed CAP discussion. Redis with replication or cluster has asynchronous replication, failover windows, and slot availability behavior. As a cache, Redis often intentionally chooses latency over strict freshness. As a lock or primary state store, its failover semantics must be treated much more carefully.

PACELC is more useful for day-to-day engineering because most production time is not partitioned. Even when the network is healthy, the system chooses latency or consistency. Majority writes wait longer. Quorum reads wait longer. Reading from the nearest replica is faster but may be stale. A Redis cache is faster but may lag the database. Architecture reviews should discuss these normal-path tradeoffs, not only disaster scenarios.

---

# Interview Q&A

## 143. What are NoSQL databases and how do they differ from relational databases?

NoSQL databases are storage systems optimized around access patterns that do not fit normalized relational tables cleanly: document aggregates, direct key lookup, partition-local writes, or graph traversal. The production difference is the contract. A relational database gives joins, constraints, and strong transactional tools by default. A NoSQL system usually gives a more specialized read/write path and pushes more responsibility into modeling, indexing, consistency settings, and application code.

In a real Java backend, SQL may own payments because constraints and transactions are central, MongoDB may own product documents because the API reads product aggregates, Redis may absorb session and cache traffic, and Cassandra may store append-heavy metrics. The tradeoff is not “SQL vs NoSQL”; it is which subsystem should own which consistency and access pattern.

## 144. What are the main types of NoSQL databases? (Document, key-value, graph, columnar)

Document databases such as MongoDB store nested documents and are effective when API responses align with aggregate documents. Key-value databases such as Redis are optimized for known-key access, TTLs, counters, sessions, locks, and cache entries. Wide-column databases such as Cassandra organize data by partition key and clustering order, which fits high-volume writes and predictable reads. Graph databases optimize multi-hop relationship traversal.

The practical answer should include failure consequences. If a team uses Redis for query-heavy data, it starts building secondary indexes manually. If it uses Cassandra for arbitrary filtering, it fights the storage model. If it uses MongoDB for deep graph traversal, `$lookup` and recursive application logic become expensive. The database type should match the dominant query and failure profile.

## 145. In what cases is it better to use NoSQL instead of SQL?

NoSQL is a better fit when the required runtime shape is specialized: a product aggregate read as one document, a Redis key that must be read in sub-milliseconds, a Cassandra partition receiving massive time-series writes, or a graph traversal where relationship depth is the query. It is also useful when horizontal distribution and controlled eventual consistency are acceptable tradeoffs.

SQL remains the stronger default when correctness depends on relational constraints, complex joins, multi-row transactions, ad-hoc reporting, or financial auditability. Many mature systems use both: SQL for order/payment invariants, MongoDB for catalog read models, Redis for caching, Kafka for propagation, and object storage for files.

## 146. What popular NoSQL databases do you know? (MongoDB, Cassandra, Redis, etc.)

MongoDB is a document database with indexes, aggregation, replica sets, and sharding. Redis is an in-memory data-structure store used heavily for caching, sessions, rate limits, locks, and counters. Cassandra is a distributed wide-column store optimized for high write throughput and tunable consistency. DynamoDB is a managed key-value/document store with partition-based scaling. Neo4j is graph-oriented. Elasticsearch or OpenSearch is often used as a search read model, although it should not be treated as a primary transactional database for most business state.

A strong answer connects each name to write path, read path, and operational cost rather than listing products.

## 147. What is a collection in MongoDB and how does it differ from a table in SQL?

A MongoDB collection is a group of BSON documents sharing operational concerns: indexes, validation, storage behavior, and query patterns. A SQL table enforces a fixed column schema and relational constraints. A collection allows flexible document shapes, nested fields, and arrays, but production systems still need schema discipline because Java deserialization, indexes, and queries assume specific field types and shapes.

The difference matters during evolution. Adding a field to product documents can be a rolling change if readers tolerate missing values. Changing a field type without migration can break queries and DTO mapping. MongoDB flexibility speeds some changes but does not remove compatibility work.

## 148. What is a document in MongoDB?

A document is the atomic record MongoDB stores, encoded as BSON. In production it should be treated as an aggregate boundary. Data that is read together, updated together, and bounded can live together. Data that grows forever or has independent access patterns should not be embedded just because JSON makes it convenient.

An order document with embedded line items is usually sensible. A user document with every notification or login event is not. The latter grows, becomes expensive to read and update, increases cache pressure, and prevents efficient pagination.

## 149. How are relationships between documents defined in MongoDB? Is there JOIN?

MongoDB relationships are modeled through embedding, references, or duplication. `$lookup` exists and behaves like a join in aggregation, but join-heavy request paths usually indicate that the document model is not aligned with the access pattern. MongoDB performs best when the request targets one document or a small indexed set rather than assembling many normalized fragments at runtime.

Embedding is appropriate for bounded owned data, such as order lines. Referencing is appropriate for shared or unbounded data, such as customer identity or comments. Duplication is appropriate when a snapshot is required, such as product name and price inside an order. The real design question is how the relationship behaves under reads, writes, growth, and consistency requirements.

## 150. How is an index implemented in MongoDB and why is it needed?

MongoDB indexes are B-Tree-like structures that keep field values ordered and point to matching documents. They avoid scanning the whole collection and can support range queries, sorting, uniqueness, TTL, partial indexing, and covered queries. Without the right index, latency grows with data size.

Indexes are not free. They consume memory and disk, and every write must maintain them. A write-heavy collection with many indexes pays write amplification. A production indexing strategy starts from endpoint query shape: equality fields, range fields, sort order, projection, and cardinality. `explain()` confirms whether the database examines a small bounded set or silently scans too much.

## 151. What are sharding and replication in NoSQL?

Replication copies the same data to multiple nodes so the system can survive node failure and optionally serve reads from replicas. Sharding splits different data across nodes so the system can exceed the capacity of one replica set or node group. In MongoDB, a sharded cluster usually has each shard implemented as a replica set, so both mechanisms are layered together.

Replication introduces lag, elections, read preference, and write concern decisions. Sharding introduces shard-key design, routing, balancing, scatter-gather queries, and resharding cost. They solve different bottlenecks and create different operational problems.

## 152. How does replication differ from sharding?

Replication is duplication for availability; sharding is partitioning for capacity. A three-node replica set stores copies of the same dataset. A sharded cluster divides the dataset so each shard owns a subset. If each shard has replicas, the system has both capacity distribution and failover protection.

In production, confusing them leads to bad decisions. Adding replicas does not reduce per-node data size or write bottleneck on the primary. Adding shards does not automatically improve a query that lacks shard-key targeting; it can make that query more expensive by broadcasting it.

## 153. What is eventual consistency

Eventual consistency means different replicas or derived systems may temporarily disagree after a write, but the system is designed to converge. The important part is the convergence design. A product update may commit to MongoDB, publish to Kafka, invalidate Redis, update search, and update another service's read model at different times.

This is acceptable only when the business flow tolerates the window. Product descriptions can usually lag. Inventory, authentication, and payment state need stronger controls. Engineers use versions, idempotent consumers, outbox, retries, dead-letter handling, and reconciliation so eventual consistency is bounded rather than accidental.

## 154. How to store binary data (e.g., images) in MongoDB?

Usually the bytes go to object storage and MongoDB stores metadata: owner, object key, content type, size, checksum, processing status, and timestamps. This keeps large file traffic out of the operational database working set and allows CDN delivery, cheaper storage, and independent lifecycle policies.

GridFS is available when files need to be chunked and stored inside MongoDB's replication and backup model. It is not the default for high-traffic images because it makes the database handle large-byte storage and serving concerns. The hard production problem is consistency between object storage and metadata, solved with pending states, idempotent keys, cleanup jobs, and checksums.

## 155. What are TTL indexes in MongoDB?

TTL indexes cause MongoDB to delete documents after a timestamp or age threshold. They are useful for reset tokens, sessions, verification codes, temporary locks, and short-lived staging records. The deletion is asynchronous, so TTL should not be treated as an exact scheduler.

For security-sensitive flows, the application must check expiration during reads. The TTL index is cleanup, not the only enforcement. Accidentally adding TTL to business data is dangerous because deletion becomes automatic and easy to miss during review.

## 156. Can transactions be used in MongoDB? If so, how?

MongoDB supports atomic single-document updates and multi-document transactions. The preferred production design is to keep core invariants inside one document when that matches the domain, then use conditional atomic updates. Multi-document transactions are available when several documents must change together, but they add coordination, latency, transaction state, and retry complexity.

In a sharded cluster, transaction cost is higher because coordination can cross shards. Across microservices, MongoDB transactions do not solve system-wide consistency. A service should use local atomic state change plus outbox, Kafka events, idempotent consumers, and reconciliation rather than trying to transact across service databases.

## 157. How is fault tolerance ensured in NoSQL databases?

Fault tolerance is a combination of replication, partitioning, leader election or quorum mechanics, failover, durable logs, backups, client timeouts, retries, and operational monitoring. MongoDB uses replica sets and majority elections. Cassandra replicates partitions and lets the application choose consistency level. Redis Sentinel or Cluster can fail over, but asynchronous replication means recent writes can be lost.

The application must participate. A Spring Boot service needs bounded connection pools, short dependency timeouts, retry budgets, idempotency keys, and circuit breakers. Otherwise a partial database failure becomes thread exhaustion, gateway retries, and a larger outage.

## 158. What is the CAP theorem? What is the extension of the CAP theorem?

CAP describes the forced choice during a network partition: preserve consistency by refusing some operations, or preserve availability by answering with potentially stale or conflicting state. Partition tolerance is not optional in a real multi-node system.

PACELC extends the discussion to normal operation: if there is a partition, choose availability or consistency; else, choose latency or consistency. This is visible in everyday settings. Majority writes are safer but slower. Reading from a nearby replica is faster but can be stale. Redis cache reads are fast but may not reflect the latest database state.

## 159. Why is it impossible to ensure all three properties simultaneously?

During a partition, one side cannot know what happened on the other side. If it answers every request, it may answer from stale state or accept conflicting writes. If it refuses uncertain operations, it preserves consistency but sacrifices availability for those requests. Since partitions happen in distributed systems, consistency and availability cannot both be absolute under that failure.

MongoDB demonstrates this through majority elections. A primary that cannot reach majority must not keep accepting writes as if nothing happened, because another side may elect a new primary. Stopping writes hurts availability but avoids split brain.

## 160. Give examples of CP and AP databases.

ZooKeeper, etcd, and Consul are CP-style coordination systems. MongoDB replica sets with primary reads and majority write concern are CP-leaning for writes because they prefer safe leadership and majority acknowledgement over accepting writes during unsafe partitions. Cassandra and Dynamo-style systems are AP-leaning when configured for availability and eventual convergence.

The nuance is configuration. Cassandra with `QUORUM` is different from Cassandra with `ONE`. MongoDB with secondary reads and weak write concern exposes different behavior than MongoDB with majority concerns. CP/AP should be explained as failure behavior, not as a permanent product label.

## 161. How does the CAP theorem manifest in MongoDB?

MongoDB manifests CAP through replica-set leadership and majority rules. With majority write concern, writes are acknowledged only after enough replicas have them. If the primary loses majority, it cannot safely continue accepting writes, so write availability can drop during a partition or election. This is the consistency-preserving side of the tradeoff.

Reads depend on settings. Primary reads give the freshest normal view. Secondary reads can be stale because replication is asynchronous. Weak write concern can allow faster responses but increases rollback risk. In production, critical flows like checkout or account changes use primary/majority-oriented behavior, while dashboards may accept secondary reads and staleness for lower latency or reduced primary load.
