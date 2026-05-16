# NoSQL Databases — Production Interview Preparation

## Table of Contents

- [NoSQL Fundamentals](#nosql-fundamentals)
- [Types of NoSQL Databases](#types-of-nosql-databases)
- [MongoDB Internals and Modeling](#mongodb-internals-and-modeling)
- [MongoDB Indexing](#mongodb-indexing)
- [Replication](#replication)
- [Sharding](#sharding)
- [Transactions and Consistency](#transactions-and-consistency)
- [Redis](#redis)
- [Cassandra](#cassandra)
- [Binary Data Storage](#binary-data-storage)
- [NoSQL in Microservices](#nosql-in-microservices)
- [Monitoring and Production Issues](#monitoring-and-production-issues)
- [CAP and PACELC](#cap-and-pacelc)
- [Interview Q&A](#interview-qa)

---

# NoSQL Fundamentals

NoSQL is a storage decision, not a performance label. A team chooses it when the required data model, write/read scale, latency, or availability model does not fit a single relational database cleanly.

## Why relational databases become problematic at scale

Relational databases are strong when data is normalized, joins are valuable, constraints are strict, and transactions are central. Problems usually appear in production for one of these reasons:

| Pressure | What happens |
|---|---|
| Write throughput exceeds one primary node | Vertical scaling delays the problem but does not remove the single-writer bottleneck. |
| Queries need many joins on hot paths | CPU, memory, locks, and query planning become latency risks. |
| Data shape changes frequently | Schema migrations become deployment coordination work. |
| Tables grow beyond memory-friendly indexes | Queries degrade into disk-heavy operations. |
| Multi-region availability is required | Cross-region strong consistency adds latency or reduces availability. |
| One schema is shared by many services | Teams become coupled through database structure and migration order. |

SQL can scale far. The issue is not that SQL is weak; the issue is that some workloads need a different contract: denormalized reads, partition-first writes, cache-like latency, or availability under partial failure.

## Horizontal vs vertical scaling

**Vertical scaling** adds CPU, memory, disk, and I/O to one machine. It keeps operations simple because data remains local, transactions remain easier, and queries do not require distributed routing. The limit is cost, hardware ceiling, and failure blast radius.

**Horizontal scaling** adds machines and distributes data. It increases capacity but introduces routing, replication lag, rebalance operations, node failure handling, cross-node consistency, and operational complexity. A horizontally scaled database is not just a bigger database; it is a distributed system.

Production decision:

- Use vertical scaling while the workload fits and operational simplicity matters.
- Move to horizontal scaling when data size, write throughput, availability, or isolation requirements justify distributed failure modes.

## Distributed systems basics

Distributed databases split responsibility across nodes. Every request now depends on several unstable conditions:

- Which node owns the data.
- Which replicas are current.
- Whether a leader exists.
- Whether a quorum can be reached.
- Whether routing metadata is fresh.
- Whether client retries are safe.

Common failure modes:

- Node crash.
- Network partition.
- Slow disk causing replication lag.
- Leader election during writes.
- Client timeout while the database later commits the operation.
- Duplicate retry after partial success.
- Stale reads from replicas.

Backend services must assume ambiguous outcomes. A timeout does not prove the write failed. A retry can duplicate a side effect unless the operation is idempotent.

## CAP theorem intuition

CAP matters during a network partition. If two groups of nodes cannot communicate, the system must choose between:

- **Consistency**: do not allow operations that could create conflicting state.
- **Availability**: keep answering reachable clients even if responses may be stale or divergent.

Partition tolerance is mandatory once data spans machines. Networks fail, packets drop, DNS breaks, nodes pause, and cross-zone links degrade.

Operational examples:

- A primary database that cannot reach majority stops accepting writes to avoid split brain.
- An AP-style store accepts writes on reachable replicas and resolves conflicts later.
- A cache keeps serving stale data because returning something is better than failing the page.

## Eventual consistency

Eventual consistency means replicas or derived systems converge after propagation completes. It is not an excuse for undefined correctness. Engineers must define:

- Source of truth.
- Maximum acceptable staleness.
- Conflict handling.
- Retry behavior.
- Idempotency keys or version checks.
- Reconciliation jobs.

Typical flow in a Spring Boot system:

1. Catalog service updates MongoDB.
2. Outbox event is published to Kafka.
3. Search service updates Elasticsearch/OpenSearch.
4. Redis cache is invalidated or expires.
5. Order service updates a product snapshot.

For seconds or minutes, different systems can show different values. That is acceptable for product descriptions, risky for inventory, and usually unacceptable for money movement without compensating controls.

## Replication

Replication keeps copies of data on multiple nodes. It solves node failure and improves durability. It can also serve reads, but read scaling through replicas introduces staleness unless reads are coordinated.

Replication does not solve:

- Bad queries.
- Hot partitions.
- Missing indexes.
- Logical corruption.
- Application bugs.
- Backup requirements.

If an application deletes the wrong data, replication quickly copies the deletion. Backups and restore drills remain mandatory.

## Partitioning

Partitioning splits data by key, range, hash, tenant, time bucket, or shard key. It solves capacity and throughput limits by making different nodes responsible for different data.

Partitioning creates new constraints:

- Queries must include the partition key to avoid scatter-gather.
- Hot keys overload one partition.
- Rebalancing consumes network and disk I/O.
- Cross-partition transactions are expensive.
- Unique constraints are harder unless scoped by partition key.

A partition key is an operational decision. It determines where write load lands, how reads are routed, and how painful future growth becomes.

## Why distributed consistency is hard

Consistency is hard because nodes observe events in different orders and communication is unreliable. A service may see:

- Write acknowledged by one replica but not yet replicated.
- Primary fail after local commit but before majority replication.
- Retry arriving after a new leader is elected.
- Consumer processing Kafka events out of order.
- Cache invalidation reaching Redis before database transaction commits.

The practical answer is not “make everything strongly consistent.” Strong coordination increases latency, reduces availability during failures, and limits throughput. Engineers choose consistency per workflow.

---

# Types of NoSQL Databases

Different NoSQL systems optimize different access patterns. Choosing the wrong type usually creates architectural debt because the application starts compensating for missing database primitives.

## Document databases

Document databases store nested records, usually JSON-like. MongoDB stores BSON documents in collections.

Production fit:

- Product catalogs with variable attributes.
- CMS/content documents.
- User profiles and preferences.
- Read models built from events.
- Aggregates usually read and updated together.

Failure pattern:

- Treating documents like normalized tables creates many application joins.
- Embedding unbounded arrays creates large documents and write contention.
- Flexible schema without validation creates incompatible document shapes.

## Key-value databases

Key-value stores retrieve values by key. Redis is the common backend example, with richer data structures than a simple map.

Production fit:

- Cache-aside reads.
- Sessions and tokens with TTL.
- Rate limits and counters.
- Feature flags or short-lived coordination.
- Leaderboard-like sorted sets.

Failure pattern:

- Expecting complex secondary queries.
- Treating cache as durable source of truth.
- Hot keys, big keys, memory eviction, and cache stampedes.

## Column-family databases

Wide-column systems such as Cassandra store rows by partition key, with clustering columns and flexible columns. They are designed around known queries and high write throughput.

Production fit:

- Time-series events.
- IoT/device metrics.
- Clickstream ingestion.
- Audit logs at massive volume.
- Multi-node availability with tunable consistency.

Failure pattern:

- Querying by fields not in the data model.
- Creating huge partitions.
- TTL/delete tombstone pressure.
- Assuming Cassandra behaves like SQL with a different query language.

## Graph databases

Graph databases store nodes and edges and optimize relationship traversal.

Production fit:

- Fraud rings.
- Social graph traversal.
- Dependency graphs.
- Recommendation paths.
- Authorization relationships with deep hierarchy.

Failure pattern:

- Using a graph database only because entities are related. Most business relationships do not require graph traversal.
- Replacing simple joins with graph infrastructure unnecessarily.

## Choosing the wrong database type

A wrong database choice forces expensive compensations:

| Wrong fit | Result |
|---|---|
| Redis for query-heavy data | Manual secondary indexes in application code. |
| MongoDB for deep graph traversal | Expensive `$lookup`, recursive logic, or duplicated graph state. |
| Cassandra for ad-hoc filtering | One table per query, duplicated data, or unacceptable scans. |
| SQL for highly variable product attributes | Complex EAV tables, sparse columns, or JSON fields with weak indexing. |
| MongoDB for strict financial ledger | Application must rebuild constraints SQL gives natively. |

The database should match the highest-risk access pattern, not the most familiar API.

---

# MongoDB Internals and Modeling

MongoDB works well when documents represent application aggregates: data that is commonly read together, updated together, and owned by one service boundary.

## BSON documents

MongoDB stores BSON, a binary representation of JSON-like documents. BSON supports nested fields, arrays, dates, decimals, ObjectIds, binary data, and typed values.

Production implication:

- Documents map naturally to Java DTOs and API payloads.
- Nested structure reduces join needs.
- Large nested documents increase network, memory, and update cost.
- Type drift breaks deserialization and query behavior.

## Collections

A collection groups documents. It is less rigid than a SQL table, but production systems still need schema discipline through:

- Spring validation.
- DTO compatibility rules.
- MongoDB JSON schema validation when useful.
- Versioned documents.
- Migration/backfill jobs.

Putting unrelated document types into one collection usually makes indexing, validation, and ownership worse.

## ObjectId

`ObjectId` is a common `_id` type generated without central coordination. It includes a timestamp component and avoids a database round trip for ID generation.

Use business IDs when external systems require stable domain identifiers, but keep uniqueness and immutability explicit. Do not expose internal IDs if they create security or coupling issues.

## Embedding vs referencing

Embedding stores child data inside the parent document. Referencing stores another document's ID.

Use embedding when:

- Child data is always read with the parent.
- Child lifecycle belongs to the parent.
- Child count is bounded.
- Single-document atomicity is useful.

Use references when:

- Child data grows without bound.
- Child is shared by many parents.
- Child changes independently at high frequency.
- Child needs independent query patterns.

Example:

- Embed order lines in an order.
- Reference customer ID from an order.
- Store product price snapshot in an order because historical checkout state must not change when catalog price changes.

## Access-pattern-first design

MongoDB modeling starts from endpoint behavior:

- Which fields filter the query?
- Which sort order is required?
- Which fields are returned?
- Which data is updated atomically?
- How large can arrays grow?
- Which fields change frequently?
- Which access path dominates traffic?

A model that looks clean but requires five queries per request is usually worse than a denormalized document with controlled duplication.

## Denormalization

Denormalization stores duplicated data to make reads cheaper and more predictable.

Production usage:

- Product summary duplicated into search read model.
- User display name copied into comments.
- Price snapshot copied into order line.
- Category breadcrumb embedded in product document.

Denormalization needs an update strategy:

- Synchronous update when strong consistency is required and scope is small.
- Kafka event propagation when eventual consistency is acceptable.
- Periodic reconciliation to repair missed updates.
- Version fields to ignore stale events.

## Aggregation pipeline

The aggregation pipeline transforms documents through stages like `$match`, `$project`, `$group`, `$sort`, `$lookup`, and `$unwind`.

Production use:

- Build reports for bounded datasets.
- Project API-specific shapes.
- Group data for admin dashboards.
- Run offline or asynchronous transformations.

Hot-path risks:

- `$lookup` can behave like a join and become expensive.
- `$group` and `$sort` can spill to disk.
- Pipeline stages before selective `$match` waste resources.
- Aggregations over large collections need careful indexes and limits.

## Why JOINs are avoided

MongoDB supports `$lookup`, but join-heavy OLTP removes the main benefit of document storage. Joins increase CPU, memory, network, and query planner complexity. In high-throughput services, engineers usually optimize for single-document or targeted multi-document reads.

Use joins when they are bounded, indexed, measured, and not on the hottest path. Avoid using MongoDB as if it were a relational database with weaker constraints.

## Document growth problems

Documents have a maximum size and practical performance limits before that. Large documents cause:

- Higher network transfer.
- More memory pressure.
- Slower serialization/deserialization.
- More expensive updates.
- Larger indexes if arrays are indexed.

Unbounded arrays are the common cause. Store comments, events, notifications, and audit entries in separate collections with pagination.

## Hot document problems

A hot document is updated by many concurrent requests. Examples:

- One counter document for all requests.
- One inventory document with heavy decrement traffic.
- One daily statistics document receiving every event.

Solutions:

- Bucket counters by time or hash.
- Use Redis atomic counters for temporary aggregation, then flush.
- Model events append-only and aggregate asynchronously.
- Use optimistic locking/version checks when conflicts matter.

## Schema evolution

MongoDB allows mixed document shapes; Java services still need compatibility.

Production approach:

1. Add new optional field with default behavior.
2. Deploy code that reads old and new shapes.
3. Start writing new shape.
4. Backfill old documents if needed.
5. Remove old read path only after data is migrated.

Use `schemaVersion` when transformations are non-trivial. Avoid big-bang migrations on high-traffic collections.

## Single-document atomicity

MongoDB updates to one document are atomic. This is one reason embedding is valuable. If an order and its items live in one document, status and line updates can be kept consistent without a multi-document transaction.

If an invariant spans multiple documents or services, single-document atomicity no longer helps. Use a transaction inside one MongoDB deployment only when necessary; use events and sagas across services.

---

# MongoDB Indexing

Indexes are part of application design. A MongoDB collection without query-matching indexes becomes unstable as data grows.

## B-Tree indexes

MongoDB indexes are B-tree-like ordered structures. They allow efficient equality lookup, range scans, and sorted reads. The database pays for that speed on writes because each insert/update/delete must maintain affected indexes.

Production impact:

- Good indexes reduce p95/p99 latency.
- Large indexes compete for memory.
- Excess indexes reduce write throughput.
- Index builds and changes are operational events.

## Compound indexes

Compound indexes support queries involving multiple fields. Field order matters.

Typical pattern:

```javascript
db.orders.createIndex({ tenantId: 1, status: 1, createdAt: -1, _id: -1 })
```

For query:

```javascript
{ tenantId: "t1", status: "PAID", createdAt: { $lt: cursor } }
```

with sort:

```javascript
{ createdAt: -1, _id: -1 }
```

Equality fields usually come before range/sort fields. Design from the real query, not from entity fields.

## Multikey indexes

A multikey index indexes array elements. It is useful for searching documents by tags or embedded values.

Risk:

- Large arrays create many index entries per document.
- Compound multikey indexes have restrictions and can grow quickly.
- Array indexing can hide document growth problems until write latency spikes.

## TTL indexes

TTL indexes delete documents after time based on a date field.

Use cases:

- Sessions.
- Password reset tokens.
- Verification codes.
- Temporary imports.
- Short-lived audit buffers.

TTL deletion is background cleanup, not a precise scheduler. Do not use it for exact business deadlines.

## Partial indexes

Partial indexes include only documents matching a filter.

Example:

```javascript
db.users.createIndex(
  { email: 1 },
  { unique: true, partialFilterExpression: { deleted: false } }
)
```

This can enforce uniqueness only for active users and reduce index size.

## Covered queries

A covered query is answered from the index without fetching documents. It requires all filtered and returned fields to be in the index.

Use it for high-traffic list endpoints returning small projections. Do not over-index large fields just to force coverage.

## Explain plans

`explain()` shows whether the query uses an index, how many keys/documents are examined, whether sorting is in memory, and where time is spent.

Operational review should check:

- `COLLSCAN` vs index scan.
- Documents examined vs documents returned.
- Sort stage behavior.
- Index bounds.
- Whether the winning plan matches the expected access pattern.

## Index cardinality

High-cardinality fields filter well: `userId`, `tenantId`, `email`, `orderId`. Low-cardinality fields like `status` or `enabled` alone often filter poorly.

Low-cardinality fields can still be useful inside compound indexes when combined with tenant, date, or other selective fields.

## Full collection scans

A full collection scan reads every document to answer a query. It may work in development and fail in production.

Common causes:

- Missing index.
- Query field order not matching compound index use.
- Regex without anchored prefix.
- Sorting without supporting index.
- Type mismatch between stored value and query value.
- Query on field with low selectivity.

## Production indexing strategy

Index per endpoint, not per field:

1. Capture query filter, sort, projection, cardinality, and expected result size.
2. Create the minimum compound index that supports it.
3. Validate with production-like data and `explain()`.
4. Monitor index usage and write latency.
5. Remove unused indexes carefully.
6. Revisit indexes when product filters change.

Indexes are part of release design. A new API filter without an index is a production risk.

---

# Replication

Replication keeps multiple copies of data so the system can survive node failures and sometimes serve read traffic from replicas.

## Replica sets

A MongoDB replica set usually contains one primary and multiple secondaries. Writes go to the primary. Secondaries replicate the primary's operation log.

Production properties:

- A majority of voting nodes is required to elect a primary.
- Failover causes a short write interruption.
- Drivers can discover topology changes but applications still see transient errors.
- Replica placement should consider zones, racks, and regions.

## Primary-secondary replication

The primary accepts writes and secondaries apply them asynchronously. This means secondaries can lag.

Read behavior:

- Primary reads give the freshest normal view.
- Secondary reads can reduce primary load but may be stale.
- Read preference must be chosen per endpoint, not globally.

## Elections and failover

If the primary fails or loses majority, eligible nodes elect a new primary. Elections prevent split brain by requiring majority.

Application impact:

- Some writes fail during election.
- In-flight operations can have ambiguous outcome.
- Retryable writes help, but business operations still need idempotency.
- Weak write concern can lose writes during failover.

## Replication lag

Lag is the delay between primary commit and secondary application.

Causes:

- Slow disks.
- Network saturation.
- Heavy write bursts.
- Large index builds.
- Secondary under-provisioning.
- Long-running operations.

Effects:

- Stale secondary reads.
- Longer recovery after failure.
- Smaller effective oplog window.
- Failover to a node missing recent writes if write concern is weak.

## Read and write concerns

**Write concern** controls acknowledgement level. `majority` means a majority of replica set members acknowledged the write.

**Read concern** controls what data is visible to reads. Stronger concerns can avoid reading data that may roll back, at latency cost.

Production choice:

- Critical state: primary reads and majority writes.
- Dashboards/reporting: secondary reads may be acceptable.
- Low-value telemetry: weaker settings may be fine if loss is acceptable.

## Fault tolerance and consistency implications

Replication increases fault tolerance but creates consistency choices. A system cannot use secondary reads everywhere and still promise read-your-writes. It cannot use weak writes and expect no rollback risk. The service contract must match database settings.

---

# Sharding

Sharding splits data across shards. Each shard owns part of the dataset. MongoDB commonly uses sharded clusters where each shard is itself a replica set.

## Why sharding exists

Sharding is used when one replica set cannot handle:

- Data volume.
- Write throughput.
- Working set size.
- Tenant isolation needs.
- Operational blast radius.

Sharding does not fix bad query design. A collection scan across ten shards is still a collection scan, now multiplied.

## Shard keys

The shard key decides where documents live. It must balance four concerns:

| Concern | Requirement |
|---|---|
| Cardinality | Enough distinct values to distribute data. |
| Write distribution | Inserts/updates should not hit one shard. |
| Query targeting | Common reads should include the shard key. |
| Stability | Values should not change frequently. |

Bad shard keys:

- `status`: low cardinality.
- `createdAt`: monotonic write hotspot.
- Field absent from common queries: scatter-gather reads.
- Tenant ID alone when one tenant is much larger than all others.

## Chunk balancing

MongoDB splits sharded data into chunks and migrates chunks between shards to balance distribution. Balancing consumes resources:

- Network bandwidth.
- Disk I/O.
- Metadata updates.
- Cache disruption.

Balancing is normal, but heavy migrations during peak traffic can affect latency.

## Hot shards

A hot shard receives disproportionate traffic. Causes:

- Monotonic shard key.
- Hot tenant.
- Hot product or account.
- Query pattern that targets one shard.
- Uneven data distribution.

Mitigation:

- Hash component for write distribution.
- Time or hash bucketing.
- Tenant tier isolation.
- Read caching for hot keys.
- Revisit data model before adding shards blindly.

## Distributed query costs

Queries with shard key can be routed to one shard. Queries without shard key may be broadcast to all shards and merged by the router.

Costs:

- Higher latency.
- More CPU across cluster.
- More network traffic.
- More complex sorting/limiting.
- Harder query predictability.

## Resharding difficulty

Changing shard key after a collection is large is expensive and operationally risky. Modern MongoDB supports resharding, but it still consumes resources and requires planning.

Before sharding, test expected data distribution and top query patterns. Shard key choice is a long-term architecture decision.

---

# Transactions and Consistency

NoSQL systems often optimize for single-record or partition-local operations because distributed transactions require coordination and reduce throughput.

## Single-document atomicity

MongoDB guarantees atomic updates within one document. This is the cheapest consistency boundary.

Use it for:

- Order status and embedded order lines.
- User preference updates.
- Inventory reservation if modeled as one document and contention is acceptable.
- Versioned state transitions with conditional update.

Conditional updates are often more scalable than transactions:

```javascript
db.orders.updateOne(
  { _id: orderId, status: "NEW" },
  { $set: { status: "PAID", paidAt: now } }
)
```

The filter enforces the state transition atomically.

## Multi-document transactions

MongoDB supports multi-document ACID transactions. They are useful when multiple documents in the same deployment must change together and the write set is small.

Costs:

- More coordination.
- More memory and transaction state.
- Longer lock/resource retention.
- Retry handling for transient transaction errors.
- Higher latency, especially in sharded clusters.

Use them deliberately, not as a default around every repository method.

## Distributed transaction costs

Cross-node or cross-service transactions are expensive because participants must coordinate commit/rollback. During failures, participants can be uncertain. In microservices, distributed transactions also couple service availability.

Production systems usually prefer:

- Outbox pattern.
- Kafka events.
- Sagas.
- Idempotent consumers.
- Compensating actions.
- Reconciliation jobs.

## Retry logic

Distributed operations can fail after partial success. Retrying safely requires:

- Idempotency key.
- Unique constraint or deduplication table/collection.
- Version checks.
- Conditional updates.
- Retry budget and backoff.
- Clear handling of ambiguous timeout.

Never retry payment, order creation, or external side effects without an idempotency design.

## Why NoSQL avoids distributed transactions

Avoiding distributed transactions improves latency, availability, and throughput. The cost is that the application must handle eventual consistency explicitly. This is acceptable when the business workflow can tolerate intermediate states and repair mechanisms.

---

# Redis

Redis is an in-memory data structure store. In backend systems it is usually used for derived, temporary, or coordination data where low latency matters more than relational querying.

## In-memory storage

Redis is fast because hot data is in memory and operations are simple. The constraint is memory cost and eviction behavior. Memory sizing must include object overhead, replication buffers, fragmentation, and peak load.

## Caching

Common pattern: cache-aside.

1. Service reads from Redis.
2. On miss, service reads MongoDB/SQL.
3. Service stores result in Redis with TTL.
4. Writes invalidate or update the cache.

Production requirements:

- Short Redis timeouts.
- Fallback behavior when Redis is down.
- TTL jitter to avoid synchronized expiry.
- Cache key versioning.
- Protection against caching invalid data.

## TTL

TTL is useful for sessions, tokens, temporary locks, and cached data. Expiry is not a business scheduler. Keys can expire before or after the exact moment depending on Redis internals and access patterns.

## Eviction

When Redis reaches memory limit, eviction policy decides whether to remove keys or reject writes. This is a correctness decision.

Examples:

- Cache: `allkeys-lru` or similar may be acceptable.
- Sessions: eviction may log users out unexpectedly.
- Rate limits: eviction weakens protection.
- Locks: eviction can break coordination.

## Distributed locks

Redis locks are commonly implemented with `SET key value NX PX ttl`. Correct usage requires:

- Unique random token per owner.
- Compare-and-delete release script.
- Short TTL.
- Handling lock expiry while work continues.
- Idempotent protected operation.

Use Redis locks for low-risk coordination. Do not rely on them as the only correctness mechanism for critical financial invariants.

## Pub/Sub

Redis Pub/Sub sends messages to currently connected subscribers. It is not durable and does not replay missed messages. Use it for live notifications where missed messages are acceptable. Use Kafka or Redis Streams when durability and consumer recovery matter.

## Persistence

Redis persistence options:

- **RDB**: point-in-time snapshots.
- **AOF**: append-only log, better durability with write overhead.
- **Both**: common when Redis data is important.

Persistence does not make Redis equivalent to a relational source of truth. Recovery time, data loss window, replication mode, and failover behavior must be understood.

## Cache invalidation

Strategies:

- TTL-only: simple, stale until expiry.
- Explicit invalidation on write: fresher, more coupling.
- Write-through: cache updated with DB write, more latency.
- Event-driven invalidation: Kafka event invalidates Redis asynchronously.
- Stale-while-revalidate: serve old data while one worker refreshes.

## Cache stampede

A stampede happens when many requests miss the same hot key and rebuild it simultaneously. It can overload MongoDB/SQL.

Controls:

- TTL jitter.
- Request coalescing.
- Short Redis lock around rebuild.
- Serve stale data briefly.
- Prewarm hot keys.
- Rate limit rebuilds.

---

# Cassandra

Cassandra is a distributed wide-column store optimized for high write throughput and availability with predictable query patterns.

## Wide-column storage

Data is organized by partition key and clustering columns. Rows in the same partition are stored together and sorted by clustering key.

Design starts from queries:

- Which partition is read?
- What range inside the partition is needed?
- How large can the partition become?
- What TTL/delete behavior will create tombstones?

## AP-oriented architecture

Cassandra can continue serving requests when some nodes are unavailable, depending on consistency level and replication factor. It is often used where availability and write throughput matter more than immediate global consistency.

AP-oriented does not mean correctness is ignored. It means the application chooses consistency level and handles staleness/conflict risks.

## Partition keys

The partition key controls distribution. A good key spreads writes and keeps partitions bounded.

Time-series example:

```text
partition key: device_id + day_bucket
clustering key: event_time
```

This avoids one endless partition per device and supports efficient device/day reads.

## Tunable consistency

Cassandra consistency levels include `ONE`, `QUORUM`, and `ALL`.

Tradeoff:

- `ONE`: lower latency and higher availability, higher stale-read risk.
- `QUORUM`: stronger consistency if read/write quorums overlap, higher latency.
- `ALL`: strongest acknowledgement, lowest availability.

## Time-series workloads

Cassandra fits append-heavy time-series workloads when queries are predictable and partitions are bounded. It is weak for ad-hoc analytics unless data is copied to analytical storage.

Production risks:

- Huge partitions.
- Hot partitions.
- Tombstone buildup from TTL/deletes.
- Compaction pressure.
- Query patterns not matching table design.

---

# Binary Data Storage

Binary data should usually be separated from operational document data.

## GridFS

GridFS stores files in MongoDB by splitting them into chunks and storing metadata. It can be useful when files must be managed inside MongoDB's security, backup, and replication model.

Costs:

- Database storage grows quickly.
- Backups and restores become heavier.
- File traffic competes with operational queries.
- CDN/object-storage features are not native.

## Object storage

Most systems store files in S3-compatible object storage and keep metadata in MongoDB.

Metadata document:

```json
{
  "_id": "img_123",
  "ownerId": "product_42",
  "bucket": "product-images",
  "objectKey": "products/42/main.jpg",
  "contentType": "image/jpeg",
  "size": 248113,
  "checksum": "sha256:...",
  "createdAt": "2026-05-16T10:00:00Z"
}
```

## Metadata patterns

Store in the database:

- Owner and authorization fields.
- Object key and bucket.
- Content type and size.
- Checksum.
- Processing status.
- Lifecycle timestamps.

Use async cleanup for orphaned objects when DB write fails after upload or object upload fails after metadata creation.

## Why images are usually outside DB

Images and videos are large, accessed differently, cached by CDN, and backed up differently. Keeping them in the primary database bloats working set and slows operational backup/restore. The database should usually store the reference and business metadata, not the bytes.

---

# NoSQL in Microservices

NoSQL in microservices is mostly about ownership and consistency boundaries.

## Database per service

Each service owns its data. Other services access it through APIs or events, not direct collection/table access. Direct database sharing creates hidden coupling and makes schema changes risky.

Example:

- `catalog-service`: MongoDB product documents.
- `order-service`: SQL or MongoDB order state.
- `session-service`: Redis.
- `analytics-service`: Cassandra or analytical storage.

## Polyglot persistence

Different services can use different databases when the workload justifies it. The cost is more operational knowledge, monitoring, backup strategy, and incident complexity.

Use polyglot persistence when one database would force bad modeling or bad runtime behavior.

## Eventual consistency between services

Cross-service workflows usually cannot rely on one local transaction. Services publish events and update their own stores asynchronously.

Production requirements:

- Source-of-truth definition.
- Idempotent event processing.
- Event ordering strategy.
- Versioning and schema evolution.
- Dead-letter/retry handling.
- Reconciliation jobs.

## Kafka integration

Kafka is commonly used to propagate changes from NoSQL-backed services.

Patterns:

- **Outbox**: write state change and outbox record in the same database transaction or atomic boundary, then publish to Kafka.
- **Read model**: consume events and build MongoDB documents optimized for reads.
- **Cache invalidation**: consume change events and delete/update Redis keys.
- **Audit/event stream**: preserve business events for replay.

## Idempotent consumers

Consumers must handle duplicate and out-of-order messages.

Techniques:

- Store processed event IDs.
- Use idempotency keys.
- Apply only if event version is newer.
- Use unique constraints for natural deduplication.
- Make updates deterministic.

## Cache-aside pattern

Cache-aside keeps Redis outside the write transaction:

- Read from cache.
- On miss, read source DB and populate cache.
- On write, update DB and invalidate cache.

Failure behavior must be explicit. If invalidation fails, TTL or event replay must eventually repair stale cache.

---

# Monitoring and Production Issues

NoSQL reliability depends on query behavior, data growth, topology, and client configuration. Monitoring must observe the database and the Java service together.

## Slow queries and query profiling

Slow queries usually come from missing indexes, bad sort, low selectivity, unbounded result size, or inefficient aggregation.

Track:

- p95/p99 query latency.
- Documents examined vs returned.
- Full collection scans.
- Sort spills.
- Slow aggregation pipelines.
- Endpoint-to-query correlation.

## Missing indexes

Missing indexes are common because development datasets are small. Production review should require an index plan for every new filter/sort endpoint.

Symptoms:

- CPU spike after feature release.
- Latency grows with collection size.
- High documents examined.
- Disk I/O increases.

## OOM issues

OOM can occur in Redis, MongoDB, or Java services.

Causes:

- Redis keys exceed memory and eviction is unsafe.
- MongoDB working set/indexes exceed memory, causing disk thrashing.
- Java service loads large query results into heap.
- Aggregations produce large intermediate results.
- Connection pools and request concurrency are unbounded.

## Replication lag

Monitor lag continuously. It affects secondary reads, failover safety, and restore options.

Alert when lag exceeds business freshness tolerance, not only when it is catastrophic.

## Hot partitions and hot keys

Hot partitions appear in sharded MongoDB, Cassandra, Redis Cluster, and Kafka-like systems. One key or partition receives too much load.

Mitigation depends on workload:

- Key bucketing.
- Hash suffixes.
- Tenant isolation.
- Local caching.
- Async aggregation.
- Data model redesign.

## Disk pressure

Disk problems cause database-wide instability:

- Journal/oplog growth.
- Index growth.
- Compaction needs.
- Backup snapshots.
- Temporary aggregation files.
- Log files.

Disk alerts should include growth rate and time-to-full, not only percentage used.

## Backup and restore

Backups are not complete until restore has been tested.

Production plan:

- Define RPO and RTO.
- Automate backups.
- Store backups separately from primary failure domain.
- Test restore regularly.
- Document restore process.
- Validate application-level integrity after restore.

## Connection pool exhaustion

Java services can overload databases through too many concurrent operations.

Controls:

- Bounded connection pools.
- Request timeouts.
- Circuit breakers.
- Bulkheads per dependency.
- Backpressure on expensive endpoints.
- Retry budgets with jitter.

A database outage often becomes a service outage because threads block waiting for connections. Pool metrics are production signals.

---

# CAP and PACELC

CAP and PACELC are useful when tied to concrete failure behavior and latency decisions.

## CP vs AP systems

**CP systems** preserve consistency during partitions by refusing some operations. They are suitable for coordination, metadata, and correctness-critical state.

**AP systems** preserve availability during partitions by allowing operations that may be stale or conflicting and resolving later. They are suitable when continuous operation is more important than immediate global agreement.

Many databases are configurable; behavior depends on read/write settings, topology, and client choices.

## Why CA is unrealistic

A distributed system cannot assume the network always works. If a partition occurs, the system cannot both answer every reachable request and guarantee every answer reflects the latest global state. CA is realistic only when there is no partition to tolerate, such as a single-node system.

## MongoDB CAP behavior

MongoDB replica sets with primary elections and majority write concern are CP-leaning for writes. If a primary cannot reach majority, it should not continue accepting writes as primary. This protects consistency and prevents split brain.

Client settings can weaken or change observed behavior:

- Secondary reads can be stale.
- Weak write concern can increase rollback/loss risk.
- Majority reads/writes add latency.

## Cassandra CAP behavior

Cassandra is commonly AP-oriented with tunable consistency. It can keep accepting operations on available replicas depending on consistency level, then converge through replication and repair mechanisms.

Using `QUORUM` moves behavior toward stronger consistency at the cost of latency and availability. Using `ONE` favors latency and availability but accepts more stale-read risk.

## Redis tradeoffs

Redis standalone is not a distributed CAP system. Redis replication, Sentinel, or Cluster introduce distributed tradeoffs:

- Replication is often asynchronous.
- Failover can lose recent writes.
- Cluster partitions can make some hash slots unavailable.
- Caches may serve stale data by design.

Redis is often used where latency matters and data is derived or temporary. If it stores critical state, persistence and failover semantics must be explicitly accepted.

## PACELC theorem

PACELC extends CAP:

- If there is a **Partition**, choose **Availability** or **Consistency**.
- Else, choose **Latency** or **Consistency**.

This matters because most tradeoffs happen during normal operation. Majority writes, quorum reads, cross-region coordination, and secondary reads all trade latency against consistency even when nothing is failing.

Production examples:

- MongoDB majority write concern: safer failover, higher latency.
- Cassandra `QUORUM`: stronger freshness, slower than `ONE`.
- Redis cache read: very low latency, possible staleness.
- Reading from nearest replica: lower latency, weaker freshness.

---

# Interview Q&A

## 143. What are NoSQL databases and how do they differ from relational databases?

NoSQL databases are storage systems designed around non-relational access patterns: documents, key-value lookups, wide-column partitions, or graph traversals. The important production difference is not syntax; it is the operational contract.

Relational databases give strong schema, joins, constraints, and transactions. NoSQL systems usually trade some of that for flexible structure, partition-friendly access, lower-latency lookups, or high availability across nodes.

In real backend systems:

- SQL fits payments, ledgers, and data with strict relational constraints.
- MongoDB fits document-shaped aggregates such as catalogs and profiles.
- Redis fits cache, sessions, counters, and short-lived state.
- Cassandra fits high-volume partitioned writes such as events or metrics.

The tradeoff is that NoSQL often shifts responsibility into application design: validation, denormalization, idempotency, conflict handling, and operational monitoring.

## 144. What are the main types of NoSQL databases? (Document, key-value, graph, columnar)

The main types are document, key-value, wide-column, and graph databases.

Document databases such as MongoDB store nested documents and work well when the service usually reads an aggregate as one object. Key-value databases such as Redis optimize direct lookup by key and are excellent for cache, sessions, counters, and TTL-based data. Wide-column databases such as Cassandra store partitioned rows and fit massive write-heavy workloads with predictable queries. Graph databases optimize relationship traversal, such as fraud networks or social graph paths.

The architectural risk is choosing a database whose query model does not match the product. If the application needs ad-hoc filters, a pure key-value store forces manual indexes. If the application needs graph traversal, MongoDB references become expensive. If the application needs joins and constraints, SQL may be the better tool.

## 145. In what cases is it better to use NoSQL instead of SQL?

Use NoSQL when the workload benefits from a specialized data model or distributed behavior that SQL would handle awkwardly or expensively.

Good production cases:

- Product catalog with variable attributes by category.
- Read-heavy document aggregates where joins would be on every request.
- Cache/session/rate-limit data with TTL.
- High-volume append events partitioned by key and time.
- Event-driven read models updated from Kafka.
- Systems that can tolerate controlled eventual consistency.

SQL is usually better when the core value is relational integrity, complex joins, strict constraints, ad-hoc reporting, or multi-row transactions. The decision is not SQL versus NoSQL globally; a production architecture often uses both.

## 146. What popular NoSQL databases do you know? (MongoDB, Cassandra, Redis, etc.)

MongoDB is a document database used for flexible JSON-like aggregates and read models. Redis is an in-memory key-value/data-structure store used for cache, sessions, locks, counters, and rate limiting. Cassandra is a wide-column distributed database used for high-throughput writes and AP-style availability. DynamoDB is a managed key-value/document database with partition-based scaling. Neo4j is a graph database for traversal-heavy domains. Elasticsearch/OpenSearch is commonly used for search-oriented read models, not as a primary transactional database.

The useful answer connects each database to workload. Listing names without access patterns does not show engineering judgment.

## 147. What is a collection in MongoDB and how does it differ from a table in SQL?

A MongoDB collection groups BSON documents. A SQL table stores rows with a fixed schema, columns, constraints, and relational structure. A collection allows documents with different fields, nested objects, and arrays.

Production implication: MongoDB flexibility helps when product data evolves, but it does not remove schema responsibility. Java services still need DTO compatibility, validation, versioning, and migration strategy. Indexes are defined on collection fields, and inconsistent document shapes can break queries or cause missing-field behavior.

A `products` collection can contain different attributes for phones and books. A SQL design might need subtype tables, EAV modeling, or JSON columns. MongoDB makes the read model simpler if the product page consumes the document as an aggregate.

## 148. What is a document in MongoDB?

A document is MongoDB's record unit, stored as BSON. It can contain nested objects, arrays, typed values, dates, decimals, binary fields, and an `_id`.

In production, a document should represent a bounded aggregate, not an infinite container. An order document with order lines is reasonable. A user document containing every login event forever is not. Document size affects network cost, memory, serialization, update cost, and index maintenance.

Good document modeling uses the request pattern: store together what is read together and updated together, but split data that grows independently or has separate query patterns.

## 149. How are relationships between documents defined in MongoDB? Is there JOIN?

Relationships are modeled with embedding or references. Embedding stores related data inside the parent document. References store another document's ID and resolve it with another query, application logic, or aggregation `$lookup`.

MongoDB has join-like `$lookup`, but production MongoDB design avoids join-heavy hot paths. If the application always needs child data with the parent and the child set is bounded, embedding reduces round trips and uses single-document atomicity. If the child data is large, shared, frequently updated, or independently queried, references are safer.

Example: embed order items in an order, reference customer ID, and copy product price/name as a checkout snapshot. This avoids changing historical orders when product catalog data changes.

## 150. How is an index implemented in MongoDB and why is it needed?

MongoDB indexes are B-tree-like structures that map field values to documents and keep entries ordered. They allow targeted lookup, range scans, and sorted reads without scanning the whole collection.

In production, indexes are required for predictable latency. A query that scans 1,000 documents in development may scan 100 million in production. Compound indexes should match real filters and sort order, for example `{ tenantId: 1, status: 1, createdAt: -1 }` for a tenant-scoped order list.

The tradeoff is write cost. Every insert, update, and delete must maintain indexes. Too many indexes increase disk, memory, and write latency. Engineers validate with `explain()` and monitor documents examined, sort stages, and index usage.

## 151. What are sharding and replication in NoSQL?

Replication copies the same data to multiple nodes for durability and failover. Sharding splits different data across nodes for capacity and throughput.

In MongoDB, a replica set provides primary-secondary replication and elections. A sharded cluster distributes a collection across shards using a shard key, and each shard is usually a replica set.

Replication helps survive node failure. Sharding helps when one replica set cannot handle data size or write/read load. They solve different problems and are commonly combined.

## 152. How does replication differ from sharding?

Replication duplicates data; sharding partitions data.

If a collection has 1 TB of data, a three-node replica set stores roughly the same 1 TB on each replica. That improves availability but not per-node data size. If the collection is sharded across four shards, each shard owns part of the data. If each shard is replicated, each partition also has failover protection.

Operationally, replication introduces lag, elections, read preference, and write concern. Sharding introduces shard keys, balancing, hot shards, scatter-gather queries, and resharding complexity.

## 153. What is eventual consistency

Eventual consistency means that after a write, not every replica or derived system must show the update immediately, but they should converge if no new conflicting updates happen and propagation succeeds.

This appears constantly in microservices. A product update may commit to MongoDB first, publish to Kafka next, invalidate Redis after that, and update search later. During that window, different clients can see different versions.

Production use requires boundaries: define which workflows tolerate stale data, how long staleness is acceptable, how duplicates are handled, how out-of-order events are rejected, and how reconciliation repairs missed updates. Eventual consistency without these controls becomes data drift.

## 154. How to store binary data (e.g., images) in MongoDB?

Usually store images in object storage and store metadata in MongoDB. MongoDB can store binary data, and GridFS can split large files into chunks, but putting large binaries directly into operational documents bloats the database and hurts backups, working set, and query performance.

A production pattern is:

- Upload file to object storage.
- Store object key, owner, content type, size, checksum, and status in MongoDB.
- Serve through CDN or signed URLs.
- Run cleanup for orphaned objects.

Use GridFS when files must be inside MongoDB's replication/security/backup model and the operational cost is acceptable.

## 155. What are TTL indexes in MongoDB?

TTL indexes automatically delete documents after a time based on a date field. They are useful for sessions, reset tokens, verification codes, temporary imports, and short-lived records.

Example:

```javascript
db.passwordResetTokens.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 })
```

Production implication: TTL cleanup is asynchronous and not exact. It should not be used as a precise scheduler or as the only enforcement for security-sensitive expiration. Application logic should still check expiration during use.

## 156. Can transactions be used in MongoDB? If so, how?

Yes. MongoDB has atomic single-document updates and supports multi-document transactions. The preferred production approach is to model invariants inside one document when possible, then use single-document atomic updates or conditional updates.

Multi-document transactions are used when several documents in the same MongoDB deployment must change atomically and the write set is small. They add coordination, latency, resource usage, and retry complexity. In sharded clusters the cost is higher.

Across microservices, MongoDB transactions are not the usual answer. Use outbox, Kafka events, sagas, idempotent consumers, and reconciliation instead of trying to create a distributed transaction across service databases.

## 157. How is fault tolerance ensured in NoSQL databases?

Fault tolerance comes from replication, leader election or quorum protocols, data partitioning, failover, client retry handling, backups, monitoring, and deployment across failure domains.

MongoDB replica sets elect a new primary when the old one fails, but writes can fail during election and weak write concern can create rollback risk. Cassandra replicates partitions across nodes and uses tunable consistency so it can continue operating with some node failures. Redis Sentinel/Cluster can fail over, but asynchronous replication can lose recent writes.

The application is part of fault tolerance: timeouts, bounded retries, idempotency, circuit breakers, and connection pool limits prevent partial database failure from becoming a full service outage.

## 158. What is the CAP theorem? What is the extension of the CAP theorem?

CAP says that during a network partition a distributed system must choose between consistency and availability while tolerating the partition. The useful extension is PACELC: if there is a partition, choose availability or consistency; else, choose latency or consistency.

Production meaning: even when the system is healthy, stronger consistency often requires coordination and therefore higher latency. Majority writes, quorum reads, cross-region reads, and secondary reads are all consistency/latency decisions.

Use CAP/PACELC to explain behavior under failure and normal operation, not to memorize database labels.

## 159. Why is it impossible to ensure all three properties simultaneously?

During a partition, nodes cannot coordinate. If one side accepts a write and the other side continues answering reads, the second side may return stale data. To preserve consistency, the system must reject or delay some requests. To preserve availability, it must answer despite not knowing the latest global state.

That is why all three cannot be guaranteed during partition. Partition tolerance is required in a real distributed system, so the real choice is consistency versus availability under failure.

Example: a MongoDB node that cannot reach majority should not continue as primary because another side may elect a new primary. Refusing writes reduces availability but prevents split brain.

## 160. Give examples of CP and AP databases.

CP-style systems prioritize consistency during partitions: ZooKeeper, etcd, Consul, and MongoDB replica sets when using majority-oriented primary writes are common examples.

AP-style systems prioritize availability and convergence: Cassandra and Dynamo-style systems are common examples, depending on consistency settings.

The important nuance is configuration. Cassandra with `QUORUM` behaves differently from Cassandra with `ONE`. MongoDB with primary reads and majority writes behaves differently from MongoDB with secondary reads and weak write concern. CP/AP is a failure-mode description, not a permanent marketing label.

## 161. How does the CAP theorem manifest in MongoDB?

MongoDB replica sets elect one primary through majority voting. With majority write concern, MongoDB favors consistency for writes: if a node cannot reach majority, it should not safely accept primary writes. During a partition or election, writes may be temporarily unavailable.

Reads depend on client settings. Primary reads give fresher data. Secondary reads can return stale data because replication is asynchronous. Weak write concern can increase rollback risk during failover. Majority read/write concerns improve safety but add latency.

In production, choose settings per workflow. Checkout or account state should use primary/majority-oriented behavior. Dashboards or non-critical reads may use secondary reads if staleness is acceptable.
