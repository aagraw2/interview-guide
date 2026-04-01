## 1. What is a Reverse Proxy?

A reverse proxy sits in front of backend servers and forwards client requests to them. From the client's perspective, they're talking to the reverse proxy — they don't know about the backend servers.

```
Client → Reverse Proxy → Backend Server(s)

Examples:
  - Nginx
  - HAProxy
  - Apache HTTP Server
  - Envoy
  - Traefik
```

**Key difference from forward proxy:** A forward proxy sits in front of clients (hides client identity). A reverse proxy sits in front of servers (hides server identity).

---

## 2. Forward Proxy vs Reverse Proxy

### Forward proxy

```
Client → Forward Proxy → Internet → Server

Client knows about the proxy
Server doesn't know the client's real IP

Use case:
  - Corporate firewall (control outbound traffic)
  - VPN (hide client IP)
  - Content filtering
```

### Reverse proxy

```
Client → Internet → Reverse Proxy → Backend Server

Client doesn't know about backend servers
Backend servers don't see client's real IP (sees proxy IP)

Use case:
  - Load balancing
  - SSL termination
  - Caching
  - Security (hide backend topology)
```

---

## 3. Core Functions

### 1. Load balancing

Distribute requests across multiple backend servers.

```
Client requests:
  Request 1 → Reverse Proxy → Server 1
  Request 2 → Reverse Proxy → Server 2
  Request 3 → Reverse Proxy → Server 3
  Request 4 → Reverse Proxy → Server 1 (round-robin)

Algorithms:
  - Round-robin
  - Least connections
  - IP hash (sticky sessions)
  - Weighted round-robin
```

### 2. SSL/TLS termination

Handle SSL encryption/decryption at the proxy, not backend servers.

```
Client → HTTPS → Reverse Proxy → HTTP → Backend Server
         (encrypted)              (plain)

Benefits:
  ✅ Offload CPU-intensive SSL from backend servers
  ✅ Centralized certificate management
  ✅ Backend servers can be simpler (no SSL config)

Security consideration:
  ❌ Traffic between proxy and backend is unencrypted
  → Use private network or mTLS for internal traffic
```

### 3. Caching

Cache responses to reduce backend load.

```
Request 1: Client → Proxy → Backend → Response (cache miss)
           Proxy caches response

Request 2: Client → Proxy → Cached response (cache hit)
           Backend not contacted

Cache headers:
  Cache-Control: public, max-age=3600
  ETag: "abc123"
  Last-Modified: Mon, 01 Jan 2024 00:00:00 GMT
```

### 4. Compression

Compress responses before sending to client.

```
Backend → Uncompressed response (1 MB) → Proxy
Proxy → Gzip compressed (200 KB) → Client

Nginx config:
  gzip on;
  gzip_types text/plain text/css application/json;
  gzip_min_length 1000;

Benefits:
  ✅ Reduce bandwidth
  ✅ Faster page load
  ❌ CPU cost for compression
```

### 5. Request routing

Route requests to different backends based on URL, headers, etc.

```
/api/*     → API servers (port 8080)
/static/*  → Static file servers (port 8081)
/admin/*   → Admin servers (port 8082)

Nginx config:
  location /api/ {
    proxy_pass http://api_servers;
  }
  location /static/ {
    proxy_pass http://static_servers;
  }
```

### 6. Security

```
- Hide backend server IPs and topology
- Rate limiting (block abusive clients)
- DDoS protection (absorb traffic spikes)
- WAF (Web Application Firewall) rules
- IP whitelisting/blacklisting
```

---

## 4. Nginx Configuration Example

### Basic reverse proxy

```nginx
server {
  listen 80;
  server_name example.com;

  location / {
    proxy_pass http://backend_server:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

### Load balancing

```nginx
upstream backend_servers {
  server 10.0.1.1:8080;
  server 10.0.1.2:8080;
  server 10.0.1.3:8080;
}

server {
  listen 80;
  server_name example.com;

  location / {
    proxy_pass http://backend_servers;
  }
}
```

### SSL termination

```nginx
server {
  listen 443 ssl;
  server_name example.com;

  ssl_certificate /etc/ssl/certs/example.com.crt;
  ssl_certificate_key /etc/ssl/private/example.com.key;

  location / {
    proxy_pass http://backend_server:8080;
  }
}
```

### Caching

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m;

server {
  listen 80;
  server_name example.com;

  location / {
    proxy_cache my_cache;
    proxy_cache_valid 200 60m;
    proxy_cache_valid 404 10m;
    proxy_pass http://backend_server:8080;
  }
}
```

---

## 5. Health Checks

Reverse proxies can monitor backend server health and stop routing to unhealthy servers.

### Active health checks

Proxy actively pings backend servers.

```
Nginx Plus (commercial):
  upstream backend_servers {
    server 10.0.1.1:8080;
    server 10.0.1.2:8080;
    server 10.0.1.3:8080;
  }

  health_check interval=5s fails=3 passes=2;

Every 5 seconds:
  - Ping each server
  - If 3 consecutive failures → mark unhealthy
  - If 2 consecutive successes → mark healthy
```

### Passive health checks

Proxy monitors actual request failures.

```
Nginx (open source):
  upstream backend_servers {
    server 10.0.1.1:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.2:8080 max_fails=3 fail_timeout=30s;
  }

If 3 requests fail within 30 seconds:
  → Mark server unhealthy for 30 seconds
  → Retry after 30 seconds
```

---

## 6. Connection Pooling

Reuse connections to backend servers instead of creating new ones for each request.

```
Without connection pooling:
  Request 1: Open connection → Send request → Close connection
  Request 2: Open connection → Send request → Close connection
  (Overhead: TCP handshake + TLS handshake for each request)

With connection pooling:
  Request 1: Open connection → Send request → Keep alive
  Request 2: Reuse connection → Send request → Keep alive
  (Overhead: Only first request pays handshake cost)
```

### Nginx keepalive

```nginx
upstream backend_servers {
  server 10.0.1.1:8080;
  server 10.0.1.2:8080;
  keepalive 32;  # Keep 32 idle connections per worker
}

server {
  location / {
    proxy_pass http://backend_servers;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
  }
}
```

---

## 7. Sticky Sessions (Session Affinity)

Route requests from the same client to the same backend server.

```
Without sticky sessions:
  User login → Server 1 (session stored in Server 1's memory)
  Next request → Server 2 (session not found, user logged out)

With sticky sessions:
  User login → Server 1 (session stored in Server 1's memory)
  Next request → Server 1 (session found, user still logged in)
```

### IP hash

```nginx
upstream backend_servers {
  ip_hash;
  server 10.0.1.1:8080;
  server 10.0.1.2:8080;
  server 10.0.1.3:8080;
}

hash(client_ip) % num_servers → always routes to same server
```

### Cookie-based

```nginx
upstream backend_servers {
  server 10.0.1.1:8080;
  server 10.0.1.2:8080;
  server 10.0.1.3:8080;
  sticky cookie srv_id expires=1h;
}

Proxy sets cookie: srv_id=server1
Next request with cookie → routes to server1
```

### Limitations

```
❌ Uneven load distribution (some users more active than others)
❌ Server failure breaks sessions
❌ Harder to scale (can't easily add/remove servers)

Better solution: Use external session store (Redis, database)
```

---

## 8. Rate Limiting

Limit requests per client to prevent abuse.

```nginx
http {
  limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

  server {
    location /api/ {
      limit_req zone=mylimit burst=20 nodelay;
      proxy_pass http://backend_servers;
    }
  }
}

Rate: 10 requests/second per IP
Burst: Allow up to 20 requests in a burst
Nodelay: Reject excess requests immediately (don't queue)
```

---

## 9. WebSocket Support

Reverse proxies must handle WebSocket upgrade requests.

```nginx
server {
  location /ws/ {
    proxy_pass http://backend_servers;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 86400;  # Keep connection open for 24 hours
  }
}

HTTP upgrade:
  Client: GET /ws/ HTTP/1.1
          Upgrade: websocket
          Connection: Upgrade

  Proxy forwards upgrade request to backend
  Backend: HTTP/1.1 101 Switching Protocols
           Upgrade: websocket
           Connection: Upgrade

  Connection upgraded to WebSocket
```

---

## 10. Reverse Proxy vs API Gateway

|Feature|Reverse Proxy|API Gateway|
|---|---|---|
|Primary purpose|Route requests, load balance|API management, orchestration|
|SSL termination|✅|✅|
|Load balancing|✅|✅|
|Caching|✅|✅|
|Rate limiting|✅|✅|
|Authentication|Basic|Advanced (OAuth, JWT, API keys)|
|Request transformation|Limited|✅ (modify headers, body)|
|Response aggregation|❌|✅ (combine multiple backend calls)|
|API versioning|Manual routing|✅ (built-in)|
|Analytics|Basic logs|✅ (detailed metrics, dashboards)|
|Developer portal|❌|✅|

**Use reverse proxy when:** You need basic routing, load balancing, and SSL termination.

**Use API gateway when:** You need advanced API management, authentication, rate limiting, and analytics.

---

## 11. Common Use Cases

### 1. Microservices routing

```
Client → Reverse Proxy
  /users/*    → User service
  /orders/*   → Order service
  /payments/* → Payment service

Single entry point for all services
```

### 2. Blue-green deployment

```
Blue environment: 10.0.1.1, 10.0.1.2 (current production)
Green environment: 10.0.2.1, 10.0.2.2 (new version)

Nginx config:
  upstream backend {
    server 10.0.1.1;  # Blue
    server 10.0.1.2;  # Blue
  }

Deploy to green:
  1. Deploy new version to green servers
  2. Test green environment
  3. Update Nginx config to point to green
  4. Reload Nginx (zero downtime)

Rollback:
  Update config back to blue, reload Nginx
```

### 3. Static asset serving

```
/api/*     → Backend servers (dynamic content)
/static/*  → Nginx serves directly from disk (static files)

Nginx is very efficient at serving static files
No need to hit backend servers for CSS, JS, images
```

### 4. Security layer

```
Internet → Reverse Proxy (DMZ) → Backend Servers (private network)

Reverse proxy:
  - Public IP address
  - Exposed to internet
  - Hardened security

Backend servers:
  - Private IP addresses
  - Not directly accessible from internet
  - Protected by reverse proxy
```

---

## 12. Common Interview Questions + Answers

### Q: What's the difference between a reverse proxy and a load balancer?

> "A reverse proxy is a broader concept that includes load balancing as one of its functions. A reverse proxy can do SSL termination, caching, request routing, compression, and security in addition to load balancing. A load balancer specifically focuses on distributing traffic across backend servers. In practice, tools like Nginx and HAProxy serve as both reverse proxies and load balancers."

### Q: Why would you use SSL termination at the reverse proxy?

> "SSL termination offloads the CPU-intensive encryption/decryption from backend servers, allowing them to focus on application logic. It also centralizes certificate management — you only need to install and renew certificates on the reverse proxy, not on every backend server. The trade-off is that traffic between the proxy and backend is unencrypted, so you should use a private network or mTLS for internal communication."

### Q: How do you handle sticky sessions with a reverse proxy?

> "You can use IP hash to route requests from the same IP to the same server, or cookie-based routing where the proxy sets a cookie identifying the backend server. However, sticky sessions have limitations — uneven load distribution and broken sessions on server failure. A better approach is to use an external session store like Redis so any backend server can handle any request."

### Q: What's the difference between a reverse proxy and an API gateway?

> "A reverse proxy handles basic routing, load balancing, SSL termination, and caching. An API gateway adds advanced features like authentication (OAuth, JWT), request transformation, response aggregation, API versioning, rate limiting per API key, and analytics. Use a reverse proxy for simple routing needs, and an API gateway when you need comprehensive API management."

---

## 13. Interview Tricks & Pitfalls

### ✅ Trick 1: Mention multiple functions

Don't just say "load balancing." Mention SSL termination, caching, compression, security — this shows you understand the full scope.

### ✅ Trick 2: Know the security benefits

Reverse proxies hide backend topology and provide a security layer. Mentioning this shows you think about security.

### ✅ Trick 3: Discuss connection pooling

Connection pooling is an important optimization. Mentioning it shows you understand performance considerations.

### ❌ Pitfall 1: Confusing forward and reverse proxy

Forward proxy sits in front of clients. Reverse proxy sits in front of servers. Don't mix them up.

### ❌ Pitfall 2: Thinking sticky sessions are always good

Sticky sessions have limitations (uneven load, broken sessions on failure). External session stores are better.

### ❌ Pitfall 3: Forgetting about WebSocket support

WebSockets require special handling (HTTP upgrade). Mentioning this shows you understand real-time communication.

---

## 14. Quick Reference

```
What is a reverse proxy?
  Sits in front of backend servers
  Forwards client requests to backends
  Client doesn't know about backend servers
  Examples: Nginx, HAProxy, Envoy

Forward vs Reverse:
  Forward proxy: Sits in front of clients (hides client)
  Reverse proxy: Sits in front of servers (hides servers)

Core functions:
  1. Load balancing (distribute requests)
  2. SSL termination (offload encryption)
  3. Caching (reduce backend load)
  4. Compression (reduce bandwidth)
  5. Request routing (route by URL/headers)
  6. Security (hide topology, rate limiting, WAF)

SSL termination:
  Client → HTTPS → Proxy → HTTP → Backend
  Benefits: Offload CPU, centralized certs
  Trade-off: Internal traffic unencrypted

Health checks:
  Active: Proxy pings backends
  Passive: Monitor actual request failures

Connection pooling:
  Reuse connections to backends
  Avoid TCP/TLS handshake overhead

Sticky sessions:
  Route same client to same server
  Methods: IP hash, cookie-based
  Limitation: Uneven load, broken on failure
  Better: External session store (Redis)

Rate limiting:
  Limit requests per client
  Prevent abuse and DDoS

WebSocket support:
  Handle HTTP upgrade requests
  Keep connection open (long timeout)

Reverse proxy vs API gateway:
  Reverse proxy: Basic routing, load balancing, SSL
  API gateway: Advanced API management, auth, analytics

Use cases:
  Microservices routing (single entry point)
  Blue-green deployment (switch backends)
  Static asset serving (Nginx serves directly)
  Security layer (DMZ, hide backend IPs)

Common tools:
  Nginx: High performance, widely used
  HAProxy: TCP/HTTP load balancing
  Envoy: Modern, service mesh integration
  Traefik: Dynamic config, container-native
```
