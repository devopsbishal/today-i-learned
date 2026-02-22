# Docker Networking - The Apartment Complex Analogy

> Understanding how Docker containers communicate: from isolated apartments to underground tunnels connecting buildings across cities.

---

## TL;DR

| Docker Concept | Apartment Complex Analogy |
|----------------|---------------------------|
| **Docker Host** | Apartment building (the property) |
| **Bridge Network** | Gated residential complex with internal streets |
| **Container** | Individual apartment/unit |
| **IP Address** | Apartment number (Unit 172.17.0.2) |
| **Port Mapping (-p)** | Main gate reception desk forwarding visitors |
| **docker0 bridge** | Internal hallway/street system |
| **veth pairs** | Fiber optic cable from apartment to hallway |
| **Default bridge** | Old complex (no intercom, must know unit numbers) |
| **Custom bridge** | Modern complex (has intercom - call by name!) |
| **Embedded DNS** | Building directory/intercom (127.0.0.11) |
| **Host network** | Living in landlord's penthouse (share everything!) |
| **None network** | Solitary unit with no doors/windows (air-gapped) |
| **Overlay network** | Underground tunnel connecting multiple buildings |
| **Macvlan** | Each apartment has own street entrance (bypass hallway) |
| **IPvlan** | Shared street entrance, different apartment visible from street |
| **Multi-network container** | Person with apartments in two complexes |
| **iptables NAT** | Security guard at main gate (visitor tracking) |
| **VXLAN** | Secret underground tunnel with disguises |

---

## The Big Picture

Docker networking isn't magic - it's a clever combination of **Linux network namespaces**, **virtual ethernet pairs**, and **network bridges** wrapped in a user-friendly interface.

```
┌─────────────────────────────────────────────────────────┐
│         🏢 APARTMENT COMPLEX (Docker Host)              │
│                                                         │
│  Main Gate (Port 8080)                                  │
│       ↓                                                 │
│  Reception Desk (iptables DNAT)                         │
│   "Looking for Building 8080? Go to Apt 80!"          │
│       ↓                                                 │
│  ┌─────────────────────────────────────────────┐       │
│  │  Internal Street (docker0 bridge)           │       │
│  │                                             │       │
│  │  Apt 172.17.0.2 ←─── fiber cable (veth) ───┤       │
│  │  Apt 172.17.0.3 ←─── fiber cable (veth) ───┤       │
│  │  Apt 172.17.0.4 ←─── fiber cable (veth) ───┤       │
│  │                                             │       │
│  │  Residents can:                             │       │
│  │  • Talk to each other via hallway           │       │
│  │  • Call outside (via reception NAT)         │       │
│  │  • Use intercom if custom complex           │       │
│  └─────────────────────────────────────────────┘       │
│                                                         │
│  Exit Guard (iptables MASQUERADE)                       │
│  "Resident from Apt 172.17.0.2 leaving? Tell outside   │
│   world they're from Building 10.0.1.50"               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Key Insight**: Each container thinks it has a complete apartment with its own address, but they're all connected via the internal street system (bridge) and share the same building address when talking to the outside world!

---

## Network Namespace - Your Private Apartment

When Docker creates a container, it uses **Linux network namespaces** to give it a completely isolated network view.

```
┌─────────────────────────────────────────────────────────┐
│     WHAT CONTAINER SEES (Inside the Apartment)          │
│                                                         │
│  "I'm alone in my own space!"                          │
│                                                         │
│  My Network Interfaces:                                 │
│  ┌────────────────────────────────┐                    │
│  │ lo: 127.0.0.1 (my bathroom)    │                    │
│  │ eth0: 172.17.0.2 (my phone)    │                    │
│  └────────────────────────────────┘                    │
│                                                         │
│  My Hostname: web-container                             │
│  My PID: 1 (I'm the first process!)                   │
│                                                         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│     WHAT HOST SEES (Building Manager's View)            │
│                                                         │
│  "I can see EVERYTHING!"                               │
│                                                         │
│  All Network Interfaces:                                │
│  ┌────────────────────────────────┐                    │
│  │ eth0: 10.0.1.50 (main entrance)│                    │
│  │ docker0: 172.17.0.1 (hallway)  │                    │
│  │ veth123: (cable to Apt 2)      │                    │
│  │ veth456: (cable to Apt 3)      │                    │
│  │ veth789: (cable to Apt 4)      │                    │
│  └────────────────────────────────┘                    │
│                                                         │
│  Containers are just processes:                         │
│  PID 5234: nginx (thinks it's PID 1!)                  │
│  PID 5899: postgres (also thinks it's PID 1!)          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**How it's created:**
```bash
# Docker (via runc) makes this Linux syscall:
clone(CLONE_NEWNET | CLONE_NEWPID | ...);
# Creates isolated network namespace - like building a new apartment!
```

---

## veth Pairs - Fiber Optic Cables to Your Apartment

**veth (virtual ethernet) pairs** are like fiber optic cables connecting your apartment to the hallway.

```
┌─────────────────────────────────────────────────────────┐
│         THE VIRTUAL CABLE CONNECTION                    │
│                                                         │
│  Inside Apartment (Container Network Namespace)         │
│  ┌──────────────────────────────────┐                  │
│  │  eth0: 172.17.0.2               │                  │
│  │  (Container sees this end)       │                  │
│  └─────────────┬────────────────────┘                  │
│                │                                         │
│                │ Virtual Cable (veth pair)              │
│                │ ═══════════════════════                │
│                │                                         │
│  ┌─────────────┴────────────────────┐                  │
│  │  veth123abc                       │                  │
│  │  (Host sees this end)             │                  │
│  └─────────────┬────────────────────┘                  │
│                │                                         │
│                ↓                                         │
│  ┌─────────────────────────────────────────────┐       │
│  │      docker0 Bridge (Hallway Switch)        │       │
│  │  Connects all apartments in the complex     │       │
│  └─────────────────────────────────────────────┘       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**What happens when container sends a packet:**
1. Container writes to `eth0` (their end of cable)
2. Packet travels through veth pair
3. Appears on `veth123abc` (host end)
4. docker0 bridge receives it
5. Bridge switches it to destination apartment's veth
6. Arrives at destination container's `eth0`

**See it in action:**
```bash
# On host - see all the cables
ip link | grep veth
# veth123abc@if7: <BROADCAST,MULTICAST,UP,LOWER_UP>
# veth456def@if9: <BROADCAST,MULTICAST,UP,LOWER_UP>

# Inside container - see your end
docker exec my-container ip link
# eth0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP>
```

---

## Bridge Network - The Gated Complex

The **bridge network** is like a gated residential complex with internal streets connecting all apartments.

### Default Bridge - The Old Complex (No Intercom)

```
┌─────────────────────────────────────────────────────────┐
│         OLD APARTMENT COMPLEX (Default Bridge)          │
│         Built in 2013, No Modern Amenities              │
│                                                         │
│  Complex Address: 172.17.0.0/16                         │
│  Internal Street: docker0                               │
│                                                         │
│  Apartments:                                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │ Apt 0.2     │  │ Apt 0.3     │  │ Apt 0.4     │    │
│  │ (nginx)     │  │ (postgres)  │  │ (redis)     │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
│                                                         │
│  ❌ NO INTERCOM SYSTEM                                  │
│  - To visit someone, you MUST know their unit number   │
│  - Can't just say "Take me to the nginx apartment"     │
│  - Must remember: "172.17.0.2"                         │
│                                                         │
│  Old residents don't mind (legacy apps)                 │
│  New residents frustrated (microservices)               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Problem:**
```bash
# Can't talk by name
docker run --name web nginx
docker run --name app python:3.9
docker exec app ping web
# ❌ ping: bad address 'web'

# Must use IP
docker exec app ping 172.17.0.2
# ✅ Works, but you have to remember the IP!
```

### Custom Bridge - The Modern Complex (With Intercom!)

```
┌─────────────────────────────────────────────────────────┐
│       MODERN APARTMENT COMPLEX (Custom Bridge)          │
│       Built in 2024, Full Amenities                     │
│                                                         │
│  Complex Name: my-network                               │
│  Complex Address: 172.18.0.0/16                         │
│  Internal Street: br-abc123                             │
│                                                         │
│  🎤 EMBEDDED DNS/INTERCOM: 127.0.0.11                   │
│     "Just say the resident's name!"                     │
│                                                         │
│  Apartments:                                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │ Apt 0.2     │  │ Apt 0.3     │  │ Apt 0.4     │    │
│  │ Name: web   │  │ Name: app   │  │ Name: db    │    │
│  │ (nginx)     │  │ (python)    │  │ (postgres)  │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
│                                                         │
│  ✅ INTERCOM WORKS                                      │
│  - Press button, say "Connect me to web"               │
│  - Intercom (DNS) looks up: web = 172.18.0.2           │
│  - Connects you automatically!                          │
│                                                         │
│  Modern, user-friendly, production-ready!               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Solution:**
```bash
# Create modern complex
docker network create my-network

# Move in residents
docker run -d --name web --network my-network nginx
docker run -d --name app --network my-network python:3.9

# Now intercom works!
docker exec app ping web
# ✅ PING web (172.18.0.2): 56 data bytes

# DNS resolution happening
docker exec app nslookup web
# Server:    127.0.0.11  ← Embedded DNS server!
# Address:   127.0.0.11:53
# Name:      web
# Address:   172.18.0.2
```

**How DNS works:**
```
App container wants to reach "web"
    ↓
Looks at /etc/resolv.conf
    nameserver 127.0.0.11
    ↓
Queries Docker's embedded DNS server
    ↓
DNS server checks its mapping:
    "web" → 172.18.0.2
    ↓
Returns IP address to container
    ↓
Container connects to 172.18.0.2
```

---

## Port Publishing - The Reception Desk

When you publish a port with `-p`, you're setting up a **reception desk at the main gate** to forward visitors to specific apartments.

```
┌─────────────────────────────────────────────────────────┐
│         PORT PUBLISHING (-p 8080:80)                    │
│                                                         │
│  Internet Visitor arrives:                              │
│  "I'm looking for Building:8080"                       │
│                  ↓                                       │
│  ┌────────────────────────────────────────────┐        │
│  │  Main Gate (Host IP: 10.0.1.50:8080)      │        │
│  │                                            │        │
│  │  Reception Desk (iptables DNAT):          │        │
│  │  "Ah yes, Building:8080 residents         │        │
│  │   are in Apartment 172.17.0.2:80"         │        │
│  │                                            │        │
│  │  Forwards visitor inside →                │        │
│  └────────────────────────────────────────────┘        │
│                  ↓                                       │
│  ┌────────────────────────────────────────────┐        │
│  │  Internal Hallway (docker0 bridge)        │        │
│  └────────────────────────────────────────────┘        │
│                  ↓                                       │
│  ┌────────────────────────────────────────────┐        │
│  │  Apartment 172.17.0.2:80                   │        │
│  │  Resident: nginx (running web server)      │        │
│  └────────────────────────────────────────────┘        │
│                                                         │
│  Visitor leaves with response:                          │
│  ┌────────────────────────────────────────────┐        │
│  │  Exit Guard (iptables MASQUERADE):         │        │
│  │  "Resident 172.17.0.2 leaving?            │        │
│  │   Tell the visitor it's from 10.0.1.50"   │        │
│  └────────────────────────────────────────────┘        │
│                  ↓                                       │
│  Visitor thinks: "I talked to 10.0.1.50!"              │
│  (Doesn't know about internal apartment number)         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**The iptables rules Docker creates:**

```bash
# DNAT (Destination NAT) - Incoming translation
# "Change where this is going"
-A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80

# Packet transformation:
Before: dst=10.0.1.50:8080
After:  dst=172.17.0.2:80

# MASQUERADE (Source NAT) - Outgoing translation  
# "Change where this came from"
-A POSTROUTING -s 172.17.0.0/16 -j MASQUERADE

# Packet transformation:
Before: src=172.17.0.2
After:  src=10.0.1.50
```

**Security consideration:**

```bash
# Publish to ALL interfaces (anyone can reach it)
docker run -p 8080:80 nginx
# Same as: -p 0.0.0.0:8080:80
# ⚠️ Accessible from internet if host has public IP!

# Publish to localhost ONLY (secure)
docker run -p 127.0.0.1:8080:80 nginx
# 🔒 Only accessible from the host machine
# Perfect for databases, internal services
```

---

## Network Modes - Different Living Arrangements

Docker supports different "living arrangements" for your containers:

### 1. Bridge Mode - Standard Apartment (Default)

```
┌─────────────────────────────────────────────────────────┐
│  Standard Apartment (Bridge Network)                    │
│                                                         │
│  ✅ Your own space (network namespace)                  │
│  ✅ Your own phone line (IP address)                    │
│  ✅ Connected to hallway (docker0)                      │
│  ✅ Can call neighbors (other containers)               │
│  ✅ Can call outside (via NAT)                          │
│  ✅ Isolated from host and other apartments             │
│                                                         │
│  Perfect for: Most applications                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

```bash
docker run -d --name web nginx
# Gets isolated network, communicates via bridge
```

### 2. Host Mode - Living in the Landlord's Penthouse

```
┌─────────────────────────────────────────────────────────┐
│  Landlord's Penthouse (Host Network)                    │
│                                                         │
│  ⚡ MAXIMUM PERFORMANCE - No walls, no translation!     │
│                                                         │
│  You share EVERYTHING with the landlord:                │
│  ❌ No separate network namespace                       │
│  ❌ No private IP address                               │
│  ❌ No isolation at all                                 │
│  ✅ Direct access to ALL host network interfaces        │
│  ✅ Bind directly to host's ports                       │
│  ✅ No NAT overhead                                     │
│                                                         │
│  ⚠️  SECURITY RISK:                                     │
│  - Can sniff ALL host network traffic                  │
│  - Can bind to ANY port (even SSH!)                    │
│  - Port conflicts possible                              │
│                                                         │
│  Perfect for: Performance-critical apps,                │
│              network monitoring tools                   │
│  Avoid for: Untrusted workloads, web apps              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

```bash
docker run --network host nginx
# Nginx binds DIRECTLY to host's port 80!
# No veth pairs, no bridge, no NAT - maximum speed!
```

### 3. None Mode - Solitary Confinement

```
┌─────────────────────────────────────────────────────────┐
│  Solitary Unit (None Network)                           │
│                                                         │
│  🔒 MAXIMUM ISOLATION - Air-gapped!                     │
│                                                         │
│  ❌ NO network interfaces (except loopback)             │
│  ❌ NO IP address                                       │
│  ❌ NO internet access                                  │
│  ❌ NO communication with other containers              │
│  ❌ NO communication with host                          │
│  ✅ ONLY loopback (127.0.0.1) works                     │
│                                                         │
│  Perfect for:                                           │
│  - Untrusted workloads (malware analysis)              │
│  - Batch processing (read volume, write volume)        │
│  - Maximum security requirements                        │
│  - Air-gapped compliance (PCI-DSS, HIPAA)             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

```bash
docker run --network none alpine
# Complete network isolation - can only access mounted volumes
```

### 4. Macvlan - Your Own Street Entrance

```
┌─────────────────────────────────────────────────────────┐
│  Macvlan - Direct Street Access                         │
│                                                         │
│  🚪 Each apartment has its OWN street entrance!         │
│                                                         │
│  ✅ Bypasses internal hallway (docker0) completely      │
│  ✅ Container gets its own MAC address                  │
│  ✅ Appears as physical device on network               │
│  ✅ Can be on same subnet as host                       │
│  ⚡ Better performance (no bridge overhead)             │
│                                                         │
│  Network sees:                                          │
│  Host:      192.168.1.10 (MAC: aa:bb:cc:11:22:33)     │
│  Container: 192.168.1.20 (MAC: aa:bb:cc:44:55:66)     │
│                                                         │
│  It's like having your own house, not an apartment!    │
│                                                         │
│  Perfect for:                                           │
│  - Network appliances (monitoring, DHCP servers)       │
│  - Legacy apps expecting physical network              │
│  - When you need separate MAC addresses                │
│                                                         │
│  ⚠️  Caveat: Host can't reach containers on macvlan    │
│              (it's a Linux kernel limitation)          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

```bash
# Create macvlan network
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  macvlan-net

# Run container on physical network
docker run -d --name monitor \
  --network macvlan-net \
  --ip 192.168.1.100 \
  network-monitor-app

# Container appears as physical device on 192.168.1.0/24!
```

### 5. IPvlan - Shared Street Entrance, Different Apartment Numbers

```
┌─────────────────────────────────────────────────────────┐
│  IPvlan - Shared MAC, Different IPs                     │
│                                                         │
│  🚪 Multiple apartments share ONE street entrance       │
│     (same MAC address, different IP addresses)         │
│                                                         │
│  Similar to Macvlan but:                                │
│  ✅ All containers share host's MAC address             │
│  ✅ Each gets unique IP address                         │
│  ✅ Less MAC address pollution on network               │
│  ✅ Better for large deployments                        │
│                                                         │
│  Two modes:                                             │
│  • L2 mode: Like macvlan (same subnet)                 │
│  • L3 mode: Routing between subnets                    │
│                                                         │
│  Network sees:                                          │
│  Host:       192.168.1.10 (MAC: aa:bb:cc:11:22:33)    │
│  Container1: 192.168.1.20 (MAC: aa:bb:cc:11:22:33) ← Same! │
│  Container2: 192.168.1.21 (MAC: aa:bb:cc:11:22:33) ← Same! │
│                                                         │
│  Perfect for:                                           │
│  - Large container deployments                         │
│  - Switches with MAC address limits                    │
│  - When you don't need unique MAC addresses            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

```bash
# Create IPvlan network (L2 mode)
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  -o ipvlan_mode=l2 \
  ipvlan-net

# Run containers
docker run -d --network ipvlan-net --ip 192.168.1.100 app1
docker run -d --network ipvlan-net --ip 192.168.1.101 app2
# Both share host's MAC but have different IPs
```

**Macvlan vs IPvlan:**

| Feature | Macvlan | IPvlan |
|---------|---------|--------|
| MAC addresses | Unique per container | Shared (host's MAC) |
| Performance | Excellent | Excellent |
| Switch MAC table | Grows with containers | Stays small |
| Use case | Need unique MACs | Large deployments |
| Complexity | Simple | Slightly more complex |

---

## Multi-Network Containers - Living in Two Places

You can connect a container to **multiple networks** - like having an apartment in two different complexes!

```
┌─────────────────────────────────────────────────────────┐
│     MULTI-NETWORK CONTAINER (Bridge Pattern)            │
│                                                         │
│  Scenario: App needs to talk to both Web and Database   │
│                                                         │
│  ┌──────────────────────────────────────────────┐      │
│  │  Frontend Complex (external-net)             │      │
│  │  ┌──────────┐         ┌──────────┐          │      │
│  │  │   Web    │ ←────→  │   App    │          │      │
│  │  │ Container│         │ Container│          │      │
│  │  └──────────┘         └────┬─────┘          │      │
│  │                            │                 │      │
│  │  IP: 172.18.0.2      eth0: 172.18.0.3      │      │
│  └────────────────────────────│─────────────────┘      │
│                                │                        │
│                           App has TWO                   │
│                           network interfaces!           │
│                                │                        │
│  ┌────────────────────────────┴─────────────────┐      │
│  │  Backend Complex (data-net)            │      │      │
│  │                        ┌────┴─────┐    │      │      │
│  │                        │   App    │    │      │      │
│  │                        │ Container│    │      │      │
│  │  ┌──────────┐         └────┬─────┘    │      │      │
│  │  │ Database │ ←────────────┘          │      │      │
│  │  │ Container│                         │      │      │
│  │  └──────────┘          eth1: 172.19.0.3      │      │
│  │                                              │      │
│  │  IP: 172.19.0.2                              │      │
│  └──────────────────────────────────────────────┘      │
│                                                         │
│  Result:                                                │
│  ✅ Web can reach App (both on external-net)           │
│  ✅ App can reach Database (both on data-net)          │
│  ❌ Web CANNOT reach Database (not on same network)    │
│                                                         │
│  This is network segmentation for security!             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Implementation:**

```bash
# Create two networks
docker network create frontend
docker network create backend

# Start containers
docker run -d --name web --network frontend nginx
docker run -d --name app --network frontend python-app
docker run -d --name db --network backend postgres

# Connect app to backend also (now has 2 interfaces!)
docker network connect backend app

# Verify
docker exec app ip addr
# eth0: 172.18.0.3 (frontend network)
# eth1: 172.19.0.2 (backend network)

# Test connectivity
docker exec web ping app       # ✅ Works (both on frontend)
docker exec app ping db        # ✅ Works (both on backend)
docker exec web ping db        # ❌ Fails (different networks!)
```

---

## Overlay Networks - Underground Tunnels Between Buildings

When you have containers on **different physical hosts** (different buildings), bridge networks don't work - they're only local to each building. You need **overlay networks** with **VXLAN tunnels**.

```
┌─────────────────────────────────────────────────────────┐
│         OVERLAY NETWORK - THE UNDERGROUND TUNNEL        │
│                                                         │
│  🏢 Building A (Host 1)       🏢 Building B (Host 2)   │
│  Physical: 192.168.1.10        Physical: 192.168.1.20  │
│                                                         │
│  ┌──────────────┐              ┌──────────────┐        │
│  │ Apt 10.0.1.2 │              │ Apt 10.0.1.3 │        │
│  │   (web-1)    │              │   (web-2)    │        │
│  └──────┬───────┘              └──────┬───────┘        │
│         │                             │                │
│         │ "I want to talk to          │                │
│         │  Apt 10.0.1.3"              │                │
│         ↓                             ↓                │
│  ┌──────────────────────┐    ┌──────────────────────┐ │
│  │  VXLAN Driver        │    │  VXLAN Driver        │ │
│  │  "Let me wrap this   │    │  "Unwrapping packet  │ │
│  │   in a disguise!"    │    │   from tunnel..."    │ │
│  └──────┬───────────────┘    └──────┬───────────────┘ │
│         │                             │                │
│         │ Encapsulates packet         │                │
│         │ in UDP (port 4789)          │                │
│         │                             │                │
│         ╠═════════════════════════════╣                │
│         ║   Physical Network          ║                │
│         ║   (The Underground Tunnel)  ║                │
│         ╠═════════════════════════════╣                │
│                                                         │
│  Container 10.0.1.2 thinks it's talking                │
│  directly to Container 10.0.1.3 on same network!       │
│  (They don't know about the tunnel encapsulation)      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**How VXLAN encapsulation works:**

```
Original Packet from Container:
┌────────────────────────────────────────┐
│ Src: 10.0.1.2                          │
│ Dst: 10.0.1.3                          │
│ Data: "Hello from web-1!"              │
└────────────────────────────────────────┘

After VXLAN Encapsulation (The Disguise):
┌────────────────────────────────────────┐
│ Outer IP Header:                       │
│   Src: 192.168.1.10 (Host A)          │
│   Dst: 192.168.1.20 (Host B)          │
│                                        │
│ UDP Header: Port 4789                  │
│                                        │
│ VXLAN Header: VNI 256                  │
│   (Virtual Network Identifier)         │
│                                        │
│ ┌────────────────────────────────┐    │
│ │ Inner/Original Packet:         │    │
│ │ Src: 10.0.1.2                  │    │
│ │ Dst: 10.0.1.3                  │    │
│ │ Data: "Hello from web-1!"      │    │
│ └────────────────────────────────┘    │
└────────────────────────────────────────┘
```

**Creating overlay networks:**

```bash
# Requires Docker Swarm mode (coordination needed!)
docker swarm init

# Create overlay network
docker network create \
  -d overlay \
  --attachable \
  my-overlay

# Deploy services across hosts
docker service create \
  --name web \
  --network my-overlay \
  --replicas 3 \
  nginx

# Containers on different hosts can now communicate!
# Docker handles all the VXLAN tunneling automatically
```

**Control plane coordination:**

```
┌─────────────────────────────────────────────────────────┐
│         SWARM MANAGERS (Coordination)                   │
│                                                         │
│  Raft Consensus Store:                                  │
│  ┌──────────────────────────────────────┐              │
│  │ Network: my-overlay                  │              │
│  │ VNI: 256                             │              │
│  │ Subnet: 10.0.1.0/24                  │              │
│  │                                      │              │
│  │ Container Mappings:                  │              │
│  │ • 10.0.1.2 → Host A (192.168.1.10)  │              │
│  │ • 10.0.1.3 → Host B (192.168.1.20)  │              │
│  │ • 10.0.1.4 → Host C (192.168.1.30)  │              │
│  └──────────────────────────────────────┘              │
│                                                         │
│  All hosts query managers for mappings                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Complete Packet Journey - Following a Visitor

Let's follow an HTTPS request from the internet all the way to the database:

```
┌─────────────────────────────────────────────────────────┐
│     THE COMPLETE VISITOR JOURNEY                        │
│                                                         │
│  1. 🌐 Internet                                         │
│     User: curl https://api.example.com/users/123        │
│        ↓                                                │
│                                                         │
│  2. 🏢 Main Gate (Host eth0: 10.0.1.50:443)            │
│     Packet arrives at host's network interface          │
│        ↓                                                │
│                                                         │
│  3. 🛡️ Reception Desk (iptables DNAT)                  │
│     Rule: -p tcp --dport 443 → 172.18.0.2:443          │
│     "Visitor for Port 443? Go to Apt 172.18.0.2!"      │
│        ↓                                                │
│                                                         │
│  4. 🌉 Internal Street (custom bridge, e.g. br-abc123) │
│     Switches packet to correct apartment                │
│     Looks up: 172.18.0.2 → veth123abc                  │
│        ↓                                                │
│                                                         │
│  5. 🔌 Fiber Cable (veth pair)                         │
│     veth123abc (host) → eth0 (container)                │
│        ↓                                                │
│                                                         │
│  6. 🏠 API Gateway Apartment (172.18.0.2)              │
│     nginx receives request                              │
│     Proxies to auth-service by name: "auth-service"    │
│        ↓                                                │
│                                                         │
│  7. 🎤 Intercom (Embedded DNS: 127.0.0.11)             │
│     "auth-service" → 172.19.0.5                         │
│        ↓                                                │
│                                                         │
│  8. 🌉 Another hallway trip (custom bridge again)      │
│     But wait! Different network!                        │
│     API Gateway has eth1 on app-net: 172.19.0.3        │
│     Routes through that interface                       │
│        ↓                                                │
│                                                         │
│  9. 🏠 Auth Service Apartment (172.19.0.5)             │
│     Validates JWT token                                 │
│     Needs to query database: "postgres"                │
│        ↓                                                │
│                                                         │
│  10. 🎤 Intercom again (DNS lookup)                    │
│      "postgres" → 172.20.0.10                           │
│        ↓                                                │
│                                                         │
│  11. 🔀 Auth has TWO apartments (multi-network!)       │
│      eth0: 172.19.0.5 (app-net)                        │
│      eth1: 172.20.0.3 (data-net)                       │
│      Routes through eth1 for database                   │
│        ↓                                                │
│                                                         │
│  12. 🏠 Database Apartment (172.20.0.10)               │
│      postgres executes:                                 │
│      SELECT * FROM users WHERE id = 123;                │
│        ↓                                                │
│                                                         │
│  === RESPONSE JOURNEY (Reverse Path) ===                │
│                                                         │
│  13. 📤 Database → Auth → API → veth → docker0 → eth0  │
│        ↓                                                │
│                                                         │
│  14. 🛡️ Exit Guard (iptables MASQUERADE)               │
│      Changes source: 172.18.0.2 → 10.0.1.50            │
│      "Tell visitor response is from building address"  │
│        ↓                                                │
│                                                         │
│  15. 🌐 Internet                                        │
│      User receives: {"id": 123, "name": "Alice"}       │
│      Thinks they talked to 10.0.1.50!                  │
│      (Has no idea about internal apartments)            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Performance implications:**

Each hop adds latency (~0.05-0.1ms):
- Host network interface
- iptables processing
- docker0 bridge switching
- veth pair traversal
- Container processing

**Total overhead:** ~0.2-0.5ms for container networking
**Solution if critical:** Use host network mode (removes all but container processing)

---

## Network Segmentation - Defense in Depth

Real-world production architecture uses **multiple networks** for security isolation:

```
┌─────────────────────────────────────────────────────────┐
│     PRODUCTION MICROSERVICES (20 Services)              │
│                                                         │
│  Tier 1: Public Network (external-net)                  │
│  ┌────────────────────────────────────────────┐        │
│  │  🌐 API Gateway                            │        │
│  │  🌐 Web Frontend                           │        │
│  └───────────────────┬────────────────────────┘        │
│                      │ Can talk down only              │
│                      ↓                                  │
│  Tier 2: Application Network (app-net)                  │
│  ┌────────────────────────────────────────────┐        │
│  │  🔧 Auth Service    🔧 User Service        │        │
│  │  🔧 Order Service   🔧 Payment Service     │        │
│  │  🔧 ... 15 more services ...              │        │
│  └───────────────────┬────────────────────────┘        │
│                      │ Can talk down only              │
│                      ↓                                  │
│  Tier 3: Data Network (data-net)                        │
│  ┌────────────────────────────────────────────┐        │
│  │  🗄️ PostgreSQL  🗄️ Redis  🗄️ MongoDB     │        │
│  └────────────────────────────────────────────┘        │
│                                                         │
│  Security Rules:                                        │
│  ✅ API Gateway can reach Auth Service                 │
│  ✅ Auth Service can reach PostgreSQL                  │
│  ❌ API Gateway CANNOT reach PostgreSQL                │
│  ❌ Internet CANNOT reach any database                 │
│                                                         │
│  Blast radius: Compromise web → Can't reach DB!        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Docker Compose example:**

```yaml
version: '3.8'

networks:
  external:
  app:
  data:

services:
  api-gateway:
    image: nginx
    networks:
      - external  # Public-facing
      - app       # Can reach app tier
    ports:
      - "443:443"

  auth-service:
    image: auth-app
    networks:
      - app       # Receives from API
      - data      # Can reach database
    # No public ports!

  postgres:
    image: postgres
    networks:
      - data      # ONLY on data network
    # Completely isolated from internet
```

---

## Troubleshooting - When Apartments Can't Communicate

**Systematic debugging approach:**

```bash
# Step 1: Are both containers running?
docker ps | grep container-name

# Step 2: Are they on the same network?
docker network inspect my-network

# Step 3: Can you ping by IP? (Eliminates DNS)
docker exec container-a ping 172.18.0.3

# Step 4: Can you ping by name? (Tests DNS)
docker exec container-a ping container-b

# Step 5: Try TCP instead of ICMP
docker exec container-a curl http://container-b:80
# (Sometimes ICMP is blocked, but TCP works)

# Step 6: Check DNS resolution
docker exec container-a nslookup container-b
docker exec container-a cat /etc/resolv.conf
# Should show: nameserver 127.0.0.11

# Step 7: Check routes
docker exec container-a ip route

# Step 8: Check network interfaces
docker exec container-a ip addr

# Step 9: Capture traffic
docker exec container-a tcpdump -i eth0 -n

# Step 10: Check iptables on host
sudo iptables -t nat -L DOCKER -n -v
```

**Common issues:**

| Symptom | Cause | Fix |
|---------|-------|-----|
| Can't ping by name | Default bridge (no DNS) | Use custom bridge |
| Can't ping by IP | Different networks | Connect to same network |
| Can ping but can't curl | Service not running | Check app logs |
| Connection refused | Wrong port | Verify listening port |
| DNS not resolving | Wrong DNS server | Check /etc/resolv.conf |

---

## Command Cheat Sheet

```bash
# ========== NETWORK MANAGEMENT ==========
docker network ls                        # List all networks
docker network inspect <network>         # See network details
docker network create my-net             # Create custom bridge
docker network rm my-net                 # Delete network

# ========== RUNNING CONTAINERS ==========
docker run --network my-net nginx        # Use custom network
docker run --network host nginx          # Host network mode
docker run --network none alpine         # No network
docker run -p 8080:80 nginx             # Publish port (all IPs)
docker run -p 127.0.0.1:8080:80 nginx   # Localhost only

# ========== MULTI-NETWORK ==========
docker network connect backend app       # Add to 2nd network
docker network disconnect backend app    # Remove from network

# ========== OVERLAY NETWORKS ==========
docker swarm init                        # Enable Swarm mode
docker network create -d overlay my-overlay
docker service create --network my-overlay nginx

# ========== MACVLAN / IPVLAN ==========
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 macvlan-net

docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  -o parent=eth0 \
  -o ipvlan_mode=l2 ipvlan-net

# ========== DEBUGGING ==========
docker exec container ip addr            # Check interfaces
docker exec container ip route           # Check routes
docker exec container nslookup hostname  # Test DNS
docker exec container tcpdump -i eth0    # Capture traffic
sudo iptables -t nat -L DOCKER -n -v    # See NAT rules
ip link | grep veth                      # See veth pairs on host
```

---

## Key Takeaways

### 1. Network Namespaces = Private Apartments
- Each container gets isolated network view
- Linux kernel provides the isolation (not magic!)

### 2. veth Pairs = Fiber Optic Cables
- Virtual ethernet connecting container to bridge
- Two ends: container sees `eth0`, host sees `veth123abc`

### 3. Bridge = Internal Hallway System
- Connects all containers on same network
- **Default bridge:** No DNS (must use IPs)
- **Custom bridge:** Has DNS (use container names)

### 4. Port Publishing = Reception Desk
- iptables DNAT forwards external traffic to container
- iptables MASQUERADE hides container IPs on outbound

### 5. Network Modes Matter
- **Bridge:** Standard (isolation + connectivity)
- **Host:** Performance (no isolation!)
- **None:** Maximum isolation (no network)
- **Macvlan:** Own MAC, appears as physical device
- **IPvlan:** Shared MAC, unique IPs

### 6. Overlay = Underground Tunnels
- VXLAN encapsulates packets for multi-host
- Requires orchestrator (Swarm/Kubernetes)
- Transparent to containers

### 7. Multi-Network = Two Apartments
- `docker network connect` adds second interface
- Use for network segmentation (defense in depth)

### 8. Always Use Custom Bridge
- Gets you embedded DNS (127.0.0.11)
- Better isolation than default bridge
- Production best practice

---



*Learned: January 1, 2026*
*Related: [Docker Architecture](../../../2025/december/docker/2025-12-25-docker-architecture.md), [Containers Fundamentals](../../../2025/december/docker/2025-12-23-containers-fundamentals.md), [Docker Image Layers](../../../2025/december/docker/2025-12-28-docker-image-layers.md)*
