# Module 5 — Testing + CI/CD — Questions 65–84

> Questions 65–84 from *Questions for Java Internship transfer.pdf* (unit vs integration testing, JUnit/Mockito, fixtures, TDD, CI/CD pipelines, artifacts, GitHub Actions, YAML, notifications). Related: `other/testing-questions-65-74.md`, `other/cicd-questions-75-84.md`.

## Table of contents

65. [What is unit testing and why is it needed?](#q65)
66. [How does a unit test differ from an integration test?](#q66)
67. [What libraries are most often used for testing in Java? (JUnit, Mockito)](#q67)
68. [What is a test fixture (a set of data for tests)?](#q68)
69. [How to properly name test methods?](#q69)
70. [What are mock, stub, and spy?](#q70)
71. [In what cases is it better to use Mockito?](#q71)
72. [What are parameterized tests?](#q72)
73. [What is code coverage and how is it measured?](#q73)
74. [Why is it important to write tests before or along with the code (TDD)?](#q74)
75. [What is CI/CD and why is it needed?](#q75)
76. [What is the difference between CI and CD?](#q76)
77. [What CI/CD tools do you know? (Jenkins, GitHub Actions, GitLab CI, TeamCity)](#q77)
78. [What does a simple CI pipeline for a Java project look like?](#q78)
79. [What stages are usually included in a pipeline? (build, tests, deploy)](#q79)
80. [What is a build artifact?](#q80)
81. [How to integrate unit tests into a CI pipeline?](#q81)
82. [How does GitHub Actions work?](#q82)
83. [What is YAML in the context of CI/CD?](#q83)
84. [How can build result notifications be implemented?](#q84)

---

<a id="q65"></a>

## 65. What is unit testing and why is it needed?

### Question Restatement
What is **unit testing**, and what problems does it solve in real projects?

### 1. Definition
**Unit testing** exercises a **small, isolated** piece of code—usually a single **class** or **method**—with **controlled inputs** and **asserted outcomes**. Dependencies are typically **replaced** with test doubles so the unit’s behavior is what you measure, not the database or the network.

### 2. Why it is needed
- **Regression safety** — behavior is locked before refactor or feature work spreads.
- **Fast feedback** — runs in milliseconds/seconds on a laptop or in CI.
- **Living documentation** — tests describe expected behavior in executable form.
- **Design pressure** — hard-to-test code often signals **tight coupling** or unclear boundaries.

### 3. Summary
Unit testing is a type of testing where individual components of an application, usually small units like methods or classes, are tested in isolation to verify that they work as expected.

The goal of unit testing is to ensure that each piece of business logic behaves correctly независимо от других частей системы, often by mocking dependencies such as databases or external services.

Unit tests are typically fast, deterministic, and focused on a single responsibility, which makes them easy to run frequently during development.

Unit testing is needed because it helps catch bugs early, prevents regressions when code changes, and improves confidence in the system. It also encourages better design, since code that is easy to test is usually well-structured and loosely coupled.

In production, unit tests are a critical part of CI/CD pipelines, allowing teams to validate changes automatically before deployment.

The key idea is that unit tests verify small pieces of logic in isolation, making the system more reliable and easier to maintain.

---

<a id="q66"></a>

## 66. How does a unit test differ from an integration test?

### Question Restatement
How do **scope**, **dependencies**, and **goals** differ between **unit** and **integration** tests?

### 1. Comparison

| Aspect | Unit test | Integration test |
|--------|-----------|---------------------|
| **Scope** | One unit (method/class) | Multiple components wired together |
| **Dependencies** | Mocked / stubbed / in-memory fakes | Real or containerized (DB, broker, HTTP) |
| **Speed** | Very fast | Slower |
| **Purpose** | Correctness of logic in isolation | Correctness of **collaboration** and configuration |

### 2. Interview nuance
Neither is “better”—they answer **different questions**. Healthy projects use **many fast unit tests** plus **targeted integration tests** for boundaries you cannot fake safely (SQL dialects, serialization, Spring context slices, Testcontainers).

### 3. Summary
The main difference between unit tests and integration tests is the scope and how dependencies are handled.

A unit test verifies a small, isolated piece of logic, such as a single method or class, with all external dependencies mocked or stubbed. The goal is to test business logic independently from infrastructure like databases or external services.

An integration test, on the other hand, verifies how multiple components work together. It typically involves real dependencies such as a database, Spring context, or external systems, and tests the interaction between layers like controller, service, and repository.

Unit tests are fast, lightweight, and deterministic, while integration tests are slower but provide higher confidence that the system works correctly as a whole.

In production systems, unit tests are used for quick feedback during development, and integration tests are used to validate that components are correctly wired and behave properly together.

The key idea is that unit tests isolate logic, while integration tests validate collaboration between components.

---

<a id="q67"></a>

## 67. What libraries are most often used for testing in Java? (JUnit, Mockito)

### Question Restatement
Which **libraries** form the typical Java testing stack in 2025-style projects?

### 1. Core stack
- **JUnit 5 (Jupiter)** — test lifecycle (`@Test`, `@BeforeEach`, `@Nested`), assertions, parameterized tests, extensions.
- **Mockito** — mocks, stubs, spies; `when`/`verify`; often with **`@ExtendWith(MockitoExtension.class)`** or Spring’s `@MockBean` in slice tests.

### 2. Common additions
- **AssertJ** — fluent assertions for readable failures.
- **Hamcrest** — matchers (legacy/interop).
- **Spring Boot Test** — `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest` for **context** and **slice** integration tests.
- **Testcontainers** — real Postgres/Kafka/etc. in Docker for integration confidence.
- **JaCoCo** — coverage attached to `mvn verify` / Gradle test task.

### 3. Summary
In Java, the most commonly used libraries for testing are JUnit and Mockito.

JUnit is the standard testing framework used to write and run tests. It provides annotations like @Test, lifecycle hooks, and assertions to verify expected behavior. It is typically used for both unit and integration tests.

Mockito is a mocking framework used alongside JUnit to create mock objects and stub dependencies. It allows isolating the unit under test by replacing real dependencies such as repositories or external services with controlled mocks.

In Spring applications, these are often combined with Spring Boot Test for integration testing, where the application context is loaded and components are tested together.

The key idea is that JUnit provides the testing structure and execution, while Mockito provides isolation by mocking dependencies, which together enable effective and reliable testing.

---

<a id="q68"></a>

## 68. What is a test fixture (a set of data for tests)?

### Question Restatement
What is a **test fixture**, and how do you use it without making tests brittle?

### 1. Definition
A **fixture** is the **known state** and **inputs** required for a test: sample entities, JSON payloads, DB seed data, mock configurations, clock offsets, feature flags—anything that makes the test **repeatable**.

### 2. In JUnit 5
Typical patterns: **`@BeforeEach` / `@BeforeAll`**, builders/factories, **Object Mother** fixtures, or **@Sql** for integration tests. Keep fixtures **minimal**—only data that affects the assertion.

### 3. Summary
A test fixture is a predefined set of data and environment setup used to run tests in a consistent and repeatable way.

It includes everything required before a test executes, such as creating objects, initializing state, preparing test data, configuring mocks, or setting up a database state. The fixture ensures that each test starts from a known, controlled condition.

In Java testing with JUnit, fixtures are typically created using setup methods like @BeforeEach or @BeforeAll, where common test data and dependencies are initialized.

The main purpose of a test fixture is to make tests deterministic, isolated, and reproducible, so that test results do not depend on external factors or previous executions.

The key idea is that a test fixture defines the initial state for a test, ensuring reliability and consistency.

---

<a id="q69"></a>

## 69. How to properly name test methods?

### Question Restatement
How should **test method names** read in code review and in failing CI logs?

### 1. Conventions
Good names encode **scenario + action + expected outcome**, e.g. `shouldRejectOrderWhenStockIsZero()` or `createUser_whenEmailExists_throwsConflict()`.

### 2. Rules of thumb
- Prefer **business language** over `test1`.
- **One behavior** per test—name should map to one failure story.
- Align with team style: **snake_case** sentence names vs **camelCase**—consistency beats personal taste.

### 3. Summary

Test methods should be named in a way that clearly describes the behavior being tested, the conditions under which it is tested, and the expected outcome.
A common and effective convention is the given–when–then format, for example:
shouldReturnUser_whenUserExists or givenValidInput_whenCreatingUser_thenSuccess.
Another widely used style is:
methodName_condition_expectedResult, for example:
getUserById_userExists_returnsUser.
The goal is that the test name reads like a sentence and makes it immediately clear what the test is verifying without looking at the implementation.
In practice, good test names improve readability, make failures easier to understand, and serve as documentation of system behavior.
The key idea is that a test name should describe behavior, not implementation details, and clearly express input conditions and expected results.

---

<a id="q70"></a>

## 70. What are mock, stub, and spy?

### Question Restatement
What are **mock**, **stub**, and **spy**, and when do you use each?

### 1. Definitions
- **Stub** — returns **canned answers**; supports the test by supplying indirect inputs. Usually **no** interaction verification.
- **Mock** — a test double that can **verify** interactions: **how many times** a method was called, **with which arguments**.
- **Spy** — wraps a **real object**; real methods run unless **stubbed**; partial override of behavior.

### 2. Quick mental model
**Stub** = “feed this value.” **Mock** = “prove this collaboration happened.” **Spy** = “mostly real, override one painful call.”

### 3. Summary
In testing, mock, stub, and spy are types of test doubles used to control and observe dependencies.

A stub is a simple object that provides predefined responses to method calls. It is used to supply controlled data to the unit under test, without any logic or verification.

A mock is a more advanced test double that not only provides behavior but also allows verification of interactions. It is used to assert that certain methods were called with specific arguments, for example verifying that a service calls a repository method.

A spy is a wrapper around a real object that allows partial mocking. It calls real methods by default but lets you override specific behavior or verify interactions.

In Java, these are commonly implemented using Mockito.

The key idea is that stubs provide data, mocks verify behavior, and spies combine real behavior with the ability to observe or override it.

---

<a id="q71"></a>

## 71. In what cases is it better to use Mockito?

### Question Restatement
When is **Mockito** the right tool—and when should you **not** mock?

### 1. Good fits
- Replace **slow or non-deterministic** dependencies (remote HTTP, clock, randomness).
- Simulate **errors** (`when(repo.findById(1)).thenThrow(...)`).
- **Verify** side effects (`verify(notificationPort).send(...)`).

### 2. Anti-patterns
- Mocking **everything** in an integration test that should prove **Spring wiring** or SQL.
- Mocking **value objects** or trivial data carriers—use real instances.

### 3. Summary
Mockito (Mockito) is best used in unit tests when you need to isolate the class under test from its dependencies.

It is appropriate when the class interacts with external components such as databases, REST clients, message brokers, or other services, and you want to avoid calling real implementations. Instead, you mock those dependencies and control their behavior.

Mockito is also useful when you need to verify interactions, for example ensuring that a method was called with specific arguments or called a certain number of times. This is important when testing orchestration logic in services.

It should be used when dependencies are slow, non-deterministic, or have side effects, and when you want fast, reliable, and isolated tests.

However, Mockito should not be overused — it is better to use real objects when possible, especially for simple logic or value objects, and avoid mocking everything, as it can make tests brittle and tightly coupled to implementation details.

The key idea is that Mockito is used to isolate behavior and verify interactions, making unit tests fast and predictable, but it should be applied selectively.

---

<a id="q72"></a>

## 72. What are parameterized tests?

### Question Restatement
What are **parameterized tests** in JUnit 5, and why use them?

### 1. Definition
A **parameterized test** runs the **same test logic** against **many input sets**, declared as data (`@CsvSource`, `@MethodSource`, `@ValueSource`, `@EnumSource`).

### 2. Benefits
- Less copy-paste for **tables of examples**.
- Forces you to think about **edge cases** explicitly.
- Failures show **which row** broke (`@ParameterizedTest` name templates).

### 3. Summary
Parameterized tests are tests that run the same test logic multiple times with different input data sets.

Instead of writing separate test methods for each case, a single test method is defined and executed with various parameters, allowing you to cover multiple scenarios in a concise and maintainable way.

In Java, parameterized tests are commonly implemented using JUnit, for example with annotations like @ParameterizedTest and data sources such as @ValueSource, @CsvSource, or @MethodSource.

This approach reduces code duplication, improves test coverage, and makes it easier to validate behavior across a range of inputs, including edge cases.

The key idea is that parameterized tests separate test logic from test data, allowing the same behavior to be verified against multiple inputs efficiently.

---

<a id="q73"></a>

## 73. What is code coverage and how is it measured?

### Question Restatement
What does **code coverage** measure, and how should teams interpret it?

### 1. Metrics
Common metrics: **line**, **branch** (`if`/`else` paths), **method**, and **class** coverage. **JaCoCo** attaches to the JVM during `test`/`verify` and produces HTML/XML reports.

### 2. Interpretation
High coverage **does not** imply good tests—you can execute lines with **weak assertions**. Branch coverage is often **more honest** than line coverage alone.

### 3. Summary
Code coverage is a metric that shows how much of your code is executed by automated tests. It helps evaluate how thoroughly the codebase is tested, but it does not guarantee correctness.

It is typically measured using tools like JaCoCo, which instrument the code during test execution and track which parts are executed.

The most common types of coverage are:

Line coverage — percentage of executed lines
Branch coverage — whether all branches of conditions (if/else) are tested
Method coverage — whether methods are invoked

Coverage is calculated as the ratio of executed code elements to the total number of elements, for example executed lines divided by total lines.

In practice, higher coverage improves confidence but does not ensure quality, because tests may execute code without asserting correct behavior.

The key idea is that code coverage measures how much code is exercised by tests, but meaningful assertions and test quality are more important than the percentage itself.

---

<a id="q74"></a>

## 74. Why is it important to write tests before or along with the code (TDD)?

### Question Restatement
Why write tests **before** or **with** production code—what is **TDD** good for?

### 1. Benefits
- **Clarifies requirements** before implementation drifts.
- **Shapes APIs** from the caller’s perspective.
- **Reduces cost** of fixing defects found late in QA or prod.

### 2. TDD cycle
**Red** (failing test) → **Green** (minimal code) → **Refactor** (clean structure while tests stay green).

### 3. Summary
Writing tests before or alongside the code, as in Test-Driven Development (TDD), is important because it forces you to define the expected behavior and API of the system before implementing it.

In TDD, you follow a cycle: write a failing test, implement the minimal code to make it pass, and then refactor. This leads to smaller, incremental changes and continuous validation of correctness.

This approach improves design because code that is easy to test is usually well-structured, loosely coupled, and has clear responsibilities. It also prevents overengineering, since you implement only what is required to satisfy the test.

Additionally, having tests from the beginning provides immediate regression protection and makes refactoring safer, which is critical in production systems.

The key idea is that TDD turns tests into a design tool, not just a verification step, resulting in more maintainable and reliable code.

---

<a id="q75"></a>

## 75. What is CI/CD and why is it needed?

### Question Restatement
What is **CI/CD**, and why do teams invest in it?

### 1. Definition
**CI/CD** automates **build, test, package, and release** so every integration is **verified** and deliverable software is **reproducible**.

- **CI (Continuous Integration)** — frequent merges; each change runs **build + tests** automatically.
- **CD** often means **Continuous Delivery** (always releasable; prod may still be manual) or **Continuous Deployment** (successful pipeline **auto-promotes** to prod).

### 2. Why it matters
Faster feedback, fewer **manual mistakes**, consistent **release hygiene**, and shorter **lead time** from commit to production.

### 3. Summary
CI/CD stands for Continuous Integration and Continuous Delivery (or Continuous Deployment), and it is a set of practices that automate building, testing, and delivering software.

Continuous Integration (CI) means that developers frequently merge code into a shared repository, and every change is automatically built and tested. This helps detect integration issues early and ensures that the codebase is always in a working state.

Continuous Delivery (CD) means that the application is always in a deployable state, and releases can be triggered at any time. Continuous Deployment goes further by automatically deploying every successful change to production without manual intervention.

CI/CD pipelines typically include steps like building the application, running unit and integration tests, performing static analysis, and deploying to staging or production environments.

The key idea is that CI/CD reduces manual work, catches errors early, speeds up delivery, and makes releases more reliable and consistent.

---

<a id="q76"></a>

## 76. What is the difference between CI and CD?

### Question Restatement
How do **CI** and **CD** differ in scope and outcome?

### 1. CI
Focuses on **developer loop quality**: compile, unit tests, static analysis on **every push/PR**—“**Is this change safe to merge?**”

### 2. CD
Focuses on **release path**: packaging, promotion across environments, deployment automation, smoke checks—“**Can we ship this revision reliably?**”

### 3. Summary
The difference between Continuous Integration (CI) and Continuous Delivery/Deployment (CD) lies in the stage of the software lifecycle they automate.

Continuous Integration (CI) focuses on integrating code changes frequently into a shared repository. Every change triggers an automated build and test process, ensuring that the code is always in a working state and that integration issues are detected early.

Continuous Delivery (CD) extends CI by ensuring that the application is always in a deployable state. After passing all tests, the system can be released to production at any time, usually with a manual approval step.

Continuous Deployment goes one step further by automatically deploying every successful change to production without manual intervention.

The key idea is that CI is about building and validating code continuously, while CD is about delivering and releasing that code reliably to production environments.

---

<a id="q77"></a>

## 77. What CI/CD tools do you know? (Jenkins, GitHub Actions, GitLab CI, TeamCity)

### Question Restatement
Which **CI/CD tools** are common in Java shops, and how do they differ?

### 1. Named tools
- **Jenkins** — self-hosted, plugin ecosystem, Groovy/declarative pipelines; flexible but ops-heavy.
- **GitHub Actions** — YAML workflows in `.github/workflows`, tight GitHub integration, marketplace **actions**.
- **GitLab CI** — `.gitlab-ci.yml`, integrated registry, environments, security scanning.
- **TeamCity** — JetBrains CI server, strong UI, enterprise features.

### 2. Also-rans
**Azure DevOps Pipelines**, **CircleCI**, **Bitbucket Pipelines**, **Travis CI** (legacy mindshare)—mention if you have experience.

### 3. Summary
Common CI/CD tools include Jenkins, GitHub Actions, GitLab CI/CD, and TeamCity.

Jenkins is a widely used, self-hosted automation server with a large plugin ecosystem, giving full control but requiring manual setup and maintenance.

GitHub Actions and GitLab CI/CD are integrated CI/CD solutions built into their respective platforms, allowing you to define pipelines as code and run workflows directly from your repository, which simplifies setup and integration.

TeamCity is a CI/CD server from JetBrains that provides a more structured and user-friendly interface, often used in enterprise environments.

In practice, the choice depends on infrastructure and team needs: self-hosted tools like Jenkins offer flexibility, while integrated solutions like GitHub Actions or GitLab CI/CD reduce operational overhead.

The key idea is that all these tools automate build, test, and deployment pipelines, but differ in setup complexity, flexibility, and integration level.

---

<a id="q78"></a>

## 78. What does a simple CI pipeline for a Java project look like?

### Question Restatement
Describe a **minimal** but realistic **Java CI pipeline**.

### 1. Typical flow
1. **Trigger** — push or pull request.
2. **Checkout** source.
3. **Setup JDK** (e.g. Temurin 17/21) and **cache** dependencies.
4. **Build + test** — `mvn -B clean verify` or `./gradlew clean test build`.
5. **Publish** — JUnit XML, JaCoCo HTML, optional artifact (JAR).

### 2. Quality gates
Fail fast on **compile errors** and **test failures**; optional **quality** job (SpotBugs, Checkstyle) as a separate stage.

### 3. Summary
A simple CI pipeline for a Java project typically consists of a sequence of automated steps that run on every commit or pull request to validate the code.

The pipeline usually starts with checking out the source code from the repository, followed by setting up the Java environment and dependencies using a build tool like Maven or Gradle.

Next, the project is compiled to ensure there are no syntax or compilation errors. After that, unit tests and, optionally, integration tests are executed. This step is critical because it validates business logic and prevents regressions.

Additional steps often include static code analysis (for example, style checks or quality gates), and generating reports such as test results or code coverage.

If all steps pass, the pipeline can produce a build artifact, such as a JAR file or Docker image, which can then be used in later stages like deployment.

In practice, the pipeline looks like: checkout → build → test → analyze → package → publish artifact.

The key idea is that every change is automatically verified through the same repeatable process, ensuring code quality and preventing broken builds from reaching further environments.
---
A typical CI pipeline for a Java project includes several stages: source code checkout, build, test, analysis, and artifact creation.

First, the pipeline checks out the code from the repository and sets up the Java environment, for example using JDK 21 and a build tool like Maven.
Then the project is compiled and tested using commands like mvn clean verify, which runs unit and integration tests to ensure correctness.

After that, static code analysis tools such as SonarQube are used to check code quality and detect potential issues.
If all steps pass, the pipeline produces an artifact, such as a JAR file, which can later be used for deployment.

In more advanced setups, the pipeline also builds a Docker image and pushes it to a registry, making the application ready for deployment.

The key idea is that every change automatically goes through the same build, test, and validation process, ensuring code quality and preventing broken code from progressing further.
---

---

<a id="q79"></a>

## 79. What stages are usually included in a pipeline? (build, tests, deploy)

### Question Restatement
What **stages** do mature pipelines include beyond “build and test”?

### 1. Common sequence
**Checkout → build/compile → static analysis → unit tests → integration tests (optional) → package → publish artifact → deploy to staging → smoke/E2E → deploy to prod (manual or auto) → post-deploy checks / rollback hooks.**

### 2. Interview angle
Stages exist to **separate concerns** and **parallelize** what is safe—e.g. **fast unit** stage before **slow integration** or **Docker** build.

### 3. Summary
A typical CI/CD pipeline consists of several stages that validate and deliver the application in a controlled and automated way.

The first stage is build, where the source code is compiled and packaged using tools like Maven or Gradle. This ensures that the code is syntactically correct and can be turned into a runnable artifact such as a JAR.

The next stage is testing, which includes unit tests and often integration tests. This stage verifies that the business logic works correctly and that components interact properly. It is critical for catching regressions early.

After that, many pipelines include static analysis, where tools check code quality, security issues, and adherence to standards.

Then comes the packaging stage, where artifacts are prepared for deployment, for example building a Docker image.

Finally, the deployment stage delivers the application to an environment such as staging or production. In Continuous Delivery this step may require manual approval, while in Continuous Deployment it is fully automated.

The key idea is that the pipeline moves code through a series of automated validation steps—from build to deployment—ensuring that only tested and verified code reaches production.

---

<a id="q80"></a>

## 80. What is a build artifact?

### Question Restatement
What is a **build artifact**, and why must it be **immutable**?

### 1. Definition
An **artifact** is the **output** of CI intended for release: **JAR**, **WAR**, **Docker image**, Helm chart, plus **metadata** (version, checksum, SBOM).

### 2. Practice
The **same binary** should flow **dev → staging → prod** to guarantee you tested what you ship. Rebuilding “the same tag” at deploy time breaks that guarantee.

### 3. Summary
A build artifact is the output produced by the build process that can be stored, versioned, and deployed.

In a Java project, typical build artifacts include JAR or WAR files generated by tools like Maven or Gradle. In modern systems, a Docker image can also be considered an artifact because it packages the application together with its runtime environment.

Artifacts are created after successful compilation and testing and are usually stored in artifact repositories or registries, making them reusable across different stages of the pipeline such as testing, staging, and production.

The key idea is that a build artifact is a reproducible, deployable unit of software that represents a specific version of the application.

---

<a id="q81"></a>

## 81. How to integrate unit tests into a CI pipeline?

### Question Restatement
How do you run **unit tests** in CI so they **protect main** without slowing everyone down?

### 1. Essentials
- Run on **every PR** and **main** branch pushes.
- **Fail the job** on any test failure—no silent skips.
- Publish **JUnit XML** and optionally **coverage** to the PR UI.

### 2. Maven / Gradle
- Maven: `mvn test` or **`mvn verify`** (includes integration phases you configured).
- Gradle: `./gradlew test` or `check` task if configured.

### 3. Summary
Unit tests are integrated into a CI pipeline by making them a mandatory step in the build process that runs automatically on every commit or pull request.

Typically, after the source code is checked out and the environment is set up, the pipeline executes a build command such as mvn test or mvn verify, which runs all unit tests using frameworks like JUnit.

If any test fails, the pipeline fails immediately, preventing the code from being merged or deployed. This ensures that only code that passes all tests moves forward in the pipeline.

In more advanced setups, test results and reports are collected and published, and code coverage tools like JaCoCo are used to track how much of the code is tested.

The key idea is that unit tests are automatically executed on every change and act as a quality gate, stopping broken or incorrect code from progressing further in the delivery pipeline.

---

<a id="q82"></a>

## 82. How does GitHub Actions work?

### Question Restatement
How does **GitHub Actions** execute automation for a repository?

### 1. Model
- **Workflow** YAML under `.github/workflows/*.yml`.
- **Event** triggers (`push`, `pull_request`, `schedule`, `workflow_dispatch`, …).
- **Jobs** run on **runners** (`ubuntu-latest`, Windows, macOS, or **self-hosted**).
- **Steps** run shell commands or **reusable actions** from the Marketplace.
- **`needs`** expresses job **DAG**; **matrix** builds across JDK versions.

### 2. Flow
Event → workflow selected → jobs scheduled → checkout → setup JDK → cache → build/test → upload artifacts / comment PR checks.

### 3. Summary
GitHub Actions is a CI/CD platform built into GitHub that allows you to automate workflows based on repository events.

It works by defining workflows as YAML files inside the .github/workflows directory. These workflows are triggered by events such as push, pull_request, or manual triggers.

A workflow consists of one or more jobs, and each job runs on a runner (a virtual machine or container). Jobs are made up of steps, which can either execute shell commands or reuse predefined actions from the GitHub marketplace.

When an event occurs, GitHub schedules the workflow, provisions a runner, and executes the defined steps sequentially or in parallel depending on the configuration.

Workflows can include tasks like building the project, running tests, performing code analysis, and deploying artifacts. They can also use secrets for secure access to external systems such as Docker registries or cloud providers.

The key idea is that GitHub Actions provides event-driven automation directly in the repository, where workflows are defined as code and executed in isolated environments for each run.

---

<a id="q83"></a>

## 83. What is YAML in the context of CI/CD?

### Question Restatement
Why is **YAML** the default for **CI/CD configuration**?

### 1. Role
YAML describes **pipelines as data**: triggers, env vars, job graphs, shell steps—**versioned** and reviewed like application code.

### 2. Trade-offs
Readable for humans; **indentation-sensitive** (merge conflicts can break semantics); needs **linting** (`actionlint`, schema validation) in larger orgs.

### 3. Summary
YAML (YAML Ain’t Markup Language) is a human-readable data serialization format used in CI/CD to define pipelines and workflows as code.

In CI/CD systems like GitHub Actions or GitLab CI/CD, YAML files are used to describe how the pipeline should run, including triggers, jobs, steps, environment variables, and dependencies between stages.

It uses indentation instead of brackets to represent structure, which makes it concise and easy to read, but also sensitive to formatting errors.

YAML allows pipelines to be version-controlled alongside the source code, making builds reproducible and changes to the pipeline traceable.

The key idea is that YAML is the configuration language that defines the behavior of CI/CD pipelines in a declarative way.

---

<a id="q84"></a>

## 84. How can build result notifications be implemented?

### Question Restatement
How do teams **notify** humans when builds or deploys fail (or succeed)?

### 1. Channels
**Slack / Microsoft Teams / Telegram**, **email**, **PR checks & comments**, **PagerDuty** for production incidents.

### 2. Implementation
- **Native integrations** in GitLab/Jenkins/GitHub (Apps).
- **Webhooks** from CI to chat APIs.
- **Custom script** step at end of job posting payload with **commit, branch, author, log URL**.

### 3. Summary
Build result notifications in CI/CD are implemented by configuring the pipeline to send messages or alerts based on the outcome of a workflow run, such as success, failure, or unstable builds.

In tools like GitHub Actions, this is typically done by adding a step that runs conditionally using expressions like if: failure() or if: success(), and then calling an external service such as Slack, email, or a webhook.

For example, a pipeline can send a notification to a Slack channel using an HTTP request or a dedicated action, or send emails using SMTP integrations. Most CI/CD platforms also provide built-in integrations with messaging systems.

Notifications can include details like build status, commit information, logs, or links to the pipeline run, helping teams quickly react to failures.

The key idea is that notifications are triggered automatically based on pipeline events and are used to provide immediate feedback about the state of the build or deployment.