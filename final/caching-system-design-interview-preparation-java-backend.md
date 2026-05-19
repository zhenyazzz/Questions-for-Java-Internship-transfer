# Caching + System Design Interview Preparation (Questions 110–130)

## Table of Contents
- [110. What is Caching?](#110-what-is-caching)
- [111. What are caching strategies?](#111-what-are-caching-strategies)
- [112. What are caching invalidation strategies?](#112-what-are-caching-invalidation-strategies)
- [113. What are cache displacement strategies?](#113-what-are-cache-displacement-strategies)
- [114. What architecture styles exist in system design?](#114-what-architecture-styles-exist-in-system-design)
- [115. Why was monolith architecture created earlier than microservice architecture?](#115-why-was-monolith-architecture-created-earlier-than-microservice-architecture)
- [116. What are the pros and cons of monolithic architecture?](#116-what-are-the-pros-and-cons-of-monolithic-architecture)
- [117. What are the pros and cons of microservice architecture?](#117-what-are-the-pros-and-cons-of-microservice-architecture)
- [118. What types of communication exist between microservices?](#118-what-types-of-communication-exist-between-microservices)
- [119. What is the problem of distributed transactions? What patterns help to solve this problem?](#119-what-is-the-problem-of-distributed-transactions-what-patterns-help-to-solve-this-problem)
- [120. What is the Service Registry pattern?](#120-what-is-the-service-registry-pattern)
- [121. What is the SideCar pattern?](#121-what-is-the-sidecar-pattern)
- [122. What is the Circuit Breaker pattern?](#122-what-is-the-circuit-breaker-pattern)
- [123. What is the CQRS pattern?](#123-what-is-the-cqrs-pattern)
- [124. What is the Anti-Corruption Layer pattern?](#124-what-is-the-anti-corruption-layer-pattern)
- [125. What is the Strangler pattern?](#125-what-is-the-strangler-pattern)
- [126. What is the API Gateway pattern?](#126-what-is-the-api-gateway-pattern)
- [127. What is the Backend for Frontend pattern?](#127-what-is-the-backend-for-frontend-pattern)
- [128. What is DDD?](#128-what-is-ddd)
- [129. Why should each microservice have its own database?](#129-why-should-each-microservice-have-its-own-database)
- [130. What are the main problems of HTTP protocol?](#130-what-are-the-main-problems-of-http-protocol)

# 110. What is Caching?
Caching is an explicit decision to trade consistency for latency and cost. In production systems, the primary database usually becomes the first bottleneck under read-heavy traffic, not because queries are always badly written, but because repeated reads of the same hot data saturate connection pools, buffer cache, and IO. A Redis layer in front of PostgreSQL or MongoDB removes a large percentage of repetitive reads and shifts the performance profile from disk/network-bound to memory-bound. The interview-relevant point is that caching is not a generic speed feature; it is a consistency model choice. You define staleness windows, key design, TTL policy, and fallback behavior under cache miss, cache outage, and partial invalidation. If those are undefined, cache introduces correctness bugs faster than it introduces performance gains.

In real systems, caching works when you align key granularity with access patterns and update frequency. Product catalog metadata with minute-level freshness tolerance is easy; user balances or inventory are dangerous because stale reads can produce business errors. Engineers often overestimate hit ratio by benchmarking steady-state traffic and underestimate cold-start effects, deploy flushes, and hot-key skew. Production quality means monitoring hit ratio by keyspace, p95/p99 Redis latency, miss storm rates, and source-of-truth amplification after cache failures.

# 111. What are caching strategies?
Caching strategies are really data ownership and write-path decisions. Cache-aside is most common in Spring Boot services because it keeps the database as source of truth and avoids hidden write semantics in the cache layer. The service reads Redis first, then DB on miss, then populates Redis. This pattern is simple but creates race windows: concurrent misses can stampede the DB, and stale values can survive if invalidation is delayed.

Read-through pushes miss handling into the cache provider and can simplify app code, but operationally it hides data-fetch logic behind cache infrastructure and is less flexible in mixed persistence stacks. Write-through updates cache and DB in the same logical operation, reducing stale reads for hot keys, but increases write latency and fails poorly if one side succeeds and the other does not. Write-behind (write-back) improves write throughput by acknowledging before persistence, but now durability depends on queueing guarantees and replay correctness; if not backed by durable logs (Kafka, local WAL semantics), you risk silent data loss.

A practical architecture often combines strategies by data class: cache-aside for most query paths, targeted write-through for highly contended read-after-write entities, and no cache for correctness-critical low-volume data.

# 112. What are caching invalidation strategies?
Invalidation is where most caching designs fail. TTL-only invalidation is operationally cheap and limits worst-case staleness, but it does not guarantee correctness after writes; it just bounds inconsistency duration. Event-driven invalidation (for example, publishing domain events to Kafka after successful PostgreSQL commit, then evicting Redis keys in consumers) improves freshness but introduces distributed delivery concerns: ordering, duplicate events, consumer lag, and backpressure.

The reliable pattern in production is commit state first, then emit invalidation via transactional outbox, then process asynchronously with idempotent consumers. This avoids “DB committed but event lost” gaps. For complex aggregates, deleting keys by prefix is expensive and error-prone; versioned keys are safer. Instead of deleting all dependent keys, increment a version token and let readers naturally move to new keys while old entries expire. This reduces synchronized invalidation spikes and avoids large key scans in Redis.

Engineers frequently misunderstand that invalidation correctness is not binary. You define acceptable inconsistency per endpoint. Some reads can tolerate seconds of drift; others need read-your-write guarantees and should bypass cache or use short-lived session-local caches.

# 113. What are cache displacement strategies?
Displacement (eviction) strategy determines which keys die when memory is constrained, and this directly affects downstream database load. LRU works for temporal locality but can collapse under scan-heavy workloads that churn recent sets. LFU is better when stable hot keys dominate, but reacts slower to abrupt traffic shifts. TTL-based expiration controls lifetime but is not enough as a memory policy because synchronized expirations create miss storms.

In Redis, eviction policy is a capacity management decision, not a default setting to ignore. `allkeys-lfu` is often a strong baseline for mixed workloads, while `volatile-*` policies only work if every relevant key has TTL. If developers forget TTL on part of the keyspace, Redis can retain non-expiring low-value keys and evict exactly the keys you care about. A production setup includes: explicit maxmemory, jittered TTLs to avoid synchronized expiry, key-size discipline to prevent memory fragmentation, and per-keyspace observability. Displacement mistakes usually appear as sudden DB QPS spikes and latency regressions during traffic peaks, not during normal load tests.

# 114. What architecture styles exist in system design?
Architecture style is the failure and scaling model of your system, not just code organization. Layered monolith, modular monolith, microservices, event-driven systems, and service-oriented hybrids all exist because they optimize different constraints: team structure, release cadence, fault isolation, and consistency boundaries.

A modular monolith with clear domain modules can outperform poorly designed microservices for years because it avoids network hops, distributed transactions, and cross-service schema coordination. Microservices become rational when domain boundaries are stable enough and organization size requires independent deployability. Event-driven architecture is usually introduced to decouple write paths and absorb load with asynchronous processing, but it adds delivery semantics work (idempotency, replay, compaction strategy, schema evolution). In real companies, the architecture is almost always hybrid: synchronous gRPC/HTTP for request-time workflows, Kafka/RabbitMQ for asynchronous integration, and edge routing with Nginx/Envoy or API Gateway.

# 115. Why was monolith architecture created earlier than microservice architecture?
Monolith came first because it matched the operational reality of earlier infrastructure. Without Kubernetes-grade orchestration, mature service discovery, cheap observability, and reliable automated deployment pipelines, splitting a system into many independently deployed processes was operationally expensive and failure-prone. A single deployable binary with one transactional PostgreSQL database gave simpler correctness guarantees, simpler debugging, and predictable performance.

Microservices are not a conceptual invention that was “missing”; they were often impractical at scale before modern platform tooling. Once container scheduling, self-healing, centralized logging, tracing, and service mesh capabilities became mainstream, the cost of distribution dropped enough for organizations to trade complexity for organizational and scaling benefits.

# 116. What are the pros and cons of monolithic architecture?
A monolith is strong when you need fast feature iteration with a small team and cohesive domain logic. In-process calls are cheap, transactions are straightforward, and refactoring across modules is easier because changes are atomic within one codebase and one release artifact. For many Spring Boot systems, this means better development throughput and fewer operational failure modes than premature microservices.

Its limits appear when scaling and ownership diverge. A single hot module can force full-system horizontal scaling. Deployment risk is coupled: one bad release affects all capabilities. Build/test cycles grow with codebase size, and dependency conflicts accumulate. Operationally, fault isolation is weak; CPU/memory leaks in one subsystem degrade unrelated endpoints. The practical mitigation is not immediate microservices, but modular monolith discipline: strict module boundaries, clear package ownership, and controlled inter-module dependencies.

# 117. What are the pros and cons of microservice architecture?
Microservices provide organizational scalability: teams own services end-to-end, release independently, and scale high-load domains separately. They also improve blast-radius control when dependencies fail, assuming resilience controls exist (timeouts, retries with budgets, circuit breakers, bulkheads). For heterogeneous workloads, teams can choose PostgreSQL, MongoDB, or specialized storage per service.

The downside is that distributed systems complexity becomes the default tax. You replace local method calls with unreliable networks, local transactions with eventual consistency, and single-process debugging with cross-service tracing. Many failures are emergent: retry storms, queue lag, consumer rebalances, thundering herds after outage recovery, and partial availability during downstream degradation. Teams often underestimate operational staffing and platform maturity required for microservices. Without strong observability and platform standards, microservices degrade delivery speed instead of improving it.

# 118. What types of communication exist between microservices?
In production, communication splits into synchronous and asynchronous flows, and most systems need both. Synchronous HTTP/gRPC is used when caller needs immediate decision or user-facing response. gRPC gives stronger contracts and efficient binary serialization; HTTP/JSON is easier for interoperability. The cost is temporal coupling: upstream latency and failures directly propagate to callers.

Asynchronous communication via Kafka or RabbitMQ decouples producer and consumer availability and smooths spikes with buffering. Kafka is typically chosen for high-throughput event streaming, replay, and partitioned ordering guarantees; RabbitMQ is often chosen for flexible routing topologies and task/work queue patterns with acknowledgement semantics. Async messaging introduces different complexity: idempotent handlers, poison-message handling, schema compatibility, dead-letter strategy, and monitoring lag as an SLO signal.

A common robust pattern is synchronous command for immediate validation + asynchronous events for side effects. This limits critical path latency while preserving decoupled downstream processing.

# 119. What is the problem of distributed transactions? What patterns help to solve this problem?
The core problem is that one business operation crosses multiple autonomous data stores, but global ACID across service boundaries is either unavailable or too costly for availability and latency. Two-phase commit coordinates atomicity but adds blocking and coordinator failure risk; at scale it often conflicts with high-availability goals.

Production systems usually adopt eventual consistency and explicit failure handling. Saga decomposes a business transaction into local transactions with compensating actions. Orchestrated saga centralizes flow control; choreographed saga distributes control through events. Orchestration is easier to reason about but can become a central dependency; choreography scales organizationally but becomes harder to debug as event graphs grow.

Transactional outbox solves the dual-write problem by writing domain state and an outbox row in one local DB transaction, then publishing asynchronously (often via Debezium/Kafka). This shifts the problem from atomic cross-service commit to reliable event delivery plus idempotent consumption. Interview-quality answer should mention that “exactly once” is usually narrowed to exactly-once effect per consumer via idempotency keys, dedup tables, or version checks.

# 120. What is the Service Registry pattern?
Service Registry addresses dynamic endpoint resolution in distributed deployments where instances are ephemeral. In Kubernetes, Pods churn, IPs change, and hardcoded addresses are invalid quickly. Registry/discovery lets clients resolve healthy instances at call time or via periodically refreshed local caches.

In Spring Cloud ecosystems, client-side discovery and load balancing were common (Eureka/Consul style). In Kubernetes-native setups, cluster DNS and Services often provide the discovery plane, while Envoy or ingress components handle traffic distribution and resilience policies. Failure scenarios are mostly around stale membership and control-plane degradation: dead instances receiving traffic due to delayed health propagation, or discovery outages causing cascading client failures if caching and fallback are poor.

Operationally, discovery must be paired with aggressive health checks, outlier detection, and sane TTLs; registry correctness directly influences call success rates.

# 121. What is the SideCar pattern?
Sidecar means colocating an auxiliary process with the application instance to externalize cross-cutting runtime concerns: mTLS, retries, circuit breaking, telemetry export, and policy enforcement. In Kubernetes this usually means an additional container in the same Pod, sharing network namespace and lifecycle coupling.

The practical win is consistency: teams avoid reimplementing connection policy in every Spring Boot service. Envoy sidecars enforce timeouts, retries, and mutual TLS uniformly and can emit rich traffic metrics without code changes. The cost is resource overhead and operational complexity. Every Pod now has more CPU/memory footprint and one more failure mode. Misconfigured sidecars can introduce latent outages across many services simultaneously, so config rollout safety matters as much as app rollout safety.

# 122. What is the Circuit Breaker pattern?
Circuit breaker is a load-shedding and fault-containment mechanism for remote calls. When downstream error rate or latency exceeds thresholds, breaker opens and fails fast instead of consuming caller threads and connection pools on doomed requests. This prevents localized dependency failures from becoming full-system collapse.

In Spring Boot, Resilience4j circuit breakers are typically combined with time limiters, retries with bounded budgets, and bulkheads. The production nuance is parameter tuning: low thresholds cause false opens during transient spikes, high thresholds react too late. Half-open probes are necessary but can re-trigger failures if probe concurrency is not limited. Fallbacks must be business-safe; returning stale Redis data can be acceptable for product metadata but dangerous for balances or authorization decisions.

# 123. What is the CQRS pattern?
CQRS separates write-side invariants from read-side query optimization. The write model enforces business rules and consistency boundaries; the read model is shaped for query performance and can be denormalized aggressively. This is attractive when read/write workloads diverge or when query patterns become too expensive on normalized transactional schemas.

In production, writes often persist to PostgreSQL and emit events; projections build read models in PostgreSQL read tables, MongoDB documents, or Redis materializations. The important operational reality is projection lag and rebuild strategy. If event consumers fall behind, read freshness degrades. If projection schema changes, rebuild can be expensive and must be planned (replay window, backfill throughput, dual-read migration). CQRS is valuable when query complexity and scale justify this operational burden; otherwise it is unnecessary architecture tax.

# 124. What is the Anti-Corruption Layer pattern?
Anti-Corruption Layer is a protective boundary between your domain model and an external model you do not control (legacy system, partner API, acquired platform). Without ACL, foreign semantics leak into internal code, making your domain harder to evolve and coupling release cadence to external contract instability.

In practice, ACL is implemented as dedicated adapters that translate payload shape, identifiers, error semantics, and business meaning. It also handles protocol-level mismatch (REST vs gRPC), retry/idempotency policy, and backward compatibility shims. The overhead is extra mapping and latency, but it localizes complexity and prevents systemic contamination. Teams frequently skip ACL to move faster, then pay later with fragile domain code full of external exceptions and legacy states.

# 125. What is the Strangler pattern?
Strangler is an incremental replacement strategy for legacy systems where full rewrite risk is unacceptable. Traffic is progressively shifted from legacy endpoints to new services, usually behind a stable routing layer (Nginx, Envoy, or API Gateway), while both systems coexist.

The hard part is not routing; it is behavioral and data consistency during coexistence. You may need dual writes, CDC synchronization, or reconciliation jobs to keep old and new data aligned. Drift between implementations is common: subtle validation differences, ordering behavior, timeout handling, and side effects. Production migration requires parity testing, traffic shadowing, feature flags, and clear rollback paths. Strangler succeeds when migration slices are small, observable, and reversible.

# 126. What is the API Gateway pattern?
API Gateway centralizes edge concerns and prevents every internal service from becoming an internet-facing policy engine. It handles authentication integration, TLS termination, rate limiting, routing, request shaping, and sometimes response aggregation. This reduces repeated boilerplate across services and gives a controllable choke point for security and traffic governance.

The failure mode is turning gateway into a monolithic bottleneck with business logic accumulation. If too much orchestration is pushed into gateway, releases become high-risk and team coupling returns at the edge. In production, gateway should enforce platform concerns and light composition, while core business workflows remain in domain services. High availability requires horizontal scaling, statelessness, and careful timeout budget partitioning so gateway does not consume the entire request latency envelope.

# 127. What is the Backend for Frontend pattern?
BFF creates client-specific backend surfaces because mobile, web, and partner clients rarely share identical payload and interaction needs. A single generic backend often overfetches for one client and underfetches for another, causing either extra network round trips or oversized responses.

A BFF can aggregate multiple downstream calls, tailor response shape, and apply client-specific caching and resilience policy. For example, mobile BFF may prioritize payload size and tolerate stale recommendation blocks, while web BFF may prioritize richer page composition. The tradeoff is duplication risk and boundary confusion: business rules should remain in domain services, not drift into each BFF. BFF works best when it is a composition layer with strict ownership and contract tests against downstream APIs.

# 128. What is DDD?
DDD is primarily a boundary-definition tool for complex domains. Its value in distributed systems is that bounded contexts provide a principled way to split service ownership, data models, and language. Without these boundaries, microservices often fracture by technical layers instead of business capabilities, leading to chatty inter-service calls and unclear ownership.

In production, DDD is useful when domain complexity is real: conflicting terms across departments, intricate invariants, and frequent rule changes. Aggregates define transactional consistency boundaries; domain events expose state changes across contexts without shared tables. The common misunderstanding is treating DDD as mandatory heavy ceremony. For simple CRUD domains, full tactical DDD patterns can be overkill. The key is strategic design: clear context boundaries, explicit ownership, and integration contracts that minimize coupling.

# 129. Why should each microservice have its own database?
Database-per-service is less about technology freedom and more about autonomy and integrity of boundaries. If multiple services share the same PostgreSQL schema, they create hidden coupling through direct table access, implicit joins, and migration conflicts. Independent deployment then becomes fiction because schema changes require cross-team coordination and can break unknown consumers.

Owning a database forces interaction through explicit APIs or events, which makes contracts visible and evolvable. It also allows each service to choose storage aligned with workload: PostgreSQL for transactional consistency, MongoDB for document-centric read/write patterns, Redis for ephemeral state. The cost is deliberate: cross-service joins disappear, reporting becomes pipeline-driven, and consistency becomes eventual for many workflows. Teams mitigate this with read models, CDC-fed analytics stores, and idempotent integration flows.

# 130. What are the main problems of HTTP protocol?
HTTP is effective for synchronous request/response, but production distributed workflows expose its limits. First, temporal coupling: caller and callee must both be available now, so dependency failure or latency directly degrades upstream SLIs. Second, retry ambiguity: timeout does not tell you whether server applied side effects, so naive retries can duplicate operations unless idempotency keys are enforced. Third, backpressure is weak compared to broker buffering; under spikes, HTTP fan-out chains can exhaust threads and connection pools quickly.

Multi-hop HTTP architectures also amplify tail latency. Even when average latency per hop is low, p99 compounds across service chains. This is why teams enforce strict timeout budgets, bounded retries, circuit breakers, and often shift non-immediate work to Kafka or RabbitMQ. HTTP is still the right tool for interactive paths and simple integration, but it is not a durable workflow substrate. Interview-grade answers should make this distinction clearly: use HTTP for synchronous decisions, use messaging for decoupled, replayable, failure-tolerant processing.
