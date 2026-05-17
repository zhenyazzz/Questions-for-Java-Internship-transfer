# Kubernetes — Production Backend Engineering Notes

Kubernetes is a distributed reconciliation system. Backend services run as replaceable Pods, but the production behavior is governed by controllers, scheduler decisions, kubelet execution, Service endpoint programming, probe state, resource pressure, and rollout state. Most incidents come from assuming Kubernetes starts containers and routes traffic immediately; in reality it continuously reconciles desired state against partial, delayed, node-local observations.

A Deployment does not “run the app.” It creates ReplicaSets, which create Pods, which are scheduled onto nodes, which are reconciled by kubelets, which ask a container runtime to start containers, while Service endpoints are updated only for Pods considered ready. Each layer can be healthy while the business request path is broken: the Pod can be Running but not Ready, Ready but not serving correctly, serving correctly but unreachable through Service routing, reachable but saturated by CPU throttling, or healthy alone but failing through a downstream dependency.

## Control Plane and Reconciliation

The API server is the serialized mutation boundary for cluster objects. Controllers do not execute a one-time script after `kubectl apply`; they watch state and repeatedly drive actual resources toward the desired spec. This makes Kubernetes resilient to partial failure but eventually consistent: status, endpoints, scheduling, and controller reactions are not instantaneous. Debugging must account for watch delays, controller backoff, stale local caches, and conflicting controllers modifying related objects.

etcd stores cluster state, not application data. Its latency and health affect every write path: deployments, leader elections, endpoint updates, ConfigMap/Secret changes, leases, and controller progress. Large object churn, excessive Events, too many endpoints, or pathological controllers can make the control plane slow while existing Pods continue serving. When etcd or the API server is degraded, the cluster may keep running current workloads but fail to deploy, reschedule, update endpoints, or converge after node loss.

Controllers encode ownership through owner references and selectors. A Deployment owns ReplicaSets; ReplicaSets own Pods. Manual edits to child objects are usually overwritten or made irrelevant by the next reconciliation. Selector mistakes are dangerous because controllers adopt or ignore objects based on labels, not intent. A Service with the wrong selector has no useful endpoints; a Deployment selector that overlaps another workload can corrupt ownership boundaries.

The scheduler chooses a node once per unscheduled Pod. It filters nodes by hard constraints and scores feasible nodes by policy: requested CPU/memory, taints and tolerations, node affinity, pod affinity/anti-affinity, topology spread, volume binding, ports, and node conditions. It schedules from requests, not live usage. A Pod can remain Pending even when `top` shows spare CPU if requests cannot fit, topology constraints cannot be satisfied, a PVC cannot bind, or a required taint is not tolerated.

kubelet is the node-local reconciler. It watches assigned Pods, pulls images, mounts volumes, starts containers through the runtime, runs probes, reports status, enforces cgroup limits, and handles termination. If the control plane disappears temporarily, kubelet continues managing already assigned Pods using local state, but cannot receive new scheduling decisions. Node problems often surface as image pull failures, volume mount timeouts, CNI failures, disk pressure evictions, runtime errors, or Pods stuck Terminating because kubelet cannot complete cleanup.

## Pods, Lifecycle, and Failure Semantics

Pods are ephemeral execution envelopes with stable identity only for their lifetime. Pod IPs, container filesystems, and restart counts are operational observations, not durable interfaces. A replacement Pod is a different network endpoint, different process, different local disk, and possibly a different node. Anything that survives replacement must live outside the Pod: database state, object storage, persistent volumes, broker offsets, caches that tolerate loss, or recreated projections.

A Pod can be Pending, Running, Succeeded, Failed, Unknown, Ready, NotReady, Terminating, or stuck behind container-specific wait states. `Running` only means at least one container is running or starting; it does not mean the application is initialized, connected to dependencies, registered in endpoints, or able to serve requests. The useful production state is the combination of Pod phase, container state, restart reason, readiness condition, Events, logs, and endpoint membership.

Container restart behavior is local to kubelet and governed by the Pod restart policy. Deployments normally use `Always`, so a crashing Spring Boot process is restarted in the same Pod until the Pod is deleted or the node fails. CrashLoopBackOff is kubelet exponential backoff after repeated container exits. The backoff is a symptom, not a cause. The cause is in the previous container logs, exit code, Events, dependency availability, configuration, image, filesystem permissions, port binding, JVM startup failure, or cgroup kill reason.

`OOMKilled` means the kernel killed the container because its memory cgroup exceeded the limit. For Java this often comes from sizing only heap while ignoring metaspace, thread stacks, direct buffers, code cache, JIT, GC structures, decompression buffers, native libraries, TLS, Netty, and memory spikes during startup. A container with `-Xmx` equal to the Kubernetes limit is already misconfigured. The JVM must be sized below the limit with headroom for non-heap and native memory, and requests must reflect realistic working-set memory so scheduling does not pack too many JVMs onto a node.

Graceful shutdown is part of the serving contract. On deletion or rollout, kubelet sets deletion timestamp, runs preStop if configured, sends SIGTERM, waits `terminationGracePeriodSeconds`, then sends SIGKILL. Endpoint removal is asynchronous; load balancers, kube-proxy rules, ingress controllers, client connection pools, and DNS caches may continue sending traffic briefly. A Java service must stop accepting new work, fail readiness quickly, drain HTTP requests, stop consuming messages if needed, commit or abandon in-flight work deliberately, close resources, and exit before grace expires.

Probes control lifecycle decisions and traffic eligibility. Startup probes protect slow applications from premature liveness failures. Readiness probes control Service endpoint membership; failing readiness removes the Pod from normal traffic but does not restart it. Liveness probes restart containers and should detect irrecoverable deadlock, not transient downstream failure. A liveness probe that depends on Postgres, Redis, Kafka, or another service can turn a dependency incident into a restart storm.

## Networking and Traffic Flow

Kubernetes networking relies on replaceable Pod IPs behind stable abstractions. Pod IPs are useful for packet-level debugging but useless as integration points because Pods are recreated, rescheduled, and removed from endpoints. Services give a stable virtual destination, and endpoint slices track the current ready backend Pod IPs. If readiness fails, the Pod is removed from Service endpoints even while the container keeps running.

Service routing is node-level data-plane programming. kube-proxy watches Services and EndpointSlices and installs routing rules, commonly iptables or IPVS, so traffic to a ClusterIP is translated to one backend endpoint. The API object existing does not guarantee traffic delivery: selectors may match no Pods, Pods may be NotReady, kube-proxy may lag or fail, CNI routing may be broken, NetworkPolicy may deny traffic, or the target application may not listen on the declared port.

DNS resolution flows through CoreDNS. A Pod resolving `orders.default.svc.cluster.local` queries cluster DNS, which returns the Service ClusterIP or headless Service records. DNS success only proves name resolution, not endpoint health. DNS failures can come from CoreDNS overload, node-local DNS cache issues, NetworkPolicy, broken `/etc/resolv.conf`, search path surprises, or excessive client lookup behavior. Java clients can also cache DNS longer than expected, which matters for external dependencies and headless Services.

Ingress is a control-plane object interpreted by an ingress controller, not a built-in load balancer by itself. External traffic typically flows from cloud load balancer to ingress controller Pod or node port, then to a Service, then to ready endpoints. Failures can occur at TLS termination, host/path rule matching, controller sync, cloud load-balancer health checks, Service selection, NetworkPolicy, application readiness, or client timeout settings. A working ClusterIP path does not prove ingress is configured correctly.

NetworkPolicy is enforced by the CNI plugin, not by the Service object. Once policies select a Pod, ingress or egress may become default-deny depending on policy shape. Production policies must include DNS egress, metrics scraping, health checks, service mesh sidecars if present, and dependency traffic. A Pod can resolve DNS and still be blocked at connection time, or connect to a Service whose selected backend denies the return path depending on CNI behavior and policy.

## Deployments, Rollouts, and Release Failure Modes

A Deployment rollout creates a new ReplicaSet and scales old and new ReplicaSets according to `maxSurge`, `maxUnavailable`, readiness, and progress deadline. Kubernetes only knows Pod readiness, not business correctness. If readiness is too shallow, the rollout can replace all old Pods with new Pods that accept traffic but fail real requests. If readiness is too strict or slow, rollout stalls even though containers are Running.

Zero downtime is conditional. It requires enough replicas, correct readiness, graceful termination, compatible schemas, connection draining, sufficient capacity during surge, stable downstream dependencies, and clients that retry safely. A single replica Deployment cannot roll without a gap unless surge capacity and readiness timing work perfectly. A Java service with long startup and no startup probe can be killed by liveness before readiness ever succeeds. A service that keeps readiness true during shutdown can receive traffic after it has stopped accepting work.

Rollout failures are often timing bugs. The new Pod starts, passes a TCP probe, joins endpoints, receives production traffic, lazily initializes a database pool, fails migrations or cache warmup, then fails requests while Kubernetes considers it ready. Readiness must represent the ability to serve the real request path without making liveness depend on every downstream dependency. Startup sequencing belongs in startup probes, init containers, application initialization, and deployment gates, not in optimistic traffic routing.

Rollbacks are not time machines. Reverting the image does not revert database migrations, external side effects, published events, cache formats, message schemas, feature flag state, or client-visible contracts. Backward-compatible migrations, expand/contract schema changes, dual-read/write periods, and event schema compatibility are deployment mechanics, not database niceties. The safer rollback is often a forward fix behind a disabled feature flag.

Canary and blue/green releases reduce blast radius only when traffic splitting, metrics, and rollback triggers are tied to user-visible behavior. A canary that receives no representative traffic proves little. A canary that shares the same broken migration, queue consumer group, or global cache key can still corrupt global state. For backend consumers, canarying requires special care because Kafka partitions or RabbitMQ queues are not naturally percentage-routed like HTTP traffic.

## Resource Management and Scaling

Requests shape scheduling and capacity planning; limits shape runtime enforcement. CPU limits throttle through cgroups and can create high latency without obvious application errors. Memory limits kill. A service with no requests can be packed aggressively and starve neighbors. A service with inflated requests wastes nodes and blocks scheduling. The right request is an operational SLO input derived from observed p95/p99 usage under realistic load, not a copied YAML default.

HPA reacts to metrics after load exists. It cannot fix cold starts, slow image pulls, exhausted database pools, Kafka partition ceilings, global locks, synchronized cron spikes, or dependency rate limits. CPU-based HPA works poorly for services bottlenecked on I/O, locks, database waits, broker lag, or memory. Custom metrics such as queue depth, Kafka lag age, request concurrency, or p95 latency can be better, but only if they correlate with safe scale-out.

Scaling Pods can amplify bottlenecks. Ten replicas may create ten times the database connections, ten times cache warmup traffic, ten times scheduled jobs if leader election is missing, and ten times retry pressure during a downstream outage. Kafka consumers in one group cannot exceed active parallelism beyond partition count. RabbitMQ workers can exceed downstream capacity if prefetch and concurrency are not bounded. Autoscaling must include dependency budgets, connection pool sizing, and per-replica concurrency limits.

Autoscaling can oscillate. Metrics lag, readiness delay, JVM warmup, GC behavior, request bursts, and HPA stabilization windows interact. Scaling up too slowly causes backlog; scaling down too aggressively drops warm capacity and can terminate Pods with in-flight requests or consumers. Java services often need startup probes, warmup-aware readiness, conservative scale-down, and enough baseline replicas to avoid every traffic spike becoming a cold-start event.

## Java Runtime in Kubernetes

Containerized Java fails when JVM ergonomics and Kubernetes limits disagree. Modern JVMs are container-aware, but heap percentage defaults are not the same as production sizing. Heap, metaspace, direct memory, code cache, thread stacks, mmap usage, native libraries, and GC overhead all consume the same container memory limit. Thread-heavy Spring services can exhaust memory through stacks even when heap looks safe.

GC behavior is resource behavior. CPU throttling can elongate GC pauses and make latency look like application slowness. Tight memory limits increase GC frequency and reduce allocation headroom. Large heaps reduce OOM risk from allocation spikes but increase pause and warmup cost if not tuned. Observability must include heap, non-heap, direct memory if relevant, GC pause distributions, allocation rate, thread count, container CPU throttling, and RSS versus limit.

Spring Boot startup often includes classpath scanning, bean initialization, migrations, connection pool creation, cache warmup, certificate loading, JIT warmup, and dependency checks. Startup probes prevent Kubernetes from treating slow startup as liveness failure. Readiness should remain false until the process can serve expected traffic at minimum viable capacity. Liveness should be narrow and stable; killing a slow JVM under load often worsens the incident.

Thread pools are hidden capacity limits. HTTP server threads, async executors, scheduler pools, JDBC pools, Netty event loops, Kafka listener concurrency, RabbitMQ listener concurrency, and client retry pools compete for CPU and memory inside one Pod. Increasing replicas without bounding per-Pod concurrency can multiply pressure on dependencies. Production deployments should make concurrency explicit and align it with CPU requests, pool sizes, and downstream budgets.

## Stateful Workloads and Storage

State in Kubernetes is not automatically safe because a volume is mounted. PersistentVolumes preserve bytes across Pod replacement, but they do not provide database-level replication, backups, corruption recovery, quorum safety, upgrade ordering, or performance isolation. Storage attachment can fail, detach can hang, zones matter, and a rescheduled Pod may wait for a volume that is still attached to a dead node.

StatefulSets provide stable Pod names, ordered rollout semantics, and stable volume claims. They do not make Kafka, Postgres, MongoDB, Redis, or Elasticsearch operationally simple. The hard parts are quorum, fencing, replication lag, backup/restore, compaction, disk latency, anti-affinity, disruption budgets, version skew, resharding, certificate rotation, and disaster recovery. A Helm chart installs manifests; it does not operate the database.

StorageClass parameters encode real infrastructure behavior: provisioner, volume type, IOPS, throughput, reclaim policy, binding mode, expansion, topology, and snapshot support. `WaitForFirstConsumer` can avoid provisioning a zonal volume in the wrong zone, but it also ties scheduling to volume placement. Reclaim policy mistakes can either leak expensive disks or delete data unexpectedly. PVC resizing may require filesystem expansion and application support.

Running databases in Kubernetes is a platform decision, not an application convenience. Managed databases often reduce operational risk because the failure domain, backups, upgrades, and replication are owned by a specialized service. Self-hosted stateful systems can be justified for latency, cost, portability, or control, but only with explicit SLOs, operators, backup validation, restore drills, and capacity planning.

## Configuration, Secrets, and Runtime Contracts

ConfigMaps and Secrets are deployment-time contracts. Environment variable changes do not update a running process unless the Pod restarts. Mounted files may update eventually, but applications often do not reload them correctly. A ConfigMap change without a rollout may leave mixed configuration across replicas. Mature deployments use checksum annotations, rollout automation, feature flags, and explicit config versioning.

Secrets are base64-encoded Kubernetes objects unless encrypted at rest and controlled by RBAC and external secret tooling. They can leak through environment dumps, logs, crash reports, shell history, debug endpoints, and broad namespace access. Secret rotation is a runtime procedure: new secret distribution, client reload or Pod restart, overlap window, revocation, and verification. Treat secret updates like deployments.

Init containers are useful for ordering local prerequisites such as migrations checks, config rendering, or certificate setup, but they can also block rollouts indefinitely. Running destructive migrations as init containers in every replica is hazardous. Database migration should be coordinated, idempotent, observable, and compatible with old and new application versions during rollout.

## Security Boundaries That Matter Operationally

Namespaces organize resources and policy; they are not hard multi-tenant isolation by themselves. RBAC controls API actions, not network access or Linux isolation. Service accounts should be scoped per workload because any Pod with a token can call the API according to its RBAC. Overbroad permissions turn application compromise into cluster compromise.

Pod security context affects runtime blast radius. Running as non-root, dropping Linux capabilities, using read-only root filesystems where possible, preventing privilege escalation, and avoiding host namespaces reduce damage from container breakout or application RCE. Image provenance and scanning matter, but runtime permissions decide what a compromised process can do after deployment.

Admission policies, resource quotas, limit ranges, network policies, and image policies are platform guardrails. They prevent entire classes of incidents: Pods without requests, privileged containers, mutable latest tags, missing probes, forbidden registries, or uncontrolled egress. Guardrails should fail bad deployments before production, not rely on after-the-fact review.

## Observability and Debugging

Logs explain local events, metrics quantify behavior over time, and traces connect request paths across services. Infrastructure metrics alone cannot tell whether checkout is failing, Kafka consumers are stale, or p99 latency violates user SLOs. Application metrics must expose request rate, error rate, latency histograms, dependency latency, pool saturation, queue/lag age, business failures, and JVM internals.

Averages hide incidents. p95 and p99 show tail behavior caused by CPU throttling, GC pauses, noisy neighbors, retries, DNS delays, cold connections, lock contention, and overloaded dependencies. Kubernetes may show Pods healthy while a small percentage of requests times out. Tail latency plus correlation IDs and trace propagation is how backend incidents become debuggable instead of anecdotal.

Distributed debugging should follow the request path and the reconciliation path separately. For request path: ingress/load balancer, Service, endpoints, Pod readiness, container port, application logs, dependency calls, network policy, DNS, and downstream saturation. For reconciliation path: desired spec, events, controller state, ReplicaSet, scheduler decision, node conditions, kubelet logs, image pull, volume mount, probe results, and container exit reason.

Symptoms mislead because Kubernetes reports multiple layers. CrashLoopBackOff can be a bad config, missing secret, failed migration, JVM OOM during startup, permission issue, or dependency exit. Service unreachable can be DNS, selector, readiness, kube-proxy, CNI, NetworkPolicy, target port mismatch, app bind address, ingress rule, or client TLS. Pending can be capacity, taints, affinity, quota, PVC binding, or image policy. Good debugging narrows by layer instead of restarting Pods blindly.

## Incident Patterns

A Deployment rolls successfully because TCP readiness passes, but real requests fail due to lazy database initialization. Fix readiness to validate minimum serving capability and add deployment-level smoke tests.

A Java service is OOMKilled despite `-Xmx` below the limit because thread stacks, direct buffers, metaspace, and native memory exceed the remaining headroom. Fix total JVM memory budgeting, reduce threads, and monitor RSS versus limit.

HPA scales from 5 to 50 Pods during latency spike, exhausting the database connection limit and making the outage worse. Fix connection budgets, max replicas, pool sizing, and scale signals that understand downstream saturation.

A rollout with incompatible schema changes breaks old Pods during rolling update. Fix with expand/contract migrations and compatibility windows.

A Pod remains Ready during SIGTERM and receives traffic while shutting down. Fix readiness fail-fast on shutdown, graceful HTTP drain, sufficient termination grace, and load-balancer drain settings.

A Service has no endpoints because labels changed during refactor. Fix selector discipline, tests for rendered manifests, and endpoint checks in deployment validation.

A StatefulSet Pod cannot start after node failure because its zonal volume is stuck attached. Fix disruption planning, storage topology awareness, detach runbooks, and backup/restore validation.

A Kafka consumer Deployment scales to 30 replicas for a 12-partition topic and throughput does not improve. Fix partition strategy, handler performance, downstream bottlenecks, or consumer group design.

## Kubernetes Interview Q&A

### 162. What is container orchestration?

Container orchestration is a system that continuously manages and reconciles the desired state of containerized workloads across a cluster by handling scheduling, scaling, networking, rollouts, and self-healing automatically.

### 163. What production capabilities does Kubernetes provide?

Kubernetes provides production capabilities for running containerized workloads at scale, including desired-state reconciliation, automated scheduling, service discovery, rolling updates, self-healing, autoscaling, configuration and secret management, resource isolation, and workload rescheduling after failures.

However, Kubernetes does not automatically guarantee application correctness, database consistency, zero downtime, or proper production configuration.

### 164. What is a Pod in Kubernetes?

A Pod is the smallest scheduled workload unit: one or more containers sharing network namespace, lifecycle, and optional volumes. Production reasoning should treat it as replaceable. Pod IPs and local filesystems disappear; readiness controls traffic; kubelet controls restarts; limits control OOM/throttling; deletion triggers SIGTERM and graceful shutdown timing.

### 165. What is a Node? What is the difference between control-plane and worker nodes?

A Node is a machine in a Kubernetes cluster that provides CPU, memory, network, and storage resources for running cluster components and workloads.

Control-plane nodes manage the cluster itself: API Server, Scheduler, Controller Manager, and etcd.

Worker nodes run application workloads: kubelet, container runtime, networking components, and application Pods.

The difference:

Control-plane nodes make decisions and manage cluster state.
Worker nodes execute containers and handle application traffic.

### 166. What is a Service and how does Service discovery work?

A Service is a stable network endpoint in Kubernetes that provides access to a dynamic set of Pods.

Service discovery works through internal cluster DNS:

Pods access other services using the Service name.
DNS resolves the name to the Service virtual IP (ClusterIP).
Kubernetes routes traffic from the Service to one of the matching healthy Pods.

The main purpose of a Service is to provide stable networking and load balancing for Pods, because Pod IPs are ephemeral and can change.

### 167. What are Pod controllers? Deployment vs StatefulSet?

Pod controllers are Kubernetes objects that manage Pods and keep the desired number and state of Pods running.

Deployment is used for stateless applications with replaceable Pods and supports scaling and rolling updates.

StatefulSet is used for stateful applications that require stable Pod identity, persistent storage, and ordered startup/shutdown.

The main difference:

Deployment Pods are interchangeable.
StatefulSet Pods have stable identity and storage association.

### 168. What are PersistentVolume and PersistentVolumeClaim?

A PersistentVolume (PV) is storage resource available in the Kubernetes cluster.

A PersistentVolumeClaim (PVC) is a request for storage made by an application or Pod.

Kubernetes binds a PVC to a matching PV, and Pods use the claim to access persistent storage.

### 169. What is StorageClass?

A StorageClass defines how Kubernetes dynamically creates persistent storage.

It specifies storage parameters such as:

storage provider
disk type
performance tier
reclaim policy
volume binding behavior

When a Pod creates a PVC, Kubernetes uses the StorageClass to provision the required volume automatically.

### 170. What are ConfigMap and Secret?

ConfigMap stores non-sensitive application configuration such as URLs, ports, or feature flags.

Secret stores sensitive data such as passwords, API keys, or tokens.

Both can be injected into Pods as environment variables or mounted files. The main difference is that Secrets are intended for confidential data and support additional security controls.

### 171. What is Horizontal Pod Autoscaler?

Horizontal Pod Autoscaler (HPA) automatically scales the number of Pod replicas based on metrics such as CPU, memory, or custom application metrics.

When resource usage increases, HPA creates more Pods. When usage decreases, it removes excess replicas.

### 172. What are probes? What happens when liveness or readiness fails?

Probes are Kubernetes health checks used to determine container state and availability.

Readiness probe checks whether the container is ready to receive traffic.
Liveness probe checks whether the container is still alive and functioning correctly.
Startup probe checks whether the application has finished starting.

If readiness fails, the Pod is removed from Service traffic.

If liveness fails, Kubernetes restarts the container.

### 173. What is a Helm chart?

A Helm chart is a package of templated Kubernetes manifests used to install, configure, and manage applications in Kubernetes.

It allows defining reusable and configurable deployments using templates and values files.

### 174. What deployment strategies matter in Kubernetes?

The main Kubernetes deployment strategies are:

Rolling Update — gradually replaces old Pods with new ones without full downtime.
Blue/Green — runs two separate environments and switches traffic from the old version to the new one.
Canary — sends a small percentage of traffic to the new version first, then gradually increases traffic if the version is stable.

Rolling Update is the default strategy in Kubernetes. Blue/Green provides safer rollback. Canary is used for controlled production testing and risk reduction.

### 175. What is CD? Delivery vs Deployment?

Continuous Delivery is a practice where software is automatically built, tested, and prepared for release, but production deployment usually requires manual approval.

Continuous Deployment automatically deploys every successful change directly to production without manual intervention.

The difference:

* Continuous Delivery → production release is manual.
* Continuous Deployment → production release is automatic.


### How would you monitor a Java service running in Kubernetes?

Monitor Kubernetes state and application behavior together: Pod restarts, readiness, OOMKilled, CPU throttling, memory RSS versus limit, node pressure, request rate, error rate, p95/p99 latency, dependency latency, JDBC pool saturation, Kafka/RabbitMQ lag, JVM heap/non-heap, GC pauses, thread count, and business failures. Alerts should map to user impact and recovery actions.

### What is observability compared with monitoring?

Monitoring tells whether known signals crossed thresholds. Observability lets engineers explain unknown failures from logs, metrics, traces, events, and runtime context. In Kubernetes, observability must correlate Pod identity, version, node, request trace, dependency calls, resource throttling, GC, rollout, and business operation ID.

### What is Prometheus used for?

Prometheus scrapes time-series metrics from Kubernetes components, exporters, and applications. It is used for alerting, dashboards, SLO calculations, capacity trends, and incident queries. Metric design matters: histograms for latency, bounded labels, useful dimensions such as namespace/workload/pod/version, and application-level signals beyond infrastructure health.

### What is Grafana used for?

Grafana visualizes metrics and traces from sources such as Prometheus, Loki, Tempo, and cloud monitoring systems. A useful dashboard shows request volume, errors, latency percentiles, saturation, dependency health, rollout version, Pod restarts, throttling, memory, and backlog. Dashboards that only show CPU and memory are insufficient for backend operations.

### How should logging work in Kubernetes?

Applications write structured logs to stdout/stderr; node agents ship them to centralized storage. Logs need timestamp, level, service, version, Pod, namespace, trace ID, correlation ID, user/business operation ID where safe, and error cause. Local Pod logs are temporary; after restart or reschedule, centralized logs are the incident record.

### How do you troubleshoot CrashLoopBackOff?

Start with `kubectl describe pod` for Events and last state, then `kubectl logs --previous` for the crashed container. Check exit code, OOMKilled, missing ConfigMap/Secret, image/runtime errors, failed migrations, port conflicts, filesystem permissions, dependency startup assumptions, probe configuration, and JVM memory. Fix the cause; deleting the Pod only resets evidence and backoff timing.

### How do you troubleshoot a Service that is not reachable?

Check DNS resolution, Service selector, EndpointSlices, Pod readiness, targetPort/containerPort alignment, application bind address, kube-proxy/CNI health, NetworkPolicy, ingress/controller rules, TLS, and client namespace. Test from inside the cluster before testing ingress. A Service with no endpoints points to labels or readiness; endpoints with failed connections points to network, port, or application binding.

### What does OOMKilled mean for a Java service?

The container exceeded its memory limit and the kernel killed it. Investigate RSS, heap, non-heap, direct buffers, metaspace, thread stacks, GC logs, allocation spikes, and native memory. Fix by lowering `-Xmx` or JVM memory percentage, adding headroom, reducing threads/buffers, tuning GC, increasing limit/request if justified, and alerting before RSS approaches the limit.

### Infrastructure monitoring vs application monitoring?

Infrastructure monitoring reports cluster and runtime health: nodes, Pods, CPU, memory, restarts, network, disk, control plane, and Services. Application monitoring reports business and service behavior: request latency, errors, throughput, dependency saturation, queue lag, failed payments, stale projections, and JVM internals. Production incidents require both because healthy Pods can serve broken business logic, and broken nodes can masquerade as application latency.
