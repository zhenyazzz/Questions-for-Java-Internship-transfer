# Testing — Questions 65–74

Interview-focused answers in English for Java testing topics.

## Table of Contents

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

---

<a id="q65"></a>
## 65. What is unit testing and why is it needed?

**Unit testing** verifies a small isolated piece of code (usually one class or method) with controlled inputs and expected outputs.

Why it is needed:
- catches regressions early,
- documents behavior of code,
- enables safer refactoring,
- short feedback loop (fast execution),
- improves design (smaller, decoupled classes are easier to test).

A good unit test is deterministic, isolated, and fast.

---

<a id="q66"></a>
## 66. How does a unit test differ from an integration test?

- **Unit test**
  - scope: one class/unit,
  - dependencies: mocked/stubbed,
  - speed: very fast,
  - goal: verify business logic in isolation.

- **Integration test**
  - scope: multiple components together (service + repository + DB, or API + message broker),
  - dependencies: real framework wiring, often real DB (or containerized),
  - speed: slower,
  - goal: verify collaboration and configuration.

Interview summary: unit tests prove logic correctness; integration tests prove components work together in real environment.

---

<a id="q67"></a>
## 67. What libraries are most often used for testing in Java? (JUnit, Mockito)

Most common stack:
- **JUnit 5 (Jupiter)**: test framework (`@Test`, assertions, lifecycle hooks, parameterized tests).
- **Mockito**: mocking framework (`@Mock`, `when`, `verify`, `@InjectMocks`).

Often used together with:
- **AssertJ**: fluent assertions,
- **Hamcrest**: matcher-based assertions,
- **Testcontainers**: integration tests with real DB/Kafka/etc. in Docker,
- **Spring Boot Test**: context-based integration/slice tests,
- **JaCoCo**: coverage reporting.

---

<a id="q68"></a>
## 68. What is a test fixture (a set of data for tests)?

A **test fixture** is the known setup required before a test runs:
- input objects,
- database state,
- mocks and their behavior,
- environment config needed by the test.

Purpose:
- make tests reproducible,
- avoid duplicated setup code,
- improve readability.

In JUnit, fixtures are commonly created in `@BeforeEach`, `@BeforeAll`, helper builders, or factory methods.

---

<a id="q69"></a>
## 69. How to properly name test methods?

Good test names should explain:
1. **scenario**,
2. **action**,
3. **expected result**.

Common style:
- `shouldReturnDiscountWhenCustomerIsPremium()`
- `createOrder_whenStockIsInsufficient_shouldThrowException()`

Rules:
- be explicit, not generic (`test1`, `checkData` are bad),
- one behavior per test,
- reflect business language.

---

<a id="q70"></a>
## 70. What are mock, stub, and spy?

- **Stub**
  - returns predefined data,
  - used to provide indirect inputs to tested code.
- **Mock**
  - test double that also verifies interaction (method calls, arguments, call count).
- **Spy**
  - wraps real object,
  - real methods are called unless explicitly stubbed.

Quick distinction:
- stub = state-based support,
- mock = behavior verification,
- spy = partial mock over real object.

---

<a id="q71"></a>
## 71. In what cases is it better to use Mockito?

Use Mockito when:
- you test a class in isolation and need to replace external dependencies (repository, HTTP client, message sender),
- dependencies are slow, non-deterministic, or hard to set up,
- you need to verify interactions (`verify(repo).save(...)`),
- you want to test edge/error branches by stubbing exceptions.

Do not overuse Mockito:
- avoid mocking value objects and simple domain models,
- avoid mocking everything in integration tests where real wiring should be tested.

---

<a id="q72"></a>
## 72. What are parameterized tests?

Parameterized tests run the **same test logic** with multiple input sets.

Benefits:
- less duplication,
- better edge-case coverage,
- readable table-like test style.

In JUnit 5:
- `@ParameterizedTest`,
- sources: `@ValueSource`, `@CsvSource`, `@MethodSource`, `@EnumSource`.

Example:
```java
@ParameterizedTest
@CsvSource({"2,4", "3,9", "4,16"})
void square_shouldReturnExpected(int input, int expected) {
    assertEquals(expected, input * input);
}
```

---

<a id="q73"></a>
## 73. What is code coverage and how is it measured?

**Code coverage** is the percentage of code executed by tests.

Common metrics:
- **Line coverage**: executed lines / total lines,
- **Branch coverage**: executed branches (`if/else`, switch paths),
- **Method coverage**,
- **Class coverage**.

Measured by tools like **JaCoCo** during test run.

Important interview point:
- high coverage does **not** guarantee high quality tests.
- 100% line coverage can still miss critical assertions and edge cases.

Coverage should be treated as a signal, not a target by itself.

---

<a id="q74"></a>
## 74. Why is it important to write tests before or along with the code (TDD)?

Writing tests before or together with code helps:
- clarify requirements early,
- design APIs from usage perspective,
- keep code modular/testable,
- prevent regressions continuously,
- provide living documentation.

**TDD cycle**:
1. Red — write failing test,
2. Green — write minimal code to pass,
3. Refactor — improve design while keeping tests green.

Even if strict TDD is not always used, writing tests in parallel with development dramatically reduces defect cost and refactoring risk.

