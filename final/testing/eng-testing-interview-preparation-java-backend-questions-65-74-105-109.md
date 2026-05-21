# Testing Interview Preparation (Questions 65–74, 105–109)

## Table of Contents
- [65. What is unit testing and why is it needed?](#65-what-is-unit-testing-and-why-is-it-needed)
- [66. How does a unit test differ from an integration test?](#66-how-does-a-unit-test-differ-from-an-integration-test)
- [67. What libraries are most often used for testing in Java? (JUnit, Mockito)](#67-what-libraries-are-most-often-used-for-testing-in-java-junit-mockito)
- [68. What is a test fixture (a set of data for tests)?](#68-what-is-a-test-fixture-a-set-of-data-for-tests)
- [69. How to properly name test methods?](#69-how-to-properly-name-test-methods)
- [70. What are mock, stub, and spy?](#70-what-are-mock-stub-and-spy)
- [71. In what cases is it better to use Mockito?](#71-in-what-cases-is-it-better-to-use-mockito)
- [72. What are parameterized tests?](#72-what-are-parameterized-tests)
- [73. What is code coverage and how is it measured?](#73-what-is-code-coverage-and-how-is-it-measured)
- [74. Why is it important to write tests before or along with the code (TDD)?](#74-why-is-it-important-to-write-tests-before-or-along-with-the-code-tdd)
- [105. What are Integration tests? What is the difference between integration tests and unit tests?](#105-what-are-integration-tests-what-is-the-difference-between-integration-tests-and-unit-tests)
- [106. What are TestContainers? How are they used in Integration tests?](#106-what-are-testcontainers-how-are-they-used-in-integration-tests)
- [107. How to load an ApplicationContext in Integration tests?](#107-how-to-load-an-applicationcontext-in-integration-tests)
- [108. How to mock external API calls in integration tests?](#108-how-to-mock-external-api-calls-in-integration-tests)
- [109. How to handle security issues in integration tests?](#109-how-to-handle-security-issues-in-integration-tests)

# 65. What is unit testing and why is it needed?
Unit tests are fast, deterministic checks of business behavior at method/class level with process-local dependencies. In Java/Spring teams, they are primarily used to protect domain rules and decision logic from regressions without paying the cost of Spring context startup, database IO, Kafka brokers, or HTTP stack. The practical value is feedback speed and failure localization: when a unit test fails, you usually know exactly which rule broke.

A common misconception is that unit tests are just for high coverage. In production teams, the better metric is whether critical business invariants are guarded. If pricing, idempotency key handling, retry policy, or validation logic changes silently, you want failures in seconds on developer machines and in CI pull requests. Unit tests should validate behavior and externally observable outcomes, not private method structure. Over-coupling tests to implementation details creates brittle suites that block refactoring.

# 66. How does a unit test differ from an integration test?
The difference is confidence scope, not class count. A unit test isolates one logical unit and controls collaborators (usually with stubs/mocks), so it is fast and precise but cannot prove that configuration, wiring, SQL mappings, serialization, transactions, security filters, or broker integrations actually work. An integration test validates real interaction between components across boundaries: Spring Boot wiring, repositories against PostgreSQL/MongoDB, Redis behavior, HTTP contracts, Kafka producers/consumers.

In delivery pipelines, you need both because they fail in different ways. Unit tests catch local logic regressions quickly; integration tests catch system-boundary failures that unit tests cannot see. Teams that rely only on unit tests miss production bugs from Flyway migrations, Jackson mappings, transaction isolation, and misconfigured beans. Teams that rely only on integration tests suffer slow feedback and flaky suites. The practical balance is a test pyramid/slice strategy with unit tests as bulk and targeted integration tests at high-risk seams.

# 67. What libraries are most often used for testing in Java? (JUnit, Mockito)
JUnit 5 is the baseline test framework in modern Java: lifecycle hooks, extensions, parameterized tests, tagging, nested tests, and assertions. Mockito is the standard mocking library for isolating collaborators in unit tests. In Spring Boot projects, these are usually combined with AssertJ/Hamcrest for richer assertions, Spring Boot Test for context-driven tests, MockMvc for MVC-layer tests, WireMock for HTTP dependency simulation, and TestContainers for real ephemeral infrastructure.

The production point is tooling composition: JUnit orchestrates execution, Mockito isolates boundaries, Spring Boot Test provides realistic app wiring where needed, and TestContainers closes the realism gap for database/broker dependencies in CI. Problems start when teams use Mockito for everything, including repository/transaction behavior that should be validated against real PostgreSQL or MongoDB.

# 68. What is a test fixture (a set of data for tests)?
A fixture is controlled test state: input objects, persisted rows/documents, expected outputs, and environment configuration required to run a test deterministically. In backend systems, fixture quality determines test reliability more than assertion count. Bad fixtures are usually shared mutable global objects, hidden ordering dependencies, or random timestamps/UUIDs without control, all of which produce flaky behavior across parallel CI runs.

Production-grade fixtures are explicit and minimal. Use builders/factories with sensible defaults, isolate per-test data, and reset state reliably (transaction rollback, truncate, or container recreation depending on scope). Prefer test data that reflects business cases (duplicate key conflicts, timezone edges, nullability constraints, race-prone states) instead of synthetic “happy path only” objects. Deterministic clocks and deterministic IDs are often necessary for stable assertions.

# 69. How to properly name test methods?
A strong test name describes behavior, condition, and expected outcome in domain terms, not technical noise. Names like `shouldRejectOrderWhenCreditLimitExceeded` or `returns409WhenDuplicateIdempotencyKey` are useful because they explain failure impact immediately in CI logs. Names such as `testCreate` or `whenServiceThenOk` are low-signal and expensive during incident triage.

In mature codebases, naming consistency matters because test reports are operational artifacts. Good names reduce debugging time, especially in parallel pipelines where hundreds of tests run concurrently. Use method naming conventions that emphasize business behavior and failure reason; avoid encoding implementation details that will change during refactoring.

# 70. What are mock, stub, and spy?
A stub provides predefined responses and is used to drive the code under test through specific branches. A mock is behavior-verification oriented: you assert interactions such as “notification publisher called once with this event.” A spy wraps a real object and allows selective stubbing/verification while preserving most real behavior.

In practice, confusion here leads to poor tests. Over-verifying interactions with mocks often tests implementation choreography instead of business outcome, making tests brittle. Spies are useful for hard-to-replace components or partial behavior checks but can hide side effects if engineers forget real methods still execute. In Spring code, prefer stubs/fakes for stable behavior tests, and use interaction verification only where side effects are the requirement (e.g., exactly one outbox publish attempt under given condition).

# 71. In what cases is it better to use Mockito?
Mockito is best at process-boundary isolation in unit tests: external API clients, message publishers, clock providers, random/UUID generators, and expensive collaborators where real execution is irrelevant to the decision logic being tested. It is also effective when you need to verify interaction contracts such as retry count, fallback invocation, or compensating action trigger.

It becomes harmful when used to mock what should be integration-validated: JPA repositories, transaction manager behavior, SQL constraints, serialization, security filters, or Kafka producer configuration. Mocking these in “integration-like” tests creates false confidence. A common anti-pattern is `@SpringBootTest` plus mocked repositories; this tests almost nothing meaningful and still pays context startup cost. Use Mockito to isolate logic; use TestContainers + real dependencies for boundary correctness.

# 72. What are parameterized tests?
Parameterized tests run the same behavioral assertion across multiple input/output sets. In JUnit 5, this is a high-leverage way to harden validation rules, mapping logic, normalization, and edge-case handling without duplicating test code. They are especially useful for boundary values, null/empty combinations, currency/precision handling, and locale/timezone-sensitive inputs.

The production benefit is compact coverage of rule surfaces that often regress. The risk is unreadable mega-parameter sets where failures are hard to diagnose. Keep datasets intentional, label cases clearly, and avoid turning parameterized tests into data dumps disconnected from business semantics.

# 73. What is code coverage and how is it measured?
Coverage measures executed code proportion during tests (line, branch, instruction metrics; typically via JaCoCo in Java pipelines). It is a lagging indicator of test reach, not proof of correctness. High line coverage can coexist with poor assertions, missing edge cases, and no integration confidence.

In production CI (GitHub Actions/Jenkins), coverage thresholds are useful as guardrails against untested growth, but treating the number as the target drives anti-patterns: trivial tests for getters, assertion-light tests, and overfitting to metrics. Branch coverage is usually more informative than line coverage for decision-heavy business logic. Teams should pair coverage with mutation testing where feasible, critical path test audits, and defect escape analysis.

# 74. Why is it important to write tests before or along with the code (TDD)?
Writing tests before or during implementation forces explicit behavior contracts early: inputs, outputs, failure modes, and edge handling. In production teams, this reduces late discovery of ambiguous requirements and lowers redesign cost. TDD is most effective on domain logic and protocols where correctness matters, not as dogma for every trivial class.

The practical advantage is design pressure: code becomes more modular and easier to test because dependencies are explicit. The practical failure mode is ritualistic TDD that optimizes for test count instead of meaningful behavior, producing fragile tests coupled to evolving internals. Mature teams use TDD selectively: high-risk logic first, then targeted integration tests to validate real wiring, persistence, and infrastructure behavior.

# 105. What are Integration tests? What is the difference between integration tests and unit tests?
Integration tests verify that independently developed components work together in realistic runtime conditions: Spring context wiring, HTTP serialization, persistence mappings, transaction boundaries, messaging, and security integration. The key distinction from unit tests is that collaborators are real (or close to real) at system boundaries. Unit tests maximize speed and precision; integration tests maximize confidence in wiring and infrastructure behavior.

In Spring Boot, integration test scope should be deliberate. Use narrow slices (`@DataJpaTest`, `@WebMvcTest`) where appropriate and full `@SpringBootTest` only for cross-layer flows that justify startup cost. Flakiness usually comes from shared state, non-deterministic timing, external dependency instability, and async race conditions. Reliable integration suites require deterministic fixture setup, explicit cleanup, stable clocks where needed, and bounded asynchronous polling (Awaitility-style) instead of `Thread.sleep()`.

# 106. What are TestContainers? How are they used in Integration tests?
TestContainers provides ephemeral Docker-backed dependencies for tests: PostgreSQL, MongoDB, Redis, Kafka, RabbitMQ, etc. Instead of mocking persistence or brokers, tests run against real services with production-like protocol behavior. This significantly increases confidence in migrations, SQL dialect behavior, index usage assumptions, transaction semantics, and message serialization.

In real CI/CD, TestContainers improves reproducibility by giving isolated per-run infrastructure, reducing “works on my machine” drift. The tradeoff is startup overhead and Docker dependency; cold starts can slow pipelines, especially with Kafka containers. Teams mitigate this with test slicing, reusable containers when safe, image pre-pull in CI, and separating smoke vs full integration stages. Reliability requires careful port/config injection via Spring dynamic properties and deterministic test ordering assumptions (ideally none).

# 107. How to load an ApplicationContext in Integration tests?
Load only as much context as the test needs. `@SpringBootTest` loads the full application and gives high realism but high startup cost and broader flake surface. For targeted concerns, prefer slices: `@WebMvcTest` for controller + MVC stack, `@DataJpaTest` for repository mapping/queries, and focused configuration imports for specific beans. Excessive full-context usage is one of the biggest test suite slowdowns in Spring projects.

For externalized dependencies, wire dynamic container endpoints using `@DynamicPropertySource` so each test run gets isolated configuration. Avoid static global mutable config that leaks between tests. Context caching in Spring can speed execution, but hidden cross-test coupling appears if tests mutate singleton state. If tests require context dirtiness, use it sparingly because `@DirtiesContext` can explode runtime.

# 108. How to mock external API calls in integration tests?
For HTTP dependencies, prefer protocol-level simulation with WireMock (or MockWebServer) over mocking internal client classes. This validates serialization, headers, timeout handling, retry behavior, and error mapping at real HTTP boundaries. Mocking the Feign/WebClient bean directly in integration tests bypasses exactly the behavior likely to fail in production.

Good integration tests model dependency scenarios explicitly: 200 with valid payload, malformed payload, 4xx business errors, 5xx transient errors, slow responses/timeouts, and idempotent retry handling. Keep stubs deterministic and local to each test to avoid inter-test interference. For contract-sensitive integrations, pair WireMock stubs with consumer/provider contract testing in CI to detect drift early.

# 109. How to handle security issues in integration tests?
Security tests should validate actual filter-chain behavior, token parsing, role/authority mapping, and method-level access control rather than bypassing security globally. In Spring Security, use test support for authenticated principals when focusing on business flows, but keep dedicated tests that run full authentication/authorization paths, including failure cases (401/403).

For JWT/OAuth flows, use deterministic test keys or local identity stubs; avoid calling real identity providers in CI. Verify security headers, CSRF behavior where applicable, and endpoint exposure rules. In distributed systems, also test service-to-service trust assumptions (mTLS/proxy headers) at boundaries represented in your test environment. A common anti-pattern is disabling security for all integration tests, which yields fast green builds and production auth failures.
