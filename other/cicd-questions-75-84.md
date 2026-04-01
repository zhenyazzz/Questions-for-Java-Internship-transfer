# CI/CD — Questions 75–84

Practical interview answers in English for CI/CD fundamentals and Java pipeline setup.

## Table of Contents

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

<a id="q75"></a>
## 75. What is CI/CD and why is it needed?

**CI/CD** is a software delivery practice that automates integration, testing, packaging, and release.

- **CI (Continuous Integration)**: developers merge code frequently; each merge triggers automatic build and tests.
- **CD** can mean:
  - **Continuous Delivery**: software is always releasable; deployment to production is usually manual approval.
  - **Continuous Deployment**: every successful pipeline is automatically deployed to production.

Why needed:
- faster feedback on bugs,
- fewer integration conflicts,
- repeatable and reliable releases,
- less manual work and fewer human errors,
- faster time-to-market.

---

<a id="q76"></a>
## 76. What is the difference between CI and CD?

- **CI** focuses on code quality during development:
  - build + tests on each push/PR,
  - early defect detection.

- **CD** focuses on release automation:
  - package + deploy to environments,
  - release safely and frequently.

Simple interview answer:
- CI = “Is code healthy and mergeable?”
- CD = “Can we deliver this version safely now?”

---

<a id="q77"></a>
## 77. What CI/CD tools do you know? (Jenkins, GitHub Actions, GitLab CI, TeamCity)

Common tools:
- **Jenkins**: highly flexible, plugin-rich, self-hosted classic.
- **GitHub Actions**: native for GitHub repos; workflow-as-code in YAML.
- **GitLab CI/CD**: integrated with GitLab; strong built-in pipeline UX.
- **TeamCity**: JetBrains CI server, strong UI and enterprise features.

Also often mentioned:
- CircleCI, Travis CI, Azure DevOps Pipelines, Bitbucket Pipelines.

---

<a id="q78"></a>
## 78. What does a simple CI pipeline for a Java project look like?

Typical Java CI flow:
1. Trigger on push or pull request.
2. Checkout source code.
3. Set up JDK (e.g., 17/21).
4. Restore dependency cache (Maven/Gradle).
5. Run build + unit tests.
6. Publish test reports and coverage.
7. (Optional) Package JAR/WAR and store artifact.

Example command:
- Maven: `mvn -B clean verify`
- Gradle: `./gradlew clean test build`

---

<a id="q79"></a>
## 79. What stages are usually included in a pipeline? (build, tests, deploy)

Typical stages:
1. **Checkout**
2. **Build/Compile**
3. **Static checks** (lint, style, SAST)
4. **Unit tests**
5. **Integration tests** (optional per branch/environment)
6. **Package** artifact
7. **Publish** artifact to repository (Nexus/Artifactory/GitHub Packages)
8. **Deploy** to staging
9. **Smoke/E2E tests**
10. **Deploy** to production (manual approval or automatic)
11. **Post-deploy checks** and rollback strategy

---

<a id="q80"></a>
## 80. What is a build artifact?

A **build artifact** is the output produced by the build process and intended for deployment or distribution.

In Java this is usually:
- JAR,
- WAR,
- Docker image,
- plus metadata (checksums, SBOM, version info).

Why important:
- immutable deliverable,
- traceable to commit/build number,
- same artifact should go through all environments.

---

<a id="q81"></a>
## 81. How to integrate unit tests into a CI pipeline?

Best practice:
1. Run tests on every push/PR.
2. Fail pipeline if any test fails.
3. Publish JUnit reports.
4. (Optional) enforce minimum coverage threshold.
5. Keep test suite fast and deterministic.

For Maven:
- `mvn test` or `mvn verify` in CI.

For Gradle:
- `./gradlew test`.

Also useful:
- split tests by category (unit vs integration),
- run unit tests first for fast feedback.

---

<a id="q82"></a>
## 82. How does GitHub Actions work?

GitHub Actions uses YAML workflow files in:
- `.github/workflows/*.yml`

Core concepts:
- **workflow**: automation definition.
- **event trigger**: `push`, `pull_request`, `workflow_dispatch`, `schedule`, etc.
- **job**: group of steps running on a runner.
- **step**: command or reusable action.
- **runner**: VM/container executing jobs (`ubuntu-latest`, self-hosted).
- **actions**: reusable building blocks from marketplace or custom.

Typical flow:
1. Event happens.
2. Workflow starts.
3. Jobs run (parallel or dependency chain via `needs`).
4. Status checks reported back to PR/commit.

---

<a id="q83"></a>
## 83. What is YAML in the context of CI/CD?

In CI/CD, **YAML** is a human-readable config format used to describe pipeline logic as code.

Why used:
- versioned in Git,
- readable structure,
- easy to review in pull requests,
- reproducible automation.

In GitHub Actions/GitLab CI/Jenkins pipelines (declarative), YAML defines:
- triggers,
- stages/jobs,
- environment variables,
- scripts,
- conditions and dependencies.

---

<a id="q84"></a>
## 84. How can build result notifications be implemented?

Common methods:
- chat integrations (Slack, Microsoft Teams, Telegram),
- email notifications,
- pull request status checks/comments,
- incident tools (PagerDuty/Opsgenie) for prod deploy failures.

Implementation options:
- built-in CI plugins/integrations,
- webhook on pipeline events,
- custom script step after job completion.

Good practice:
- notify only actionable events (failed main branch build, failed deploy),
- include context: repo, branch, commit, author, failed stage, link to logs.

