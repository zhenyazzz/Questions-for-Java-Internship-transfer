# Module 9 — Messaging (Kafka / Brokers) — Questions 131–142

> Questions 131–142 from *Questions for Java Internship transfer.pdf* (message brokers, classic broker protocols, pros/cons, Kafka vs classic brokers, topics/partitions, push vs pull, delivery semantics, consumer groups, rebalance, ZooKeeper vs **KRaft**). **Related:** HTTP limitations and when to prefer messaging (**question 130** in `Модуль 8 — Caching + System Design.md`); NoSQL / CAP continues in **Module 10** (from **question 143**).

> Each numbered section ends with a **Middle+ answer** you can adapt on interview; bullets and tables above are for follow-up drilling.

## Module summary (middle+ — how I’d frame it on interview)

<a id="summary"></a>

**Message brokers** sit between producers and consumers to **decouple** availability, **smooth spikes** (buffer), and enable **async workflows**. “Classic” brokers (RabbitMQ, ActiveMQ, etc.) often expose **AMQP**, **MQTT**, or **JMS**-style APIs and **push** (or broker-scheduled) delivery semantics tuned for **queues** and **relatively low** retained volume. **Apache Kafka** is a **distributed append-only log**: **topics** split into **partitions** for **ordering, scale, and replay**; consumers **pull** long batches; retention is first-class—not “delete after ack” only.

**Kafka-specific** interview spine: **topic vs partition** (ordering scope = partition), **consumer groups** for **parallelism + scale-out** with partition assignment, **rebalance** when membership changes, and **delivery semantics** (**at-most-once**, **at-least-once**, **exactly-once** with transactions/idempotent producer—**EOS** has real operational cost). **ZooKeeper** used to coordinate Kafka metadata; modern Kafka moves to **KRaft** (Kafka Raft) for **metadata quorum** without ZK.

**Trade-offs:** brokers add **latency**, **duplicates**, **ordering** puzzles, and **ops** (monitoring lag, consumer health, **DLQ** patterns). Middle+ means naming **idempotency**, **outbox**, ** poison messages**, and **when not to use Kafka** (small fire-and-forget might be a queue).

### Quick map (question → one line)

| # | Topic | One-line takeaway |
|---|--------|-------------------|
| **131** | Message broker | Middleware decoupling producers/consumers; buffering, routing, delivery. |
| **132** | Classic broker protocols | AMQP, MQTT, STOMP; JMS as API often over vendor wire. |
| **133** | Pros/cons of brokers | Decouple & absorb spikes vs ops complexity, duplicates, ordering. |
| **134** | What is Kafka? | Distributed commit log; partitions; retention; consumer groups. |
| **135** | Kafka vs classic | Log + pull + retention vs queue + push-ish + delete-on-ack mindset. |
| **136** | Kafka topic | Named log category; logical stream of events/messages. |
| **137** | Kafka partition | Ordered append-only sub-log; unit of parallelism and key ordering. |
| **138** | Push vs pull | Classic often push-oriented; Kafka consumers long-poll **pull**. |
| **139** | Delivery semantics | At-most / at-least / exactly-once (EOS) and what they really mean. |
| **140** | Consumer group | Shared `group.id` → partition assignment; scale with partition count. |
| **141** | Consumer rebalance | Reassign partitions on join/leave/subscription change—pauses throughput. |
| **142** | ZooKeeper vs KRaft | ZK was external metadata quorum; KRaft replaces it in modern Kafka. |

## Table of contents

- [Module summary (middle+)](#summary)

131. [What is a message broker?](#q131)
132. [Which protocol is used in classic message brokers?](#q132)
133. [What are the pros and cons of message brokers?](#q133)
134. [What is Kafka?](#q134)
135. [What is the difference between Kafka and classic message brokers?](#q135)
136. [What is Topic in Kafka?](#q136)
137. [What is Partition in Kafka?](#q137)
138. [What are push and pull strategies? What is used in Kafka? What is used in classic message brokers?](#q138)
139. [What are delivery semantics in Kafka?](#q139)
140. [What is the Consumer group in Kafka?](#q140)
141. [What is the Consumer rebalance in Kafka?](#q141)
142. [What is Apache Zookeeper? What is the new approach used in Kafka instead of ZooKeeper?](#q142)

---

<a id="q131"></a>

## 131. What is a message broker?

### Question Restatement
What is a **message broker**?

### 1. Definition
A **message broker** is middleware that **accepts messages** from **producers**, **stores** them (often transiently), and **delivers** them to **consumers** according to routing rules, subscriptions, and quality-of-service settings.

### 2. Roles
**Decouple** producers from consumers in time and availability, **buffer** bursts, enable **fan-out** (pub/sub) or **work queues** (competing consumers).

### 3. Middle+ answer (as on interview)

A message broker is middleware that sits between services and enables asynchronous communication by receiving messages from producers, temporarily storing them, and delivering them to consumers.

In practice, it decouples services so the producer does not need to know whether the consumer is available at the moment. Instead of direct calls, services communicate through the broker.

Message brokers provide:

Decoupling — services are independent in time and availability
Buffering — they absorb traffic spikes via queues or logs
Routing — messages can be distributed using queues or pub/sub
Reliability mechanisms — retries, acknowledgments, and persistence

Compared to synchronous HTTP communication, brokers improve resilience and scalability, but introduce trade-offs such as duplicate messages, eventual consistency, ordering constraints, and the need for idempotent consumers.

In production, a message broker is not just a transport layer—it’s a core infrastructure component that must be monitored and operated carefully.

---

<a id="q132"></a>

## 132. Which protocol is used in classic message brokers?

### Question Restatement
Which **protocol(s)** typify **classic** message brokers?

### 1. Common protocols / APIs
- **AMQP 0-9-1** — RabbitMQ’s native lingua franca; exchanges, queues, bindings.
- **MQTT** — lightweight pub/sub; IoT and telemetry heavy.
- **STOMP** — simple text framing; easy from many languages.
- **JMS** — **Java API**; wire protocol is **vendor-specific** (OpenWire, HornetQ, etc.) behind a standard client surface.

### 2. Middle+ answer (as on interview)

There is no single universal protocol, but the most common one in classic message brokers is AMQP (Advanced Message Queuing Protocol).

AMQP is a binary protocol that defines how messages are structured, routed, and delivered using concepts like exchanges, queues, and bindings. It is widely used in systems like RabbitMQ.

Other protocols include:

MQTT — a lightweight publish/subscribe protocol used in IoT and low-bandwidth environments
STOMP — a simple text-based protocol, often used for interoperability
JMS — not a protocol, but a Java API that abstracts messaging; the actual wire protocol depends on the broker implementation

In practice, when people refer to “classic message brokers,” they usually mean systems like RabbitMQ, where AMQP is the primary protocol.

---

<a id="q133"></a>

## 133. What are the pros and cons of message brokers?

### Question Restatement
**Pros and cons** of **message brokers**.

### 1. Pros
- **Temporal decoupling** — producer and consumer need not be up together.
- **Load leveling** — absorb bursts; protect databases.
- **Fan-out / fan-in** — pub/sub patterns, competing consumers.
- **Cross-language integration** — stable wire APIs.

### 2. Cons
- **Operational complexity** — HA clusters, upgrades, monitoring, **disk full** incidents.
- **Debugging** — distributed traces must include **message spans**.
- **Consistency** — **at-least-once** duplicates unless consumers are **idempotent**.
- **Wrong tool** — synchronous query workflows forced into Kafka become **slow RPC**.

### 3. Middle+ answer (as on interview)

Pros:

Decoupling — producers and consumers are independent in time and availability
Load leveling — brokers buffer spikes and protect downstream services
Scalability — support for pub/sub and competing consumers enables horizontal scaling
Resilience — failures in one service do not immediately break others

Cons:

Operational complexity — requires managing clusters, monitoring, and scaling
Eventual consistency — data is not immediately consistent across services
Duplicates and ordering issues — systems must handle at-least-once delivery and ensure idempotency
Harder debugging — distributed flows are less transparent than synchronous calls

In practice, message brokers improve system resilience and scalability, but they introduce complexity and require careful design. They are not a default choice for simple request-response workflows.

---

<a id="q134"></a>

## 134. What is Kafka?

### Question Restatement
What is **Apache Kafka**?

### 1. Definition
**Kafka** is a **distributed, fault-tolerant, append-only log** platform. Producers **append** **records** to **topics** (split into **partitions**); consumers **read** with offsets; brokers replicate partitions for **durability**.

### 2. Distinctive properties
**Retention by time/size** (not only “delete on ack”), **replay**, **high throughput**, **horizontal scale** by adding partitions/brokers.

### 3. Middle+ answer (as on interview)

Apache Kafka is a distributed, fault-tolerant platform for storing and processing streams of data.

At its core, Kafka works as a distributed append-only log: producers write records to topics, which are split into partitions, and consumers read those records using offsets.

Kafka is designed for:

High throughput — efficient sequential disk writes and batching
Scalability — data is distributed across partitions and brokers
Fault tolerance — partitions are replicated across multiple nodes

A key difference from traditional message brokers is that Kafka retains data for a configured time or size, allowing consumers to replay events instead of consuming them only once.

In practice, Kafka is used for event-driven architectures, data pipelines, and real-time processing systems.

---

<a id="q135"></a>

## 135. What is the difference between Kafka and classic message brokers?

### Question Restatement
**Kafka vs classic** message brokers (RabbitMQ, ActiveMQ, …).

### 1. Comparison table (speaking)

| Aspect | Classic queue broker (e.g. Rabbit) | Kafka |
|--------|-----------------------------------|--------|
| **Mental model** | Queue/work queue, often delete after ack | Durable **log**, retention-based |
| **Consumption** | Often **push** / broker-driven dispatch | **Pull** (long poll), batching |
| **Ordering** | Per queue; complicated global ordering | **Per partition** strong order |
| **Replay** | Not primary design | **First-class** |
| **Routing** | Exchanges, bindings, DLX patterns | Topic + partition key → partition |

### 2. Middle+ answer (as on interview)

The key difference is the data model and consumption approach.

Classic message brokers like RabbitMQ are queue-based systems. Messages are delivered to consumers and typically removed after acknowledgment, focusing on task distribution and low-latency processing.

Kafka, on the other hand, is a distributed log-based system. Messages are stored in append-only logs, retained for a configured period, and not deleted after consumption. Consumers read messages using offsets and can replay data.

Other important differences:

Consumption model: classic brokers use push, Kafka uses pull
Ordering: Kafka guarantees order within a partition, while classic brokers handle ordering per queue
Replay: Kafka supports replay as a core feature, while classic brokers do not
Use case: classic brokers are better for task queues and routing, Kafka is better for event streaming and data pipelines

In practice, the choice depends on whether you need reliable task delivery or a scalable event log with replay capabilities.

Kafka is not just a broker, it is a storage system for events, while classic brokers focus on message delivery.

---

<a id="q136"></a>

## 136. What is Topic in Kafka?

### Question Restatement
What is a **topic** in Kafka?

### 1. Definition
A **topic** is a **named category** or **feed** of records—a **logical** stream (e.g. `orders.events`). Topics are **divided into partitions** for parallelism.

### 2. Configuration knobs (interview)
**Retention** (`retention.ms`, `retention.bytes`), **compaction** (`cleanup.policy=compact` for changelog topics), **replication factor**, **min.insync.replicas** for durability vs availability.

### 3. Middle+ answer (as on interview)

A topic in Kafka is a named stream of records that producers write to and consumers read from.

Logically, it represents a category of data, for example orders or payments.

Physically, a topic is divided into partitions, where each partition is an append-only log. This allows Kafka to scale horizontally and process data in parallel.

An important detail is that ordering is guaranteed only within a partition, not across the entire topic. To preserve order for related data, a key is typically used to ensure messages go to the same partition.

Topics can also be configured with retention policies, meaning data is stored for a certain time or size, allowing consumers to replay events.
Topics are not just logical names — they are backed by physical partitions distributed across brokers.

---

<a id="q137"></a>

## 137. What is Partition in Kafka?

### Question Restatement
What is a **partition** in Kafka?

### 1. Definition
A **partition** is an **ordered, immutable sequence** of records assigned a monotonically increasing **offset**. Partitions are **replicated** across brokers (leader + followers).

### 2. Why partitions exist
**Parallelism** (consumers ≤ partitions for same group), **horizontal scale**, and **per-key ordering** when keyed producer routing is used.

### 3. Middle+ answer (as on interview)

A partition is an ordered, append-only log of records within a Kafka topic. Each record in a partition has a unique, monotonically increasing offset.

Partitions are the key mechanism that allows Kafka to achieve scalability and parallelism. A topic is split into multiple partitions, which are distributed across different brokers (nodes) in the cluster and processed independently.

Each partition is replicated across multiple brokers based on the replication factor. These replicas are stored on different Kafka instances to provide fault tolerance.

For every partition, Kafka elects a leader replica, while the remaining replicas act as followers.

The leader handles all read and write requests from producers and consumers.
Followers continuously replicate data from the leader to stay in sync.

In case the leader broker fails, Kafka automatically promotes one of the in-sync follower replicas to become the new leader, ensuring high availability.

Ordering is guaranteed only within a partition, not across the entire topic. To maintain order for related data, producers typically use a key, so all related messages are routed to the same partition.

In practice, partitions define both the maximum parallelism (one consumer per partition in a group) and the ordering guarantees of the system.

---

<a id="q138"></a>

## 138. What are push and pull strategies? What is used in Kafka? What is used in classic message brokers?

### Question Restatement
**Push vs pull**; what **Kafka** vs **classic** brokers tend to do.

### 1. Push model
Broker **pushes** messages to consumers (or schedules delivery threads). Consumer must **signal readiness** (prefetch limits) or risk **overwhelm**.

### 2. Pull model
Consumer **requests** messages when ready (**long polling**). **Kafka consumers pull** batches from brokers.

### 3. Middle+ answer (as on interview)

In a push model, the broker actively sends messages to consumers as soon as they are available. The consumer must be ready to handle incoming messages, often using mechanisms like prefetch limits to avoid being overwhelmed.

In a pull model, the consumer requests messages from the broker when it is ready. This gives the consumer control over the rate of consumption.

Kafka uses a pull-based model. Consumers actively fetch messages in batches, which allows better control over load and enables efficient batching.

Classic message brokers like RabbitMQ typically use a push-based model, where the broker delivers messages to consumers.

The main trade-off is that push can provide lower latency but requires careful backpressure handling, while pull provides better scalability and control, especially in high-throughput systems like Kafka.

In push-based systems like RabbitMQ, the broker tracks consumer readiness using acknowledgments and prefetch limits. It distributes messages among consumers, typically in a round-robin manner, but only sends new messages to consumers that have capacity.

---

<a id="q139"></a>

## 139. What are delivery semantics in Kafka?

### Question Restatement
What **delivery semantics** exist in Kafka?

### 1. Three classical semantics
- **At-most-once** — offsets committed **before** processing; you may **lose** messages on crash, never duplicate processing.
- **At-least-once** — process then commit offset (or careful ordering); **duplicates** possible on failure—need **idempotent** consumers.
- **Exactly-once (EOS)** — end-to-end once semantics within defined boundaries: **idempotent producer** + **transactions** + **read-process-write** patterns; operational and latency cost.

### 2. Kafka levers
**Producer `acks`**, **idempotence**, **`enable.idempotence`**, **transactions** (`transactional.id`), **isolation levels** for consumers (`read_committed`).

### 3. Middle+ answer (as on interview)

Kafka supports three main delivery semantics:

At-most-once — messages are committed before processing, so they are never redelivered, but can be lost if a failure occurs. In practice, this is achieved by configuring the producer to not wait for broker acknowledgments (acks=0) and disabling retries, while the consumer uses automatic offset commits. This setup minimizes latency but sacrifices reliability, since any failure during processing can result in permanent data loss. It is typically used in scenarios like metrics, logging, or telemetry, where occasional data loss is acceptable.

At-least-once — messages are processed first and offsets are committed afterward; this guarantees no data loss, but can result in duplicates. In practice, producers are configured with acks=all and retries enabled, often combined with idempotence, while consumers disable auto-commit and explicitly commit offsets only after successful processing. This ensures that messages are not lost, but if a consumer fails after processing and before committing the offset, the same message may be processed again. This is the most commonly used model in real systems such as order processing, payments, and other business-critical workflows, where duplicates must be handled through idempotent logic.

Exactly-once — Kafka provides transactional support and idempotent producers to ensure that messages are processed exactly once within Kafka’s boundaries. This is achieved by enabling idempotent producers and using transactions with a transactional.id, allowing the application to atomically write results and commit offsets together. However, this guarantee only holds within Kafka itself. When external systems such as databases are involved, additional mechanisms like idempotency or the outbox pattern are still required. Exactly-once is typically used in Kafka-to-Kafka pipelines, such as stream processing with Kafka Streams or ETL workflows.

The key idea is that delivery semantics depend on when offsets are committed relative to processing. In real systems, achieving true end-to-end exactly-once semantics across multiple services is extremely difficult, so most production systems rely on at-least-once delivery combined with idempotent processing.

---

<a id="q140"></a>

## 140. What is the Consumer group in Kafka?

### Question Restatement
What is a **consumer group**?

### 1. Definition
Consumers sharing the same **`group.id`** form a **consumer group**. The group **coordinates** so **each partition** of a subscribed topic is assigned to **at most one consumer** in the group at a time (for standard subscribers).

### 2. Implications
**Horizontal scale** of consumption is limited by **partition count**; multiple groups independently read the **same topic** with their own offsets.

### 3. Middle+ answer (as on interview)

A consumer group is a set of consumers that share the same group.id and work together to consume data from Kafka topics.

Within a group, Kafka ensures that each partition is assigned to only one consumer at a time, which allows parallel processing of data.

This means that the level of parallelism is limited by the number of partitions. For example, if a topic has 10 partitions, at most 10 consumers in a group can actively process data.

Different consumer groups can read the same topic independently, each maintaining their own offsets. This allows multiple systems to process the same stream of events in different ways.

In practice, a consumer group is Kafka’s mechanism for scaling consumption and distributing load across instances.

If a consumer joins or leaves the group, Kafka triggers a rebalance to redistribute partitions.

---

<a id="q141"></a>

## 141. What is the Consumer rebalance in Kafka?

### Question Restatement
What is **consumer rebalance** in Kafka?

### 1. Definition
**Rebalance** is the process of **reassigning topic partitions** among consumers in a group when **membership changes** (consumer join/leave/crash), **subscription changes**, or **partition count** changes.

### 2. Effects
During rebalance, partitions may be **revoked** temporarily → **stop-the-world** pauses for those partitions unless using **incremental cooperative** protocols (modern consumer).

### 3. Middle+ answer (as on interview)

A consumer rebalance is the process where Kafka redistributes topic partitions among consumers in a group when the group membership changes.

This happens when:

a consumer joins or leaves the group
a consumer crashes or stops sending heartbeats
the number of partitions changes

During a rebalance, Kafka reassigns partitions so that each partition is still processed by only one consumer in the group.

Rebalancing can temporarily pause consumption, because consumers must stop processing, release their partitions, and receive new assignments.

In practice, rebalances can cause issues such as temporary downtime, duplicate processing, or uneven load distribution. That’s why it’s important to keep consumers stable and avoid long processing times that can trigger unnecessary rebalances.

---

<a id="q142"></a>

## 142. What is Apache Zookeeper? What is the new approach used in Kafka instead of ZooKeeper?

### Question Restatement
What was **ZooKeeper’s** role for Kafka, and what **replaces** it?

### 1. ZooKeeper role (legacy Kafka)
**Apache ZooKeeper** is a **distributed coordination service** (quorum-based consistency) used historically by Kafka for **controller election**, **cluster metadata**, **ACLs**, **broker registration**, and **consumer group** coordination internals (depending on version).

### 2. KRaft (Kafka Raft)
**KRaft mode** (**Kafka Raft metadata quorum**) stores Kafka metadata **inside Kafka itself** using the **Raft consensus** protocol—**no ZooKeeper dependency** for new clusters.

### 3. Middle+ answer (as on interview)

Apache ZooKeeper is a distributed coordination service that was historically used by Kafka to manage cluster metadata and coordination tasks.

In older Kafka versions, ZooKeeper was responsible for:

controller election
broker registration
topic and partition metadata
cluster coordination

However, using ZooKeeper meant running and maintaining a separate distributed system alongside Kafka, which increased operational complexity.

Kafka has introduced a new approach called KRaft (Kafka Raft mode), which removes the dependency on ZooKeeper. In KRaft mode, Kafka manages its own metadata internally using the Raft consensus protocol.

This simplifies the architecture by eliminating the need for ZooKeeper and improves scalability and reliability.

In practice, ZooKeeper-based Kafka is considered legacy, and modern Kafka deployments use KRaft mode.

KRaft removes the need to operate two distributed systems and simplifies deployment and scaling.

---
