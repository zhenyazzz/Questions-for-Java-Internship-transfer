# Module 8 — Caching + System Design — Questions 110–130

> Questions 110–130 from *Questions for Java Internship transfer.pdf* (caching fundamentals, strategies, invalidation, eviction; architecture styles; monolith vs microservices; inter-service communication; distributed transactions; common integration and resilience patterns; DDD; database-per-service; HTTP limitations). **Related:** Hibernate cache levels at **question 40** in `other/spring-questions-30-52.md`; CAP / NoSQL depth continues in **Module 10** (questions 143–161).

> Each numbered section ends with a **Middle+ answer** you can adapt on interview; bullets and tables above are for follow-up drilling.

## Module summary (middle+ — how I’d frame it on interview)

<a id="summary"></a>

**Caching** is a **latency and cost** tool: you trade **freshness and complexity** for fewer reads on the primary store (DB, remote API) and faster responses. The interview spine is four buckets—**where** you cache (browser, CDN, reverse proxy, app in-process, distributed Redis), **how** you populate (**cache-aside**, **read-through**, **write-through**, **write-behind**), **how** you invalidate (**TTL**, **event-driven**, **version keys**), and **how** you evict when memory is full (**LRU/LFU/TTL**, Redis `volatile-*` / `allkeys-*` policies). Middle+ means saying **which inconsistency window** you accept and **what breaks** under **double writes** or **thundering herd**.

**System design** in this module is mostly **architectural styles and patterns**, not drawing boxes for Netflix: **monolith vs microservices**, **sync vs async** communication, **distributed transactions** (why 2PC hurts, what **Saga / outbox / idempotency** buy you), and **cross-cutting patterns**—**API Gateway**, **BFF**, **service registry**, **sidecar**, **circuit breaker**, **CQRS**, **ACL**, **strangler**, **DDD**, and **database per service**. I tie answers to **operational reality**: deployability, blast radius, consistency, observability.

**HTTP** (question 130) is the default **inter-service** duct tape—**stateless**, **text-heavy**, **request/response**—fine for many edges, painful for **streaming**, **binary efficiency**, and **tight latency** without layering (gRPC, message brokers—**Module 9**). I also point forward: **CAP** and NoSQL trade-offs are **Module 10**.

### Quick map (question → one line)

| # | Topic | One-line takeaway |
|---|--------|-------------------|
| **110** | What is caching? | Copy of data closer to consumer to cut latency/cost; always a consistency trade-off. |
| **111** | Caching strategies | Cache-aside, read-through, write-through, write-behind—who writes cache and when. |
| **112** | Invalidation | TTL, explicit delete, event/pub-sub, version keys—hard problem, pick per domain. |
| **113** | Displacement / eviction | When cache is full: LRU/LFU/TTL/random; Redis `maxmemory-policy`. |
| **114** | Architecture styles | Monolith, layered, SOA, microservices, event-driven—different decomposition axes. |
| **115** | Monolith first historically | Simpler hardware/tooling; single process deployment before distributed systems maturity. |
| **116** | Monolith pros/cons | Simple dev/deploy/refactor vs scaling limits and coupling blast radius. |
| **117** | Microservices pros/cons | Independent scale/teams vs distributed complexity and consistency pain. |
| **118** | Microservice communication | Sync HTTP/gRPC, async messaging, hybrid; timeouts and contracts matter. |
| **119** | Distributed transactions | 2PC fragility; Saga, outbox, TCC, idempotency as pragmatic patterns. |
| **120** | Service registry | Dynamic discovery (Eureka, Consul, K8s Services+DNS). |
| **121** | Sidecar | Co-located helper container (Envoy, logging, mesh). |
| **122** | Circuit breaker | Stop hammering unhealthy deps; fail fast + half-open recovery. |
| **123** | CQRS | Split read/write models; eventual read models, complexity cost. |
| **124** | Anticorruption layer | Translation boundary between bounded contexts / legacy models. |
| **125** | Strangler | Incrementally replace legacy by routing traffic to new implementation. |
| **126** | API Gateway | Single edge for auth, routing, rate limit, cross-cutting concerns. |
| **127** | BFF | Per-client API façade shaping aggregation and UX needs. |
| **128** | DDD | Domain-driven modeling, bounded contexts, ubiquitous language. |
| **129** | DB per microservice | Loose coupling, independent scale/schema; avoid shared DB anti-pattern. |
| **130** | HTTP limitations | Stateless overhead, no streaming first-class in REST, head-of-line, verbosity, etc. |

## Table of contents

- [Module summary (middle+)](#summary)

110. [What is Caching?](#q110)
111. [What are caching strategies?](#q111)
112. [What are caching invalidation strategies?](#q112)
113. [What are cache displacement strategies?](#q113)
114. [What architecture styles exist in system design?](#q114)
115. [Why was monolith architecture created earlier than microservice architecture?](#q115)
116. [What are the pros and cons of monolithic architecture?](#q116)
117. [What are the pros and cons of microservice architecture?](#q117)
118. [What types of communication exist between microservices?](#q118)
119. [What is the problem of distributed transactions? What patterns help to solve this problem?](#q119)
120. [What is the Service Registry pattern?](#q120)
121. [What is the Sidecar pattern?](#q121)
122. [What is the Circuit breaker pattern?](#q122)
123. [What is the CQRS pattern?](#q123)
124. [What is the Anticorruption layer pattern?](#q124)
125. [What is the Strangler pattern?](#q125)
126. [What is the API Gateway pattern?](#q126)
127. [What is the Backend for Frontend pattern?](#q127)
128. [What is DDD?](#q128)
129. [Why should each microservice have its own db?](#q129)
130. [What are the main problems of HTTP protocol?](#q130)

---

<a id="q110"></a>

## 110. What is Caching?

### Question Restatement
What is **caching**, and why do systems use it?

### 1. Definition
**Caching** means storing a **copy of data** closer to the consumer (in memory, on another machine, or at the edge) so repeated reads are **faster** and cheaper than always hitting the **source of truth** (database, remote HTTP service, file system).

### 2. What you trade away
Caches introduce **staleness** and **invalidation complexity**. Strong consistency between cache and primary store is **hard**; most designs accept an **eventual** window or a **TTL** bound.

### 3. Middle+ answer (as on interview)

Caching is the practice of storing frequently accessed or expensive-to-compute data in a faster storage layer, such as memory, to reduce latency and avoid repeated calls to the main data source.

Instead of always querying a database or external service, the system first checks the cache and returns the data quickly if it is available.

The main benefit of caching is improved performance and scalability, as it reduces load on downstream systems.

However, caching introduces trade-offs. The most important one is data staleness, since cached data may become outdated. It also adds complexity around cache invalidation and consistency.

In practice, caching is used at different levels, such as in-memory caches, distributed caches like Redis, or edge caches like CDNs.

---

<a id="q111"></a>

## 111. What are caching strategies?

### Question Restatement
What **caching strategies** (population / write patterns) do you know?

### 1. Common strategies

| Strategy | Flow (simplified) | Typical use |
|----------|-------------------|-------------|
| **Cache-aside (lazy)** | App reads cache → miss → load DB → populate cache | General purpose CRUD |
| **Read-through** | App asks cache; cache loader pulls from DB on miss | Centralized caching layer |
| **Write-through** | Write goes to cache **and** DB synchronously | Need cache always fresh on write path |
| **Write-behind (write-back)** | Write accepted to cache, async flush to DB | Extreme write perf; risk if cache dies |

### 2. Middle+ answer (as on interview)

Caching strategies define how data is loaded into the cache and how updates are handled.

The most common strategies are:

Cache-aside (lazy loading) — the application checks the cache first, and on a miss, loads data from the database and stores it in the cache. This is the most widely used approach.
Read-through — the cache itself loads data from the database on a miss, so the application only interacts with the cache
Write-through — data is written to both the cache and the database synchronously, keeping the cache always up to date
Write-behind (write-back) — data is written to the cache first and persisted to the database asynchronously

Each strategy involves trade-offs:

cache-aside is simple but can lead to stale data or race conditions
write-through ensures consistency but increases write latency
write-behind improves performance but risks data loss if the cache fails

In practice, cache-aside is the most common strategy in backend systems, especially with tools like Redis.

---

<a id="q112"></a>

## 112. What are caching invalidation strategies?

### Question Restatement
How do you **invalidate** or refresh cache entries?

### 1. Strategy families
- **TTL / TTI** — time-to-live or idle expiry; simplest, bounded staleness.
- **Explicit delete/update** — on write path, **evict** keys or **write new value**; risk if write fails halfway.
- **Event-driven** — domain events (**Kafka**, Redis pub/sub) tell all instances to invalidate.
- **Version / cache key bump** — include **schema version** or `updatedAt` in key so old entries die naturally.

### 2. Middle+ answer (as on interview)

Cache invalidation strategies define how and when cached data is refreshed or removed to keep it consistent with the source of truth.

The main approaches are:

TTL (time-to-live) — cache entries expire after a fixed time. This is simple but allows temporary staleness
Explicit invalidation — cache is updated or cleared when the underlying data changes, usually after a successful database write
Event-driven invalidation — services publish events (e.g., via Kafka), and all consumers invalidate or update their caches accordingly
Versioned keys — cache keys include a version or timestamp, so updates automatically create new entries without deleting old ones

Each approach has trade-offs:

TTL is simple but imprecise
explicit invalidation is accurate but error-prone
event-driven scales well but is eventually consistent

In practice, systems often combine strategies, for example TTL with explicit invalidation or event-driven updates.

---

<a id="q113"></a>

## 113. What are cache displacement strategies?

### Question Restatement
When the cache is **full**, how do you **evict** entries (**displacement** / eviction policies)?

### 1. Classic policies
- **LRU** — evict least recently used (common approximation in Redis with sampling).
- **LFU** — evict least frequently used.
- **FIFO** — oldest inserted first (simple, not always “hotness” aware).
- **TTL-driven** — expire soonest deadline first.
- **Random** — cheap, sometimes used at scale.

### 2. Redis note
With `maxmemory`, Redis policies include **`allkeys-lru`**, **`volatile-lru`**, **`allkeys-lfu`**, **`volatile-ttl`**, **`noeviction`** (writes fail when full—watch for errors).

### 3. Middle+ answer (as on interview)

Cache displacement (or eviction) strategies define how the system decides which data to remove when the cache is full.

The most common strategies are:

LRU (Least Recently Used) — removes items that have not been accessed recently; works well for most workloads
LFU (Least Frequently Used) — removes items that are rarely accessed; useful when access patterns are stable
FIFO (First In, First Out) — removes the oldest entries regardless of usage
TTL-based eviction — removes entries that are closest to expiration
Random eviction — removes random entries, simple but less efficient

Each strategy has trade-offs:

LRU adapts to recent access patterns but can be affected by burst traffic
LFU is better for long-term popularity but slower to adapt
FIFO is simple but ignores usage patterns

In practice, LRU or LFU are the most commonly used strategies, often combined with TTL to control data lifetime.

---

<a id="q114"></a>

## 114. What architecture styles exist in system design?

### Question Restatement
What **architecture styles** (high-level patterns) exist?

### 1. Speaking list
- **Monolithic** — single deployable unit; modular monolith as a variant.
- **Layered (n-tier)** — presentation / application / domain / infrastructure separation inside one deployable.
- **Service-Oriented (SOA)** — coarse services, often ESB-centric (older enterprise default).
- **Microservices** — fine-grained services, independent deploy, **bounded contexts** ideally.
- **Event-driven / CQRS** — asynchronous reactions, projections, sometimes separate read/write paths.
- **Serverless / FaaS** — functions + managed services (style overlaps with microservices operationally).

### 2. Middle+ answer (as on interview)

Architecture styles describe how a system is structured and how responsibilities are divided.

The most common styles are:

Monolithic architecture — a single deployable unit that contains all functionality. It is simple to develop and deploy but can become hard to scale and maintain as the system grows
Layered architecture — separates concerns into layers such as presentation, business logic, and data access. It is often used inside monoliths or services
Service-Oriented Architecture (SOA) — systems are split into larger services, often coordinated through a central integration layer
Microservices architecture — systems are divided into small, independent services that can be deployed and scaled separately
Event-driven architecture — components communicate through events asynchronously, improving decoupling but adding complexity
Serverless architecture — uses managed cloud functions and services, reducing operational overhead

In practice, these styles are often combined. For example, microservices may use layered architecture internally and communicate using event-driven patterns.

---

<a id="q115"></a>

## 115. Why was monolith architecture created earlier than microservice architecture?

### Question Restatement
Historically, why did **monoliths** dominate before **microservices**?

### 1. Drivers
- **Hardware and cost** — fewer machines, smaller clusters; vertical scale first.
- **Tooling maturity** — RPC frameworks, containers, orchestration, CI/CD, tracing were weaker.
- **Network reliability and latency** — remote calls were expensive and flaky compared to in-process calls.
- **Organizational simplicity** — one artifact to ship; fewer cross-team contracts.

### 2. Middle+ answer (as on interview)

Monolithic architecture appeared earlier because it was the most practical and feasible approach given the limitations of early systems.

At that time:

Hardware was expensive, so systems were designed to run on a small number of machines
Networks were slower and less reliable, making remote communication costly compared to in-process calls
Tooling for distributed systems was immature, including deployment, monitoring, and service discovery
Operational simplicity was critical, and a single deployable unit was easier to manage

As a result, monoliths were the natural choice—they minimized complexity and made development and deployment simpler.

Microservices became viable later, when infrastructure and tooling improved, such as cloud platforms, containerization, CI/CD pipelines, and observability systems.

In practice, microservices are not “better by default”—they introduce significant complexity and are only justified when system scale and team size require them.

---

<a id="q116"></a>

## 116. What are the pros and cons of monolithic architecture?

A monolithic architecture is a single deployable application that contains all system components.

Pros:

Simple development and deployment — one codebase, one artifact
Easier debugging and testing — no network boundaries
Strong consistency — ACID transactions across modules
Fast development for small teams — fewer operational concerns

Cons:

Limited scalability — you must scale the entire application, even if only one part is under load
Low fault isolation — a failure in one component can affect the whole system
Team coupling — changes are harder to coordinate as the codebase grows
Technology lock-in — all components must use the same stack

In practice, monoliths are very effective in early stages, especially when designed as a modular monolith, but they can become difficult to maintain and scale as the system and team grow.

---

<a id="q117"></a>

## 117. What are the pros and cons of microservice architecture?

### Question Restatement
**Pros and cons** of **microservices**.

### 1. Pros
- **Independent deploy and scale** per service.
- **Team autonomy** aligned to **bounded contexts**.
- **Fault isolation** (when boundaries are real).
- **Polyglot** option per service (sometimes valuable).

### 2. Cons
- **Distributed system complexity** — latency, partial failures, **consistency**, **debugging**.
- **Operational overhead** — CI/CD × N, observability, service mesh, **on-call**.
- **Data duplication** and **eventual consistency** when avoiding shared DB.
- **Integration testing** and **contract** discipline required.

### 3. Middle+ answer (as on interview)

Microservice architecture splits a system into multiple independent services that communicate over the network.

Pros:

Independent deployment and scaling — each service can evolve and scale separately
Team autonomy — teams can work independently on different services
Fault isolation — failures are less likely to bring down the entire system
Flexibility in technology — different services can use different stacks

Cons:

Distributed system complexity — network latency, failures, and eventual consistency
Operational overhead — managing many services, deployments, and monitoring
Data consistency challenges — no shared transactions across services
Harder debugging and testing — requires tracing and good observability

In practice, microservices provide scalability and flexibility, but they come with significant complexity. They are most beneficial when system size and team structure require independent development and deployment.

---

<a id="q118"></a>

## 118. What types of communication exist between microservices?

### Question Restatement
What **communication types** exist between services?

### 1. Axes
- **Synchronous** — caller waits: **HTTP/REST**, **gRPC**, GraphQL.
- **Asynchronous** — message **queue/topic** (Kafka, RabbitMQ), **event log**, sometimes email/webhooks.
- **Hybrid** — sync user-facing path + async background processing.

### 2. Middle+ answer (as on interview)

Microservices communicate using two main types: synchronous and asynchronous communication.

Synchronous communication — the caller waits for a response. Common protocols include HTTP/REST and gRPC. It is simple and suitable for request-response scenarios, but introduces tight coupling and can lead to cascading failures if services are unavailable
Asynchronous communication — services communicate via messages or events using brokers like Kafka or RabbitMQ. The producer does not wait for the consumer, which improves resilience and scalability, but introduces complexity such as eventual consistency, retries, and ordering issues
Hybrid approach — most real systems combine both. For example, synchronous communication for user-facing requests and asynchronous messaging for background processing or event propagation

In practice, synchronous communication is used for immediate responses, while asynchronous communication is used for decoupling, scalability, and handling workflows.

---

<a id="q119"></a>

## 119. What is the problem of distributed transactions? What patterns help to solve this problem?

### Question Restatement
What goes wrong with **distributed transactions**, and which **patterns** address it?

### 1. The core problem
A transaction spanning **two networks and two databases** cannot get **ACID** as cheaply as one local transaction: **partial failure**, **blocking locks**, **CAP** trade-offs, **coordinator** outages.

### 2. Classic vs pragmatic
- **Two-phase commit (2PC)** — strong atomicity idea; **blocking**, fragile under partitions, not loved in cloud-native stacks.
- **Saga** — sequence of **local transactions** + **compensating actions** (choreography or orchestration).
- **Outbox pattern** — write business row + **outbox event** in **one DB transaction**, relay to broker reliably.
- **TCC** (Try-Confirm-Cancel) — explicit reservation phases; powerful, heavy to implement.
- **Idempotency keys** — make retries safe at business level.

### 3. Middle+ answer (as on interview)

The main problem of distributed transactions is that it is very difficult to guarantee atomicity and consistency across multiple services and databases.

In a distributed system:

services can fail independently
network communication is unreliable
there is no single transaction boundary

This makes traditional approaches like two-phase commit (2PC) impractical in most modern systems, because they are slow, blocking, and fragile under failures.

Instead, systems typically use eventual consistency with the following patterns:

Saga pattern — a sequence of local transactions where each step has a compensating action in case of failure
Outbox pattern — ensures that database updates and events are saved atomically, and then reliably published to a message broker
TCC (Try-Confirm-Cancel) — splits operations into reservation and confirmation phases, giving more control but adding complexity
Idempotency — ensures that repeated operations do not break the system

In practice, distributed transactions are avoided in favor of local ACID transactions combined with eventual consistency and retry-safe workflows.

---

<a id="q120"></a>

## 120. What is the Service Registry pattern?

### Question Restatement
What is a **service registry**, and why use it?

### 1. Definition
A **service registry** stores **network locations** (host:port, metadata) of **running instances** for a logical service name. Instances **register** on startup and **deregister** on shutdown; clients **discover** dynamically instead of hardcoding IPs.

### 2. Examples
**Eureka** (Spring Cloud Netflix era), **Consul**, **etcd**, **ZooKeeper** (older), **Kubernetes Service + DNS/CoreDNS** (platform-level registry abstraction).

### 3. Middle+ answer (as on interview)

A service registry is a system that keeps track of available service instances and their network locations.

In a microservices architecture, service instances are dynamic — they can start, stop, or change addresses. Instead of hardcoding endpoints, services register themselves in a registry, and clients discover them at runtime.

The typical flow is:

a service instance starts and registers itself
it periodically sends health checks
when it stops, it deregisters
clients query the registry to find available instances

This allows systems to perform service discovery and route requests dynamically.

Examples include tools like Eureka, Consul, and in Kubernetes, the Service + DNS mechanism acts as a built-in registry.

In practice, the service registry provides indirection between service names and actual instances, enabling scalability and fault tolerance.

---

<a id="q121"></a>

## 121. What is the Sidecar pattern?

### Question Restatement
What is the **Sidecar** pattern?

### 1. Definition
A **sidecar** is a **co-located auxiliary process/container** deployed alongside the **main application container** in the same **Pod** (K8s) or host—sharing network namespace (often) and lifecycle, but separate binary.

### 2. Typical responsibilities
**Envoy** proxy (mTLS, retries, traffic split), **logging agents** (Fluent Bit), **metrics exporters**, **vault agent** for secrets, **service mesh** data plane.

### 3. Middle+ answer (as on interview)

The Sidecar pattern means running an additional helper container alongside the main application container, typically in the same environment (e.g., Kubernetes Pod).

The sidecar handles cross-cutting concerns such as:

networking (proxies, retries, TLS)
logging and monitoring
configuration and secrets

This allows the main application to stay simple, while infrastructure concerns are handled externally.

In practice, sidecars are commonly used in service meshes, where a proxy (e.g., Envoy) manages communication, security, and observability.

The trade-off is additional resource usage and complexity, but it provides consistency and reduces duplicated logic across services.

---

<a id="q122"></a>

## 122. What is the Circuit breaker pattern?

### Question Restatement
What is the **Circuit breaker** pattern?

### 1. States (typical)
- **Closed** — calls pass through; failures are counted.
- **Open** — fast-fail without hitting the dependency after threshold.
- **Half-open** — trial calls to probe recovery.

### 2. Middle+ answer (as on interview)

The Circuit Breaker pattern prevents cascading failures in distributed systems by stopping calls to a failing service.

It has three main states:

Closed — requests are allowed, failures are monitored
Open — requests are blocked and fail fast
Half-open — limited requests are allowed to check recovery

If a service starts failing or timing out, the circuit breaker opens and stops sending requests, protecting both the caller and the failing service.

After a cooldown period, it allows a few test requests to determine if the service has recovered.

In practice, circuit breakers improve system resilience, but must be combined with timeouts, retries, and fallback logic to be effective.

---

<a id="q123"></a>

## 123. What is the CQRS pattern?

### Question Restatement
What is **CQRS**?

### 1. Definition
**Command Query Responsibility Segregation** — separate **models** (and sometimes **data stores**) for **writes (commands)** and **reads (queries)**. Reads may use **denormalized projections** optimized for UI/reporting.

### 2. Trade-offs
**Pros:** scale reads independently, **shape data per query**, offload heavy reporting. **Cons:** **eventual consistency**, **complexity**, duplicate logic, risk of **projection bugs**.

### 3. Middle+ answer (as on interview)

CQRS (Command Query Responsibility Segregation) separates write operations (commands) from read operations (queries).

Commands modify state and go through a write model with business logic and validation. Queries read data from a read model, which is often optimized for specific use cases (e.g., denormalized views or caches).

This allows systems to:

scale reads and writes independently
optimize data structures for different access patterns

However, CQRS introduces complexity and eventual consistency, since the read model may lag behind the write model.

In practice, CQRS is useful when read and write workloads differ significantly, but is often overkill for simple CRUD systems.

---

<a id="q124"></a>

## 124. What is the Anticorruption layer pattern?

### Question Restatement
What is the **Anticorruption Layer (ACL)**?

### 1. Definition (DDD)
An **ACL** is a **translation boundary** between your **bounded context** and a **foreign model** (legacy system, partner API, messy schema) so their terminology and invariants **do not leak** into your core domain.

### 2. Middle+ answer (as on interview)

The Anticorruption Layer (ACL) is a design pattern that protects your system’s domain model from external systems.

It acts as a translation layer between your internal model and an external or legacy system, converting data and concepts into your own domain language.

This prevents external models, formats, or inconsistencies from leaking into your core business logic.

In practice, an ACL is implemented as a separate module or adapter that:

maps external data to internal domain objects
enforces your system’s rules and invariants

The goal is to keep your domain clean and decoupled, even when integrating with poorly designed or legacy systems.

---

<a id="q125"></a>

## 125. What is the Strangler pattern?

### Question Restatement
What is the **Strangler Fig** pattern?

### 1. Definition
Incrementally **replace** a legacy system by placing a **router/proxy/gateway** in front that routes **some traffic** to the **new implementation** while the rest still hits legacy—**growing** the new slice until legacy can be **retired**.

### 2. Middle+ answer (as on interview)

The Strangler pattern is a migration strategy used to gradually replace a legacy system with a new one.

Instead of rewriting everything at once, a proxy or gateway is placed in front of the system and routes some requests to the new implementation while the rest still go to the legacy system.

Over time, more functionality is moved to the new system until the legacy system can be fully removed.

This approach reduces risk, allows incremental delivery, and provides an easy rollback by switching routing back if needed.

---

<a id="q126"></a>

## 126. What is the API Gateway pattern?

### Question Restatement
What is an **API Gateway**?

### 1. Responsibilities (typical)
**Single entry** for clients: **authentication**, **rate limiting**, **routing**, **SSL termination**, **request shaping**, **API composition** (light), **caching**, **A/B** routing.

### 2. Products / implementations
Kong, AWS API Gateway, Spring Cloud Gateway, Envoy + control plane, NGINX.

### 3. Middle+ answer (as on interview)

An API Gateway is a single entry point for client requests in a microservices architecture.

It handles cross-cutting concerns such as:

routing requests to the correct service
authentication and authorization
rate limiting and security
request transformation and aggregation

This allows backend services to focus on business logic instead of infrastructure concerns.

In practice, an API Gateway improves security and simplifies client interaction, but it can become a bottleneck if overloaded or overloaded with business logic.

---

<a id="q127"></a>

## 127. What is the Backend for Frontend (BFF) pattern?

### Question Restatement
What is **BFF**?

### 1. Definition
A **BFF** is a **backend tailored to a specific client type** (iOS BFF, Web BFF) that **aggregates** domain APIs, **shapes** payloads for UI needs, and hides **chattiness** from the device.

### 2. Middle+ answer (as on interview)

The Backend for Frontend (BFF) pattern means creating a separate backend service tailored for a specific client, such as web or mobile.

Instead of having a single generic API, each client communicates with its own backend, which:

aggregates data from multiple services
shapes responses according to UI needs
reduces the number of requests from the client

This improves performance and simplifies frontend logic.

In practice, BFF is used together with an API Gateway: the gateway handles cross-cutting concerns, while the BFF handles client-specific logic.

The trade-off is increased system complexity and potential duplication of logic across BFFs.

---

<a id="q128"></a>

## 128. What is DDD?

### Question Restatement
What is **Domain-Driven Design (DDD)**?

### 1. Core ideas
- **Ubiquitous language** — code names match business language.
- **Bounded context** — model consistency boundary; explicit **context maps** between contexts.
- **Aggregates** — consistency cluster rooted at an **aggregate root**.
- **Strategic vs tactical** DDD — big-picture boundaries vs patterns (entities, value objects, repositories, domain events).

### 2. Middle+ answer (as on interview)

Domain-Driven Design (DDD) is an approach to software design that focuses on modeling the system based on the business domain.

The main idea is to align code with business concepts using a ubiquitous language shared between developers and domain experts.

Key concepts include:

Bounded contexts — clear boundaries where a specific model applies
Entities and value objects — representing domain data
Aggregates — consistency boundaries for data changes

DDD helps manage complexity by structuring the system around real business problems instead of technical layers.

In practice, DDD is most useful for complex domains, while simple applications may not need its full set of patterns.

---

<a id="q129"></a>

## 129. Why should each microservice have its own db?

### Question Restatement
Why **database per service**?

### 1. Reasons
- **Loose coupling** — no hidden joins across teams.
- **Independent schema evolution** — migrations do not block other services.
- **Independent scaling** — different storage engines/SKUs per workload.
- **Failure isolation** — DB outage scope is smaller.
- **Polyglot persistence** — Redis for one, Postgres for another.

### 2. Trade-off
**Distributed queries** become **orchestration**, **eventual consistency**, **data duplication**—that is intentional.

### 3. Middle+ answer (as on interview)

Each microservice should have its own database to ensure loose coupling and clear ownership of data.

If multiple services share the same database, they become tightly coupled:

changes in one service’s schema can break others
teams cannot evolve independently
hidden dependencies appear through direct database access

With a database per service:

each service controls its own schema and data
services communicate through APIs or events, not direct queries
systems are easier to scale and maintain

The trade-off is that you lose simple joins across services and must handle eventual consistency and data duplication.

In practice, database per service enforces proper boundaries and autonomy, which is critical in microservice architecture.

---

<a id="q130"></a>

## 130. What are the main problems of HTTP protocol?

### Question Restatement
What are **HTTP’s limitations** (especially for large-scale / internal systems)?

### 1. Common talking points
- **Request/response bias** — no first-class bidirectional streaming in classic REST (WebSockets/gRPC layered separately).
- **Overhead** — textual headers (mitigated by **HTTP/2 HPACK**, **HTTP/3 QPACK**).
- **Head-of-line blocking** (HTTP/1.1 keep-alive with many objects; **HTTP/2** multiplexes but has TCP-level HoL; **HTTP/3** over QUIC improves).
- **Statelessness** as double-edged sword — every call may repeat **auth/context** unless cookies/tokens optimized.
- **No built-in discovery/semantics** — patterns (REST, OpenAPI) are conventions, not the wire.
- **Polling inefficiency** for real-time unless you add **SSE/WebSockets**.

### 2. Middle+ answer (as on interview)

HTTP is widely used because of its simplicity and interoperability, but it has limitations in large-scale systems.

The main issues are:

Request-response model — not suitable for real-time or streaming without additional protocols
Overhead — headers and text-based format increase payload size
Latency — each request involves network round trips
Statelessness — requires sending context (e.g., authentication) with every request
Inefficiency for frequent updates — polling wastes resources compared to event-driven approaches

In practice, these limitations lead systems to use alternatives like gRPC for efficient communication or message brokers for asynchronous processing.

HTTP is not bad, but it is not always the best choice for internal high-load communication.

---
