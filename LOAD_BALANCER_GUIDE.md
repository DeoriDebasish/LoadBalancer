# Load Balancer — Complete Guide

> Goal: Read this document end-to-end and you will have expert-level understanding of load balancers — concepts, types, algorithms, architecture, and implementation.

---

## Table of Contents

1. [What is a Load Balancer?](#1-what-is-a-load-balancer)
2. [Why Do We Need a Load Balancer?](#2-why-do-we-need-a-load-balancer)
3. [Where Does a Load Balancer Sit?](#3-where-does-a-load-balancer-sit)
4. [Types of Load Balancers](#4-types-of-load-balancers)
   - 4.1 [Layer 4 (Transport Layer) Load Balancer](#41-layer-4-transport-layer-load-balancer)
   - 4.2 [Layer 7 (Application Layer) Load Balancer](#42-layer-7-application-layer-load-balancer)
   - 4.3 [DNS Load Balancer](#43-dns-load-balancer)
   - 4.4 [Hardware vs Software Load Balancer](#44-hardware-vs-software-load-balancer)
   - 4.5 [Client-Side Load Balancer](#45-client-side-load-balancer)
   - 4.6 [Global Server Load Balancer (GSLB)](#46-global-server-load-balancer-gslb)
5. [Load Balancing Algorithms](#5-load-balancing-algorithms)
   - 5.1 [Round Robin](#51-round-robin)
   - 5.2 [Weighted Round Robin](#52-weighted-round-robin)
   - 5.3 [Least Connections](#53-least-connections)
   - 5.4 [Weighted Least Connections](#54-weighted-least-connections)
   - 5.5 [IP Hash (Sticky Hashing)](#55-ip-hash-sticky-hashing)
   - 5.6 [Least Response Time](#56-least-response-time)
   - 5.7 [Random](#57-random)
   - 5.8 [Resource Based (Adaptive)](#58-resource-based-adaptive)
   - 5.9 [Consistent Hashing](#59-consistent-hashing)
6. [Health Checks](#6-health-checks)
7. [Session Persistence (Sticky Sessions)](#7-session-persistence-sticky-sessions)
8. [SSL/TLS Termination](#8-ssltls-termination)
9. [Connection Draining (Graceful Shutdown)](#9-connection-draining-graceful-shutdown)
10. [High Availability of the Load Balancer Itself](#10-high-availability-of-the-load-balancer-itself)
11. [Load Balancer vs Reverse Proxy vs API Gateway](#11-load-balancer-vs-reverse-proxy-vs-api-gateway)
12. [Real-World Implementations](#12-real-world-implementations)
13. [Key Metrics to Monitor](#13-key-metrics-to-monitor)
14. [Common Interview / Design Questions](#14-common-interview--design-questions)
15. [What We Will Build](#15-what-we-will-build)

---

## 1. What is a Load Balancer?

A **load balancer** is a component that distributes incoming network traffic across a pool of backend servers (also called an *upstream* or *server pool*).

```
                         ┌──────────────┐
                         │   Server A   │
Client ──► Load Balancer ├──────────────┤
                         │   Server B   │
                         ├──────────────┤
                         │   Server C   │
                         └──────────────┘
```

The load balancer is the single entry point. It decides **which server** handles each request based on a configurable algorithm.

---

## 2. Why Do We Need a Load Balancer?

| Problem | Without LB | With LB |
|---|---|---|
| Single server overload | One server gets all traffic → crashes | Traffic is spread across many servers |
| Server failure | Entire service goes down | LB detects failure and routes around it |
| Horizontal scaling | Adding more servers doesn't help | New servers register and receive traffic |
| Latency | Distant servers are always used | LB can pick geographically closer servers |
| Maintenance | Taking a server offline = downtime | LB drains connections, zero downtime |

**Core benefits:**
- **Scalability** — add or remove servers without downtime
- **High Availability** — no single point of failure in the backend
- **Performance** — distribute CPU/memory load evenly
- **Flexibility** — route traffic by content, user, geography, etc.

---

## 3. Where Does a Load Balancer Sit?

Load balancers can exist at multiple layers of your infrastructure:

```
Internet
   │
   ▼
[DNS / GSLB]          ← routes between data centers / regions
   │
   ▼
[Edge / CDN LB]       ← absorbs static traffic, DDoS protection
   │
   ▼
[L4/L7 LB]           ← your main application load balancer
   │
   ├── Web Servers
   │
   ├── [Internal LB]  ← between microservices
   │       ├── Auth Service
   │       ├── Payment Service
   │       └── Inventory Service
   │
   └── [DB LB]        ← read replicas, shards
```

---

## 4. Types of Load Balancers

### 4.1 Layer 4 (Transport Layer) Load Balancer

Operates at the **TCP/UDP layer**. It does NOT look at the content of packets — it only looks at IP addresses and port numbers.

#### How it works:
1. Client establishes a TCP connection to the LB's IP.
2. LB picks a backend server.
3. LB forwards the TCP stream to that server using **NAT (Network Address Translation)** or **IP tunneling**.
4. The backend server responds directly to the client (Direct Server Return) or back through the LB.

#### Two forwarding modes:

**NAT Mode:**
```
Client (10.0.0.1:50001)  →  LB (1.1.1.1:80)  →  Server A (192.168.1.1:80)
                           LB rewrites dst IP → Server A's IP
                           Return traffic comes back through LB
```

**Direct Server Return (DSR):**
```
Client (10.0.0.1:50001)  →  LB (1.1.1.1:80)  →  Server A (192.168.1.1:80)
                                                   Server A replies DIRECTLY to client
                           (LB only handles inbound — much faster)
```

#### Pros:
- Extremely fast — no packet inspection
- Works for any TCP/UDP protocol (HTTP, SMTP, MySQL, DNS…)
- Very low latency overhead

#### Cons:
- No content-based routing (can't route `/api` to one pool and `/static` to another)
- No HTTP header awareness
- Can't do SSL termination (stream is opaque)

#### Examples: AWS NLB, HAProxy (TCP mode), LVS (Linux Virtual Server)

---

### 4.2 Layer 7 (Application Layer) Load Balancer

Operates at the **HTTP/HTTPS layer**. It reads the full HTTP request before deciding where to route it.

#### How it works:
1. Client establishes a TCP + TLS connection to the LB.
2. LB reads the HTTP request (URL, headers, cookies, body).
3. LB picks a backend based on routing rules.
4. LB opens a **new TCP connection** to the backend and forwards the request.
5. LB buffers the backend response and sends it to the client.

```
Client ──TLS──► LB reads: GET /api/users HTTP/1.1
                           Host: example.com
                           Cookie: session=abc123
                LB decision: route to "api-pool"
                LB ──HTTP──► API Server
```

#### Routing rules possible at L7:
| Rule Type | Example |
|---|---|
| URL path | `/api/*` → API pool, `/static/*` → CDN pool |
| Host header | `api.example.com` → API servers |
| HTTP method | `GET` → read replicas, `POST` → primary |
| Cookie | `session_server=A` → Server A (sticky) |
| Header | `X-Region: EU` → EU servers |
| Query param | `?version=v2` → v2 pool |
| Request body | JSON field value routing |

#### Pros:
- Full content-aware routing
- SSL termination (offloads crypto from backend)
- Request/response modification (add headers, compress)
- Advanced health checks (HTTP 200 check)
- WebSocket support
- Rate limiting, authentication at the edge

#### Cons:
- Higher latency (must parse full request)
- Higher CPU usage (TLS handshake, HTTP parsing)
- Only works for HTTP/HTTPS (not raw TCP)

#### Examples: Nginx, HAProxy (HTTP mode), AWS ALB, Envoy, Traefik

---

### 4.3 DNS Load Balancer

Uses **DNS responses** to distribute traffic. When a client resolves a domain, the DNS server returns a different IP each time.

```
Client asks: "What is the IP of api.example.com?"
DNS responds: "10.0.0.1"  (next client gets "10.0.0.2", then "10.0.0.3"...)
```

#### Types:
- **Round-robin DNS** — rotate IPs in response
- **Geo DNS** — return IPs closest to the client's location
- **Latency-based DNS** — return IPs based on measured latency

#### Pros:
- No infrastructure needed — built into DNS
- Global distribution across data centers
- No single bottleneck

#### Cons:
- **No health awareness** — DNS doesn't know if a server is down
- **DNS caching (TTL)** — clients cache the IP; a server failure isn't instant
- **No session persistence** — client may get a different IP on retry
- **Uneven distribution** — one DNS response can serve thousands of clients

#### Used for: multi-region routing, initial traffic distribution between data centers

---

### 4.4 Hardware vs Software Load Balancer

| Aspect | Hardware LB | Software LB |
|---|---|---|
| Examples | F5 BIG-IP, Citrix ADC | Nginx, HAProxy, Envoy |
| Performance | Very high (custom ASIC) | High (commodity hardware) |
| Cost | Extremely expensive ($50k–$500k) | Free / open-source |
| Flexibility | Limited, vendor-locked | Fully configurable |
| Scalability | Fixed capacity | Scale horizontally |
| Deployment | Physical appliance | Any server / container / VM |
| Adoption today | Legacy enterprise | Modern standard |

Modern systems almost universally use **software load balancers** deployed on commodity hardware or in containers.

---

### 4.5 Client-Side Load Balancer

The **client itself** decides which server to call. No central LB component.

```
Service A (client)
    │
    ├── Service Registry (Consul / Eureka / etcd)
    │       └── "Service B has instances: 10.0.0.1, 10.0.0.2, 10.0.0.3"
    │
    └── Client-side LB logic picks instance → calls 10.0.0.2
```

#### Used in:
- Microservices (Netflix Ribbon, gRPC client-side LB)
- Service mesh sidecar proxies (Envoy in Istio)
- Database client libraries (MongoDB driver, Redis Cluster client)

#### Pros:
- No central LB bottleneck
- LB logic is per-service, fully customizable
- Works well in service-mesh architectures

#### Cons:
- Every client must implement LB logic
- Service discovery required
- Harder to observe and debug

---

### 4.6 Global Server Load Balancer (GSLB)

Routes traffic between **multiple data centers or regions** based on:
- Geographic proximity
- Data center health
- Latency measurements
- Capacity

```
User in India → GSLB → Mumbai Data Center (closest, healthy)
User in USA   → GSLB → Virginia Data Center (closest, healthy)
Mumbai goes down → GSLB shifts Indian users to Singapore
```

Implemented via Anycast routing or latency-based DNS.

#### Examples: AWS Route 53 (latency/geo routing), Cloudflare, Akamai

---

## 5. Load Balancing Algorithms

### 5.1 Round Robin

Requests are distributed sequentially across all servers, cycling back to the first.

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  (cycle repeats)
```

**State needed:** A single integer counter (current index).

```python
servers = ["A", "B", "C"]
index = 0

def get_server():
    global index
    server = servers[index % len(servers)]
    index += 1
    return server
```

**Best for:** Stateless servers with uniform capacity and similar request costs.

**Weakness:** Ignores server load. If one request takes 10 seconds and another takes 1ms, round-robin doesn't care.

---

### 5.2 Weighted Round Robin

Same as round robin but servers with higher weights receive proportionally more requests.

```
Server A: weight 3
Server B: weight 1
Server C: weight 2

Sequence: A, A, A, B, C, C, A, A, A, B, C, C ...
```

**Best for:** Heterogeneous server pools (e.g., one server has 2x CPU).

```
Server capacities: A=4 cores, B=4 cores, C=8 cores
Weights:          A=1,       B=1,       C=2
```

---

### 5.3 Least Connections

Route each new request to the server with the **fewest active connections**.

```
Server A: 10 active connections
Server B: 3 active connections   ← new request goes here
Server C: 7 active connections
```

**State needed:** Active connection count per server (atomically updated).

**Best for:** Long-lived connections (WebSockets, streaming, database connections) where connection count correlates with load.

**Weakness:** Doesn't account for request weight (a heavy query and a ping both count as "1 connection").

---

### 5.4 Weighted Least Connections

Combines weights with active connection count:

```
score = active_connections / weight

Server A: 10 connections / weight 2 = score 5
Server B: 3  connections / weight 1 = score 3   ← lowest, pick this
Server C: 7  connections / weight 3 = score 2.3 ← actually lowest!
```

Pick the server with the **lowest score**.

---

### 5.5 IP Hash (Sticky Hashing)

Hash the client's IP address to consistently route the same client to the same server.

```
hash(client_IP) % num_servers = server_index

Client 10.0.0.1 always → Server A
Client 10.0.0.2 always → Server B
Client 10.0.0.3 always → Server A
```

**Best for:** Stateful applications where the server caches session data locally and you want session affinity.

**Weakness:**
- Uneven distribution (some IP ranges may hash to the same bucket)
- If a server is removed, ALL clients rehash (use Consistent Hashing instead)
- Clients behind NAT share an IP → all go to same server

---

### 5.6 Least Response Time

Route to the server with the **lowest average response time** AND fewest active connections.

```
score = avg_response_time_ms * active_connections

Server A: 200ms * 5 = 1000
Server B: 50ms  * 3 = 150   ← lowest score, pick this
Server C: 100ms * 2 = 200
```

**Best for:** Applications with highly variable response times.

**State needed:** Rolling average of response times per server.

---

### 5.7 Random

Pick a backend server randomly.

```python
import random
def get_server():
    return random.choice(servers)
```

**Surprisingly effective** — by the law of large numbers, distribution is statistically even over many requests. No coordination needed (great for distributed LBs).

**Power of Two Random Choices (P2C):** Pick 2 random servers, route to the one with fewer connections. This dramatically improves over pure random with very little overhead.

---

### 5.8 Resource Based (Adaptive)

Servers periodically report their resource utilization (CPU%, memory%) to the LB. The LB routes traffic to the server with the most available capacity.

```
Server A reports: CPU 80%, Memory 70%  → busy
Server B reports: CPU 20%, Memory 30%  → idle  ← route here
```

**Best for:** Heterogeneous workloads where CPU is the bottleneck.

**Implementation:** Requires a side channel (health check endpoint that returns metrics, or agent on each server).

---

### 5.9 Consistent Hashing

A smarter version of IP Hash that minimizes redistribution when servers are added or removed.

#### The Problem with Simple Modulo Hashing:
```
3 servers: hash(key) % 3
Add 1 server: hash(key) % 4  → ~75% of keys remap to different servers!
```

#### Consistent Hashing Solution:
1. Place servers on a **virtual ring** (0 to 2^32).
2. Hash each server to a position on the ring.
3. For each request, hash the key and walk clockwise to the first server.
4. Adding/removing a server only remaps keys between two neighbors (~1/N of keys).

```
Ring (0 ──────────────── 2^32):

         Server A (hash=100)
        /
     ──●────────────────────────────────●──
      /                                  \
    ●  key=90 → walks clockwise → A       ●  Server C (hash=300)
      \                                  /
     ──●────────────────────────────────●──
                  \
                   Server B (hash=200)

key=150 hashes to 150 → walks clockwise → hits Server B at 200
```

#### Virtual Nodes:
Each physical server maps to **multiple positions** on the ring (virtual nodes). This ensures even distribution even with few servers.

```
Server A → positions: 50, 200, 400, 700  (4 virtual nodes)
Server B → positions: 100, 300, 500, 800
```

**Used in:** Cassandra, DynamoDB, Redis Cluster, Nginx upstream hashing, CDN cache routing.

---

## 6. Health Checks

The LB must detect when a backend is down and stop routing to it.

### Types of Health Checks

| Type | What it checks | Example |
|---|---|---|
| **TCP check** | Can we open a connection? | `TCP connect to :8080` |
| **HTTP check** | Does it return 200? | `GET /health → 200 OK` |
| **HTTPS check** | HTTP + TLS validation | `GET /health over TLS` |
| **Script check** | Custom logic | Shell script exit code 0 = healthy |
| **gRPC check** | gRPC health protocol | `grpc.health.v1.Health/Check` |

### Health Check Parameters

```
interval:           10s   — how often to check
timeout:            2s    — how long to wait for response
healthy_threshold:  2     — consecutive successes before marking healthy
unhealthy_threshold: 3    — consecutive failures before marking unhealthy
```

### State Machine

```
           ┌─────────────────────────────────┐
           │                                 │
STARTING ──► HEALTHY ──(3 fails)──► UNHEALTHY
                ▲                       │
                └──────(2 passes)───────┘
```

When a backend is `UNHEALTHY`:
- New connections are not routed to it
- Existing connections may be allowed to finish (drain)
- LB may alert/notify operators

### Active vs Passive Health Checks

- **Active:** LB proactively sends health check requests on a timer.
- **Passive:** LB monitors real traffic. If responses are errors or timeouts, mark unhealthy. No extra traffic, but detection is slower.

---

## 7. Session Persistence (Sticky Sessions)

Some applications store user state on the server (shopping cart in memory, file upload in progress). The same user must always hit the same server.

### Methods

**1. Cookie-based stickiness (preferred):**
```
Client makes first request → LB picks Server A → LB injects cookie:
  Set-Cookie: LB_SERVER=server_a; Path=/

Next request has cookie → LB reads it → routes to Server A
```

**2. IP Hash (as described in §5.5).**

**3. SSL Session ID:**
The TLS session ID or TLS session ticket is hashed to pick a server.

### Downsides of Sticky Sessions
- Uneven load distribution (popular users hit one server)
- If the server fails, that user's session is lost anyway
- **Better approach:** Store session in a shared store (Redis, Memcached) — then any server can handle any user.

---

## 8. SSL/TLS Termination

The LB handles the TLS handshake so backend servers don't have to.

```
Client ──TLS──► LB (decrypt) ──HTTP──► Backend Server
                 ▲
                 Certificate lives here
```

### Modes

| Mode | Description | Use case |
|---|---|---|
| **SSL Termination** | LB decrypts, backends get plain HTTP | Most common, easiest |
| **SSL Passthrough** | LB forwards encrypted stream (L4) | End-to-end encryption required |
| **SSL Re-encryption** | LB decrypts, re-encrypts to backend | Compliance (internal traffic encrypted) |

### Benefits of Termination at LB:
- Backends are simpler (no cert management)
- LB can inspect HTTP headers (required for L7 routing)
- Central certificate renewal (one place)
- Offloads CPU-intensive crypto from app servers

---

## 9. Connection Draining (Graceful Shutdown)

When a backend is taken out of rotation (deploy, maintenance, failure), existing in-flight requests should complete.

```
1. LB marks Server A as "draining" (no new connections)
2. Existing connections to Server A continue until:
   - They complete naturally, OR
   - A drain timeout is reached (e.g., 30 seconds)
3. After drain timeout, LB forcefully closes remaining connections
4. Server A is fully removed
```

This enables **zero-downtime deploys**: take servers out one by one, deploy, bring them back.

---

## 10. High Availability of the Load Balancer Itself

The LB can become a single point of failure. Solutions:

### Active-Passive (Hot Standby)
```
         ┌─── LB Primary (active) ───►  Backends
Client ──┤
         └─── LB Standby (passive)  (monitors primary via heartbeat)

If primary fails → Standby takes over Virtual IP → Client traffic continues
```

Uses **VRRP (Virtual Router Redundancy Protocol)** or **keepalived** for VIP failover.

**Failover time:** Seconds (VIP moves to standby).

### Active-Active
```
         ┌─── LB Node 1 ───►  Backends
Client ──┤  (DNS round-robin or Anycast)
         └─── LB Node 2 ───►  Backends
```

Both LBs handle traffic simultaneously. If one fails, DNS/Anycast naturally routes to the other.

**Benefit:** Better utilization, no idle standby.

**Challenge:** Both LBs must share state (connection tables) if using stateful algorithms.

### Anycast Routing
Multiple LBs share the same IP address in different locations. BGP routes traffic to the nearest one. Used by Cloudflare, Fastly.

---

## 11. Load Balancer vs Reverse Proxy vs API Gateway

These terms are often used interchangeably but have distinct roles:

| Feature | Load Balancer | Reverse Proxy | API Gateway |
|---|---|---|---|
| Primary purpose | Distribute load | Proxy requests to backend | API management |
| Multiple backends | Yes | Sometimes | Yes |
| Routing logic | Server selection | Simple proxy | URL/method/header routing |
| Auth / authz | No | No | Yes |
| Rate limiting | Basic | Basic | Full |
| Request transformation | No | Sometimes | Yes |
| API versioning | No | No | Yes |
| Analytics / billing | No | No | Yes |
| SSL termination | Yes | Yes | Yes |
| Examples | AWS NLB, HAProxy | Nginx | Kong, AWS API Gateway |

**In practice:** Nginx and HAProxy can function as all three. Modern systems often layer them:

```
Internet → API Gateway (auth, rate limit) → LB (traffic distribution) → Backends
```

---

## 12. Real-World Implementations

### Nginx
- Software L7 LB + Reverse Proxy
- Event-driven, single-threaded event loop (handles 10k+ concurrent connections)
- Config example:
```nginx
upstream backend_pool {
    least_conn;
    server 10.0.0.1:8080 weight=3;
    server 10.0.0.2:8080 weight=1;
    server 10.0.0.3:8080 backup;      # only used if others fail
}

server {
    listen 80;
    location / {
        proxy_pass http://backend_pool;
    }
}
```

### HAProxy
- High-performance TCP + HTTP LB
- Used by GitHub, Reddit, Stack Overflow
- Exceptional observability (stats socket, live reloads)

### Envoy
- Modern L4/L7 proxy, designed for service meshes
- Used as sidecar in Istio, as edge proxy at Lyft/Airbnb
- gRPC native, HTTP/2 native, observability built-in

### AWS Elastic Load Balancing
| Type | Layer | Use case |
|---|---|---|
| ALB (Application LB) | 7 | HTTP/HTTPS microservices, path routing |
| NLB (Network LB) | 4 | Ultra-low latency, TCP/UDP, static IPs |
| GLB (Gateway LB) | 3 | Network appliances, firewall insertion |
| CLB (Classic LB) | 4+7 | Legacy, avoid for new systems |

---

## 13. Key Metrics to Monitor

| Metric | What it tells you |
|---|---|
| **Requests per second (RPS)** | Throughput |
| **Active connections** | Current load |
| **Backend latency (p50/p95/p99)** | Performance distribution |
| **Error rate (4xx/5xx)** | Health of backends |
| **Connection queue depth** | Saturation (are clients waiting?) |
| **Health check pass/fail ratio** | Backend fleet health |
| **Bytes in/out** | Network utilization |
| **SSL handshake time** | TLS overhead |
| **Connection draining time** | Deploy health |

---

## 14. Common Interview / Design Questions

**Q: How do you handle a backend server going down mid-request?**
The LB should detect the failure (timeout or connection refused), mark the server unhealthy, and retry the request on a different server (only safe for idempotent requests — GET, not POST).

**Q: What happens if the load balancer itself goes down?**
Deploy LBs in active-active or active-passive pairs with a shared Virtual IP. Use VRRP/keepalived for failover.

**Q: How do you do zero-downtime deployment?**
Connection draining: remove one server from the pool, wait for in-flight requests to complete, deploy, add back. Repeat for each server.

**Q: Round robin vs Least Connections — when to use each?**
Round robin: uniform request cost, stateless. Least connections: long-lived connections or highly variable request cost.

**Q: How does consistent hashing help with cache servers?**
When a cache node is added or removed, only ~1/N keys are remapped (vs 100% with modulo hashing), minimizing cache misses.

**Q: How do you make a load balancer stateless?**
Use cookie-based session routing but store actual session data in Redis/Memcached. Any LB instance can handle any request.

**Q: L4 vs L7 — which is faster and why?**
L4 is faster because it doesn't parse the HTTP payload. It just rewrites IP headers and forwards bytes. L7 must buffer and parse the full HTTP request, perform a TLS handshake, and open a new connection to the backend.

---

## 15. What We Will Build

We will implement the following load balancers in code, in order of increasing complexity:

| # | Type | Algorithm | Key Learning |
|---|---|---|---|
| 1 | **Basic Round Robin LB** | Round Robin | Core proxy loop, request forwarding |
| 2 | **Weighted Round Robin LB** | Weighted RR | Server weights, proportional distribution |
| 3 | **Least Connections LB** | Least Connections | Concurrency tracking, atomic counters |
| 4 | **IP Hash LB** | IP Hash | Consistent client affinity |
| 5 | **Health-Aware LB** | Round Robin + Health | Health check goroutines, circuit breaker |
| 6 | **L7 Content-Based LB** | Path routing | HTTP parsing, routing rules |
| 7 | **Consistent Hashing LB** | Consistent Hashing | Virtual ring, minimal redistribution |

Each will be a working Go HTTP proxy you can run locally with multiple backend servers.

---

## Quick Reference Cheat Sheet

```
CHOOSE YOUR ALGORITHM:
─────────────────────
Equal capacity, stateless requests?          → Round Robin
Unequal server capacity?                     → Weighted Round Robin
Long-lived connections (WebSocket, DB)?      → Least Connections
Need same client → same server (session)?    → IP Hash or Cookie Sticky
Variable response times?                     → Least Response Time
Cache / shard routing?                       → Consistent Hashing
Simple, distributed, no coordination?        → Random / P2C

CHOOSE YOUR LB TYPE:
─────────────────────
Any TCP/UDP protocol, ultra-low latency?     → L4 Load Balancer
HTTP routing, SSL termination, inspection?   → L7 Load Balancer
Multi-region / multi-data-center?            → DNS LB / GSLB
Between microservices?                       → Client-Side / Service Mesh
```

---

*Next step: We build Load Balancer #1 — Basic Round Robin in Go.*
