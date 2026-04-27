# Module 7 — Docker + Integration Testing — Questions 95–109

> Questions 95–109 from *Questions for Java Internship transfer.pdf* (virtualization vs containers, Docker images/containers/builds/layers/volumes/networks, Compose, integration testing, Testcontainers, Spring test context, mocking outbound HTTP, security in tests). Related: `other/docker-virtualization-questions-95-104.md` (95–104 overlap), `Модуль 5 — Testing + CI-CD.md` (unit vs integration at **66**).

> Each numbered section ends with a **Middle+ answer** you can adapt on interview (a short monologue plus optional bullets above for follow-ups).

## Module summary (middle+ — how I’d frame it on interview)

<a id="summary"></a>

**Docker** is packaging and isolation at the **OS process boundary** (Linux **namespaces** + **cgroups**): you ship an **immutable image**—stacked **read-only layers**, default command, user, exposed ports—and run it as an **ephemeral container** with a thin writable layer. **Volumes** and **networks** are first-class so you can separate **compute** (replace anytime) from **data** and **connectivity policy**. **Docker Compose** turns that into a **declarative multi-service stack** for local dev and small deployments: one YAML file, one `docker compose up`, same stack for onboarding and often for CI-like environments.

**Virtualization (VMs)** is different: a **hypervisor** presents virtual hardware; each VM runs a **full guest kernel**. You buy **maximum isolation** and **OS diversity** (Windows on Linux, strict zones) but pay in **RAM, boot time, patching many OS images**, and lower **density** than containers. In interviews I usually say we are not “VM vs Docker forever”—many prod setups are **VMs or metal running a container runtime**, or **Kubernetes** scheduling containers on top of cloud VMs.

**Integration tests** exercise **real seams** your unit tests cannot honestly fake: Spring **autowiring**, **Flyway/Liquibase** against a real dialect, **transaction boundaries**, **HTTP serialization**, message mapping. **Testcontainers** runs **real Postgres/Kafka** (etc.) next to the JVM; **WireMock / MockWebServer** stubs **outbound HTTP** so the client stack sees real status codes, headers, and timeouts. **Security** stays under **`@ActiveProfiles("test")`** or dedicated test beans—never a global “security disabled” flag shared with prod. The line I repeat: **load the smallest `ApplicationContext` that still covers the boundary you do not trust to mocks** (see **question 66** in `Модуль 5 — Testing + CI-CD.md`).

### Quick map (question → one line)

| # | Topic | One-line takeaway |
|---|--------|-------------------|
| **95** | Virtualization | Hypervisor splits hardware into VMs—each full guest OS; strong isolation, higher overhead. |
| **96** | VM vs containers | VM = hardware + kernel; container = shared host kernel + isolated userspace; density vs trust boundary. |
| **97** | Image | Immutable blueprint: layered FS + defaults; **digest** pins content. |
| **98** | Container | Runnable instance: cgroups + namespaces; data in writable layer is **lost** unless volumes. |
| **99** | Build image | `Dockerfile` + `docker build`; `.dockerignore`, non-root, pin bases; glibc vs musl for JNI. |
| **100** | Multi-stage | Several `FROM`s; final image only ships runtime artifact—smaller, fewer CVEs. |
| **101** | Layers | Ordered diffs → cache, incremental pull/push; stable instructions first in Dockerfile. |
| **102** | Volumes | Named vs bind vs tmpfs—**separate compute lifecycle from data**. |
| **103** | Networks | Bridge/host/overlay—**DNS on custom bridge**; publish ports vs internal-only. |
| **104** | Compose | YAML multi-container **local/small prod**; not K8s control plane. |
| **105** | Integration vs unit | Unit = isolated fast logic; integration = **real collaborators** at seams you cannot fake. |
| **106** | Testcontainers | JUnit-managed Docker deps—real fidelity; Docker in CI, reuse, pin images. |
| **107** | ApplicationContext | `@SpringBootTest` full vs **slices** (`@DataJpaTest`, `@WebMvcTest`)—smallest context that fits. |
| **108** | Mock outbound HTTP | WireMock/MockWebServer at **HTTP layer**; `@MockBean` only if wire is not the seam. |
| **109** | Security in tests | Test-only profile/chain/users; do not weaken prod `application.yml`. |

## Table of contents

- [Module summary (middle+)](#summary)

95. [What is Virtualization?](#q95)
96. [What is the difference between Virtualization and Containerization? What are the advantages and disadvantages of both approaches?](#q96)
97. [What is an Image in Docker?](#q97)
98. [What is a Container in Docker?](#q98)
99. [How to create an Image?](#q99)
100. [What are multi-staged builds?](#q100)
101. [What are layers in images?](#q101)
102. [What is Volume in Docker? What are Volume types?](#q102)
103. [What is Network in Docker? What are Network types?](#q103)
104. [What is docker-compose?](#q104)
105. [What are Integration tests? What is the difference between integration tests and unit tests?](#q105)
106. [What are Testcontainers? How are they used in Integration tests?](#q106)
107. [How to load an ApplicationContext in Integration tests?](#q107)
108. [How to mock external APIs calls in integration tests?](#q108)
109. [How to handle security issues in integration tests?](#q109)

---

<a id="q95"></a>

## 95. What is Virtualization?

### Question Restatement
What is **virtualization**, and what role does a **hypervisor** play?

### 1. Definition
**Virtualization** lets you run **multiple isolated environments** on **one physical machine** by abstracting **CPU, memory, storage, and network** through a **hypervisor** (VMM). Each **virtual machine (VM)** behaves like a separate computer with its **own guest operating system** and applications.

### 2. Hypervisor types
- **Type 1 (bare-metal)** — runs directly on hardware (e.g. VMware ESXi, Hyper-V on server roles, Xen). Common in datacenters.
- **Type 2 (hosted)** — runs on top of a host OS (e.g. VirtualBox, VMware Workstation). Common on developer laptops.

### 3. Why teams use VMs
**Consolidation** (many workloads on fewer hosts), **isolation** boundaries, **mixed OS** requirements, and **legacy** systems that expect a full OS.

### 4. Middle+ answer (as on interview)

Virtualization is a technique that allows running multiple isolated virtual machines on a single physical host.

It is implemented using a Hypervisor, which abstracts hardware resources like CPU, memory, storage, and networking, and allocates them to each virtual machine.

Each VM runs its own operating system and kernel, so it behaves like an independent computer.

There are two main types of hypervisors:

Type 1, like VMware ESXi or Microsoft Hyper-V, run directly on hardware and are commonly used in production environments.
Type 2, like VirtualBox, run on top of a host OS and are mostly used for development.

Teams use virtualization for better resource utilization, isolation between workloads, and the ability to run different operating systems on the same machine.

Compared to containers like Docker, virtual machines provide stronger isolation because each VM has its own kernel, while containers share the host OS kernel and are more lightweight.

---

<a id="q96"></a>

## 96. What is the difference between Virtualization and Containerization? What are the advantages and disadvantages of both approaches?

### Question Restatement
Compare **VMs** and **containers**—isolation model, density, and when each wins.

### 1. Core difference
- **Virtualization (VM)** — virtualizes **hardware**; each VM includes a **kernel + OS userland**.
- **Containerization** — virtualizes at the **OS process** boundary (Linux **namespaces**, **cgroups**); containers **share the host kernel** and run as isolated processes with their own filesystem view and network stack.

### 2. Comparison

| Aspect | VM | Container |
|--------|-----|-------------|
| **Isolation** | Very strong (separate kernels) | Strong process isolation; **shared kernel** is the trust boundary |
| **Startup / density** | Slower boot; lower density | Fast start; higher density |
| **OS flexibility** | Different guest OS per VM | Typically same **kernel family** as host |
| **Overhead** | Higher RAM/CPU/disk | Lower per instance |

### 3. Pros / cons (interview framing)

**VM pros:** kernel-level boundary; run Windows on Linux host (with right stack) and strict compliance zones. **VM cons:** heavy; patching many OS images.

**Container pros:** reproducible images; CI/CD friendly; microservice packaging. **Container cons:** wrong mental model (“free security”); orchestration and supply-chain **image** risks; **noisy neighbor** tuning still required.

### 4. Middle+ answer (as on interview)

**VMs** virtualize **hardware**: each VM boots a **full guest OS** with its **own kernel**. That gives very **strong isolation**—different kernel versions, different OS families, hard separation for compliance—and it is still the default mental model for “I need a machine.” The downside is **cost**: large images, minutes to provision sometimes, **lower density**, and you must **patch every guest OS**.

**Containers** (on Linux) virtualize at the **OS process** level: **namespaces** (PID, mount, network, …) + **cgroups** (CPU/memory/IO limits). Processes in different containers **share one host kernel** but see separate filesystem roots and network stacks. That means **seconds to start**, **many instances per host**, and images that are mostly **app + libs**, not a whole OS.

**Pros/cons in one breath:** VMs win on **kernel-level isolation** and **OS heterogeneity**; containers win on **speed, density, reproducible artifacts**, and **CI/CD**. Containers lose if you need a **different kernel** than the host or treat isolation as “free”—you still care about **image supply chain**, **capabilities**, **seccomp**, **resource limits** (**noisy neighbor**), and orchestration. In real life I often see **both**: VMs or bare metal run a **container runtime** or Kubernetes, so the interview answer is about **where the boundary is drawn**, not “Docker replaced VMware everywhere.”

---

<a id="q97"></a>

## 97. What is an Image in Docker?

### Question Restatement
What is a Docker **image**, and how does it relate to a **container**?

### 1. Definition
A Docker **image** is an **immutable template** from which containers are created. It bundles a **stacked read-only filesystem** (layers), **metadata** (default command/entrypoint, exposed ports, env, labels), and configuration needed to run the process.

### 2. Mental model
**Image** = class / blueprint. **Container** = instance / running process with a **writable thin layer** on top.

### 3. Naming
`repository:tag` (e.g. `eclipse-temurin:21-jre-alpine`). **Digest** (`@sha256:…`) pins exact content for reproducible deploys.

### 4. Middle+ answer (as on interview)

A Docker **image** is the **immutable build artifact** you store in a **registry** and promote through environments: a **stack of read-only filesystem layers** plus **metadata**—default `ENTRYPOINT`/`CMD`, working dir, env defaults, `EXPOSE` hints, labels, and often a **non-root user** in hardened setups. I explain it as **image ≈ class**, **container ≈ instance**: many containers from one image share the **same read-only stack** on disk.

Tags like `eclipse-temurin:21-jre` are convenient, but **tags move**; for anything serious I prefer **pinning by digest** (`image@sha256:…`) or at least **controlled internal tags** so prod always runs **byte-identical** bits and security scans map to a known artifact.

Images are built once, scanned (CVE policy), signed (optional), then **pulled** by runtime—so the interview point is: **the image is the unit of delivery**, not the Git commit alone; the container is just **how you execute that artifact** with optional mounts, env, and network attachment.

---

<a id="q98"></a>

## 98. What is a Container in Docker?

### Question Restatement
What is a Docker **container**, and what happens to **data** by default?

### 1. Definition
A **container** is a **runnable instance** of an image: an isolated process (or process tree) with its own **PID, mount, network, UTS, user namespaces** (as configured), limited by **cgroups** for CPU/memory/IO.

### 2. Filesystem behavior
Image layers are **read-only**; writes go to a **container writable layer**. When the container is removed, that writable layer is **lost** unless you use **volumes** or **bind mounts**.

### 3. Lifecycle
**create → start → stop → rm**; restart policies and orchestrators (Kubernetes) build on this model.

### 4. Middle+ answer (as on interview)

A **container** is a **running process tree** created from an image, isolated with **namespaces** (its own view of PIDs, mounts, network interfaces, hostname, etc.) and constrained with **cgroups** for **CPU shares, memory limits, IO**, sometimes **pids limit**—so it behaves like a lightweight VM from the app’s perspective, but without a separate kernel.

Filesystem-wise the image is **read-only**; writes go to a **container writable layer** (copy-on-write). **`docker rm`** drops that layer—so logs, uploads, or local H2 files **disappear** unless I mount a **volume/bind mount** or send data to **Postgres/S3** outside the container.

Lifecycle-wise I think **create → start → stop → restart** (restart policies in Compose/K8s build on this): containers should be **cattle**, not pets—**replace anytime**; **state** is either in a **volume** or an **external system**. Optional senior bit: PID 1 / signal handling matters for graceful shutdown (`SIGTERM`), same as in Kubernetes pods.

---

<a id="q99"></a>

## 99. How to create an Image?

### Question Restatement
How do you **build** a Docker image for a typical Java service?

### 1. Primary path: `Dockerfile` + `docker build`
You declare instructions (`FROM`, `WORKDIR`, `COPY`, `RUN`, `EXPOSE`, `USER`, `ENTRYPOINT`/`CMD`) and run:

```bash
docker build -t my-service:1.0.0 .
```

### 2. Minimal Spring Boot–style example

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/my-service.jar app.jar
EXPOSE 8080
USER nobody
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 3. Best practices (production-minded)
- **Pin** base image by **digest** or controlled tags; refresh for CVEs.
- **`.dockerignore`** — exclude `target/`, `.git`, IDE files to speed builds and shrink context.
- **Non-root** `USER` when the runtime allows it.
- Prefer **small** bases (Alpine/distroless) when compatible with native deps (**glibc** vs **musl** matters for some JNI stacks).

### 4. Middle+ answer (as on interview)

The standard path is **`Dockerfile` + `docker build`** (often **BuildKit** enabled in CI): you describe the filesystem and runtime command, Docker produces immutable layers, you **tag** and **push** to a registry (**Harbor**, ECR, GCR, Docker Hub, etc.) so **staging/prod pull the same digest** you tested.

For a **Spring Boot** service I typically **`COPY` the fat JAR** (or layered JAR with exploded classpath for faster starts), set **`WORKDIR`**, **`EXPOSE`**, run as **`USER` non-root** when the base image allows file permissions on the JAR, and use **`ENTRYPOINT`** with exec form so **PID 1 is Java** and signals propagate correctly.

Production hygiene I always name-check: **`.dockerignore`** so the build context does not include `.git`, IDE junk, or huge `target/` trees; **pin base images** (or digests) and **rebuild on CVE** cadence; watch **Alpine/musl vs Debian/glibc** if we have **JNI**, native agents, or weird DNS/ssl stacks; optional **`HEALTHCHECK`** for local/small prod. Same idea in **rootless** or **Kaniko/buildah** builds in Kubernetes CI—**Dockerfile stays the source of truth**, the executor varies.

---

<a id="q100"></a>

## 100. What are multi-staged builds?

### Question Restatement
What is a **multi-stage** Dockerfile, and why use it?

### 1. Definition
A **multi-stage build** uses **multiple `FROM` stages** in one Dockerfile. Earlier stages compile or package; the **final stage** copies only **runtime artifacts** (e.g. JAR) into a **small** base image.

### 2. Benefits
- **Smaller attack surface** — no Maven/Gradle JDK toolchain in prod image.
- **Faster deploys** — less to pull and scan.
- **Clear separation** between **build environment** and **runtime environment**.

### 3. Example

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src
COPY pom.xml .
COPY src ./src
RUN mvn -q -DskipTests package

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /src/target/my-service.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 4. Middle+ answer (as on interview)

**Multi-stage** means **more than one `FROM`** in a single Dockerfile, each stage is a separate temporary image. The **build stage** carries **JDK, Maven/Gradle, compilers, test tooling**—whatever you need to produce `target/*.jar`. The **final stage** is a **small JRE** (or **distroless**) image and only **`COPY --from=build`** the **runtime artifact** (JAR, static assets, native binary).

Why interviewers like this: the image that reaches prod **does not contain** `~/.m2`, `src/`, or `apt` build packages—**less to scan**, **less CVE surface**, **faster `docker pull`**, smaller blast radius if the registry is compromised.

Advanced bits I sometimes add: **BuildKit cache mounts** (`RUN --mount=type=cache`) for Maven/Gradle deps so CI stays fast **without** baking the cache into layers; **never bake secrets** into `RUN` lines (they end up in history)—use **build args from CI secrets** carefully or **runtime** injection. One-liner: **multi-stage separates “factory” from “product.”**

---

<a id="q101"></a>

## 101. What are layers in images?

### Question Restatement
How are Docker **layers** formed, and why do they matter for **build and pull** performance?

### 1. Definition
Images are composed of ordered **read-only layers**. Each Dockerfile step that modifies the filesystem typically creates a **new layer** on top of the previous stack.

### 2. Why layers matter
- **Build cache** — if a layer’s inputs are unchanged, Docker **reuses** cached layer output.
- **Registry transport** — pushes/pulls send **missing** layers only.
- **Deduplication** — shared bases (same distro) are stored once on disk where possible.

### 3. Optimization pattern
Put **slow-changing** steps early (`dependency:go-offline`, `COPY pom` + `RUN mvn dependency:resolve`) and **fast-changing** app sources **late** to maximize cache hits.

### 4. Middle+ answer (as on interview)

Conceptually an image is an **ordered stack of read-only layers**; each Dockerfile instruction that changes the filesystem usually creates a **new layer** on top. The engine stores layers **by content hash**, so identical layers are **reused** across images on the same host.

**Three practical wins:** (1) **build cache**—if `COPY pom.xml` and `RUN mvn dependency:go-offline` did not change inputs, Docker **reuses** cached layers and your CI build drops from minutes to seconds; (2) **registry transport**—`docker push`/`pull` ships only **layers the remote does not have**; (3) **deduplication** on disk when many services share the same base (`eclipse-temurin`, `alpine`, …).

**Ordering recipe** I say out loud: dependencies and **rarely changing** files **high** in the Dockerfile, frequently changing **application code low**—otherwise every commit invalidates the cache at the first `COPY . .` line. I also mention **smaller layers** help with **security review** (fewer opaque blobs) and that **`RUN` chaining** vs many `RUN`s is a trade-off between **readability** and **layer count**.

---

<a id="q102"></a>

## 102. What is Volume in Docker? What are Volume types?

### Question Restatement
What is a Docker **volume**, and what **mount types** exist?

### 1. Definition
A **volume** (broadly: persistent mount) stores data **outside** the container’s ephemeral writable layer so it survives **container delete** (depending on type) and can be shared.

### 2. Common mount types

| Type | What it is | Typical use |
|------|--------------|-------------|
| **Named volume** | Docker-managed storage (`postgres_data:/var/lib/postgresql/data`) | DB data, broker data |
| **Bind mount** | Host path mapped into container (`./local.conf:/app/config`) | dev configs, hot reload |
| **tmpfs** | RAM-backed, not persisted | sensitive scratch, strict no-disk policy |

### 3. Interview nuance
**Bind mounts** couple container to host layout—great locally, risky for prod portability. **Named volumes** align better with “data is an attachment.”

### 4. Middle+ answer (as on interview)

When people say “volume” they often mean **any persistent mount**—technically Docker distinguishes **volumes**, **bind mounts**, and **tmpfs**. The core idea is the same: **do not store durable state only in the container writable layer** unless you are OK losing it on `rm`.

- **Named volumes** — Docker (or the plugin) allocates storage under the hood; in Compose you write `postgres_data:/var/lib/postgresql/data`. Great for **DBs**, **Kafka logs**, anything that should **survive container recreation** and not depend on a developer’s folder layout.

- **Bind mounts** — map a **host path** (`./config:/app/config`). Perfect for **local iteration** (hot reload, local certs), but in prod they **tie you to host filesystem layout** and permissions—usually I avoid them in real clusters unless there is a deliberate reason (agent sockets, mounted secrets from orchestrator).

- **tmpfs** — RAM-backed, **not persisted** after restart; useful for **sensitive temp files** or policies that forbid writing secrets to disk.

Backup/ops angle I sometimes add: named volumes still need **backup strategy** (dump DB, snapshot volume driver)—**Docker does not magically backup your data**. Tagline: **compute is cattle, data is explicit infrastructure.**

---

<a id="q103"></a>

## 103. What is Network in Docker? What are Network types?

### Question Restatement
How does Docker **networking** connect containers, and which **drivers** matter?

### 1. Definition
Docker **networks** control which containers can reach each other, how **DNS names** resolve (service discovery on a user-defined bridge), and how traffic reaches the host (`publish` ports).

### 2. Main drivers

| Driver | Behavior |
|--------|-----------|
| **bridge** | Default single-host L2 network; containers on same user-defined bridge resolve each other by **name**. |
| **host** | Container shares host network namespace (Linux nuances; no port mapping). |
| **none** | Disabled networking. |
| **overlay** | Multi-host (Swarm / similar patterns); relevant conceptually before Kubernetes CNI. |
| **macvlan** | Containers appear on physical L2 segment with own MAC (advanced). |

### 3. Practice
Use a **custom bridge network** per stack instead of default `bridge` when you want stable **DNS aliases** and isolation from unrelated containers.

### 4. Middle+ answer (as on interview)

Docker **networks** answer three questions at once: **what IP am I on**, **who can I talk to without leaving the host**, and **how does traffic get in/out** (`-p 8080:8080` publishes container port to host interfaces).

On a single host the workhorse is **bridge**: attach containers to a **user-defined bridge network** (Compose does this) and you get **DNS-based service discovery**—`http://postgres:5432` resolves inside the stack. That is nicer than the legacy default bridge where **name resolution** is weaker and isolation from random containers is messier.

Other drivers for completeness: **`host`** means the container uses the **host network namespace** (no port mapping; fast, but you lose network isolation—**Linux-only** semantics matter); **`none`** disables networking for hard isolation tests; **overlay** is the **multi-host** story you still hear from **Docker Swarm** days; **macvlan** gives containers **their own MAC** on the LAN—niche. Interview punchline: **network attachment is security and topology policy**, not just “plumbing”—same image on **internal-only** network vs **published** port is a different attack surface.

---

<a id="q104"></a>

## 104. What is docker-compose?

### Question Restatement
What is **Docker Compose**, and what problem does it solve?

### 1. Definition
**Compose** is a **YAML specification** (`compose.yaml` / legacy `docker-compose.yml`) that describes **services**, **networks**, **volumes**, **env**, **dependencies**, and **published ports** so you can run a **multi-container** stack with one command (`docker compose up`).

### 2. What it gives teams
- **Reproducible local stacks** (app + Postgres + Redis + broker).
- **Shared developer onboarding** — same topology as integration tests sometimes.
- **Sensible defaults** for healthchecks and restart policies (when configured).

### 3. Limits (senior framing)
Compose is excellent for **dev/small prod** on a single host or simple deployments; **large-scale production** usually moves to **Kubernetes** or a managed platform for scheduling, rolling upgrades, autoscaling, and multi-tenant networking policy.

### 4. Middle+ answer (as on interview)

**Docker Compose** is a **declarative YAML** file (`compose.yaml`, legacy `docker-compose.yml`) that describes **services** (containers to run), **images/build contexts**, **ports**, **env files**, **depends_on**, **healthchecks**, **restart policies**, **networks**, and **volumes**. `docker compose up` (v2 CLI) brings the whole graph up on **one Docker engine**—the whole team gets the same **Postgres + Redis + app** stack with one command.

What it solves for me: **onboarding** (“clone repo, compose up”), **reproducible demos**, and often a **baseline for integration tests** (same dependency versions as CI if we align images). I can **override** env per developer with `.env` (gitignored) without touching committed defaults.

What I **do not** claim: Compose is **not** a full production orchestrator at hyperscale—no built-in **bin packing**, **self-healing across nodes**, **network policies**, **rolling upgrades**, **HPA**. For that you graduate to **Kubernetes**, Nomad, ECS, etc. Compose can still be valid for **single-node prod**, **edge**, or **small internal tools**—the senior answer is **fit for purpose**, not religion.

---

<a id="q105"></a>

## 105. What are Integration tests? What is the difference between integration tests and unit tests?

### Question Restatement
Define **integration tests** and contrast them with **unit tests** in scope, dependencies, and goals.

### 1. Definition
**Integration tests** verify that **multiple real components** work together: your code **plus** Spring context slices, **SQL** against a real dialect, **HTTP** adapters, serialization, transaction boundaries, etc.—with **fewer mocks** than unit tests, but still bounded in scope.

### 2. Comparison (recap)

| Aspect | Unit test | Integration test |
|--------|-----------|-------------------|
| **Scope** | One class/method behavior | Collaboration across modules or infrastructure |
| **Dependencies** | Test doubles | Real DB/broker/container or in-memory equivalents where faithful |
| **Speed** | Very fast | Slower |
| **Failure signal** | Wrong logic in isolation | Wiring, SQL, schema, config, classpath issues |

### 3. Interview nuance
Neither label is a law—**the pyramid** still applies: many fast unit tests, fewer integration tests that hit **true boundaries** you do not trust to mocks.

### 4. Middle+ answer (as on interview)

**Unit tests** isolate a **small unit**—usually a class or pure function—with **test doubles** for collaborators. They are **fast**, give **precise failure location**, and are perfect for **algorithmic** logic, edge cases, and pure domain rules. What they **cannot** prove is that Spring **wires the right bean**, that Hibernate generates **SQL your dialect accepts**, or that Jackson + validation + controller annotations combine the way you think.

**Integration tests** intentionally **remove some mocks** and boot **real subsystems together**: slice or full **`ApplicationContext`**, **real JDBC** to Postgres (often **Testcontainers**), **transaction rollback** behavior, **Flyway migrations**, sometimes **MockMvc** hitting real filters. They are **slower**, harder to debug when they fail, and can become **flaky** if every test boots the world—so I keep them **narrow and few**, focused on **seams I do not trust** to fakes.

I still respect the **test pyramid**: lots of unit tests, a **thin layer** of integration tests on **risky boundaries**, and **e2e** only where product risk demands it. Naming is fuzzy—what matters is **what risk you buy with each test type**. More detail in **question 66** (`Модуль 5 — Testing + CI-CD.md`).

---

<a id="q106"></a>

## 106. What are Testcontainers? How are they used in Integration tests?

### Question Restatement
What is **Testcontainers**, and how does it improve **integration tests**?

### 1. Definition
**Testcontainers** is a library that **starts disposable Docker containers** from JUnit tests (lifecycle managed per test class or suite): **Postgres**, **Kafka**, **LocalStack**, etc., using **real** implementations instead of fragile in-memory simulators.

### 2. How it is used
- Add dependency (`org.testcontainers:*`).
- Declare `@Container static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");` (or JUnit 5 `@DynamicPropertySource` / Spring Boot **module** auto-wiring).
- Spring Boot: **`spring-boot-testcontainers`** + `@ServiceConnection` (Boot 3.1+) can inject JDBC URL properties automatically.

### 3. Trade-offs
**Pros:** high fidelity (SQL dialect, extensions), catches migration issues. **Cons:** needs **Docker** in CI, slower than pure mocks, flaky if resource limits are tight—mitigate with **reuse**, **Ryuk** tuning, and pinned image versions.

### 4. Middle+ answer (as on interview)

**Testcontainers** is a **JVM library** (JUnit 4/5 integration) that talks to the **Docker API**, **starts real containers** for dependencies—Postgres, MySQL, Kafka, Redis, LocalStack, etc.—and **waits until they are ready** (listening ports, logs) before tests run. Spring Boot tests then get **real JDBC URLs**, broker bootstrap servers, and so on—either manually via **`@DynamicPropertySource`** or cleanly via Boot 3 **`spring-boot-testcontainers`** + **`@ServiceConnection`** on the container bean.

Why I reach for it: **H2** is not Postgres—**JSON types, partial indexes, locking, migration scripts** diverge; **in-memory Kafka** is not Kafka. Testcontainers catches the bugs you only see **against the real technology**, especially after **schema changes**.

Trade-offs I always state: CI must have a **Docker daemon** (or compatible runtime), tests are **slower**, parallel suites need **enough RAM/CPU** or they flap. Mitigations: **pin image versions** (`postgres:16-alpine@sha256:…`), **reuse containers** / **singleton containers** per JVM where safe, tune **Ryuk** and timeouts, and avoid starting **ten** heavy containers per class if one shared instance is enough.

Elevator pitch: **production-shaped dependencies with throwaway lifecycle**—still not full e2e, but much more honest than mocking a database interface.

---

<a id="q107"></a>

## 107. How to load an ApplicationContext in Integration tests?

### Question Restatement
How do you bring up a **Spring `ApplicationContext`** in tests—full vs **slice**?

### 1. Full context (`@SpringBootTest`)
Loads the **entire** application context (or `@SpringBootTest(classes=…)` for partial). Use **`@AutoConfigureMockMvc`** / **`WebEnvironment.RANDOM_PORT`** when testing the web layer without a real browser.

### 2. Context slices (faster, narrower)
- **`@DataJpaTest`** — JPA + transaction; auto-configures in-memory DB unless overridden with Testcontainers `@ServiceConnection`.
- **`@WebMvcTest(YourController.class)`** — MVC slice + **MockMvc**; **security** auto-config unless excluded; collaborators **mocked**.
- **`@JsonTest`**, **`@RestClientTest`** — focused stacks.

### 3. Custom wiring
`@Import`, **`@TestConfiguration`**, **`@MockBean`** (full/some slices) to replace beans; **`@ActiveProfiles("test")`** for test-only properties.

### 4. Middle+ answer (as on interview)

Spring tests are really “**how much of the `ApplicationContext` do I pay to start?**” **`@SpringBootTest`** loads the **full** application (or a **subset** via `classes = …` / `@Import`)—good when I need **real wiring** across layers: security filters, message converters, `DataSource`, schedulers, etc. For HTTP I either use **`@AutoConfigureMockMvc`** (in-process, fast) or **`WebEnvironment.RANDOM_PORT`** + **`TestRestTemplate`/`WebTestClient`** when I want a **real socket** and full servlet stack behavior.

When the test is **narrower**, I use **slice annotations** to cut startup time and accidental coupling: **`@DataJpaTest`** for JPA repositories (still hits SQL—pair with Testcontainers if I distrust H2); **`@WebMvcTest(SomeController.class)`** loads **MVC + Jackson + validation** but **mocks** collaborators behind `@MockBean`; **`@RestClientTest`** focuses on **`RestClient`/`WebClient` beans** with a mock engine.

Cross-cutting recipe: **`@ActiveProfiles("test")`**, **`@TestConfiguration`**, **`@Sql`** for seed data when it clarifies the scenario, and for Testcontainers **`@DynamicPropertySource`** or **`@ServiceConnection`** so ports are **dynamic**—never hardcode `localhost:5432` from prod YAML into tests. The rule I repeat: **smallest context that still proves the integration claim**; otherwise `@SpringBootTest` becomes a **15-second integration-ish unit test**.

---

<a id="q108"></a>

## 108. How to mock external APIs calls in integration tests?

### Question Restatement
How do you **stub outbound HTTP** while still exercising your **client + mapping** code?

### 1. Options (from most to least “integration”)

| Approach | What it simulates | When |
|----------|-------------------|------|
| **WireMock** / **MockWebServer** | Real HTTP server with canned responses | Client stack, timeouts, retries, mapping |
| **Spring Cloud Contract / Pact** | Contract-driven stubs + consumer verification | Cross-team APIs |
| **`@MockBean` on `RestClient`/`WebClient` collaborators** | Skip network entirely | Fast but less faithful to HTTP edge cases |

### 2. Practice
Prefer **HTTP-level** stubs when your risk is **serialization, status handling, and resilience policies**. Keep fixtures **versioned** next to tests.

### 3. Pitfall
Do not “mock away” the thing you claim to integrate—if the test never hits your HTTP client adapter, it is closer to a **unit** test.

### 4. Middle+ answer (as on interview)

If I need to validate **our outbound HTTP client**—timeouts, retries, error mapping on **429/503**, **Content-Type**, gzip, large payloads—I put a **real HTTP server** in front of it: **WireMock** (Jetty-based, rich DSL) or **OkHttp MockWebServer** (lightweight). The test hits **`localhost:dynamicPort`**, so serializers, interceptors, and **connection pooling** behave like production.

For **provider/consumer** boundaries across teams I bring up **contract tests**: **Spring Cloud Contract** (producer generates stubs) or **Pact** (consumer-driven contracts) so stubs **do not drift** from reality—stubs are **versioned artifacts**, not copy-paste JSON.

**`@MockBean` on a `RestClient` wrapper** is valid when the class under test is **above** the wire (e.g. a service orchestrating calls) and the HTTP client is already covered elsewhere—but then I label the test honestly: it is **not** proving wire behavior. Anti-pattern: claiming “integration” while **never opening a socket**. I keep **fixtures** next to tests (JSON bodies, mappings) so failures are **diffs you can read in PRs**.

---

<a id="q109"></a>

## 109. How to handle security issues in integration tests?

### Question Restatement
How do you test **secured endpoints** without weakening **production security configuration**?

### 1. Principles
- **Never** disable security globally in `application.properties` used by prod.
- Prefer **`@Import` of a test-only `SecurityFilterChain`** or **`@MockBean` authentication** that applies **only under `@ActiveProfiles("test")`**.
- Use **`@WithMockUser`**, **`@WithUserDetails`**, or **`SecurityMockMvcRequestPostProcessors`** for **role/authority** assertions.

### 2. MVC integration
`@WebMvcTest` + **`@AutoConfigureMockMvc(addFilters = false)`** is sometimes used—be explicit that you are testing **controller logic**, not the **real filter chain**. For **filter behavior**, use **`@SpringBootTest` + `@AutoConfigureMockMvc`** with security enabled.

### 3. OAuth2 / JWT tests
Use **static JWT fixtures** signed with a **test key**, or **Spring Security’s test support** for resource servers; avoid calling real IdPs in unit/integration unless you intend **e2e**.

### 4. Middle+ answer (as on interview)

Production security config stays **strict**; tests get **parallel mechanisms**. Typical patterns: **`@ActiveProfiles("test")`** loading **`application-test.yml`** that only relaxes what is safe (e.g. fake OAuth issuer URL), or a dedicated **`@TestConfiguration`** that declares a **`SecurityFilterChain` bean** with order higher than prod for tests—**scoped to the test slice**, not committed as default beans.

For **method security** and MVC I combine **`@WithMockUser(roles = …)`** / **`@WithUserDetails`** with **`MockMvc`** perform calls, or **`SecurityMockMvcRequestPostProcessors`** when I need fine-grained headers. For **OAuth2 resource server** I prefer **static JWTs** signed with a **well-known test private key** loaded only in tests, or Spring Security’s **JWT `MockMvc` request post-processors** (`jwt()` / `opaqueToken()` style helpers depending on stack)—**no live call** to Okta/Keycloak in the default suite (that belongs to **e2e** with a mocked IdP or a staging realm).

**Scope honesty:** `@WebMvcTest` with **`@AutoConfigureMockMvc(addFilters = false)`** is about **controller mapping + validation**, not **filter chain behavior**—if the interview is about **CSRF, JWT filters, rate limiting**, I move to **`@SpringBootTest` + `@AutoConfigureMockMvc`** with filters **enabled** and maybe **Testcontainers** for session/redis. Red lines: no **`permitAll()`** sneaking into prod because “tests were flaky,” and no secrets in repo—**CI secrets** or **test keystores** only where appropriate.

---
