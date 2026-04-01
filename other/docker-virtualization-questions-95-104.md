# Virtualization & Docker — Questions 95–104

Interview-ready English answers focused on practical backend usage.

## Table of Contents

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

---

<a id="q95"></a>
## 95. What is Virtualization?

Virtualization is a technology that allows running multiple isolated operating systems on one physical machine using a **hypervisor**.

- A **virtual machine (VM)** includes:
  - virtual CPU/RAM/disk/network,
  - a full guest OS,
  - application runtime + app.

Hypervisor types:
- **Type 1 (bare-metal)**: runs directly on hardware (VMware ESXi, Hyper-V Server, Xen).
- **Type 2 (hosted)**: runs on top of host OS (VirtualBox, VMware Workstation).

Why used:
- server consolidation,
- strong isolation,
- run different OSes on same host,
- easier infrastructure management.

---

<a id="q96"></a>
## 96. What is the difference between Virtualization and Containerization? What are the advantages and disadvantages of both approaches?

### Core difference
- **Virtualization (VMs)** virtualizes hardware; each VM has full guest OS.
- **Containerization** virtualizes at OS level; containers share host kernel and isolate processes via namespaces/cgroups.

### Comparison

| Aspect | Virtualization (VM) | Containerization |
|---|---|---|
| Isolation | Very strong (full OS boundary) | Strong for processes, but shared kernel |
| Startup time | Slower (boot OS) | Fast (start process) |
| Resource overhead | Higher | Lower |
| Density | Lower | Higher |
| OS flexibility | Different OS per VM | Same kernel family as host |
| Typical use | Mixed OS, strict isolation, legacy apps | Microservices, CI/CD, rapid scaling |

### Pros/Cons

**VM pros**
- strong isolation/security boundary,
- OS-level flexibility.

**VM cons**
- heavier RAM/CPU/disk overhead,
- slower provisioning and scaling.

**Container pros**
- lightweight, fast startup,
- reproducible environments,
- great for DevOps pipelines and microservices.

**Container cons**
- weaker isolation than full VM,
- kernel dependency constraints,
- operational complexity at scale (orchestration, networking, observability).

---

<a id="q97"></a>
## 97. What is an Image in Docker?

A Docker **image** is an immutable template used to create containers.

Contains:
- filesystem layers (OS libs, runtime, app files),
- metadata (entrypoint, cmd, env, exposed ports).

Think of image as a **blueprint** and container as a **running instance**.

Image naming format:
- `repository:tag` (e.g., `openjdk:21-jdk-slim`).

---

<a id="q98"></a>
## 98. What is a Container in Docker?

A Docker **container** is a runnable instance of an image.

Properties:
- isolated process space (PID, network, filesystem, user namespaces),
- share host kernel,
- writable container layer on top of read-only image layers,
- can be started/stopped/restarted/deleted quickly.

Important: container is **ephemeral** by default; persistent data should use volumes/bind mounts.

---

<a id="q99"></a>
## 99. How to create an Image?

Usually with a `Dockerfile` + `docker build`.

Example (`Dockerfile`) for Spring Boot JAR:
```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Build command:
```bash
docker build -t my-order-service:1.0.0 .
```

Best practices:
- use minimal base image,
- pin versions/tags,
- avoid copying unnecessary files (`.dockerignore`),
- run as non-root user where possible.

---

<a id="q100"></a>
## 100. What are multi-staged builds?

Multi-stage build uses multiple `FROM` blocks in one Dockerfile:
- first stage builds artifact (Maven/Gradle),
- final stage copies only runtime output.

Benefits:
- much smaller final image,
- no build tools in production image,
- lower attack surface.

Example:
```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src
COPY pom.xml .
COPY src ./src
RUN mvn -q -DskipTests package

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /src/target/app.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

<a id="q101"></a>
## 101. What are layers in images?

Docker images are built as stacked read-only **layers**.

Each Dockerfile instruction (`RUN`, `COPY`, etc.) usually creates a new layer.

Why layers matter:
- **cache reuse**: unchanged layers are reused during build,
- **faster pull/push**: only missing layers transfer,
- **shared storage**: many images can share common base layers.

Optimization tip:
- put rarely changing steps earlier (dependency install),
- put frequently changing app code later.

---

<a id="q102"></a>
## 102. What is Volume in Docker? What are Volume types?

A **volume** provides persistent storage outside a container writable layer.

Why needed:
- keep data after container deletion,
- share data between containers,
- better I/O characteristics for persistence use cases.

Common mount types:
1. **Named volume**
   - managed by Docker (`mydata:/var/lib/postgresql/data`).
2. **Bind mount**
   - host path mounted into container (`./config:/app/config`).
3. **tmpfs mount**
   - in-memory, non-persistent (good for sensitive temporary data).

Interview note:
- For databases in Docker, use named volumes for persistence.

---

<a id="q103"></a>
## 103. What is Network in Docker? What are Network types?

Docker network controls container connectivity and name resolution.

Main network drivers:
1. **bridge** (default on single host)
   - containers communicate on same bridge network.
2. **host**
   - container shares host network stack (no NAT; Linux-only typical usage).
3. **none**
   - no networking.
4. **overlay**
   - multi-host networking (used in Swarm, conceptually similar in orchestrated setups).
5. **macvlan** (advanced)
   - container gets MAC/IP on physical network segment.

Best practice:
- create custom bridge networks for app stacks,
- avoid exposing unnecessary ports publicly.

---

<a id="q104"></a>
## 104. What is docker-compose?

`docker-compose` (Compose specification) is a tool to define and run multi-container applications using a single YAML file (`compose.yml`).

What it gives:
- declarative service definitions,
- networks and volumes setup,
- dependency startup order,
- local environment reproducibility.

Typical use:
- run app + DB + cache + message broker locally with one command:
```bash
docker compose up -d
```

Example services in one compose file:
- `order-service`,
- `postgres`,
- `redis`.

Interview caveat:
- Compose is great for local/dev/small deployments; for large production orchestration use Kubernetes or managed platforms.

