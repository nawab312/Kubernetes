# Kubernetes Interview Mastery
# CATEGORY 13: DOCKER & CONTAINER FUNDAMENTALS

---

> **How to use this document:**
> Each topic: Simple Explanation → Why It Exists → Internal Working → Commands/Examples → Short Answer → Gotchas.
> No Docker Compose covered — pure container and image fundamentals.
> ⚠️ = High priority, frequently asked in interviews.

---

## Table of Contents

| # | Topic | Level |
|---|-------|-------|
| 13.1 | What is a container — vs VMs, vs bare processes | 🟢 Beginner |
| 13.2 | Docker architecture — daemon, client, containerd | 🟢 Beginner |
| 13.3 | Images — layers, union filesystem (OverlayFS), OCI spec ⚠️ | 🟢 Beginner |
| 13.4 | Dockerfile — instructions, layer caching, best practices ⚠️ | 🟢 Beginner |
| 13.5 | Container lifecycle — create, start, stop, kill, exit codes | 🟢 Beginner |
| 13.6 | Volumes & bind mounts — named volumes, tmpfs | 🟢 Beginner |
| 13.7 | Docker networking — bridge, host, none, user-defined bridge | 🟡 Intermediate |
| 13.8 | Image registries — push, pull, tags vs digests, ECR auth | 🟡 Intermediate |
| 13.9 | Linux namespaces — pid, net, mnt, uts, ipc, user ⚠️ | 🟡 Intermediate |
| 13.10 | cgroups v1 vs v2 — CPU, memory, blkio enforcement ⚠️ | 🟡 Intermediate |
| 13.11 | OCI spec — image spec, runtime spec, CRI relationship | 🟡 Intermediate |
| 13.12 | containerd & runc — the actual container start path | 🟡 Intermediate |
| 13.13 | Multi-stage builds — smaller, more secure images ⚠️ | 🟡 Intermediate |
| 13.14 | Image security — vulnerability scanning, rootless, distroless | 🔴 Advanced |
| 13.15 | BuildKit — parallel builds, cache mounts, secrets in build | 🔴 Advanced |
| 13.16 | Container runtime security — capabilities, seccomp, user namespaces | 🔴 Advanced |

---

# 13.1 What Is a Container — vs VMs, vs Bare Processes

## 🟢 Beginner

### What it is in simple terms

A container is a **process (or group of processes) running on the host OS kernel, isolated from other processes using Linux kernel features**. There is no guest OS, no hypervisor, and no hardware emulation. The host kernel executes everything. Containers are lightweight precisely because they skip the entire OS boot stack — they're just a process with walls around it.

---

### The Isolation Stack

```
WHAT ACTUALLY RUNS WHEN YOU START A CONTAINER
═══════════════════════════════════════════════════════════════

BARE PROCESS (no isolation):
  ┌──────────────────────────────────────┐
  │  Your App Process (PID 2847)         │
  │  Sees: ALL processes on host         │
  │  Sees: ALL files on host filesystem  │
  │  Sees: ALL network interfaces        │
  │  Limits: none (can use all RAM/CPU)  │
  └──────────────────────────────────────┘
  Host kernel

CONTAINER (process + kernel isolation):
  ┌──────────────────────────────────────┐
  │  Your App Process (PID 1 inside)     │
  │  Sees: only its own processes        │  ← PID namespace
  │  Sees: only its own filesystem       │  ← Mount namespace + image
  │  Sees: only its own network (eth0)   │  ← Network namespace
  │  Limits: CPU/memory enforced         │  ← cgroups
  └──────────────────────────────────────┘
  Host kernel (SHARED — same kernel running everything)

VM (full hardware virtualization):
  ┌──────────────────────────────────────┐
  │  Your App Process                    │
  │  Guest OS (Linux/Windows kernel)     │
  │  Virtual hardware (emulated CPU/RAM) │
  └──────────────────────────────────────┘
  Hypervisor (KVM, Xen, VMware)
  Host kernel
  Physical hardware

KEY INSIGHT:
  Container = process with Linux namespace + cgroup isolation
  VM        = full OS + virtual hardware on a hypervisor
  Container shares the host kernel. Always. No exceptions.
  "Ubuntu container on Mac" = Linux VM running containers,
  the container process runs in the Linux VM's kernel, not macOS.
```

---

### Comparison Table

```
DIMENSION          BARE PROCESS    CONTAINER       VM
═══════════════════════════════════════════════════════════════
Startup time       < 1ms           50-500ms        10-60 seconds
Memory overhead    ~0              ~5-50MB          512MB - 2GB
Disk overhead      0               50MB - 2GB       5GB - 50GB
Process isolation  none            namespace-based  full (separate kernel)
Filesystem         host fs         image layers     virtual disk
Network            host            private veth     virtual NIC
Resource limits    none            cgroups          hypervisor
Kernel             shared          shared           separate
Security boundary  none            namespace+cgroup hypervisor
Density per host   100s            100s             10-20
```

---

### Why containers exist

```
BEFORE CONTAINERS — THE DEPENDENCY PROBLEM:
  Dev:  "It works on my machine!"
  Ops:  "Your machine has Python 3.8, prod has Python 3.6"
  Dev:  "My app needs libpq 14, the other app needs libpq 12"
  Ops:  "They can't both be installed on the same server"

CONTAINER SOLUTION:
  Each app ships WITH its dependencies (image layers)
  App A container: Python 3.8 + libpq 14 (isolated)
  App B container: Python 3.6 + libpq 12 (isolated)
  Both run on same host, no conflict

CONTAINER vs VM CHOICE:
  Use containers when: microservices, CI/CD, high density, fast deploy
  Use VMs when: strong security isolation required, different OS kernels,
                compliance (PCI/HIPAA), legacy applications needing full OS
  Both in Kubernetes: nodes are VMs, pods are containers inside those VMs
```

---

### 🎤 Short Crisp Interview Answer

> *"A container is a process on the host kernel isolated by Linux namespaces and constrained by cgroups. Namespaces provide visibility boundaries — the process gets its own PID tree, network stack, and filesystem view. cgroups enforce resource limits — CPU quota and memory cap. There's no guest OS, no hypervisor, and no hardware emulation. VMs are fundamentally different: they run a separate guest kernel via a hypervisor, providing true hardware-level isolation. The trade-off is startup time (milliseconds vs seconds) and density (100+ containers vs 10-20 VMs per host). In Kubernetes, nodes are typically VMs and containers run inside them — two layers of isolation."*

---

### ⚠️ Gotchas

1. **"Windows container on Linux" is impossible without a VM** — a Windows container requires a Windows kernel. Docker Desktop on Mac/Windows runs a hidden Linux VM and containers run in that VM's kernel, not the host OS kernel.
2. **Container isolation is weaker than VM isolation** — a kernel vulnerability can potentially escape container isolation since all containers share the kernel. VM kernel exploits require breaking the hypervisor too.
3. **PID 1 in a container has special significance** — the first process in a container receives all signals forwarded by Docker/Kubernetes. If PID 1 is a shell (from shell-form CMD), signals like SIGTERM don't reach the actual app.

---

---

# 13.2 Docker Architecture — Daemon, Client, containerd

## 🟢 Beginner

### What it is in simple terms

Docker is a **client-server system**. The `docker` CLI is the client. `dockerd` is the server (daemon). The daemon delegates actual container execution to `containerd`, which delegates to `runc` at the kernel level. Understanding this stack explains why Kubernetes can use `containerd` directly without Docker at all.

---

### The Full Architecture Stack

```
DOCKER ARCHITECTURE — COMPLETE STACK
═══════════════════════════════════════════════════════════════

USER SPACE:
  docker build / docker run / docker push
        │
        │  REST API over Unix socket
        │  /var/run/docker.sock
        ▼
  dockerd (Docker Daemon)
    Responsibilities:
    - Image management (pull, build, tag, push)
    - Container management (create, start, stop)
    - Network management (bridge creation, DNS)
    - Volume management
    - Exposes Docker API to clients
        │
        │  gRPC over Unix socket
        │  /run/containerd/containerd.sock
        ▼
  containerd (Container Runtime)
    Responsibilities:
    - Container lifecycle (create, start, stop, delete)
    - Image pull and storage (content store)
    - Snapshot management (OverlayFS layers)
    - OCI bundle creation
        │
        │  OCI runtime spec
        │  fork/exec
        ▼
  runc (OCI Runtime)
    Responsibilities:
    - Read OCI bundle (config.json + rootfs)
    - Set up Linux namespaces (clone() syscall)
    - Configure cgroups (resource limits)
    - Set up mounts (proc, sys, dev)
    - exec() the container process
    - Exits after handing off to container process

CONTAINER PROCESS:
  Your app (nginx, node, python, etc.)
  PID 1 inside container
  Running directly on host kernel

KUBERNETES REMOVES DOCKERD:
  kubectl → kubelet → containerd → runc → container
  Kubernetes uses containerd directly (via CRI — Container Runtime Interface)
  dockerd is not involved at all in Kubernetes pod lifecycle
  "Docker is deprecated in Kubernetes" = dockerd removed, containerd remains
```

---

### Docker Socket Security

```
/var/run/docker.sock — THE MOST DANGEROUS FILE ON A HOST
═══════════════════════════════════════════════════════════════

Mounting docker.sock in a container:
  docker run -v /var/run/docker.sock:/var/run/docker.sock myapp
  → Container can now send API calls to dockerd
  → Can create new containers with --privileged
  → Can mount host filesystem at /
  → FULL ROOT ACCESS TO THE HOST

Common (dangerous) use cases:
  - CI/CD agents that need to build images (Jenkins, GitLab runner)
  - Container monitoring tools (Portainer, cAdvisor)
  - Docker-in-Docker setups

SAFER ALTERNATIVES:
  Kaniko:    Build images in Kubernetes without docker.sock
  Buildah:   Build OCI images without daemon
  ko:        Build Go container images without Dockerfile
  Jib:       Build Java container images without Docker

In Kubernetes:
  NEVER mount docker.sock in pods
  Use IRSA + ECR + Kaniko for image builds
  PodSecurityAdmission (Restricted profile) blocks hostPath mounts
```

---

### Docker Commands — Architecture Inspection

```bash
# Check docker daemon info
docker info
# Shows: storage driver, cgroup driver, containerd version, runc version
# Server Version: 24.0.6
# Storage Driver: overlay2       ← OverlayFS in use
# Cgroup Driver: systemd         ← cgroup v2 integration
# Containerd Version: 1.6.24
# Runc Version: 1.1.9

docker version
# Client: Docker Engine - Community
#  Version: 24.0.6
# Server: Docker Engine - Community
#  Engine: Version: 24.0.6
#  containerd: Version: 1.6.24
#  runc: Version: 1.1.9

# Inspect the actual system calls (what runc does)
# strace the container start:
strace -f docker run --rm alpine echo hello 2>&1 | \
  grep -E "clone|unshare|setns" | head -10
# clone(... CLONE_NEWPID|CLONE_NEWNS|CLONE_NEWNET ... ) ← namespace creation

# See containerd is managing containers
sudo ctr containers list   # containerd's own CLI
sudo ctr namespaces list   # containerd namespaces (moby = docker's)
sudo ctr --namespace moby containers list
```

---

### 🎤 Short Crisp Interview Answer

> *"Docker is layered: the docker CLI talks to dockerd over a Unix socket REST API. dockerd delegates container execution to containerd via gRPC, and containerd uses runc to actually create the container at the kernel level. runc reads the OCI bundle, makes the clone() syscall to create Linux namespaces, configures cgroups for resource limits, and then exec()s the container process. runc itself exits after handoff — it's not a daemon. Kubernetes removed dockerd from the path entirely — kubelet talks directly to containerd via the CRI interface. The critical security point is /var/run/docker.sock — mounting this in any container gives that container root-equivalent access to the entire host."*

---

---

# ⚠️ 13.3 Images — Layers, Union Filesystem (OverlayFS), OCI Spec

## 🟢 Beginner — HIGH PRIORITY

### What it is in simple terms

A Docker image is a **stack of read-only filesystem layers** where each layer is a set of filesystem changes from one Dockerfile instruction. OverlayFS presents all layers as a single unified filesystem to the container. Multiple containers share the same read-only layers — only the thin writable layer on top is unique per container.

---

### Image Layer Architecture

```
IMAGE LAYERS — HOW THEY STACK
═══════════════════════════════════════════════════════════════

DOCKERFILE:                        RESULTING LAYERS:
  FROM ubuntu:22.04           →    Layer 0: ubuntu base filesystem (72MB)
  RUN apt-get install python3 →    Layer 1: python3 + deps added (45MB)
  COPY requirements.txt .     →    Layer 2: requirements.txt file (1KB)
  RUN pip install -r req.txt  →    Layer 3: pip packages installed (120MB)
  COPY src/ /app/src/         →    Layer 4: application source (2MB)
  CMD ["python3", "app.py"]   →    (no layer — just metadata)

TOTAL IMAGE SIZE: 239MB
Each layer stored as a tar.gz blob identified by sha256 hash.

HOW CONTAINERS USE THE IMAGE:
  Container A (my-api:v1):
  ┌─────────────────────────────────┐ ← writable layer (unique per container)
  │ Container write layer (empty)   │   changes go here, deleted on rm
  ├─────────────────────────────────┤
  │ Layer 4: src/ files      (2MB)  │ ← read-only, SHARED
  ├─────────────────────────────────┤
  │ Layer 3: pip packages  (120MB)  │ ← read-only, SHARED
  ├─────────────────────────────────┤
  │ Layer 2: requirements.txt (1KB) │ ← read-only, SHARED
  ├─────────────────────────────────┤
  │ Layer 1: python3        (45MB)  │ ← read-only, SHARED
  ├─────────────────────────────────┤
  │ Layer 0: ubuntu base    (72MB)  │ ← read-only, SHARED
  └─────────────────────────────────┘

  Container B (my-api:v1):
  ┌─────────────────────────────────┐ ← different writable layer
  │ Container write layer           │
  ├─────────────────────────────────┤
  │ SAME shared image layers        │ ← only one copy on disk!
  └─────────────────────────────────┘

100 containers from the same image = 100 writable layers + 1 set of image layers
Not 100x the image size on disk.
```

---

### OverlayFS — How Union Mounts Work

```
OVERLAYFS INTERNALS
═══════════════════════════════════════════════════════════════

OverlayFS is a union filesystem — presents multiple directories
as a single merged view.

COMPONENTS:
  lowerdir:  image layers (read-only, multiple dirs)
  upperdir:  container writable layer (read-write)
  workdir:   internal OverlayFS working directory
  merged:    the unified view shown to the container

MOUNT COMMAND (what Docker/containerd does internally):
  mount -t overlay overlay \
    -o lowerdir=/layer4:/layer3:/layer2:/layer1:/layer0 \
    -o upperdir=/containers/abc123/upper \
    -o workdir=/containers/abc123/work \
    /containers/abc123/merged

HOW READS WORK:
  Container reads /app/config.py:
  → OverlayFS checks upperdir first → not there
  → Checks lowerdir from top to bottom
  → Found in Layer 4 → returns it
  No copy, no overhead — direct read from layer blob

HOW WRITES WORK (Copy-on-Write):
  Container modifies /app/config.py:
  → File exists in Layer 4 (read-only, cannot write)
  → OverlayFS copies the ENTIRE file up to upperdir
  → Modification made in upperdir copy
  → Container sees modified version (upperdir shadows lowerdir)
  → Layer 4 original untouched (other containers unaffected)
  Cost: first write to a file = one file copy (amortized after that)

HOW DELETES WORK:
  Container deletes /app/config.py:
  → Cannot remove from lowerdir (read-only)
  → OverlayFS creates a "whiteout" file in upperdir
  → Whiteout: special file named .wh.config.py
  → OverlayFS hides any matching file in lowerdir
  → Container sees the file as gone
  → Layer blob still contains the original (size not reclaimed!)

ON-DISK LOCATION:
  /var/lib/docker/overlay2/<layer-id>/diff/    ← each image layer content
  /var/lib/docker/overlay2/<container-id>/     ← container upper+work dirs
  ls /var/lib/docker/overlay2/
  # Many directories, each a layer's content (diff/)

CHECK OVERLAYFS MOUNTS:
  cat /proc/mounts | grep overlay
  # overlay /var/lib/docker/containers/<id>/merged
  #   overlay rw,lowerdir=/var/lib/docker/overlay2/L4:L3:L2:L1:L0,
  #   upperdir=/var/lib/docker/overlay2/<container>/upper,...
```

---

### OCI Image Specification

```
OCI (OPEN CONTAINER INITIATIVE) IMAGE SPEC
═══════════════════════════════════════════════════════════════

OCI = industry standard for container images and runtimes.
Ensures any OCI-compliant image runs on any OCI-compliant runtime.
Docker, containerd, CRI-O, Podman all implement OCI.

AN IMAGE CONSISTS OF:
  1. Image Manifest (JSON)      — lists layers and config
  2. Image Config (JSON)        — runtime configuration
  3. Layer blobs (tar.gz files) — actual filesystem content

IMAGE MANIFEST:
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:abc123...",      ← config blob hash
    "size": 7023
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:def456...",    ← layer content hash
      "size": 32654322
    },
    {
      "digest": "sha256:ghi789...",
      "size": 1234567
    }
  ]
}

IMAGE CONFIG (what becomes OCI runtime bundle config):
{
  "architecture": "amd64",
  "os": "linux",
  "config": {
    "Env": ["PATH=/usr/local/bin:/usr/bin"],
    "Cmd": ["python3", "app.py"],
    "WorkingDir": "/app",
    "User": "1000",
    "ExposedPorts": {"8080/tcp": {}}
  },
  "rootfs": {
    "type": "layers",
    "diff_ids": ["sha256:abc...", "sha256:def..."]  ← uncompressed layer hashes
  }
}

CONTENT ADDRESSABILITY:
  Every component identified by sha256 hash of its content.
  If content changes → hash changes → different identity.
  Same hash = byte-identical content (tamper-proof).
  Layer shared between images if they have same digest.

TAGS vs DIGESTS:
  Tag:    nginx:1.25              — MUTABLE pointer (can be re-pushed)
  Digest: nginx@sha256:abc123...  — IMMUTABLE content address
  Production best practice: pin to digest for reproducibility
  CI/CD: build produces digest, deploy uses that exact digest
```

---

### Image Commands — Deep Inspection

```bash
# Inspect image layers
docker history nginx:1.25
# IMAGE          CREATED        CREATED BY                          SIZE
# sha256:abc123  2 weeks ago    CMD ["nginx" "-g" "daemon off;"]    0B
# <missing>      2 weeks ago    EXPOSE map[80/tcp:{}]               0B
# <missing>      2 weeks ago    COPY /docker-entrypoint.sh /         1.2kB
# <missing>      2 weeks ago    RUN /bin/sh -c set -x && addgroup   63.6MB
# <missing>      2 weeks ago    ENV PKG_RELEASE=1~bookworm           0B
# <missing>      2 weeks ago    FROM debian:bookworm-slim            74.8MB
# ↑ layers from bottom-up. SIZE=0B = metadata only instruction (no layer)

# Full image spec as JSON
docker image inspect nginx:1.25
# Shows: layers, config, env, cmd, exposed ports, etc.

# Get the image digest (immutable reference)
docker inspect nginx:1.25 \
  --format='{{index .RepoDigests 0}}'
# nginx@sha256:abc123def456...

# Check image layer disk usage
docker system df
# TYPE            TOTAL    ACTIVE   SIZE      RECLAIMABLE
# Images          15       3        4.2GB     2.8GB (66%)
# Containers      5        2        150MB     100MB (66%)
# Volumes         3        2        5GB       0B

# Inspect specific layer content (advanced)
# Save image as tar, inspect layers
docker save nginx:1.25 | tar -t | head -20
# manifest.json         ← image manifest
# config.json           ← image config
# <layer-hash>/layer.tar ← each layer as tar

# Pull by digest (immutable, secure, reproducible)
docker pull nginx@sha256:abc123def456789...
# Always fetches the exact same image

# Dive tool — interactive layer explorer
# brew install dive
dive nginx:1.25
# Shows each layer's filesystem changes interactively
```

---

### ⚠️ Critical Layer Gotchas

```
GOTCHA 1: DELETING IN A LATER LAYER DOESN'T REDUCE IMAGE SIZE
═══════════════════════════════════════════════════════════════
WRONG approach:
  RUN wget https://example.com/bigfile.tar.gz     ← Layer N: +500MB
  RUN rm bigfile.tar.gz                           ← Layer N+1: whiteout (still 500MB total!)

The layer N blob still contains the file.
Image size = Layer N + Layer N+1 = still 500MB+

CORRECT approach:
  RUN wget https://example.com/bigfile.tar.gz \   ← one layer
      && tar xzf bigfile.tar.gz \
      && rm bigfile.tar.gz                        ← all in same layer = small!

GOTCHA 2: LAYER CACHE INVALIDATED FROM CHANGED LAYER DOWNWARD
═══════════════════════════════════════════════════════════════
  FROM ubuntu:22.04              ← cache hit (unchanged)
  RUN apt-get install python3    ← cache hit (unchanged)
  COPY requirements.txt .        ← cache MISS if requirements.txt changed
  RUN pip install -r req.txt     ← REBUILDS (cache invalidated)
  COPY src/ .                    ← REBUILDS (cache invalidated)
  All layers after cache miss rebuild from scratch.

CORRECT order: least-frequently-changed instructions FIRST.
```

---

### 🎤 Short Crisp Interview Answer

> *"A Docker image is a stack of read-only content-addressed layers defined by the OCI image spec. Each layer is a tar.gz blob identified by sha256 hash. OverlayFS presents them as a unified filesystem to the container — reads walk layers from top to bottom, and writes use copy-on-write (the file is copied up to the container's writable upperdir before modification, leaving image layers untouched). Multiple containers share the same image layers on disk — 100 nginx containers use one set of shared layers plus 100 tiny per-container writable layers. The critical gotcha: deleting a file in a later Dockerfile RUN instruction doesn't reduce image size because the original layer blob still exists on disk. The fix is combining the add and delete in a single RUN command."*

---

---

# ⚠️ 13.4 Dockerfile — Instructions, Layer Caching, Best Practices

## 🟢 Beginner — HIGH PRIORITY

### Complete Instruction Reference

```dockerfile
# ═══════════════════════════════════════════════════
# ALL DOCKERFILE INSTRUCTIONS WITH EXPLANATION
# ═══════════════════════════════════════════════════

# FROM — base image (must be first non-ARG instruction)
FROM ubuntu:22.04
FROM node:20-alpine          # alpine base = minimal (~5MB)
FROM scratch                 # empty base — for static Go binaries

# ARG — build-time variable (NOT in final image, NOT in running container)
ARG VERSION=1.0.0
ARG BUILD_DATE
# Use: docker build --build-arg VERSION=2.0.0 .
# ARG before FROM: only usable in FROM
# ARG after FROM: available in build steps

# ENV — runtime environment variable (persisted in image config)
ENV NODE_ENV=production
ENV PORT=8080
ENV PATH="/app/bin:${PATH}"
# Available at runtime: docker run → container sees these env vars

# WORKDIR — set working directory (creates if not exists)
WORKDIR /app
WORKDIR /app/src              # nested: creates /app then /app/src
# Prefer over: RUN mkdir -p /app && cd /app (WORKDIR is cleaner and cacheable)

# COPY — copy files from build context into image (PREFERRED)
COPY package.json .                         # copy to current WORKDIR
COPY package.json package-lock.json ./      # multiple files → WORKDIR
COPY src/ ./src/                            # copy directory
COPY --chown=node:node . .                  # set ownership (saves a RUN chown)
COPY --from=builder /app/dist ./dist        # from another build stage
COPY --chmod=755 entrypoint.sh /entrypoint.sh  # set file permissions

# ADD — like COPY but with magic features (prefer COPY for clarity)
ADD app.tar.gz /app/          # auto-extracts local tar archives
ADD https://example.com/file /app/  # downloads URL (no cache busting!)
# Rule: use COPY unless you specifically need tar extraction or URL fetch

# RUN — execute command during build (creates a new layer if filesystem changes)
RUN apt-get update && apt-get install -y curl
RUN --mount=type=cache,target=/var/cache/apt \  # BuildKit cache mount
    apt-get update && apt-get install -y curl
# Shell form: RUN command          → /bin/sh -c "command"
# Exec form:  RUN ["cmd", "arg"]   → directly exec, no shell

# USER — set user for subsequent RUN, CMD, ENTRYPOINT instructions
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser
USER 1001:1001                # by UID:GID (more portable, no name lookup)
# Best practice: never run as root (UID 0) in production

# EXPOSE — document port (informational ONLY, does NOT publish)
EXPOSE 8080
EXPOSE 8080/tcp 9090/udp
# This is metadata. Actual port publishing: docker run -p 8080:8080

# VOLUME — declare mount point
VOLUME ["/data"]
VOLUME /logs /tmp
# Creates anonymous volume at this path if no external mount provided
# Best practice: prefer explicit -v at runtime over VOLUME instruction

# CMD — default command when container starts (overridable)
CMD ["node", "server.js"]                    # exec form ✓ (preferred)
CMD ["nginx", "-g", "daemon off;"]
CMD node server.js                            # shell form ✗ (sh becomes PID 1)

# ENTRYPOINT — container's fixed executable (not easily overridden)
ENTRYPOINT ["python3", "app.py"]             # exec form
ENTRYPOINT ["/docker-entrypoint.sh"]         # init script pattern
# To override: docker run --entrypoint /bin/sh myimage

# CMD + ENTRYPOINT together (most flexible pattern):
ENTRYPOINT ["python3", "manage.py"]
CMD ["runserver", "0.0.0.0:8080"]           # default args to ENTRYPOINT
# docker run myapp                    → python3 manage.py runserver 0.0.0.0:8080
# docker run myapp migrate            → python3 manage.py migrate
# docker run myapp shell              → python3 manage.py shell

# HEALTHCHECK — container health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
HEALTHCHECK NONE                             # disable any inherited healthcheck

# LABEL — arbitrary metadata
LABEL maintainer="team@company.com"
LABEL version="1.0.0"
LABEL org.opencontainers.image.source="https://github.com/org/repo"
LABEL org.opencontainers.image.revision="${GIT_SHA}"

# STOPSIGNAL — signal to send when stopping container (default: SIGTERM)
STOPSIGNAL SIGQUIT            # nginx uses SIGQUIT for graceful drain+shutdown
STOPSIGNAL SIGINT             # gunicorn, some Node apps prefer SIGINT

# ONBUILD — trigger when this image is used as a base
ONBUILD COPY . /app
ONBUILD RUN npm ci
# Useful for base images used by teams
# Executed when: FROM mybase-image is used in another Dockerfile

# SHELL — change the shell used for RUN, CMD, ENTRYPOINT shell form
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
# Now: RUN false | true  → fails (pipefail) instead of silently succeeding
```

---

### CMD vs ENTRYPOINT — The Critical Difference

```
CMD vs ENTRYPOINT DEEP DIVE
═══════════════════════════════════════════════════════════════

RULE:
  ENTRYPOINT = the executable (what the container IS)
  CMD         = default arguments (what it does by default)

EXEC FORM vs SHELL FORM — THIS IS CRITICAL:

  Exec form:  CMD ["node", "server.js"]
  → docker creates: node server.js
  → node is PID 1 inside container
  → SIGTERM from Docker/Kubernetes goes directly to node ✓
  → Graceful shutdown works

  Shell form: CMD node server.js
  → docker creates: /bin/sh -c "node server.js"
  → sh is PID 1 inside container
  → SIGTERM from Docker/Kubernetes goes to sh, NOT node ✗
  → sh doesn't forward signals to children
  → node gets SIGKILL after timeout (ungraceful) ✗
  → ALWAYS use exec form for CMD and ENTRYPOINT

PATTERNS:

Pattern 1 — CMD only (simple apps):
  CMD ["nginx", "-g", "daemon off;"]
  # docker run nginx              → nginx -g "daemon off;"
  # docker run nginx /bin/sh      → /bin/sh (CMD overridden)

Pattern 2 — ENTRYPOINT + CMD (configurable tools):
  ENTRYPOINT ["python3", "manage.py"]
  CMD ["runserver", "0.0.0.0:8080"]
  # docker run myapp                  → python3 manage.py runserver...
  # docker run myapp migrate          → python3 manage.py migrate

Pattern 3 — init script (most production-ready):
  COPY entrypoint.sh /entrypoint.sh
  ENTRYPOINT ["/entrypoint.sh"]       # does setup (wait for DB, env var checks)
  CMD ["node", "server.js"]           # then execs this

  # entrypoint.sh:
  #!/bin/sh
  set -e
  # Wait for dependencies
  until nc -z postgres 5432; do sleep 1; done
  # Check required env vars
  : "${DATABASE_URL:?DATABASE_URL must be set}"
  # Hand off to CMD
  exec "$@"                           # ← exec replaces shell as PID 1!
  # Without exec: entrypoint.sh is PID 1, node is child
  # SIGTERM goes to entrypoint.sh which may not forward it
```

---

### Layer Caching — The Most Important Performance Topic

```
LAYER CACHE RULES
═══════════════════════════════════════════════════════════════

RULE 1: Cache invalidated from first changed layer downward
  If Layer 3 is a cache miss → Layers 4, 5, 6... all rebuild

RULE 2: COPY/ADD always check file content (not just mtime)
  COPY package.json .    → cache miss if package.json content changed
  COPY src/ .            → cache miss if ANY file in src/ changed

RULE 3: RUN cache based on the command string
  RUN apt-get update     → always cache hit (string unchanged!)
                           DANGEROUS: apt-get update may be stale
  Fix: combine update+install in one RUN:
  RUN apt-get update && apt-get install -y curl  → fresh every rebuild

OPTIMAL ORDERING STRATEGY:
  ① Base image and system packages   (changes: rarely)
  ② Runtime dependencies             (changes: monthly)
  ③ Dependency manifest files        (changes: weekly — package.json, pom.xml)
  ④ Install dependencies             (changes: when ③ changes)
  ⑤ Application configuration        (changes: occasionally)
  ⑥ Application source code          (changes: every commit)

EXAMPLE — WRONG ORDER (slow builds):
  FROM node:20
  COPY . .               ← ALL source copied first → any change = full rebuild
  RUN npm ci             ← always reinstalls all packages!
  CMD ["node", "server.js"]

EXAMPLE — CORRECT ORDER (fast builds):
  FROM node:20
  WORKDIR /app
  COPY package.json package-lock.json ./   ← only manifests (rarely changes)
  RUN npm ci                               ← cached unless package.json changes
  COPY src/ ./src/                         ← source code last (changes often)
  COPY config/ ./config/
  CMD ["node", "server.js"]
  # Source code change: only last COPY is a cache miss
  # npm ci: cache HIT → no re-download of packages
```

---

### Production Dockerfile Best Practices

```dockerfile
# ═══════════════════════════════════════════════════
# PRODUCTION-READY DOCKERFILE — ALL BEST PRACTICES
# ═══════════════════════════════════════════════════

# 1. PIN EXACT BASE IMAGE VERSION
FROM node:20.11.0-alpine3.19
# Never: FROM node:latest  (breaks on next release)
# Never: FROM node:20      (breaks on 20.X patch)

# 2. SET WORKDIR (not RUN mkdir)
WORKDIR /app

# 3. CREATE NON-ROOT USER EARLY
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# 4. COPY DEPENDENCY MANIFESTS BEFORE SOURCE
COPY --chown=appuser:appgroup package.json package-lock.json ./

# 5. INSTALL DEPENDENCIES (cached until package.json changes)
RUN npm ci --only=production \
    && npm cache clean --force

# 6. COPY SOURCE LAST (changes most often)
COPY --chown=appuser:appgroup src/ ./src/
COPY --chown=appuser:appgroup config/ ./config/

# 7. SWITCH TO NON-ROOT USER BEFORE CMD
USER appuser

# 8. EXPOSE ONLY WHAT'S NEEDED
EXPOSE 8080

# 9. HEALTHCHECK
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:8080/health || exit 1

# 10. CORRECT STOPSIGNAL
STOPSIGNAL SIGTERM

# 11. EXEC FORM CMD
CMD ["node", "src/server.js"]

# 12. LABELS FOR TRACEABILITY
LABEL org.opencontainers.image.source="https://github.com/org/repo"
LABEL org.opencontainers.image.revision="${GIT_SHA}"
```

---

### .dockerignore

```
# .dockerignore — excludes files from build context
# Build context = everything sent to Docker daemon before build starts
# Large context = slow build startup even if files aren't COPYed

node_modules/         # npm install will recreate inside container
.git/                 # version history not needed in image
*.log                 # runtime logs not needed
.env                  # NEVER put secrets in image
.env.*                # all env file variants
.nyc_output/          # test coverage output
coverage/             # test coverage HTML
dist/                 # if building inside container (multi-stage)
test/                 # tests (don't run in prod container)
*.test.js             # test files
*.spec.js
docs/                 # documentation
*.md                  # README etc.
.github/              # CI config
Dockerfile            # don't need Dockerfile inside image
docker-compose*.yml   # compose files not needed inside
.dockerignore         # don't need .dockerignore inside
```

---

### 🎤 Short Crisp Interview Answer

> *"The most important Dockerfile rules: order instructions from least-to-most-frequently-changed to maximize cache reuse — copy package.json before source code so npm install is cached across source-only changes. Combine RUN commands that add and delete files in a single instruction — deleting in a separate layer doesn't reduce image size because the original layer blob still exists. Always use exec form for CMD and ENTRYPOINT (square brackets) — shell form makes sh PID 1 which doesn't forward SIGTERM to the app, breaking graceful shutdown. Never run as root — add a non-root user and switch to it with USER. Use .dockerignore to exclude node_modules and .git from the build context."*

---

---

# 13.5 Container Lifecycle — create, start, stop, kill, exit codes

## 🟢 Beginner

### Container State Machine

```
CONTAINER STATE TRANSITIONS
═══════════════════════════════════════════════════════════════

                 docker create
  [nonexistent] ──────────────→ [created]
                                    │
                              docker start
                                    │
                                    ▼
  [paused] ←── docker pause ── [running] ──── docker stop ──→ [stopped]
     │                             │                              │
  docker unpause                   │                           docker start
     │                        process exits                       │
     └──────────────────────→ [stopped] ◄─────────────────────────┘
                                    │
                               docker rm
                                    ▼
                               [deleted]

SHORTCUT:
  docker run = docker create + docker start (most common)

SIGNALS:
  docker stop  → SIGTERM → wait (--time=10s default) → SIGKILL
  docker kill  → SIGKILL immediately (or any signal with --signal)
  docker pause → SIGSTOP to all processes in cgroup (freeze, not kill)

RESTART POLICIES:
  --restart=no              default, never restart
  --restart=always          restart even after docker daemon restart
  --restart=unless-stopped  like always but NOT after manual docker stop
  --restart=on-failure:5    restart up to 5 times on non-zero exit
```

---

### Exit Codes — Diagnostic Reference

```
EXIT CODES AND THEIR MEANING
═══════════════════════════════════════════════════════════════

Exit Code   Meaning                   Common Cause
─────────────────────────────────────────────────────────────
0           Clean exit                Process finished successfully
1           Generic error             App-level error (check app logs)
2           Misuse of shell command   Bad command syntax in shell form
126         Command not executable    ENTRYPOINT/CMD not executable (chmod)
127         Command not found         ENTRYPOINT/CMD binary doesn't exist
128+N       Killed by signal N        N = signal number
137         128 + 9  (SIGKILL)        OOM killed, or docker kill
143         128 + 15 (SIGTERM)        Graceful stop requested
130         128 + 2  (SIGINT)         Ctrl+C pressed
139         128 + 11 (SIGSEGV)        Segmentation fault (memory bug)
255         runc/runtime error        Container couldn't start

IN KUBERNETES:
  Exit 137:  kubectl describe pod → "OOMKilled"
             OR: manual kubectl delete pod / node drain
  Exit 143:  Pod terminated gracefully (expected for rolling update)
  Exit 1:    Application crash — check logs
  Exit 127:  Wrong CMD path — image built incorrectly
```

---

```bash
# Container lifecycle commands with all options
docker create --name myapp nginx:1.25       # create (not started)
docker start myapp                           # start created container
docker stop myapp                            # SIGTERM → wait → SIGKILL
docker stop --time=30 myapp                 # 30s grace period before SIGKILL
docker kill myapp                            # immediate SIGKILL
docker kill --signal=SIGUSR1 myapp          # send specific signal (e.g. log rotate)
docker restart myapp                         # stop + start
docker restart --time=5 myapp               # 5s grace period
docker pause myapp                           # freeze all processes (SIGSTOP)
docker unpause myapp                         # resume
docker rm myapp                              # delete stopped container
docker rm -f myapp                           # force delete running container
docker run --rm nginx                        # auto-delete on exit

# Inspect exit code
docker inspect myapp \
  --format='{{.State.ExitCode}} {{.State.Status}} {{.State.Error}}'
# 137 exited OOMKilled

# All container states
docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Resource limits at run time
docker run \
  --cpus="1.5" \           # max 1.5 CPUs
  --memory="512m" \        # max 512MB RAM
  --memory-swap="1g" \     # max 1GB (RAM + swap combined)
  --memory-reservation="256m" \  # soft limit (request)
  --pids-limit=100 \       # max 100 processes
  --ulimit nofile=1024:1024 \    # file descriptor limit
  nginx:1.25
```

---

# 13.6 Volumes & Bind Mounts — Named Volumes, tmpfs

## 🟢 Beginner

### Storage Types Comparison

```
DOCKER STORAGE OPTIONS
═══════════════════════════════════════════════════════════════

BIND MOUNT:
  Source: specific path on host filesystem
  Destination: path inside container
  Managed by: you (must exist on host)
  Syntax: -v /host/path:/container/path
          --mount type=bind,source=/host/path,target=/container/path
  Use: development (live code reload), sharing host configs
  Risk: host path dependency, sensitive host files accessible

NAMED VOLUME:
  Source: Docker-managed directory (/var/lib/docker/volumes/<name>/)
  Destination: path inside container
  Managed by: Docker
  Syntax: -v myvolume:/container/path
          --mount type=volume,source=myvolume,target=/container/path
  Survives: container deletion (data persists)
  Shareable: multiple containers can mount same volume
  Use: persistent data (database files, uploads, certificates)

ANONYMOUS VOLUME:
  Like named volume but auto-named with UUID
  Created by: VOLUME instruction in Dockerfile
  Hard to manage: no meaningful name to reference
  Prefer: named volumes for all persistent data needs

TMPFS MOUNT:
  Source: memory (RAM), never touches disk
  Survives: container lifetime only (deleted when container stops)
  Syntax: --tmpfs /container/path:size=100m,mode=1777
  Use: sensitive temporary data, high-speed scratch space,
       secrets that must not be written to disk

COMPARISON:
  Persistence     bind=host survives    named=survives    tmpfs=container lifetime
  Performance     bind=native           named=native      tmpfs=RAM speed
  Portability     bind=host-dependent   named=portable    tmpfs=portable
  Secrets-safe    bind=no               named=no          tmpfs=yes (no disk write)
```

---

```bash
# ─── NAMED VOLUMES ────────────────────────────────────────────────
docker volume create mydata
docker volume ls
# DRIVER    VOLUME NAME
# local     mydata
# local     postgres-data

docker volume inspect mydata
# "Mountpoint": "/var/lib/docker/volumes/mydata/_data"  ← on host here

docker run -d \
  -v postgres-data:/var/lib/postgresql/data \
  --name postgres \
  postgres:16

docker volume rm mydata                     # only works if no container using it
docker volume prune                         # remove all unused volumes

# ─── BIND MOUNTS ─────────────────────────────────────────────────
# Development workflow — live reload
docker run -it \
  -v $(pwd)/src:/app/src \                  # bind source code
  -v $(pwd)/config:/app/config:ro \         # read-only config
  -p 3000:3000 \
  node:20 \
  npm run dev

# ─── TMPFS ───────────────────────────────────────────────────────
docker run \
  --tmpfs /tmp:size=100m,mode=1777 \        # 100MB in-memory /tmp
  --tmpfs /run:size=10m \
  myapp

# ─── BACKUP AND RESTORE VOLUMES ──────────────────────────────────
# Backup: tar volume content out via temp container
docker run --rm \
  -v mydata:/source:ro \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/mydata-backup.tar.gz -C /source .

# Restore: tar content into new volume
docker volume create mydata-restored
docker run --rm \
  -v mydata-restored:/target \
  -v $(pwd):/backup:ro \
  alpine \
  tar xzf /backup/mydata-backup.tar.gz -C /target

# ─── VOLUME DRIVERS (plugins) ─────────────────────────────────────
# NFS volume (share across hosts)
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/exports/mydata \
  nfs-volume

# In Kubernetes: PVC replaces all Docker volume management
# emptyDir  ≈ anonymous volume / tmpfs
# hostPath  ≈ bind mount
# PVC       ≈ named volume (but much more powerful)
```

---

# 13.7 Docker Networking — bridge, host, none, user-defined bridge

## 🟡 Intermediate

### Network Driver Types

```
DOCKER NETWORK DRIVERS
═══════════════════════════════════════════════════════════════

BRIDGE (default network driver):
  Creates a virtual bridge (docker0) on the host
  Container gets its own network namespace + virtual ethernet
  Container IP: 172.17.0.x (default) or custom subnet
  Container → outside: NAT via iptables masquerade rules
  Outside → container: requires explicit port publishing (-p)
  Container ↔ container (default bridge): by IP only (no DNS)
  Container ↔ container (user bridge): by container NAME (DNS)

  default bridge (docker0):
    Legacy. No automatic DNS by container name.
    Containers can only find each other by IP.
    docker run --link is the old hack for this (deprecated).

  user-defined bridge (recommended):
    docker network create mynet
    Containers on same user bridge: DNS by container name
    Isolated from default bridge traffic
    Better security and flexibility

HOST:
  Container shares host network namespace completely
  No isolation: binds directly to host interfaces
  No private IP: container IS the host from network perspective
  Port -p not needed (and doesn't work): container just binds to host port
  Faster: no NAT overhead (~5% improvement for network-intensive apps)
  docker run --network=host nginx
  Risk: port conflicts, no network isolation between containers

NONE:
  No network interfaces (except loopback lo)
  Completely isolated: cannot send or receive external traffic
  Use: security-critical one-shot jobs (process local data only)
  docker run --network=none myapp

OVERLAY:
  Spans multiple Docker hosts via VXLAN tunnels
  Used by Docker Swarm for multi-host container networking
  Kubernetes does NOT use overlay networks from Docker
  Kubernetes uses CNI plugins (Flannel, Calico, Cilium) instead

MACVLAN:
  Container gets a real MAC address on the physical network
  Appears as a physical device on the LAN
  Use: legacy applications that need to be directly on the network
  Requires promiscuous mode on NIC (not available in most VMs)

IPvlan:
  Similar to macvlan but all containers share host's MAC address
  Layer 2 (L2) and Layer 3 (L3) modes
  Better VM compatibility than macvlan (no promiscuous mode needed)
```

---

```bash
# ─── NETWORK MANAGEMENT ───────────────────────────────────────────
docker network ls
# NETWORK ID     NAME       DRIVER    SCOPE
# abc123def456   bridge     bridge    local   ← default bridge (docker0)
# ghi789jkl012   host       host      local
# mnop345qrs6    none       null      local

# Create user-defined bridge
docker network create \
  --driver bridge \
  --subnet=10.10.0.0/24 \
  --gateway=10.10.0.1 \
  --opt com.docker.network.bridge.name=br-myapp \  # name on host
  myapp-network

# Run containers on user-defined network (can reach each other by name)
docker run -d \
  --name postgres \
  --network myapp-network \
  postgres:16

docker run -d \
  --name api \
  --network myapp-network \
  -p 8080:8080 \
  my-api:v1

# From 'api' container: can reach postgres by name
docker exec api ping postgres        # works! DNS resolves postgres → 10.10.0.x
docker exec api curl http://postgres:5432  # works!

# Inspect network
docker network inspect myapp-network
# Shows: subnet, gateway, containers and their IPs

# Connect running container to another network
docker network connect myapp-network my-other-container
docker network disconnect myapp-network my-other-container

# ─── PORT PUBLISHING ──────────────────────────────────────────────
docker run -p 8080:80 nginx           # host port 8080 → container port 80
docker run -p 127.0.0.1:8080:80 nginx # bind to localhost only (secure)
docker run -p 80 nginx                # random available host port
docker run -P nginx                   # publish all EXPOSEd ports (random)

# See assigned ports
docker port my-nginx
# 80/tcp -> 0.0.0.0:32768

# ─── NETWORK DIAGNOSTICS ──────────────────────────────────────────
# Container's network config
docker exec my-container ip addr show
docker exec my-container ip route show
docker exec my-container cat /etc/resolv.conf

# Which containers are on which networks
docker network inspect bridge \
  --format='{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{"\n"}}{{end}}'
```

---

# 13.8 Image Registries — push, pull, tags vs digests, ECR auth

## 🟡 Intermediate

### Registry Concepts

```
IMAGE NAMING CONVENTION
═══════════════════════════════════════════════════════════════

Full image reference:
  [registry/][namespace/]repository[:tag][@digest]

Examples:
  nginx:1.25
    = docker.io/library/nginx:1.25            (Docker Hub official)
  myuser/myapp:v1.2.3
    = docker.io/myuser/myapp:v1.2.3           (Docker Hub personal)
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.2.3
                                              (AWS ECR)
  gcr.io/my-project/myapp:v1.2.3             (Google Container Registry)
  ghcr.io/myorg/myapp:latest                 (GitHub Container Registry)
  registry.company.com:5000/myapp:stable     (private registry with port)

TAGS:
  :latest    — mutable, points to most recent push (DON'T rely on it)
  :1.25      — version tag, could be re-pushed (mostly stable)
  :1.25.3    — patch version, rarely re-pushed
  :main      — branch-based tag, changes with every merge
  :abc1234   — git SHA tag (effectively immutable)

DIGESTS:
  @sha256:abc123def456...
  Immutable content address. If content changes, digest changes.
  Guarantees you always get EXACTLY this image.
  Production deployments should use digests for reproducibility.
  Or: pin to an exact semver tag that your registry policy forbids re-pushing.
```

---

```bash
# ─── BASIC REGISTRY OPERATIONS ────────────────────────────────────
# Login
docker login                                         # Docker Hub (interactive)
docker login registry.company.com -u user -p pass    # private registry
docker login ghcr.io -u username --password-stdin    # stdin (no history)

# Pull
docker pull nginx:1.25                               # by tag
docker pull nginx@sha256:abc123...                   # by digest (immutable)

# Tag (create an alias pointing to same image)
docker tag myapp:v1.2.3 \
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.2.3
docker tag myapp:v1.2.3 myapp:latest                 # also tag as latest

# Push
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.2.3

# ─── ECR AUTHENTICATION ───────────────────────────────────────────
# ECR tokens expire every 12 hours — must refresh before push/pull

# Get login token and authenticate docker
aws ecr get-login-password \
  --region us-east-1 | \
docker login \
  --username AWS \
  --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# Create ECR repository
aws ecr create-repository \
  --repository-name myapp \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=KMS

# CI/CD pipeline pattern (GitHub Actions / GitLab CI)
# 1. Configure IRSA or AWS credentials in CI
# 2. Run: aws ecr get-login-password | docker login ...
# 3. docker build -t $ECR_REPO:$GIT_SHA .
# 4. docker push $ECR_REPO:$GIT_SHA
# 5. kubectl set image deployment/myapp app=$ECR_REPO:$GIT_SHA

# ─── INSPECT REGISTRY CONTENT ─────────────────────────────────────
# Get image digest without pulling
docker manifest inspect nginx:1.25 | python3 -m json.tool | grep digest
# Pulls only manifest, not layers

# List ECR image tags
aws ecr list-images \
  --repository-name myapp \
  --region us-east-1 \
  --query 'imageIds[*].imageTag' \
  --output table

# Get digest of local image
docker inspect myapp:v1.2.3 \
  --format='{{index .RepoDigests 0}}'
# 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp@sha256:abc123...

# ─── REGISTRY CREDENTIALS IN KUBERNETES ──────────────────────────
# Create imagePullSecret for private registry
kubectl create secret docker-registry regcred \
  --docker-server=123456789.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1) \
  -n production

# Attach to ServiceAccount (all pods using SA get pull access)
kubectl patch serviceaccount default \
  -n production \
  -p '{"imagePullSecrets": [{"name": "regcred"}]}'

# OR: specify per-pod
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: app
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.2.3

# EKS best practice: use IRSA for node group + ECR policy
# Nodes automatically authenticate to ECR in same account — no imagePullSecret needed
```

---

# ⚠️ 13.9 Linux Namespaces — pid, net, mnt, uts, ipc, user

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

Linux namespaces are the **kernel mechanism that creates isolation boundaries** for containers. Each namespace type wraps a different resource domain, making the process believe it has exclusive access to that resource. A container is simply a process that has been placed into multiple namespaces simultaneously.

---

### All Seven Namespace Types

```
LINUX NAMESPACES — COMPLETE REFERENCE
═══════════════════════════════════════════════════════════════

PID NAMESPACE (CLONE_NEWPID):
  Isolates: process IDs
  Effect: container has its own PID tree starting at PID 1
          processes outside the namespace are invisible
          PID 1 in container ≠ PID 1 on host (host has its own PID 1)
  Container PID 1: your app (nginx, node, python)
  Host PID:        your app appears with a high PID like 23847
  Impact: kill -9 23847 on HOST kills the container process
          kill -9 1 INSIDE container kills only that container

  Verify:
  docker run --rm alpine ps aux
  # PID USER  COMMAND
  # 1   root  ps aux   ← only sees its own processes, PID 1

  On host: ps aux | grep alpine
  # 23847 root ps aux  ← same process, different PID

NETWORK NAMESPACE (CLONE_NEWNET):
  Isolates: network interfaces, routing tables, iptables rules, sockets, ports
  Effect: container gets its own eth0 with private IP
          cannot see host's eth0 or other containers' interfaces
          can bind port 80 even if host port 80 is taken
  Implementation: veth pair (virtual ethernet cable)
    One end (eth0) inside container namespace
    Other end (vethXXXXXX) on host, bridged to docker0

  Verify:
  docker run --rm alpine ip addr
  # lo:   127.0.0.1
  # eth0: 172.17.0.2   ← private IP, not host IP

  On host: ip addr | grep docker0
  # docker0: 172.17.0.1  ← bridge, not the container's eth0

MOUNT NAMESPACE (CLONE_NEWNS):
  Isolates: filesystem mount points
  Effect: container sees its own filesystem root from image layers
          host filesystem not visible (unless bind-mounted in)
          container can mount/unmount without affecting host
  Implementation: OverlayFS with image layers as lowerdir
  Verify:
  docker run --rm alpine ls /
  # app  bin  etc  home  lib  ...  ← image filesystem
  # NOT host filesystem

UTS NAMESPACE (CLONE_NEWUTS):  [Unix Time-sharing System]
  Isolates: hostname and NIS domain name
  Effect: container can have its own hostname
          hostname inside container ≠ hostname on host
  Verify:
  docker run --rm --hostname my-container alpine hostname
  # my-container   ← container's hostname
  hostname                                  # on host
  # worker-node-1  ← host's hostname

IPC NAMESPACE (CLONE_NEWIPC):  [Inter-Process Communication]
  Isolates: System V IPC objects (semaphores, message queues, shared memory)
            POSIX message queues
  Effect: containers cannot communicate via IPC with host or other containers
          each container has its own set of IPC resources
  Use case: two tightly-coupled processes needing shared memory
  docker run --ipc=container:my-other-container app  # share IPC namespace

USER NAMESPACE (CLONE_NEWUSER):
  Isolates: user and group IDs
  Effect: UID 0 (root) inside container → maps to non-root UID on host
          container believes it has root but host sees it as regular user
  Most security benefit: privilege escalation inside container ≠ host root
  Status: NOT used by default in Docker (still optional, off by default)
          required for rootless Docker and Podman
  Verify (rootless Docker):
  docker run --rm alpine id
  # uid=0(root)        ← sees itself as root
  # on host: ps aux shows the process running as uid=1000

CGROUP NAMESPACE (CLONE_NEWCGROUP):  [added in Linux 4.6]
  Isolates: cgroup hierarchy view
  Effect: container sees its own cgroup tree root (not host's)
          prevents container from seeing host cgroup structure
  Usually created automatically with other namespaces
```

---

### Namespace Inspection Commands

```bash
# See namespaces of a running container
CONTAINER_PID=$(docker inspect my-container \
  --format='{{.State.Pid}}')
echo "Container PID on host: $CONTAINER_PID"

# List the container's namespaces
ls -la /proc/$CONTAINER_PID/ns/
# lrwxrwxrwx cgroup → cgroup:[4026531835]
# lrwxrwxrwx ipc    → ipc:[4026532456]
# lrwxrwxrwx mnt    → mnt:[4026532457]    ← different from host
# lrwxrwxrwx net    → net:[4026532459]    ← different from host
# lrwxrwxrwx pid    → pid:[4026532460]    ← different from host
# lrwxrwxrwx uts    → uts:[4026532458]    ← different from host
# lrwxrwxrwx user   → user:[4026531837]   ← SAME as host (user ns not isolated by default)

# Host namespaces for comparison
ls -la /proc/1/ns/
# All numbers are different from container (different namespaces)

# Enter a container's namespace manually (nsenter)
nsenter --target $CONTAINER_PID \
  --pid --net --mount \
  -- /bin/sh
# Now running in container's namespaces but as host user (useful for debugging)

# Share namespaces between containers
docker run --pid=container:my-app debug-tools
# Shares PID namespace: debug container sees my-app's processes
# kubectl debug equivalent: --target=container-name

# Network namespace sharing (pods in Kubernetes)
docker run --network=container:my-app debug-tools
# Shares network namespace: same IP, same ports
# This is how Kubernetes pod containers share a network — all join pause container's netns

# Inspect network namespace directly
nsenter --target $CONTAINER_PID --net ip addr
# Shows container's network interfaces from host side
```

---

### The Kubernetes Pause Container

```
HOW KUBERNETES USES NAMESPACES FOR PODS
═══════════════════════════════════════════════════════════════

Kubernetes Pod = multiple containers sharing certain namespaces.

PAUSE CONTAINER:
  Every Pod has a "pause" (aka "infra") container running google/pause
  Pause container's purpose: hold the network and IPC namespaces
  Pause just runs: sleep infinity (does nothing)

  All other containers in the pod JOIN pause container's namespaces:
    --network=container:pause    → share network namespace
    --ipc=container:pause        → share IPC namespace
    PID namespace: optionally shared (shareProcessNamespace: true)
    Mount namespace: NOT shared (each container has own filesystem)

RESULT:
  All containers in a pod:
  ✓ Same IP address
  ✓ Same port space (cannot both bind port 8080)
  ✓ Can reach each other via localhost
  ✓ Can share memory via IPC
  ✗ Cannot see each other's filesystems (separate mnt namespaces)
  ✗ Cannot see each other's processes by default (separate PID namespaces)

WHY PAUSE?
  If main container dies and restarts → new container ID → would lose namespace
  Pause container is stable — it never restarts
  Network/IPC namespace persists across container restarts
  Pod gets stable IP even as app containers restart
```

---

### 🎤 Short Crisp Interview Answer

> *"Linux namespaces are the kernel feature that isolates containers. There are seven types: PID (own process tree starting at PID 1), Network (own ethernet interface, routing table, ports), Mount (own filesystem view from image layers), UTS (own hostname), IPC (own semaphores and message queues), User (UID mapping — root in container maps to non-root on host), and cgroup (own cgroup hierarchy view). A container is just a process running in multiple namespaces simultaneously. In Kubernetes, the pause container holds the network and IPC namespaces for a pod — all app containers join the pause container's namespaces, which is why all containers in a pod share the same IP and can communicate via localhost."*

---

---

# ⚠️ 13.10 cgroups v1 vs v2 — CPU, Memory, blkio Enforcement

## 🟡 Intermediate — HIGH PRIORITY

### What it is in simple terms

**Control Groups (cgroups)** are the Linux kernel mechanism that **limits, accounts for, and isolates resource usage** of process groups. Where namespaces control what a process can *see*, cgroups control what a process can *use*. Without cgroups, a container could consume all CPU, memory, or disk I/O on a host, starving other workloads.

---

### cgroups Architecture

```
CGROUPS — RESOURCE CONTROL HIERARCHY
═══════════════════════════════════════════════════════════════

cgroups form a HIERARCHY (tree structure):
  /sys/fs/cgroup/
  └── system.slice/
      ├── docker-abc123.scope/       ← container A's cgroup
      │     memory.max = 512m
      │     cpu.max = 500000 100000  ← 500ms per 100ms period = 0.5 CPU
      │     memory.current           ← current usage (read-only)
      └── docker-def456.scope/       ← container B's cgroup
            memory.max = 1g
            cpu.max = 1000000 100000 ← 1000ms per 100ms period = 1 CPU

Each container gets its own cgroup directory.
Kernel enforces limits via this directory's settings.
Controllers: memory, cpu, blkio (I/O), pids, cpuset, net_cls

CGROUPS v1:
  Each controller is a separate hierarchy
  /sys/fs/cgroup/memory/  → memory controller
  /sys/fs/cgroup/cpu/     → CPU controller
  /sys/fs/cgroup/blkio/   → block I/O controller
  Downside: controllers are independent, hard to coordinate
  Downside: memory accounting inconsistencies
  Status: deprecated, still supported (Linux 4.x era)

CGROUPS v2 (unified):
  Single hierarchy for all controllers
  /sys/fs/cgroup/             ← single root
  All controllers under one tree
  Better memory accounting (includes page cache)
  Rootless container support (user cgroup delegation)
  Pressure stall information (PSI) for resource pressure metrics
  Status: default on modern distros (Ubuntu 22.04+, RHEL 9, Amazon Linux 2023)
  Kubernetes 1.25+: full cgroup v2 support
```

---

### How CPU Limits Work

```
CPU ENFORCEMENT — CFS (Completely Fair Scheduler)
═══════════════════════════════════════════════════════════════

CPU QUOTA MODEL:
  cpu.cfs_quota_us  = CPU time allowed per period (microseconds)
  cpu.cfs_period_us = period length (default 100,000 = 100ms)

  CPU limit = 0.5 → quota=50000, period=100000
  Meaning: this cgroup can use 50ms of CPU per 100ms window

THROTTLING (the dangerous part):
  If container uses 50ms CPU in first 30ms of window:
  → Remaining 70ms: container is THROTTLED (kernel pauses it)
  → Even if no other processes are running!
  → Container cannot use "idle" CPU beyond its quota
  → Results in: high latency spikes, slow responses

  This is why CPU limits cause latency even at low average usage.
  A Java GC pause consuming 500ms of CPU: uses quota for 1000ms
  → All requests pause for 1 second

CPU REQUEST (not a limit):
  cpu.shares in v1 / cpu.weight in v2
  Used for SCHEDULING WEIGHT when CPUs are contended
  No enforcement when CPUs are idle — can use any available CPU
  Request = 250m (0.25 CPU) means: get at least 25% of a CPU when contested

KUBERNETES CPU MODEL:
  requests.cpu = cpu.shares/weight (scheduling, not a cap)
  limits.cpu   = cpu.cfs_quota (hard cap, causes throttling)

  No limits.cpu = container can use any available CPU
  This is sometimes preferred for latency-sensitive apps:
    → No throttling
    → Risk: one container can monopolize CPU
    → Mitigated by: requests (guarantee minimum) + node anti-affinity

cgroups v2 CPU example:
  cat /sys/fs/cgroup/kubepods.slice/.../cpu.max
  # 500000 100000   ← 0.5 CPU limit
  cat /sys/fs/cgroup/kubepods.slice/.../cpu.weight
  # 256              ← proportional weight (from requests.cpu)
```

---

### How Memory Limits Work

```
MEMORY ENFORCEMENT — OOM KILLER
═══════════════════════════════════════════════════════════════

MEMORY LIMIT:
  memory.limit_in_bytes (v1) / memory.max (v2)
  Hard cap: container cannot allocate beyond this

WHAT HAPPENS AT LIMIT:
  Container tries to allocate more memory than limit:
  Kernel invokes OOM (Out Of Memory) killer
  OOM killer sends SIGKILL (signal 9) to the biggest process in cgroup
  Container process killed → exit code 137
  In Kubernetes: pod shows "OOMKilled" in kubectl describe

MEMORY vs SWAP:
  memory.memsw.limit (v1) = RAM + swap combined
  memory.swap.max (v2) = swap limit
  Docker default: memory-swap = 2x memory limit
  Kubernetes: swap disabled by default on nodes (now optional in K8s 1.28+)

MEMORY REQUEST vs LIMIT:
  requests.memory = memory.soft_limit (v1) / memory.high (v2)
    Reservation: scheduler uses this to place pod
    Soft limit: kernel can exceed this if memory available
    NOT a hard cap
  limits.memory = memory.limit_in_bytes (v1) / memory.max (v2)
    Hard cap: OOMKill if exceeded

PAGE CACHE COUNTING (v1 vs v2 difference!):
  cgroups v1: page cache NOT counted toward container memory
              Container using 500MB RAM + 300MB page cache = reports 500MB
  cgroups v2: page cache IS counted toward container memory
              Container using 500MB RAM + 300MB page cache = reports 800MB!
  GOTCHA: After migrating to v2, containers may appear to use more memory
          and get OOMKilled that never were before. Increase limits!

MEMORY PRESSURE HIERARCHY:
  memory.low (v2):   protected from global reclaim below this threshold
  memory.high (v2):  throttle allocations above this, force reclaim
  memory.max (v2):   hard limit, OOMKill if exceeded
```

---

### How blkio / I/O Limits Work

```
BLOCK I/O THROTTLING
═══════════════════════════════════════════════════════════════

blkio controller (v1) / io controller (v2):
  Limits: bytes/second and IOPS per block device

  # Limit container to 10MB/s read on /dev/sda (device 8:0)
  echo "8:0 10485760" > /sys/fs/cgroup/.../blkio.throttle.read_bps_device

  Docker syntax:
  docker run \
    --device-read-bps /dev/sda:10mb \
    --device-write-bps /dev/sda:10mb \
    --device-read-iops /dev/sda:1000 \
    --device-write-iops /dev/sda:1000 \
    myapp

  Kubernetes: no direct blkio limits in pod spec (as of K8s 1.28)
  Use: io.kubernetes.container.blkio-weight annotation (limited support)
  Better approach: Karpenter node pools with storage-optimized instances

PID LIMIT:
  pids.max in cgroups
  Prevents fork bombs
  docker run --pids-limit=100 myapp
  Kubernetes: Pod spec pid limit via pod.spec.os.name or node config
```

---

```bash
# ─── INSPECT CONTAINER CGROUP ─────────────────────────────────────
# Find container PID
CONTAINER_PID=$(docker inspect my-container --format='{{.State.Pid}}')

# Find cgroup path
cat /proc/$CONTAINER_PID/cgroup
# 0::/system.slice/docker-abc123.scope    ← cgroups v2 path

# Read limits (cgroups v2)
CGROUP_PATH="/sys/fs/cgroup/system.slice/docker-abc123.scope"
cat $CGROUP_PATH/memory.max          # memory limit (bytes or "max")
cat $CGROUP_PATH/cpu.max             # "quota period" or "max period"
cat $CGROUP_PATH/pids.max            # PID limit

# Read current usage (cgroups v2)
cat $CGROUP_PATH/memory.current      # current memory usage in bytes
cat $CGROUP_PATH/cpu.stat            # CPU usage stats
cat $CGROUP_PATH/memory.stat         # detailed memory breakdown

# Verify cgroups version on host
stat -fc %T /sys/fs/cgroup/
# cgroup2fs  ← v2
# tmpfs      ← v1

# Docker stats (reads from cgroups)
docker stats my-container
# CONTAINER ID   NAME    CPU %   MEM USAGE / LIMIT   MEM %   NET I/O   BLOCK I/O
# abc123def456   myapp   15.3%   245MiB / 512MiB     47.9%   1.2MB / 0.8MB  0B / 0B

# In Kubernetes: kubectl top reads from metrics-server which reads from cgroups
kubectl top pod my-pod --containers
```

---

### 🎤 Short Crisp Interview Answer

> *"cgroups are the Linux kernel mechanism that enforces resource limits on container processes. The memory controller uses a hard cap — when a container exceeds its memory.max, the OOM killer sends SIGKILL, which is why you see exit code 137 and OOMKilled in Kubernetes. The CPU controller uses CFS quota — a container with 0.5 CPU limit gets 50ms of CPU time per 100ms window, and if it uses that up early in the window, it's throttled (paused) for the rest even if other CPUs are idle. This is why CPU limits cause latency spikes but not OOMKills. The critical v1 vs v2 difference: cgroups v2 counts page cache toward the memory limit while v1 doesn't — containers migrated from v1 hosts to v2 hosts may suddenly get OOMKilled with the same limits because their apparent memory usage jumps when page cache is included."*

---

---

# 13.11 OCI Spec — Image Spec, Runtime Spec, CRI Relationship

## 🟡 Intermediate

### OCI Specifications

```
OCI (OPEN CONTAINER INITIATIVE) SPECIFICATIONS
═══════════════════════════════════════════════════════════════

OCI was created in 2015 by Docker, CoreOS, and others.
Goal: prevent vendor lock-in, allow any image on any runtime.
Two main specs:

1. OCI IMAGE SPEC (image-spec):
   Defines: what an image IS
   Contents: manifest JSON, config JSON, layer tar.gz blobs
   Any OCI-compliant tool can build images: Docker, Buildah, Kaniko, ko
   Any OCI-compliant registry can store them: ECR, GCR, Docker Hub, Harbor
   Any OCI-compliant runtime can run them: Docker, containerd, CRI-O, Podman

2. OCI RUNTIME SPEC (runtime-spec):
   Defines: what a runtime MUST do to run a container
   Contents: config.json (capabilities, mounts, namespaces, cgroups, process)
   The "OCI bundle": config.json + rootfs directory
   runc is the reference implementation of the runtime spec
   Others: crun (C, faster), youki (Rust), gVisor runsc, Kata qemu-based

CRI (CONTAINER RUNTIME INTERFACE) — Kubernetes specific:
   NOT an OCI spec. Defined by Kubernetes.
   Protocol: gRPC API between kubelet and container runtime
   Methods: RunPodSandbox, CreateContainer, StartContainer, StopContainer, etc.
   Implementations: containerd (with CRI plugin), CRI-O
   Bridge: CRI is the kubelet-facing API; OCI is the container-facing standard

FULL STACK WITH SPECS:
  kubectl → API Server → kubelet
                            │
                            │ CRI (gRPC)
                            ▼
                        containerd
                            │
                            │ OCI runtime-spec (create bundle)
                            ▼
                          runc  (reads config.json, calls clone()+exec())
                            │
                            ▼
                        container process
```

---

### OCI Bundle Structure

```bash
# What runc receives to create a container (OCI bundle):
ls /run/containerd/io.containerd.runtime.v2.task/moby/<container-id>/
# bundle/
#   config.json    ← OCI runtime spec
#   rootfs/        ← merged filesystem (OverlayFS mount point)

cat config.json | python3 -m json.tool | head -80
# {
#   "ociVersion": "1.0.2",
#   "process": {
#     "terminal": false,
#     "user": {"uid": 1000, "gid": 1000},
#     "args": ["node", "server.js"],     ← CMD from Dockerfile
#     "env": ["PATH=...", "NODE_ENV=production"],
#     "cwd": "/app",                     ← WORKDIR from Dockerfile
#     "capabilities": {
#       "bounding":  ["CAP_CHOWN", "CAP_NET_BIND_SERVICE", ...],
#       "effective": [...],
#       "permitted": [...]
#     },
#     "rlimits": [{"type": "RLIMIT_NOFILE", "hard": 1024, "soft": 1024}],
#     "noNewPrivileges": true             ← from securityContext.allowPrivilegeEscalation: false
#   },
#   "root": {"path": "rootfs", "readonly": false},
#   "mounts": [
#     {"destination": "/proc", "type": "proc", "source": "proc"},
#     {"destination": "/dev",  "type": "tmpfs", ...},
#     {"destination": "/etc/config", "type": "bind", ...}  ← ConfigMap mount
#   ],
#   "linux": {
#     "namespaces": [
#       {"type": "pid"},
#       {"type": "network", "path": "/var/run/netns/cni-abc123"},  ← shared netns
#       {"type": "ipc"},
#       {"type": "uts"},
#       {"type": "mount"}
#     ],
#     "cgroupsPath": "kubepods/besteffort/pod-abc123/container-xyz",
#     "seccomp": { ... }   ← seccomp profile
#   }
# }
```

---

# 13.12 containerd & runc — The Actual Container Start Path

## 🟡 Intermediate

### The Complete Container Start Sequence

```
WHAT HAPPENS WHEN: kubectl apply → pod running
═══════════════════════════════════════════════════════════════

1. kubectl apply -f pod.yaml
   → API Server stores pod spec in etcd

2. Scheduler watches for unscheduled pods
   → Assigns pod to worker-2
   → Writes node name to pod spec in etcd

3. kubelet on worker-2 watches for assigned pods
   → kubelet calls containerd via CRI gRPC: RunPodSandbox()
   → containerd creates pause container (network namespace holder)
   → containerd calls CNI plugin to configure pod network (IP assignment)

4. kubelet calls: CreateContainer() + StartContainer() for each app container
   → containerd pulls image if not cached:
      a. Fetches image manifest from registry
      b. Checks which layers are already cached (by digest)
      c. Downloads missing layers (parallel)
      d. Verifies sha256 of each layer

5. containerd prepares OCI bundle:
   a. Creates OverlayFS mount (layers as lowerdir, writable as upperdir)
   b. Generates config.json from pod spec:
      - namespaces from pod (joins pause container's net+ipc namespace)
      - capabilities from securityContext
      - cgroup path from pod resource limits
      - mounts for volumes, ConfigMaps, Secrets, ServiceAccount token
      - environment variables
      - command from CMD/ENTRYPOINT

6. containerd calls runc: runc create <container-id> <bundle-path>
   runc reads config.json:
   a. Calls clone() with namespace flags:
      CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC
      (NOT CLONE_NEWNET — joins pause container's net namespace instead)
   b. Configures cgroups: writes to /sys/fs/cgroup/.../memory.max etc.
   c. Sets up mounts: proc, sys, dev, bind mounts for volumes
   d. Drops capabilities not in allowlist
   e. Applies seccomp profile (syscall filter)
   f. Sets up AppArmor profile if configured
   g. Calls exec() with container command (node, nginx, python, etc.)

7. runc exits (not a daemon)
   containerd monitors the process (via /proc/<pid> polling)
   kubelet polls containerd for container status

8. Container process runs as PID 1 in its namespaces
   kubelet reports Ready once probes pass
```

---

```bash
# ─── CONTAINERD CLI (ctr) ─────────────────────────────────────────
# containerd's own CLI (not docker, not crictl)
sudo ctr version
sudo ctr namespaces ls
# NAME    LABELS
# moby           ← Docker's containerd namespace
# k8s.io         ← Kubernetes/containerd namespace

# List containers in kubernetes namespace
sudo ctr --namespace k8s.io containers ls
sudo ctr --namespace k8s.io images ls | head -5

# ─── CONTAINERD SOCKET DIRECTLY ───────────────────────────────────
# containerd exposes gRPC at /run/containerd/containerd.sock
# containerd-shim-runc-v2 is the process between containerd and container:
ps aux | grep containerd
# /usr/bin/containerd               ← main daemon
# containerd-shim-runc-v2 -id abc   ← one shim per container
# (shim survives containerd restart — containers keep running)

# containerd shim architecture:
# containerd → shim (stays alive) → runc (exits after create) → container

# ─── RUNC DIRECTLY ────────────────────────────────────────────────
# runc is the OCI runtime — create/run containers from OCI bundles
runc --version
# runc version 1.1.9

# What runc does per container:
strace -f runc create ... 2>&1 | grep -E "clone|unshare|setns|mount" | head -20
# clone(... CLONE_NEWPID|CLONE_NEWNS|CLONE_NEWUTS|CLONE_NEWIPC ...) ← namespaces
# mount("overlay", "/run/containerd/.../rootfs", "overlay", ...)    ← OverlayFS
# write("/sys/fs/cgroup/.../memory.max", "536870912\n", ...)        ← 512MB limit
# execve("/app/node", ["node", "server.js"], [...])                  ← app starts

# Alternative runtimes (all OCI-compliant, drop-in runc replacements):
# crun:    C implementation, faster startup (~50% less overhead)
# youki:   Rust implementation
# gVisor:  Google's sandbox runtime (intercepts syscalls via ptrace/KVM)
#          Provides stronger isolation: app → gVisor kernel → host kernel
# Kata:    Runs container in a lightweight VM (QEMU/Firecracker)
#          Full VM isolation with container UX

# In Kubernetes: specify runtime class to use non-default runtime
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc           # gVisor runtime handler
---
spec:
  runtimeClassName: gvisor  # pod uses gVisor instead of runc
```

---

# ⚠️ 13.13 Multi-Stage Builds — Smaller, More Secure Images

## 🟡 Intermediate — HIGH PRIORITY

### What it is and why it matters

```
THE PROBLEM WITHOUT MULTI-STAGE BUILDS
═══════════════════════════════════════════════════════════════

Single-stage build (naive approach):
  FROM golang:1.21                   # 800MB base image
  WORKDIR /app
  COPY . .
  RUN go build -o server .           # installs compiler, etc.
  CMD ["./server"]

  Final image: 800MB (Go compiler, build tools, source code, binary)
  Problems:
    - Huge image (slow pull, more storage)
    - Attack surface: compiler, source code, build tools in production
    - If source is compromised and in image: secrets in build steps visible
    - go compiler has no place in production — pure waste

MULTI-STAGE BUILD SOLUTION:
  Stage 1 (builder):  uses large build image, compiles binary
  Stage 2 (final):    uses tiny runtime image, copies only the binary
  Result: production image contains ONLY what's needed to run

  Final image: 10MB (just the static binary + scratch base)
```

---

### Multi-Stage Build Examples

```dockerfile
# ═══════════════════════════════════════════════════
# EXAMPLE 1: Go binary (smallest possible result)
# ═══════════════════════════════════════════════════
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s" \              # strip debug symbols (smaller binary)
    -o server .

FROM scratch AS final               # empty base — nothing else
COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
USER 65534:65534                    # nobody user (scratch has no /etc/passwd)
EXPOSE 8080
CMD ["/server"]
# Final image: ~8MB
# Contains: only the compiled binary + CA certs
# No shell, no OS, no attack surface

# ═══════════════════════════════════════════════════
# EXAMPLE 2: Node.js app
# ═══════════════════════════════════════════════════
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production        # production deps only

FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci                          # includes devDeps for build
COPY . .
RUN npm run build                   # TypeScript compile, webpack, etc.

FROM node:20-alpine AS final
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY package.json ./
USER app
EXPOSE 3000
CMD ["node", "dist/server.js"]
# Final image: ~150MB
# Contains: node runtime + production deps + compiled output
# No: TypeScript compiler, webpack, devDependencies, source .ts files

# ═══════════════════════════════════════════════════
# EXAMPLE 3: Java Spring Boot with GraalVM native
# ═══════════════════════════════════════════════════
FROM ghcr.io/graalvm/native-image:21 AS builder
WORKDIR /app
COPY . .
RUN ./mvnw -Pnative native:compile -DskipTests

FROM ubuntu:22.04 AS final          # need glibc for native image
RUN addgroup --system app && adduser --system --ingroup app app
COPY --from=builder /app/target/myapp /myapp
USER app
EXPOSE 8080
ENTRYPOINT ["/myapp"]
# Final image: ~80MB (vs 500MB+ JVM image)
# Startup: ~50ms (vs 5-10s JVM cold start)

# ═══════════════════════════════════════════════════
# EXAMPLE 4: Reusing test stage in CI
# ═══════════════════════════════════════════════════
FROM node:20-alpine AS base
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM base AS test
COPY . .
RUN npm test                        # fails build if tests fail

FROM base AS build
COPY . .
RUN npm run build

FROM node:20-alpine AS final
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY package*.json ./
RUN npm ci --only=production
CMD ["node", "dist/server.js"]

# CI: docker build --target test .      ← run only tests
# CD: docker build --target final .     ← build production image
```

---

### Build Commands for Multi-Stage

```bash
# Build final stage (default — last FROM)
docker build -t myapp:v1 .

# Build only up to a specific stage
docker build --target builder -t myapp:builder .
docker build --target test -t myapp:test .

# BuildKit parallel stage building
DOCKER_BUILDKIT=1 docker build -t myapp:v1 .
# BuildKit executes independent stages in PARALLEL
# builder stage and deps stage (if independent) build simultaneously

# Pass build args into stages
docker build \
  --build-arg VERSION=1.2.3 \
  --build-arg GIT_SHA=$(git rev-parse HEAD) \
  -t myapp:v1.2.3 .

# Use external image as named stage
COPY --from=nginx:1.25 /etc/nginx/nginx.conf /etc/nginx/nginx.conf
# Copy a file from any image (not just a stage in this Dockerfile)

# Stage naming patterns
FROM golang:1.21 AS builder          # descriptive name
FROM scratch AS final                # clear intent
FROM alpine:3.19 AS certs            # utility stage for copying artifacts
```

---

### Size Comparison

```
IMAGE SIZE COMPARISON: single-stage vs multi-stage
═══════════════════════════════════════════════════════════════
Language    Single-stage    Multi-stage (optimized)
─────────────────────────────────────────────────────────────
Go          800MB           8MB (scratch)
             ↑ includes compiler, go stdlib source
Node.js     900MB           150MB (alpine + prod deps only)
             ↑ includes npm, devDependencies, source .ts
Java (JVM)  650MB           350MB (JRE only, no JDK)
Java Native 800MB           80MB  (binary only)
Python      900MB           200MB (slim + deps only)

VULNERABILITY SURFACE:
  Single-stage Go:     1,200+ packages (build toolchain)
  Multi-stage Go:      0 packages (scratch base has nothing)
  Fewer packages = fewer CVEs = less scanning noise
```

---

### 🎤 Short Crisp Interview Answer

> *"Multi-stage builds solve the problem of build tools ending up in production images. The pattern: a builder stage uses a large image with compilers and build tools, compiles the artifact, and a final stage copies only the binary or compiled output into a minimal base like Alpine or scratch. A Go binary built this way goes from 800MB (with the Go toolchain) to 8MB (just the static binary on scratch). Beyond size, it's a security win — the final image contains no compiler, no source code, no devDependencies, dramatically reducing the attack surface and CVE count from vulnerability scanning. In Node.js I use three stages: deps (production npm install), build (full npm install + TypeScript compile), and final (copies node_modules from deps and dist from build)."*

---

---

# 13.14 Image Security — Vulnerability Scanning, Rootless, Distroless

## 🔴 Advanced

### Vulnerability Scanning

```bash
# ─── TRIVY (most popular, CNCF) ──────────────────────────────────
# Scan local image
trivy image myapp:v1.2.3
# 2024-01-15T10:00:00Z INFO Vulnerability scanning is enabled
# myapp:v1.2.3 (debian 12.2)
# ════════════════════════════════
# Total: 45 (HIGH: 12, CRITICAL: 2, MEDIUM: 20, LOW: 11)
#
# ┌──────────────────┬──────────────┬──────────────┬─────────────────────────┐
# │ Library          │ Vulnerability│ Severity     │ Fixed Version           │
# ├──────────────────┼──────────────┼──────────────┼─────────────────────────┤
# │ openssl          │ CVE-2023-XXXX│ CRITICAL     │ 3.0.9                   │
# │ libssl3          │ CVE-2023-XXXX│ HIGH         │ 3.0.9                   │

# Scan only HIGH and CRITICAL
trivy image --severity HIGH,CRITICAL myapp:v1.2.3

# Fail CI build if CRITICAL found
trivy image --exit-code 1 --severity CRITICAL myapp:v1.2.3
# Returns exit 1 if any CRITICAL CVE found → CI pipeline fails

# Scan filesystem (for scanning before build)
trivy fs .

# Scan Dockerfile for misconfigurations
trivy config ./Dockerfile
trivy config ./k8s-manifests/

# Output formats for CI integration
trivy image --format json -o report.json myapp:v1.2.3
trivy image --format sarif -o trivy.sarif myapp:v1.2.3  # GitHub security tab

# ─── GRYPE (anchore) ─────────────────────────────────────────────
grype myapp:v1.2.3
grype --fail-on critical myapp:v1.2.3

# ─── ECR BUILT-IN SCANNING ────────────────────────────────────────
# Enable scan on push
aws ecr put-image-scanning-configuration \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true

# Get scan results after push
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=v1.2.3 \
  --query 'imageScanFindings.findings[?severity==`CRITICAL`]'

# ─── CI/CD INTEGRATION PATTERN ────────────────────────────────────
# GitHub Actions:
# - name: Scan image with Trivy
#   uses: aquasecurity/trivy-action@master
#   with:
#     image-ref: '${{ env.ECR_REGISTRY }}/myapp:${{ github.sha }}'
#     format: 'sarif'
#     output: 'trivy-results.sarif'
#     exit-code: '1'
#     severity: 'CRITICAL,HIGH'
```

---

### Distroless Images

```dockerfile
# DISTROLESS: Google's minimal images with no shell, no package manager
# Dramatically reduces attack surface
# Available for: Java, Node, Python, Go, .NET, static binaries

# Regular alpine-based Node:
FROM node:20-alpine
# Contains: sh, ash, apk, wget, curl, etc. (100+ packages)

# Distroless Node:
FROM gcr.io/distroless/nodejs20-debian12
# Contains: node runtime only
# No shell, no package manager, no curl, no wget
# CVE count: ~90% reduction vs full base images

# EXAMPLE: Node.js with distroless final stage
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY dist/ ./dist/

FROM gcr.io/distroless/nodejs20-debian12 AS final
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
USER nonroot
EXPOSE 3000
CMD ["dist/server.js"]   # distroless node: no "node" prefix needed

# DEBUGGING DISTROLESS (no shell available):
# Use ephemeral debug containers:
kubectl debug -it my-pod \
  --image=gcr.io/distroless/nodejs20-debian12:debug \
  --target=app
# :debug tag includes busybox shell for emergency debugging

# Distroless variants:
# gcr.io/distroless/static-debian12         → statically compiled binaries only
# gcr.io/distroless/base-debian12           → glibc + openssl (for dynamic C)
# gcr.io/distroless/java21-debian12         → Java 21 JRE
# gcr.io/distroless/nodejs20-debian12       → Node 20
# gcr.io/distroless/python3-debian12        → Python 3
```

---

### Rootless Docker

```bash
# ROOTLESS DOCKER: run Docker daemon as non-root user
# Normal Docker: dockerd runs as root → any escape = root on host
# Rootless Docker: dockerd runs as your user → escape = your user only

# Install rootless Docker
dockerd-rootless-setuptool.sh install
# Creates: ~/.config/docker/daemon.json
#          ~/.local/share/docker/ (storage)
#          ~/bin/dockerd (user daemon)

# User namespaces: container root → host non-root
# UID 0 inside container → maps to your UID on host
docker run --rm alpine id
# uid=0(root) gid=0(root)     ← inside container, appears as root
# ps aux on HOST: see same process running as uid=1000 (your user)

# Rootless limitations:
# - Can't bind ports < 1024 (host ports require CAP_NET_BIND_SERVICE)
# - Network performance slightly worse (slirp4netns user-space networking)
# - Some storage drivers not supported

# Check if rootless
docker info | grep -i rootless
# rootless: true

# ROOTLESS IN KUBERNETES (Podman, not Docker):
# Podman is daemonless and rootless by default
# Used in some K8s environments as drop-in docker replacement
podman run --rm alpine id
# Same container semantics, no daemon, no root required
```

---

# 13.15 BuildKit — Parallel Builds, Cache Mounts, Secrets in Build

## 🔴 Advanced

```dockerfile
# ─── ENABLE BUILDKIT ──────────────────────────────────────────────
# Docker 23.0+: BuildKit is default
# Older Docker: DOCKER_BUILDKIT=1 docker build .
# Or: set in daemon.json: {"features": {"buildkit": true}}

# ─── PARALLEL STAGES ──────────────────────────────────────────────
# BuildKit analyzes the DAG (directed acyclic graph) of stages
# Independent stages build in PARALLEL

FROM node:20 AS deps
RUN npm ci                          #  ┐ These two stages
                                    #  │ have no dependency
FROM golang:1.21 AS api-builder     #  ├ on each other
RUN go build .                      #  ┘ → BuildKit builds them PARALLEL

FROM ubuntu:22.04 AS final
COPY --from=deps /app/node_modules ./   # depends on deps
COPY --from=api-builder /app/server ./  # depends on api-builder
# Both must complete before final stage runs

# ─── CACHE MOUNTS (build cache that persists across builds) ────────
# RUN --mount=type=cache,target=<path>
# The cache directory persists between docker build invocations
# Not included in the final image (build-time only)

# Python pip cache mount (no re-downloading packages between builds)
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Node npm cache mount
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Go module cache mount
RUN --mount=type=cache,target=/go/pkg/mod \
    go build ./...

# Apt package cache mount
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y curl git

# RESULT: Second build with same dependencies: near-instant (cache hit)
# First build: normal time (downloads packages)
# Second build with source change: deps from cache, only compile step runs

# ─── SECRET MOUNTS (build-time secrets, NOT in image) ─────────────
# Problem: pip install from private repo needs token
# Wrong:   ARG PYPI_TOKEN=xxx → stored in image layer!
# Right:   --mount=type=secret → never stored in any layer

# Build command:
# docker build \
#   --secret id=pypi_token,env=PYPI_TOKEN \
#   -t myapp:v1 .

# Dockerfile:
RUN --mount=type=secret,id=pypi_token \
    pip install \
      --extra-index-url \
      "https://$(cat /run/secrets/pypi_token)@pypi.company.com/simple/" \
      -r requirements.txt
# /run/secrets/pypi_token exists ONLY during this RUN instruction
# Not in any layer, not in final image, not in build history

# SSH mount (for private git repos during build):
# docker build --ssh default -t myapp:v1 .
RUN --mount=type=ssh \
    git clone git@github.com:company/private-lib.git

# ─── BIND MOUNTS (read-only source in RUN) ─────────────────────────
# Mount files into RUN without COPYing them (no layer created)
RUN --mount=type=bind,source=.,target=/src,readonly \
    cd /src && go vet ./...
# Source mounted read-only for linting — not copied into image

# ─── DOCKERFILE SYNTAX DIRECTIVE ──────────────────────────────────
# Pin BuildKit frontend version for reproducible builds
# syntax=docker/dockerfile:1.6
FROM node:20-alpine
# Uses specific Dockerfile syntax parser (from Docker Hub)
# Enables latest features: heredocs, --chmod, etc.

# ─── HEREDOC SYNTAX (BuildKit 1.4+) ───────────────────────────────
COPY <<EOF /app/config.json
{
  "env": "production",
  "port": 8080
}
EOF

RUN <<-SCRIPT
  set -e
  echo "Running setup..."
  useradd -r appuser
  mkdir -p /app/data
  chown appuser:appuser /app/data
SCRIPT
```

---

# 13.16 Container Runtime Security — Capabilities, Seccomp, User Namespaces

## 🔴 Advanced

### Linux Capabilities

```
LINUX CAPABILITIES — FINE-GRAINED ROOT PRIVILEGES
═══════════════════════════════════════════════════════════════

Traditional Unix: privileged (root) or unprivileged
Capabilities: split root privileges into ~40 distinct capabilities
  Each can be granted/revoked independently

RELEVANT CAPABILITIES FOR CONTAINERS:
CAP_NET_BIND_SERVICE  bind ports below 1024 (e.g., port 80 without root)
CAP_NET_ADMIN         configure network interfaces, iptables (dangerous!)
CAP_SYS_ADMIN         mount filesystems, ptrace, many admin ops (very dangerous!)
CAP_CHOWN             change file ownership (most apps don't need this)
CAP_DAC_OVERRIDE      bypass file permission checks (dangerous)
CAP_SYS_PTRACE        attach to and debug other processes
CAP_KILL              send signals to any process
CAP_SETUID            change UID (enables privilege escalation!)
CAP_SETGID            change GID
CAP_NET_RAW           send raw packets (for ping, packet sniffing)
CAP_SYS_TIME          set system time (most apps don't need)

DEFAULT DOCKER CAPABILITIES (containers start with these):
  CAP_CHOWN, CAP_DAC_OVERRIDE, CAP_FSETID, CAP_FOWNER,
  CAP_MKNOD, CAP_NET_RAW, CAP_SETGID, CAP_SETUID,
  CAP_SETFCAP, CAP_SETPCAP, CAP_NET_BIND_SERVICE,
  CAP_SYS_CHROOT, CAP_KILL, CAP_AUDIT_WRITE

BEST PRACTICE: Drop ALL, add back only what's needed
```

---

```bash
# ─── MANAGING CAPABILITIES ────────────────────────────────────────
# Docker: drop all, add specific
docker run \
  --cap-drop=ALL \                  # drop all default capabilities
  --cap-add=NET_BIND_SERVICE \      # only add what's needed
  nginx:1.25

# Kubernetes equivalent (in pod spec):
# spec:
#   containers:
#   - name: app
#     securityContext:
#       capabilities:
#         drop: ["ALL"]
#         add: ["NET_BIND_SERVICE"]   # if binding port 80

# Check container's capabilities
docker run --rm alpine cat /proc/1/status | grep Cap
# CapInh: 0000000000000000
# CapPrm: 00000000a80425fb  ← permitted capabilities (hex bitmask)
# CapEff: 00000000a80425fb  ← effective capabilities
# CapBnd: 00000000a80425fb  ← bounding set

# Decode capability bitmask
capsh --decode=00000000a80425fb
# 0x00000000a80425fb=cap_chown,cap_dac_override,...

# Container with ALL dropped (most secure):
docker run --cap-drop=ALL --rm alpine cat /proc/1/status | grep Cap
# CapPrm: 0000000000000000   ← no capabilities!
```

---

### Seccomp — Syscall Filtering

```
SECCOMP (SECURE COMPUTING MODE)
═══════════════════════════════════════════════════════════════

seccomp filters which SYSTEM CALLS a process can make.
Processes use syscalls to request kernel services:
  open(), read(), write(), fork(), clone(), execve(), etc.

SECCOMP PROFILES:
  Unconfined:      all syscalls allowed (no seccomp)
  RuntimeDefault:  Docker/containerd default profile (blocks ~40 dangerous syscalls)
  Strict mode:     only read, write, exit allowed (too restrictive for most apps)
  Custom:          your profile for your specific app

DOCKER RUNTIME DEFAULT blocks these (examples):
  keyctl         (kernel keyring — privilege escalation risk)
  ptrace         (attach to other processes)
  reboot         (reboot the system)
  mount          (mount filesystems)
  create_module  (load kernel modules)
  init_module    (load kernel modules)
  delete_module  (remove kernel modules)
  kexec_load     (load new kernel)
  pivot_root     (change root filesystem)
  settimeofday   (change system time)

IN KUBERNETES:
  Pod Security Standard "Restricted" requires: seccompProfile: RuntimeDefault
  PSA enforces this with: pod-security.kubernetes.io/enforce: restricted
```

---

```bash
# ─── SECCOMP IN DOCKER ────────────────────────────────────────────
# Apply default seccomp profile (Docker does this automatically)
docker run --security-opt seccomp=default nginx

# Disable seccomp (dangerous, for debugging only)
docker run --security-opt seccomp=unconfined nginx

# Apply custom seccomp profile
docker run \
  --security-opt seccomp=/path/to/custom-profile.json \
  myapp

# Custom seccomp profile JSON structure:
cat custom-seccomp.json
# {
#   "defaultAction": "SCMP_ACT_ERRNO",     ← block by default
#   "architectures": ["SCMP_ARCH_X86_64"],
#   "syscalls": [
#     {
#       "names": ["read","write","open","close","stat","fstat","mmap",
#                 "mprotect","munmap","brk","futex","nanosleep","exit",
#                 "exit_group","gettid","getpid","clone","execve","wait4"],
#       "action": "SCMP_ACT_ALLOW"         ← allow these
#     }
#   ]
# }

# Find what syscalls your app actually uses (profile it)
strace -f -c myapp 2>&1
# Counts calls per syscall — use this to build minimal allowlist

# ─── SECCOMP IN KUBERNETES ────────────────────────────────────────
# Apply RuntimeDefault seccomp profile:
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault      # use containerd's default profile

# Apply custom profile (stored on each node at /var/lib/kubelet/seccomp/):
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/my-custom.json

# ─── COMPLETE SECURITY CONTEXT (all protections combined) ─────────
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false   # no setuid/setgid binaries
      readOnlyRootFilesystem: true       # immutable container fs
      capabilities:
        drop: ["ALL"]
        add: []                          # nothing added back
```

---

### User Namespaces in Containers

```bash
# USER NAMESPACE REMAPPING
# Maps container UID 0 → host UID 100000+ (non-root on host)
# Even if container process is root, on host it's a normal user

# Configure Docker daemon for userns-remap:
cat /etc/docker/daemon.json
# {"userns-remap": "default"}
sudo systemctl restart docker

# Verify: process appears as non-root on host
docker run -d --name myapp nginx
CONTAINER_PID=$(docker inspect myapp --format='{{.State.Pid}}')
ps aux | grep $CONTAINER_PID
# 100000  12345  nginx: master process  ← UID 100000, NOT root!

# Inside container: still sees root
docker exec myapp id
# uid=0(root) gid=0(root)   ← appears as root inside

# ID mapping
cat /proc/$CONTAINER_PID/uid_map
# 0      100000    65536
# ↑ container 0 → host 100000, 65536 IDs mapped

# LIMITATIONS of userns-remap:
# - Volumes owned by root on host are inaccessible (permission denied)
# - Some features disabled (e.g., --pid=host)
# - Performance overhead (UID translation in every file operation)

# ROOTLESS CONTAINERS (user namespace without daemon):
# Podman and rootless Docker run entirely as regular user
# No setuid binary, no privileged daemon, container root = your user on host
```

---

# Quick Reference — Category 13 Cheat Sheet

| Topic | Key Facts |
|-------|-----------|
| **Container vs VM** | Container = process + namespaces + cgroups. Shared host kernel. VM = separate kernel via hypervisor |
| **Namespaces** | 7 types: PID, NET, MNT, UTS, IPC, USER, CGROUP. Each isolates a different resource domain |
| **cgroups** | Enforce limits: memory.max → OOMKill (exit 137). cpu.max → CFS throttle (latency, no kill) |
| **cgroups v2 gotcha** | Counts page cache in memory usage. Containers may OOMKill after v1→v2 migration |
| **Image layers** | Read-only, content-addressed. OverlayFS presents unified view. Shared across containers |
| **Copy-on-write** | First write to file: copied to writable upper layer. Image layer untouched |
| **Delete in later layer** | Doesn't reduce size! Layer blob still exists. Fix: combine add+delete in ONE RUN |
| **Layer cache** | Invalidated from changed layer downward. Copy manifests before source code |
| **CMD exec form** | `["node","server.js"]` — node is PID 1. Shell form: sh is PID 1, SIGTERM not forwarded |
| **ENTRYPOINT + CMD** | ENTRYPOINT = fixed executable. CMD = default args. CMD overridable at runtime |
| **OCI image spec** | Manifest (lists layers) + Config (runtime metadata) + layer blobs (tar.gz) |
| **OCI runtime spec** | config.json + rootfs. runc reads this to create namespaces, configure cgroups |
| **Docker → K8s** | kubelet → containerd (CRI) → runc (OCI). dockerd not involved in K8s pod lifecycle |
| **Pause container** | Holds network+IPC namespaces for pod. All containers join it. Stable across restarts |
| **Multi-stage** | Builder stage = large image for compiling. Final stage = minimal image + artifact only |
| **Distroless** | No shell, no package manager. 90% fewer CVEs. Use :debug tag for ephemeral debugging |
| **docker.sock** | Root-equivalent host access. Never mount in containers. Use Kaniko/Buildah for builds |
| **CAP_DROP ALL** | Drop all default capabilities. Add back only what the app actually needs |
| **Seccomp RuntimeDefault** | Blocks ~40 dangerous syscalls. Required by K8s Restricted PSS profile |
| **Tags vs digests** | Tags are mutable. Digests (sha256:...) are immutable content addresses — use in prod |
| **ECR auth** | Token expires every 12h. `aws ecr get-login-password | docker login` to refresh |
| **BuildKit cache mount** | `--mount=type=cache` persists between builds, not in image. Huge build speed improvement |
| **BuildKit secret mount** | `--mount=type=secret` available only during RUN, never in image or history |

---

## Key Numbers to Remember

| Fact | Value |
|------|-------|
| Container startup time | 50–500ms |
| VM startup time | 10–60 seconds |
| Default Docker stop grace period | 10 seconds |
| OOMKill exit code | 137 (128 + signal 9) |
| SIGTERM exit code | 143 (128 + signal 15) |
| Default CPU CFS period | 100ms (100,000 microseconds) |
| Default capabilities count | ~14 default capabilities |
| RuntimeDefault seccomp blocks | ~40 dangerous syscalls |
| Distroless CVE reduction vs full base | ~90% |
| Alpine Linux base image size | ~5MB |
| ECR auth token expiry | 12 hours |
| Scratch base image size | 0 bytes |
| Docker Hub rate limit (unauthenticated) | 100 pulls / 6 hours per IP |
| OCI layer hash algorithm | SHA-256 |
