# Tier 1 — обязательно знать идеально

То, на чём чаще всего валят.

---

## Java Core

**Модуль:** [Модуль 1 — Core Java + Concurrency + JVM](Модуль%201%20—%20Core%20Java%20+%20Concurrency%20+%20JVM.md)

### Темы

- HashMap internal
- `equals` / `hashCode`
- HashSet
- TreeMap + Comparator / Comparable
- Stream API
- `map` vs `flatMap`
- Optional
- immutable objects
- Thread
- `synchronized`
- `volatile`
- atomics
- ExecutorService
- Future vs CompletableFuture
- concurrent collections
- Java Memory Model
- GC

### Что нужно уметь объяснить

- как HashMap ресайзится
- что такое bucket
- почему `hashCode` важен
- почему mutable key ломает map
- difference between synchronized and concurrent collections
- почему CompletableFuture лучше Future
- что делает `volatile` и **чего НЕ делает**
- race condition
- deadlock
- thread pool starvation

### Production focus

- почему parallel stream опасен в backend
- почему common fork-join pool может убить latency
- почему нельзя создавать thread руками в web app

---

## SQL + Transactions

**Модуль:** [Модуль 2 — Databases (SQL + JDBC + Migrations)](<Модуль%202%20—%20Databases%20(SQL%20+%20JDBC%20+%20Migrations).md>)

### Темы

- JOINs
- indexes
- constraints
- transactions
- ACID
- isolation levels
- optimistic / pessimistic lock
- `EXPLAIN ANALYZE`
- normalization
- migrations

### Что реально спрашивают

- difference between `READ_COMMITTED` and `REPEATABLE_READ`
- phantom read
- why optimistic lock scales better
- what happens without index
- why N+1 destroys DB
- why Flyway preferred in many teams

### Production focus

- index not used because of function / cast
- transactions too long
- lock contention
- deadlocks
- migration rollback problems

---

## Spring

**Модуль:** [Модуль 3 — Spring (Core + Hibernate)](<Модуль%203%20—%20Spring%20(Core%20+%20Hibernate).md>)

Критично.

### Темы

- IoC
- DI
- bean lifecycle
- scopes
- AOP
- proxies
- `@Transactional`
- cyclic dependencies
- prototype inside singleton

### Ты должен понимать

- Spring — в основном **proxy framework**
- `@Transactional` работает через proxy
- self-invocation ломает transaction
- singleton bean shared between threads
- prototype inside singleton **не обновляется** автоматически

### Production focus

- transaction not applied because method `private`
- lazy loading outside transaction
- circular dependency = bad design usually

---

## Hibernate / JPA

**Модуль:** [Модуль 3 — Spring (Core + Hibernate)](<Модуль%203%20—%20Spring%20(Core%20+%20Hibernate).md>)

### Темы

- ORM
- entity mapping
- N+1
- cache levels
- lazy / eager loading

### Самое важное

- persistence context
- dirty checking
- why EAGER bad by default
- fetch join
- entity lifecycle

### Production problems

- N+1
- huge persistence context
- memory leak in batch operations
- `LazyInitializationException`

---

## REST + Security

**Модули:**

- [Модуль 6 — Web Fundamentals (HTTP + REST)](<Модуль%206%20—%20Web%20Fundamentals%20(HTTP%20+%20REST).md>)
- [Модуль 4 — Web (Spring MVC + Security)](<Модуль%204%20—%20Web%20(Spring%20MVC%20+%20Security).md>)

### Темы

- REST
- HTTP methods
- idempotency
- status codes
- JWT
- OAuth2
- Authentication vs Authorization
- CORS
- CSRF
- SQL injection
- XSS

JWT и gateway — твой плюс.

### Что надо уметь

- структура JWT
- why JWT cannot be revoked easily
- refresh token flow
- stateless auth
- why HTTPS mandatory with JWT

### Production focus

- token leakage
- CORS misconfiguration
- storing JWT in `localStorage`
- brute force protection

---

## Docker

**Модуль:** [Модуль 7 — Docker + Integration Testing](Модуль%207%20—%20Docker%20+%20Integration%20Testing.md)

### Темы

- image
- container
- layers
- volumes
- networks
- docker compose
- multi-stage builds

### Нужно понимать

- image immutable
- container = runtime instance
- why multi-stage reduces size
- difference between bind mount and volume

### Production problems

- huge images
- secrets inside image
- container restart loops
- state inside container

---

## Microservices

**Модуль:** [Модуль 8 — Caching + System Design](Модуль%208%20—%20Caching%20+%20System%20Design.md)

Очень важно для твоего стека.

### Темы

- monolith vs microservices
- service registry
- API gateway
- distributed transactions
- circuit breaker
- communication patterns
- CQRS
- DDD basics

### Говори как инженер

**НЕ:** «микросервисы масштабируются»

**А:** «микросервисы увеличивают operational complexity ради независимого deployment и bounded context isolation»

### Production focus

- network failures
- retries
- duplicate events
- idempotency
- partial failure
- eventual consistency

---

## Kafka

**Модуль:** [Модуль 9 — Messaging (Kafka + Brokers)](<Модуль%209%20—%20Messaging%20(Kafka%20+%20Brokers).md>)

Must have для твоего стека.

### Темы

- topic
- partition
- consumer group
- rebalance
- delivery semantics
- Kafka vs RabbitMQ
- pull model

### Ключевые production-вещи

- ordering only inside partition
- at least once = duplicates possible
- idempotent consumer
- rebalance pauses processing
- partition count affects parallelism

---

## MongoDB / NoSQL

**Модуль:** [Модуль 10 — NoSQL (Mongo + CAP)](<Модуль%2010%20—%20NoSQL%20(Mongo%20+%20CAP).md>)

Супер важно — здесь могут проверить сильно.

### Темы

- document model
- indexes
- sharding
- replication
- eventual consistency
- TTL indexes
- CAP theorem
- transactions in Mongo

### Ты должен понимать

- Mongo проектируется от **access patterns**
- embedding vs referencing
- why joins avoided
- why over-normalization bad in Mongo
- eventual consistency tradeoff

### Production focus

- missing indexes
- large documents
- hot shard
- unbounded arrays
- shard key selection
- TTL cleanup delay

---

# Tier 2 — знать нормально

Часто спрашивают, но глубина меньше.

---

## Testing

**Модуль:** [Модуль 5 — Testing + CI-CD](Модуль%205%20—%20Testing%20+%20CI-CD.md)

### Темы

- unit vs integration
- Mockito
- mock / stub / spy
- Testcontainers
- code coverage

---

## CI/CD

**Модуль:** [Модуль 5 — Testing + CI-CD](Модуль%205%20—%20Testing%20+%20CI-CD.md)

### Темы

- GitHub Actions
- pipeline stages
- artifact
- YAML

---

## Kubernetes

Базовое понимание (отдельного модуля в репозитории нет — добрать по документации).

### Темы

- pod
- deployment
- service
- configmap
- secret
- probes
- HPA

---

# Что почти гарантированно спросят именно тебя

Потому что это в твоих проектах:

1. JWT через API Gateway
2. Почему проверка JWT только в gateway недостаточна
3. Kafka vs Feign
4. Redis TTL корзина
5. MongoDB indexing
6. Microservice communication
7. Eureka / service discovery
8. Docker compose
9. Spring Security filter chain
10. `@Transactional` internal behavior
11. N+1
12. optimistic locking
13. idempotency

---

# Самые опасные темы (добить особенно)

Где чаще всего валят стажёров:

| Тема                              | Связь         |
| --------------------------------- | ------------- |
| transactions isolation            | Модуль 2      |
| JMM                               | Модуль 1      |
| `volatile`                        | Модуль 1      |
| CompletableFuture                 | Модуль 1      |
| optimistic vs pessimistic locking | Модуль 2      |
| Spring proxies                    | Модуль 3      |
| `@Transactional`                  | Модуль 3      |
| N+1                               | Модуль 2 + 3  |
| Kafka semantics                   | Модуль 9      |
| idempotency                       | Модуль 6 + 9  |
| CAP theorem                       | Модуль 10     |
| eventual consistency              | Модуль 8 + 10 |

---

# План на последние дни

1. Повторить **Java multithreading** (Модуль 1)
2. Повторить **transactions + isolation** (Модуль 2)
3. Повторить **Spring proxy / AOP** (Модуль 3)
4. Повторить **Kafka guarantees** (Модуль 9)
5. Повторить **Mongo indexes / sharding** (Модуль 10)
6. Прогнать **свои проекты** и подготовить объяснение архитектуры
7. Подготовить ответы на:
   - «самая сложная проблема»
   - «что бы улучшил»
   - «где были bottlenecks»

> Это любят спрашивать **больше**, чем сухую теорию.

---

## Чеклист готовности (самопроверка)

Перед собесом пройди по Tier 1 и отметь для себя:

- [ ] Java Core — могу объяснить HashMap, JMM, пулы потоков без шпаргалки
- [ ] SQL + Transactions — isolation levels, locks, N+1, индексы
- [ ] Spring — proxies, scopes, `@Transactional` pitfalls
- [ ] Hibernate — persistence context, lazy/eager, fetch join
- [ ] REST + Security — JWT, OAuth2, CORS, CSRF
- [ ] Docker — image vs container, volumes, multi-stage
- [ ] Microservices — trade-offs, failures, idempotency
- [ ] Kafka — partitions, semantics, rebalance
- [ ] MongoDB — schema design, sharding, CAP
- [ ] Проекты — 3 истории (проблема / улучшение / bottleneck)
