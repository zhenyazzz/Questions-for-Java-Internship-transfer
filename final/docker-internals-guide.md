# Docker Under the Hood: A Filesystem-and-Kernel-Level Guide

> Goal: understand Docker **without magic**, through Linux filesystem mechanics, metadata formats, and process isolation.

---

## 0) Mental Model in One Minute

Docker is not a mini-VM. In practice:

1. **Image** = immutable stack of filesystem deltas + JSON metadata.
2. **Container** = a normal Linux process started with:
   - isolated namespaces (pid/net/mnt/uts/ipc/user)
   - cgroups limits/accounting
   - root filesystem assembled from image layers + writable layer
3. **OverlayFS** in the Linux kernel presents those layers as one merged tree.

So the “magic” is mostly:

- content-addressable files on disk
- OverlayFS mount options
- OCI JSON descriptors
- ordinary process creation (`clone`/`execve`) with namespace/cgroup setup

---

## 1) Physical Nature of Docker Layers

## 1.1 What a layer really is

A Docker layer is a **real directory delta on disk** (plus metadata), not an abstract concept.

For Docker Engine with `overlay2` driver, data is typically under:

- **`/var/lib/docker/overlay2/`**

Each layer has content like:

- `diff/` — files introduced/changed by this layer delta
- `link` and internal references for efficient mount argument length handling
- `lower` (for non-base layers) — parent layer chain reference
- `merged/` — runtime merged view (usually for active containers)
- `work/` — OverlayFS workdir for kernel operations

> Think: each layer is like a Git commit snapshot delta, but represented in filesystem structures optimized for union mounts.

## 1.2 Which Dockerfile instructions create filesystem layers

Instructions that modify filesystem contents create new filesystem layers:

- **`RUN`** — executes commands; resulting file changes become a new layer delta.
- **`COPY`** — copied files are added as layer changes.
- **`ADD`** — similar to `COPY`, plus archive auto-extract / URL fetch behavior.

### Example

```Dockerfile
FROM eclipse-temurin:21-jre
RUN apt-get update && apt-get install -y curl
COPY app.jar /app/app.jar
RUN useradd -r appuser
```

Possible layer chain (simplified):

1. Base image layers from `eclipse-temurin:21-jre`
2. Layer A: apt index + installed packages
3. Layer B: `/app/app.jar`
4. Layer C: `/etc/passwd` update from `useradd`

## 1.3 Why `ENV`, `WORKDIR`, `CMD` usually do not create filesystem layers

These instructions primarily alter **image configuration metadata** (JSON config), not file content:

- `ENV` -> `config.Env`
- `WORKDIR` -> `config.WorkingDir`
- `CMD` -> `config.Cmd`
- `ENTRYPOINT` -> `config.Entrypoint`
- `EXPOSE`, `LABEL`, `USER`, `STOPSIGNAL` -> config fields

So they usually create a **new image history entry** and new config object, but no additional `diff/` filesystem delta if no file changes happen.

> Important nuance: tooling may still record history items; “no layer” here means no new filesystem delta tar/diff layer.

---

## 2) Layer Immutability and Reuse Economics

## 2.1 What immutability means here

After a layer is created and content-hashed, it is **immutable**:

- bytes define digest (`sha256:...`)
- if bytes change, digest changes, so it is a different layer

This gives deterministic reuse and safe deduplication.

## 2.2 Why immutability is required

If layer bytes could mutate in place:

- caches become invalid unpredictably
- signatures/trust chains break
- multiple images pointing to “same” digest could silently diverge

With immutability:

- digest uniquely identifies exact content forever
- build cache lookup is reliable
- registries and hosts deduplicate storage safely

## 2.3 One Java base layer reused by 100 images

Suppose 100 services all use `eclipse-temurin:21-jre` base.

Disk-wise on one host:

- base layers are stored once (content-addressed)
- each service image stores only its extra deltas
- each running container adds only a small writable upper layer

ASCII view:

```text
Shared immutable store
┌──────────────────────────────────────────────┐
│ Layer L0: debian base                        │  <- stored once
│ Layer L1: JVM files (/opt/java/...)          │  <- stored once
│ Layer L2: certs/timezone updates             │  <- stored once
└──────────────────────────────────────────────┘

Service A image = [L0,L1,L2,LA]
Service B image = [L0,L1,L2,LB]
...
Service Z image = [L0,L1,L2,LZ]

100 containers:
  each -> read-only lower stack [L0,L1,L2,Lx] + own writable UpperDir
```

That is exactly where gigabytes are saved.

---

## 3) Linux Kernel Mechanics: OverlayFS in Detail

## 3.1 OverlayFS mount anatomy

Docker uses OverlayFS (`overlay2`) to build unified rootfs.

Conceptual mount:

```bash
mount -t overlay overlay \
  -o lowerdir=<lowerN>:...:<lower1>,upperdir=<upper>,workdir=<work> \
  <merged>
```

Key dirs:

- **LowerDir**: read-only image layers (possibly many)
- **UpperDir**: container writable layer (one per container)
- **WorkDir**: OverlayFS internal scratch/work area (same fs as upper)
- **MergedDir**: the resulting unified tree seen by the container process as `/`

In Docker paths this commonly maps into subdirs under:

- `/var/lib/docker/overlay2/<id>/diff` (upper or layer diff)
- `/var/lib/docker/overlay2/<id>/work`
- `/var/lib/docker/overlay2/<id>/merged`

## 3.2 How read path resolution works

When process opens `/etc/ssl/certs/ca-certificates.crt` in merged view:

1. OverlayFS checks UpperDir first.
2. If missing, scans lower stack from topmost lower to base.
3. Returns first match.

So upper overrides lower, giving expected “latest change wins.”

## 3.3 Copy-on-Write (CoW) on file modification

If a file exists only in lower layers and process tries to modify it:

1. Kernel copies file (and needed metadata) from lower to UpperDir (“copy up”).
2. Write happens against copied upper file.
3. Lower layer remains untouched (immutable).

Effects:

- first write to a large lower file can be expensive (copy-up cost)
- subsequent writes are cheap (already upper-owned)

Analogy: laminated blueprint pages (lower) cannot be edited; you photocopy page to your notebook (upper) and edit copy.

## 3.4 Whiteout on deletion

Container cannot delete from read-only lower directly. So deletion is represented in upper as a **whiteout marker**.

Mechanism (conceptually):

- create special marker in UpperDir for deleted path
- during merged lookup, marker hides same-named object in lower layers

Result: file appears deleted in container view, though bytes still exist in immutable lower storage.

For directory replacement/opacity semantics, OverlayFS uses opaque directory markers (e.g., xattrs) to hide lower directory contents.

ASCII example:

```text
Lower: /app/config.yaml
Upper: /app/.wh.config.yaml   (whiteout marker)
Merged view: /app/config.yaml  -> NOT VISIBLE
```

---

## 4) Manifest vs Image vs Container

## 4.1 OCI/Docker distribution objects (registry side)

At registry/protocol level:

- **Manifest** (JSON): points to config object + list of layer blobs (digests/sizes/mediaTypes)
- **Config JSON**: runtime/build metadata (env, cmd, entrypoint, user, history, rootfs diff_ids)
- **Layer blobs**: compressed tar archives with filesystem changes

So manifest is like an index card saying “image = this config + these layer digests.”

## 4.2 What “Image” means locally

Locally, “image” usually means the resolved tuple:

- config JSON
- ordered layer set
- tag/digest reference in image store

Analogy:

- **Blueprint package** (image) = instructions + material list
- **House** (container) = a built, running instance from blueprint

## 4.3 What “Container” physically is

Container is not a special file format; it is a **running (or stopped) Linux process state** with metadata:

- process created with namespaces/cgroups/seccomp/capabilities constraints
- rootfs mounted as OverlayFS merged dir
- writable UpperDir attached
- runtime state files/logs/network namespace endpoints

In short:

- Image = immutable recipe + read-only content layers
- Container = process execution context + one writable layer

---

## 5) Data Path Walkthrough: `docker run myapp:1.0`

1. Docker resolves `myapp:1.0` -> image manifest/config/layers.
2. Missing layers are pulled by digest.
3. Engine creates container-specific UpperDir + WorkDir.
4. Kernel overlay mount constructed into MergedDir.
5. OCI runtime (e.g., `runc`) sets namespaces/cgroups.
6. Process starts with MergedDir as `/` (root filesystem).
7. Runtime writes go to UpperDir (CoW/whiteout semantics).

---

## 6) Practical Observability Commands

Useful commands for verification on Linux host:

```bash
docker image inspect <image>
docker inspect <container>
mount | grep overlay
cat /proc/<container-pid>/mountinfo
sudo ls /var/lib/docker/overlay2/
```

What to check:

- `GraphDriver` fields (`LowerDir`, `UpperDir`, `MergedDir`, `WorkDir`)
- exact overlay mount options in `mountinfo`
- how deletions produce whiteout entries in upper layer

---

## 7) Common Misconceptions (and precise correction)

- “Container = lightweight VM.”
  - More precise: container = isolated Linux process group sharing host kernel.
- “Each container stores full filesystem copy.”
  - More precise: shared immutable lowers + tiny writable upper.
- “Deleting file in container shrinks image.”
  - More precise: deletion in running container creates whiteout in upper; base layers unchanged.

---

## 8) Quick Recap Table

| Term | Physical reality | Mutable? | Shared? |
|---|---|---:|---:|
| Layer | Filesystem delta blob/dir + hash | No | Yes |
| Image | Metadata (config JSON) + ordered layer references | Effectively no (new image on change) | Yes |
| Container UpperDir | Writable delta for one container | Yes | No |
| Container process | Linux process with namespaces/cgroups | Runtime state changes | N/A |
| MergedDir | OverlayFS unified mount view | Virtual view | Per container mount |

---

## Final Analogy (Blueprint + Layer Cake)

- **Image** is blueprint + ingredient list.
- **Layers** are pre-baked cake sheets stored once in warehouse.
- **Container** is a served slice where waiter can add cream on top (UpperDir).
- Removing a cherry from a shared sheet is impossible directly, so waiter places a “hide cherry” note (whiteout) on your slice.

No magic: just hashes, dirs, mounts, and processes.
