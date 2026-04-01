
## 1. What Is a Load Balancer?

A load balancer sits between clients and backend servers, distributing incoming requests so no single server becomes a bottleneck.

```
                      ┌──────────┐
                   ┌─▶│ Server A │
                   │  └──────────┘
Client ──▶ [LB] ───┼─▶ Server B │
                   │  └──────────┘
                   └─▶│ Server C │
                      └──────────┘
```

Core goals:

- **Distribute load** evenly so no server is overwhelmed
- **Eliminate single points of failure** — if one server dies, traffic reroutes
- **Enable horizontal scaling** — add servers transparently to clients
- **SSL termination** — decrypt HTTPS once at the LB, use HTTP internally
- **Provide a single entry point** — clients only know one IP

---

## 2. L4 vs L7 Load Balancing

This is one of the most common interview topics. Know the difference deeply.

### L4 — Transport Layer (TCP/UDP)

Operates at the **network transport layer**. Makes routing decisions based on:

- Source/destination IP address
- Source/destination port
- Protocol (TCP/UDP)

It does **not** look at the content of the request — it doesn't know if it's HTTP, gRPC, or raw binary.

```
Client → LB [sees: IP=1.2.3.4, port=443] → Server A
              (doesn't know it's an /api/users request)
```

**Characteristics:**

- Very fast — minimal processing
- Cannot route based on URL path, headers, cookies
- Simpler to implement and configure
- Used for: raw TCP traffic, streaming, gaming, any non-HTTP protocol

**Examples:** AWS Network Load Balancer (NLB), HAProxy in TCP mode

---

### L7 — Application Layer (HTTP/HTTPS)

Operates at the **application layer**. Terminates the connection, reads the full request, then makes routing decisions based on:

- HTTP method (GET, POST)
- URL path (`/api/users` vs `/api/orders`)
- HTTP headers (`Host`, `Authorization`, `Cookie`)
- Request body (less common, expensive)

```
Client → LB [sees: GET /api/users → route to User Service]
              [sees: GET /api/orders → route to Order Service]
```

**Characteristics:**

- Content-aware routing (path-based, host-based)
- Can modify requests/responses (add headers, redirect)
- SSL termination happens here
- Can make intelligent decisions (e.g., send large uploads to specific servers)
- Slightly more overhead than L4

**Examples:** AWS Application Load Balancer (ALB), Nginx, Envoy, Traefik

---

### When to use which

|Use Case|Layer|
|---|---|
|Route `/api` to backend, `/static` to CDN|L7|
|Microservices routing by path or hostname|L7|
|Raw TCP (database, game server, streaming)|L4|
|Maximum throughput, minimum latency|L4|
|gRPC (HTTP/2 based)|L7|
|WebSocket connections|L7 (or L4)|

---

## 3. Load Balancing Algorithms

### Round Robin

Requests are distributed sequentially to each server in turn.

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  (wraps around)
```

**Pros:** Simple, even distribution if all requests are similar cost. **Cons:** Doesn't account for varying request complexity or server load. A slow `POST /upload` and a fast `GET /health` are treated equally.

---

### Weighted Round Robin

Like round robin but servers get a weight proportional to their capacity.

```
Server A (weight=3): handles 3 requests per cycle
Server B (weight=1): handles 1 request per cycle

Cycle: A, A, A, B, A, A, A, B...
```

**Use case:** Heterogeneous servers — newer/more powerful servers get a higher weight.

---

### Least Connections

New requests go to the server with the **fewest active connections**.

```
Server A: 5 active connections
Server B: 2 active connections  ← next request goes here
Server C: 8 active connections
```

**Pros:** Better than round robin for long-lived connections or variable request duration. **Cons:** Doesn't account for connection weight — a server with 2 heavy connections may be busier than one with 5 light ones.

---

### Least Response Time

New requests go to the server with the lowest combination of active connections and average response time.

```
Score = active_connections × avg_response_time

Server A: 2 connections × 50ms = 100
Server B: 5 connections × 10ms = 50  ← pick this
Server C: 1 connection  × 200ms = 200
```

**Use case:** Heterogeneous workloads where some requests are inherently slower.

---

### IP Hash

Hash the client's IP to always route to the same server.

```
hash("192.168.1.1") % 3 = 2 → always Server C
hash("10.0.0.5")    % 3 = 0 → always Server A
```

**Use case:** When you need session stickiness without cookies (e.g., the client doesn't support cookies, or you're doing L4 load balancing).

**Caveat:** If a server goes down, all clients mapped to it suddenly get remapped. Consider consistent hashing instead.

---

### Random

Pick a server at random. Surprisingly effective at scale (by the law of large numbers, distribution evens out). Used by some modern service meshes.

---

## 4. Sticky Sessions

**The problem:** HTTP is stateless, but many applications store session state in-memory on the server. If a user's requests go to different servers, their session data isn't there.

**Solution: Sticky sessions (session affinity)** — route a user's requests to the same server for the lifetime of their session.

### Methods

**Cookie-based stickiness (L7):**

```
First request:
  Client → LB → Server A
  LB sets cookie: SERVERID=A

Subsequent requests:
  Client sends SERVERID=A → LB routes to Server A
```

**IP hash-based stickiness (L4 or L7):** Route based on client IP (as described above in IP Hash).

### Problems with sticky sessions

- **Server failure** breaks all sessions on that server
- **Load imbalance** — a few heavy users all stick to one server
- **Scaling** — adding servers doesn't help current users since they're stuck

**Better alternative:** Externalize session state to a shared store (Redis, Memcached). Then any server can handle any request, making load balancers truly stateless.

```
Before:  Client → LB → Server A (session in memory)
After:   Client → LB → any server → Redis (session store)
```

---

## 5. Health Checks

Load balancers need to know which servers are healthy before routing traffic.

### Active health checks

The LB periodically polls each server:

```
LB → GET /health → Server A → 200 OK (healthy)
LB → GET /health → Server B → timeout  (unhealthy → remove from pool)
```

**Configuration parameters:**

```
interval:          5s   (how often to check)
timeout:           2s   (how long to wait for response)
unhealthy_threshold: 3  (fail N times before marking unhealthy)
healthy_threshold:   2  (pass N times before marking healthy again)
```

### Passive health checks

Monitor real traffic responses. If a server returns too many 5xx errors or timeouts, remove it.

**Combination:** Use active checks for fast detection of dead servers, passive checks for detecting degraded servers still returning 200s but slowly.

---

## 6. Load Balancer Architectures

### Single load balancer (single point of failure)

```
Client → [LB] → Servers
```

The LB itself is a SPOF. Unacceptable for production.

### Active-Passive (HA pair)

```
Client → [LB-Primary] → Servers
              ↕ heartbeat
         [LB-Standby]
```

Primary handles all traffic. Standby monitors primary. On failure, standby takes over the virtual IP (VIP) via VRRP/keepalived. Failover typically takes 2-30 seconds.

### Active-Active (HA pair)

Both LBs handle traffic simultaneously. DNS round-robins between them.

```
Client → DNS → [LB-1] → Servers
            ↘ [LB-2] → Servers
```

### Multi-tier load balancing

Large systems use multiple LB tiers:

```
Internet → [Global LB / Anycast] → Data Center A
                                 → Data Center B

Inside DC:
           → [External LB] → [Internal LB] → Service Instances
```

**Global LB** (L4 Anycast or GeoDNS) routes to the nearest data center. **External LB** (L7) routes to the correct service. **Internal LB** distributes within a service cluster.

---

## 7. DNS Load Balancing

Return multiple A records for a domain. Clients pick one (usually the first).

```
api.example.com → 1.2.3.4 (Server A)
api.example.com → 1.2.3.5 (Server B)
api.example.com → 1.2.3.6 (Server C)
```

**Pros:** Simple, works at global scale (GeoDNS). **Cons:** Clients cache DNS, so failover is slow (TTL-dependent). Not true load balancing — clients may all pick the first record.

**GeoDNS:** Return different IPs based on the client's location. US clients get US servers, EU clients get EU servers.

---

## 8. Real-World Tools

|Tool|Type|Notes|
|---|---|---|
|AWS ALB|L7|Path/host routing, WebSocket, gRPC, WAF integration|
|AWS NLB|L4|Ultra-low latency, static IP, TLS passthrough|
|Nginx|L4 + L7|Widely used, highly configurable, reverse proxy|
|HAProxy|L4 + L7|High performance, fine-grained health checks|
|Envoy|L7|Service mesh sidecar, observability built-in|
|Traefik|L7|Cloud-native, auto-discovers Kubernetes services|
|Cloudflare|L4 + L7|Global, DDoS protection, WAF|

---

## 9. Common Interview Questions + Answers

### Q: What's the difference between L4 and L7 load balancing?

> "L4 operates at the TCP layer and routes based on IP/port without inspecting the request content. It's fast but dumb. L7 operates at the HTTP layer — it reads the request and can route based on URL path, headers, or cookies. L7 is needed for path-based routing (e.g., `/api` to one service, `/static` to another), but L4 is better for non-HTTP protocols and maximum throughput."

### Q: How do you make a load balancer itself highly available?

> "Run two LBs in an active-passive or active-active pair. They share a virtual IP (VIP). In active-passive, the standby monitors the primary via heartbeat and takes over the VIP on failure using VRRP. In active-active, DNS round-robins between them. At global scale, Anycast routing lets multiple PoPs share the same IP — traffic naturally routes to the nearest healthy one."

### Q: Round robin vs least connections — when do you use each?

> "Round robin works well when requests have similar cost and duration — like a simple API. Least connections is better for workloads with high variance in request duration, like an API that has both fast reads and slow report generation. If some requests take 10x longer than others, round robin will pile them all on a few servers while least connections distributes active work more evenly."

### Q: How would you handle session stickiness at scale?

> "Ideally, I'd eliminate the need for it by moving session state out of application servers and into a shared Redis cluster. Then any server can handle any request. If I had to use sticky sessions — say, for a legacy app — I'd use cookie-based affinity at L7. But I'd treat it as technical debt and design a path to externalise state."

---

## 10. Interview Tricks & Pitfalls

### ✅ Trick 1: Always mention SSL termination

SSL termination at the LB is almost always the right answer for HTTPS. Say:

> "The LB terminates TLS, so backend servers communicate over plain HTTP on the internal network. This simplifies certificate management and reduces compute overhead on app servers."

### ✅ Trick 2: Know the LB is itself a SPOF

Whenever you draw a load balancer, the interviewer may ask "what if it fails?" Have the answer ready: active-passive pair with VIP failover, or active-active.

### ✅ Trick 3: Connect to the broader system

A load balancer question is rarely standalone. Tie it to:

- Auto-scaling (LB needs to register/deregister new instances)
- Health checks (LB needs to know when to remove instances)
- Service discovery (dynamic backends in microservices)

### ❌ Pitfall 1: Drawing a single LB without addressing HA

A single LB is a SPOF. In any serious design, address how the LB is made redundant.

### ❌ Pitfall 2: Forgetting health checks

A LB without health checks will happily route to dead servers. Health check configuration is part of the design.

### ❌ Pitfall 3: Using sticky sessions as the first answer

Sticky sessions are a band-aid. The better answer is always: "externalise session state, then you don't need stickiness."

---

## 11. Quick Reference

```
L4 (Transport):
  Routes by IP + port
  Fast, protocol-agnostic
  Use for: TCP, UDP, raw streaming, DB proxies

L7 (Application):
  Routes by URL, headers, cookies
  SSL termination here
  Use for: HTTP, gRPC, WebSocket, microservices

Algorithms:
  Round Robin        → equal servers, equal requests
  Weighted RR        → heterogeneous server capacity
  Least Connections  → variable request duration
  Least Response     → best when response time matters
  IP Hash            → simple stickiness (use cookies instead)

High Availability:
  Active-Passive     → VIP failover via VRRP, ~seconds downtime
  Active-Active      → DNS round-robin both, no downtime

Health checks:
  Active  → LB polls /health endpoint
  Passive → LB monitors live traffic error rates

Sticky sessions:
  Cookie-based (L7) or IP hash (L4)
  Better: externalise session to Redis → no stickiness needed

Tools: ALB (L7), NLB (L4), Nginx, HAProxy, Envoy
```