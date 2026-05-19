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
## Definition
Caching is storing a temporary copy of data in a faster layer (usually memory) closer to consumers than the source of truth.

## Why it exists
It reduces latency, backend load, and infrastructure cost. Without caching, high read traffic can saturate PostgreSQL or MongoDB long before CPU in application servers is exhausted.

## Production usage
Typical layered setup: in-process cache (Caffeine) for ultra-low latency + distributed cache (Redis) for cross-instance sharing. Use TTL, size limits, and metrics (hit ratio, evictions, stale-read rate).

## Problems and tradeoffs
Main tradeoff: speed vs freshness. Common failures: stale data, cache stampede, hot keys, and inconsistent invalidation after writes.

## Common technologies
Redis, Caffeine, Spring Cache abstraction, Nginx proxy cache, CDN edge caches.

# 111. What are caching strategies?
## Definition
Caching strategies define when cache is read/written and who is responsible for population.

## Why it exists
Different workloads need different consistency and write-latency behavior.

## Production usage
- Cache-aside: app reads cache, falls back to DB, then populates cache (most common in Spring Boot + Redis).
- Read-through: cache layer fetches from store on miss (less app code, tighter cache coupling).
- Write-through: write DB and cache synchronously (better read consistency, higher write latency).
- Write-behind: write cache first, persist asynchronously (high throughput, durability risk).

## Problems and tradeoffs
Cache-aside risks stale entries on missed invalidation. Write-through increases p99 write latency. Write-behind can lose data on crash unless backed by durable queue/log.

## Common technologies
Spring Cache, Redis, Kafka (for write-behind/event propagation), RabbitMQ.

# 112. What are caching invalidation strategies?
## Definition
Invalidation strategies control when cached entries are removed or refreshed so reads stay acceptably correct.

## Why it exists
A cache without invalidation eventually becomes wrong data at low latency.

## Production usage
- TTL/expire-after-write for bounded staleness.
- Event-driven invalidation: publish entity-change events to Kafka/RabbitMQ and evict/update related keys.
- Versioned keys: include version/timestamp in key to avoid deleting fan-out key sets.
- Write-path eviction: after successful DB commit, evict relevant cache keys.

## Problems and tradeoffs
Short TTL improves freshness but lowers hit ratio. Event-driven invalidation can lag or drop messages. Multi-key invalidation is hard with denormalized cached views.

## Common technologies
Redis TTL, Kafka topics for invalidation events, Spring @CacheEvict.

# 113. What are cache displacement strategies?
## Definition
Displacement (eviction) strategies decide which entries are removed when cache memory is full.

## Why it exists
Memory is finite; uncontrolled growth causes OOM or forced process restarts.

## Production usage
Use policies by access pattern:
- LRU for temporal locality.
- LFU for stable hot keys.
- TTL-aware policies for naturally expiring datasets.
In Redis choose explicit `maxmemory` and eviction policy (`allkeys-lru`, `allkeys-lfu`, etc.).

## Problems and tradeoffs
Wrong policy destroys hit ratio and shifts load back to DB. LFU adapts slower to sudden traffic shifts. LRU can evict infrequent but expensive-to-rebuild objects.

## Common technologies
Redis eviction policies, Caffeine size/weight-based eviction.

# 114. What architecture styles exist in system design?
## Definition
Architecture style is the high-level structural model of system decomposition and communication.

## Why it exists
It defines scalability model, deployment unit, failure isolation, and team ownership boundaries.

## Production usage
Common styles: layered monolith, modular monolith, microservices, event-driven architecture, SOA, serverless, and hexagonal/clean architecture (inside services). Real systems are hybrid: e.g., microservices + event-driven + API Gateway.

## Problems and tradeoffs
No style is universally best. More distribution increases fault tolerance options but adds network failures, observability complexity, and operational overhead.

## Common technologies
Spring Boot, Kubernetes, Kafka, Nginx/Envoy, PostgreSQL, MongoDB.

# 115. Why was monolith architecture created earlier than microservice architecture?
## Definition
Monolith means one deployable unit with tightly integrated modules and usually a shared database.

## Why it exists
Earlier infrastructure favored simplicity: limited cloud tooling, weak container orchestration, and expensive distributed-systems operations.

## Production usage
Monoliths were practical with single-host or small-cluster deployments, simpler debugging, and straightforward ACID transactions in one DB.

## Problems and tradeoffs
Microservices became feasible later due to Docker/Kubernetes, mature CI/CD, service discovery, cheap infrastructure automation, and stronger observability stacks.

## Common technologies
Earlier: app servers + single RDBMS. Modern transition enablers: Kubernetes, Spring Cloud, Envoy.

# 116. What are the pros and cons of monolithic architecture?
## Definition
A monolith packages all business modules into one runtime and one deployment artifact.

## Why it exists
Optimizes for development speed and operational simplicity at small-to-medium scale.

## Production usage
Good fit for early products or stable domains with one team. Single process calls are fast, transactional boundaries are simple, testing and local development are easier.

## Problems and tradeoffs
Independent scaling is impossible; one hotspot can force scaling the whole app. Large codebase coupling slows releases. Fault isolation is weak; one memory leak can degrade all domains.

## Common technologies
Spring Boot modular monolith, PostgreSQL, Redis, Nginx.

# 117. What are the pros and cons of microservice architecture?
## Definition
Microservices split system capabilities into independently deployable services aligned to business domains.

## Why it exists
It enables team autonomy, independent scaling, and failure isolation for large, evolving systems.

## Production usage
Each service owns its API and data model. Communication is via HTTP/gRPC and asynchronous messaging (Kafka/RabbitMQ). Deployments run on Kubernetes with centralized observability.

## Problems and tradeoffs
Distributed systems costs dominate: network failures, eventual consistency, duplicate data, contract drift, and operational complexity (tracing, retries, rate limits, security between services).

## Common technologies
Spring Cloud, Kubernetes, Envoy service mesh, Kafka, PostgreSQL/MongoDB per service.

# 118. What types of communication exist between microservices?
## Definition
Communication is either synchronous request/response or asynchronous event/message exchange.

## Why it exists
Different interactions need different latency, coupling, and reliability characteristics.

## Production usage
- Synchronous: HTTP/gRPC for immediate response requirements.
- Asynchronous: Kafka/RabbitMQ for decoupling, buffering, retries, and eventual consistency.
- Hybrid: sync command + async domain events.

## Problems and tradeoffs
Synchronous chains increase tail latency and cascading failure risk. Async messaging adds ordering, idempotency, deduplication, and schema-evolution concerns.

## Common technologies
REST over HTTP, gRPC, Kafka, RabbitMQ, Spring Cloud OpenFeign.

# 119. What is the problem of distributed transactions? What patterns help to solve this problem?
## Definition
Distributed transaction means one business action updates multiple services/data stores.

## Why it exists
With separate service databases, classic single-DB ACID no longer spans the full workflow.

## Production usage
2PC is rarely used at scale due to coordinator blocking, latency, and availability tradeoffs. Practical approach: eventual consistency with compensations.
Patterns:
- Saga (choreography via events or orchestration via coordinator).
- Transactional Outbox + CDC to atomically persist state change and publish event.
- Idempotent consumers + retries + deduplication keys.

## Problems and tradeoffs
Temporary inconsistency is expected. Compensation logic is complex and can fail. Exactly-once end-to-end is usually replaced by at-least-once + idempotency.

## Common technologies
Kafka, Debezium CDC, PostgreSQL outbox table, Spring state machine/orchestrators.

# 120. What is the Service Registry pattern?
## Definition
Service Registry is a dynamic directory where service instances register their network location and health status.

## Why it exists
In Kubernetes or autoscaled environments, instance IPs are ephemeral; hardcoded endpoints are invalid.

## Production usage
Providers register on startup and heartbeat. Consumers discover instances and load-balance client-side or via mesh/proxy. Kubernetes often uses built-in service discovery (DNS/Service) instead of external registries.

## Problems and tradeoffs
Stale registrations route traffic to dead instances. Registry outages affect discovery. Needs health checks, TTL, and resilient client caching.

## Common technologies
Eureka/Consul, Spring Cloud Netflix, Kubernetes Services/CoreDNS, Envoy.

# 121. What is the SideCar pattern?
## Definition
Sidecar runs a helper container/process alongside the main service instance to provide shared platform capabilities.

## Why it exists
It offloads cross-cutting concerns from business code.

## Production usage
Typical sidecar responsibilities: mTLS, retries/timeouts, traffic shaping, metrics/log shipping, local proxying. In Kubernetes, sidecars are colocated in one Pod (same network namespace).

## Problems and tradeoffs
Extra CPU/memory cost per Pod, more moving parts, and debugging path complexity (app -> sidecar -> network).

## Common technologies
Envoy sidecar in service mesh, Fluent Bit logging sidecar.

# 122. What is the Circuit Breaker pattern?
## Definition
Circuit Breaker prevents repeated calls to an unhealthy dependency after failure threshold is reached.

## Why it exists
Without it, retries amplify outages and exhaust thread pools/connections.

## Production usage
States: closed (normal), open (fail fast), half-open (probe recovery). Configure thresholds by error rate and slow-call rate, plus timeout and fallback behavior.

## Problems and tradeoffs
Aggressive thresholds cause false opens; loose thresholds delay protection. Fallbacks can hide incidents and serve degraded data longer than acceptable.

## Common technologies
Resilience4j with Spring Boot, Envoy/Nginx upstream protection.

# 123. What is the CQRS pattern?
## Definition
CQRS separates write model (commands) from read model (queries), often using different storage/view models.

## Why it exists
Read and write paths have different scaling and modeling needs.

## Production usage
Writes persist domain state; events update read projections optimized for query latency (e.g., denormalized PostgreSQL tables, MongoDB views, Redis materialized data).

## Problems and tradeoffs
Eventual consistency between write and read sides. More infrastructure, projection rebuild procedures, and schema/version management.

## Common technologies
Kafka event streams, PostgreSQL + MongoDB split, Spring Boot projection workers.

# 124. What is the Anti-Corruption Layer pattern?
## Definition
ACL is a translation boundary that isolates your domain model from external/legacy model semantics.

## Why it exists
Direct coupling to external contracts leaks external terminology/rules into internal domain and damages maintainability.

## Production usage
ACL maps protocols, payloads, and error semantics. It can be implemented as adapter service/module with DTO mapping and policy translation.

## Problems and tradeoffs
Adds latency and mapping code, but prevents widespread contamination from unstable or poor legacy contracts.

## Common technologies
Spring adapters, mapper layers, API clients with explicit contract translation.

# 125. What is the Strangler pattern?
## Definition
Strangler is an incremental migration pattern: route selected flows from legacy system to new services until legacy can be retired.

## Why it exists
Full rewrites are high-risk and long-running with delayed value.

## Production usage
Put routing layer (API Gateway/Nginx) in front, migrate endpoints/capabilities slice by slice, keep data sync during transition, and measure parity.

## Problems and tradeoffs
Dual-write or data-sync complexity, behavioral drift between old/new paths, and longer temporary architecture complexity.

## Common technologies
Nginx/Envoy routing, Kafka CDC sync, feature flags.

# 126. What is the API Gateway pattern?
## Definition
API Gateway is a single entry point for clients that routes requests to internal services and applies edge policies.

## Why it exists
It centralizes concerns that should not be duplicated in each service.

## Production usage
Gateway handles authN/authZ integration, rate limiting, request routing, TLS termination, response aggregation, and protocol translation.

## Problems and tradeoffs
Gateway can become bottleneck or single point of failure if not horizontally scaled. Over-centralization can create a “mega-gateway” with high change risk.

## Common technologies
Spring Cloud Gateway, Nginx, Envoy, Kong.

# 127. What is the Backend for Frontend pattern?
## Definition
BFF means separate backend endpoints per client type (web, mobile, partner), each optimized for that client’s data and interaction model.

## Why it exists
Different clients need different payload granularity, latency budgets, and release cadence.

## Production usage
BFF aggregates calls to internal services, shapes payloads, and can apply client-specific caching and auth flows.

## Problems and tradeoffs
More services to maintain, potential duplicated logic across BFFs, and risk of drifting business rules into BFF layer.

## Common technologies
Spring Boot BFF services, GraphQL BFF, API Gateway in front.

# 128. What is DDD?
## Definition
Domain-Driven Design is an approach to model software around business domains, bounded contexts, and explicit domain language.

## Why it exists
Large systems fail when technical structure ignores business boundaries and ownership.

## Production usage
Use bounded contexts to define service boundaries, aggregates to control invariants, and domain events for cross-context integration.

## Problems and tradeoffs
Requires strong domain collaboration and disciplined modeling. Over-applying tactical patterns to simple CRUD systems adds unnecessary complexity.

## Common technologies
Spring Boot per bounded context, Kafka for domain events, separate PostgreSQL/MongoDB schemas per context.

# 129. Why should each microservice have its own database?
## Definition
Database-per-service means each service exclusively owns its persistence and schema evolution.

## Why it exists
It enforces autonomy and prevents tight coupling through shared tables.

## Production usage
Other services access data through APIs/events, not direct SQL joins across services. Cross-service queries are built via read models, replication, or CQRS projections.

## Problems and tradeoffs
No cross-service ACID transaction, data duplication, and eventual consistency. Reporting across domains requires dedicated analytics/read pipelines.

## Common technologies
PostgreSQL per service, MongoDB per document-centric service, Kafka for data propagation.

# 130. What are the main problems of HTTP protocol?
## Definition
HTTP is a request/response protocol optimized for synchronous interactions, not guaranteed delivery workflows.

## Why it exists
Its model is simple and universal, but distributed backend workflows often need stronger decoupling and reliability semantics.

## Production usage
HTTP works well for low-latency queries and command calls with immediate outcomes. For long-running or bursty workloads, pair it with async messaging.

## Problems and tradeoffs
- Tight temporal coupling: caller and callee must be available at the same time.
- Cascading latency in multi-hop chains.
- No native durable queueing or replay.
- Retry ambiguity (did server execute before timeout?).
- Hard backpressure under spikes compared to broker buffering.

## Common technologies
HTTP/REST or gRPC for sync paths; Kafka/RabbitMQ for asynchronous durability and decoupling.
