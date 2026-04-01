## 1. What is Service Discovery?

Service discovery is the process of automatically detecting and locating services in a distributed system. Instead of hardcoding service locations (IP addresses and ports), services dynamically register themselves and clients dynamically discover them.

```
Problem:
  Service A needs to call Service B
  Service B runs on 3 instances: 10.0.1.1, 10.0.1.2, 10.0.1.3
  How does Service A know which instances exist and which are healthy?

Solution: Service Discovery
  Service B instances register themselves
  Service A queries service registry to find healthy instances
```

**Examples:** Consul, Eureka, etcd, ZooKeeper, Kubernetes DNS

---

## 2. Why Service Discovery?

### Without service discovery (hardcoded IPs)

```
Service A config:
  service_b_url = "http://10.0.1.1:8080"

Problems:
  ❌ What if 10.0.1.1 goes down?
  ❌ What if we add more instances?
  ❌ What if IP addresses change?
  ❌ Need to redeploy Service A to update config
```

### With service discovery

```
Service A:
  instances = service_registry.discover("service-b")
  → Returns: [10.0.1.1:8080, 10.0.1.2:8080, 10.0.1.3:8080]
  Pick one instance (round-robin, random, etc.)

Benefits:
  ✅ Automatic detection of new instances
  ✅ Automatic removal of unhealthy instances
  ✅ No config changes needed
  ✅ Dynamic scaling
```

---

## 3. Service Discovery Patterns

### Client-side discovery

Client queries the service registry and chooses which instance to call.

```
1. Service B instances register with registry
   10.0.1.1:8080 → "service-b"
   10.0.1.2:8080 → "service-b"
   10.0.1.3:8080 → "service-b"

2. Service A queries registry
   GET /services/service-b
   → Returns: [10.0.1.1:8080, 10.0.1.2:8080, 10.0.1.3:8080]

3. Service A picks an instance (client-side load balancing)
   → Calls 10.0.1.2:8080 directly

Examples: Netflix Eureka, Consul (with client library)
```

**Pros:**
- Client has full control over load balancing
- No extra network hop
- Can implement custom routing logic

**Cons:**
- Client must implement discovery logic
- Client must handle failures and retries
- Couples client to service registry

### Server-side discovery

Client calls a load balancer, which queries the service registry and routes the request.

```
1. Service B instances register with registry
   10.0.1.1:8080 → "service-b"
   10.0.1.2:8080 → "service-b"
   10.0.1.3:8080 → "service-b"

2. Service A calls load balancer
   GET http://load-balancer/service-b

3. Load balancer queries registry
   → Returns: [10.0.1.1:8080, 10.0.1.2:8080, 10.0.1.3:8080]

4. Load balancer picks an instance and forwards request
   → Calls 10.0.1.2:8080

Examples: Kubernetes Service, AWS ALB + ECS, Consul Connect
```

**Pros:**
- Client is simple (just calls load balancer)
- Centralized load balancing logic
- Decouples client from service registry

**Cons:**
- Extra network hop (load balancer)
- Load balancer is a potential bottleneck
- Less flexibility in routing logic

---

## 4. Service Registry

A database that stores service locations and health status.

### Registration

Services register themselves on startup.

```
POST /services
{
  "name": "service-b",
  "id": "service-b-1",
  "address": "10.0.1.1",
  "port": 8080,
  "health_check": "http://10.0.1.1:8080/health"
}

Registry stores:
  service-b → [
    {id: "service-b-1", address: "10.0.1.1", port: 8080, status: "healthy"},
    {id: "service-b-2", address: "10.0.1.2", port: 8080, status: "healthy"},
    {id: "service-b-3", address: "10.0.1.3", port: 8080, status: "healthy"}
  ]
```

### Deregistration

Services deregister on shutdown.

```
DELETE /services/service-b/service-b-1

Registry removes instance from list
```

### Health checks

Registry periodically checks if services are healthy.

```
Every 10 seconds:
  GET http://10.0.1.1:8080/health
  → 200 OK → mark as healthy
  → Timeout or 5xx → mark as unhealthy

Unhealthy instances are not returned in discovery queries
```

### Heartbeats

Services send periodic heartbeats to prove they're alive.

```
Every 30 seconds:
  PUT /services/service-b/service-b-1/heartbeat

If no heartbeat for 60 seconds:
  → Mark instance as unhealthy
  → Remove from discovery results

This is called TTL (Time To Live) registration
```

---

## 5. Consul

Distributed service mesh with service discovery, health checking, and KV store.

### Architecture

```
Consul Cluster:
  - Consul servers (3-5 nodes, Raft consensus)
  - Consul agents (on every service node)

Service registration:
  Service → Consul agent → Consul server cluster
```

### Service registration

```json
{
  "service": {
    "name": "service-b",
    "id": "service-b-1",
    "address": "10.0.1.1",
    "port": 8080,
    "check": {
      "http": "http://10.0.1.1:8080/health",
      "interval": "10s",
      "timeout": "1s"
    }
  }
}

Register via HTTP API or config file
```

### Service discovery

```bash
# DNS query
dig @consul service-b.service.consul
→ Returns: 10.0.1.1, 10.0.1.2, 10.0.1.3

# HTTP API
curl http://consul:8500/v1/catalog/service/service-b
→ Returns: [
  {"Address": "10.0.1.1", "ServicePort": 8080},
  {"Address": "10.0.1.2", "ServicePort": 8080},
  {"Address": "10.0.1.3", "ServicePort": 8080}
]
```

### Health checks

```
Consul agent runs health checks:
  - HTTP: GET /health (expect 200)
  - TCP: Connect to port (expect success)
  - Script: Run command (expect exit code 0)
  - TTL: Service must send heartbeat

Unhealthy services are marked as "critical" and excluded from DNS/API results
```

---

## 6. Netflix Eureka

Client-side service discovery used in Spring Cloud.

### Architecture

```
Eureka Server:
  - Service registry
  - Stores service instances

Eureka Client:
  - Embedded in each service
  - Registers with Eureka Server
  - Fetches registry and caches locally
```

### Service registration

```java
@SpringBootApplication
@EnableEurekaClient
public class ServiceBApplication {
  public static void main(String[] args) {
    SpringApplication.run(ServiceBApplication.class, args);
  }
}

application.yml:
  eureka:
    client:
      service-url:
        defaultZone: http://eureka-server:8761/eureka/
    instance:
      hostname: service-b-1
```

### Service discovery

```java
@Autowired
private DiscoveryClient discoveryClient;

List<ServiceInstance> instances = discoveryClient.getInstances("service-b");
ServiceInstance instance = instances.get(0);
String url = "http://" + instance.getHost() + ":" + instance.getPort();

// Or use Ribbon for client-side load balancing
@LoadBalanced
@Bean
public RestTemplate restTemplate() {
  return new RestTemplate();
}

// Ribbon automatically discovers and load balances
restTemplate.getForObject("http://service-b/api/data", String.class);
```

### Heartbeats

```
Eureka client sends heartbeat every 30 seconds
If no heartbeat for 90 seconds → instance is evicted

Self-preservation mode:
  If >15% of instances miss heartbeats → assume network partition
  → Don't evict instances (better to serve stale data than no data)
```

---

## 7. Kubernetes Service Discovery

Kubernetes has built-in service discovery via DNS and environment variables.

### Kubernetes Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-b
spec:
  selector:
    app: service-b
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

### DNS-based discovery

```
Service name: service-b
Namespace: default

DNS name: service-b.default.svc.cluster.local

From any pod:
  curl http://service-b
  → Kubernetes DNS resolves to service IP
  → Service load balances to backend pods
```

### Environment variables

```
Kubernetes injects env vars for each service:

SERVICE_B_SERVICE_HOST=10.96.0.10
SERVICE_B_SERVICE_PORT=80

Application can read these to discover services
```

### How it works

```
1. Pods register with Kubernetes API server
   Pod: service-b-1 (10.244.1.5:8080)
   Pod: service-b-2 (10.244.1.6:8080)

2. Service selects pods by label
   selector: app=service-b

3. Kubernetes creates endpoints
   service-b → [10.244.1.5:8080, 10.244.1.6:8080]

4. kube-proxy updates iptables rules
   Traffic to service IP → load balanced to pod IPs

5. CoreDNS resolves service name to service IP
   service-b → 10.96.0.10 (service IP)
```

---

## 8. etcd and ZooKeeper

Distributed key-value stores used for service discovery and coordination.

### etcd (used by Kubernetes)

```
Service registration:
  PUT /services/service-b/instance-1
  {
    "address": "10.0.1.1",
    "port": 8080
  }

Service discovery:
  GET /services/service-b
  → Returns all keys under /services/service-b

Watch for changes:
  WATCH /services/service-b
  → Notified when instances are added/removed
```

### ZooKeeper

```
Service registration:
  CREATE /services/service-b/instance-1 (ephemeral node)
  Data: {"address": "10.0.1.1", "port": 8080}

Ephemeral nodes:
  - Automatically deleted when client disconnects
  - Perfect for service registration (auto-cleanup on crash)

Service discovery:
  GET /services/service-b (list children)
  → Returns: [instance-1, instance-2, instance-3]

Watch for changes:
  WATCH /services/service-b
  → Notified when children are added/removed
```

---

## 9. Health Checks

### Active health checks

Service registry actively probes services.

```
Consul:
  Every 10 seconds:
    GET http://10.0.1.1:8080/health
    → 200 OK → healthy
    → Timeout or 5xx → unhealthy

After 3 consecutive failures:
  → Mark as critical
  → Exclude from discovery results
```

### Passive health checks

Monitor actual request failures.

```
Load balancer:
  If 3 consecutive requests to 10.0.1.1 fail:
    → Mark as unhealthy for 30 seconds
    → Stop routing traffic
    → Retry after 30 seconds
```

### Health check endpoint

```
GET /health

Response:
  200 OK → healthy
  {
    "status": "UP",
    "checks": {
      "database": "UP",
      "redis": "UP",
      "disk_space": "UP"
    }
  }

503 Service Unavailable → unhealthy
  {
    "status": "DOWN",
    "checks": {
      "database": "DOWN"
    }
  }
```

---

## 10. Service Mesh

Modern approach to service discovery and communication.

### What is a service mesh?

```
Sidecar proxy (Envoy) deployed alongside each service
All traffic goes through the sidecar

Service A → Sidecar A → Sidecar B → Service B

Sidecar handles:
  - Service discovery
  - Load balancing
  - Retries and timeouts
  - Circuit breaking
  - Metrics and tracing
  - mTLS encryption
```

### Istio (service mesh on Kubernetes)

```
Control plane:
  - Pilot: Service discovery and traffic management
  - Citadel: Certificate management (mTLS)
  - Galley: Configuration management

Data plane:
  - Envoy sidecar proxies

Service discovery:
  1. Pilot watches Kubernetes API for service changes
  2. Pilot pushes updated service endpoints to Envoy sidecars
  3. Envoy load balances requests to healthy endpoints
```

---

## 11. Common Interview Questions + Answers

### Q: What's the difference between client-side and server-side service discovery?

> "Client-side discovery means the client queries the service registry directly and chooses which instance to call. This gives the client full control over load balancing but requires the client to implement discovery logic. Server-side discovery means the client calls a load balancer, which queries the registry and routes the request. This simplifies the client but adds an extra network hop. Kubernetes uses server-side discovery, while Netflix Eureka uses client-side."

### Q: How do you handle service instances that crash without deregistering?

> "Use heartbeats or TTL-based registration. Services send periodic heartbeats to the registry. If the registry doesn't receive a heartbeat within the TTL period, it automatically removes the instance. Alternatively, use health checks where the registry actively probes services and removes unhealthy ones. Ephemeral nodes in ZooKeeper automatically handle this — they're deleted when the client connection is lost."

### Q: What's the difference between Consul and Eureka?

> "Consul is a distributed service mesh with strong consistency (Raft consensus), health checks, KV store, and DNS interface. It supports both client-side and server-side discovery. Eureka is a client-side discovery system designed for high availability over consistency (AP in CAP theorem). It has self-preservation mode to handle network partitions. Consul is more feature-rich but more complex; Eureka is simpler and integrates well with Spring Cloud."

### Q: How does Kubernetes service discovery work?

> "Kubernetes uses DNS-based service discovery. When you create a Service, Kubernetes assigns it a cluster IP and creates a DNS record. Pods can resolve the service name to the cluster IP via CoreDNS. The Service load balances traffic to backend pods selected by labels. kube-proxy updates iptables rules to route traffic from the service IP to pod IPs. This is server-side discovery — clients just call the service name, and Kubernetes handles the rest."

---

## 12. Interview Tricks & Pitfalls

### ✅ Trick 1: Know both patterns

Client-side vs server-side discovery is a common interview question. Know the trade-offs and examples of each.

### ✅ Trick 2: Mention health checks

Health checks are critical for service discovery. Always mention how unhealthy instances are detected and removed.

### ✅ Trick 3: Discuss service mesh

Service mesh is the modern approach. Mentioning Istio or Envoy shows you're up-to-date with current practices.

### ❌ Pitfall 1: Forgetting about deregistration

Services must deregister on shutdown or use heartbeats/TTL. Otherwise, the registry will return dead instances.

### ❌ Pitfall 2: Thinking service discovery is just DNS

DNS is one mechanism, but service discovery also includes health checks, load balancing, and dynamic updates. Don't oversimplify.

### ❌ Pitfall 3: Ignoring consistency vs availability

Consul prioritizes consistency (CP), Eureka prioritizes availability (AP). Know the CAP theorem trade-offs.

---

## 13. Quick Reference

```
What is service discovery?
  Automatically detect and locate services in distributed systems
  Services register themselves, clients discover them dynamically
  No hardcoded IPs or ports

Why service discovery?
  ✅ Automatic detection of new instances
  ✅ Automatic removal of unhealthy instances
  ✅ No config changes needed
  ✅ Dynamic scaling

Client-side discovery:
  Client queries registry, picks instance, calls directly
  Examples: Netflix Eureka, Consul (with client library)
  Pros: Full control, no extra hop
  Cons: Client complexity, coupled to registry

Server-side discovery:
  Client calls load balancer, which queries registry and routes
  Examples: Kubernetes Service, AWS ALB, Consul Connect
  Pros: Simple client, centralized logic
  Cons: Extra hop, potential bottleneck

Service registry:
  Database storing service locations and health status
  Registration: Services register on startup
  Deregistration: Services deregister on shutdown
  Health checks: Registry probes services
  Heartbeats: Services send periodic heartbeats (TTL)

Consul:
  Distributed service mesh
  Strong consistency (Raft)
  Health checks, KV store, DNS interface
  Both client-side and server-side discovery

Netflix Eureka:
  Client-side discovery
  High availability (AP in CAP)
  Self-preservation mode (network partition tolerance)
  Spring Cloud integration

Kubernetes:
  DNS-based service discovery
  Service → DNS name → cluster IP → pod IPs
  kube-proxy updates iptables rules
  Server-side discovery (built-in)

etcd / ZooKeeper:
  Distributed KV stores for coordination
  Ephemeral nodes (auto-cleanup on disconnect)
  Watch for changes (notifications)

Health checks:
  Active: Registry probes services
  Passive: Monitor actual request failures
  Endpoint: GET /health → 200 OK (healthy) or 503 (unhealthy)

Service mesh:
  Sidecar proxy (Envoy) alongside each service
  Handles discovery, load balancing, retries, mTLS
  Examples: Istio, Linkerd, Consul Connect

Consul vs Eureka:
  Consul: CP (consistency), feature-rich, complex
  Eureka: AP (availability), simple, Spring Cloud

Kubernetes discovery:
  DNS: service-b.default.svc.cluster.local
  Env vars: SERVICE_B_SERVICE_HOST, SERVICE_B_SERVICE_PORT
  Server-side: Client calls service name, K8s routes
```
