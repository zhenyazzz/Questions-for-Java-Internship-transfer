# Docker Under the Hood: Interview-Ready Deep Explanation

> **Purpose of this material:** give you a narrative you can confidently explain at an interview, from disk structures to kernel behavior, without vague phrases or “it just works”.

---

## 1) The Core Idea You Should Say First at an Interview

If an interviewer asks, “What is Docker under the hood?”, the strongest concise opening is:

**Docker is a packaging and runtime system for Linux processes.**

A container is not a separate machine and not a separate kernel. It is a normal process started with controlled isolation boundaries and a specially assembled root filesystem.

In practical terms, three Linux mechanisms are central:

- **Namespaces** define what the process can *see*.
- **cgroups** define what the process can *consume*.
- **OverlayFS** defines what the process can *read/write* in its root filesystem.

A good analogy for interviews:

- **Image** = architectural blueprint + immutable material list.
- **Container** = one active construction site created from that blueprint.

You can repeat this model multiple times during the conversation because it is both technically accurate and easy to remember.

---

## 2) Docker Layers: Physical Reality on Disk, Not an Abstraction

A Docker layer corresponds to concrete filesystem data.

On Linux hosts using Docker’s `overlay2` storage driver, layer-related data is typically stored under:

- **`/var/lib/docker/overlay2/`**

Inside that storage area, Docker keeps directories and metadata that represent filesystem deltas and parent relationships. Conceptually, each layer is a “difference package” relative to its parent. That difference package contains only what changed at that step.

When you describe layer anatomy, explain it as follows:

- `diff/` contains changed or added files for that layer delta.
- `lower` points to parent lower layers.
- `merged/` is the unified view used by a running container.
- `work/` is required by OverlayFS internals.

The key interview message:

**A layer is not just a line in Docker history; it is represented by real on-disk objects.**

---

## 3) Which Dockerfile Instructions Create Real Filesystem Layers

Dockerfile instructions split into filesystem-changing and metadata-changing categories.

### Filesystem-changing instructions

The instructions that normally create a new filesystem delta are:

- `RUN`
- `COPY`
- `ADD`

Why? Because they actually alter files in the image rootfs state.

### Metadata-oriented instructions

Instructions such as `ENV`, `WORKDIR`, `CMD`, `ENTRYPOINT`, `EXPOSE`, `LABEL`, `USER` mostly update configuration fields in the image config JSON.

These instructions are still part of image history, but they typically do not add a new `diff/` payload containing filesystem content changes.

How to phrase this in an interview:

> “Some Dockerfile instructions modify bytes on disk, others modify only image configuration metadata. Both appear in history, but only filesystem-mutating steps produce layer file deltas.”

That sentence usually signals deep understanding.

---

## 4) Immutability: Why Docker Layers Are Frozen and Reusable

The next core concept is **immutability**.

A layer is content-addressed by digest (for example `sha256:...`). The digest is derived from content. If content changes, the digest changes. Therefore, changed content cannot remain “the same layer”; it necessarily becomes a new layer.

Why this matters in production:

1. **Deterministic caching:** build systems can safely reuse previous layers.
2. **Deduplication:** identical layers are stored once even if referenced by many images.
3. **Integrity:** digest references make tampering detectable.
4. **Portability:** different hosts can fetch by digest and get exact same content.

Practical storytelling example:

If 100 services share the same Java base image, the Java-related base layers are physically stored once on the host. Each service contributes only its own app-specific layers, and each running container gets a small writable top layer.

So instead of 100 full copies of a giant base filesystem, Docker reuses shared immutable layers and adds only small per-image/per-container differences.

---

## 5) OverlayFS: How Linux “Glues” Layers at Runtime

This is the mechanical heart of Docker storage behavior.

OverlayFS combines multiple directories into a single unified mount view. For container runtime semantics, the most important terms are:

- **LowerDir**: read-only stack of image layers.
- **UpperDir**: writable layer unique to the container.
- **WorkDir**: internal scratch area for overlay operations.
- **MergedDir**: the final unified tree seen by the container process as `/`.

Interview-friendly explanation:

> “The container process does not manually traverse many layers. The kernel overlay module presents one merged filesystem. The process sees a normal rootfs, while OverlayFS resolves where each path actually comes from.”

### Read behavior

When a process reads a file path, OverlayFS checks UpperDir first. If not found there, it looks through lower layers from topmost to base. First match wins.

### Write behavior (Copy-on-Write / copy-up)

If a file exists only in a lower (read-only) layer and the process tries to modify it, OverlayFS performs a copy-up into UpperDir and then applies the write there.

This gives two important properties:

- lower layers remain immutable and shareable;
- container writes remain isolated to that container.

### Delete behavior (whiteout)

A container cannot physically remove files from immutable lower layers. Deletion is expressed in UpperDir through whiteout semantics that hide lower objects in the merged view.

So a deleted file inside a container is often “masked from view,” not erased from the underlying immutable base data.

This detail is very interview-relevant because it explains why deleting files in a running container does not rewrite base image layers.

---

## 6) Manifest vs Image vs Container (Precise Definitions)

These terms are often mixed up; interviewers intentionally test this.

### Manifest

A **manifest** is a registry-side JSON descriptor that points to:

- config object digest,
- ordered list of layer blob digests,
- related media type and size metadata.

Think of it as a cryptographic index card.

### Image

An **image** is the local resolved artifact composed of config metadata plus ordered immutable layers referenced by digest/tag.

In short, image is a blueprint package, not a running entity.

### Container

A **container** is runtime state: a process (or process group) launched with namespace/cgroup constraints and mounted merged rootfs plus one writable layer.

Short interview line:

- **Image** = immutable blueprint.
- **Container** = active instance of that blueprint.

---

## 7) 90-Second Interview Answer

You can present it like this:

1. Docker image is immutable metadata + filesystem layers.
2. Layers are real on-disk deltas (commonly under `/var/lib/docker/overlay2/`).
3. `RUN/COPY/ADD` usually create filesystem layer deltas; many other Dockerfile instructions modify config metadata only.
4. Container startup uses Linux isolation primitives (namespaces/cgroups) and OverlayFS to create one merged root filesystem.
5. Runtime writes go to container UpperDir via copy-on-write; deletes are represented by whiteouts.
6. Because layers are immutable and content-addressed, many images/containers can share the same base layers efficiently.

If you can explain these six points cleanly, this demonstrates strong Docker fundamentals.

---

## 8) Final Takeaway

Docker becomes much less mysterious when you map every concept to concrete Linux objects: **directories, hashes, mount behavior, and process isolation**.

That is exactly the level expected in serious backend, platform, and DevOps interviews.
