## 1. What are Containers?

**Containers** package an application and its dependencies into a single unit that runs consistently across environments.

```
Traditional deployment:
  App depends on specific OS, libraries, runtime versions
  "Works on my machine" → doesn't work in production

Container:
  App + dependencies + runtime → packaged together
  Runs identically on dev laptop, staging, production
```

**Key difference from VMs:**

```
Virtual Machine:
  Hardware → Hypervisor → Guest OS → App
  Each VM runs a full OS (GBs of memory, slow startup)

Container:
  Hardware → Host OS → Container Runtime → App
  Containers share the host OS kernel (MBs of memory, fast startup)
```

---

## 2. Docker Basics

**Docker** is the most popular container runtime.

### Dockerfile

Defines how to build a container image.

```dockerfile
# Base image
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy application code
COPY . .

# Expose port
EXPOSE 3000

# Start command
CMD ["node", "server.js"]
```

---

### Build and Run

```bash
# Build image
docker build -t myapp:1.0 .

# Run container
docker run -p 3000:3000 myapp:1.0

# Run in background
docker run -d -p 3000:3000 myapp:1.0

# View running containers
docker ps

# Stop container
docker stop <container_id>
```

---

### Image Layers

Docker images are built in layers. Each instruction in the Dockerfile creates a layer.

```
FROM node:18-alpine       → Layer 1 (base image)
WORKDIR /app              → Layer 2
COPY package*.json ./     → Layer 3
RUN npm install           → Layer 4 (dependencies)
COPY . .                  → Layer 5 (app code)
```

**Caching:** Layers are cached. If a layer hasn't changed, Docker reuses the cached layer. This speeds up builds.

**Best practice:** Put frequently changing files (app code) at the end. Put rarely changing files (dependencies) at the beginning.

---

## 3. Why Kubernetes?

Running containers on a single machine is easy. Running thousands of containers across hundreds of machines is hard.

**Kubernetes (K8s)** is a container orchestration platform that automates:

```
- Deployment (rolling updates, rollbacks)
- Scaling (horizontal pod autoscaler)
- Load balancing (services)
- Self-healing (restart failed containers)
- Service discovery (DNS)
- Configuration management (ConfigMaps, Secrets)
- Storage orchestration (persistent volumes)
```

---

## 4. Kubernetes Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Control Plane                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ API Server   │  │  Scheduler   │  │  Controller  │  │
│  │              │  │              │  │   Manager    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│  ┌──────────────┐                                       │
│  │    etcd      │  (distributed key-value store)        │
│  └──────────────┘                                       │
└─────────────────────────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
┌───────▼──────┐  ┌───────▼──────┐  ┌───────▼──────┐
│   Node 1     │  │   Node 2     │  │   Node 3     │
│              │  │              │  │              │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
│ │  kubelet │ │  │ │  kubelet │ │  │ │  kubelet │ │
│ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
│ │ kube-proxy│ │  │ │kube-proxy│ │  │ │kube-proxy│ │
│ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
│              │  │              │  │              │
│  Pods:       │  │  Pods:       │  │  Pods:       │
│  [App1][App2]│  │  [App3][App4]│  │  [App5][App6]│
└──────────────┘  └──────────────┘  └──────────────┘
```

**Control Plane:**

```
API Server:        Entry point for all operations (kubectl, CI/CD)
Scheduler:         Assigns pods to nodes based on resources
Controller Manager: Maintains desired state (restart failed pods, scale replicas)
etcd:              Stores cluster state (configuration, metadata)
```

**Worker Nodes:**

```
kubelet:      Agent that runs on each node, manages pods
kube-proxy:   Network proxy, handles service routing
Container Runtime: Docker, containerd, CRI-O
```

---

## 5. Kubernetes Core Concepts

### Pod

The smallest deployable unit. A pod contains one or more containers.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    ports:
    - containerPort: 3000
```

**Usually, one container per pod.** Multiple containers in a pod share network and storage (sidecar pattern).

---

### Deployment

Manages a set of identical pods. Handles rolling updates and rollbacks.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 3  # Run 3 pods
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

**Rolling update:**

```bash
# Update image
kubectl set image deployment/myapp-deployment myapp=myapp:2.0

# Kubernetes gradually replaces old pods with new ones
# Zero downtime
```

---

### Service

Exposes pods to the network. Provides stable IP and DNS name.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80        # Service port
    targetPort: 3000  # Container port
  type: LoadBalancer  # Expose externally
```

**Service types:**

```
ClusterIP:    Internal only (default)
NodePort:     Expose on each node's IP at a static port
LoadBalancer: Provision cloud load balancer (AWS ELB, GCP LB)
ExternalName: Map to external DNS name
```

---

### ConfigMap

Store configuration data (environment variables, config files).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  DATABASE_URL: "postgres://db:5432/mydb"
  LOG_LEVEL: "info"
```

**Use in pod:**

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    envFrom:
    - configMapRef:
        name: myapp-config
```

---

### Secret

Store sensitive data (passwords, API keys). Base64-encoded.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=  # base64("password123")
```

**Use in pod:**

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: myapp-secret
          key: DB_PASSWORD
```

**Note:** Secrets are base64-encoded, not encrypted. Use external secret managers (AWS Secrets Manager, HashiCorp Vault) for production.

---

### Namespace

Logical isolation within a cluster. Separate environments (dev, staging, prod).

```bash
# Create namespace
kubectl create namespace staging

# Deploy to namespace
kubectl apply -f deployment.yaml -n staging

# List pods in namespace
kubectl get pods -n staging
```

---

## 6. Health Checks

Kubernetes monitors pod health and restarts unhealthy pods.

### Liveness Probe

Checks if the container is alive. If it fails, Kubernetes restarts the container.

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 30
      periodSeconds: 10
```

**Use case:** Detect deadlocks, infinite loops. Restart the container to recover.

---

### Readiness Probe

Checks if the container is ready to receive traffic. If it fails, Kubernetes removes the pod from the service.

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 10
      periodSeconds: 5
```

**Use case:** During startup, the app may need time to warm up (load data, connect to DB). Don't send traffic until ready.

---

### Startup Probe

Checks if the container has started. If it fails, Kubernetes restarts the container.

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    startupProbe:
      httpGet:
        path: /startup
        port: 3000
      failureThreshold: 30
      periodSeconds: 10
```

**Use case:** Slow-starting containers (legacy apps). Gives more time before liveness probe kicks in.

---

## 7. Resource Management

### Requests and Limits

```yaml
resources:
  requests:
    memory: "128Mi"  # Minimum guaranteed
    cpu: "100m"      # 0.1 CPU core
  limits:
    memory: "256Mi"  # Maximum allowed
    cpu: "200m"      # 0.2 CPU core
```

**Requests:** Scheduler uses this to decide which node to place the pod on. Guaranteed resources.

**Limits:** If the pod exceeds limits, it's throttled (CPU) or killed (memory).

**Best practice:** Always set requests and limits. Prevents resource starvation and noisy neighbors.

---

### Horizontal Pod Autoscaler (HPA)

Automatically scale pods based on CPU, memory, or custom metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale when CPU > 70%
```

**How it works:**

```
1. HPA checks metrics every 15 seconds
2. If average CPU > 70%, increase replicas
3. If average CPU < 70%, decrease replicas (after cooldown)
```

---

## 8. Rolling Updates and Rollbacks

### Rolling Update

Gradually replace old pods with new ones. Zero downtime.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # Max 1 pod down at a time
      maxSurge: 1        # Max 1 extra pod during update
```

**Process:**

```
1. Create 1 new pod (v2)
2. Wait for new pod to be ready
3. Terminate 1 old pod (v1)
4. Repeat until all pods are v2
```

---

### Rollback

Revert to a previous version.

```bash
# View rollout history
kubectl rollout history deployment/myapp-deployment

# Rollback to previous version
kubectl rollout undo deployment/myapp-deployment

# Rollback to specific revision
kubectl rollout undo deployment/myapp-deployment --to-revision=2
```

---

## 9. Persistent Storage

Containers are ephemeral — data is lost when the container restarts. Use persistent volumes for stateful apps.

### PersistentVolume (PV)

Storage provisioned by admin (or dynamically).

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: myapp-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /mnt/data
```

---

### PersistentVolumeClaim (PVC)

Request for storage by a pod.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

---

### Use in Pod

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    volumeMounts:
    - name: myapp-storage
      mountPath: /data
  volumes:
  - name: myapp-storage
    persistentVolumeClaim:
      claimName: myapp-pvc
```

---

## 10. Common Interview Questions + Answers

### Q: What's the difference between a container and a VM?

> "Containers share the host OS kernel and isolate processes using namespaces and cgroups. VMs run a full guest OS on a hypervisor. Containers are lighter (MBs vs GBs), start faster (seconds vs minutes), and have less overhead. VMs provide stronger isolation because each VM has its own kernel. Containers are better for microservices and cloud-native apps. VMs are better when you need different OS versions or stronger security boundaries."

---

### Q: What's the difference between liveness and readiness probes?

> "Liveness probes check if the container is alive. If it fails, Kubernetes restarts the container. Use it to detect deadlocks or crashes. Readiness probes check if the container is ready to receive traffic. If it fails, Kubernetes removes the pod from the service but doesn't restart it. Use it during startup when the app needs time to warm up, or during temporary issues like database connection loss. Liveness is 'is it alive?' and readiness is 'is it ready to serve traffic?'"

---

### Q: How does Kubernetes handle rolling updates?

> "Kubernetes gradually replaces old pods with new ones. It creates a new pod with the updated image, waits for it to pass readiness checks, then terminates an old pod. This repeats until all pods are updated. The maxUnavailable and maxSurge parameters control how many pods can be down or extra during the update. This ensures zero downtime — there are always healthy pods serving traffic. If the update fails, you can rollback to the previous version with kubectl rollout undo."

---

### Q: How do you scale a Kubernetes deployment?

> "Manually, you can use kubectl scale deployment/myapp --replicas=5 to set the number of replicas. For automatic scaling, use a Horizontal Pod Autoscaler (HPA). The HPA monitors metrics like CPU or memory and adjusts replicas to maintain a target utilization. For example, if CPU > 70%, it increases replicas. If CPU < 70%, it decreases replicas after a cooldown period. You can also scale based on custom metrics like request rate or queue depth."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Know the difference between liveness and readiness

This is one of the most common Kubernetes interview questions. Be able to explain both clearly.

---

### ✅ Trick 2: Mention resource requests and limits

Always set requests and limits to prevent resource starvation. Shows you understand production best practices.

---

### ✅ Trick 3: Explain rolling updates

Know how Kubernetes achieves zero-downtime deployments. Mention maxUnavailable and maxSurge.

---

### ❌ Pitfall 1: Confusing pods and containers

A pod can contain multiple containers, but usually contains one. Don't say "deploy a container" — say "deploy a pod."

---

### ❌ Pitfall 2: Not setting resource limits

Without limits, a single pod can consume all node resources and starve other pods. Always set limits.

---

### ❌ Pitfall 3: Storing secrets in ConfigMaps

ConfigMaps are for non-sensitive data. Use Secrets for passwords and API keys. Better yet, use external secret managers.

---

## 12. Quick Reference

```
Containers: Package app + dependencies, run consistently across environments
Docker: Most popular container runtime

Kubernetes: Container orchestration platform
  - Automates deployment, scaling, load balancing, self-healing

Architecture:
  Control Plane: API Server, Scheduler, Controller Manager, etcd
  Worker Nodes:  kubelet, kube-proxy, container runtime

Core Concepts:
  Pod:        Smallest unit, contains 1+ containers
  Deployment: Manages replicas, rolling updates, rollbacks
  Service:    Exposes pods, stable IP/DNS, load balancing
  ConfigMap:  Configuration data (env vars, config files)
  Secret:     Sensitive data (passwords, API keys)
  Namespace:  Logical isolation (dev, staging, prod)

Health Checks:
  Liveness:  Is container alive? (restart if fails)
  Readiness: Is container ready for traffic? (remove from service if fails)
  Startup:   Has container started? (for slow-starting apps)

Resources:
  Requests: Minimum guaranteed (used by scheduler)
  Limits:   Maximum allowed (throttle/kill if exceeded)

Scaling:
  Manual: kubectl scale deployment/myapp --replicas=5
  Auto:   Horizontal Pod Autoscaler (HPA) based on CPU/memory/custom metrics

Rolling Updates:
  Gradually replace old pods with new ones
  maxUnavailable: Max pods down during update
  maxSurge:       Max extra pods during update
  Zero downtime

Rollback:
  kubectl rollout undo deployment/myapp

Storage:
  PersistentVolume (PV):      Storage provisioned by admin
  PersistentVolumeClaim (PVC): Request for storage by pod
  Mount PVC in pod for persistent data

Service Types:
  ClusterIP:    Internal only (default)
  NodePort:     Expose on node IP
  LoadBalancer: Provision cloud load balancer
```
