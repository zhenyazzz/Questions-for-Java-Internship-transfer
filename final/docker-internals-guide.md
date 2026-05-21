# Docker Under the Hood: Filesystems, Layers, and Runtime Reality

> **Goal:** understand Docker without “magic” by tracing concrete Linux primitives: filesystem layers, OverlayFS behavior, and process isolation.

---

## 1. What a Container Actually Is

A container is not a virtual machine. A VM starts a separate guest kernel; a container runs as a regular Linux process on the host kernel. The difference is isolation and control boundaries applied by the kernel.

In practice, three mechanisms define container behavior:

- **Namespaces** isolate what the process can see (PID tree, mounts, hostname, IPC, and optionally user mappings).
- **cgroups** limit and account what the process can consume (CPU, memory, PIDs, I/O).
- **OverlayFS** provides a unified root filesystem assembled from read-only image layers plus one writable container layer.

Useful analogy: an **image** is a blueprint package, while a **container** is a live construction instance created from that blueprint.

---

## 2. Physical Meaning of Docker Layers

A Docker layer is a real filesystem delta on disk. With Docker Engine on Linux using the `overlay2` storage driver, this data is typically located under `/var/lib/docker/overlay2/`.

Inside layer/container-related directories you commonly encounter:

| Entry | Meaning |
|---|---|
| `diff/` | Files introduced or changed by this layer |
| `lower` | Reference chain to parent lower layers |
| `merged/` | Unified mount view (present for active containers) |
| `work/` | OverlayFS work directory |
| `link`, `l/` | Indirection used to shorten lowerdir references |

So “layer” is not abstract metadata. It maps to host directories and objects managed by the storage driver.

---

## 3. Which Dockerfile Instructions Create Filesystem Deltas

Only instructions that change filesystem state create new layer deltas:

- `RUN`
- `COPY`
- `ADD`

By contrast, instructions like `ENV`, `WORKDIR`, `CMD`, `ENTRYPOINT`, `EXPOSE`, `LABEL`, and `USER` primarily update image configuration metadata fields (`config.Env`, `config.WorkingDir`, `config.Cmd`, and others).

Important nuance: these metadata steps still appear in image history, but they normally do not add a new filesystem `diff/` payload.

---

## 4. Why Layer Immutability Matters

Layers are content-addressed by digest (`sha256:...`). If layer bytes change, the digest changes, so that result becomes a different layer instead of an in-place edit.

This property enables deterministic caching, safe deduplication, and reproducible pulls/builds. It also allows broad reuse on one host. For example, if many services use the same Java base image, that base is stored once, while each service adds only its own small delta layer set.

```text
Shared immutable base: L0 (OS) + L1 (JRE) + L2 (certs)
Image A = L0+L1+L2+LA
Image B = L0+L1+L2+LB
Container A1 = (L0..LA) + UpperA1
Container B1 = (L0..LB) + UpperB1
```

That is where major disk savings come from: shared read-only lower layers plus small per-container writable layers.

---

## 5. OverlayFS Internals in Operational Terms

OverlayFS merges layer directories into one mount:

```bash
mount -t overlay overlay   -o lowerdir=<lowerN>:...:<lower1>,upperdir=<upper>,workdir=<work>   <merged>
```

Concepts:

- **LowerDir**: immutable image layer stack.
- **UpperDir**: writable delta for one container.
- **WorkDir**: internal scratch/work area required by overlayfs.
- **MergedDir**: unified view that becomes the process root filesystem.

### Read behavior

The kernel checks `UpperDir` first. If path is absent there, it resolves through the `LowerDir` stack from top to base. First match is returned.

### Write behavior (Copy-on-Write / copy-up)

If a file exists only in lower layers and a process writes to it, the kernel copies the file into `UpperDir`, then applies writes to that upper copy. Lower data remains unchanged.

### Delete behavior (whiteout)

A container cannot physically delete from lower read-only layers. Deletion is recorded in `UpperDir` via whiteout semantics (commonly represented as `.wh.<name>` in layer form), which hides the lower object from merged view.

So “deleted in container” often means “hidden by upper-layer metadata,” not “erased from immutable lower storage.”

---

## 6. Manifest vs Image vs Container

These terms refer to different technical objects.

A **manifest** is a registry-side JSON descriptor containing pointers to the config object and ordered layer blobs (digests, sizes, media types).

An **image** is the local resolved artifact: config JSON + ordered immutable layer set + local references.

A **container** is runtime process state: namespaces, cgroups, merged rootfs mount, and one writable upper layer.

Short form:

- Image = immutable blueprint.
- Container = live execution instance.

---

## 7. Host-Level Observability Commands

To verify these mechanics on a Linux host:

```bash
docker image inspect <image>
docker inspect <container>
mount | grep overlay
cat /proc/<pid>/mountinfo
```

Look specifically at `GraphDriver` data (`LowerDir`, `UpperDir`, `MergedDir`, `WorkDir`) and actual overlay mount options.

---

## 8. Quick Misconception Corrections

| Misconception | Accurate statement |
|---|---|
| “Container = lightweight VM” | Container = isolated process sharing host kernel |
| “Every container stores full OS copy” | Containers share lower layers; only upper layer is private/writable |
| “Deleting file in container deletes it from image” | Deletion is represented by whiteout/hiding in upper layer |

---

Docker becomes predictable once each term is mapped to concrete Linux objects: directories, hashes, mounts, and processes.
