# Kubernetes Interview Preparation for Java Backend Engineers

> Target: Java backend engineer, Junior+/Middle final interview level.  
> Goal: explain Kubernetes like a production engineer, not like someone memorizing definitions.

---

# Table of Contents

- [Kubernetes Fundamentals](#kubernetes-fundamentals)
- [Cluster Architecture](#cluster-architecture)
- [Core Kubernetes Objects](#core-kubernetes-objects)
- [Networking](#networking)
- [Storage](#storage)
- [Configuration Management](#configuration-management)
- [Scaling and Reliability](#scaling-and-reliability)
- [Deployments](#deployments)
- [Helm](#helm)
- [Kubernetes in Backend Systems](#kubernetes-in-backend-systems)
- [Security Basics](#security-basics)
- [Monitoring and Observability](#monitoring-and-observability)
- [Common Production Problems](#common-production-problems)
- [Kubernetes Interview Q&A](#kubernetes-interview-qa)
- [Final Revision Checklist](#final-revision-checklist)

---

# Kubernetes Fundamentals

## What it is
Kubernetes is a platform for running containerized applications across a cluster of machines. It schedules containers, keeps them running, exposes them through networking, manages configuration, supports rolling deployments, and helps teams operate applications reliably in production.

For a Java backend engineer, Kubernetes is usually where Spring Boot services, API gateways, background workers, and sometimes infrastructure components such as Kafka, Redis, or databases are deployed.

## Why Kubernetes needs it
A single Docker container is easy to run locally, but production systems need much more:

- multiple application replicas;
- automatic restart after failures;
- traffic routing to healthy instances;
- rolling updates without downtime;
- configuration and secret injection;
- scaling based on load;
- service discovery between microservices;
- isolation between teams and environments;
- resource control so one service does not starve others.

Kubernetes was created to solve the operational problem of running many containers reliably across many machines.

## How it works internally
Kubernetes is based on a declarative desired-state model:

1. You submit YAML manifests to the Kubernetes API.
2. The cluster stores the desired state.
3. Controllers continuously compare desired state with actual state.
4. If actual state differs, Kubernetes takes action.

Example: if a Deployment says `replicas: 3`, Kubernetes tries to keep three Pods running. If one Pod crashes or a node disappears, Kubernetes creates replacement Pods.

At a high level:

- the **control plane** makes decisions and stores cluster state;
- **worker nodes** run application workloads;
- **Pods** are the smallest deployable runtime units;
- **Services** provide stable networking to changing Pods;
- **controllers** reconcile desired state.

## Production usage
In production, teams usually do not manually run containers. They define manifests or Helm charts and apply them through CI/CD pipelines. A typical backend service deployment includes:

- a Docker image for the Spring Boot application;
- a Deployment with multiple replicas;
- resource requests and limits;
- readiness and liveness probes;
- ConfigMaps and Secrets;
- a Service for internal access;
- optionally an Ingress or API gateway route for external access;
- metrics scraping and log collection.

Kubernetes becomes the execution environment for backend services, while CI/CD decides when and how new versions are deployed.

## Common production problems
- Treating Kubernetes like “Docker on servers” and ignoring Services, probes, resources, and rollout behavior.
- Running only one replica of a critical stateless service.
- Missing readiness probes, causing traffic to reach an application before it is ready.
- Wrong memory limits for Java services, causing `OOMKilled` restarts.
- Hardcoding Pod IPs instead of using Services and DNS.
- Storing application state inside containers.
- Deploying databases without understanding persistence, backups, failover, and ordering.

## Tradeoffs
Advantages:

- standard platform for containerized workloads;
- strong deployment and scaling model;
- self-healing behavior;
- cloud-provider portability at the workload level;
- large ecosystem.

Disadvantages:

- operational complexity;
- YAML and configuration sprawl;
- debugging requires understanding multiple layers;
- stateful workloads are harder than stateless services;
- bad configuration can create production incidents quickly.

## Interview traps
Interviewers usually check whether you understand that Kubernetes is not just a container runner. They expect you to know:

- Kubernetes manages desired state, not individual containers manually.
- Pods are replaceable and ephemeral.
- Services provide stable access to dynamic Pods.
- Docker alone does not provide orchestration.
- Kubernetes helps with reliability, but it does not magically make bad applications reliable.

## Key points to remember
- Kubernetes runs containers across a cluster.
- You declare desired state; Kubernetes reconciles actual state.
- Control plane manages decisions; worker nodes run workloads.
- Pods are ephemeral; Services provide stable networking.
- Production Kubernetes is about reliability, deployment, scaling, and operations.

---

# Cluster Architecture

## What it is
Kubernetes cluster architecture is the set of control-plane and worker-node components that make the cluster function. The control plane stores state and makes decisions. Worker nodes run Pods and report their status.

## Why Kubernetes needs it
A distributed container platform needs separate responsibilities:

- accepting user/API requests;
- storing cluster state;
- deciding where workloads should run;
- creating and monitoring containers;
- routing traffic;
- reacting to failures.

Splitting responsibilities makes Kubernetes extensible and fault-tolerant.

## How it works internally

![Альтернативный текст](https://images.openai.com/static-rsc-4/7JgYDJi9Ypn3oxa27vCSF7wBhedaSV42-XNM641YmCRyX1E9tmAaStHmwINiZ7Ubw1yTtCvjQyUqEjh-0P69oSvYyo8IAiVmA-M5MoBDuOpxtbsYFnneqEwUEUqV8I3Ftn_4IdVXK5fU3T6_DhmlmDzpKvdOZfbvdIOM1JP9gZ6dkOqkmATv4MKWYuMA5YXt?purpose=fullsize)

The most important components are:

### API Server
The API Server is the front door of Kubernetes. `kubectl`, CI/CD tools, controllers, and operators talk to it. It validates requests, applies authentication and authorization, and writes state to `etcd`.

### etcd
`etcd` is the strongly consistent key-value store that keeps cluster state. It stores information such as Deployments, Pods, Services, ConfigMaps, Secrets, and node status.

### Scheduler
The Scheduler watches for Pods without assigned nodes and chooses a suitable worker node based on CPU/memory requests, constraints, taints, tolerations, affinity rules, and available capacity.

### Controller Manager
The Controller Manager runs controllers that continuously reconcile desired state. For example, the Deployment controller ensures ReplicaSets exist, and the ReplicaSet controller ensures the right number of Pods exists.

### kubelet
`kubelet` runs on every worker node. It receives Pod specifications from the API Server and asks the container runtime to start, stop, and monitor containers. It also reports node and Pod status back to the control plane.

### kube-proxy
`kube-proxy` runs on worker nodes and implements Service networking rules, typically using Linux networking mechanisms. It helps traffic sent to a Service reach one of the matching Pods.

### Container runtime
The container runtime actually runs containers. Common runtimes include containerd and CRI-O. Kubernetes does not require Docker as the runtime.

## Production usage
In managed Kubernetes services such as EKS, GKE, and AKS, teams often do not directly manage the control plane. They still need to understand the architecture because failures often involve:

- API Server access problems;
- Pods stuck in `Pending` due to scheduling issues;
- node pressure;
- kubelet failures;
- Service routing or DNS problems;
- cluster capacity limits.

For a backend engineer, the most practical focus is understanding what happens when you deploy a service:

1. CI/CD applies a Deployment manifest.
2. API Server stores the desired state.
3. Controllers create or update ReplicaSets and Pods.
4. Scheduler assigns Pods to worker nodes.
5. kubelet starts containers through the runtime.
6. Services route traffic to ready Pods.

## Common production problems
- Pods stuck in `Pending` because no node has enough requested CPU or memory.
- Pods scheduled but failing because the image cannot be pulled.
- Node is `NotReady`, so workloads cannot run there reliably.
- kubelet cannot start containers due to runtime or disk issues.
- Service exists but has no endpoints because Pods do not match labels or are not ready.
- Control plane/API access issues block deployments even though existing Pods may keep running.

## Tradeoffs
Advantages:

- clear separation of responsibilities;
- extensible architecture;
- controllers allow automatic recovery;
- workers can be added or removed.

Disadvantages:

- more moving parts to understand;
- debugging often requires checking several layers;
- control-plane health affects deployment and reconciliation;
- worker-node failures can still impact applications if replicas are insufficient.

## Interview traps
Interviewers may ask for “master vs worker node” components. Modern terminology is **control plane** and **worker nodes**, but you should understand both names.

A common trap is saying the control plane runs application containers. In normal architecture, worker nodes run application Pods. Some clusters may allow control-plane nodes to run workloads, but production clusters usually isolate them.

## Key points to remember
- API Server is the entry point.
- etcd stores cluster state.
- Scheduler places Pods on nodes.
- Controllers reconcile desired state.
- kubelet manages containers on a worker node.
- kube-proxy supports Service networking.
- Container runtime runs the actual containers.

---

# Core Kubernetes Objects

## What it is
Kubernetes objects are declarative resources that describe what should exist in the cluster. They include Pods, Deployments, Services, ConfigMaps, Secrets, Jobs, and other workload abstractions.

## Why Kubernetes needs it
Kubernetes needs objects so users can describe application intent rather than manually controlling processes. Instead of saying “run this container on server X,” you say “run three replicas of this application with these labels, resources, probes, and environment variables.”

## How it works internally
The key core objects are:

### Pods
A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share network namespace, storage volumes, and lifecycle. Usually, a backend service Pod contains one main application container and sometimes sidecars.

### ReplicaSets
A ReplicaSet ensures a specified number of identical Pods exists. You rarely create ReplicaSets directly; Deployments manage them.

### Deployments
A Deployment manages stateless applications and rolling updates. It creates ReplicaSets and gradually replaces old Pods with new Pods during releases.

### StatefulSets
A StatefulSet manages stateful applications that need stable network identity, stable storage, and ordered startup/shutdown. It is used for systems such as databases, Kafka, and some clustered services.

### DaemonSets
A DaemonSet runs one Pod on every matching node. It is commonly used for logging agents, monitoring agents, network plugins, or security agents.

### Jobs and CronJobs
A Job runs a task to completion. A CronJob runs Jobs on a schedule. Backend examples include batch imports, cleanup tasks, report generation, or periodic reconciliation jobs.

### Labels and selectors
Labels are key-value metadata attached to objects. Selectors find objects by labels. Services, Deployments, NetworkPolicies, and monitoring tools heavily depend on correct labels.

### Namespaces
Namespaces divide cluster resources into logical groups such as `dev`, `staging`, `prod`, or team-specific spaces. They help with organization, access control, quotas, and naming separation.

## Production usage
A typical Spring Boot service uses:

- Deployment for the stateless application;
- Service for stable internal access;
- ConfigMap for non-sensitive configuration;
- Secret for credentials;
- HPA for scaling;
- Ingress/API gateway for external routing;
- labels for traffic routing, monitoring, and ownership;
- namespace per environment or team.

Stateful applications may use StatefulSets, but many production teams prefer managed cloud services for databases and Kafka because operations are complex.

## Common production problems
- Confusing Pod with Deployment and manually creating Pods.
- Incorrect labels causing Services to route to no Pods or the wrong Pods.
- Using Deployments for stateful systems that need stable identity.
- Running scheduled tasks as always-running services instead of Jobs/CronJobs.
- Putting unrelated services in one namespace with no access or quota boundaries.
- Assuming a restarted Pod will keep local filesystem data.

## Tradeoffs
Advantages:

- objects model real operational needs;
- controllers automate lifecycle management;
- labels provide flexible grouping;
- namespaces improve organization.

Disadvantages:

- YAML complexity increases with production requirements;
- wrong selectors can cause subtle traffic failures;
- not every workload fits a simple Deployment;
- too many namespaces can complicate operations.

## Interview traps
Interviewers often check whether you know:

- a Pod is not the same as a container;
- a Deployment is not the same as a Service;
- ReplicaSets maintain count, Deployments manage rollout;
- StatefulSets are not “just Deployments with storage”; they also provide identity and ordering;
- labels and selectors are central to Kubernetes behavior.

## Key points to remember
- Use Deployments for stateless backend services.
- Use StatefulSets for workloads needing stable identity and storage.
- Services select Pods using labels.
- Namespaces provide logical isolation.
- Jobs and CronJobs are for finite or scheduled work.

---

# Networking

## What it is
Kubernetes networking connects Pods, Services, and external clients. It provides a model where Pods can communicate across nodes, Services expose stable virtual endpoints, and Ingress or LoadBalancer resources handle external traffic.

## Why Kubernetes needs it
Pods are ephemeral. They are created, destroyed, rescheduled, and assigned new IPs. Production applications need stable ways to communicate even when individual Pods change.

Kubernetes networking solves:

- Pod-to-Pod communication;
- stable service addresses;
- internal service discovery;
- external access;
- load distribution across replicas;
- optional network isolation.

## How it works internally
### Cluster networking model
Each Pod gets its own IP. Kubernetes expects Pods to communicate with each other without NAT inside the cluster network, though the exact implementation depends on the CNI plugin.

### Pod-to-Pod communication
A Pod can call another Pod IP directly, but this is not recommended for applications because Pod IPs are temporary.

### Services
A Service provides a stable virtual IP and DNS name for a group of Pods selected by labels. Traffic sent to the Service is load-balanced to ready endpoints.

### ClusterIP
`ClusterIP` is the default Service type. It exposes a service only inside the cluster. This is the standard choice for internal microservice communication.

### NodePort
`NodePort` opens a port on every worker node and forwards traffic to a Service. It is useful for simple exposure or debugging, but it is rarely the preferred production external-entry model.

### LoadBalancer
`LoadBalancer` asks the cloud provider to provision an external load balancer that routes traffic to the Service.

### Ingress
Ingress defines HTTP/HTTPS routing rules, usually implemented by an Ingress Controller such as NGINX, Traefik, HAProxy, or a cloud controller. It can route by host and path, terminate TLS, and centralize external traffic management.

### DNS and service discovery
Kubernetes DNS creates names for Services. A service named `orders` in namespace `prod` can typically be reached as:

- `orders`
- `orders.prod`
- `orders.prod.svc.cluster.local`

Applications should call Service DNS names, not Pod IPs.

### Internal vs external traffic
Internal traffic stays inside the cluster through ClusterIP Services. External traffic usually enters through a cloud load balancer, Ingress Controller, or API gateway, then reaches Services and Pods.

### Network Policies
NetworkPolicies define allowed network flows between Pods and namespaces. They are like application-level firewall rules for cluster traffic, but they require a CNI plugin that enforces them.

## Production usage
A common backend traffic flow:

1. User calls `https://api.example.com/orders`.
2. DNS resolves to a cloud load balancer.
3. Load balancer forwards to an Ingress Controller or API gateway.
4. Ingress routes by host/path to the `orders-service` Service.
5. Service forwards traffic to a ready `orders` Pod.
6. The `orders` Pod calls `payment-service.prod.svc.cluster.local` for internal communication.

For Java microservices, teams usually expose only edge services externally. Internal services use ClusterIP Services and DNS.

## Common production problems
- Service selector does not match Pod labels, so the Service has no endpoints.
- Pods are running but not ready, so they are not included as Service endpoints.
- Calling Pod IPs directly instead of Service DNS.
- Exposing internal services with LoadBalancer or public Ingress by mistake.
- Missing NetworkPolicies in multi-tenant clusters.
- DNS resolution failures causing service-to-service communication errors.
- Assuming Service load balancing behaves like advanced application load balancing with retries, circuit breaking, and request awareness.

## Tradeoffs
Advantages:

- stable discovery for dynamic Pods;
- simple internal service communication;
- flexible external routing through Ingress;
- NetworkPolicies improve isolation.

Disadvantages:

- network debugging can be hard;
- Service load balancing is relatively basic;
- Ingress behavior depends on controller implementation;
- NetworkPolicies add operational complexity;
- exposing services incorrectly creates security risk.

## Interview traps
Interviewers often check whether you know:

- Service is not the application itself; it is a stable network abstraction.
- ClusterIP is internal only.
- Ingress is for HTTP/HTTPS routing and requires a controller.
- DNS points to Services, not individual application semantics.
- Readiness affects whether a Pod receives traffic.

## Key points to remember
- Never rely on Pod IPs for service communication.
- Use ClusterIP for internal services.
- Use Ingress/API gateway/LoadBalancer for external access.
- Service endpoints depend on labels and readiness.
- NetworkPolicies restrict traffic but require enforcement support.

---

# Storage

## What it is
Kubernetes storage provides a way for Pods to use persistent data even though Pods themselves are ephemeral. The main abstractions are PersistentVolumes, PersistentVolumeClaims, and StorageClasses.

## Why Kubernetes needs it
Containers are designed to be replaceable. If a container or Pod restarts, local container filesystem changes may be lost. Some workloads, however, need durable storage:

- databases;
- message brokers;
- file uploads;
- caches with persistence;
- batch processing outputs;
- stateful application components.

Kubernetes separates compute lifecycle from storage lifecycle.

## How it works internally
### PersistentVolume
A PersistentVolume, or PV, represents actual storage available to the cluster, such as a cloud disk, network volume, or local storage.

### PersistentVolumeClaim
A PersistentVolumeClaim, or PVC, is a request for storage by a workload. A Pod uses a PVC, and Kubernetes binds it to a suitable PV.

### StorageClass
A StorageClass defines how storage should be provisioned dynamically. For example, it may specify SSD-backed cloud disks, standard disks, replication behavior, or expansion support.

### Stateful workloads
Stateful workloads often use StatefulSets with volume claim templates. Each replica gets its own stable PVC. If a Pod is recreated, it can reattach to the same storage.

### Storage lifecycle
The lifecycle depends on reclaim policies and storage provider behavior. Deleting a Pod usually does not delete the PVC. Deleting a PVC may release or delete the underlying storage depending on configuration.

## Production usage
For Java backend services, the best production pattern is usually stateless application Pods:

- keep user/session state outside the container;
- store files in object storage such as S3-compatible storage;
- store business data in databases;
- use external managed Postgres, MongoDB, Kafka, or Redis when possible.

When teams run stateful systems in Kubernetes, they need careful planning for:

- backups;
- restore testing;
- disk performance;
- failover;
- data corruption handling;
- node affinity;
- upgrades;
- storage expansion.

## Common production problems
- Writing important data to container local filesystem.
- Deleting PVCs accidentally and losing data.
- Assuming Kubernetes automatically backs up PV data.
- Using the wrong storage class for database performance.
- Multiple Pods writing to storage that supports only single-writer access.
- Rescheduling stateful Pods across zones where volumes cannot attach.
- Treating Kafka/Postgres/Mongo as easy to run just because a Helm chart exists.

## Tradeoffs
Advantages:

- separates Pod lifecycle from data lifecycle;
- supports dynamic provisioning;
- enables stateful workloads when needed;
- integrates with cloud storage providers.

Disadvantages:

- storage failures are more serious than stateless Pod failures;
- stateful systems are harder to scale and upgrade;
- performance depends heavily on the storage backend;
- backups and disaster recovery are not automatic;
- cloud disks often have zone and attachment constraints.

## Interview traps
Interviewers often check whether you know:

- PV is the storage resource; PVC is the request for it.
- Pods should generally be stateless.
- StatefulSet helps with identity and storage but does not solve database operations by itself.
- StorageClass enables dynamic provisioning.
- Persistence does not mean backups.

## Key points to remember
- Containers are ephemeral; persistent data needs volumes.
- PV = actual storage; PVC = request for storage.
- StorageClass defines provisioning behavior.
- Stateless backend services are easier to scale and recover.
- Databases in Kubernetes require serious operational maturity.

---

# Configuration Management

## What it is
Configuration management in Kubernetes separates application configuration from container images. ConfigMaps store non-sensitive configuration, and Secrets store sensitive values such as credentials and tokens.

## Why Kubernetes needs it
The same container image should run in multiple environments: dev, staging, and production. Environment-specific values should not be baked into the image.

Examples:

- database URL;
- feature flags;
- external service endpoints;
- log levels;
- credentials;
- API keys;
- TLS certificates.

## How it works internally
### ConfigMaps
ConfigMaps store plain configuration data as key-value pairs or files. Pods consume them as environment variables, command-line arguments, or mounted files.

### Secrets
Secrets store sensitive data. They are base64-encoded in manifests and should be encrypted at rest in production. They can also be consumed as environment variables or mounted files.

### Environment variables
Environment variables are simple and common for Spring Boot apps. Spring Boot maps env vars like `SPRING_DATASOURCE_URL` to configuration properties.

### Configuration injection
Configuration is injected into Pods at startup or mounted as files. Environment-variable changes generally require a Pod restart to take effect. Mounted ConfigMap/Secret files may update eventually, but many applications still need reload logic.

## Production usage
A typical Spring Boot service uses:

- ConfigMap for non-sensitive properties;
- Secret for database passwords, OAuth client secrets, JWT signing keys, or API tokens;
- external secret managers in more mature setups;
- separate values per environment;
- CI/CD or GitOps to apply configuration changes.

Production teams avoid rebuilding images just to change environment-specific configuration.

## Common production problems
- Putting passwords in ConfigMaps.
- Committing real Secrets into Git.
- Assuming base64 encoding means encryption.
- Forgetting to restart Pods after changing env-based configuration.
- Overusing environment variables for large structured config.
- Accidentally logging secret values at application startup.
- Inconsistent configuration between replicas during rollout.

## Tradeoffs
Advantages:

- clean separation between image and environment;
- easy environment-specific configuration;
- integrates naturally with Spring Boot;
- Secrets provide a standard sensitive-data mechanism.

Disadvantages:

- Kubernetes Secrets need additional hardening;
- configuration sprawl can become hard to manage;
- runtime reload is not automatic for most apps;
- env vars are visible in some debugging contexts.

## Interview traps
Interviewers often check whether you understand:

- ConfigMaps are not for secrets.
- Kubernetes Secrets are not automatically safe just because they are called Secrets.
- Changing a ConfigMap may not automatically reconfigure a running application.
- Config should not be baked into Docker images.

## Key points to remember
- ConfigMap = non-sensitive config.
- Secret = sensitive config, but must be protected properly.
- Same image should run across environments.
- Spring Boot works well with env-var-based configuration.
- Secret management is a security responsibility, not just a YAML feature.

---

# Scaling and Reliability

## What it is
Scaling and reliability features keep applications available under normal operation, failures, and changing load. Kubernetes supports replicas, autoscaling, self-healing, probes, rolling updates, and rollbacks.

## Why Kubernetes needs it
Production systems fail constantly:

- processes crash;
- nodes disappear;
- deployments introduce bugs;
- traffic spikes;
- services start slowly;
- dependencies are temporarily unavailable.

Kubernetes provides mechanisms to reduce manual intervention and keep healthy replicas serving traffic.

## How it works internally
### Replica scaling
A Deployment declares how many replicas should run. Kubernetes creates or removes Pods to match the desired count.

### Horizontal Pod Autoscaler
HPA adjusts replica count based on metrics such as CPU usage, memory usage, or custom metrics like request rate. It does not optimize application performance automatically; it only changes Pod count based on configured signals.

### Self-healing
If a Pod crashes or a node fails, Kubernetes tries to restore desired state by restarting containers or scheduling replacement Pods.

### Restart policies
Restart policies define whether containers should be restarted. Deployments typically use `Always`. Jobs often use `OnFailure` or `Never` depending on the task.

### Liveness probes
A liveness probe checks whether the application is alive. If it fails repeatedly, Kubernetes restarts the container.

### Readiness probes
A readiness probe checks whether the application is ready to receive traffic. If it fails, the Pod stays running but is removed from Service endpoints.

### Startup probes
A startup probe gives slow-starting applications more time before liveness checks begin. This is useful for Java applications with long initialization.

### Rolling updates
A rolling update gradually replaces old Pods with new Pods while keeping service available.

### Rollbacks
A rollback returns a Deployment to a previous ReplicaSet version when a release fails.

### High availability basics
High availability requires multiple replicas, proper anti-affinity or topology spreading, healthy probes, enough cluster capacity, and no single critical dependency without failover.

## Production usage
For Spring Boot services, production manifests commonly include:

- at least two replicas for critical services;
- resource requests and limits;
- readiness endpoint such as `/actuator/health/readiness`;
- liveness endpoint such as `/actuator/health/liveness`;
- startup probe for slow services;
- HPA for variable traffic;
- Pod disruption budgets for controlled maintenance;
- rolling update settings like `maxUnavailable` and `maxSurge`.

## Common production problems
- Liveness probe checks a dependency such as database and restarts healthy apps during DB outage.
- Readiness probe is missing, so traffic hits the app before startup completes.
- HPA cannot scale because resource requests are missing.
- Java memory limit is too low, causing `OOMKilled`.
- Scaling Pods does not help because the bottleneck is database connections or Kafka partitions.
- Too aggressive autoscaling causes instability.
- One replica only, so rolling deployment causes downtime.

## Tradeoffs
Advantages:

- automatic recovery from many failures;
- easy horizontal scaling of stateless services;
- safer deployments;
- probes protect users from unhealthy Pods.

Disadvantages:

- bad probes can create outages;
- autoscaling can hide inefficient code or overload dependencies;
- more replicas cost more money;
- Kubernetes cannot fix broken application logic;
- scaling stateful systems is much harder than scaling stateless services.

## Interview traps
Interviewers often check whether you know:

- readiness failure stops traffic but does not restart the container;
- liveness failure restarts the container;
- HPA needs metrics and meaningful resource requests;
- more Pods do not always mean better performance;
- self-healing does not replace application resilience patterns.

## Key points to remember
- Replicas provide redundancy and capacity.
- HPA scales based on metrics, not magic.
- Liveness restarts; readiness removes from traffic.
- Startup probes help slow Java apps.
- Reliability requires correct application design and Kubernetes configuration.

---

# Deployments

## What it is
Deployment strategy defines how new application versions are released to users. Kubernetes supports rolling and recreate strategies directly, while blue/green and canary are often implemented with Services, Ingress, service meshes, or progressive delivery tools.

## Why Kubernetes needs it
Deployments are risky. A new version may have bugs, slow startup, broken migrations, bad configuration, or performance regressions. Release strategies reduce downtime and limit blast radius.

## How it works internally
### Rolling deployment
Old Pods are gradually replaced by new Pods. Kubernetes controls how many Pods can be unavailable and how many extra Pods can be created during the rollout.

### Recreate deployment
All old Pods are stopped before new Pods start. This can cause downtime but may be necessary for applications that cannot run old and new versions simultaneously.

### Blue/Green deployment
Two environments exist: blue is current production, green is the new version. After validation, traffic is switched from blue to green.

### Canary deployment
A small percentage of traffic goes to the new version first. If metrics and logs look good, traffic gradually increases.

### Zero downtime deployments
Zero downtime requires more than Kubernetes rolling update. The application must support:

- readiness probes;
- graceful shutdown;
- backward-compatible APIs;
- backward-compatible database migrations;
- enough replicas;
- no shared mutable local state.

## Production usage
Backend teams often use rolling deployments for ordinary services. They use canary or blue/green for high-risk changes, critical APIs, or systems with strict availability requirements.

For Spring Boot services, production readiness includes:

- handling `SIGTERM` gracefully;
- stopping acceptance of new requests during shutdown;
- allowing in-flight requests to finish;
- keeping database migrations backward compatible;
- exposing meaningful health endpoints.

## Common production problems
- New Pods become ready too early and receive traffic before warmup finishes.
- Old Pods are killed before finishing in-flight requests.
- Database migration is incompatible with the old app version during rolling deployment.
- Only one replica exists, so rolling update causes visible downtime.
- Canary is deployed but no metrics are checked, so it only gives false confidence.

## Tradeoffs
Advantages:

- rolling updates are simple and built in;
- blue/green enables fast rollback by switching traffic;
- canary limits blast radius;
- recreate is simple for incompatible workloads.

Disadvantages:

- blue/green needs duplicate capacity;
- canary requires good metrics and routing control;
- rolling requires version compatibility;
- recreate causes downtime;
- advanced strategies increase operational complexity.

## Interview traps
Interviewers often check whether you understand:

- rolling deployment is not automatically zero downtime;
- canary is about partial traffic and observation;
- blue/green is about switching between two complete environments;
- database changes are often the hardest part of safe deployment;
- rollback is easy only if data and schema are compatible.

## Key points to remember
- Rolling is default and common.
- Recreate stops old before starting new.
- Blue/green switches traffic between two versions.
- Canary gradually exposes users to risk.
- Zero downtime requires application and database compatibility.

---

# Helm

## What it is
Helm is a package manager and templating tool for Kubernetes. It packages related Kubernetes manifests into charts and allows environment-specific configuration through values files.

## Why Kubernetes needs it
Raw Kubernetes YAML becomes repetitive and hard to maintain across many services and environments. Helm helps teams reuse templates, parameterize configuration, version releases, and install complex applications consistently.

## How it works internally
### Helm charts
A chart is a directory containing Kubernetes templates, default values, metadata, and optional helper templates.

### Templating
Helm templates use Go-template syntax to generate Kubernetes YAML from parameters.

### Values files
`values.yaml` contains default configuration. Teams often use additional files such as `values-dev.yaml`, `values-staging.yaml`, and `values-prod.yaml`.

### Releases
When a chart is installed into a cluster, Helm creates a release. Upgrades apply new rendered manifests and keep release history.

## Production usage
Teams use Helm to package:

- internal microservice deployment templates;
- third-party infrastructure such as ingress controllers, Prometheus, Grafana, Redis, Kafka, or cert-manager;
- common labels, probes, resources, and deployment settings.

A backend team may own service-specific values while a platform team owns shared chart templates.

## Common production problems
- Overcomplicated templates that are harder to understand than plain YAML.
- Different environments drifting because values files are inconsistent.
- Installing third-party charts without reviewing security, persistence, and resource defaults.
- Secrets accidentally stored in Helm values files committed to Git.
- Helm upgrade succeeds syntactically but deploys a bad application configuration.

## Tradeoffs
Advantages:

- reduces repetitive YAML;
- supports reusable deployment patterns;
- versioned releases;
- large ecosystem of charts.

Disadvantages:

- templating can become complex;
- rendered output must still be reviewed;
- chart abstraction can hide important Kubernetes details;
- secret handling needs care.

## Interview traps
Interviewers often check whether you know Helm does not replace Kubernetes. Helm generates and applies Kubernetes manifests. You still need to understand Deployments, Services, ConfigMaps, Secrets, and probes.

## Key points to remember
- Helm packages Kubernetes manifests.
- Charts are templates plus values.
- Values files customize environments.
- Helm is useful but can hide complexity.
- Always understand the rendered Kubernetes resources.

---

# Kubernetes in Backend Systems

## What it is
Kubernetes in backend systems means using Kubernetes as the runtime platform for microservices, APIs, workers, and supporting infrastructure around Java backend applications.

## Why Kubernetes needs it
Modern backend systems need repeatable deployment, horizontal scaling, isolation, service discovery, configuration management, and safe releases. Kubernetes provides these capabilities consistently across teams and services.

## How it works internally
A typical Spring Boot microservice in Kubernetes includes:

- a container image built from the application artifact;
- a Deployment for replicas and rollout;
- a Service for stable internal networking;
- ConfigMaps and Secrets for configuration;
- probes mapped to Spring Boot Actuator endpoints;
- resource requests and limits;
- optional HPA;
- logs written to stdout/stderr;
- metrics exposed for Prometheus;
- traces exported to a tracing backend.

### API Gateway exposure
External clients usually call an API Gateway or Ingress, not every backend service directly. The gateway handles routing, authentication integration, rate limiting, TLS, and sometimes request transformation.

### Internal services
Internal services are usually exposed as ClusterIP Services. They communicate through DNS names and HTTP/gRPC/messaging protocols.

### Stateful databases and brokers
Kafka, MongoDB, and Postgres can run on Kubernetes, but they require careful state management, stable identity, persistent storage, backups, upgrade planning, and operational expertise. Many teams use managed cloud services instead.

### Why databases are harder to scale
Stateless services can be scaled by adding replicas. Databases must preserve consistency, handle replication, coordinate writes, maintain indexes, manage storage, and recover from failures without data loss.

## Production usage
For a Java backend engineer, practical Kubernetes understanding means being able to explain:

- how a Spring Boot app starts inside a Pod;
- how traffic reaches it;
- how it discovers dependencies;
- how configuration is injected;
- how readiness protects users;
- how logs and metrics are collected;
- why memory limits must account for JVM behavior;
- why database migrations must be compatible with rolling deployment.

Example service-to-service flow:

1. `api-gateway` receives public request.
2. Gateway calls `orders-service` through Kubernetes Service DNS.
3. `orders-service` calls `payment-service` and publishes event to Kafka.
4. Metrics are scraped by Prometheus.
5. Logs go to centralized logging.
6. Traces connect the request across services.

## Common production problems
- JVM memory not aligned with container limits.
- Missing graceful shutdown causes dropped requests during rollout.
- Readiness probe uses only “process is up” but not “server is ready.”
- Too many database connections after scaling replicas.
- Scaling consumers beyond Kafka partition count with no benefit.
- Internal service exposed publicly by mistake.
- Database migration breaks old version during rolling update.
- Missing timeouts and retries cause cascading failures.

## Tradeoffs
Advantages:

- strong platform for stateless Java services;
- consistent deployment across environments;
- easy horizontal scaling;
- standard observability integration;
- supports microservice architecture.

Disadvantages:

- local development differs from cluster behavior;
- debugging requires app and platform knowledge;
- stateful dependencies remain complex;
- network latency and failure handling become important;
- Kubernetes does not replace good application architecture.

## Interview traps
Interviewers often check if a backend engineer can connect Kubernetes concepts to application behavior:

- What happens during startup?
- When should a Pod receive traffic?
- How does the service find another service?
- What happens when a dependency is down?
- What happens during rolling deployment with database migration?
- Why might scaling Pods overload a database?

## Key points to remember
- Use Deployments for Spring Boot stateless services.
- Use Services and DNS for communication.
- Use Actuator for probes and metrics.
- Keep containers stateless.
- Be careful with JVM memory, DB connections, and migrations.

---

# Security Basics

## What it is
Kubernetes security is about controlling who can access the cluster, what workloads can do, what traffic is allowed, how secrets are protected, and which services are exposed.

## Why Kubernetes needs it
A Kubernetes cluster often runs many services, teams, and environments. A mistake in exposure, permissions, or secret handling can compromise data or production systems.

## How it works internally
### Cluster isolation
Isolation is usually built with namespaces, RBAC, NetworkPolicies, separate clusters for sensitive environments, and resource quotas.

### Internal trusted network
Many teams initially treat the cluster network as trusted. This is convenient but dangerous. Production systems should assume internal services can be misconfigured or compromised.

### RBAC basics
Role-Based Access Control defines what users, groups, and service accounts can do. Permissions should be least-privilege.

### Secrets handling
Secrets should be encrypted at rest, access-controlled, rotated, and ideally integrated with external secret managers. Applications should not log secrets.

### Ingress security
Ingress should enforce TLS, restrict administrative endpoints, integrate with authentication where appropriate, and avoid exposing internal-only services.

### Why exposing internal services is dangerous
Internal services often lack public-grade authentication, rate limiting, input hardening, or abuse protection. Exposing them directly can bypass gateway controls.

## Production usage
Backend teams usually interact with security through:

- namespace-level permissions;
- service accounts for applications;
- Secrets for credentials;
- private ClusterIP Services;
- Ingress rules for public APIs;
- TLS certificates;
- image scanning and admission policies;
- network restrictions between namespaces or service groups.

## Common production problems
- Giving developers or CI/CD cluster-admin permissions unnecessarily.
- Exposing actuator, admin, metrics, or internal APIs publicly.
- Reusing production secrets in lower environments.
- Storing secrets in Git.
- Using default service accounts with broad permissions.
- No NetworkPolicies in shared clusters.
- Pulling images from untrusted registries.

## Tradeoffs
Advantages:

- Kubernetes provides standard security primitives;
- RBAC and namespaces support team isolation;
- NetworkPolicies reduce lateral movement;
- Secrets provide central handling for sensitive values.

Disadvantages:

- secure-by-default depends heavily on cluster setup;
- RBAC can be complex;
- NetworkPolicies can break communication if poorly designed;
- Secrets need external hardening for serious production use.

## Interview traps
Interviewers may check whether you know:

- Kubernetes Secrets are base64-encoded, not automatically encrypted in every setup.
- Internal services should not be publicly exposed.
- RBAC controls API permissions, not application-level user authorization.
- NetworkPolicies control network traffic, not Java method calls or business access.

## Key points to remember
- Use least privilege.
- Keep internal services internal.
- Treat Secrets as sensitive even inside the cluster.
- Secure Ingress and admin endpoints.
- Do not assume internal network equals safe network.

---

# Monitoring and Observability

## What it is
Monitoring and observability help teams understand system health, detect incidents, debug failures, and improve reliability. Monitoring tells you whether known signals are healthy. Observability helps you investigate unknown problems using logs, metrics, and traces.

## Why Kubernetes needs it
Kubernetes systems are dynamic. Pods move, restart, scale, and change IPs. Without centralized visibility, it is difficult to know:

- which version is running;
- why a Pod restarted;
- whether traffic increased;
- whether latency is caused by app code, database, network, or resource pressure;
- whether a deployment caused errors.

## How it works internally
### Logs
Containers should write logs to stdout/stderr. Node agents collect logs and send them to systems such as Elasticsearch/OpenSearch, Loki, Splunk, or another logging platform.

### Metrics
Metrics are numeric time-series data. Examples include CPU, memory, request count, error rate, latency percentiles, JVM heap usage, GC time, and database connection pool usage.

### Traces
Traces follow a request across services. They show which service calls happened, how long each step took, and where errors occurred.

### Prometheus
Prometheus scrapes metrics from applications and Kubernetes components. Java services often expose metrics through Spring Boot Actuator and Micrometer.

### Grafana
Grafana visualizes metrics and logs through dashboards. Teams use it to track service health, latency, error rates, saturation, JVM behavior, and infrastructure usage.

### ELK stack
ELK usually means Elasticsearch, Logstash, and Kibana, though modern stacks may use Beats, Fluent Bit, or OpenSearch. It is commonly used for centralized logging and search.

### Health monitoring
Health monitoring checks whether services are alive, ready, and meeting SLOs such as latency and error-rate targets.

### Alerting basics
Alerts should be actionable. Good alerts usually focus on user impact, error rate, latency, saturation, and critical dependency failures.

### Resource monitoring
Resource monitoring tracks CPU, memory, disk, network, node pressure, Pod restarts, and Kubernetes object status.

### Application monitoring
Application monitoring tracks business and service-level signals such as request latency, HTTP status codes, queue lag, Kafka consumer lag, database pool exhaustion, and JVM metrics.

## Production usage
For Java services, a practical observability setup includes:

- structured JSON logs with correlation/request IDs;
- Prometheus metrics via Actuator/Micrometer;
- Grafana dashboards for RED metrics: rate, errors, duration;
- JVM dashboards for heap, GC, threads, and class loading;
- alerts for high error rate, high latency, repeated restarts, and resource saturation;
- distributed tracing with OpenTelemetry or similar tooling.

## Common production problems
- Logs exist only inside Pods and disappear after restart.
- No correlation ID, so debugging across services is painful.
- Alerting on CPU only while users experience high latency.
- Too many noisy alerts that nobody trusts.
- Missing application metrics; only infrastructure metrics exist.
- Prometheus scrape endpoint exposed publicly.
- Dashboards show averages that hide p95/p99 latency issues.

## Tradeoffs
Advantages:

- faster incident response;
- better release confidence;
- capacity planning;
- visibility into JVM and service behavior;
- easier root-cause analysis.

Disadvantages:

- telemetry has cost;
- too much logging can be expensive and noisy;
- bad dashboards create false confidence;
- tracing every request can be costly without sampling;
- alerts need continuous tuning.

## Interview traps
Interviewers often check whether you distinguish:

- logs vs metrics vs traces;
- monitoring vs observability;
- infrastructure health vs user-impact health;
- liveness/readiness probes vs real monitoring;
- dashboards vs actionable alerts.

## Key points to remember
- Logs explain events.
- Metrics show trends and health signals.
- Traces follow requests across services.
- Prometheus collects metrics; Grafana visualizes them.
- Observability must include application-level signals, not only Kubernetes status.

---

# Common Production Problems

## What it is
Common Kubernetes production problems are recurring failure patterns involving application crashes, resource limits, probes, networking, images, scheduling, autoscaling, and stateful workloads.

## Why Kubernetes needs it
Knowing Kubernetes concepts is not enough. In production, engineers must reason from symptoms to causes and know where to look first.

## How it works internally
### CrashLoopBackOff
A container starts, crashes, restarts, and repeats. Kubernetes backs off restart attempts. Causes include bad config, missing env vars, failed startup, wrong command, dependency failure, or application exception.

### OOMKilled
The container exceeded its memory limit and was killed by the runtime. Java services often hit this when heap and non-heap memory are not aligned with container limits.

### Failing probes
Liveness probe failures restart containers. Readiness probe failures remove Pods from traffic. Misconfigured probes can cause outages even when the app is mostly healthy.

### Networking issues
A Service may have no endpoints, DNS may fail, NetworkPolicy may block traffic, Ingress rules may be wrong, or the target app may not listen on the expected port.

### Misconfigured resources
Wrong CPU/memory requests can cause scheduling problems. Wrong limits can cause throttling or OOM kills.

### Image pull failures
Kubernetes cannot pull the container image due to wrong image name, missing tag, registry auth failure, network issue, or missing imagePullSecret.

### Resource starvation
Nodes or Pods lack CPU, memory, disk, or network capacity. This causes latency, evictions, failed scheduling, or node pressure.

### Bad autoscaling
HPA may not work due to missing metrics or requests. It may also scale too slowly, too aggressively, or overload dependencies.

### Stateful workload issues
Persistent volume attachment, replication, backup, ordering, corruption, and failover issues can cause serious incidents.

## Production usage
A practical troubleshooting flow:

1. Check Pod status: `kubectl get pods`.
2. Describe the Pod: `kubectl describe pod <pod>`.
3. Check logs: `kubectl logs <pod>` and previous logs with `--previous`.
4. Check Deployment rollout: `kubectl rollout status deployment/<name>`.
5. Check Service endpoints: `kubectl get endpoints` or EndpointSlices.
6. Check events: `kubectl get events --sort-by=.metadata.creationTimestamp`.
7. Check resource usage: `kubectl top pods` and dashboards.
8. Check app metrics, logs, and traces.

## Common production problems
- Debugging only Kubernetes status and ignoring application logs.
- Restarting Pods without understanding the root cause.
- Rolling back application version when the real issue is configuration or dependency failure.
- Scaling the app when the database is the bottleneck.
- Ignoring recent deployments as a likely incident cause.
- Not checking previous container logs after restarts.

## Tradeoffs
Advantages:

- Kubernetes exposes status, events, and logs through standard tools;
- many failures follow recognizable patterns;
- controllers can recover from simple failures automatically.

Disadvantages:

- symptoms can be misleading;
- many layers are involved;
- automatic restarts can hide root causes;
- stateful incidents are difficult and risky.

## Interview traps
Interviewers often ask troubleshooting scenarios to test reasoning. They care less about memorized commands and more about whether you can narrow the problem:

- Is the Pod scheduled?
- Did the image pull?
- Did the app start?
- Is it ready?
- Does the Service select it?
- Is traffic routed correctly?
- Are resources sufficient?
- Did a recent deployment change behavior?

## Key points to remember
- `CrashLoopBackOff` means repeated start/crash cycles.
- `OOMKilled` means memory limit was exceeded.
- Readiness affects traffic; liveness affects restart.
- Service traffic depends on selectors and ready endpoints.
- Always combine Kubernetes events with application logs and metrics.

---

# Kubernetes Interview Q&A

## 162. What is the concept of Container orchestration?

Container orchestration is the automated management of containerized applications across multiple machines. It includes scheduling containers, restarting failed containers, scaling replicas, exposing services, managing configuration, rolling out new versions, and handling resource allocation.

In production, orchestration means you do not SSH into servers and manually run `docker run`. Instead, you declare desired state, such as “run three replicas of this service,” and the orchestrator keeps that state.

**Real-world example:** an `orders-service` has three replicas. One node fails. Kubernetes detects that one replica disappeared and schedules a replacement Pod on another node.

**Tradeoffs:** orchestration improves reliability and consistency, but it adds platform complexity and requires correct configuration.

**Common interview traps:**

- Saying orchestration is only “running Docker containers.”
- Forgetting service discovery, scaling, deployment, and self-healing.
- Assuming orchestration removes the need for application resilience.

---

## 163. What are the advantages of using Kubernetes?

Kubernetes provides a standard way to deploy and operate containerized applications in production.

Main advantages:

- **self-healing:** restarts failed containers and replaces failed Pods;
- **scaling:** supports manual scaling and HPA-based autoscaling;
- **service discovery:** Services and DNS provide stable communication;
- **rolling updates and rollbacks:** safer releases with controlled rollout;
- **resource management:** CPU/memory requests and limits;
- **configuration management:** ConfigMaps and Secrets;
- **portability:** similar workload model across cloud providers;
- **ecosystem:** Helm, Prometheus, Ingress controllers, operators, GitOps tools.

**Real-world example:** a Spring Boot payment API can be deployed with two replicas, a readiness probe, a ConfigMap for environment settings, a Secret for database credentials, and a Service for internal access.

**Tradeoffs:** Kubernetes is powerful but complex. For a small app with one server, it may be overkill. For many services and teams, the operational consistency is often worth it.

**Common interview traps:**

- Claiming Kubernetes automatically makes apps highly available.
- Ignoring the need for multiple replicas, good probes, and resilient application design.
- Overstating portability: workload YAML may be portable, but storage, load balancers, and IAM are often cloud-specific.

---

## 164. What is a Pod in Kubernetes?

A Pod is the smallest deployable unit in Kubernetes. It contains one or more containers that share the same network namespace, IP address, ports, and volumes.

Usually, a Java backend Pod has one main Spring Boot container. Sometimes it also has sidecars, such as a logging agent, proxy, or service mesh sidecar.

**Real-world example:** an `orders-service` Pod runs the Spring Boot application container. The Pod gets its own IP, but clients should not call this IP directly because Pods are temporary.

**Tradeoffs:** Pods are lightweight and replaceable, which is good for scaling and recovery. The downside is that local Pod state is not reliable.

**Common interview traps:**

- Saying a Pod equals a container. A Pod can contain multiple containers.
- Treating Pod IPs as stable.
- Creating standalone Pods manually for production instead of using Deployments or Jobs.

---

## 165. What is a Node in Kubernetes? What is the difference between Master and Worker nodes? What components are inside each?

A Node is a machine in the Kubernetes cluster. It can be a VM or physical server.

Modern terminology uses **control plane node** instead of “master node.” The control plane manages the cluster. Worker nodes run application workloads.

**Control plane responsibilities:**

- accept API requests;
- store desired and actual state;
- schedule Pods;
- run controllers;
- coordinate cluster reconciliation.

**Control plane components:**

- API Server;
- etcd;
- Scheduler;
- Controller Manager;
- sometimes Cloud Controller Manager.

**Worker node responsibilities:**

- run application Pods;
- manage containers;
- report health and status;
- implement Service networking.

**Worker node components:**

- kubelet;
- kube-proxy;
- container runtime such as containerd;
- Pods.

**Real-world example:** when a Deployment is applied, the control plane decides where Pods should run. Worker-node kubelets start the containers.

**Common interview traps:**

- Saying etcd runs on every worker node.
- Saying the Scheduler starts containers directly. It only assigns Pods to nodes; kubelet starts them.
- Forgetting kubelet and container runtime on worker nodes.

---

## 166. What is a Service in Kubernetes? How Service Discovery in Kubernetes works?

A Service is a stable network abstraction over a set of Pods. It gives changing Pods a stable IP and DNS name.

Pods are selected by labels. When matching Pods are ready, Kubernetes includes them as Service endpoints. Traffic to the Service is forwarded to one of those endpoints.

Service discovery works mainly through Kubernetes DNS. If a Service is named `payment-service` in namespace `prod`, other Pods can call:

```text
http://payment-service.prod.svc.cluster.local
```

or often just:

```text
http://payment-service
```

from the same namespace.

**Real-world example:** `orders-service` calls `payment-service` using the Service DNS name, not a Pod IP. If payment Pods are replaced during deployment, the Service continues to work.

**Tradeoffs:** Services provide simple load balancing and discovery, but not advanced application-level behavior like retries, circuit breaking, or request-aware routing.

**Common interview traps:**

- Thinking Service equals Deployment.
- Forgetting that Services use labels/selectors.
- Not knowing that readiness controls whether a Pod receives Service traffic.

---

## 167. What are Pod Controllers? What is the difference between Deployment and StatefulSet?

Pod controllers manage Pods and keep them in the desired state. Common controllers include Deployments, ReplicaSets, StatefulSets, DaemonSets, Jobs, and CronJobs.

A **Deployment** manages stateless applications. It creates ReplicaSets, supports rolling updates, rollbacks, and scaling. It is the standard choice for Spring Boot APIs.

A **StatefulSet** manages stateful applications that need stable identity, ordered startup/shutdown, and stable persistent storage. Each replica has a predictable name and usually its own PVC.

**Real-world examples:**

- Deployment: `orders-service`, `payment-service`, API gateway.
- StatefulSet: Kafka broker, MongoDB replica, ZooKeeper, some database clusters.

**Tradeoffs:** Deployments are simpler and easier to scale. StatefulSets support stateful behavior but are operationally more complex.

**Common interview traps:**

- Saying StatefulSet is only a Deployment with a volume.
- Using StatefulSet for every application that has a database connection.
- Forgetting stable network identity and ordered behavior.

---

## 168. What are Persistent Volume and Persistent Volume Claims in Kubernetes?

A PersistentVolume, or PV, is actual storage available in the cluster. A PersistentVolumeClaim, or PVC, is a request for storage by a workload.

A Pod does not usually care about the exact disk implementation. It references a PVC. Kubernetes binds the PVC to a matching PV or dynamically provisions one through a StorageClass.

**Real-world example:** a Postgres Pod uses a PVC requesting `100Gi` of storage. Kubernetes binds it to a cloud disk. If the Pod restarts, it can reattach to the same data.

**Tradeoffs:** PV/PVC separates application specs from storage details, but storage is still complex. Backups, performance, corruption, and failover are not solved automatically.

**Common interview traps:**

- Confusing PV and PVC.
- Thinking deleting a Pod deletes the persistent data.
- Thinking persistence means backup.

---

## 169. What is StorageClass? What are the types of StorageClass?

A StorageClass defines how Kubernetes dynamically provisions storage. It describes the storage provider and parameters such as disk type, performance tier, reclaim policy, volume expansion, and binding mode.

There is no universal fixed list of StorageClass “types” because they depend on the cluster and cloud provider. In practice, teams define classes such as:

- standard HDD-backed storage;
- SSD or premium storage;
- encrypted storage;
- regional or replicated storage;
- local storage;
- network file storage with shared access.

**Real-world example:** production Postgres may use a fast SSD StorageClass, while temporary batch output may use cheaper standard storage.

**Tradeoffs:** StorageClasses make provisioning easy and consistent, but choosing the wrong class can cause poor performance, high cost, or availability problems.

**Common interview traps:**

- Giving a fake universal list of StorageClass types.
- Ignoring cloud-provider dependency.
- Forgetting reclaim policy and volume binding behavior.

---

## 170. What are ConfigMap and Secret? What are the differences between them?

A ConfigMap stores non-sensitive configuration. A Secret stores sensitive configuration such as passwords, tokens, keys, and certificates.

Both can be injected into Pods as environment variables or mounted files.

Key differences:

- ConfigMap is for plain configuration.
- Secret is intended for sensitive data.
- Secret values are base64-encoded in manifests and should be encrypted at rest.
- Access to Secrets should be tightly controlled with RBAC.

**Real-world example:**

- ConfigMap: `SPRING_PROFILES_ACTIVE=prod`, feature flag, external API URL.
- Secret: database password, OAuth client secret, JWT signing key.

**Tradeoffs:** ConfigMaps and Secrets separate config from images. But Secrets require hardening; base64 is not encryption.

**Common interview traps:**

- Saying Secrets are secure because they are base64-encoded.
- Putting passwords in ConfigMaps.
- Forgetting that environment-variable changes usually require Pod restart.

---

## 171. What is Horizontal Pod AutoScaler?

Horizontal Pod Autoscaler, or HPA, automatically changes the number of Pod replicas based on metrics such as CPU usage, memory usage, or custom metrics.

It is commonly used for stateless services with variable load.

**Real-world example:** an API service normally runs three replicas. When CPU usage stays above the target threshold, HPA scales it to six replicas. When traffic drops, it scales down.

**Tradeoffs:** HPA improves elasticity, but it needs good metrics and resource requests. Scaling Pods can overload downstream systems such as databases if connection pools are not controlled.

**Common interview traps:**

- Thinking HPA scales nodes. HPA scales Pods; Cluster Autoscaler scales nodes.
- Forgetting that CPU-based HPA needs CPU requests.
- Assuming HPA helps if the bottleneck is database locks, external API latency, or Kafka partition count.

---

## 172. What are probes in Kubernetes? What happens when a liveness probe fails? What happens when a readiness probe fails?

Probes are health checks Kubernetes uses to understand container state.

- **Startup probe:** checks whether the application has finished starting.
- **Liveness probe:** checks whether the application is alive.
- **Readiness probe:** checks whether the application can receive traffic.

If a **liveness probe fails** repeatedly, Kubernetes restarts the container.

If a **readiness probe fails**, Kubernetes keeps the container running but removes the Pod from Service endpoints, so it stops receiving traffic.

**Real-world example:** a Spring Boot app uses `/actuator/health/liveness` for liveness and `/actuator/health/readiness` for readiness. During startup, readiness should fail until the HTTP server and required initialization are complete.

**Tradeoffs:** Probes improve reliability but bad probes cause incidents. A liveness probe should not fail just because a database is temporarily down; otherwise, all app Pods may restart during a DB outage.

**Common interview traps:**

- Confusing liveness and readiness behavior.
- Making liveness depend on external dependencies.
- Missing startup probes for slow Java services.

---

## 173. What is Helm chart?

A Helm chart is a package of Kubernetes manifests with templates and values. It lets teams install and configure applications consistently across environments.

A chart may include Deployments, Services, ConfigMaps, Secrets, Ingress rules, HPAs, and other resources.

**Real-world example:** a company has a common Spring Boot Helm chart. Each service provides values such as image name, replica count, resources, probes, and environment variables.

**Tradeoffs:** Helm reduces repetitive YAML, but complex templates can become hard to debug. You must still understand the Kubernetes resources Helm renders.

**Common interview traps:**

- Saying Helm replaces Kubernetes.
- Not knowing that values files customize rendered manifests.
- Trusting third-party charts without reviewing defaults.

---

## 174. What are deployment strategies? What is the difference between Blue/Green deployment and Canary?

Deployment strategies define how new versions are released.

Common strategies:

- **Rolling:** gradually replace old Pods with new Pods.
- **Recreate:** stop old Pods, then start new Pods.
- **Blue/Green:** run two complete environments and switch traffic from old to new.
- **Canary:** send a small percentage of traffic to the new version, then increase gradually.

Blue/Green uses two full versions and switches traffic at once after validation. Canary gradually exposes real traffic to the new version and observes metrics before full rollout.

**Real-world example:**

- Blue/Green: deploy `green`, run smoke tests, switch API gateway traffic from `blue` to `green`.
- Canary: route 5% of users to version `v2`, monitor errors and latency, then increase to 25%, 50%, and 100%.

**Tradeoffs:** Blue/Green enables quick rollback but needs duplicate capacity. Canary limits blast radius but requires traffic splitting and strong observability.

**Common interview traps:**

- Saying rolling update is always zero downtime.
- Forgetting database backward compatibility.
- Calling any gradual rollout “blue/green.”

---

## 175. What is CD? What is the difference between Delivery and Deployment?

CD can mean Continuous Delivery or Continuous Deployment.

**Continuous Delivery** means every change is built, tested, packaged, and made ready for production, but production release requires a manual approval or trigger.

**Continuous Deployment** means every change that passes the pipeline is automatically deployed to production without manual approval.

**Real-world example:**

- Delivery: merge to `main`, CI builds image, tests pass, staging deploy succeeds, production deploy waits for approval.
- Deployment: merge to `main`, tests pass, canary starts automatically, and production rollout completes automatically if metrics are healthy.

**Tradeoffs:** Continuous Deployment is faster but requires excellent automated tests, observability, rollback, and team confidence. Continuous Delivery gives more human control but can slow releases.

**Common interview traps:**

- Confusing CI with CD.
- Saying CD always means automatic production deployment.
- Ignoring release safety mechanisms such as canary, rollback, and monitoring.

---

## Additional: How would you monitor a Java service running in Kubernetes?

I would monitor it at three levels: application, container/Pod, and user impact.

Application signals:

- request rate;
- error rate;
- latency percentiles;
- JVM heap and GC;
- thread pools;
- database connection pool usage;
- business metrics where relevant.

Kubernetes signals:

- Pod restarts;
- readiness failures;
- CPU and memory usage;
- OOMKilled events;
- Deployment rollout status;
- node pressure.

User-impact signals:

- p95/p99 latency;
- 5xx rate;
- failed transactions;
- SLO burn rate.

**Real-world example:** expose Spring Boot Actuator metrics through Micrometer, scrape them with Prometheus, visualize in Grafana, and alert on high error rate and latency.

**Common interview traps:**

- Monitoring only CPU and memory.
- Using averages instead of percentiles.
- Having dashboards but no actionable alerts.

---

## Additional: What is observability and how is it different from monitoring?

Monitoring checks known health signals: CPU, memory, error rate, latency, and uptime. Observability is the ability to understand why the system behaves a certain way, especially for unknown problems.

Observability is usually built from three pillars:

- logs;
- metrics;
- traces.

**Real-world example:** monitoring says `orders-service` latency increased. Observability helps determine that latency increased because calls to `payment-service` became slow after a deployment.

**Tradeoffs:** Observability improves debugging but adds cost, storage volume, instrumentation work, and operational complexity.

**Common interview traps:**

- Saying observability is just logging.
- Ignoring traces in microservice systems.
- Treating probes as full observability.

---

## Additional: What is Prometheus used for in Kubernetes?

Prometheus is used to collect, store, and query metrics. In Kubernetes, it commonly scrapes metrics from applications, nodes, Kubernetes components, and exporters.

Java services usually expose Prometheus-compatible metrics through Spring Boot Actuator and Micrometer.

**Real-world example:** Prometheus scrapes `/actuator/prometheus` from `orders-service` and stores metrics such as HTTP request duration, status counts, JVM heap usage, and database pool usage.

**Tradeoffs:** Prometheus is powerful for metrics and alerting, but it is not a log store or distributed tracing system. High-cardinality labels can cause performance and cost problems.

**Common interview traps:**

- Using Prometheus for logs.
- Creating labels with user IDs or request IDs, causing high cardinality.
- Thinking Prometheus automatically knows application-specific metrics without instrumentation.

---

## Additional: What is Grafana used for?

Grafana is a visualization and dashboarding tool. It connects to data sources such as Prometheus, Loki, Elasticsearch, or cloud monitoring systems and displays metrics, logs, and sometimes traces.

**Real-world example:** a Grafana dashboard for a Spring Boot service shows request rate, error rate, p95 latency, Pod restarts, CPU/memory, JVM heap, GC pauses, and database pool usage.

**Tradeoffs:** Grafana makes signals visible, but dashboards are only useful if they show meaningful metrics. A beautiful dashboard with irrelevant averages does not help during incidents.

**Common interview traps:**

- Saying Grafana collects metrics. Usually Prometheus collects metrics; Grafana visualizes them.
- Building dashboards without alerts.
- Ignoring service-level and user-impact metrics.

---

## Additional: How should logging work in Kubernetes?

Applications should write logs to stdout/stderr. Kubernetes and node-level logging agents collect those logs and send them to a centralized logging backend.

For Java services, logs should be structured, include timestamp, level, service name, trace ID or correlation ID, request path, error details, and relevant business context.

**Real-world example:** `orders-service` logs JSON to stdout. Fluent Bit collects logs from nodes and sends them to Elasticsearch/OpenSearch or Loki. Engineers search logs by `traceId` during incidents.

**Tradeoffs:** Detailed logs help debugging, but excessive logs increase cost and noise. Sensitive data must not be logged.

**Common interview traps:**

- Writing logs only to local files inside the container.
- Forgetting correlation IDs.
- Logging passwords, tokens, or personal data.

---

## Additional: How do you troubleshoot a Pod in CrashLoopBackOff?

I would follow a layered approach:

1. Check Pod status with `kubectl get pods`.
2. Describe the Pod with `kubectl describe pod <pod>` to inspect events, exit codes, probe failures, and image issues.
3. Check current logs with `kubectl logs <pod>`.
4. Check previous container logs with `kubectl logs <pod> --previous` because the container may have restarted.
5. Verify ConfigMaps, Secrets, environment variables, command, ports, and dependencies.
6. Check whether liveness probes are killing the app too early.
7. Check resource limits and OOMKilled events.

**Real-world example:** a Spring Boot app crashes because `SPRING_DATASOURCE_URL` is missing. The Pod enters CrashLoopBackOff. Logs show configuration binding failure.

**Common interview traps:**

- Restarting the Pod repeatedly without checking logs.
- Forgetting `--previous` logs.
- Assuming CrashLoopBackOff is always Kubernetes infrastructure failure.

---

## Additional: How do you troubleshoot a Service that is not reachable?

I would check whether the traffic path is correct:

1. Does the Service exist?
2. Does the Service selector match Pod labels?
3. Are there ready endpoints?
4. Are Pods listening on the expected port?
5. Is the Service `targetPort` correct?
6. Is DNS resolving from the client Pod?
7. Is a NetworkPolicy blocking traffic?
8. If external, are Ingress rules, TLS, and load balancer configuration correct?

**Real-world example:** a Service selector uses `app: payment`, but Pods have `app: payments`. The Service has no endpoints, so calls fail even though Pods are running.

**Common interview traps:**

- Checking only whether Pods are running.
- Ignoring readiness status.
- Confusing `port`, `targetPort`, and container port.

---

## Additional: What does OOMKilled mean and how would you fix it for a Java service?

`OOMKilled` means the container exceeded its memory limit and was killed. For Java services, memory includes heap, metaspace, threads, direct buffers, code cache, and native memory, not only `-Xmx`.

Fix approach:

- inspect memory limit and actual usage;
- check JVM heap settings and container awareness;
- reduce memory usage or increase limits;
- tune `-XX:MaxRAMPercentage` or heap settings;
- check for memory leaks;
- inspect traffic spikes and large payloads;
- review thread pools and buffering;
- add alerts before memory reaches the limit.

**Real-world example:** a container has `512Mi` memory limit, but the JVM heap is configured near `512Mi`. Non-heap memory pushes total usage over the limit, and the container is killed.

**Tradeoffs:** increasing limits may fix incidents quickly but can hide leaks and increase cost. Tuning too low may cause frequent GC and latency spikes.

**Common interview traps:**

- Thinking OOMKilled is a Java exception. It is a container kill event.
- Looking only at heap and ignoring non-heap memory.
- Raising memory limits without investigating why usage grows.

---

## Additional: What is the difference between infrastructure monitoring and application monitoring?

Infrastructure monitoring tracks the platform: nodes, Pods, CPU, memory, disk, network, restarts, and cluster health.

Application monitoring tracks service behavior: request rate, errors, latency, business operations, dependency calls, queue lag, database pool usage, and JVM metrics.

**Real-world example:** infrastructure monitoring says CPU is normal. Application monitoring shows p99 latency is high because database connection pool is exhausted.

**Tradeoffs:** infrastructure monitoring is necessary but not sufficient. Application monitoring requires instrumentation but gives better user-impact visibility.

**Common interview traps:**

- Assuming green Kubernetes status means the application is healthy.
- Ignoring business and dependency metrics.
- Alerting only on resource usage.

---

# Final Revision Checklist

Before the interview, be able to explain these flows clearly:

- What happens after `kubectl apply` creates a Deployment.
- How traffic reaches a Spring Boot Pod from outside the cluster.
- How one microservice calls another through Service DNS.
- What happens when a Pod crashes.
- What happens when readiness fails versus liveness fails.
- Why stateless services are easy to scale and databases are hard.
- How rolling deployment can still cause downtime if the app is not designed correctly.
- How Prometheus, Grafana, logs, and traces work together.
- How you would debug CrashLoopBackOff, OOMKilled, and a Service with no endpoints.
