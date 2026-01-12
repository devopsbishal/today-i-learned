# Containers Fundamentals - The Prison Analogy

> Understanding what containers really are at the Linux OS level - they're not magic, just clever use of Linux primitives.

---

## TL;DR

| Container Concept | Prison Analogy |
|-------------------|----------------|
| Host Kernel | Prison building infrastructure (plumbing, electricity, HVAC) |
| Container | Individual cell with inmate |
| Process | Inmate (prisoner) |
| Namespaces | Smart walls making each cell feel like own universe |
| Cgroups | Warden's utility meters limiting resources per cell |
| Union FS (OverlayFS) | Standard cell furniture + inmate's personal belongings |
| Image Layers | Base cell → Standard furniture → Inmate's stuff |
| Copy-on-Write | Want to modify standard poster? Copy to your layer first |
| Docker | Prison administration system (fancy UI) |
| runc | The actual builder constructing cells |
| Container Escape | Prison break! Security breach |
| Privileged Container | Giving inmate the master key 🔑 |
| VM | Completely separate prison building |

---

## The Big Picture

Containers are **NOT magic virtual machines**. They're just **Linux processes with fancy isolation**.

```
┌─────────────────────────────────────────────────────────┐
│              PRISON = HOST MACHINE                      │
│         (One building, shared infrastructure)           │
│                                                         │
│  Cell A (Container 1)    Cell B (Container 2)          │
│  ┌──────────────────┐    ┌──────────────────┐          │
│  │ Inmate: nginx    │    │ Inmate: postgres │          │
│  │ PID 1 (own view) │    │ PID 1 (own view) │          │
│  │ IP: 172.17.0.2   │    │ IP: 172.17.0.3   │          │
│  │ Hostname: web-1  │    │ Hostname: db-1   │          │
│  │                  │    │                  │          │
│  │ "I'm alone!"     │    │ "I'm alone!"     │          │
│  └──────────────────┘    └──────────────────┘          │
│                                                         │
│  Shared Infrastructure (One Linux Kernel):              │
│  • Plumbing (Network stack)                             │
│  • Electricity (CPU scheduler)                          │
│  • HVAC (Memory manager)                                │
│                                                         │
│  From host: Both are just PIDs 5234 and 5899!          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Key Insight**: Containers don't create new kernels. They make processes **believe** they're isolated using Linux features that have existed since 2008!

---

## The Three Pillars of Containers

### 1. Namespaces - The Smart Walls

Namespaces make each cell feel like its own universe. The inmate thinks they're alone, but they're just in a cleverly constructed cell.

```
┌─────────────────────────────────────────────────────────┐
│          THE 7 TYPES OF SMART WALLS                     │
│                                                         │
│  1. PID Namespace (Cell Numbering)                     │
│     ┌──────────────────────────────────────┐           │
│     │ Cell A: "I'm prisoner #1!"           │           │
│     │ Cell B: "I'm prisoner #1!"           │           │
│     │                                      │           │
│     │ Warden sees: A=#5234, B=#5899       │           │
│     └──────────────────────────────────────┘           │
│                                                         │
│  2. Network Namespace (Phone Lines)                    │
│     ┌──────────────────────────────────────┐           │
│     │ Cell A: Own phone (172.17.0.2)       │           │
│     │ Cell B: Own phone (172.17.0.3)       │           │
│     │ Can't hear each other's calls        │           │
│     └──────────────────────────────────────┘           │
│                                                         │
│  3. Mount Namespace (Wall Decorations)                 │
│     ┌──────────────────────────────────────┐           │
│     │ Cell A: Sees only their posters      │           │
│     │ Cell B: Sees only their posters      │           │
│     │ Can mount/unmount independently      │           │
│     └──────────────────────────────────────┘           │
│                                                         │
│  4. UTS Namespace (Cell Name)                          │
│     ┌──────────────────────────────────────┐           │
│     │ Cell A: "My name is web-server"      │           │
│     │ Cell B: "My name is database"        │           │
│     │ Different hostnames                  │           │
│     └──────────────────────────────────────┘           │
│                                                         │
│  5. IPC Namespace (Note Passing)                       │
│     ┌──────────────────────────────────────┐           │
│     │ Cells can't pass notes unless        │           │
│     │ explicitly allowed (shared memory)   │           │
│     └──────────────────────────────────────┘           │
│                                                         │
│  6. User Namespace (Identity)                          │
│     ┌──────────────────────────────────────┐           │
│     │ Cell A: UID 0 (root inside)          │           │
│     │         = UID 100000 on host         │           │
│     │                                      │           │
│     │ Inmate is "king" in cell,           │           │
│     │ but nobody outside!                  │           │
│     └──────────────────────────────────────┘           │
│                                                         │
│  7. Cgroup Namespace (Utility Panel View)              │
│     ┌──────────────────────────────────────┐           │
│     │ Inmates can't see full utility       │           │
│     │ control panel, only their meters     │           │
│     └──────────────────────────────────────┘           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Creating a namespace** is just a Linux syscall:

```c
// This is what Docker/runc does under the hood
clone(CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | 
      CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWUSER |
      CLONE_NEWCGROUP);
```

### 2. Cgroups - The Warden's Utility Meters

Without cgroups, one inmate could use all the electricity and leave everyone else in the dark!

```
┌─────────────────────────────────────────────────────────┐
│          WARDEN'S CONTROL ROOM                          │
│                                                         │
│  Cell A Meters:                                         │
│  ┌──────────────────────────────────────┐              │
│  │ ⚡ Electricity (CPU): 0.5 cores      │              │
│  │ 💧 Water (Memory): 512 MB            │              │
│  │ 📡 Phone (Network): 100 Mbps         │              │
│  │ 📦 Storage I/O: 10 MB/s              │              │
│  └──────────────────────────────────────┘              │
│                                                         │
│  Cell B Meters:                                         │
│  ┌──────────────────────────────────────┐              │
│  │ ⚡ Electricity (CPU): 2.0 cores      │              │
│  │ 💧 Water (Memory): 2 GB              │              │
│  │ 📡 Phone (Network): 1 Gbps           │              │
│  │ 📦 Storage I/O: 50 MB/s              │              │
│  └──────────────────────────────────────┘              │
│                                                         │
│  If Cell A tries to use more than allocated:           │
│  • CPU: Throttled (slowed down)                        │
│  • Memory: OOM Killer terminates process! 💀           │
│  • I/O: Queued/throttled                               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Cgroups in filesystem:**

```bash
# This is where cgroups live
/sys/fs/cgroup/
├── memory/
│   └── docker/
│       └── <container-id>/
│           └── memory.limit_in_bytes  # 512M
├── cpu/
│   └── docker/
│       └── <container-id>/
│           └── cpu.shares  # 512 = 0.5 CPU
└── blkio/
    └── docker/
        └── <container-id>/
            └── blkio.throttle.read_bps_device
```

### 3. Union Filesystem - The Cell Furniture

Every cell has **standard furniture** (base image layers) + **inmate's personal belongings** (container's writable layer).

```
┌─────────────────────────────────────────────────────────┐
│          CELL CONSTRUCTION (OverlayFS)                  │
│                                                         │
│  Layer 5: Inmate's Personal Items (R/W)                │
│  ┌────────────────────────────────────────────┐        │
│  │ • Modified config.txt                      │        │
│  │ • New photos on wall                       │        │
│  │ • Personal notes                           │        │
│  │                                            │        │
│  │ THIS is the only writable layer!          │        │
│  └────────────────────────────────────────────┘        │
│                     ▲                                   │
│                     │ Copy-on-Write                     │
│  ─────────────────────────────────────────────         │
│  Layer 4: Application (Read-Only)                      │
│  ┌────────────────────────────────────────────┐        │
│  │ • nginx binary                             │        │
│  │ • config files                             │        │
│  └────────────────────────────────────────────┘        │
│  ─────────────────────────────────────────────         │
│  Layer 3: Dependencies (Read-Only)                     │
│  ┌────────────────────────────────────────────┐        │
│  │ • Libraries (libc, libssl)                 │        │
│  └────────────────────────────────────────────┘        │
│  ─────────────────────────────────────────────         │
│  Layer 2: Package Manager (Read-Only)                  │
│  ┌────────────────────────────────────────────┐        │
│  │ • apt, dpkg                                │        │
│  └────────────────────────────────────────────┘        │
│  ─────────────────────────────────────────────         │
│  Layer 1: Base OS (Read-Only)                          │
│  ┌────────────────────────────────────────────┐        │
│  │ • Ubuntu core files                        │        │
│  │ • /bin, /usr, /lib                         │        │
│  └────────────────────────────────────────────┘        │
│                                                         │
│  All layers stacked = What inmate sees as "/"          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Copy-on-Write Magic:**

```
Inmate wants to edit /etc/nginx.conf

Step 1: Check Layer 5 (R/W)
  Not found.

Step 2: Check Layer 4 (R/O)
  Found it! But read-only...

Step 3: COPY /etc/nginx.conf from Layer 4 → Layer 5

Step 4: Modify in Layer 5

Step 5: Future reads use Layer 5 version
        (Layer 4 original untouched!)
```

**Why this matters:**

```
Scenario: 100 nginx containers from same image

❌ Without Union FS:
   100 × 200 MB = 20 GB disk used!

✅ With Union FS:
   1 base image (200 MB)
   + 100 thin layers (2 MB each)
   = 200 MB + 200 MB = 400 MB total!
   
   50x disk savings! 🎉
```

---

## What Happens When You Run `docker run`?

The prison administration system in action:

```
You: "docker run -it ubuntu bash"
     "Put someone in a cell with bash terminal"
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│ 1. Docker CLI (Front Desk)                              │
│    Parses command, sends REST API to dockerd            │
└────────────────┬────────────────────────────────────────┘
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 2. dockerd (Prison Administrator)                       │
│    • Checks if "ubuntu" blueprint exists                │
│    • Downloads from Docker Hub if needed                │
│    • Sends gRPC to containerd: "Build this cell!"       │
└────────────────┬────────────────────────────────────────┘
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 3. containerd (Cell Designer)                           │
│    • Unpacks image layers (furniture blueprints)        │
│    • Prepares rootfs (arranges furniture)               │
│    • Generates OCI spec (construction plans)            │
│    • Spawns containerd-shim (foreman)                   │
└────────────────┬────────────────────────────────────────┘
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 4. containerd-shim (Construction Foreman)               │
│    • Stays with cell even if designer leaves            │
│    • Manages inmate's communication channels            │
│    • Calls runc to actually build                       │
└────────────────┬────────────────────────────────────────┘
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 5. runc (The Builder) - THE MAGIC HAPPENS!              │
│                                                         │
│    Step 1: Build Smart Walls (Create Namespaces)       │
│    ┌──────────────────────────────────────┐            │
│    │ clone(CLONE_NEWPID | CLONE_NEWNET |  │            │
│    │       CLONE_NEWNS | CLONE_NEWUTS |   │            │
│    │       CLONE_NEWIPC | CLONE_NEWUSER)  │            │
│    └──────────────────────────────────────┘            │
│                                                         │
│    Step 2: Install Utility Meters (Setup Cgroups)      │
│    ┌──────────────────────────────────────┐            │
│    │ /sys/fs/cgroup/memory/<id>/          │            │
│    │   memory.limit_in_bytes = 512M       │            │
│    │ /sys/fs/cgroup/cpu/<id>/             │            │
│    │   cpu.shares = 512                   │            │
│    └──────────────────────────────────────┘            │
│                                                         │
│    Step 3: Layer Cell Furniture (Mount OverlayFS)      │
│    ┌──────────────────────────────────────┐            │
│    │ mount -t overlay overlay             │            │
│    │  -o lowerdir=/layer1:/layer2,        │            │
│    │     upperdir=/writable,              │            │
│    │     workdir=/work,                   │            │
│    │  /merged                             │            │
│    └──────────────────────────────────────┘            │
│                                                         │
│    Step 4: Install Phone Line (Network Setup)          │
│    ┌──────────────────────────────────────┐            │
│    │ • Create veth pair (virtual cable)   │            │
│    │ • One end in cell (eth0)             │            │
│    │ • Other end on prison network        │            │
│    │ • Assign IP: 172.17.0.2              │            │
│    └──────────────────────────────────────┘            │
│                                                         │
│    Step 5: Change Perspective (pivot_root)              │
│    ┌──────────────────────────────────────┐            │
│    │ pivot_root("/merged", "/old")        │            │
│    │ Inmate now sees /merged as /         │            │
│    │ Can't see host's real /              │            │
│    └──────────────────────────────────────┘            │
│                                                         │
│    Step 6: Set Security Rules (Drop Capabilities)       │
│    ┌──────────────────────────────────────┐            │
│    │ Drop dangerous privileges:            │            │
│    │ • CAP_SYS_ADMIN (can't mount)        │            │
│    │ • CAP_NET_ADMIN (can't change net)   │            │
│    │ • CAP_SYS_MODULE (can't load kernel) │            │
│    └──────────────────────────────────────┘            │
│                                                         │
│    Step 7: Apply Seccomp Filter (Block Syscalls)       │
│    ┌──────────────────────────────────────┐            │
│    │ Block dangerous system calls:         │            │
│    │ • Can't ptrace (debug other process) │            │
│    │ • Can't reboot                       │            │
│    │ • Can't modify kernel                │            │
│    └──────────────────────────────────────┘            │
│                                                         │
│    Step 8: Put Inmate in Cell (exec bash)              │
│    ┌──────────────────────────────────────┐            │
│    │ execve("/bin/bash", ...)             │            │
│    │ Bash now running in isolated cell!   │            │
│    └──────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────┘
                 ▼
            You get: root@abc123:/# 
            (Inmate thinks they're alone in universe!)
```

**Key Insight**: Docker is just a fancy UI. **runc** (or any OCI-compliant runtime) does the actual work using basic Linux syscalls!

---

## Containers vs Virtual Machines

### VMs = Separate Prison Buildings

```
┌─────────────────────────────────────────────────────────┐
│           PHYSICAL SERVER (Land)                        │
│                                                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐       │
│  │ Prison A   │  │ Prison B   │  │ Prison C   │       │
│  │            │  │            │  │            │       │
│  │ OS: Ubuntu │  │ OS: CentOS │  │ OS: Windows│       │
│  │ Kernel     │  │ Kernel     │  │ Kernel     │ ← Each has own!
│  │ Apps       │  │ Apps       │  │ Apps       │       │
│  │            │  │            │  │            │       │
│  │ Full infra │  │ Full infra │  │ Full infra │       │
│  └────────────┘  └────────────┘  └────────────┘       │
│        ▲                ▲                ▲             │
│        └────────────────┴────────────────┘             │
│              Hypervisor (Land Manager)                 │
│              Host OS & Kernel                          │
└─────────────────────────────────────────────────────────┘

Boot time: Minutes (full OS boot)
Isolation: Strong (different kernels)
Resource overhead: High (full OS per VM)
```

### Containers = Cells in One Building

```
┌─────────────────────────────────────────────────────────┐
│           PHYSICAL SERVER (Prison Building)             │
│                                                         │
│  Host OS & Kernel (Shared Infrastructure)               │
│  ────────────────────────────────────────               │
│  One plumbing system, one HVAC, one power grid          │
│                                                         │
│  🏢Cell1   🏢Cell2   🏢Cell3   🏢Cell4                 │
│  nginx     postgres  redis     app                     │
│  (Isolated)(Isolated)(Isolated)(Isolated)              │
│                                                         │
│  All share same kernel - just think they're alone!     │
└─────────────────────────────────────────────────────────┘

Boot time: Milliseconds (just start a process)
Isolation: Good (same kernel = potential escape)
Resource overhead: Low (no duplicate OS)
```

**Trade-offs:**

| Aspect | VM | Container |
|--------|----|-----------| 
| **Startup** | Minutes | Milliseconds |
| **Size** | GBs | MBs |
| **Isolation** | Strong (different kernels) | Good (shared kernel) |
| **Security** | More secure | Requires defense in depth |
| **Density** | 10-20 VMs per host | 100s of containers |
| **Use Case** | Complete OS isolation | Microservices, dev environments |

---

## Container Security - Preventing Prison Breaks

### Defense in Depth Strategy

Think of security as multiple layers - even if one fails, others protect:

```
┌─────────────────────────────────────────────────────────┐
│         CONTAINER SECURITY LAYERS                       │
│                                                         │
│  Layer 1: User Namespaces (Identity Mapping)           │
│  ┌────────────────────────────────────────┐            │
│  │ UID 0 in cell = UID 100000 on host     │            │
│  │ Inmate thinks they're king,            │            │
│  │ but they're nobody outside!            │            │
│  └────────────────────────────────────────┘            │
│                                                         │
│  Layer 2: Dropped Capabilities (Reduced Powers)        │
│  ┌────────────────────────────────────────┐            │
│  │ --cap-drop=ALL                         │            │
│  │ --cap-add=NET_BIND_SERVICE             │            │
│  │ Only specific permissions granted      │            │
│  └────────────────────────────────────────┘            │
│                                                         │
│  Layer 3: Seccomp Profile (Syscall Filter)             │
│  ┌────────────────────────────────────────┐            │
│  │ Block dangerous system calls:           │            │
│  │ • mount, ptrace, reboot                │            │
│  │ • Only safe syscalls allowed           │            │
│  └────────────────────────────────────────┘            │
│                                                         │
│  Layer 4: AppArmor/SELinux (MAC)                       │
│  ┌────────────────────────────────────────┐            │
│  │ Mandatory Access Control               │            │
│  │ Even root can't bypass these rules     │            │
│  └────────────────────────────────────────┘            │
│                                                         │
│  Layer 5: Read-only Root FS                            │
│  ┌────────────────────────────────────────┐            │
│  │ --read-only --tmpfs /tmp               │            │
│  │ Can't modify system files              │            │
│  └────────────────────────────────────────┘            │
│                                                         │
│  Layer 6: Network Policies                             │
│  ┌────────────────────────────────────────┐            │
│  │ Limit what cell can call               │            │
│  │ Only allowed connections               │            │
│  └────────────────────────────────────────┘            │
│                                                         │
│  Break one layer → Others still protect! 🛡️            │
└─────────────────────────────────────────────────────────┘
```

### Common Prison Break Scenarios

#### 🚨 Scenario 1: The Master Key (--privileged)

```bash
# NEVER DO THIS IN PRODUCTION!
docker run --privileged ubuntu
```

This is like giving the inmate the master key to the entire prison!

```
┌─────────────────────────────────────────────┐
│ What --privileged does:                     │
│                                             │
│ ❌ Disables ALL security features           │
│ ❌ Gives ALL Linux capabilities             │
│ ❌ Mounts host's /dev (device access)       │
│ ❌ Can load kernel modules!                 │
│ ❌ Disables seccomp                         │
│ ❌ Disables AppArmor/SELinux                │
│                                             │
│ Result: Container = Root on host! 💀        │
└─────────────────────────────────────────────┘
```

**Real breakout with privileged container:**

```bash
# Inside privileged container
mkdir /tmp/hostfs
mount /dev/sda1 /tmp/hostfs  # Mount host's disk!
chroot /tmp/hostfs           # Change root to host
# Now you're on the HOST, not in container!
```

#### 🚨 Scenario 2: Exposing Docker Socket

```bash
# DANGEROUS: Giving inmate control of prison security!
docker run -v /var/run/docker.sock:/var/run/docker.sock ubuntu
```

```
┌─────────────────────────────────────────────┐
│ Now container can control Docker!           │
│                                             │
│ Inside container:                           │
│   docker run -v /:/host --privileged alpine │
│                                             │
│ Created NEW container with:                 │
│ • Host's entire filesystem mounted          │
│ • Privileged mode                           │
│                                             │
│ Game over. 🏴‍☠️                              │
└─────────────────────────────────────────────┘
```

#### 🚨 Scenario 3: Kernel Vulnerability

```
┌─────────────────────────────────────────────┐
│         THE SHARED KERNEL PROBLEM           │
│                                             │
│  Container → Uses Host Kernel → Exploit     │
│                                             │
│  Example: Dirty COW (CVE-2016-5195)         │
│  • Bug in kernel's memory management        │
│  • Container exploits it                    │
│  • Gains root on host                       │
│                                             │
│  This is why VMs are MORE secure:           │
│  Different kernels = exploit doesn't cross! │
└─────────────────────────────────────────────┘
```

#### ✅ Scenario 4: Proper Security (Defense in Depth)

```bash
# The RIGHT way to run containers
docker run \
  --user 1000:1000 \              # Non-root user
  --cap-drop=ALL \                # Drop all capabilities
  --cap-add=NET_BIND_SERVICE \    # Add only what's needed
  --read-only \                   # Read-only root FS
  --tmpfs /tmp \                  # Writable /tmp in memory
  --security-opt=no-new-privileges \ # Can't escalate
  --security-opt=seccomp=profile.json \ # Custom seccomp
  --memory=512m \                 # Memory limit
  --cpus=0.5 \                    # CPU limit
  --pids-limit=100 \              # Process limit
  ubuntu
```

This is a well-secured cell with multiple layers of protection!

---

## The OCI Standard - Universal Prison Blueprint

**OCI (Open Container Initiative)** is like building codes for prisons:

```
┌─────────────────────────────────────────────────────────┐
│              OCI STANDARD                               │
│                                                         │
│  Two Specifications:                                    │
│                                                         │
│  1. Runtime Spec (How to build a cell)                 │
│     ┌──────────────────────────────────────┐           │
│     │ • config.json (blueprint)            │           │
│     │ • What namespaces to create          │           │
│     │ • What cgroups to set                │           │
│     │ • What to mount                      │           │
│     │ • What capabilities to drop          │           │
│     └──────────────────────────────────────┘           │
│                                                         │
│  2. Image Spec (Furniture blueprints)                  │
│     ┌──────────────────────────────────────┐           │
│     │ • Layer tarballs                     │           │
│     │ • manifest.json                      │           │
│     │ • config.json (metadata)             │           │
│     └──────────────────────────────────────┘           │
│                                                         │
│  Any compliant runtime can build these cells:          │
│  • runc (Docker's builder)                             │
│  • crun (Written in C, faster)                         │
│  • runsc (gVisor - extra sandbox)                      │
│  • kata-runtime (Lightweight VMs!)                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Why this matters:**

```
Docker image → OCI format → Works with:
  ├─ Docker
  ├─ Podman
  ├─ containerd
  ├─ CRI-O (Kubernetes)
  └─ Any OCI runtime!

No vendor lock-in! 🎉
```

---

## Container Runtimes - Different Builders

```
┌─────────────────────────────────────────────────────────┐
│          CONTAINER RUNTIME LANDSCAPE                    │
│                                                         │
│  High-Level Runtimes (Architects):                     │
│  ┌────────────────────────────────────────┐            │
│  │ Docker:      Full featured, daemon     │            │
│  │ Podman:      Daemonless, rootless      │            │
│  │ containerd:  Core runtime, K8s uses    │            │
│  │ CRI-O:       Built for Kubernetes      │            │
│  └────────────────────────────────────────┘            │
│                     │                                   │
│                     ▼                                   │
│  Low-Level Runtimes (Builders):                        │
│  ┌────────────────────────────────────────┐            │
│  │ runc:     Reference OCI implementation │            │
│  │ crun:     Faster, written in C         │            │
│  │ runsc:    gVisor (extra sandbox layer) │            │
│  │ kata:     Lightweight VMs for extra    │            │
│  │           isolation                    │            │
│  └────────────────────────────────────────┘            │
│                                                         │
│  Kubernetes doesn't care which builder!                │
│  Just needs OCI compliance.                            │
└─────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Containers are just processes** - with fancy Linux isolation (namespaces + cgroups + union FS)
2. **Namespaces create isolation** - make processes think they're alone (7 types)
3. **Cgroups limit resources** - prevent one container from consuming everything
4. **Union FS saves disk space** - share base layers, only differences stored
5. **Shared kernel = speed but risk** - Faster than VMs, but kernel exploits affect all containers
6. **Docker is just UI** - runc (OCI runtime) does actual work with Linux syscalls
7. **Security requires layers** - User namespaces, capabilities, seccomp, read-only FS
8. **Never use --privileged** - It's giving inmates the master key
9. **OCI standard = portability** - Images work across different runtimes
10. **Containers ≠ VMs** - Different kernels vs shared kernel is the key difference

---

## Common Pitfalls

| Problem | Why | Solution |
|---------|-----|----------|
| **Running as root** | Container root = host root if escapes | Use USER in Dockerfile, --user flag |
| **Privileged mode** | Disables ALL security | Only use for specific use cases (DinD) |
| **Exposing Docker socket** | Container controls Docker | Never mount /var/run/docker.sock |
| **No resource limits** | One container crashes host | Always set memory/CPU limits |
| **Large images** | Slow pulls, wasted disk | Multi-stage builds, Alpine base |
| **Secrets in env vars** | Visible in docker inspect | Use Docker secrets, K8s secrets |
| **Latest tag in prod** | Unpredictable behavior | Pin to specific versions |
| **No health checks** | Dead containers still running | Add HEALTHCHECK in Dockerfile |

---

## Related Reading

- [AWS VPC - The Compound Analogy](../november/2025-11-30-aws-vpc.md)
- [AWS EKS - The Managed Office Building Analogy](./2025-12-02-aws-eks.md)

---

*Written on December 23, 2025*
