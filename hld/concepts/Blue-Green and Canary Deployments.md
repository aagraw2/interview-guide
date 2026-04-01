## 1. Why Deployment Strategies Matter

Deploying new code to production is risky. Bugs, performance issues, or incompatibilities can cause outages.

**Goal:** Deploy safely with minimal downtime and easy rollback.

```
Bad deployment:
  Deploy v2 to all servers at once
  → Bug in v2 → All users affected → Scramble to rollback

Good deployment:
  Deploy v2 gradually, monitor metrics, rollback if issues
  → Bug in v2 → Only small % of users affected → Quick rollback
```

---

## 2. Blue-Green Deployment

### Concept

Run two identical production environments: **Blue** (current) and **Green** (new). Switch traffic all at once.

```
┌─────────────────────────────────────────────────────────┐
│                    Load Balancer                        │
│                  (routes 100% to Blue)                  │
└──────────────┬──────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────┐    ┌──────────────────────────┐
│   Blue Environment (v1)  │    │  Green Environment (v2)  │
│  [v1] [v1] [v1] [v1]     │    │  [v2] [v2] [v2] [v2]     │
│  ← 100% traffic          │    │  ← 0% traffic (testing)  │
└──────────────────────────┘    └──────────────────────────┘

After testing Green:
  Switch load balancer to Green

┌─────────────────────────────────────────────────────────┐
│                    Load Balancer                        │
│                 (routes 100% to Green)                  │
└──────────────┬──────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────┐    ┌──────────────────────────┐
│   Blue Environment (v1)  │    │  Green Environment (v2)  │
│  [v1] [v1] [v1] [v1]     │    │  [v2] [v2] [v2] [v2]     │
│  ← 0% traffic (standby)  │    │  ← 100% traffic          │
└──────────────────────────┘    └──────────────────────────┘
```

---

### Process

```
1. Blue environment runs v1 (current production)
2. Deploy v2 to Green environment
3. Run smoke tests on Green (internal testing)
4. Switch load balancer to route 100% traffic to Green
5. Monitor metrics (error rate, latency)
6. If issues: switch back to Blue (instant rollback)
7. If stable: keep Green as production, Blue becomes standby
```

---

### Implementation

#### AWS Elastic Beanstalk

```bash
# Deploy to Green environment
eb deploy green-env

# Test Green
curl https://green-env.elasticbeanstalk.com/health

# Swap URLs (switch traffic)
eb swap green-env blue-env
```

---

#### Kubernetes

```yaml
# Blue deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:1.0

---
# Green deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:2.0

---
# Service (routes to Blue initially)
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue  # Switch to "green" to route to Green
  ports:
  - port: 80
    targetPort: 8080
```

**Switch traffic:**

```bash
# Update service selector to route to Green
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

# Rollback to Blue
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'
```

---

### Pros

```
✅ Instant rollback (switch back to Blue)
✅ Zero downtime (Green is fully tested before switch)
✅ Simple to understand and implement
✅ Full production environment for testing
```

---

### Cons

```
❌ Requires 2x infrastructure (expensive)
❌ Database migrations tricky (both environments share DB)
❌ All-or-nothing switch (if bug affects 1% of users, still affects everyone)
❌ Stateful services (sessions, caches) need special handling
```

---

## 3. Canary Deployment

### Concept

Deploy new version to a small subset of users first. Gradually increase traffic if no issues.

```
Step 1: Deploy v2 to 10% of servers
┌─────────────────────────────────────────────────────────┐
│                    Load Balancer                        │
│              (90% to v1, 10% to v2)                     │
└──────────────┬──────────────────────────────────────────┘
               │
    ┌──────────┴──────────┐
    ▼                     ▼
[v1] [v1] [v1]          [v2]
90% traffic             10% traffic (canary)

Monitor metrics for 10 minutes
If stable, increase to 50%

Step 2: 50% to v2
┌─────────────────────────────────────────────────────────┐
│                    Load Balancer                        │
│              (50% to v1, 50% to v2)                     │
└──────────────┬──────────────────────────────────────────┘
               │
    ┌──────────┴──────────┐
    ▼                     ▼
[v1] [v1]              [v2] [v2]
50% traffic            50% traffic

Monitor metrics for 10 minutes
If stable, increase to 100%

Step 3: 100% to v2
┌─────────────────────────────────────────────────────────┐
│                    Load Balancer                        │
│                  (100% to v2)                           │
└──────────────┬──────────────────────────────────────────┘
               │
               ▼
         [v2] [v2] [v2] [v2]
         100% traffic
```

---

### Process

```
1. Deploy v2 to 10% of servers (canary)
2. Route 10% of traffic to canary
3. Monitor metrics (error rate, latency, business metrics)
4. If metrics are good:
   - Increase to 25%, monitor
   - Increase to 50%, monitor
   - Increase to 100%
5. If metrics degrade:
   - Rollback (route 0% to canary)
   - Investigate and fix
```

---

### Implementation

#### Kubernetes with Istio

```yaml
# Deployment v1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v1
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        image: myapp:1.0

---
# Deployment v2 (canary)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
      - name: myapp
        image: myapp:2.0

---
# Istio VirtualService (traffic splitting)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: v1
      weight: 90  # 90% to v1
    - destination:
        host: myapp
        subset: v2
      weight: 10  # 10% to v2 (canary)

---
# DestinationRule (define subsets)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

**Increase canary traffic:**

```bash
# Update VirtualService to 50/50
kubectl patch virtualservice myapp --type merge -p '
{
  "spec": {
    "http": [{
      "route": [
        {"destination": {"host": "myapp", "subset": "v1"}, "weight": 50},
        {"destination": {"host": "myapp", "subset": "v2"}, "weight": 50}
      ]
    }]
  }
}'
```

---

#### AWS ALB (Application Load Balancer)

```bash
# Create target groups for v1 and v2
aws elbv2 create-target-group --name myapp-v1 ...
aws elbv2 create-target-group --name myapp-v2 ...

# Configure listener rule with weighted routing
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:... \
  --conditions Field=path-pattern,Values=/* \
  --actions Type=forward,ForwardConfig='{
    "TargetGroups": [
      {"TargetGroupArn": "arn:...:myapp-v1", "Weight": 90},
      {"TargetGroupArn": "arn:...:myapp-v2", "Weight": 10}
    ]
  }'
```

---

### Canary Analysis

Monitor metrics to decide whether to proceed or rollback.

```
Metrics to monitor:
  - Error rate (5xx, 4xx)
  - Latency (p50, p95, p99)
  - Request rate (RPS)
  - Business metrics (conversion rate, revenue)

Comparison:
  v1 error rate: 0.1%
  v2 error rate: 0.5%  ← 5x higher, rollback!

  v1 p99 latency: 500ms
  v2 p99 latency: 1200ms  ← 2.4x higher, rollback!
```

**Automated canary analysis:** Tools like Flagger (Kubernetes) automatically analyze metrics and rollback if thresholds are exceeded.

---

### Pros

```
✅ Low risk (catch issues with small user base)
✅ Gradual rollout (can stop at any %)
✅ Real production traffic (not synthetic tests)
✅ No need for 2x infrastructure (just a few canary instances)
```

---

### Cons

```
❌ More complex (requires traffic splitting)
❌ Slower rollout (gradual vs instant)
❌ Requires good monitoring (need metrics to decide)
❌ Stateful services tricky (sessions may hit different versions)
```

---

## 4. Blue-Green vs Canary

|Feature|Blue-Green|Canary|
|---|---|---|
|Rollout speed|Instant (all at once)|Gradual (10% → 50% → 100%)|
|Risk|Higher (all users affected if bug)|Lower (small % affected first)|
|Rollback|Instant (switch back)|Instant (route 0% to canary)|
|Infrastructure|2x (Blue + Green)|1x + small canary|
|Complexity|Simple|More complex (traffic splitting)|
|Monitoring|Less critical (can test before switch)|Critical (need metrics to decide)|
|Use case|Low-risk changes, need instant rollback|High-risk changes, want gradual validation|

---

## 5. Advanced Patterns

### Canary with User Segmentation

Route specific users to canary (e.g., internal employees, beta users).

```yaml
# Istio: Route beta users to v2
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - match:
    - headers:
        user-type:
          exact: beta
    route:
    - destination:
        host: myapp
        subset: v2
  - route:
    - destination:
        host: myapp
        subset: v1
```

---

### Canary with Geographic Routing

Route users in specific regions to canary.

```yaml
# Istio: Route US users to v2
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - match:
    - headers:
        x-country:
          exact: US
    route:
    - destination:
        host: myapp
        subset: v2
  - route:
    - destination:
        host: myapp
        subset: v1
```

---

### Automated Canary with Flagger

Flagger automates canary deployments on Kubernetes.

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
    - name: request-duration
      thresholdRange:
        max: 500
```

**How it works:**

```
1. Deploy new version
2. Flagger routes 10% traffic to canary
3. Monitors metrics for 1 minute
4. If success rate > 99% and latency < 500ms:
   - Increase to 20%, monitor
   - Increase to 30%, monitor
   - ...
   - Increase to 50%
5. If metrics degrade:
   - Rollback to 0%
```

---

## 6. Database Migrations

Both Blue-Green and Canary deployments are tricky with database changes.

### Backward-Compatible Migrations

Make schema changes backward-compatible so both old and new code work.

```
Bad (breaking change):
  v1 code expects column "name"
  v2 code expects column "full_name"
  → Deploy v2 → v1 code breaks

Good (backward-compatible):
  Step 1: Add "full_name" column (keep "name")
  Step 2: Deploy v2 (writes to both columns, reads from "full_name")
  Step 3: Backfill "full_name" from "name"
  Step 4: Deploy v3 (removes "name" column)
```

---

### Expand-Contract Pattern

```
Expand:   Add new schema (column, table) without removing old
Deploy:   Deploy new code that uses new schema
Contract: Remove old schema after all code is migrated
```

---

## 7. Common Interview Questions + Answers

### Q: What's the difference between blue-green and canary deployments?

> "Blue-green deployment runs two identical environments. You deploy the new version to the green environment, test it, then switch all traffic at once. It allows instant rollback by switching back to blue. Canary deployment gradually routes a small percentage of traffic to the new version, monitors metrics, and increases traffic if there are no issues. Blue-green is all-or-nothing and requires 2x infrastructure. Canary is gradual, lower risk, and only needs a few extra instances. Blue-green is simpler but higher risk. Canary is more complex but catches issues with a small user base first."

---

### Q: How do you handle database migrations with blue-green deployments?

> "Database migrations are tricky because both blue and green environments share the same database. The key is to make migrations backward-compatible. For example, if renaming a column, first add the new column while keeping the old one. Deploy the new code that uses the new column. Backfill data. Then remove the old column in a later deployment. This ensures both old and new code work during the transition. This is called the expand-contract pattern — expand the schema, deploy code, then contract by removing old schema."

---

### Q: When would you use canary over blue-green?

> "I'd use canary for high-risk changes where I want to validate with real production traffic before rolling out to everyone. For example, a major algorithm change, a new payment provider, or a performance optimization. Canary lets me catch issues with 10% of users before affecting everyone. I'd use blue-green for lower-risk changes where I want instant rollback capability, like a bug fix or a minor feature. Blue-green is simpler and faster to roll out, but canary is safer for risky changes."

---

### Q: How do you decide when to increase canary traffic?

> "I monitor key metrics like error rate, latency, and business metrics. I compare the canary version to the baseline version. If the canary's error rate is within 10% of baseline and latency is within 20%, I increase traffic. I also set thresholds — if error rate exceeds 1% or p99 latency exceeds 1 second, I rollback immediately. I use automated canary analysis tools like Flagger to make this decision automatically based on metrics. I also monitor for at least 5-10 minutes at each traffic level to catch issues that only appear under load."

---

## 8. Interview Tricks & Pitfalls

### ✅ Trick 1: Know the trade-offs

Blue-green is simpler but requires 2x infrastructure. Canary is safer but more complex. Be able to explain when to use each.

---

### ✅ Trick 2: Mention database migrations

This is a common follow-up question. Show you understand backward-compatible migrations.

---

### ✅ Trick 3: Talk about monitoring

Canary deployments require good monitoring. Mention specific metrics (error rate, latency) and automated analysis.

---

### ❌ Pitfall 1: Saying blue-green has zero risk

Blue-green still deploys to all users at once. If there's a bug, everyone is affected. Canary is lower risk.

---

### ❌ Pitfall 2: Ignoring stateful services

Sessions, caches, and databases make deployments tricky. Mention how you'd handle them (sticky sessions, backward-compatible migrations).

---

### ❌ Pitfall 3: Not mentioning rollback

Always mention rollback strategy. Both blue-green and canary allow instant rollback, but you need to monitor metrics to know when to rollback.

---

## 9. Quick Reference

```
Blue-Green Deployment:
  - Two identical environments (Blue = current, Green = new)
  - Deploy to Green, test, switch all traffic at once
  - Instant rollback (switch back to Blue)
  - Requires 2x infrastructure
  - Simple, but all-or-nothing

Canary Deployment:
  - Deploy to small % of servers (10%)
  - Gradually increase traffic (10% → 50% → 100%)
  - Monitor metrics at each step
  - Rollback if metrics degrade
  - Low risk (small % affected first)
  - More complex (requires traffic splitting)

Comparison:
  Blue-Green:  Instant rollout, higher risk, 2x infrastructure, simple
  Canary:      Gradual rollout, lower risk, 1x + canary, complex

Metrics to Monitor:
  - Error rate (5xx, 4xx)
  - Latency (p50, p95, p99)
  - Request rate (RPS)
  - Business metrics (conversion, revenue)

Database Migrations:
  - Backward-compatible changes (expand-contract pattern)
  - Add new schema, deploy code, remove old schema

Tools:
  Blue-Green:  AWS Elastic Beanstalk, Kubernetes (patch service selector)
  Canary:      Istio, AWS ALB, Flagger (automated)

Advanced Patterns:
  - Canary with user segmentation (beta users)
  - Canary with geographic routing (specific regions)
  - Automated canary analysis (Flagger)

When to Use:
  Blue-Green:  Low-risk changes, need instant rollback, simple setup
  Canary:      High-risk changes, want gradual validation, have good monitoring
```
