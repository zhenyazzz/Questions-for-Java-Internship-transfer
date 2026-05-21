# Message Brokers, Kafka, and RabbitMQ — Backend Engineering Notes

### Reliability Boundary

Messaging is a reliability boundary, not a reliability guarantee. A broker removes synchronous availability coupling between producer and consumer, then replaces it with durable state, offset/ack protocols, retries, replay, ordering scopes, schema contracts, and operational queues of unfinished work. The hard parts move from request latency to delayed failure, duplicate side effects, stale projections, lag, poison messages, and incident ownership.

### Production Failure Points

A production message path has at least four independently failing steps: the producer's local state change, the broker append/enqueue, consumer side effects, and consumer progress acknowledgement. No ordinary broker makes those four steps one atomic transaction across databases, APIs, email providers, and payment gateways. Timeouts are ambiguous. A send may have been accepted before the client lost the response. A consumer may have updated the database and crashed before acking. A rebalance may revoke a partition while work is still running. At-least-once systems therefore converge toward idempotency keys, transactional outbox, bounded retries, dead-letter isolation, schema compatibility, and observability around backlog age rather than just message count.

## Table of Contents

- [Broker Semantics at Runtime](#broker-semantics-at-runtime)
- [Kafka Runtime Model](#kafka-runtime-model)
- [Producers, Replication, ISR, and Leader Election](#producers-replication-isr-and-leader-election)
- [Consumer Groups and Rebalancing](#consumer-groups-and-rebalancing)
- [Offset and Ack Timing](#offset-and-ack-timing)
- [Retention, Replay, and Compaction](#retention-replay-and-compaction)
- [RabbitMQ Runtime Model](#rabbitmq-runtime-model)
- [RabbitMQ DLQ, Retries, and Poison Messages](#rabbitmq-dlq-retries-and-poison-messages)
- [Kafka vs RabbitMQ in System Design](#kafka-vs-rabbitmq-in-system-design)
- [Event-Driven Consistency and the Outbox Boundary](#event-driven-consistency-and-the-outbox-boundary)
- [Idempotency, Duplicates, and Exactly-Once Misconceptions](#idempotency-duplicates-and-exactly-once-misconceptions)
- [Retry Architecture and Failure Containment](#retry-architecture-and-failure-containment)
- [Observability and Operations](#observability-and-operations)
- [Common Failure Patterns](#common-failure-patterns)
- [Message Brokers Interview Q&A](#message-brokers-interview-qa)

---

## Broker Semantics at Runtime

### Messages vs Function Calls

Messages are not function calls. They are durable or semi-durable records whose processing is decoupled from the producer's request lifecycle. That decoupling creates hidden queues between every stage: producer buffer, broker page cache/disk, replication pipeline, consumer fetch buffer, application thread pool, downstream DB pool, retry topic/queue, and DLQ. Under load, one of those queues becomes the real system boundary.

### Delivery Guarantees

### Delivery Guarantees

Delivery guarantees define what can happen to a message during failures, retries, consumer crashes, broker outages, and offset/acknowledgement timing.

The core problem is that message processing and progress acknowledgement are separate operations and are not atomic by default.

Typical flow:

1. Consumer receives a message.
2. Consumer processes business logic.
3. Consumer commits offset or sends acknowledgement.

If failure happens between these steps, the system may lose or duplicate work depending on commit timing and retry behavior.

---

### At-most-once

Message is considered processed before business processing completes.

Typical flow:

1. Consumer receives message.
2. Consumer commits offset or acknowledges immediately.
3. Business logic executes.

If the consumer crashes after commit but before processing finishes, the message is lost permanently.

Configuration characteristics:

- offset commit happens before processing;
- auto-commit is often enabled;
- retries are usually disabled or limited.

Guarantees:

- no duplicate delivery;
- messages may be lost.

Tradeoff:

- lowest latency and simplest flow;
- unsafe for critical business operations.

Typical usage:

- metrics;
- analytics;
- non-critical logs.

---

### At-least-once

Message is acknowledged only after successful processing.

Typical flow:

1. Consumer receives message.
2. Business logic executes.
3. Offset commit or acknowledgement happens after success.

If the consumer crashes after processing but before commit, the broker redelivers the message.

Configuration characteristics:

- manual acknowledgements or manual offset commits;
- retries enabled;
- commit after successful processing.

Guarantees:

- messages should not be lost;
- duplicate delivery is possible and expected.

Tradeoff:

- safest common production model;
- requires idempotent consumers.

Typical production setup:

- retry topics or retry queues;
- DLQ after max attempts;
- idempotency keys;
- unique constraints;
- processed-event tracking.

Typical usage:

- payments;
- orders;
- inventory updates;
- email/event processing.

---

### Exactly-once

Exactly-once means the system attempts to prevent duplicate processing effects within a limited scope.

In Kafka, this is achieved through:

- idempotent producers;
- Kafka transactions;
- atomic offset commit with produced records.

Typical flow:

1. Consumer reads records.
2. Processing happens.
3. Produced Kafka records and offset commits happen in one transaction.

Configuration characteristics:

- `enable.idempotence=true`;
- transactional producer enabled;
- transactional.id configured;
- read_committed isolation for consumers.

Guarantees:

- Kafka-to-Kafka processing can avoid duplicates;
- offsets and produced records are committed atomically.

Important limitation:
Exactly-once does NOT automatically protect:

- database writes;
- REST calls;
- payment providers;
- emails;
- external side effects.

If the service writes to a database and then crashes before transaction coordination completes, duplicate business effects may still happen.

Because of this, business-level exactly-once usually still requires:

- idempotency keys;
- unique business operation IDs;
- transactional outbox;
- deduplication logic.

---

### Producer-side retries

Producers may retry when:

- timeout occurs;
- acknowledgement is delayed;
- network connection fails.

The broker may already have stored the message before the retry occurs.

Without idempotent producers, retries can create duplicate records.

Kafka reliability settings:

- `acks=0` → fastest, weakest durability;
- `acks=1` → leader acknowledges;
- `acks=all` → all in-sync replicas acknowledge.

Higher durability usually increases latency.

---

### Consumer-side retries

Consumers retry when:

- business processing fails;
- downstream dependency is unavailable;
- database/network timeout occurs.

Common production patterns:

- immediate retry;
- exponential backoff;
- retry topics/queues;
- DLQ after max attempts.

Bad retry design causes:

- retry storms;
- duplicated side effects;
- partition blocking;
- queue backlog growth.

---

### Offset Commit Timing

Offset timing determines delivery semantics.

Commit before processing:

- at-most-once;
- possible message loss.

Commit after processing:

- at-least-once;
- possible duplicate processing.

This is one of the most important concepts in distributed messaging systems.

---

### Real Production Mental Model

Exactly-once is not a property of a business workflow unless:

- every side effect participates in the same atomic transaction;
  or
- all side effects are idempotent.

In real distributed systems, at-least-once + idempotency is the dominant production approach because true distributed atomicity across brokers, databases, and external APIs is usually impractical.

### Backpressure

Backpressure is also broker-specific. Kafka consumers pull and can slow polling, but lag grows in partitions and retention starts becoming a recovery clock. RabbitMQ pushes deliveries up to prefetch limits; an oversized prefetch moves backlog from broker-visible queues into consumer memory and unacked state, which can create unfair dispatch, long redelivery delays, and misleading queue depth.

### Ordering Scope

Ordering is always scoped. Global ordering is usually incompatible with horizontal scaling because a single ordered stream has one effective executor. Production systems normally choose an aggregate-level order key, accept cross-aggregate reordering, and make consumers robust to stale or duplicate state transitions. If a handler assumes event `N+1` never arrives before event `N`, that assumption must be encoded in partitioning, version checks, buffering, or reconciliation.

## Kafka Runtime Model

### Topics, Partitions, Producers, and Consumers

Kafka stores records in append-only partition logs. A topic is an operational namespace; partitions are the actual units of ordering, parallelism, replication, leadership, lag, and hot-spot failure. Producers append to the leader replica of the selected partition. Consumers fetch from partitions and store progress as offsets. Consumption does not delete records; retention does.

### Partition Order and Keys

A partition is a single ordered log. Within it, offsets monotonically increase and records are read in offset order. Across partitions there is no total order. If `orderId` is the key, all events for that order usually map to the same partition and preserve per-order order. If the key is absent, random, low-cardinality, or changed over time, ordering and load distribution change accordingly. Low-cardinality keys such as `country`, `tenantType`, or `status` often create hot partitions where one broker and one consumer instance carry disproportionate traffic while the rest of the group appears underused.

### Partition Count

Partition count is a capacity decision with long-term consequences. Too few partitions cap consumer parallelism and concentrate traffic. Too many partitions increase file handles, metadata, leader election work, memory overhead, recovery time, controller load, and rebalance cost. Increasing partitions can also change key-to-partition mapping for future records, which can break naive ordering assumptions if old and new events for the same key are not routed consistently by the partitioner strategy.

### Throughput Shape

Kafka throughput comes from batching, sequential append, compression, page cache, partition parallelism, and efficient fetches. Small synchronous sends with no batching produce poor throughput and high broker request overhead. Large messages reduce batching efficiency, increase memory pressure, slow replication, and make retries expensive. Mature designs keep events compact, store large payloads in object storage or a database, and publish references plus immutable metadata.

## Producers, Replication, ISR, and Leader Election

### Producer Acknowledgements

A producer send succeeds only according to its acknowledgement configuration, not according to business durability. With `acks=0`, the client does not know whether the broker received the record. With `acks=1`, the partition leader accepted the append, but followers may not have replicated it. With `acks=all`, success requires acknowledgement from the configured in-sync replica set, constrained by `min.insync.replicas`. Stronger settings reduce loss windows but can turn broker degradation into producer unavailability when the ISR shrinks below the minimum.

### Replication, ISR, and Leader Election

Each partition has one leader and zero or more followers. Followers fetch from the leader and remain in the ISR while sufficiently caught up. If the leader fails, Kafka elects a new leader from eligible replicas. With unclean leader election disabled, Kafka prefers availability loss over acknowledged data loss when no in-sync replica is available. With unsafe election, the cluster may recover writes faster but can truncate acknowledged records. Operationally, under-replicated partitions and ISR churn are not cosmetic metrics; they mean durability margin is shrinking and producer `acks=all` may soon start failing.

### Producer Retries and Transactions

Producer retries handle ambiguous send failures. If the client times out after the broker appended the record, retrying can duplicate unless the idempotent producer is enabled and sequencing constraints are preserved. Idempotent producers deduplicate producer-session sequence retries per partition; they do not deduplicate two application attempts with different producer sessions, and they do not make downstream consumers exactly-once. Transactional producers extend atomicity to Kafka writes and consumed offset commits for Kafka-to-Kafka pipelines, but they do not automatically include an external database, REST API, email provider, or payment processor.

## Consumer Groups and Rebalancing

### Partition Ownership

A consumer group coordinates partition ownership. At any instant, one partition is assigned to at most one active consumer in a group. Additional consumers beyond the subscribed partition count are idle for that topic. Different groups have independent offsets and each receives the stream independently, which is how the same event can feed inventory, billing, analytics, and search without producer changes.

### Rebalancing

Rebalancing is distributed coordination in the middle of application processing. It happens when members join, leave, miss heartbeats, exceed poll intervals, change subscriptions, or when topic metadata changes. Eager rebalancing revokes broad ownership and can pause the group. Cooperative rebalancing reduces disruption by incrementally moving partitions, but handlers still need to handle revocation, in-flight work, and commits carefully.

### Slow Consumers

Slow consumers cause coordination failures even when the process is alive. If processing blocks the poll loop long enough to exceed `max.poll.interval.ms`, the group may consider the member unhealthy and revoke partitions. The old consumer may still be executing side effects while a new owner starts reading the same partition. This is a common source of duplicate writes and out-of-order external effects. Separate polling from processing only if offset commits, partition pause/resume, bounded work queues, and shutdown/revocation hooks are designed deliberately.

### Lag

Lag is per group/topic/partition: latest broker offset minus committed or current consumed offset, depending on the metric. Lag count is not enough; lag age and business SLA matter more. Ten million analytics records may be acceptable if catch-up rate is high. A hundred payment-confirmation records may be an incident if they are old. Lag grows when production rate exceeds effective processing rate, when one partition is hot, when a downstream dependency is slow, when retries block progress, or when rebalances repeatedly reset work.

## Offset and Ack Timing

### Offsets as Progress Markers

Offsets are consumer progress markers, not record identities. Auto-commit often hides dangerous timing: offsets may be committed for records fetched into memory but not durably processed. Manual commits make the timing explicit, but correctness still depends on side-effect ordering.

### At-Most-Once and At-Least-Once Timing

For at-most-once, commit before processing. This avoids duplicates but loses records on failure. For at-least-once, process first and commit after durable success. This preserves retryability but duplicates when the process crashes after side effects and before commit, when commit fails, or when a rebalance races with in-flight work. For batch processing, committing the highest offset implies all lower offsets in that partition are complete; parallel processing within a partition must track gaps or avoid committing past unfinished records.

### Database Side Effects

A consumer that writes to a database should usually make the database write idempotent and commit the offset only after the database transaction commits. If the database commit succeeds and offset commit fails, replay occurs and the database constraint or idempotency table must absorb it. If offset commit succeeds and the database commit later fails, the event is lost for that consumer group. That is the core timing problem behind most messaging incidents.

## Retention, Replay, and Compaction

### Retention

Kafka retention is a recovery contract. Records remain until time/size retention deletes them or compaction removes superseded keyed values. Consumers can replay only what still exists. If a service is down longer than retention, it may resume from an offset whose record has been deleted and require a full rebuild from another source. Retention must be set from recovery objectives, not just disk cost.

### Replay

Replay is operationally dangerous because it re-executes history against today's code, schemas, dependencies, and side-effect handlers. Rebuilding a projection is safe only if the handler is pure with respect to external effects or can run in a replay mode that disables emails, payments, webhooks, and irreversible commands. Replaying into a consumer that was written only for live traffic can duplicate notifications, overload databases, violate rate limits, or resurrect obsolete state.

### Compaction

Log compaction keeps the latest value per key eventually, not immediately, and tombstones have retention behavior of their own. Compacted topics are useful for reconstructing latest state, not for audit trails that require every transition. Consumers of compacted topics must tolerate missing intermediate states and records arriving with old schema versions.

## RabbitMQ Runtime Model

### Exchanges, Bindings, and Queues

RabbitMQ is routing- and queue-oriented. Producers publish to exchanges. Exchanges route to queues through bindings. Queues hold messages until consumers acknowledge them or until broker policy expires, dead-letters, or drops them. The central operational object is the queue, not a replayable partition log.

### Exchange Topology

Exchange behavior matters because routing is topology, not application code. A direct exchange routes by exact key, a topic exchange by pattern, a fanout exchange to all bound queues, and a headers exchange by header predicates. A publish to an exchange with no matching binding can be silently unroutable unless mandatory publishing, returns, confirms, or topology tests are used. Production RabbitMQ systems treat exchanges, bindings, queue arguments, durability flags, DLX settings, TTLs, and quorum/classic choices as deployable infrastructure, not incidental startup declarations.

### Push Delivery and Prefetch

RabbitMQ delivery is usually push-based. The broker sends messages to consumers up to the channel's prefetch limit. Prefetch is the main consumer-side backpressure knob: too low underutilizes workers; too high creates large unacked inventories, unfair dispatch to slow consumers, high memory use, and slow recovery after a worker dies. For heterogeneous task durations, large prefetch values cause head-of-line effects because long tasks occupy delivery slots while faster workers may starve.

### Acknowledgement Timing

Acknowledgement timing determines redelivery. With manual ack, a worker should ack only after durable completion. If it dies before ack, the broker requeues or dead-letters according to behavior and policy. Negative acknowledgements and rejects can requeue immediately; immediate requeue of deterministic failures creates tight poison-message loops. Auto-ack is effectively fire-and-forget from the broker's perspective and should not be used for work that must survive consumer crashes.

### Queue Depth

Queues are not free infinite buffers. Deep queues increase memory/disk pressure, paging, recovery time, mirror/quorum replication load, and management overhead. RabbitMQ can apply memory and disk alarms that block publishers. Long-lived backlogs in RabbitMQ often mean the queue is being used as a log; Kafka is usually a better fit when independent consumers, long retention, or replay are core requirements.

## RabbitMQ DLQ, Retries, and Poison Messages

### Dead-Lettering

Dead-lettering moves messages when they are rejected, expire, exceed delivery limits, or cannot be routed under configured policy. A DLQ is not a retry strategy by itself; it is an isolation lane for messages the normal path cannot process. If nobody owns DLQ inspection, replay tooling, and discard policy, the DLQ becomes silent data loss with better branding.

### Delayed Retries

RabbitMQ delayed retries are often implemented with TTL queues plus dead-letter exchanges, delayed-message plugins, or application scheduling. The dangerous pattern is `nack(requeue=true)` on every failure: it hot-loops the same dependency, preserves the poison message at the front of work, burns CPU/network, and can starve valid messages. Production retry flows add attempt counts, exponential or tiered delays, jitter, maximum attempts, error classification, and final DLQ isolation.

### RabbitMQ Ordering

Ordering in RabbitMQ is fragile once multiple consumers, retries, priorities, redeliveries, or dead-letter paths enter the system. A single FIFO queue with one consumer can process in order, but that is also a single-threaded bottleneck. If business ordering matters, shard by entity into separate queues or serialize at the application/state-machine level; do not assume queue order survives realistic failure handling.

## Kafka vs RabbitMQ in System Design

### Kafka Fit

Kafka behaves like a replicated, partitioned event log with independent consumer groups and retention. It fits domain event streams, CDC, analytics pipelines, audit-style ingestion, stream processing, and rebuilding read models when consumers can tolerate partition-scoped ordering and replay semantics. Scaling is mostly partition-driven; operational risk concentrates around keys, lag, retention, replication, and rebalances.

### RabbitMQ Fit

RabbitMQ behaves like a routing broker and work dispatcher. It fits commands, task queues, low-latency job distribution, complex routing topologies, request/reply, and workflows where each message should be consumed by one worker and then leave the queue. Scaling is mostly queue/consumer/prefetch-driven; operational risk concentrates around unacked deliveries, queue depth, poison loops, DLX policies, memory alarms, and broker-side push pressure.

### Choosing by Failure Mode

Choosing between them is less about brand and more about the failure mode wanted during consumer outage. Kafka retains the log and allows consumers to resume or replay within retention, but lag can become huge and replay can be hazardous. RabbitMQ accumulates queue depth for the target queue and removes messages after ack; it is direct for work dispatch but not a natural multi-subscriber event history. Many systems use both: Kafka for durable facts and fan-out, RabbitMQ or a job runner for command execution and task scheduling.

## Event-Driven Consistency and the Outbox Boundary

### Dual-Write Problem

The dual-write problem is the normal failure at the edge between a service database and a broker. If the service writes the database then publishes to Kafka, it can crash after commit and before publish. If it publishes first then the database transaction rolls back, consumers observe a fact that never became true. Retrying the HTTP request can make both cases worse unless operation IDs and database constraints define exactly what was already accepted.

### Transactional Outbox

The transactional outbox pattern narrows the atomic boundary to one local database transaction: write business state and an outbox row together, then publish outbox rows asynchronously. The publisher marks rows sent only after broker acknowledgement, and duplicates are still possible if marking fails after publish. Therefore event IDs, aggregate versions, and consumer idempotency remain required. CDC-based outbox publishing removes polling but not the need for schema ownership, deduplication, and replay discipline.

### Event Schemas

Event schemas become operational contracts because producers and consumers deploy independently. Removing a field, changing meaning, narrowing enum values, altering key semantics, or reusing event names breaks consumers at runtime. Mature systems version schemas, enforce backward/forward compatibility in CI, keep fields additive where possible, document semantic ownership, and treat an event as an immutable fact rather than a mutable DTO shaped for one consumer.

### Async Workflow States

Async workflows fail later than the user request. A frontend may see “order created” while inventory reservation, payment capture, invoice generation, and search indexing are still pending. The product model must expose pending/failed states, reconciliation jobs must repair drift, and support tooling must show where the flow stopped. Without this, eventual consistency becomes invisible inconsistency.

## Idempotency, Duplicates, and Exactly-Once Misconceptions

### Duplicate Sources

Duplicates are unavoidable because acknowledgement, offset commit, broker append, database commit, network response, and process lifetime are not one indivisible action. Even if one broker feature suppresses one duplicate source, retries, failover, manual replay, blue/green deployments, producer restarts, DLQ reprocessing, and upstream bugs remain.

### Consumer Idempotency

Consumer idempotency should be implemented at the side-effect boundary, not just in memory. Common mechanisms are unique constraints on event IDs or business operation IDs, processed-message tables, aggregate version checks, idempotent upserts, state-machine transitions that ignore stale events, and external API idempotency keys. In-memory deduplication disappears on restart and fails across replicas.

### Kafka Exactly-Once Semantics

Kafka exactly-once semantics are often misread. Idempotent producers and transactions can make Kafka consume-transform-produce pipelines atomic with offset commits and output records when all participants are Kafka-aware and configured correctly. They do not make `sendEmail()`, `chargeCard()`, `updatePostgres()`, or `callPartnerApi()` exactly once. The safe statement in design reviews is: Kafka can reduce duplicates inside Kafka; business exactly-once is achieved through idempotent side effects and transactional boundaries.

## Retry Architecture and Failure Containment

### Retry Load

Retries amplify load unless controlled. A downstream outage turns every waiting message into a future retry. If every consumer retries immediately with high concurrency, the dependency receives more traffic while unhealthy than while healthy. When the dependency recovers, delayed retries can stampede. Backoff, jitter, concurrency caps, circuit breakers, and retry budgets are production controls, not niceties.

### Retry Classification

Retry classification matters. Transient network errors, rate limits, deadlocks, and temporary 5xx responses are retry candidates. Validation errors, unknown schema versions, missing required business entities, and invariant violations are often poison messages that should move to DLQ after minimal attempts. Retrying non-transient failures hides producer bugs and blocks partitions or queues.

### Retry Topics and Ordering

Kafka retry topics avoid blocking a source partition forever, but they can break strict ordering because later events may continue while a failed event waits in a delayed topic. If order must be preserved per aggregate, consumers need per-key blocking, state-machine guards, or a design that rejects out-of-order transitions until the missing predecessor is resolved. RabbitMQ retry queues can preserve or break order depending on topology; TTL-based shared retry queues can also release many messages simultaneously.

## Observability and Operations

### Application Metrics

Broker health metrics are necessary but insufficient. The application must expose processing latency, success/failure counts by handler, retry attempts, DLQ writes, idempotency conflicts, downstream dependency latency, batch sizes, poll/ack/commit timing, and per-partition or per-queue backlog age. Async systems need correlation IDs and event IDs in logs; otherwise the incident path disappears after the HTTP request returns.

### Kafka Dashboard

For Kafka, the operational dashboard starts with consumer lag by group/topic/partition, lag age, consume rate versus produce rate, rebalance frequency, commit latency, processing latency, under-replicated partitions, offline partitions, ISR shrink/expand, broker disk, request latency, controller health, and producer error/retry rates. A single hot partition can hide behind healthy average lag; alerts must preserve partition cardinality where it matters.

### RabbitMQ Dashboard

For RabbitMQ, watch ready messages, unacked messages, publish/deliver/ack rates, redelivery rate, consumer count, prefetch configuration, queue memory/disk, DLQ depth, connection/channel churn, flow control, memory alarms, disk alarms, and quorum health. High unacked with low ready means work is stuck in consumers. High ready with low unacked means consumers are absent, saturated, or dispatch is blocked. Rising redelivery usually means handler failures or consumer crashes.

### Runbook Questions

Operational runbooks should answer: can producers publish, are consumers alive, which partition/queue is oldest, which error dominates retries, did a deployment change schema or handler behavior, is the downstream dependency saturated, is the broker throttling, is retention at risk, and can replay be performed without side effects. Without these answers, messaging incidents degrade into manual offset edits and unsafe DLQ replays.

## Common Failure Patterns

- A consumer commits before processing and permanently skips a payment event when the database write fails. The fix is process-then-commit plus idempotent database writes.

- A consumer writes successfully, crashes before commit, and writes again after restart. The fix is a unique operation key, processed-event table, or state transition guard.

- A producer times out, retries, and creates duplicate records. The fix is idempotent producer configuration for retry duplicates and consumer idempotency for logical duplicates.

- A topic uses `status` as key and all `CREATED` events land on one hot partition. The fix is high-cardinality aggregate keys or deliberate sharding.

- A poison record blocks a Kafka partition because the handler retries forever before committing. The fix is bounded attempts, classified errors, retry topics or DLQ, and manual repair tooling.

- RabbitMQ prefetch is set too high; one slow worker receives a large unacked batch while other workers idle. The fix is right-sized prefetch, fair dispatch, concurrency tuning, and visibility into unacked messages.

- A DLQ grows for weeks because no team owns it. The fix is alerting, ownership, replay/discard tooling, and SLOs for failed-message resolution.

- A schema field is repurposed without compatibility checks. Old consumers parse successfully but apply wrong business logic. The fix is schema governance plus semantic versioning discipline, not just JSON parsing tolerance.

- A replay rebuilds a projection and also resends customer emails. The fix is separating pure projection handlers from side-effecting command handlers or adding explicit replay modes and idempotency keys.

## Message Brokers Interview Q&A

### 131. What is a message broker?

- A broker is a stateful delivery boundary between producers and consumers.
- The useful production answer is not “decoupling”; it is that the broker stores or routes work while producers, consumers, networks, and downstream dependencies fail independently.
- The cost is explicit handling of duplicates, ordering scope, acknowledgement timing, retries, backpressure, schema evolution, and monitoring.

### 132. Which protocol is used in classic message brokers?

- RabbitMQ commonly uses AMQP, though it can support other protocols through plugins.
- AMQP exposes exchanges, queues, bindings, routing keys, acknowledgements, channels, and message properties.
- In practice the protocol matters because routing and ack semantics are broker-level concepts, not just client library behavior.

### 133. What are the main production tradeoffs of brokers?

- Brokers improve temporal decoupling, buffering, fan-out, and failure recovery, but they introduce delayed failure, duplicate processing, stale reads, poison messages, replay risk, schema contracts, and operational state.
- They are appropriate when asynchronous completion is acceptable and the team can operate backlog, retries, DLQs, and idempotency.

### 134. What is Kafka?

- Kafka is a replicated partition log.
- Producers append records to partition leaders, followers replicate them, consumers fetch by offset, and retention deletes records independently of consumption.
- Its core engineering properties are partition-scoped order, consumer-group parallelism, replay within retention, high-throughput batching, and durability controlled by replication/ISR/producer acknowledgements.

### 135. How is Kafka different from classic queue brokers?

- Kafka keeps records for retention and lets many consumer groups read the same log independently.
- RabbitMQ-style queues generally deliver a message to a queue consumer and remove it after ack.
- Kafka scales by partitions and offsets; RabbitMQ scales by queues, consumers, routing, acknowledgements, and prefetch.
- Kafka is usually better for event streams and replayable integration; RabbitMQ is usually better for command/task dispatch and routing-heavy workflows.

### 136. What is a Kafka topic?

- A topic is a named collection of partition logs.
- It is not the unit of ordering or parallelism; partitions are.
- Topic design should encode event ownership, retention, schema compatibility, key strategy, access control, and consumer expectations.
- A topic that mixes unrelated event types or unstable schemas becomes an operational contract nobody can safely evolve.

### 137. What is a Kafka partition?

- A partition is an ordered append-only log with one leader, replicas, offsets, and one active consumer owner per group.
- It is the scaling and ordering primitive.
- More partitions can increase parallelism but also increase broker/controller overhead and rebalance cost.
- A bad key can make one partition the bottleneck even when the topic has many partitions.

### 138. What are push and pull strategies?

- Kafka uses pull: consumers fetch batches and control pace, which supports batching and consumer-side backpressure but exposes lag when processing falls behind.
- RabbitMQ commonly uses push: the broker delivers messages to consumers up to prefetch.
- Push gives low-latency dispatch but requires careful prefetch and ack handling to avoid overloading consumers or hiding backlog in unacked deliveries.

### 139. What are Kafka delivery semantics?

- At-most-once commits progress before processing and can lose records.
- At-least-once processes before committing and can duplicate records.
- Kafka transactions support exactly-once-style Kafka-to-Kafka pipelines by atomically writing output records and offsets, but external side effects still require idempotency and local transactional design.

### 140. What is a Kafka consumer group?

- A consumer group is the coordination unit for sharing partitions.
- Each partition is assigned to at most one consumer in the group, so parallelism is capped by partition count.
- Separate groups read independently from their own offsets.
- Scaling a service beyond partition count does not increase throughput for that topic; it only adds standby or idle instances.

### 141. What is a Kafka rebalance?

- A rebalance redistributes partition ownership after membership, subscription, timeout, or metadata changes.
- It is normal but expensive: consumption can pause, in-flight work may race with revocation, and duplicates occur around failed commits.
- Frequent rebalances usually indicate slow processing, blocked poll loops, unstable pods, aggressive timeouts, or deployment churn.

### 142. What is ZooKeeper and what replaced it in Kafka?

- ZooKeeper historically stored Kafka cluster metadata and participated in controller coordination.
- Modern Kafka can run in KRaft mode, where Kafka controllers manage metadata through an internal Raft-based quorum.
- The operational point is that newer clusters can remove the external ZooKeeper dependency, but metadata quorum health remains critical infrastructure.

### How does RabbitMQ routing work?

- A producer publishes to an exchange.
- The exchange evaluates bindings and routes to queues based on exchange type and routing key or headers.
- Direct exchanges match exact keys, topic exchanges match patterns, fanout exchanges broadcast, and headers exchanges match header predicates.
- If no binding matches, the message can be unroutable unless mandatory publishing/returns or alternate exchanges are configured.

### How do you scale Kafka consumers?

- First identify whether lag is uniform or partition-skewed.
- Add consumers only up to partition count.
- If all partitions are behind, increase processing capacity, batch safely, optimize downstream calls, or add partitions.
- If one partition is behind, fix the key distribution or split the hot aggregate.
- If retries dominate, isolate failures instead of adding consumers that hammer the same dependency.

### Why is idempotency mandatory in consumers?

- Because the common safe delivery mode is process-then-commit, and any crash or commit failure after side effects causes replay.
- Idempotency belongs at the durable side-effect boundary: unique constraints, processed-event tables, aggregate versions, upserts, state-machine guards, and external idempotency keys.
- Without it, normal broker recovery creates duplicate business actions.

### What ordering guarantees does Kafka provide?

- Kafka guarantees order only within a partition.
- There is no topic-wide order.
- To preserve order for an entity, use a stable key that maps all events for that entity to one partition and keep per-partition processing ordered.
- Retries, parallel handlers, retry topics, and stateful downstream writes can still break effective business ordering unless guarded by versions or state machines.

### What is a DLQ and when should it be used?

- A DLQ isolates messages that the normal path cannot process after bounded attempts or non-retryable classification.
- It prevents one poison message from blocking a queue or partition forever.
- A DLQ must have alerts, ownership, replay/discard tooling, and root-cause workflow; otherwise it becomes delayed data loss.

### How should retry handling be designed?

- Classify errors, cap attempts, use backoff with jitter, limit concurrency, preserve attempt metadata, and route exhausted messages to DLQ.
- Avoid immediate infinite retries.
- In Kafka, retry topics can prevent partition blockage but may break ordering.
- In RabbitMQ, TTL/DLX retry loops must avoid synchronized release storms and poison-message hot loops.

### What is eventual consistency in event-driven systems?

- It means services commit local state at different times and converge through events, retries, and reconciliation rather than one global transaction.
- The engineering requirement is not accepting inconsistency vaguely; it is defining source of truth, event versions, idempotent consumers, pending/failed states, repair jobs, and user-visible semantics while async work is incomplete.

### What is the outbox pattern?

- The outbox pattern writes business state and an event record in the same local database transaction, then publishes the event asynchronously to the broker.
- It removes the database/broker dual-write loss window but still allows duplicate publishes, so consumers need idempotency.
- It is the default pattern when a service must publish facts reliably after committing its own state.

### How do you monitor Kafka in production?

- Monitor lag and lag age by group/topic/partition, produce and consume rates, processing latency, commit latency, rebalance frequency, producer retries/errors, under-replicated and offline partitions, ISR churn, broker disk, request latency, and controller/quorum health.
- Application metrics must identify the handler, event type, partition, offset, key, attempt count, and downstream dependency latency.

### What is consumer lag and why does it matter?

- Lag is the distance between produced records and consumer progress for a group.
- It is unfinished work and stale state.
- Count alone is insufficient: partition distribution, oldest record age, catch-up rate, and business SLA decide severity.
- Lag can indicate insufficient consumers, hot partitions, slow downstream systems, poison messages, retry storms, or rebalance instability.
