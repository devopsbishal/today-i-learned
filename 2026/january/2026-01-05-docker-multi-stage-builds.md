# Docker Multi-stage Builds - The Restaurant Kitchen Analogy

> Understanding how to build tiny, secure Docker images: from messy kitchens with all the tools to elegant dining rooms serving only the final dish.

---

## TL;DR

| Docker Concept | Restaurant Kitchen Analogy |
|----------------|----------------------------|
| **Single-stage build** | Serving customers IN the kitchen (messy, unsafe, huge) |
| **Multi-stage build** | Separate kitchen (builder) and dining room (runtime) |
```
┌─────────────────────────────────────────────────────────┐
│    🏢 THE RESTAURANT BUILDING (Docker Build Process)    │
│                                                         │
│  KITCHEN (Builder Stage) - 1000 sq ft                   │
│  ┌────────────────────────────────────────────┐        │
│  │  🔪 Professional Equipment:                │        │
│  │  • Industrial oven (gcc compiler)          │        │
│  │  • Food processor (webpack)                │        │
│  │  • Mixers (npm, pip)                       │        │
│  │  • Sharp knives (build tools)              │        │
│  │                                            │        │
│  │  📦 Raw Ingredients:                       │        │
│  │  • Source code (.ts, .go, .py)            │        │
│  │  • 500MB node_modules                      │        │
│  │  • Dev dependencies                        │        │
│  │                                            │        │
│  │  👨‍🍳 Chef at Work:                           │        │
│  │  $ npm run build                           │        │
│  │  $ go build -o app                         │        │
│  │                                            │        │
│  │  Result: ONE BEAUTIFUL DISH ✨              │        │
│  │  (Compiled binary or bundled JS)          │        │
│  └────────────────────────────────────────────┘        │
│                      │                                  │
│                      │ COPY --from=builder              │
│                      │ (Plate the dish)                 │
│                      ↓                                  │
│  ┌────────────────────────────────────────────┐        │
│  │  DINING ROOM (Runtime Stage) - 200 sq ft  │        │
│  │                                            │        │
│  │  🪑 Minimal, Elegant:                      │        │
│  │  • Table                                   │        │
│  │  • Chair                                   │        │
│  │  • Plate with FINISHED DISH                │        │
│  │  • Fork and knife                          │        │
│  │                                            │        │
│  │  ❌ NO kitchen equipment                   │        │
│  │  ❌ NO raw ingredients                     │        │
│  │  ❌ NO food scraps                         │        │
│  │  ✅ ONLY what customer needs               │        │
│  │                                            │        │
│  │  Size: 150MB (87% smaller!)               │        │
│  └────────────────────────────────────────────┘        │
│                                                         │
│  💰 You Pay Rent On: Dining Room ONLY                  │
│     (Kitchen is discarded after cooking!)              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**The Problem We're Solving:**
- Traditional Dockerfiles include everything: build tools + runtime needs
- This creates bloated images (1GB+) full of security vulnerabilities
- You're shipping TypeScript compilers to production (why?!)
- Every deployment pulls gigabytes of unnecessary tools

**The Solution:**
- **Stage 1 (Builder/Kitchen):** Install build tools, compile code, generate artifacts
- **Stage 2 (Runtime/Dining Room):** Copy ONLY the final artifacts, skip everything else
- Final image: Clean, minimal, production-ready

**Key Insight**: Just like you don't serve customers in the kitchen, you don't ship build tools to production deployment time |
| **ubuntu:22.04** | Full-service restaurant (77MB, everything) |
| **alpine:3.19** | Casual dining (7MB, most common) |
| **distroless** | Fine dining (2MB, curated, secure) |
| **scratch** | Food truck (0MB, just the binary!) |
| **Layer caching** | Prep station organization (stable ingredients first) |
| **Three-stage pattern** | Prep kitchen + Cooking kitchen + Dining room |
| **.dockerignore** | "Don't bring trash into the kitchen" |
| **BuildKit cache mount** | Shared pantry that stays stocked between services |


---

## 🍽️ The Restaurant Kitchen Analogy

### **Multi-stage Builds = Restaurant Kitchen vs Dining Room**

Imagine you're running a high-end restaurant. Where do you serve your customers?

#### **The Wrong Way (Single-Stage Build):**

You seat customers in the **KITCHEN** 🤦

```
CUSTOMER'S VIEW:
├─ Dirty pots and pans everywhere
├─ Raw chicken on the counter
├─ Chef's knives, mixers, food processors
├─ Garbage cans full of food scraps
├─ Grease-covered stove
└─ Your beautiful dish... somewhere in the mess
```

**Problems:**
- ❌ Takes up 1000 sq ft (huge space cost!)
- ❌ Unsafe (customer near knives, hot stoves)
- ❌ Unprofessional (messy, cluttered)
- ❌ Health code violations (raw food near cooked food)
- ❌ Customer experience: Terrible!

**Docker Equivalent:**
```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install        # Installs 500MB of dependencies
COPY . .
RUN npm run build      # TypeScript compiler, webpack
CMD ["node", "dist/server.js"]

# Result: 1.2GB image with build tools in production 🤦
```

---

#### **The Right Way (Multi-Stage Build):**

**Stage 1 - The KITCHEN (Builder)** 🔪

```
THE KITCHEN (NO CUSTOMERS ALLOWED):
├─ Chef prepares the meal
├─ Uses all professional equipment:
│   ├─ Industrial oven
│   ├─ Food processor
│   ├─ Mixers, blenders
│   └─ Sharp knives
├─ Makes a mess (that's okay!)
├─ Generates food scraps
└─ Produces: ONE BEAUTIFUL DISH ✨
```

**Stage 2 - The DINING ROOM (Runtime)** 🪑

```
THE DINING ROOM (WHERE CUSTOMERS DINE):
├─ Clean, elegant space
├─ Only essentials:
│   ├─ Table
│   ├─ Chair
│   ├─ Plate with the finished dish
│   └─ Fork and knife
├─ No cooking equipment visible
├─ No raw ingredients
└─ Perfect presentation
```

**The Key Insight:**
- Kitchen is 1000 sq ft (expensive!)
- Dining room is 200 sq ft (efficient!)
- **You pay rent on the dining room, NOT the kitchen** 💰
- Customer never sees the mess

---

### **The Docker Parallel:**

| Restaurant Concept | Docker Multi-stage |
|--------------------|-------------------|
| **Raw ingredients** | Source code (`.ts`, `.go`, `.py`) |
| **Kitchen tools** | Build tools (npm, gcc, maven, pip) |
| **Food processor** | TypeScript compiler, webpack |
| **Chef cooking** | `npm run build`, `go build` |
| **Food scraps** | `node_modules`, build cache, temp files |
| **Dirty dishes** | Dev dependencies, test frameworks |
| **Finished dish** | Compiled binary, bundled JavaScript |
| **Dining room** | Production container (runtime stage) |
| **Customer** | End user, Kubernetes cluster |
| **Monthly rent** | Registry storage costs, pull time |

---

### **The Dockerfile Translation:**

```dockerfile
# ============================================
# STAGE 1: THE KITCHEN (Builder Stage)
# ============================================
FROM node:18 AS kitchen

WORKDIR /restaurant

# Bring in raw ingredients
COPY package*.json ./

# Install ALL kitchen equipment (dev tools)
RUN npm install  # 500MB of cooking tools!

# Bring in the recipe (source code)
COPY . .

# CHEF COOKS! (Compilation)
RUN npm run build  # TypeScript → JavaScript

# Kitchen is now MESSY (1.2GB of stuff)
# But we have our finished dish in /dist 🍽️


# ============================================
# STAGE 2: THE DINING ROOM (Runtime Stage)
# ============================================
FROM node:18-alpine AS dining_room

WORKDIR /table

# Bring ONLY the finished dish from the kitchen
COPY --from=kitchen /restaurant/dist ./dist

# Bring minimal utensils (production dependencies only)
COPY --from=kitchen /restaurant/node_modules ./node_modules

# Serve to customer
CMD ["node", "dist/server.js"]

# Dining room: Clean, 150MB 🎉
# No kitchen tools, no raw ingredients!
```

---

### **The Real-World Impact:**

**Scenario:** You run a restaurant chain with 50 locations

**Option A: Customers eat in the kitchen**
- Space cost: 1000 sq ft × 50 = 50,000 sq ft rent
- Safety issues: Knives, hot stoves near customers
- Cleanliness: Constant mess visible
- **Monthly cost: $50,000/month**

**Option B: Separate kitchen and dining room**
- Kitchen: 800 sq ft (back of house)
- Dining room: 200 sq ft × 50 = 10,000 sq ft rent
- Safety: Customers never near hazards
- Cleanliness: Always pristine
- **Monthly cost: $10,000/month**
- **Savings: $40,000/month = $480,000/year** 💰

**Docker Translation:**
- Without multi-stage: 1.2GB × 50 microservices = 60GB storage
- With multi-stage: 150MB × 50 = 7.5GB storage
- **Registry cost savings: ~$400/month**
- **Deployment speed: 5 min → 30 sec (10x faster!)**
- **Security: 237 CVEs → 3 CVEs (99% reduction)**

---

## 🔧 Core Concepts

### 1. **The Multi-Stage Pattern**

```dockerfile
# Pattern:
FROM <base> AS <stage_name>
# ... do work ...
COPY --from=<stage_name> <source> <dest>
```

**Example:**
```dockerfile
# Stage 1: Build
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Stage 2: Runtime  
FROM alpine:3.19
COPY --from=builder /app/myapp /usr/local/bin/
CMD ["myapp"]
```

**What happens:**
1. Docker builds `builder` stage (creates large temporary image)
2. Docker builds final stage (starts fresh from alpine)
3. Only `COPY --from=builder` brings artifacts forward
4. Builder stage is **discarded** (not in final image!)

---

### 2. **Three Common Patterns**

#### **Pattern A: Compiled Languages (Go, Rust, C++)**

**Kitchen:** Full development environment with compilers
**Dining Room:** Tiny runtime (often just the binary!)

```dockerfile
# KITCHEN: Compile static binary
FROM golang:1.21 AS builder
COPY . .
RUN CGO_ENABLED=0 go build -o app

# DINING ROOM: Binary only!
FROM scratch
COPY --from=builder /app/app /app
CMD ["/app"]
```

**Result:** 1GB build → 8MB runtime (99% reduction!)

---

#### **Pattern B: Interpreted Languages (Node, Python)**

**Kitchen:** Install ALL dependencies (dev + prod)
**Dining Room:** Only production dependencies + compiled code

```dockerfile
# KITCHEN: Build with dev tools
FROM node:18 AS builder
COPY package*.json ./
RUN npm install  # Dev + prod dependencies
COPY . .
RUN npm run build  # TypeScript compilation

# DINING ROOM: Runtime only
FROM node:18-alpine AS runtime
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/server.js"]
```

**Result:** 1.2GB → 150MB (87% reduction)

---

#### **Pattern C: Multiple Builders (Frontend + Backend)**

**Two Kitchens, One Dining Room!**

```dockerfile
# KITCHEN 1: Build React frontend
FROM node:18 AS frontend
WORKDIR /frontend
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ .
RUN npm run build  # → /frontend/build

# KITCHEN 2: Build Go backend
FROM golang:1.21 AS backend
WORKDIR /backend
COPY backend/ .
RUN go build -o server

# DINING ROOM: Serve both!
FROM nginx:alpine
COPY --from=frontend /frontend/build /usr/share/nginx/html
COPY --from=backend /backend/server /usr/local/bin/
COPY nginx.conf /etc/nginx/nginx.conf
CMD ["sh", "-c", "server & nginx -g 'daemon off;'"]
```

**Result:** React (300MB) + Go (400MB) → Single 45MB image!

---

### 3. **Base Image Selection (The Restaurant Grade)**

| Base Image | Size | Use Case | Restaurant Analogy |
|------------|------|----------|-------------------|
| **ubuntu:22.04** | 77MB | Maximum compatibility | Full-service restaurant |
| **alpine:3.19** | 7MB | Most common choice | Casual dining |
| **distroless** | 2MB | Security-focused | Fine dining (curated) |
| **scratch** | 0MB | Static binaries only | Food truck (minimal!) |

**Example:**
```dockerfile
# Ubuntu: Full restaurant
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y python3
# Result: 200MB

# Alpine: Casual dining
FROM python:3.11-alpine
# Result: 50MB

# Distroless: Fine dining
FROM gcr.io/distroless/python3
# Result: 25MB

# Scratch: Food truck
FROM scratch
COPY --from=builder /app/static-binary /app
# Result: 8MB (just the binary!)
```

---

### 4. **The COPY --from Syntax**

**The Magic Command:**
```dockerfile
COPY --from=<stage_name> <source_in_stage> <destination>
```

**Examples:**

```dockerfile
# Copy from named stage
COPY --from=builder /app/dist ./dist

# Copy from numbered stage (stage 0)
COPY --from=0 /app/binary /usr/local/bin/

# Copy from external image!
COPY --from=nginx:latest /etc/nginx/nginx.conf /etc/nginx/
```

**Restaurant analogy:**
- `builder` stage = Kitchen
- `COPY --from=builder` = Plating the finished dish
- Destination = Dining room table

---

## 🌍 Real-World Examples

### **Example 1: TypeScript Node.js API (1.2GB → 150MB)**

**Before (Single-stage - Kitchen dining):**
```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
CMD ["node", "dist/server.js"]

# Result: 1.2GB with TypeScript compiler in production 🤦
```

**After (Multi-stage - Separate kitchen/dining):**
```dockerfile
# KITCHEN: Build environment
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci  # Clean install
COPY tsconfig.json ./
COPY src ./src
RUN npm run build  # TypeScript → JavaScript

# DINING ROOM: Production runtime
FROM node:18-alpine
WORKDIR /app

# Create non-root user (safety!)
RUN addgroup -g 1001 nodejs && \
    adduser -S nodejs -u 1001 -G nodejs

# Copy only production dependencies
COPY --from=builder /app/package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy compiled code
COPY --from=builder /app/dist ./dist

USER nodejs
EXPOSE 3000
CMD ["node", "dist/server.js"]

# Result: 150MB, no dev tools, secure ✅
```

**Impact:**
- Size: 1.2GB → 150MB (87% smaller)
- Vulnerabilities: 237 → 3 (99% reduction)
- Deploy time: 8 min → 2 min (75% faster)
- Cost: $180/month → $15/month (registry storage)

---

### **Example 2: Go Static Binary (700MB → 8MB)**

```dockerfile
# KITCHEN: Build with full Go toolchain
FROM golang:1.21-alpine AS builder
WORKDIR /app

COPY go.* ./
RUN go mod download

COPY . .

# Compile static binary (no dependencies!)
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s" \  # Strip debug info
    -o myapp

# DINING ROOM: Just the binary!
FROM scratch

# Copy SSL certs (for HTTPS)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy binary
COPY --from=builder /app/myapp /myapp

CMD ["/myapp"]

# Result: 8MB total! 🎉
```

**Why scratch works:**
- Go compiles to static binary (no runtime needed)
- Binary has everything (like a pre-packaged meal!)
- No shell, no OS, just the app

---

### **Example 3: Python with Optimized Dependencies**

```dockerfile
# KITCHEN 1: Build dependencies
FROM python:3.11 AS deps
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# KITCHEN 2: Build application
FROM python:3.11 AS builder
WORKDIR /app
COPY . .
RUN python -m compileall .

# DINING ROOM: Minimal runtime
FROM python:3.11-slim
WORKDIR /app

# Copy installed packages from deps kitchen
COPY --from=deps /root/.local /root/.local

# Copy compiled Python bytecode
COPY --from=builder /app /app

# Update PATH for pip packages
ENV PATH=/root/.local/bin:$PATH

CMD ["python", "app.py"]

# Result: 800MB → 200MB
```

---

### **Example 4: Java Spring Boot (500MB → 180MB)**

```dockerfile
# KITCHEN: Build with Maven
FROM maven:3.8-openjdk-17 AS builder
WORKDIR /app

# Cache dependencies layer
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Build application
COPY src ./src
RUN mvn package -DskipTests -B

# DINING ROOM: JRE only (no JDK!)
FROM openjdk:17-jre-slim
WORKDIR /app

# Copy only the JAR
COPY --from=builder /app/target/*.jar app.jar

# Run with optimized JVM flags
CMD ["java", "-XX:+UseContainerSupport", "-jar", "app.jar"]

# Result: 180MB (vs 500MB with full JDK)
```

---

## ✅ Best Practices (Restaurant Operations Manual)

### 1. **Order Matters (Maximize Layer Caching)**

**❌ Bad Kitchen Layout:**
```dockerfile
FROM node:18 AS builder
COPY . .  # ❌ Copies EVERYTHING first (cache bust!)
RUN npm install  # Re-runs even if package.json unchanged
RUN npm run build
```

**✅ Good Kitchen Layout:**
```dockerfile
FROM node:18 AS builder
# Copy recipe first (rarely changes)
COPY package*.json ./
RUN npm install  # ✅ Cached if package.json unchanged!

# Then copy ingredients (changes often)
COPY . .
RUN npm run build  # Only re-runs if source changed
```

**Why:** Like a restaurant prep station - ingredients that don't change often go in the back!

---

### 2. **Three-Stage Pattern (Separate Production Dependencies)**

**The Full Kitchen Flow:**

```dockerfile
# STAGE 1: Production dependencies only
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# STAGE 2: Build with dev dependencies
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci  # Installs ALL dependencies
COPY . .
RUN npm run build

# STAGE 3: Runtime (copy from both!)
FROM node:18-alpine
WORKDIR /app

# Production dependencies from Stage 1
COPY --from=deps /app/node_modules ./node_modules

# Built code from Stage 2
COPY --from=builder /app/dist ./dist

CMD ["node", "dist/server.js"]
```

**Why separate:**
- Stage 1: Clean prod dependencies (no webpack, no jest)
- Stage 2: Can use dev tools (eslint, typescript)
- Stage 3: Gets best of both worlds

---

### 3. **Security Hardening (Food Safety)**

```dockerfile
FROM node:18-alpine

# Don't run as root! (Health code violation)
RUN addgroup -g 1001 nodejs && \
    adduser -S nodejs -u 1001 -G nodejs

# Set ownership
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist

# Switch to non-root
USER nodejs

# Only expose necessary ports
EXPOSE 3000

CMD ["node", "dist/server.js"]
```

---

### 4. **Use .dockerignore (Keep Kitchen Clean)**

```
# .dockerignore - Don't bring this into the kitchen!
node_modules
npm-debug.log
.git
.env
*.md
tests/
.vscode/
coverage/
```

**Restaurant analogy:** Don't bring trash cans into the kitchen!

---

### 5. **BuildKit Cache Mounts (Reusable Ingredients)**

```dockerfile
# Mount package manager cache (persist across builds)
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Like having a pantry that stays stocked between services!
```

**Benefit:**
- Without: Download 250MB every build
- With: Download only new packages (~5MB)
- 95% faster dependency installation!

---

## ⚠️ Common Pitfalls (Kitchen Disasters)

### **Pitfall 1: Forgetting --from in COPY**

```dockerfile
FROM golang:1.21 AS builder
RUN go build -o app

FROM alpine
COPY app /app  # ❌ Where is app? Not in alpine!
CMD ["/app"]

# Fix:
COPY --from=builder /app/app /app  # ✅
```

**Restaurant analogy:** Trying to serve a dish from the OTHER restaurant's kitchen!

---

### **Pitfall 2: Using Wrong Base for Runtime**

```dockerfile
# Building Go app
FROM golang:1.21 AS builder
RUN go build -o app

# ❌ Wrong! Still using full Go image as runtime
FROM golang:1.21
COPY --from=builder /app/app /app

# ✅ Better! Go binary doesn't need Go runtime
FROM alpine:3.19
COPY --from=builder /app/app /app
```

---

### **Pitfall 3: Not Pinning Versions**

```dockerfile
FROM node:18  # ❌ Which 18? Could change tomorrow!

# ✅ Better:
FROM node:18.19-alpine3.19  # Specific version
```

**Restaurant analogy:** "Get me some flour" vs "Get me 2 cups of King Arthur all-purpose flour"

---

### **Pitfall 4: Copying Too Much**

```dockerfile
# ❌ Brings EVERYTHING from builder
COPY --from=builder /app /app

# ✅ Be specific!
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
```

---

### **Pitfall 5: Scratch Image Without Essentials**

```dockerfile
FROM scratch
COPY --from=builder /app/app /app
CMD ["/app"]

# ❌ Crashes on HTTPS! No CA certificates
```

**Fix:**
```dockerfile
FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/app /app
```

**Or better yet:**
```dockerfile
# Use distroless (includes essentials)
FROM gcr.io/distroless/static-debian11
COPY --from=builder /app/app /app
```

---

## 🎯 When to Use Multi-stage Builds

### **✅ Perfect For:**

1. **Compiled Languages** (Go, Rust, C++, Java)
   - Compile with full toolchain
   - Ship tiny runtime

2. **Transpiled Languages** (TypeScript, CoffeeScript)
   - Build with compiler
   - Ship JavaScript only

3. **Bundled Frontend Apps** (React, Vue, Angular)
   - Build with webpack/vite
   - Ship static files

4. **Production Services** (APIs, microservices)
   - Security critical
   - Deployed frequently

5. **Large Dependency Trees**
   - 200+ npm packages
   - Many dev dependencies

---

### **❌ Overkill For:**

1. **Simple Scripts**
   ```dockerfile
   # Overkill for a 50-line Python script
   FROM python:3.11-slim
   COPY script.py .
   CMD ["python", "script.py"]
   # This is fine!
   ```

2. **Development Environments**
   - Need debugging tools
   - Need shell access
   - Need hot reload

3. **One-off Jobs**
   - Migration scripts
   - Batch jobs
   - CI/CD tasks that run once

4. **Apps Needing Build Tools at Runtime**
   - LaTeX document generators
   - JIT compilation systems

---

## 🎓 Quick Decision Tree

```
Should I use multi-stage builds?
│
├─ Is there a COMPILE step? (ts → js, go → binary)
│  ├─ YES → Multi-stage! ✅
│  └─ NO → Continue...
│
├─ Are my dependencies > 200MB?
│  ├─ YES → Multi-stage! ✅
│  └─ NO → Continue...
│
├─ Is this for PRODUCTION?
│  ├─ YES → Multi-stage! ✅
│  └─ NO (dev/testing) → Single-stage is fine
│
└─ Do I run this FREQUENTLY? (> 10 times/day)
   ├─ YES → Multi-stage worth it! ✅
   └─ NO → Single-stage is fine
```

---

## 📊 Performance Comparison

### **Real-World Metrics:**

| Metric | Single-Stage | Multi-Stage | Improvement |
|--------|--------------|-------------|-------------|
| **Image Size** | 1.2GB | 150MB | 87% smaller |
| **Build Time (first)** | 6 min | 4 min | 33% faster |
| **Build Time (cached)** | 6 min | 30 sec | 92% faster |
| **Registry Pull** | 5 min | 15 sec | 95% faster |
| **Vulnerabilities** | 237 | 3 | 99% reduction |

---

## 📝 Summary

Multi-stage builds are Docker's way of saying: **"Build messy, ship clean."**

**The Pattern:**
1. **Stage 1 (Kitchen):** Use full toolchain, make a mess, compile/build
2. **Stage 2 (Dining Room):** Copy ONLY what's needed, ship minimal image

**The Benefits:**
- 🎯 **87% smaller images** (1.2GB → 150MB typical)
- 🔒 **99% fewer vulnerabilities** (no dev tools in production)
- ⚡ **10x faster deployments** (smaller pulls)
- 💰 **$165/month savings** (registry costs)
- 🛡️ **Better security** (minimal attack surface)

**The Trade-offs:**
- ⚠️ Dockerfile complexity increases (30 lines vs 8)
- ⚠️ Initial build slightly slower (layer overhead)
- ⚠️ Requires BuildKit (Docker 18.09+)
- ⚠️ Learning curve for team

**The Bottom Line:**
If you're compiling code (TypeScript, Go, Java) or shipping to production, **multi-stage builds are a must**. The complexity cost is tiny compared to the operational benefits.

**Remember:** 
> "A restaurant with 50 locations doesn't make customers eat in the kitchen. Your microservices shouldn't ship with build tools in production."

---

**Resources:**
- [Docker Multi-stage Build Docs](https://docs.docker.com/build/building/multi-stage/)
- [BuildKit Reference](https://docs.docker.com/build/buildkit/)
- [Distroless Images](https://github.com/GoogleContainerTools/distroless)
- [Best Practices for Writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

