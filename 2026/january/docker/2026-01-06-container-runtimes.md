# Container Runtimes - The Construction Site Analogy

> Understanding container runtimes through a real-world analogy of architects, foremen, construction crews, and building codes.

---

## TL;DR

| Container Concept | Construction Site Analogy |
|-------------------|--------------------------|
| High-Level Runtime (containerd, CRI-O) | Architecture Firm / General Contractor |
| Shim (containerd-shim, conmon) | Site Foreman |
| Low-Level Runtime (runc, crun) | Construction Crew / Workers |
| Container | Building that gets constructed |
| OCI Runtime Spec | Building Codes & Blueprint Standards |
| OCI Image Spec | Warehouse Packaging Standards |
| Docker | Full-service construction company (design + build + manage) |
| containerd | Just the construction operations (no retail/design services) |
| CRI-O | Specialized contractor for one client (Kubernetes neighborhoods only) |
| Podman | DIY homeowner construction (no company needed) |
| config.json | Detailed blueprint with specifications |
| Dockerfile | Recipe for building materials and steps |
| Dockershim | Unnecessary middleman contractor (now removed) |
| runc | Standard construction crew |
| crun | Speed construction crew (faster tools) |
| runsc (gVisor) | Reinforced bunker construction (extra security) |
| Kata Containers | Separate isolated lot per building (VM-level isolation) |

---

## The Big Picture

Imagine you want to **build a house**. You don't just grab a hammer and start - you need:

1. **An architecture firm** that designs blueprints and manages the project
2. **A site foreman** who stays on-site to supervise and inspect
3. **Construction crews** who do the actual building work, then move to the next job
4. **Building codes** that ensure any crew can read any blueprint

Once the house is built, the **crew leaves** for the next project, but the **foreman stays** to monitor, maintain, and handle any issues. The house stands independently - you don't need the workers to remain for it to exist.

This is exactly how **container runtimes** work: a multi-layered system where each layer has a specific job, and the actual "building" (container creation) is done by workers (runc) who exit immediately after finishing.

---

## The Three-Layer Construction System

### Layer 1: Architecture Firm (High-Level Runtime)

**containerd, CRI-O, Docker Engine** are like **architecture firms** or general contractors:

- Receive construction orders from clients (Kubernetes, developers)
- Design blueprints (convert images to OCI bundles)
- Manage logistics: acquire building materials (pull images from registries)
- Set up utilities: arrange power lines (networking), water pipes (volumes)
- Coordinate multiple projects simultaneously
- **Long-running operation** - the office stays open 24/7

**What they DON'T do**: They never pick up a shovel. They don't create namespaces, mount filesystems, or execute processes directly.

```
Architecture Firm (containerd):
├── Blueprint Department (image management)
├── Logistics (registry pulls, storage)
├── Utilities Coordination (CNI networking, CSI storage)
├── Project Management (API server for client requests)
└── Subcontractor Dispatch (spawning foremen & crews)
```

---

### Layer 2: Site Foreman (Shim)

**containerd-shim, conmon** are the **site foremen**:

- Receive blueprints from the architecture firm
- Call the construction crew to build
- **Stay on-site after the crew leaves**
- Monitor the building 24/7
- Handle maintenance requests (logs, status checks)
- Report back to the architecture firm
- **One foreman per building** (one shim per container)

**Critical job**: If the architecture firm needs to close for renovations (containerd restarts), the foreman keeps watching all the buildings. Construction projects don't stop just because the main office closed!

```
Site Foreman (containerd-shim):
1. Receives blueprint from containerd
2. Calls runc: "Build according to these specs"
3. Watches runc work
4. When runc finishes and leaves → Foreman adopts the building
5. Monitors forever: "Is the building still standing? Any issues?"
6. Reports status to containerd when asked
```

---

### Layer 3: Construction Crew (Low-Level Runtime)

**runc, crun, runsc** are the **construction crews**:

- Receive detailed blueprints (config.json) from the foreman
- **Do the actual building work**:
  - Pour foundation (create namespaces: PID, NET, MNT, UTS, IPC)
  - Build frame (set up cgroups for resource limits)
  - Install utilities (mount filesystems, configure networking)
  - Apply security measures (seccomp filters, capabilities, SELinux)
  - Install the first resident (execute container process with execve)
- **Finish the job and leave immediately** for the next construction site
- **Short-lived** - they don't stay to maintain the building

**Key insight**: The building (container) doesn't need the crew to remain standing. Once constructed, it operates independently with the foreman supervising.

```
Construction Crew (runc) Actions:

Step 1: Read blueprint (config.json)
   "Build 2-bedroom house at 123 Main St
    Foundation: 6 separate zones (namespaces)
    Frame: 512MB memory, 1 CPU core (cgroups)
    Security: Reinforced doors (seccomp), alarm system (capabilities)"

Step 2: Build foundation
   unshare() syscalls → Create isolated zones:
   - PID zone: House has its own room numbering
   - NET zone: House has its own utility connections
   - MNT zone: House has its own internal walls
   - UTS zone: House has its own address sign
   - IPC zone: House has its own intercom system

Step 3: Build frame
   mkdir /sys/fs/cgroup/memory/my-house
   echo 536870912 > memory.limit_in_bytes
   echo $PID > cgroup.procs

Step 4: Install utilities
   mount -t overlay (combine material layers into one structure)
   pivot_root (make the house's interior the main living space)

Step 5: Install first resident
   execve("/bin/sh") → Replace crew with actual resident

Step 6: Crew exits, building stands!
   runc exits (PID disappears)
   Container process continues (adopted by foreman)
```

---

## Complete Construction Flow: docker run nginx

Let's watch a construction project from order to completion:

```
┌─────────────────────────────────────────────────────────┐
│  YOU: "I want an nginx house built"                     │
│  $ docker run nginx                                     │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  LAYER 1: Architecture Firm (dockerd + containerd)      │
│                                                         │
│  1. "Let me check our warehouse for nginx materials"   │
│     → Pull image layers from registry                  │
│                                                         │
│  2. "Prepare the building materials"                   │
│     → Unpack layers to storage                         │
│                                                         │
│  3. "Design detailed blueprint"                        │
│     → Create OCI bundle:                               │
│       - config.json (specifications)                   │
│       - rootfs/ (building materials)                   │
│                                                         │
│  4. "Arrange utilities"                                │
│     → Allocate IP address, configure network           │
│                                                         │
│  5. "Dispatch site foreman"                            │
│     → Spawn containerd-shim process                    │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  LAYER 2: Site Foreman (containerd-shim)               │
│                                                         │
│  1. "Received blueprints and site location"            │
│  2. "Calling construction crew..."                     │
│     → fork() and exec() runc                           │
│  3. "Crew is working... monitoring progress"           │
│  4. "Crew finished! Building is complete"              │
│  5. "Crew left for next job, I'll watch this building" │
│     → Adopt container process as parent                │
│  6. "Monitoring 24/7, reporting to HQ as needed"       │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  LAYER 3: Construction Crew (runc)                      │
│                                                         │
│  1. Read config.json blueprint                         │
│  2. Pour foundation:                                   │
│     clone(CLONE_NEWPID | CLONE_NEWNET | ...)          │
│  3. Build frame:                                       │
│     echo $PID > /sys/fs/cgroup/.../cgroup.procs        │
│  4. Mount materials:                                   │
│     mount -t overlay → Create merged structure         │
│  5. Install security:                                  │
│     prctl(PR_SET_SECCOMP, ...)                         │
│  6. Install first resident:                            │
│     execve("/docker-entrypoint.sh", ["nginx", ...])    │
│  7. Job done! Moving to next site...                   │
│     → runc process exits                               │
└─────────────────────────────────────────────────────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │  🏠 BUILDING STANDS   │
          │                      │
          │  nginx running       │
          │  PID 1 inside house  │
          │  Foreman supervising │
          └──────────────────────┘


Process Tree on Host:

systemd (PID 1)
└─ containerd (Architecture Firm, PID 1234)
   └─ containerd-shim (Foreman, PID 5678)
      └─ nginx (Building Resident, PID 9012)
         └─ nginx worker (PID 9013)

Notice: runc is NOT in the tree (crew left!)
```

---

## Building Codes: OCI Specifications

Just like construction follows **universal building codes** (electrical standards, plumbing codes, fire safety), containers follow **OCI (Open Container Initiative) specifications**.

### OCI Runtime Spec - The Building Codes

**Purpose**: Defines HOW to construct a building (container)

**What it standardizes:**

**1. Blueprint Format (config.json)**

```json
{
  "ociVersion": "1.0.2",
  "process": {
    "args": ["nginx", "-g", "daemon off;"],
    "env": ["PATH=/usr/local/sbin:/usr/local/bin"],
    "cwd": "/"
  },
  "root": {
    "path": "rootfs"  ← Building materials location
  },
  "linux": {
    "namespaces": [
      {"type": "pid"},      ← Foundation section 1
      {"type": "network"}   ← Foundation section 2
    ],
    "resources": {
      "memory": {"limit": 536870912},  ← Frame strength
      "cpu": {"shares": 1024}          ← Support beams
    }
  }
}
```

**2. Construction Site Layout (Filesystem Bundle)**

```
/construction-site/
├── config.json          ← Blueprint
└── rootfs/              ← Building materials warehouse
    ├── bin/
    ├── etc/
    ├── usr/
    └── ...
```

**3. Standard Construction Steps (Lifecycle Operations)**

```bash
# ANY crew that follows OCI building codes understands these commands:

runc create --bundle /site container-id   # Start construction
runc start container-id                   # Install first resident
runc kill container-id SIGTERM           # Evacuate building
runc delete container-id                 # Demolish building
```

**Why this matters**: You can **swap construction crews** without changing anything else:

```yaml
# Kubernetes RuntimeClass - Choose your crew!

apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: reinforced
handler: runsc  ← Switch from standard (runc) to bunker builders (gVisor)

---
apiVersion: v1
kind: Pod
spec:
  runtimeClassName: reinforced  ← This building needs extra security!
  containers:
  - name: app
    image: nginx

# Same blueprint (config.json), different crew!
# Architecture firm (containerd) doesn't change
# Foreman (shim) doesn't change
# Only the construction method changes
```

---

### OCI Image Spec - The Warehouse Packaging Standards

**Purpose**: Defines HOW building materials are packaged and shipped

**What it standardizes:**

**1. Material Layers (Like plywood sheets)**

```
Image layers (tar.gz compressed):
sha256:abc123... ← Foundation materials (base OS)
sha256:def456... ← Framing materials (nginx installation)
sha256:ghi789... ← Finishing materials (config files)
```

**2. Shipping Manifest (Inventory list)**

```json
{
  "schemaVersion": 2,
  "config": {
    "digest": "sha256:abc123...",
    "size": 7023
  },
  "layers": [
    {
      "digest": "sha256:layer1...",
      "size": 32654
    },
    {
      "digest": "sha256:layer2...",
      "size": 16724
    }
  ]
}
```

**3. Material Specifications**

```json
{
  "architecture": "amd64",      ← Materials for x86 buildings
  "os": "linux",                ← Linux-style construction
  "config": {
    "Cmd": ["nginx"],           ← Default resident
    "WorkingDir": "/app",       ← Main room
    "ExposedPorts": {"80/tcp": {}}  ← Windows/doors
  }
}
```

---

## The Dockershim Story: Removing the Middleman

### The Old Way (Too Many Contractors!)

```
┌──────────────────────────────────────────────┐
│  Kubernetes: "I need a building"             │
└───────────────┬──────────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────────┐
│  Dockershim: "Let me translate to Docker"    │
│  (Built into Kubernetes, extra translator)   │
└───────────────┬──────────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────────┐
│  Docker: "Got it, passing to my team"        │
│  (Another layer, more overhead)              │
└───────────────┬──────────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────────┐
│  containerd: "Okay, dispatching foreman"     │
│  (Finally, the actual construction starts)   │
└───────────────┬──────────────────────────────┘
                │
                ▼
         Foreman → Crew → Building

Problem: 5 layers just to build! 😱
```

### The New Way (Direct Construction)

```
┌──────────────────────────────────────────────┐
│  Kubernetes: "I need a building"             │
└───────────────┬──────────────────────────────┘
                │ CRI (direct communication)
                ▼
┌──────────────────────────────────────────────┐
│  containerd: "Dispatching foreman"           │
└───────────────┬──────────────────────────────┘
                │
                ▼
         Foreman → Crew → Building

Result: 3 layers! Much cleaner! ✅
```

**What happened**: Kubernetes learned to speak directly to containerd (using CRI - Container Runtime Interface). No need for dockershim middleman or Docker wrapper.

**Did buildings stop working?** NO! 
- Same containerd doing the actual work
- Same blueprints (OCI specs)
- Same construction crews (runc)
- Same buildings (containers)
- Just removed unnecessary paperwork!

**Timeline**:
- Kubernetes 1.20 (Dec 2020): "We're removing dockershim next year"
- Kubernetes 1.24 (May 2022): Dockershim removed
- All buildings still standing! No disruption!

---

## Different Construction Companies

### Docker - Full-Service Construction Company

**Docker is the complete package**:

```
Docker Inc. Services:
├── Retail Stores (Docker Desktop GUI)
├── Design Department (docker build)
├── Project Management (docker-compose)
├── Construction Operations (dockerd → containerd)
├── Customer Service (docker CLI)
└── Construction Crews (runc)
```

**When you use Docker**:
- Beautiful storefront (Docker Desktop)
- Easy commands: `docker run`, `docker build`, `docker-compose up`
- Automatic networking setup (docker0 bridge)
- Volume management
- Everything integrated

**Trade-off**: Heavier (250MB binary), more components, runs a daemon (dockerd + containerd)

---

### containerd - Just the Construction Operations

**containerd is like hiring just the construction manager**:

```
containerd Services:
├── Project Management (API, lifecycle)
├── Material Logistics (image pulling, storage)
├── Foreman Coordination (spawning shims)
└── Crew Dispatch (calling runc)

NOT included:
❌ Retail stores (no GUI)
❌ Design services (no build tools)
❌ Advanced project management (no compose)
```

**When you use containerd standalone**:
- Lightweight (50MB binary)
- Production Kubernetes default
- CLI tools: `ctr` (basic), `nerdctl` (Docker-compatible)
- 70%+ of production Kubernetes clusters

**Trade-off**: Less user-friendly, need separate tools for building images

---

### CRI-O - Specialized Contractor for Kubernetes

**CRI-O only builds for one client: Kubernetes**

```
CRI-O Philosophy:
"We ONLY build in Kubernetes neighborhoods.
 No retail customers.
 No side projects.
 Just Kubernetes. That's it."
```

**What makes it special**:
- Smaller codebase (50K lines vs 150K for containerd)
- Version aligned with Kubernetes (CRI-O 1.28 = K8s 1.28)
- Zero features beyond what K8s needs
- CRI-only interface (no other APIs)

**When to use CRI-O**:
- Red Hat OpenShift (default)
- RHEL environments
- Security-focused (smaller attack surface)
- Pure Kubernetes (no other tools needed)

**Trade-off**: Can't use for anything except Kubernetes

---

### Podman - DIY Homeowner Construction

**Podman is like being your own general contractor**:

```
Traditional (Docker):
You → Call construction company (dockerd daemon)
    → Company dispatches foreman
    → Foreman calls crew
    → Building gets built
    → Company keeps office open 24/7 (daemon overhead)

Podman (DIY):
You → Directly call foreman (fork conmon)
    → Foreman calls crew
    → Building gets built
    → No company overhead needed!
```

**Key difference: No Daemon!**

```bash
# Docker (daemon required)
$ ps aux | grep dockerd
root  1234  dockerd  ← Always running in background

$ docker run nginx
# dockerd processes the request

# Podman (no daemon)
$ ps aux | grep podman
# (nothing running!)

$ podman run nginx
# podman forks, spawns conmon, exits immediately
# No background process!
```

**Rootless Construction (Podman's Superpower)**:

```
Traditional (Docker):
├── Construction company runs as ROOT
├── Can build anywhere, access anything
└── If hacked → Entire property at risk! 💀

Podman (Rootless):
├── YOU are the builder (your user account)
├── Can only build on your property
├── If hacked → Only your stuff at risk, not entire system ✅
└── Uses "user namespace magic": Building thinks it's owned by root,
    but host sees it as owned by you!
```

**When to use Podman**:
- Security-first environments (no root daemon)
- Multi-user systems (each user builds independently)
- Development (Docker-compatible commands)
- Systemd services (user services without root)

**Trade-off**: Can't bind privileged ports (<1024) without root

---

## Different Construction Crews (Runtime Alternatives)

All follow the same building codes (OCI), but use different methods:

### runc - Standard Construction Crew

**The industry standard** (85% of all construction):

- Written in Go
- Reference OCI implementation
- Reliable, proven, everywhere
- Used by default in Docker, containerd, CRI-O, Podman

```bash
# Standard crew at work
runc create --bundle /site my-building
# Uses standard tools: unshare(), mount(), execve()
```

---

### crun - Speed Construction Crew

**The performance specialists**:

- Written in C (lighter, faster)
- 33% faster startup than runc
- 50% less memory
- Default in RHEL 9 / OpenShift

```
Benchmark: Building 100 houses

runc:  120ms per building
crun:   80ms per building  ← 33% faster!

Memory per building:
runc:  12MB
crun:   6MB  ← 50% less!
```

**When to use crun**:
- Performance-critical (high pod churn)
- Resource-constrained (embedded systems)
- Red Hat ecosystem

---

### runsc / gVisor - Reinforced Bunker Construction

**The security specialists**:

- Builds with extra isolation layer
- User-space kernel (like a bunker within a bunker)
- Used by Google Cloud Run

```
Standard building (runc):
Building → Direct foundation on ground (host kernel)
         → 300+ connection points to ground (syscalls)
         → If compromised, ground at risk

Reinforced bunker (gVisor):
Building → Reinforced floor (gVisor kernel in userspace)
         → Only 54 connection points to ground (limited syscalls)
         → If compromised, bunker contains damage! ✅
```

**Trade-off**: 
- 2x slower startup
- 30% slower I/O
- But MUCH safer for untrusted tenants

**When to use runsc**:
- Multi-tenant platforms (SaaS)
- Running untrusted code (CI/CD, serverless)
- Security > performance

```yaml
# Kubernetes: Run untrusted code in bunker
apiVersion: v1
kind: Pod
spec:
  runtimeClassName: gvisor  ← Extra security!
  containers:
  - name: untrusted
    image: customer-uploaded:latest
```

---

### Kata Containers - Separate Isolated Lot

**The ultimate isolation**:

- Each building gets its OWN LOT (lightweight VM)
- Each building has its OWN FOUNDATION (separate kernel)
- Uses hypervisor (QEMU, Firecracker)

```
Standard construction (runc):
10 buildings on shared lot → Share same foundation (host kernel)

Kata construction:
10 buildings on 10 separate lots → Each has own foundation (own kernel)
```

**Trade-off**:
- 500ms startup (4x slower)
- 130MB overhead per building
- Lower density (200 buildings/property vs 1000)
- But strongest isolation (VM-level)

**When to use Kata**:
- Extreme security (banking, healthcare)
- Compliance requirements (PCI-DSS, HIPAA)
- Mixed-trust workloads

---

## Building Without a Construction Company: CI/CD Scenarios

**Problem**: Your construction site (CI/CD environment) doesn't allow traditional construction companies (Docker daemon). What are your options?

### Option 1: Kaniko - Prefab Construction

**Kaniko builds buildings in a warehouse, ships completed**:

```
Traditional Docker Build:
Developer → Docker daemon → BuildKit → Image

Kaniko (No daemon needed):
Developer → Kaniko executor (runs in pod)
         → Reads blueprint (Dockerfile)
         → Builds in userspace
         → Ships directly to registry
```

**Pros**:
✅ No daemon required
✅ No root privileges needed
✅ Kubernetes-native (runs in pods)
✅ Secure by default

**Cons**:
❌ 30-50% slower than Docker build
❌ Must push to registry (can't build locally)

**When to use**: Kubernetes-native CI/CD (GitLab, Tekton, Argo)

```yaml
# GitLab CI with Kaniko
build:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:latest
    entrypoint: [""]
  script:
    - /kaniko/executor
      --context $CI_PROJECT_DIR
      --dockerfile Dockerfile
      --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
```

---

### Option 2: Buildah - Scriptable DIY Construction

**Buildah is like hiring individual tradespeople directly**:

```
Traditional: Follow rigid blueprint (Dockerfile)

Buildah: Hire workers one by one (scriptable):
1. Pour foundation → buildah from alpine
2. Install plumbing → buildah run $container apk add nginx
3. Add fixtures → buildah copy $container config.conf /etc/
4. Finish → buildah commit $container myapp:v1
```

**Pros**:
✅ No daemon required
✅ Scriptable (full programmatic control)
✅ Works with Dockerfiles OR custom scripts
✅ Rootless capable

**Cons**:
❌ Different syntax (not Docker CLI)
❌ Learning curve

**When to use**: Red Hat ecosystem, scriptable builds, custom workflows

---

### Option 3: BuildKit - Modern Construction Methods

**Docker's modern build engine, can run standalone**:

```
Traditional Docker: Old construction methods
BuildKit: Modern techniques:
├── Parallel building (multiple crews working simultaneously)
├── Smart caching (reuse materials efficiently)
├── Secret management (hide sensitive materials)
└── Multi-platform (build for different climates)
```

**Pros**:
✅ Fastest build performance
✅ Advanced features (cache mounts, secrets)
✅ Multi-platform builds (amd64, arm64)

**Cons**:
❌ Requires buildkitd daemon (lighter than dockerd though)

**When to use**: Performance-critical builds, need advanced features

---

## Choosing Your Construction Company for Production

### Decision Framework

**Factor 1: Who's your property manager? (Cloud Provider)**

| Property Manager | Default Construction Company |
|-----------------|------------------------------|
| Google (GKE) | containerd |
| Amazon (EKS) | containerd |
| Microsoft (AKS) | containerd |
| Red Hat (OpenShift) | CRI-O |

**Rule**: Use the default! Fighting it adds complexity.

---

**Factor 2: What's your existing infrastructure? (Ecosystem)**

```
If you have:
├── Red Hat property (RHEL, OpenShift)
├── Buildah for design
├── Podman for testing
└── Quay registry
    → Use CRI-O (tight integration)

If you have:
├── Multi-cloud properties
├── Standard Kubernetes
└── Docker for development
    → Use containerd (industry standard)
```

---

**Factor 3: How important is security? (Requirements)**

```
Standard Security:
└── Both containerd and CRI-O are secure ✅

Extra Security Needed:
├── Multi-tenant platform
├── Untrusted code
└── Compliance requirements
    → Use alternative crews: gVisor or Kata
    → Works with BOTH containerd and CRI-O!
```

---

**Factor 4: Team experience? (Operations)**

```
Large team, many tools, need flexibility:
└── containerd (more debugging tools, broader ecosystem)

Small team, Kubernetes-only:
└── CRI-O (simpler, one way to do things)

Want Docker-like experience:
└── containerd + nerdctl (Docker-compatible CLI)
```

---

### Recommendation Summary

**Choose containerd if**:
- ✅ Using managed Kubernetes (GKE, EKS, AKS)
- ✅ Multi-cloud strategy
- ✅ Want industry standard (70% market share)
- ✅ Need broader tooling ecosystem

**Choose CRI-O if**:
- ✅ Using Red Hat OpenShift
- ✅ Running on RHEL
- ✅ Pure Kubernetes environment
- ✅ Want version alignment (CRI-O 1.28 = K8s 1.28)

**Use Podman if**:
- ✅ Local development (Docker alternative)
- ✅ Rootless security requirements
- ✅ Building images without daemon

**Use alternative crews (gVisor, Kata) if**:
- ✅ Extreme security requirements
- ✅ Multi-tenant platforms
- ✅ Compliance mandates (PCI-DSS, HIPAA)

---

## Process Tree Visualization

Let's see the construction team hierarchy on a real system:

```bash
$ ps aux | grep -E "containerd|shim|nginx"

systemd (PID 1) ← Your computer's main supervisor
│
├─ containerd (PID 1234) ← Architecture Firm (running 24/7)
│  │
│  ├─ containerd-shim (PID 5678) ← Foreman for building #1
│  │  └─ nginx (PID 5679) ← Building #1 resident
│  │     └─ nginx worker (PID 5680)
│  │
│  ├─ containerd-shim (PID 6789) ← Foreman for building #2
│  │  └─ redis (PID 6790) ← Building #2 resident
│  │
│  └─ containerd-shim (PID 7890) ← Foreman for building #3
│     └─ postgres (PID 7891) ← Building #3 resident

Notice:
1. containerd runs continuously (architecture firm office open)
2. Each building has its own foreman (shim)
3. runc is NOT in the tree (crews already left!)
4. Buildings (containers) run independently
```

**Inside a building** (container's perspective):

```bash
$ docker exec building-1 ps aux

PID   USER     COMMAND
1     root     nginx: master process    ← Thinks it's PID 1 (main resident)
7     nginx    nginx: worker process

From inside the building:
- Looks like you're the only resident (isolated PID namespace)
- You're PID 1 (the foundation of this building)
- No visibility into other buildings or the construction company
```

---

## Key Takeaways

1. **Three layers, three jobs**: Architecture Firm (high-level runtime) plans, Site Foreman (shim) supervises, Construction Crew (runc) builds then leaves

2. **Buildings stand independently**: Once constructed, containers don't need the construction crew to remain - foreman supervises, building operates

3. **OCI specs are building codes**: Universal standards allow any crew (runc, crun, runsc) to build from any blueprint (config.json)

4. **Dockershim was a middleman**: Removed unnecessary layer between Kubernetes and containerd - buildings kept standing!

5. **containerd vs CRI-O**: Both excellent, choose based on ecosystem (containerd = broad, CRI-O = Kubernetes-focused)

6. **Podman is DIY**: No company daemon overhead, you directly coordinate foreman and crew - better security (rootless)

7. **Different crews, same codes**: runc (standard), crun (fast), runsc (secure), Kata (isolated) - all follow OCI specs

8. **Alternative construction methods exist**: Kaniko, Buildah, BuildKit enable building without traditional Docker daemon

9. **The foreman's critical role**: Shim staying after crew leaves allows the architecture firm to restart without disrupting buildings

10. **Choose based on context**: Cloud provider defaults, existing ecosystem, and team experience matter more than marginal technical differences

---

## See Also

- [Docker Networking](2026-01-01-docker-networking.md) - Understanding container networking
- [Docker Storage](2026-01-02-docker-storage.md) - How container storage works
- [Docker Multi-stage Builds](2026-01-05-docker-multi-stage-builds.md) - Optimizing image builds

---

*Written on January 6, 2026*
