# Docker Under the Hood: Filesystems, Kernel Mechanics, and Runtime Reality

> **Goal:** understand Docker without “magic” by tracing what happens in Linux filesystems, kernel namespaces/cgroups, and process runtime.

---

## 1) Mental Model: What a Container *Actually* Is

A container is **not** a virtual machine.

- A VM boots a separate guest kernel.
- A container is a **regular Linux process** running on the **host kernel**, but constrained by kernel primitives.

Three core building blocks:

1. **Namespaces** — isolate what a process can see (PID tree, mounts, network interfaces, hostname, IPC, optionally user IDs).
2. **cgroups** — control what a process can consume (CPU, memory, PIDs, I/O).
3. **Union filesystem (OverlayFS)** — provides a merged root filesystem from immutable image layers plus one writable container layer.

Analogy:

- **Image** = architectural blueprint + fixed material list.
- **Container** = construction crew actively building/running from that blueprint in a fenced zone.

---

## 2) Physical Nature of Docker Layers

## 2.1 Layer = concrete filesystem delta on disk

With Docker Engine on Linux using `overlay2`, layer data is typically in:

- **`/var/lib/docker/overlay2/`**

For layer/container directories, you’ll commonly see:

- `diff/` — the actual files in this delta.
- `lower` — references to parent lower layers.
- `merged/` — mounted unified view (for active containers).
- `work/` — OverlayFS work directory.
- `link` / `l/` indirection entries used by Docker to shorten long lowerdir paths.

So “layer” is not abstract — it maps to real host filesystem data.

## 2.2 Which Dockerfile instructions create filesystem layers

Filesystem-changing instructions create delta layers:

- **`RUN`**
- **`COPY`**
- **`ADD`**

Why:

- `RUN` changes files by executing commands.
- `COPY` adds/overwrites files from build context.
- `ADD` does what `COPY` does, plus extras (like local tar auto-extract and URL source support in classic Docker behavior).

## 2.3 Why `ENV`, `WORKDIR`, `CMD` are metadata (not fs deltas)

Instructions such as:

- `ENV`, `WORKDIR`, `CMD`, `ENTRYPOINT`, `EXPOSE`, `LABEL`, `USER`

primarily update **image config JSON** fields (`config.Env`, `config.WorkingDir`, `config.Cmd`, etc.), rather than writing files into rootfs.

Important nuance:

- Build history can still show these steps.
- But they normally do **not** produce a new filesystem `diff/` payload.

---

## 3) Immutability: Why Layers Never Change In Place

A layer is content-addressed by digest (`sha256:...`).

- If one byte changes, digest changes.
- Therefore changed content becomes a **new layer**, never an in-place mutation.

Why this matters:

- deterministic caching
- safe deduplication
- reliable integrity/signature chains
- repeatable pulls/builds across hosts

## 3.1 Reuse economics: one Java base shared by many images

Suppose 100 services use `eclipse-temurin:21-jre`.

On one host:

- base layers are stored once
- each service adds only its own small app-specific layers
- each running container adds one writable layer

```text
Shared immutable layers (stored once):
  L0 (OS), L1 (JRE), L2 (certs)

Image A = L0+L1+L2+LA
Image B = L0+L1+L2+LB
...
Image Z = L0+L1+L2+LZ

Container A1 = (L0..LA) + UpperA1
Container A2 = (L0..LA) + UpperA2
Container B1 = (L0..LB) + UpperB1
```

This is where multi-gigabyte savings come from.

---

## 4) OverlayFS Internals: LowerDir, UpperDir, MergedDir

OverlayFS mount conceptually looks like:

```bash
mount -t overlay overlay \
  -o lowerdir=<lowerN>:...:<lower1>,upperdir=<upper>,workdir=<work> \
  <merged>
```

Components:

- **LowerDir**: read-only image layer stack.
- **UpperDir**: writable container layer.
- **WorkDir**: internal work area required by overlayfs.
- **MergedDir**: unified virtual tree seen by process as `/`.

## 4.1 Read path

When reading `/path/file`:

1. check UpperDir first
2. if absent, walk LowerDir stack top-down
3. first match wins

## 4.2 Write path (Copy-on-Write / copy-up)

If file exists only in lower layer and process writes to it:

1. kernel copies object into UpperDir (copy-up)
2. write occurs on upper copy
3. lower object stays immutable

Implication: first write to large files may be expensive.

## 4.3 Delete path (whiteout)

Container cannot delete from read-only lower layer directly.

Instead OverlayFS records deletion in UpperDir with a **whiteout marker** (commonly `.wh.<name>` semantics in layer representation), hiding lower object from merged view.

So “deleted” in container view often means “hidden by upper metadata,” not physically erased from immutable lower bytes.

---

## 5) Manifest vs Image vs Container

## 5.1 Manifest (registry object)

An OCI/Docker **manifest JSON** references:

- config object digest
- ordered layer blob digests
- media types/sizes

Think of manifest as an index card of cryptographic pointers.

## 5.2 Image (local resolved artifact)

Locally, an **image** is effectively:

- config JSON metadata
- ordered immutable layer set
- local tags/digest references in image store

## 5.3 Container (runtime state)

A **container** is a running/stopped process context with:

- namespace boundaries
- cgroup constraints
- merged rootfs mount
- one writable upper layer
- runtime metadata (state, logs, network endpoints)

Short version:

- **Image** = immutable blueprint.
- **Container** = live execution instance.

---

## 6) Networking Under the Hood (Bridge + veth + NAT)

## 6.1 Bridge networking

Default Docker networking typically uses Linux bridge `docker0`.

For each container:

1. kernel creates a **veth pair**
2. one end stays on host bridge (`docker0`)
3. other end moves into container net namespace as `eth0`
4. container gets IP (often from `172.17.0.0/16`)

That is plain Linux networking primitives — no VM switch required.

## 6.2 Port publishing (`-p 8080:80`)

Docker programs host packet rules (historically iptables; on newer systems this may map through nftables backend).

Mechanism is DNAT/SNAT in host networking stack so host port traffic is translated/routed into container IP:port.

So external clients hit host:8080, kernel rewrites/routes to container:80.

---

## 7) Observability: Verify on a Real Host

Useful commands:

```bash
docker image inspect <image>
docker inspect <container>
mount | grep overlay
cat /proc/<pid>/mountinfo
ip link show
ip netns list   # if namespaces are exposed via iproute2 helpers
iptables -t nat -S || true
nft list ruleset || true
```

Look for:

- `GraphDriver` paths (`LowerDir`, `UpperDir`, `MergedDir`, `WorkDir`)
- overlay mount options
- veth endpoints and bridge attachments
- NAT rules for published ports

---

## 8) Fast Misconception Corrections

- “Container is a lightweight VM.”
  - More accurate: isolated process sharing host kernel.
- “Each container has full OS copy.”
  - More accurate: shared lower layers + private writable upper.
- “Deleting file in container removes it from image.”
  - More accurate: deletion is represented by whiteout/hiding in upper layer.

---

## 9) Recap Table

| Object | Physical reality | Mutable? | Shared? |
|---|---|---:|---:|
| Layer | Content-addressed filesystem delta | No | Yes |
| Image | Config JSON + ordered layer references | Rebuilt on change | Yes |
| Container upper layer | Writable overlay delta | Yes | No |
| Container | Linux process + namespace/cgroup state | Runtime mutable | N/A |
| Merged rootfs | OverlayFS virtual mount view | Virtual | Per container mount |

---

## Final Analogy: Layer Cake + Tracing Paper

- Immutable layers are pre-baked cake sheets in cold storage.
- Container upper layer is your personal frosting layer.
- If you “remove” a cherry from shared sheet, you cannot erase the original; you place a “hide cherry” note (whiteout) on your own tracing paper.

Docker feels magical only until you map each concept to a concrete Linux primitive: **directories, hashes, mounts, network devices, and processes**.
