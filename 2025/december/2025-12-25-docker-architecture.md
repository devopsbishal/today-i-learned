# Docker Architecture - The Construction Company Analogy

> Understanding how Docker actually works: the chain from `docker run` to running container, and why each component exists.

---

## TL;DR

| Component | Construction Analogy | Real Job |
|-----------|---------------------|----------|
| **docker CLI** | You (the customer) | Parses commands, sends requests |
| **dockerd** | General Contractor | Customer-facing, image builds, volumes, networks |
| **containerd** | Project Manager | Container lifecycle, industry standard (CNCF) |
| **containerd-shim** | Site Supervisor | Monitors each container, stays even if PM leaves |
| **runc** | Construction Crew | Builds the container (namespaces, cgroups), then leaves |
| **Container** | Finished House | Running process, monitored by supervisor |

---

## The Big Picture

When you run `docker run nginx`, you're not talking to one person - you're talking to an entire construction company with specialized roles!

```
┌─────────────────────────────────────────────────────────┐
│           DOCKER CONSTRUCTION COMPANY                   │
│                                                         │
│  You: "I need an nginx house built!"                   │
│  (docker run nginx)                                     │
│                                                         │
│          ↓                                              │
│  ┌─────────────────────────────────────────┐           │
│  │  dockerd (General Contractor)           │           │
│  │  ───────────────────────────────────    │           │
│  │  • Customer-facing                      │           │
│  │  • Manages blueprints (images)          │           │
│  │  • Custom designs (docker build)        │           │
│  │  • Handles paperwork & permits          │           │
│  │  • Storage rental (volumes)             │           │
│  │  • Networking permits                   │           │
│  └─────────────────────────────────────────┘           │
│          ↓ (gRPC call)                                  │
│  ┌─────────────────────────────────────────┐           │
│  │  containerd (Project Manager)           │           │
│  │  ────────────────────────────────────   │           │
│  │  • Coordinates construction             │           │
│  │  • Manages materials (image layers)     │           │
│  │  • Industry standard (CNCF)             │           │
│  │  • Works for multiple contractors       │           │
│  │  • Focused ONLY on building houses      │           │
│  └─────────────────────────────────────────┘           │
│          ↓                                              │
│  ┌─────────────────────────────────────────┐           │
│  │  containerd-shim (Site Supervisor)      │           │
│  │  ──────────────────────────────────     │           │
│  │  • ONE per house                        │           │
│  │  • STAYS on site 24/7                   │           │
│  │  • Monitors even if PM goes on vacation │           │
│  │  • Reports status back to PM            │           │
│  │  • Collects final inspection when done  │           │
│  └─────────────────────────────────────────┘           │
│          ↓                                              │
│  ┌─────────────────────────────────────────┐           │
│  │  runc (Construction Crew)               │           │
│  │  ────────────────────────────────────   │           │
│  │  • Pours foundation (namespaces)        │           │
│  │  • Installs utilities (cgroups)         │           │
│  │  • Assembles structure (mount overlayfs)│           │
│  │  • Connects infrastructure (network)    │           │
│  │  • Finishes job and LEAVES              │           │
│  │  • Moves to next project                │           │
│  └─────────────────────────────────────────┘           │
│          ↓                                              │
│  ┌─────────────────────────────────────────┐           │
│  │  🏠 Running Container                   │           │
│  │  ────────────────────────────────       │           │
│  │  • nginx running (PID 1 in house)       │           │
│  │  • Supervisor watching 24/7             │           │
│  │  • Ready to serve customers             │           │
│  └─────────────────────────────────────────┘           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## The Players - Detailed Roles

### 1. dockerd - The General Contractor

The customer-facing business that handles all your requests.

```
┌─────────────────────────────────────────────────────────┐
│              GENERAL CONTRACTOR (dockerd)               │
│                                                         │
│  What they do:                                          │
│  ✓ Take customer orders (docker run, docker build)     │
│  ✓ Design custom blueprints (Dockerfile → Image)       │
│  ✓ Manage blueprint library (image registry)           │
│  ✓ Rent storage units (volumes)                        │
│  ✓ Handle networking permits (networks)                │
│  ✓ Orchestrate teams (Docker Swarm)                    │
│  ✓ Provide customer API (REST on Unix socket)          │
│                                                         │
│  What they DON'T do:                                    │
│  ✗ Actually build houses (delegates to containerd)     │
│  ✗ Monitor finished houses (shim does this)            │
│                                                         │
│  Why they exist:                                        │
│  Docker-specific features that aren't needed by        │
│  everyone. Kubernetes doesn't need custom blueprints   │
│  or storage rental - they just need houses built!      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Listens on:** Unix socket `/var/run/docker.sock` (front desk)

**Process:** `dockerd` running as daemon

**Can restart?** ✅ Yes! Running containers unaffected (they're with PM and supervisors)

---

### 2. containerd - The Project Manager

Industry-standard builder focused solely on construction.

```
┌─────────────────────────────────────────────────────────┐
│            PROJECT MANAGER (containerd)                 │
│                                                         │
│  What they do:                                          │
│  ✓ Unpack blueprint materials (image layers)           │
│  ✓ Prepare construction site (rootfs)                  │
│  ✓ Create construction plans (OCI spec)                │
│  ✓ Hire site supervisors (spawn shims)                 │
│  ✓ Coordinate with construction crews (runc)           │
│  ✓ Track all active construction sites                 │
│                                                         │
│  Why industry standard (CNCF):                          │
│  • Multiple contractors can use same PM                │
│  • Docker uses them                                     │
│  • Kubernetes uses them (via CRI)                      │
│  • Podman can use them                                  │
│                                                         │
│  Focused scope:                                         │
│  Just building and managing houses. No custom          │
│  blueprint design, no customer relations.              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Communicates via:** gRPC

**Process:** `containerd` daemon

**Can restart?** ✅ Yes! Supervisors keep houses running, PM reconnects after restart

---

### 3. containerd-shim - The Site Supervisor

One dedicated supervisor per house, staying on site 24/7.

```
┌─────────────────────────────────────────────────────────┐
│            SITE SUPERVISOR (containerd-shim)            │
│                                                         │
│  The Critical Job:                                      │
│  ┌──────────────────────────────────────┐              │
│  │ Project Manager: "Build House A"     │              │
│  │           ↓                           │              │
│  │ Supervisor hired for House A         │              │
│  │           ↓                           │              │
│  │ Calls construction crew (runc)       │              │
│  │           ↓                           │              │
│  │ Crew builds and LEAVES               │              │
│  │           ↓                           │              │
│  │ Supervisor STAYS on site!            │              │
│  │ • Monitors house 24/7                │              │
│  │ • Keeps utilities connected          │              │
│  │ • Reports status to PM               │              │
│  │ • Collects final report when house   │              │
│  │   is demolished                      │              │
│  └──────────────────────────────────────┘              │
│                                                         │
│  Why this matters:                                      │
│                                                         │
│  Scenario: PM goes on vacation (containerd crashes)    │
│  ❌ Without supervisors: All houses collapse!          │
│  ✅ With supervisors: Houses fine! Supervisor stays,   │
│     PM reconnects when back                            │
│                                                         │
│  Process tree:                                          │
│  containerd                                             │
│    ├─ shim (house 1) → nginx process                   │
│    ├─ shim (house 2) → postgres process                │
│    └─ shim (house 3) → redis process                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Process:** `containerd-shim-runc-v2` (one per container)

**Parent of:** Your actual container process (nginx, postgres, etc.)

**Job security:** ✅ Maximum! Stays until container dies

---

### 4. runc - The Construction Crew

The actual builders who create the container using Linux syscalls.

```
┌─────────────────────────────────────────────────────────┐
│            CONSTRUCTION CREW (runc)                     │
│                                                         │
│  The Build Process:                                     │
│                                                         │
│  1️⃣ Pour Foundation (Create Namespaces)                │
│     clone(CLONE_NEWPID | CLONE_NEWNET | ...)           │
│     → Isolates PID, Network, Mount, UTS, IPC, User     │
│                                                         │
│  2️⃣ Install Utility Meters (Setup Cgroups)             │
│     /sys/fs/cgroup/memory/<id>/memory.limit_in_bytes   │
│     → Limits CPU, memory, I/O                          │
│                                                         │
│  3️⃣ Assemble Structure (Mount OverlayFS)               │
│     mount -t overlay overlay -o lowerdir=...,upperdir..│
│     → Layer image filesystem                           │
│                                                         │
│  4️⃣ Connect to City Infrastructure (Network)           │
│     Create veth pair, assign IP, connect to bridge     │
│     → Container gets network connectivity              │
│                                                         │
│  5️⃣ Change Perspective (pivot_root)                    │
│     pivot_root("/merged", "/old")                      │
│     → Container sees only its own filesystem           │
│                                                         │
│  6️⃣ Install Security System (Drop Capabilities)        │
│     Drop CAP_SYS_ADMIN, CAP_NET_ADMIN, etc.            │
│     → Limit what container can do                      │
│                                                         │
│  7️⃣ Apply Seccomp Filter (Block Dangerous Syscalls)    │
│     Block mount, ptrace, reboot, etc.                  │
│     → Additional security layer                        │
│                                                         │
│  8️⃣ Start Resident (exec nginx)                        │
│     execve("/usr/sbin/nginx", ...)                     │
│     → Nginx now running as PID 1 in container          │
│                                                         │
│  9️⃣ Crew Leaves! (runc exits)                          │
│     Job done, move to next construction project        │
│     Supervisor (shim) takes over monitoring            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Process:** `runc` (short-lived! Exits after creating container)

**Job duration:** Seconds! Creates container and leaves

**Why exit?** Would waste resources having crew sit around. You might have 100+ houses!

---

## Why This Architecture?

### Question 1: Why not just have dockerd build directly?

```
┌─────────────────────────────────────────────┐
│  OLD DOCKER (Pre-2016)                      │
│                                             │
│  dockerd (Monolithic)                       │
│  ├─ Customer relations                      │
│  ├─ Custom blueprints (build)               │
│  ├─ Construction management                 │
│  ├─ House monitoring                        │
│  └─ Everything!                             │
│                                             │
│  Problems:                                  │
│  ❌ Upgrade dockerd → Kill all containers   │
│  ❌ Dockerd crash → Lose all houses         │
│  ❌ Kubernetes can't use (too much extra)   │
│  ❌ Vendor lock-in                          │
│                                             │
└─────────────────────────────────────────────┘

         ↓ Evolution ↓

┌─────────────────────────────────────────────┐
│  MODERN DOCKER (Post-2016)                  │
│                                             │
│  dockerd → containerd → shim → runc        │
│                                             │
│  Benefits:                                  │
│  ✅ Upgrade dockerd → Containers keep running│
│  ✅ Dockerd crash → Containers unaffected   │
│  ✅ Kubernetes uses containerd directly     │
│  ✅ Industry standard (OCI compliance)      │
│  ✅ Multiple tools can share containerd     │
│                                             │
└─────────────────────────────────────────────┘
```

### Question 2: What happens when each component restarts?

```
┌─────────────────────────────────────────────────────────┐
│          RESTART SCENARIOS                              │
│                                                         │
│  1. Restart dockerd (General Contractor)                │
│     ┌──────────────────────────────────┐               │
│     │ While dockerd is down:           │               │
│     │ ✅ Containers keep running       │               │
│     │ ❌ Can't create new containers   │               │
│     │ ❌ Can't build images            │               │
│     │ ❌ Can't manage volumes/networks │               │
│     │                                  │               │
│     │ After dockerd comes back:        │               │
│     │ ✅ docker ps shows all containers│               │
│     │ ✅ docker exec works             │               │
│     │ ✅ docker logs works             │               │
│     │ ❌ Crashed containers won't      │               │
│     │    auto-restart (restart policy  │               │
│     │    managed by dockerd)           │               │
│     └──────────────────────────────────┘               │
│                                                         │
│  2. Restart containerd (Project Manager)                │
│     ┌──────────────────────────────────┐               │
│     │ ✅ Containers keep running!       │               │
│     │    Supervisors (shims) stay      │               │
│     │                                  │               │
│     │ After containerd comes back:     │               │
│     │ ✅ Reconnects with shims         │               │
│     │ ✅ Full control restored         │               │
│     └──────────────────────────────────┘               │
│                                                         │
│  3. Kill containerd-shim (Site Supervisor)              │
│     ┌──────────────────────────────────┐               │
│     │ ❌ That container dies!           │               │
│     │    Supervisor is parent process  │               │
│     │    Killing it kills child        │               │
│     └──────────────────────────────────┘               │
│                                                         │
│  4. runc already exited!                                │
│     ┌──────────────────────────────────┐               │
│     │ runc is short-lived              │               │
│     │ Exits right after creating       │               │
│     │ container                        │               │
│     └──────────────────────────────────┘               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Question 3: Why does runc exit? Who manages the container?

**Before runc exits:**
```
containerd-shim
  └─ runc
      └─ nginx (PID 1 in container)
```

**After runc exits:**
```
containerd-shim
  └─ nginx (PID 1 in container)
```

**Why this works:**

1. **runc's job is BUILDING** (creating namespaces, cgroups, mounting, etc.)
2. Once built, the container is just a **running process**
3. **shim becomes the parent** and monitors it
4. Having runc stick around would waste resources
5. You might have 100+ containers - can't have 100 runc processes doing nothing!

**Shim's ongoing job:**
- Forward signals (`docker stop` → SIGTERM → container)
- Keep STDIN/STDOUT/STDERR connected
- Collect exit code when container dies
- Report status back to containerd

---

## Real-World Example: The Complete Flow

Let's trace `docker run -d nginx` through the entire company:

```
┌─────────────────────────────────────────────────────────┐
│  Step 1: You place the order                            │
│  ────────────────────────────                           │
│  $ docker run -d nginx                                  │
│                                                         │
│  Docker CLI → Parses command                            │
│             → Sends REST API to dockerd                 │
│               POST /containers/create                   │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│  Step 2: General Contractor receives order              │
│  ──────────────────────────────────────────             │
│  dockerd receives API call                              │
│                                                         │
│  Checks:                                                │
│  • Do we have nginx blueprint? (docker images)          │
│  • If not, download from Docker Hub                     │
│  • Validate image layers and manifest                   │
│                                                         │
│  Calls Project Manager:                                 │
│  → gRPC to containerd                                   │
│     "Build me a house from nginx blueprint"             │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│  Step 3: Project Manager prepares materials             │
│  ───────────────────────────────────────                │
│  containerd receives gRPC request                       │
│                                                         │
│  Actions:                                               │
│  • Unpack image layers from content store              │
│    /var/lib/containerd/io.containerd.content.v1.content/│
│  • Prepare root filesystem (stack layers)               │
│  • Generate OCI runtime spec (config.json):             │
│    {                                                    │
│      "namespaces": [PID, NET, MNT, UTS, IPC, USER],    │
│      "cgroups": {memory: "512M", cpu: "0.5"},          │
│      "mounts": [...]                                    │
│    }                                                    │
│                                                         │
│  Hires Site Supervisor:                                 │
│  → Spawns containerd-shim-runc-v2                      │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│  Step 4: Site Supervisor takes charge                   │
│  ────────────────────────────────────                   │
│  containerd-shim spawned                                │
│                                                         │
│  Supervisor's setup:                                    │
│  • Sets up stdio pipes (for docker logs later)         │
│  • Prepares to monitor container                        │
│  • Will stay for entire container lifetime              │
│                                                         │
│  Calls Construction Crew:                               │
│  → Executes: runc create <container-id>                 │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│  Step 5: Construction crew builds (runc)                │
│  ────────────────────────────────────                   │
│  THIS IS WHERE THE ACTUAL LINUX MAGIC HAPPENS!          │
│                                                         │
│  runc performs these Linux syscalls:                    │
│                                                         │
│  1. clone() - Create namespaces                         │
│  2. Write to /sys/fs/cgroup/ - Setup resource limits    │
│  3. mount() - Setup OverlayFS                           │
│  4. Create veth pair - Network setup                    │
│  5. pivot_root() - Change root filesystem               │
│  6. prctl() - Drop capabilities                         │
│  7. prctl() - Apply seccomp filter                      │
│  8. execve() - Execute nginx binary                     │
│                                                         │
│  Container now running with nginx as PID 1!             │
│                                                         │
│  runc exits (job done!)                                 │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│  Step 6: House is built and occupied                    │
│  ───────────────────────────────────                    │
│  Final process tree:                                    │
│                                                         │
│  systemd                                                │
│    └─ containerd                                        │
│        └─ containerd-shim                               │
│            └─ nginx (master)                            │
│                ├─ nginx (worker 1)                      │
│                └─ nginx (worker 2)                      │
│                                                         │
│  Supervisor (shim):                                     │
│  • Monitors nginx process                               │
│  • Forwards your commands (docker stop → SIGTERM)       │
│  • Collects logs                                        │
│  • Will report exit code when nginx dies                │
│                                                         │
│  You get: Container ID and "Running" status             │
│           $ docker ps                                   │
└─────────────────────────────────────────────────────────┘
```

---

## Kubernetes Simplification

Here's why Kubernetes dropped Docker (dockerd):

```
┌─────────────────────────────────────────────────────────┐
│  OLD: Kubernetes → Docker → containerd → runc          │
│  ──────────────────────────────────────────             │
│                                                         │
│  Kubernetes: "Run this pod"                             │
│       ↓                                                 │
│  dockerd: "Here, let me translate that"                 │
│           "Also, I have all these features you          │
│            don't need: build, volumes, swarm..."        │
│       ↓                                                 │
│  containerd: "OK, building container"                   │
│                                                         │
│  Problem: Extra layer (dockerd) not needed!             │
│           Docker-specific features unused               │
│                                                         │
└─────────────────────────────────────────────────────────┘

         ↓ Kubernetes v1.24+ ↓

┌─────────────────────────────────────────────────────────┐
│  NEW: Kubernetes → containerd → runc                    │
│  ────────────────────────────────────────               │
│                                                         │
│  Kubernetes (via CRI): "Run this pod"                   │
│       ↓                                                 │
│  containerd: "Got it! Building container"               │
│       ↓                                                 │
│  runc: Builds the container                             │
│                                                         │
│  Result: Simpler, faster, less overhead                 │
│          Still running containers the same way!         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**What "Docker removed" actually meant:**
- ❌ NOT removed: Container runtime (containerd/runc still there!)
- ❌ NOT removed: Docker images (OCI format works everywhere!)
- ✅ REMOVED: dockerd (the General Contractor layer)

**Translation:** Kubernetes hired the Project Manager directly instead of going through the General Contractor!

---

## Communication Protocols

How these components talk to each other:

```
┌─────────────────────────────────────────────────────────┐
│              COMMUNICATION CHAIN                        │
│                                                         │
│  docker CLI → dockerd                                   │
│    Protocol: REST API over Unix socket                 │
│    Socket: /var/run/docker.sock                        │
│    Example: POST /containers/create                     │
│                                                         │
│  dockerd → containerd                                   │
│    Protocol: gRPC                                       │
│    Socket: /run/containerd/containerd.sock             │
│    Example: containerd.services.containers.v1.Create    │
│                                                         │
│  containerd → containerd-shim                           │
│    Protocol: ttrpc (lighter than gRPC)                 │
│    Example: Start, Delete, State calls                 │
│                                                         │
│  containerd-shim → runc                                 │
│    Protocol: Command-line execution                     │
│    Example: /usr/bin/runc create <id>                  │
│                                                         │
│  containerd-shim → container process                    │
│    Protocol: Unix signals & stdio pipes                │
│    Example: kill(pid, SIGTERM)                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Alternative Runtimes

Different construction companies following the same building code (OCI):

```
┌─────────────────────────────────────────────────────────┐
│           CONTAINER RUNTIME ECOSYSTEM                   │
│                                                         │
│  High-Level Runtimes (General Contractors):             │
│  ┌────────────────────────────────────────┐            │
│  │ Docker      Full-featured, daemon      │            │
│  │ Podman      Daemonless, rootless       │            │
│  │ containerd  Kubernetes uses directly   │            │
│  │ CRI-O       Built specifically for K8s │            │
│  └────────────────────────────────────────┘            │
│                     │                                   │
│                     ↓                                   │
│  Low-Level Runtimes (Construction Crews):               │
│  ┌────────────────────────────────────────┐            │
│  │ runc        Reference OCI runtime      │            │
│  │ crun        Faster (written in C)      │            │
│  │ runsc       gVisor (extra sandbox)     │            │
│  │ kata        Lightweight VMs            │            │
│  └────────────────────────────────────────┘            │
│                                                         │
│  All follow OCI standard:                               │
│  • Same image format                                    │
│  • Same runtime spec                                    │
│  • Interchangeable!                                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Example - Podman:**
- No daemon (no General Contractor!)
- You directly talk to a lightweight coordinator
- Still uses containerd concepts
- Still uses runc or crun for actual building
- Uses buildah for custom blueprints (docker build equivalent)

---

## Key Takeaways

1. **Separation of Concerns** - Each component has a specific job:
   - dockerd = Customer features (build, volumes, networks)
   - containerd = Container lifecycle (build, run, manage)
   - shim = Monitoring (per-container supervisor)
   - runc = Building (create namespaces/cgroups, then exit)

2. **Resilience by Design** - Components can restart without killing containers:
   - dockerd restart → Containers keep running
   - containerd restart → Shims keep containers alive
   - Only killing shim kills its container

3. **Industry Standard** - containerd + runc = OCI compliance:
   - Not Docker-specific
   - Kubernetes uses them
   - Podman uses them
   - Any OCI-compliant tool can use them

4. **runc is Short-Lived** - It builds and exits:
   - Would waste resources staying around
   - Shim is lighter-weight for monitoring
   - Scales to hundreds of containers

5. **Evolution** - Docker split for good reasons:
   - Old: Monolithic dockerd (risky, vendor lock-in)
   - New: Modular architecture (stable, portable)

6. **Kubernetes Efficiency** - Why K8s dropped dockerd:
   - Didn't need build features
   - Didn't need volume/network management
   - Direct containerd use = simpler, faster

---

## Common Questions

**Q: If runc exits, who stops the container when I run `docker stop`?**

A: The shim! Here's the flow:
```
You: docker stop nginx-container
  ↓
dockerd: Receives API call
  ↓
containerd: "Stop container XYZ"
  ↓
shim: Sends SIGTERM to nginx process (PID 1)
  ↓
(waits 10 seconds)
  ↓
shim: If still running, sends SIGKILL
  ↓
nginx: Process terminates
  ↓
shim: Collects exit code, reports to containerd
```

**Q: Can I use containerd without Docker?**

A: Yes! Kubernetes does exactly this:
```bash
# Using containerd CLI (ctr)
ctr images pull docker.io/library/nginx:latest
ctr run docker.io/library/nginx:latest my-nginx

# Or use nerdctl (Docker-compatible CLI for containerd)
nerdctl run -d nginx
```

**Q: What's in the OCI spec that runc reads?**

A: It's a JSON file with construction plans:
```json
{
  "ociVersion": "1.0.0",
  "process": {
    "args": ["nginx", "-g", "daemon off;"]
  },
  "root": {
    "path": "/var/lib/containerd/snapshots/1/fs"
  },
  "linux": {
    "namespaces": [
      {"type": "pid"},
      {"type": "network"},
      {"type": "mount"}
    ],
    "resources": {
      "memory": {"limit": 536870912},
      "cpu": {"shares": 512}
    }
  }
}
```

**Q: Why not just use one process for everything?**

A: Same reason you don't have one person in a company do everything:
- **Specialization** - Each component is expert at one thing
- **Resilience** - One component failing doesn't cascade
- **Flexibility** - Swap components (use crun instead of runc)
- **Reusability** - Multiple tools share containerd
- **Upgrades** - Upgrade one layer without affecting others

---

## Related Reading

- [Containers Fundamentals - The Prison Analogy](./2025-12-23-containers-fundamentals.md)
- [AWS EKS - The Managed Office Building Analogy](./2025-12-02-aws-eks.md)

---

*Written on December 25, 2025*
