## 1. What Is an API Gateway?

An API Gateway is a single entry point for all client requests to backend services. It sits between clients and microservices, handling cross-cutting concerns.

```
                    ┌──────────────┐
Clients ────────▶   │ API Gateway  │
(Web, Mobile,       │              │
 Third-party)       └──────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   [User Service]    [Order Service]   [Payment Service]
```

**Without API Gateway:**

```
Client → User Service (auth)
Client → Order Service (auth)
Client → Payment Service (auth)

Each service handles: auth, rate limiting, logging, SSL
Code duplication, inconsistent behavior
```

**With API Gateway:**

```
Client → API Gateway (auth, rate limiting, logging, SSL)
         → Routes to correct service

Cross-cutting concerns centralized
Services focus on business logic
```

---

## 2. Core Responsibilities

### 1. Request routing

Route requests to the correct backend service.

```
GET /api/users/123        → User Service
GET /api/orders/456       → Order Service
POST /api/payments        → Payment Service
GET /api/products?q=laptop → Product Service
```

**Path-based routing:**

```
/api/users/*    → User Service
/api/orders/*   → Order Service
/api/payments/* → Payment Service
```

**Host-based routing:**

```
api.example.com     → API Gateway → Services
admin.example.com   → Admin Gateway → Admin Services
```

---

### 2. Authentication & Authorization

Verify identity and permissions before routing.

```
Request flow:
  1. Client sends request with JWT token
  2. Gateway validates token
     - Check signature
     - Check expiration
     - Extract user ID and roles
  3. If valid → route to service with user context
  4. If invalid → return 401 Unauthorized
```

**Example:**

```
GET /api/orders/123
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

Gateway:
  1. Decode JWT
  2. Verify signature
  3. Check expiration
  4. Extract: user_id=456, role=customer
  5. Check: Can user 456 access order 123?
  6. If yes → forward to Order Service with headers:
     X-User-ID: 456
     X-User-Role: customer
```

**Benefits:**
- Services don't need to validate tokens (trust gateway)
- Centralized auth logic
- Easy to change auth mechanism

---

### 3. Rate limiting

Protect services from overload and abuse.

```
Rate limit: 1000 requests/minute per API key

Request arrives:
  1. Gateway checks: How many requests from this API key in last minute?
  2. If < 1000 → allow, increment counter
  3. If >= 1000 → reject with 429 Too Many Requests
```

**Per-user, per-IP, or per-API-key limits.**

---

### 4. SSL/TLS termination

Decrypt HTTPS at gateway, use HTTP internally.

```
Client ──HTTPS──▶ Gateway ──HTTP──▶ Services
                   (decrypt)        (internal network)
```

**Benefits:**
- Services don't need SSL certificates
- Reduced compute on services (no decryption)
- Centralized certificate management

---

### 5. Request/Response transformation

Modify requests or responses.

```
Request transformation:
  Client sends: GET /v1/users/123
  Gateway transforms: GET /users/123
  (removes version prefix)

Response transformation:
  Service returns: {"user_id": 123, "name": "Alice"}
  Gateway adds: {"user_id": 123, "name": "Alice", "api_version": "v1"}
```

**Use cases:**
- API versioning
- Legacy API compatibility
- Adding/removing headers
- Response filtering (remove sensitive fields)

---

### 6. Load balancing

Distribute requests across service instances.

```
Order Service has 3 instances:
  Instance 1: 10.0.1.1
  Instance 2: 10.0.1.2
  Instance 3: 10.0.1.3

Gateway load balances:
  Request 1 → Instance 1
  Request 2 → Instance 2
  Request 3 → Instance 3
  Request 4 → Instance 1 (round-robin)
```

---

### 7. Caching

Cache responses to reduce backend load.

```
GET /api/products/123

First request:
  Gateway → Product Service → response
  Cache response for 60 seconds

Subsequent requests (within 60s):
  Gateway → return cached response (no service call)
```

**Cache headers:**

```
Cache-Control: public, max-age=60
```

---

### 8. Logging & Monitoring

Centralized logging for all requests.

```
Log entry:
  timestamp: 2024-01-15T10:30:00Z
  method: GET
  path: /api/orders/123
  user_id: 456
  status: 200
  latency: 45ms
  service: order-service
```

**Metrics:**
- Request rate (requests/second)
- Error rate (5xx responses)
- Latency (p50, p95, p99)
- Per-service metrics

---

## 3. API Gateway Patterns

### Backend for Frontend (BFF)

Different gateways for different clients.

```
Mobile App  → Mobile Gateway → Services
Web App     → Web Gateway    → Services
Partner API → Partner Gateway → Services
```

**Why:** Different clients have different needs.

```
Mobile Gateway:
  - Smaller payloads (bandwidth-constrained)
  - Aggregated responses (reduce round trips)
  - Push notifications

Web Gateway:
  - Full payloads
  - Separate requests OK (fast network)

Partner Gateway:
  - Strict rate limiting
  - API key authentication
  - Detailed billing/usage tracking
```

---

### API Composition (Aggregation)

Gateway calls multiple services and combines responses.

```
GET /api/user-dashboard/123

Gateway:
  1. Call User Service: GET /users/123
  2. Call Order Service: GET /orders?user_id=123
  3. Call Payment Service: GET /payment-methods?user_id=123
  4. Combine responses:
     {
       "user": {...},
       "recent_orders": [...],
       "payment_methods": [...]
     }
  5. Return to client

Client makes 1 request instead of 3
```

**Tradeoff:** Gateway becomes more complex, couples services.

---

### Protocol translation

Convert between protocols.

```
Client (REST) → Gateway → gRPC → Services

Gateway translates:
  HTTP GET /api/users/123
  →
  gRPC GetUser(id=123)
```

**Use case:** Internal services use gRPC for performance, external API uses REST for compatibility.

---

## 4. API Gateway vs Reverse Proxy vs Load Balancer

|Feature|API Gateway|Reverse Proxy|Load Balancer|
|---|---|---|---|
|Routing|Path/host-based|Host-based|Round-robin/least-conn|
|Auth|Yes|Limited|No|
|Rate limiting|Yes|Yes|Limited|
|Request transformation|Yes|Limited|No|
|SSL termination|Yes|Yes|Yes|
|Caching|Yes|Yes|No|
|Service discovery|Yes|No|No|
|Protocol translation|Yes|No|No|
|Use case|Microservices API|Web server frontend|Distribute load|

**API Gateway = Reverse Proxy + Auth + Rate Limiting + More**

---

## 5. Service Discovery Integration

Gateway needs to know where services are.

### Static configuration

```yaml
services:
  user-service:
    url: http://10.0.1.1:8080
  order-service:
    url: http://10.0.2.1:8080
```

**Problem:** Manual updates when services move or scale.

---

### Dynamic service discovery

```
Gateway → Service Registry (Consul, Eureka) → "Where is user-service?"
Registry → "3 instances: 10.0.1.1, 10.0.1.2, 10.0.1.3"
Gateway → Load balance across instances
```

**Services register themselves on startup, deregister on shutdown.**

**Benefits:**
- Auto-scaling works seamlessly
- No manual configuration
- Health checks (registry removes unhealthy instances)

---

## 6. API Gateway Technologies

### AWS API Gateway

**Type:** Managed service

**Features:**
- Serverless (no infrastructure)
- Integrates with Lambda, HTTP endpoints
- Built-in auth (Cognito, IAM, custom authorizers)
- Rate limiting, caching, monitoring
- Pay per request

**Use for:** AWS-native apps, serverless architectures

---

### Kong

**Type:** Open-source, self-hosted

**Features:**
- Plugin ecosystem (auth, rate limiting, logging, etc.)
- Built on Nginx (high performance)
- Service discovery (Consul, Eureka)
- Admin API for configuration

**Use for:** On-prem or self-managed cloud, need full control

---

### Nginx / Nginx Plus

**Type:** Reverse proxy + API gateway features

**Features:**
- High performance (C-based)
- SSL termination, caching, load balancing
- Lua scripting for custom logic
- Nginx Plus adds: service discovery, health checks, dashboard

**Use for:** High-performance needs, existing Nginx expertise

---

### Envoy

**Type:** Modern proxy, service mesh sidecar

**Features:**
- HTTP/2, gRPC support
- Advanced load balancing
- Observability (metrics, tracing)
- Used in Istio service mesh

**Use for:** Kubernetes, service mesh, modern protocols

---

### Traefik

**Type:** Cloud-native, Kubernetes-friendly

**Features:**
- Auto-discovers services (Kubernetes, Docker)
- Let's Encrypt integration (auto SSL)
- Dashboard, metrics
- Middleware (auth, rate limiting, etc.)

**Use for:** Kubernetes, Docker, dynamic environments

---

## 7. API Gateway Challenges

### Single point of failure

Gateway is critical path for all requests.

**Mitigation:**
- Run multiple gateway instances (active-active)
- Load balancer in front of gateways
- Health checks and auto-scaling

```
         ┌──────────────┐
Clients ─┤ Load Balancer│
         └──────┬───────┘
                │
        ┌───────┼───────┐
        │       │       │
   [Gateway] [Gateway] [Gateway]
```

---

### Latency overhead

Every request goes through gateway.

```
Without gateway: Client → Service (10ms)
With gateway: Client → Gateway (5ms) → Service (10ms) = 15ms
```

**Mitigation:**
- Keep gateway logic lightweight
- Cache aggressively
- Use fast gateway (Envoy, Nginx)

---

### Complexity

Gateway can become a monolith.

```
Gateway handles:
  - Auth for 10 services
  - Rate limiting for 20 API keys
  - Custom logic for 5 partners
  - Response transformation for 3 clients

→ Gateway becomes complex, hard to change
```

**Mitigation:**
- Keep gateway thin (routing + cross-cutting concerns only)
- Push business logic to services
- Use BFF pattern (separate gateways per client type)

---

## 8. Common Interview Questions + Answers

### Q: What's the purpose of an API Gateway?

> "An API Gateway is a single entry point for all client requests. It handles cross-cutting concerns like authentication, rate limiting, SSL termination, and logging so individual services don't have to. It routes requests to the correct backend service, can transform requests/responses, and provides a unified API to clients even when the backend is composed of many microservices. This centralizes common functionality and simplifies client integration."

### Q: What's the difference between an API Gateway and a load balancer?

> "A load balancer distributes traffic across multiple instances of the same service using algorithms like round-robin or least-connections. It operates at L4 (TCP) or L7 (HTTP) but doesn't understand application logic. An API Gateway operates at L7 and understands the API — it routes based on URL paths, handles authentication and authorization, enforces rate limits, transforms requests, and can aggregate responses from multiple services. A load balancer is about distributing load; an API Gateway is about managing the API."

### Q: How does an API Gateway handle authentication?

> "The gateway validates authentication tokens (like JWTs) before routing requests. It checks the token signature, expiration, and extracts user identity and roles. If valid, it forwards the request to the backend service with user context in headers (like X-User-ID). If invalid, it returns 401 Unauthorized without hitting the backend. This centralizes auth logic — services trust the gateway and don't need to validate tokens themselves. For authorization, the gateway can check if the user has permission to access the resource based on roles or policies."

### Q: What are the downsides of an API Gateway?

> "The gateway becomes a single point of failure — if it goes down, all services are unreachable. You mitigate this with multiple gateway instances behind a load balancer. It adds latency — every request goes through an extra hop. It can become a bottleneck if not scaled properly. And it can become complex if you add too much business logic to it — the gateway should stay thin and focused on cross-cutting concerns. There's also the operational overhead of managing another component."

---

## 9. Interview Tricks & Pitfalls

### ✅ Trick 1: Mention specific responsibilities

Don't just say "API Gateway routes requests." List the key responsibilities:

> "The API Gateway handles authentication, rate limiting, SSL termination, request routing, logging, and can cache responses. It's the single entry point for all client requests."

### ✅ Trick 2: Discuss the BFF pattern

Shows you understand different clients have different needs:

> "For a mobile app, I'd use a Backend for Frontend pattern — a separate gateway that returns smaller payloads and aggregates multiple service calls to reduce round trips over slow mobile networks."

### ✅ Trick 3: Address the SPOF concern

Whenever you propose an API Gateway, mention high availability:

> "The gateway is a single point of failure, so I'd run multiple instances behind a load balancer with health checks and auto-scaling. This ensures no single gateway failure takes down the entire API."

### ❌ Pitfall 1: Putting business logic in the gateway

The gateway should be thin:

> "I'd keep the gateway focused on cross-cutting concerns like auth and rate limiting. Business logic belongs in services, not the gateway. Otherwise, the gateway becomes a monolith."

### ❌ Pitfall 2: Not mentioning service discovery

In a dynamic environment, static configuration doesn't work:

> "The gateway integrates with a service registry like Consul. Services register themselves on startup, and the gateway queries the registry to find healthy instances. This enables auto-scaling without manual configuration."

### ❌ Pitfall 3: Forgetting about latency

The gateway adds a hop:

> "The gateway adds latency — typically 5-10ms. To minimize this, I'd keep gateway logic lightweight, use a fast implementation like Envoy or Nginx, and cache aggressively where appropriate."

---

## 10. Quick Reference

```
API Gateway = single entry point for all client requests

Core responsibilities:
  1. Request routing (path/host-based)
  2. Authentication & authorization
  3. Rate limiting
  4. SSL/TLS termination
  5. Request/Response transformation
  6. Load balancing
  7. Caching
  8. Logging & monitoring

Patterns:
  BFF (Backend for Frontend):  Different gateways per client type
  API Composition:              Aggregate multiple service calls
  Protocol Translation:         REST → gRPC, etc.

vs Load Balancer:
  Load Balancer: Distributes load, L4/L7, no app logic
  API Gateway:   Routes, auth, rate limiting, L7, app-aware

vs Reverse Proxy:
  Reverse Proxy: SSL termination, caching, basic routing
  API Gateway:   All of above + auth, rate limiting, service discovery

Technologies:
  AWS API Gateway:  Managed, serverless, AWS-native
  Kong:             Open-source, plugin ecosystem
  Nginx:            High performance, mature
  Envoy:            Modern, gRPC, service mesh
  Traefik:          Kubernetes-native, auto-discovery

Challenges:
  - Single point of failure (mitigate: multiple instances + LB)
  - Latency overhead (mitigate: keep thin, cache)
  - Complexity (mitigate: keep thin, BFF pattern)

Service discovery:
  Static:   Manual configuration (doesn't scale)
  Dynamic:  Service registry (Consul, Eureka)
            Services register/deregister automatically

Best practices:
  ✅ Keep gateway thin (cross-cutting concerns only)
  ✅ Run multiple instances (HA)
  ✅ Integrate with service discovery
  ✅ Monitor latency and error rates
  ❌ Don't put business logic in gateway
  ❌ Don't make gateway a monolith
```
