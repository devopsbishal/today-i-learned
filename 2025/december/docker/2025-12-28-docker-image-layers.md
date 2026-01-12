# Docker Image Layers & OverlayFS - The Transparent Overlay Sheets Analogy

> Understanding how Docker efficiently stores images and runs containers using layered filesystems.

---

## TL;DR

| Docker Concept | Transparent Sheets Analogy |
|----------------|---------------------------|
| Image Layers (lowerdir) | Pre-printed laminated sheets (read-only, shared) |
| Container Layer (upperdir) | Your personal notepad sheet (writable, unique) |
| Merged View | Combined image on projector screen |
| Copy-on-Write | Trace section from laminated sheet to your notepad |
| workdir | Staging table (atomic tracing) |
| Multiple Containers | 10 students with same base sheets, different notepads |
| Layer Caching | Reusing existing laminated sheets instead of creating new ones |
| docker commit | Laminating your notepad sheet (make it permanent) |
| Deleting Files | Drawing cover-up patch (original sheet still underneath) |
| Multi-stage Build | Two projectors - build on one, copy results to clean projector |

---

## The Big Picture

Docker images aren't monolithic files - they're **stacks of transparent layers** that combine to create what your container sees as its filesystem.

```
┌─────────────────────────────────────────────────────────┐
│           THE OVERHEAD PROJECTOR CLASSROOM              │
│                                                         │
│  Teacher's Laminated Sheets (Image Layers):             │
│  ┌────────────────────────────────────────┐            │
│  │  Sheet 4: COPY app.py /app/            │            │
│  │  (Your application code)               │            │
│  ├────────────────────────────────────────┤            │
│  │  Sheet 3: RUN pip install flask        │            │
│  │  (Dependencies)                        │            │
│  ├────────────────────────────────────────┤            │
│  │  Sheet 2: RUN apt-get install python3  │            │
│  │  (Python runtime)                      │            │
│  ├────────────────────────────────────────┤            │
│  │  Sheet 1: FROM ubuntu:22.04            │            │
│  │  (Base OS)                             │            │
│  └────────────────────────────────────────┘            │
│         All laminated (read-only, shared)              │
│                                                         │
│  Students' Personal Notepads (Container Layers):        │
│                                                         │
│  Student A:          Student B:          Student C:    │
│  ┌──────────┐       ┌──────────┐       ┌──────────┐   │
│  │ Notepad  │       │ Notepad  │       │ Notepad  │   │
│  │ • Modified│       │ • Created│       │ • Deleted│   │
│  │   configs│       │   temp   │       │   logs   │   │
│  └──────────┘       └──────────┘       └──────────┘   │
│       ↓                  ↓                  ↓          │
│  [Same laminated sheets underneath all three]          │
│                                                         │
│  Projector Screen (Merged View):                       │
│  Each student sees:                                     │
│  • Their notepad changes (on top)                       │
│  • Combined with all laminated sheets (below)           │
│  • Looks like one complete filesystem                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Key Insight**: The laminated sheets are **shared** by all students (containers), but each student's notepad is **unique**. This is how Docker runs 100 containers from one image efficiently!

---

## The Transparent Sheets Explained

### What Are These Sheets?

Think of building a presentation using transparent overhead projector sheets:

```
┌─────────────────────────────────────────────────────────┐
│        CREATING A PRESENTATION (Building an Image)      │
│                                                         │
│  Step 1: Take a blank transparent sheet                 │
│  ┌────────────────────────────────────┐                │
│  │ FROM ubuntu:22.04                  │                │
│  │ (Pre-made sheet with OS outline)   │                │
│  └────────────────────────────────────┘                │
│                                                         │
│  Step 2: Add another transparent sheet on top           │
│  ┌────────────────────────────────────┐                │
│  │ RUN apt-get install python3        │                │
│  │ (Draw Python installation diagram) │                │
│  └────────────────────────────────────┘                │
│                                                         │
│  Step 3: Add another sheet                              │
│  ┌────────────────────────────────────┐                │
│  │ RUN pip install flask              │                │
│  │ (Draw dependency tree)             │                │
│  └────────────────────────────────────┘                │
│                                                         │
│  Step 4: Add final sheet                                │
│  ┌────────────────────────────────────┐                │
│  │ COPY app.py /app/                  │                │
│  │ (Add your application code)        │                │
│  └────────────────────────────────────┘                │
│                                                         │
│  Now LAMINATE all sheets → Read-only, permanent!       │
│  These are your IMAGE LAYERS.                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Running Containers = Students Using the Presentation

```
┌─────────────────────────────────────────────────────────┐
│       STUDENTS WORKING WITH THE PRESENTATION            │
│                                                         │
│  10 students need to use the same presentation          │
│                                                         │
│  Option A: Make 10 complete copies                      │
│  ❌ 4 sheets × 10 students = 40 sheets                  │
│  ❌ Wasteful, expensive, slow                           │
│                                                         │
│  Option B: Smart sharing (Docker's approach)            │
│  ✅ 1 set of laminated sheets (shared)                  │
│  ✅ 10 blank notepad sheets (one per student)           │
│  ✅ 14 total sheets instead of 40!                      │
│                                                         │
│  Each student's setup:                                  │
│  ┌────────────────────────────────────┐                │
│  │ Student's Notepad (writable)       │ ← Unique       │
│  ├────────────────────────────────────┤                │
│  │ Sheet 4: app.py (laminated)        │ ←┐             │
│  │ Sheet 3: dependencies (laminated)  │  │             │
│  │ Sheet 2: python (laminated)        │  ├ Shared      │
│  │ Sheet 1: ubuntu (laminated)        │ ←┘             │
│  └────────────────────────────────────┘                │
│                                                         │
│  When student writes:                                   │
│  • Goes on their notepad (not on laminated sheets)     │
│  • Other students can't see it                          │
│  • Laminated sheets stay pristine                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## OverlayFS - The Magic Behind the Scenes

Docker uses a filesystem called **OverlayFS** (Overlay File System) that makes this layering possible.

### The Four Key Directories

```
┌─────────────────────────────────────────────────────────┐
│           OVERLAYFS MOUNTING EXPLAINED                  │
│                                                         │
│  1. lowerdir (The Laminated Sheets)                     │
│  ┌────────────────────────────────────┐                │
│  │ /layer1:/layer2:/layer3            │                │
│  │ Multiple read-only image layers    │                │
│  │ Shared by ALL containers           │                │
│  │ Example:                            │                │
│  │ /var/lib/docker/overlay2/l/ABC123  │                │
│  └────────────────────────────────────┘                │
│                                                         │
│  2. upperdir (Your Notepad)                             │
│  ┌────────────────────────────────────┐                │
│  │ /container-id/diff/                │                │
│  │ Read-write layer                   │                │
│  │ UNIQUE to this container           │                │
│  │ All modifications go here          │                │
│  │ Example:                            │                │
│  │ /var/lib/docker/overlay2/abc.../diff│               │
│  └────────────────────────────────────┘                │
│                                                         │
│  3. workdir (The Staging Table)                         │
│  ┌────────────────────────────────────┐                │
│  │ /container-id/work/                │                │
│  │ Internal OverlayFS scratch space   │                │
│  │ Used for atomic operations         │                │
│  │ Ensures crash-safe copying         │                │
│  └────────────────────────────────────┘                │
│                                                         │
│  4. merged (The Projector Screen)                       │
│  ┌────────────────────────────────────┐                │
│  │ /container-id/merged/              │                │
│  │ The unified view                   │                │
│  │ THIS is what container sees as /   │                │
│  │ Combination of upperdir + lowerdir │                │
│  └────────────────────────────────────┘                │
│                                                         │
│  Mount command:                                         │
│  mount -t overlay overlay \                             │
│    -o lowerdir=/layer1:/layer2,\                        │
│       upperdir=/container/diff,\                        │
│       workdir=/container/work,\                         │
│    /container/merged                                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Inside a Running Container

```bash
# Container sees this as its root filesystem:
/container-id/merged/
├─ bin/       # From layer 1 (ubuntu)
├─ etc/       # From multiple layers
├─ usr/       # From multiple layers
├─ app/       # From layer 4 (your app)
└─ tmp/       # From upperdir (your changes)

# But it's actually:
lowerdir (read-only):
  Layer 1: /bin, /usr, /etc (base OS)
  Layer 2: /usr/bin/python3
  Layer 3: /usr/lib/python3/flask/
  Layer 4: /app/app.py

upperdir (writable):
  /tmp/cache.db      # Created by container
  /etc/nginx.conf    # Modified by container
  /app/.env          # Created by container
```

---

## Copy-on-Write - Tracing from Laminated Sheets

When you need to modify something from a laminated sheet, you can't write on it directly. You must **trace it to your notepad first**.

```
┌─────────────────────────────────────────────────────────┐
│         COPY-ON-WRITE PROCESS                           │
│                                                         │
│  Scenario: Modify /etc/nginx/nginx.conf                │
│                                                         │
│  Step 1: Container wants to edit the file               │
│  ┌────────────────────────────────────┐                │
│  │ echo "worker_processes 4;" >>      │                │
│  │   /etc/nginx/nginx.conf            │                │
│  └────────────────────────────────────┘                │
│                                                         │
│  Step 2: OverlayFS checks upperdir                      │
│  ┌────────────────────────────────────┐                │
│  │ Check: upperdir/etc/nginx/         │                │
│  │ Result: nginx.conf NOT found       │                │
│  └────────────────────────────────────┘                │
│                                                         │
│  Step 3: OverlayFS searches lowerdir                    │
│  ┌────────────────────────────────────┐                │
│  │ Check layer 3: Found nginx.conf!   │                │
│  │ But it's LAMINATED (read-only)     │                │
│  │ Can't write on it directly         │                │
│  └────────────────────────────────────┘                │
│                                                         │
│  Step 4: Trigger Copy-on-Write                          │
│  ┌────────────────────────────────────┐                │
│  │ 4a: Place on staging table         │                │
│  │     lowerdir → workdir/#12345      │                │
│  │                                    │                │
│  │ 4b: Trace the file                 │                │
│  │     (Copy entire file content)     │                │
│  │                                    │                │
│  │ 4c: Make modifications             │                │
│  │     (Append new line)              │                │
│  │                                    │                │
│  │ 4d: Atomic placement               │                │
│  │     workdir → upperdir (all or nothing)│           │
│  └────────────────────────────────────┘                │
│                                                         │
│  Step 5: File now in upperdir                           │
│  ┌────────────────────────────────────┐                │
│  │ upperdir/etc/nginx/nginx.conf      │                │
│  │ (Modified version on YOUR notepad) │                │
│  │                                    │                │
│  │ lowerdir/etc/nginx/nginx.conf      │                │
│  │ (Original still on laminated sheet)│                │
│  └────────────────────────────────────┘                │
│                                                         │
│  Step 6: Future operations                              │
│  ┌────────────────────────────────────┐                │
│  │ All reads/writes go to upperdir    │                │
│  │ No more copying needed             │                │
│  │ Original in lowerdir ignored       │                │
│  └────────────────────────────────────┘                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Visual Representation:**

```
Before Modification:
┌─────────────────┐
│ Your Notepad    │  (empty)
├─────────────────┤
│ Sheet 3         │  nginx.conf (original)
│ (laminated)     │
└─────────────────┘

After Modification:
┌─────────────────┐
│ Your Notepad    │  nginx.conf (modified) ← You see this!
├─────────────────┤
│ Sheet 3         │  nginx.conf (original) ← Hidden/ignored
│ (laminated)     │
└─────────────────┘
```

**Key Insights:**

1. **Entire file is copied**, not just changed bytes
2. **Original remains unchanged** (other containers unaffected)
3. **Happens only once** per file (subsequent writes are direct)
4. **Atomic operation** (staging table ensures no corruption)
5. **Performance impact** for large files (use volumes for databases!)

---

## Why Deleting Doesn't Reduce Image Size

This is one of the most confusing concepts - let's visualize it:

```
┌─────────────────────────────────────────────────────────┐
│     THE IMMUTABLE LAYER PROBLEM                         │
│                                                         │
│  Dockerfile:                                            │
│  ┌────────────────────────────────────┐                │
│  │ FROM ubuntu:22.04          # 77MB  │                │
│  │ COPY bigfile.tar.gz /tmp/  # 250MB │ ← Sheet 1      │
│  │ RUN tar -xzf ...           # 250MB │ ← Sheet 2      │
│  │ RUN rm /tmp/bigfile.tar.gz # 0MB   │ ← Sheet 3      │
│  └────────────────────────────────────┘                │
│                                                         │
│  Visual:                                                │
│                                                         │
│  Sheet 3: RUN rm bigfile.tar.gz                         │
│  ┌────────────────────────────────────┐                │
│  │ Cover-up sticker:                  │                │
│  │ "⨯ /tmp/bigfile.tar.gz"            │ 0MB            │
│  │ (Whiteout file - marks as deleted) │                │
│  └────────────────────────────────────┘                │
│                                                         │
│  Sheet 2: RUN tar -xzf                                  │
│  ┌────────────────────────────────────┐                │
│  │ /app/extracted-files/              │ 250MB          │
│  └────────────────────────────────────┘                │
│                                                         │
│  Sheet 1: COPY bigfile.tar.gz                           │
│  ┌────────────────────────────────────┐                │
│  │ /tmp/bigfile.tar.gz                │ 250MB ← Still here!
│  └────────────────────────────────────┘                │
│                                                         │
│  Sheet 0: FROM ubuntu                                   │
│  ┌────────────────────────────────────┐                │
│  │ Ubuntu base files                  │ 77MB           │
│  └────────────────────────────────────┘                │
│                                                         │
│  Total when stacked: 577MB                              │
│  (Even though tar.gz looks "deleted"!)                  │
│                                                         │
│  Why? All sheets are LAMINATED.                         │
│  You can't erase from a laminated sheet!                │
│  You can only add cover-up stickers on top.             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**The Fix: Never Create the Sheet**

```dockerfile
# ❌ BAD: tar.gz becomes a permanent laminated sheet
COPY bigfile.tar.gz /tmp/
RUN tar -xzf /tmp/bigfile.tar.gz && rm bigfile.tar.gz

# ✅ GOOD: Download, extract, delete in ONE sheet
RUN curl -O https://example.com/bigfile.tar.gz && \
    tar -xzf bigfile.tar.gz && \
    rm bigfile.tar.gz
# Only extracted files make it onto the laminated sheet!
```

**Visual:**

```
Bad Approach (2 sheets):
┌─────────────────┐
│ Sheet 2: rm     │ (whiteout) 0MB
├─────────────────┤
│ Sheet 1: COPY   │ 250MB ← Wasted!
└─────────────────┘

Good Approach (1 sheet):
┌─────────────────┐
│ Sheet 1:        │ Only extracted files
│ curl + tar + rm │ 200MB (tar.gz never saved)
└─────────────────┘
```

---

## Layer Caching - Reusing Existing Sheets

Building images is like creating presentations. If you already have some sheets, reuse them!

```
┌─────────────────────────────────────────────────────────┐
│          LAYER CACHING EXAMPLE                          │
│                                                         │
│  First Build (Initial Presentation):                    │
│  ┌────────────────────────────────────┐                │
│  │ FROM ubuntu:22.04                  │ New sheet      │
│  │ RUN apt-get install python3        │ New sheet      │
│  │ COPY requirements.txt .            │ New sheet      │
│  │ RUN pip install -r requirements.txt│ New sheet      │
│  │ COPY app.py .                      │ New sheet      │
│  └────────────────────────────────────┘                │
│  Time: 5 minutes (creating 5 sheets)                    │
│                                                         │
│  Second Build (You changed app.py):                     │
│  ┌────────────────────────────────────┐                │
│  │ FROM ubuntu:22.04                  │ ✅ Reuse existing│
│  │ RUN apt-get install python3        │ ✅ Reuse existing│
│  │ COPY requirements.txt .            │ ✅ Reuse existing│
│  │ RUN pip install -r requirements.txt│ ✅ Reuse existing│
│  │ COPY app.py .                      │ ❌ Create new!  │
│  └────────────────────────────────────┘                │
│  Time: 5 seconds (only creating 1 new sheet)            │
│                                                         │
│  What Docker checks for cache hits:                     │
│  • Dockerfile instruction unchanged? ✓                  │
│  • Files being copied unchanged? (checksums)            │
│  • Parent layer unchanged? ✓                            │
│  • All match → CACHE HIT!                               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Cache Invalidation (Waterfall Effect):**

```
Dockerfile:
  Layer 1: FROM ubuntu          ✅ Cached
  Layer 2: RUN apt-get...       ✅ Cached
  Layer 3: COPY requirements.txt ✅ Cached
  Layer 4: RUN pip install...   ✅ Cached
  Layer 5: COPY app.py          ❌ Changed! (app.py modified)
  Layer 6: CMD python app.py    ❌ Rebuild (cache invalidated)
                                ↑
                        All layers below rebuild
                        (waterfall effect)
```

**Optimization Strategy:**

```dockerfile
# ❌ BAD: Changes to code invalidate dependency installation
FROM python:3.11
COPY . /app/                    # ← Copies EVERYTHING
RUN pip install -r requirements.txt
# Change any file → reinstall all dependencies!

# ✅ GOOD: Separate frequently-changing from rarely-changing
FROM python:3.11
COPY requirements.txt /app/     # ← Copy only dependencies file
RUN pip install -r /app/requirements.txt  # ← Cached unless requirements.txt changes
COPY . /app/                    # ← Copy code last
# Change code → dependencies still cached!
```

---

## Multi-Stage Builds - Two Projectors

Sometimes you need a messy workspace to build, but a clean presentation for the final result.

```
┌─────────────────────────────────────────────────────────┐
│          MULTI-STAGE BUILD ANALOGY                      │
│                                                         │
│  Projector 1 (Builder Stage):                           │
│  ┌────────────────────────────────────┐                │
│  │ Sheet 5: Compiler output           │ 100MB          │
│  │ Sheet 4: Build tools & temp files  │ 200MB          │
│  │ Sheet 3: Source code               │ 50MB           │
│  │ Sheet 2: Dependencies              │ 150MB          │
│  │ Sheet 1: Base OS                   │ 100MB          │
│  └────────────────────────────────────┘                │
│  Total: 600MB (messy, but we need it to build)         │
│                                                         │
│  Projector 2 (Final Stage):                             │
│  ┌────────────────────────────────────┐                │
│  │ Sheet 2: COPY --from=builder       │                │
│  │          /app/binary               │ 20MB           │
│  │          (Only compiled output)    │                │
│  │ Sheet 1: FROM alpine (minimal OS)  │ 5MB            │
│  └────────────────────────────────────┘                │
│  Total: 25MB (clean final image)                        │
│                                                         │
│  Projector 1 thrown away! ✅                            │
│  Only Projector 2 sheets kept.                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Real Example:**

```dockerfile
# ─────────────────────────────────────────
# Stage 1: Builder (Projector 1 - Messy)
# ─────────────────────────────────────────
FROM node:18 AS builder
WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm install              # Includes dev dependencies (500MB)

# Build application
COPY . .
RUN npm run build            # Compiles TypeScript → JavaScript

# Run tests (creates test artifacts)
RUN npm test

# Total builder stage: ~800MB

# ─────────────────────────────────────────
# Stage 2: Production (Projector 2 - Clean)
# ─────────────────────────────────────────
FROM node:18-alpine
WORKDIR /app

# Copy ONLY what's needed from builder
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package.json ./

# Total production image: ~150MB ✅ (5x smaller!)

CMD ["node", "dist/index.js"]
```

**What Gets Discarded:**

```
Builder stage (thrown away):
  ❌ Source code (.ts files)
  ❌ Dev dependencies (eslint, typescript, jest)
  ❌ Test files and artifacts
  ❌ Build tools and intermediates
  ❌ node_modules bloat (dev deps)

Final image (kept):
  ✅ Compiled code only (dist/)
  ✅ Production dependencies only
  ✅ Minimal base OS (alpine)
```

---

## Real-World Scenarios

### Scenario 1: Multiple Containers from Same Image

```
┌─────────────────────────────────────────────────────────┐
│     EFFICIENCY OF SHARED LAYERS                         │
│                                                         │
│  Image: nginx:alpine (5 laminated sheets)               │
│                                                         │
│  Running 100 containers:                                │
│                                                         │
│  Without Layer Sharing (hypothetical):                  │
│  100 containers × 5 sheets × 10MB = 5000MB             │
│  😱 5GB storage used!                                   │
│                                                         │
│  With OverlayFS Layer Sharing (Docker's approach):      │
│  1 set of laminated sheets (5 sheets × 10MB) = 50MB    │
│  + 100 notepads (100 × 0.1MB)           = 10MB         │
│  Total: 60MB                                            │
│  🎉 98.8% savings!                                      │
│                                                         │
│  Breakdown:                                             │
│  ┌────────────────────────────────────┐                │
│  │ Shared Image Layers (lowerdir):    │                │
│  │ /var/lib/docker/overlay2/l/ABC...  │ 50MB           │
│  │ (One copy, referenced 100 times)   │                │
│  │                                    │                │
│  │ Container 1 (upperdir):            │                │
│  │ /overlay2/container1/diff/         │ 0.1MB          │
│  │                                    │                │
│  │ Container 2 (upperdir):            │                │
│  │ /overlay2/container2/diff/         │ 0.1MB          │
│  │                                    │                │
│  │ ... (98 more) ...                  │                │
│  │                                    │                │
│  │ Container 100 (upperdir):          │                │
│  │ /overlay2/container100/diff/       │ 0.1MB          │
│  └────────────────────────────────────┘                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Scenario 2: Pull Optimization

```
┌─────────────────────────────────────────────────────────┐
│     PULLING IMAGES WITH LAYER DEDUPLICATION             │
│                                                         │
│  You have: nginx:1.21 (5 layers, 50MB total)           │
│                                                         │
│  Pull: nginx:1.22                                       │
│                                                         │
│  Layer comparison:                                      │
│  nginx:1.21          nginx:1.22          Action         │
│  ─────────────────────────────────────────────────      │
│  Layer 1: sha256:abc  Layer 1: sha256:abc  ✅ Reuse!   │
│  Layer 2: sha256:def  Layer 2: sha256:def  ✅ Reuse!   │
│  Layer 3: sha256:ghi  Layer 3: sha256:ghi  ✅ Reuse!   │
│  Layer 4: sha256:jkl  Layer 4: sha256:jkl  ✅ Reuse!   │
│  Layer 5: sha256:mno  Layer 5: sha256:xyz  ⬇️  Download│
│                                                         │
│  Pull summary:                                          │
│  • 4 layers already exist (40MB) - skipped              │
│  • 1 layer new (10MB) - downloaded                      │
│  • Total download: 10MB instead of 50MB                 │
│  • 80% bandwidth saved!                                 │
│                                                         │
│  How Docker knows:                                      │
│  1. Downloads manifest (list of layer SHA256 hashes)    │
│  2. Checks local storage for each hash                  │
│  3. Only downloads missing layers                       │
│  4. Content-addressable: Same content = Same hash       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Scenario 3: Developer Workflow

```
┌─────────────────────────────────────────────────────────┐
│     DAILY DEVELOPMENT CYCLE                             │
│                                                         │
│  Morning: Build image                                   │
│  ┌────────────────────────────────────┐                │
│  │ FROM python:3.11         5min      │                │
│  │ RUN apt-get install...   3min      │                │
│  │ COPY requirements.txt    1sec      │                │
│  │ RUN pip install...       2min      │                │
│  │ COPY app.py              1sec      │                │
│  └────────────────────────────────────┘                │
│  Total: 10 minutes                                      │
│                                                         │
│  Afternoon: Fixed bug in app.py                         │
│  ┌────────────────────────────────────┐                │
│  │ FROM python:3.11         ✅ cached │                │
│  │ RUN apt-get install...   ✅ cached │                │
│  │ COPY requirements.txt    ✅ cached │                │
│  │ RUN pip install...       ✅ cached │                │
│  │ COPY app.py              2sec      │ ← Only this!   │
│  └────────────────────────────────────┘                │
│  Total: 2 seconds! 🚀                                   │
│                                                         │
│  Without caching:                                       │
│  10 minutes × 20 builds/day = 200 minutes wasted       │
│                                                         │
│  With caching:                                          │
│  2 seconds × 20 builds/day = 40 seconds                 │
│                                                         │
│  Time saved per day: 199 minutes (3.3 hours!)          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Common Pitfalls

### Pitfall 1: Invalidating Cache Too Early

```dockerfile
# ❌ BAD: Copying everything invalidates cache on any file change
FROM node:18
COPY . /app/                     # ← Copies package.json + code
WORKDIR /app
RUN npm install                  # ← Reinstalls on every code change

# ✅ GOOD: Copy dependencies separately
FROM node:18
WORKDIR /app
COPY package*.json ./            # ← Copy only dependency files
RUN npm install                  # ← Cached unless package.json changes
COPY . .                         # ← Copy code last
```

**Visual:**

```
Bad approach:
┌─────────────────┐
│ Sheet 3: npm    │ ← Rebuilds when any file changes
├─────────────────┤
│ Sheet 2: COPY . │ ← Invalidated by code change
├─────────────────┤
│ Sheet 1: FROM   │
└─────────────────┘

Good approach:
┌─────────────────┐
│ Sheet 4: COPY . │ ← Only this rebuilds
├─────────────────┤
│ Sheet 3: npm    │ ← Cached!
├─────────────────┤
│ Sheet 2: package│ ← Cached!
├─────────────────┤
│ Sheet 1: FROM   │
└─────────────────┘
```

### Pitfall 2: Not Using .dockerignore

```
Without .dockerignore:
COPY . /app/
Copies:
  ✓ Source code (needed)
  ❌ node_modules/ (500MB, unnecessary)
  ❌ .git/ (100MB, unnecessary)
  ❌ build/ (cached artifacts)
  ❌ *.log (log files)

Result: Cache invalidated by irrelevant files!
```

```bash
# .dockerignore (like .gitignore for Docker)
node_modules/
.git/
*.log
.env
build/
dist/
__pycache__/
```

### Pitfall 3: Large Files in Early Layers

```dockerfile
# ❌ BAD: Large file in early layer
FROM ubuntu:22.04
COPY huge-dataset.tar.gz /tmp/     # ← 5GB in layer 2
RUN tar -xzf /tmp/huge-dataset.tar.gz
RUN rm /tmp/huge-dataset.tar.gz    # ← Doesn't help! Still in layer 2

# Every change to later layers still includes 5GB tar.gz!

# ✅ GOOD: Multi-stage or download in RUN
FROM ubuntu:22.04
RUN curl -O https://example.com/huge-dataset.tar.gz && \
    tar -xzf huge-dataset.tar.gz && \
    rm huge-dataset.tar.gz
# tar.gz never saved to any layer!
```

### Pitfall 4: Too Many Layers vs Too Few

```dockerfile
# ❌ BAD: Every command = separate layer (50+ layers)
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN apt-get install -y vim
RUN apt-get install -y git
# ... 45 more RUN commands ...

# ❌ ALSO BAD: Everything in one massive layer (no caching)
RUN apt-get update && apt-get install -y \
    curl wget vim git python3 gcc g++ make cmake \
    ... 100 packages ... && \
    pip install ... 50 packages ... && \
    npm install ... 30 packages ... && \
    gem install ... 20 gems ...
# Change one thing → rebuild everything!

# ✅ GOOD: Logical grouping (5-10 layers)
RUN apt-get update && apt-get install -y \
    curl wget vim git \
    && rm -rf /var/lib/apt/lists/*

RUN pip install -r requirements.txt

RUN npm install

COPY . .
```

### Pitfall 5: Not Understanding Ephemeral Container Layer

```bash
# ❌ BAD: Storing data in container layer
docker run -d myapp
docker exec myapp bash -c "echo 'important' > /app/data.txt"
docker stop myapp
docker rm myapp
# 💀 data.txt is GONE! (container layer deleted)

# ✅ GOOD: Use volumes for persistent data
docker run -d -v mydata:/app/data myapp
docker exec myapp bash -c "echo 'important' > /app/data/data.txt"
docker stop myapp
docker rm myapp
# ✅ data.txt survives in volume!
```

**Visual:**

```
Container lifecycle:
┌─────────────────┐
│ Your Notepad    │ ← Created on docker run
│ (container layer)│
├─────────────────┤
│ Laminated sheets│ ← Permanent
│ (image layers)  │
└─────────────────┘
     docker rm
        ↓
┌─────────────────┐
│ Notepad GONE!   │ ← All changes lost!
├─────────────────┤
│ Laminated sheets│ ← Still exist
└─────────────────┘
```

---

## Key Takeaways

1. **Image layers are transparent sheets** - Stack on top of each other, combine to create the full filesystem view

2. **Container layer is your notepad** - Writable, unique per container, sits on top of shared image layers

3. **OverlayFS provides the magic** - lowerdir (image), upperdir (container), workdir (staging), merged (final view)

4. **Copy-on-Write is tracing** - Copy file from laminated sheet to notepad before modifying

5. **Deletion doesn't shrink images** - Laminated sheets are permanent, you can only add cover-up stickers

6. **Layer caching is reusing sheets** - Don't recreate what you already have; organize Dockerfile by change frequency

7. **Multi-stage builds are two projectors** - Build messy, copy clean; discard the mess

8. **Shared layers save space** - 100 containers sharing 5 image layers = massive efficiency

9. **Atomic operations via workdir** - Staging table ensures crash-safe copying

10. **Container layer is ephemeral** - Your notepad disappears with the container; use volumes for data

---

## Optimization Checklist

**Dockerfile Best Practices:**

```dockerfile
# ✅ Use specific base image versions
FROM python:3.11-slim          # Not 'latest'

# ✅ Combine related commands
RUN apt-get update && apt-get install -y \
    package1 package2 \
    && rm -rf /var/lib/apt/lists/*

# ✅ Order by change frequency (rarely → frequently)
COPY requirements.txt .        # Changes rarely
RUN pip install -r requirements.txt
COPY . .                       # Changes often

# ✅ Use .dockerignore
# Create .dockerignore with irrelevant files

# ✅ Use multi-stage for compiled languages
FROM golang:1.21 AS builder
COPY . .
RUN go build -o app

FROM alpine:latest
COPY --from=builder /app /app

# ✅ Don't store secrets in layers
# Use build secrets or runtime injection

# ✅ Clean up in same layer
RUN curl -O file.tar.gz && \
    tar -xzf file.tar.gz && \
    rm file.tar.gz           # ← Same RUN

# ✅ Use volumes for data
VOLUME /app/data
```

---

## A Note on Modern Docker (Engine 29.0+)

**Important Update**: As of Docker Engine 29.0 (released in 2024), Docker has transitioned to a new default storage architecture:

```
┌─────────────────────────────────────────────────────────┐
│         DOCKER STORAGE EVOLUTION                        │
│                                                         │
│  Legacy (Docker < 29.0):                                │
│  ┌────────────────────────────────────┐                │
│  │ Docker Daemon                      │                │
│  │ └─ overlay2 storage driver         │                │
│  │    (Direct OverlayFS management)   │                │
│  └────────────────────────────────────┘                │
│                                                         │
│  Modern (Docker ≥ 29.0):                                │
│  ┌────────────────────────────────────┐                │
│  │ containerd image store (default)   │                │
│  │ └─ overlayfs snapshotter           │                │
│  │    (Containerd-managed layers)     │                │
│  └────────────────────────────────────┘                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**What Changed:**

- **Old**: Docker daemon directly managed image layers using the `overlay2` storage driver
- **New**: Docker delegates image and layer management to `containerd` with the `overlayfs` snapshotter
- **Impact**: More efficient, better integration with Kubernetes and other container runtimes, improved performance

**What Stays the Same:**

✅ All concepts in this TIL remain valid:
- Layered filesystem architecture
- Copy-on-Write mechanism  
- OverlayFS directories (lowerdir, upperdir, workdir, merged)
- Layer caching and optimization strategies
- Multi-stage builds and efficiency patterns

**Why This Matters:**

The `containerd` image store provides:
- **Better performance**: Optimized snapshot management
- **Standardization**: Same storage backend used by Kubernetes (CRI)
- **Improved reliability**: Better garbage collection and layer management
- **Future-proof**: Aligns with industry direction (OCI standards)

**Visual Analogy:**

```
Think of it like this:

Old way (overlay2):
  Teacher manages the laminated sheets directly
  (Docker daemon handles OverlayFS mounting)

New way (containerd + overlayfs snapshotter):
  Library manages all sheets, teacher requests what's needed
  (Containerd handles storage, Docker coordinates)
```

**For Developers:**

You don't need to change anything! Your Dockerfiles, build commands, and workflows remain identical. This is an internal architectural improvement that's transparent to users.

**Learn More:**
- [Containerd Image Store Documentation](https://docs.docker.com/engine/storage/containerd/)
- Migration from overlay2 to containerd snapshotter happens automatically on upgrade

---

## Glossary (Analogy Terms)

For readers unfamiliar with overhead projector technology:

**Overhead Projector (OHP)**: A presentation device popular from 1960s-2000s that projects images from transparent sheets onto a wall or screen. Light shines up through the sheets, and a mirror reflects the image forward. Common in classrooms and business meetings before digital projectors.

**Transparency Sheet (Acetate Sheet)**: A thin, clear plastic sheet (like rigid plastic wrap) that you can write on or print images on. When placed on an overhead projector, the content becomes visible on the screen. Similar to how modern presentation slides work, but physical.

**Laminated Sheet**: A transparency sheet sealed between protective plastic layers using heat/pressure. Once laminated, the content is **permanent and cannot be changed** - you cannot write on it or erase it. Used for content you want to preserve and reuse.

**Notepad Sheet**: A blank, writable sheet of paper or transparency where you can write temporary notes or modifications. Unlike laminated sheets, you can erase and change content freely.

**Stacking Sheets**: Placing multiple transparent sheets on top of each other. When projected, all sheets are visible simultaneously as one combined image. This allowed teachers to build complex diagrams layer by layer.

**Tracing**: Copying content from one sheet to another by placing a blank sheet over the original and drawing over it. This is how you "modify" content from a laminated sheet - copy it to your notepad and make changes.

**Projector Screen**: A white wall or special screen where the projected image appears, making it visible to the audience.

**Staging Table**: In the analogy, refers to a temporary workspace where you prepare materials before the final placement (matches Docker's workdir concept).

*Note: If you've never seen an overhead projector, imagine a glass table with a bright light underneath and a mirror/lens above. You place transparent sheets on the glass, light shines through them, and the mirror projects the combined image onto a wall - like a physical version of PowerPoint slides that you can stack and layer.*

---

## Related Reading

- [Containers Fundamentals - The Prison Analogy](./2025-12-23-containers-fundamentals.md)
- [Docker Architecture - The Construction Company Analogy](./2025-12-25-docker-architecture.md)

---

*Written on December 28, 2025*
