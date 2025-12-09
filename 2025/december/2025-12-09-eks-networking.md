# EKS Networking - The Mail & Delivery System

> Understanding how traffic reaches your pods in EKS - Services, Ingress, and Load Balancers explained through the office campus mail system.

---

## TL;DR

| EKS Networking Concept | Office Campus Analogy |
|------------------------|----------------------|
| ClusterIP Service | Internal mail system (campus only) |
| NodePort Service | Building mailboxes (numbered slots on each building) |
| LoadBalancer Service (NLB) | External delivery dock (courier drops packages) |
| Ingress (ALB) | Reception desk (checks labels, routes to departments) |
| Instance target mode | Courier → Building → internal sorting → desk |
| IP target mode | Courier → directly to your desk |
| externalTrafficPolicy: Cluster | Package forwarded between buildings if needed |
| externalTrafficPolicy: Local | Only accept at buildings with that department |
| AWS Load Balancer Controller | The logistics company managing all deliveries |

---

## The Big Picture

In our office campus (EKS cluster), we need ways for:
1. **Desks (pods) to communicate with each other** - Internal mail
2. **Outside world to reach specific desks** - External deliveries

```
EXTERNAL WORLD
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│                    YOUR CAMPUS (VPC/EKS)                        │
│                                                                 │
│   ┌─────────────────┐    ┌─────────────────┐                   │
│   │ Reception Desk  │    │ Delivery Dock   │                   │
│   │ (ALB/Ingress)   │    │ (NLB/LB Svc)    │                   │
│   └────────┬────────┘    └────────┬────────┘                   │
│            │                      │                             │
│            ▼                      ▼                             │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │              Internal Mail System (Services)            │  │
│   └─────────────────────────────────────────────────────────┘  │
│            │                      │                             │
│            ▼                      ▼                             │
│   ┌─────────────┐         ┌─────────────┐                      │
│   │ Desks (Pods)│         │ Desks (Pods)│                      │
│   └─────────────┘         └─────────────┘                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Service Types: The Mail System

### ClusterIP - Internal Campus Mail

Like an **internal mail system** - only works within the campus. External couriers can't use it.

```
┌─────────────────────────────────────────────────────────────────┐
│                         CAMPUS                                  │
│                                                                 │
│   Building A                    Building B                      │
│   ┌─────────────────┐          ┌─────────────────┐             │
│   │ Frontend Desk   │          │ Backend Desk 1  │             │
│   │                 │──MAIL───▶│ Backend Desk 2  │             │
│   │                 │          │ Backend Desk 3  │             │
│   └─────────────────┘          └─────────────────┘             │
│                                        ▲                        │
│                            ┌───────────┴───────────┐           │
│                            │  backend-service      │           │
│                            │  (Internal Mailroom)  │           │
│                            │  Address: backend:80  │           │
│                            └───────────────────────┘           │
│                                                                 │
│   ❌ External courier cannot reach this mailroom!              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Use when:** Internal pod-to-pod communication (databases, backend APIs, caches)

---

### NodePort - Building Mailboxes

Like having a **numbered mailbox slot on every building**. External couriers can drop packages, but they need to know which building to go to.

```
┌─────────────────────────────────────────────────────────────────┐
│                         CAMPUS                                  │
│                                                                 │
│   Building A              Building B              Building C    │
│   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐│
│   │ Mailbox #30080  │    │ Mailbox #30080  │    │ Mailbox #30080│
│   │       ↓         │    │       ↓         │    │       ↓     ││
│   │ Internal Sort   │    │ Internal Sort   │    │ Internal Sort││
│   │       ↓         │    │       ↓         │    │       ↓     ││
│   │   Desk (Pod)    │    │   (no desk)     │    │   Desk (Pod)││
│   └─────────────────┘    └─────────────────┘    └─────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
        ▲                         ▲                      ▲
        │                         │                      │
   Courier can                Courier can           Courier can
   deliver here              deliver here          deliver here

Same mailbox number (30080) on EVERY building!
```

**Use when:** Rarely used directly - usually as a building block for LoadBalancer

---

### LoadBalancer (NLB) - External Delivery Dock

Like hiring a **courier company** that sets up a delivery dock. They handle receiving all external packages and routing them inside.

```
                    EXTERNAL WORLD
                          │
                          ▼
                 ┌─────────────────┐
                 │  DELIVERY DOCK  │
                 │  (NLB)          │
                 │  dock.myapp.com │
                 └────────┬────────┘
                          │
┌─────────────────────────┼───────────────────────────────────────┐
│                  CAMPUS │                                       │
│                         ▼                                       │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │              Building Mailboxes (NodePort)              │  │
│   └─────────────────────────────────────────────────────────┘  │
│         │                    │                    │             │
│         ▼                    ▼                    ▼             │
│   Building A            Building B           Building C        │
│   ┌───────────┐        ┌───────────┐        ┌───────────┐     │
│   │ Desk(Pod) │        │   (empty) │        │ Desk(Pod) │     │
│   └───────────┘        └───────────┘        └───────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key point:** LoadBalancer = ClusterIP + NodePort + External Load Balancer

**Use when:** L4 (TCP/UDP) traffic, non-HTTP services, high-performance needs

---

## The Service Progression

They **build on each other**:

```
ClusterIP
    │
    ├── Virtual IP inside campus
    │
    ▼
NodePort = ClusterIP + Port on every building
    │
    ├── Same ClusterIP
    ├── + Mailbox #30000-32767 on each building
    │
    ▼
LoadBalancer = ClusterIP + NodePort + Delivery Dock
    │
    ├── Same ClusterIP
    ├── + Same NodePort
    └── + External dock (NLB) managed by courier company
```

---

## Ingress (ALB) - The Reception Desk

Unlike the delivery dock (NLB) that just forwards packages, the **reception desk (ALB)** actually **reads the labels** and routes intelligently.

```
                    EXTERNAL WORLD
                          │
                          ▼
                 ┌─────────────────────────────────────┐
                 │         RECEPTION DESK (ALB)        │
                 │              myapp.com              │
                 │                                     │
                 │  "Let me check that label..."       │
                 │                                     │
                 │  📦 /api/* → Backend Department    │
                 │  📦 /*     → Frontend Department   │
                 │  📦 /admin → Admin Department      │
                 └──────────────┬──────────────────────┘
                                │
┌───────────────────────────────┼─────────────────────────────────┐
│                        CAMPUS │                                 │
│          ┌────────────────────┼────────────────────┐           │
│          │                    │                    │           │
│          ▼                    ▼                    ▼           │
│   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐   │
│   │ frontend-svc│      │ backend-svc │      │  admin-svc  │   │
│   │ (ClusterIP) │      │ (ClusterIP) │      │ (ClusterIP) │   │
│   └──────┬──────┘      └──────┬──────┘      └──────┬──────┘   │
│          ▼                    ▼                    ▼           │
│   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐   │
│   │Frontend Pods│      │Backend Pods │      │ Admin Pods  │   │
│   └─────────────┘      └─────────────┘      └─────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### What Reception Desk (ALB/Ingress) Can Do

| Capability | Example |
|------------|---------|
| Path-based routing | `/api` → backend, `/` → frontend |
| Host-based routing | `api.myapp.com` → backend, `myapp.com` → frontend |
| TLS termination | Handle HTTPS, forward HTTP internally |
| Redirects | HTTP → HTTPS |
| Health checks | L7 health checks on specific paths |

**Use when:** HTTP/HTTPS traffic, need routing rules, multiple services behind one domain

---

## Cost Comparison

```
WITHOUT Ingress:
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│ NLB ($18/mo)│   │ NLB ($18/mo)│   │ NLB ($18/mo)│
└──────┬──────┘   └──────┬──────┘   └──────┬──────┘
       ▼                 ▼                 ▼
   Frontend          Backend            Admin
   
   Total: ~$54/month + data processing

WITH Ingress:
┌─────────────────────────────────────────────────┐
│              ALB ($18/mo)                       │
│  /        → Frontend                            │
│  /api     → Backend                             │
│  /admin   → Admin                               │
└─────────────────────────────────────────────────┘

   Total: ~$18/month + data processing
   
   💰 SAVINGS: ~$36/month!
```

---

## Instance Mode vs IP Mode

### Instance Mode - Package Goes to Building First

```
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│   Courier → Building A → Internal Sorting → Forward to Desk  │
│             (Node)       (kube-proxy)        (Pod)            │
│                                                               │
│   Package might even go to Building B if desk is there!      │
│                                                               │
└───────────────────────────────────────────────────────────────┘

Traffic: LB → Node:NodePort → kube-proxy → Pod (maybe different node)
```

### IP Mode - Package Goes Directly to Desk

```
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│   Courier → Directly to Desk (Pod)                            │
│                                                               │
│   Courier knows exact desk location (VPC CNI = real IPs!)    │
│                                                               │
└───────────────────────────────────────────────────────────────┘

Traffic: LB → Pod directly!
```

### Comparison

| Aspect | Instance Mode | IP Mode |
|--------|---------------|---------|
| Traffic path | LB → Node → Pod | LB → Pod |
| Extra hop | Yes | No |
| Latency | Higher | Lower ✅ |
| Works with VPC CNI | Yes | Yes (required) |
| Works with Overlay CNI | Yes | No |

### Defaults in AWS

| Load Balancer | Default Target Type |
|---------------|---------------------|
| NLB (Service) | Instance |
| ALB (Ingress) | **IP** ✅ |

---

## externalTrafficPolicy

**Only applies to Instance mode!** (IP mode goes directly to pods)

### Cluster Mode (Default)

Package can be **forwarded between buildings** if the desk isn't in the building that received it.

```
┌─────────────────────────────────────────────────────────────────┐
│                         CAMPUS                                  │
│                                                                 │
│   Building A              Building B              Building C    │
│   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐│
│   │ Mailbox #30080  │    │ Mailbox #30080  │    │ Mailbox #30080│
│   │       ↓         │    │       ↓         │    │       ↓     ││
│   │  📦 received    │    │                 │    │             ││
│   │       ↓         │    │                 │    │             ││
│   │ "Desk not here" │    │                 │    │             ││
│   │       ↓         │────┼─── forward ────▶│    │  Desk (Pod) ││
│   │                 │    │                 │    │  📦 arrives ││
│   └─────────────────┘    └─────────────────┘    └─────────────┘│
│                                                                 │
│   ✅ Always works                                              │
│   ❌ Extra hop between buildings                               │
│   ❌ Original sender address lost (building rewrites it)       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Local Mode

Package is **only accepted at buildings that have the desk**.

```
┌─────────────────────────────────────────────────────────────────┐
│                         CAMPUS                                  │
│                                                                 │
│   Building A              Building B              Building C    │
│   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐│
│   │ Mailbox #30080  │    │ Mailbox CLOSED  │    │ Mailbox #30080│
│   │       ↓         │    │ (no desk here)  │    │       ↓     ││
│   │                 │    │                 │    │             ││
│   │   Desk (Pod)    │    │       ❌        │    │   Desk (Pod)││
│   │   📦 arrives    │    │                 │    │   📦 arrives││
│   └─────────────────┘    └─────────────────┘    └─────────────┘│
│                                                                 │
│   ✅ No extra hop - direct delivery                            │
│   ✅ Original sender address preserved                         │
│   ⚠️  Only works at buildings with desks                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Courier company (NLB) health checks remove Building B from targets.
```

### When to Use

| Use Case | Policy |
|----------|--------|
| Need client's real IP | Local |
| Pods on every node | Cluster (safe) |
| Latency-sensitive | Local |
| Don't care about source IP | Cluster |
| Using IP mode | Doesn't matter! |

---

## AWS Load Balancer Controller

The **logistics company** that manages both the reception desk (ALB) and delivery dock (NLB).

```
┌─────────────────────────────────────────────────────────────────┐
│               AWS LOAD BALANCER CONTROLLER                      │
│                  (The Logistics Company)                        │
│                                                                 │
│   "I watch for Ingress and LoadBalancer Service resources,     │
│    and create/manage ALBs and NLBs in AWS for you"             │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Ingress Resource ──────────────▶ Creates ALB                 │
│   (ingressClassName: alb)                                       │
│                                                                 │
│   Service type: LoadBalancer ────▶ Creates NLB                 │
│   (annotations)                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Installation Required

```bash
# The controller must be installed in your cluster
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster
```

---

## Putting It All Together

### Example Architecture

```
                         INTERNET
                             │
                             ▼
┌────────────────────────────────────────────────────────────────┐
│                    ALB (Ingress)                               │
│                   app.example.com                              │
│                                                                │
│   Path Routing:                                                │
│   • /          → frontend-service                              │
│   • /api/*     → backend-service                               │
└───────────────────────────┬────────────────────────────────────┘
                            │
┌───────────────────────────┼────────────────────────────────────┐
│                     EKS CLUSTER                                │
│                           │                                    │
│         ┌─────────────────┴─────────────────┐                 │
│         │                                   │                 │
│         ▼                                   ▼                 │
│  ┌─────────────────┐              ┌─────────────────┐        │
│  │ frontend-service│              │ backend-service │        │
│  │   (ClusterIP)   │              │   (ClusterIP)   │        │
│  └────────┬────────┘              └────────┬────────┘        │
│           │                                │                  │
│           ▼                                ▼                  │
│  ┌─────────────────┐              ┌─────────────────┐        │
│  │  Frontend Pods  │              │  Backend Pods   │        │
│  └─────────────────┘              └────────┬────────┘        │
│                                            │                  │
│                                            ▼                  │
│                                   ┌─────────────────┐        │
│                                   │ database-service│        │
│                                   │   (ClusterIP)   │        │
│                                   └────────┬────────┘        │
│                                            │                  │
│                                            ▼                  │
│                                   ┌─────────────────┐        │
│                                   │  Database Pod   │        │
│                                   │ (Internal Only) │        │
│                                   └─────────────────┘        │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Need | Use | Layer |
|------|-----|-------|
| Pod-to-pod internal communication | ClusterIP | L4 |
| Expose on node ports (rarely direct) | NodePort | L4 |
| External TCP/UDP access | LoadBalancer (NLB) | L4 |
| External HTTP/HTTPS with routing | Ingress (ALB) | L7 |
| Multiple services, one domain | Ingress (ALB) | L7 |
| TLS termination | Ingress (ALB) | L7 |

---

## Key Takeaways

1. **Services build on each other** - ClusterIP → NodePort → LoadBalancer
2. **Ingress is L7, LoadBalancer is L4** - Different use cases
3. **IP mode is better in EKS** - Direct to pod, lower latency (thanks to VPC CNI!)
4. **ALB defaults to IP mode, NLB defaults to Instance mode**
5. **One ALB can serve multiple services** - Cost savings with path/host routing
6. **externalTrafficPolicy matters for Instance mode** - Cluster vs Local
7. **AWS Load Balancer Controller** - Required to create ALB/NLB from K8s resources

---

## Related Reading

- [AWS VPC - The Compound Analogy](../november/2025-11-30-aws-vpc.md)
- [AWS EKS - The Managed Office Building Analogy](./2025-12-02-aws-eks.md)
- [AWS VPC CNI - The Phone Extension System](./2025-12-03-aws-vpc-cni.md)

---

*Written on December 9, 2025*
