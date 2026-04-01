## 1. What is Auto-Scaling?

Auto-scaling automatically adjusts the number of compute resources (servers, containers, instances) based on demand. When load increases, add more resources. When load decreases, remove resources.

```
Low traffic (100 RPS):
  → 2 servers

High traffic (10,000 RPS):
  → 20 servers (auto-scaled up)

Traffic drops (500 RPS):
  → 5 servers (auto-scaled down)
```

**Benefits:** Cost optimization, handle traffic spikes, maintain performance, no manual intervention.

---

## 2. Scaling Dimensions

### Horizontal scaling (scale out/in)

Add or remove instances.

```
Scale out: 2 servers → 4 servers → 8 servers
Scale in: 8 servers → 4 servers → 2 servers

Pros:
  ✅ No downtime (add/remove instances gradually)
  ✅ Unlimited scaling (add more instances)
  ✅ Fault tolerance (multiple instances)

Cons:
  ❌ Requires stateless design
  ❌ Load balancer needed
  ❌ More complex (distributed system)
```

### Vertical scaling (scale up/down)

Increase or decrease instance size.

```
Scale up: t2.small → t2.medium → t2.large
Scale down: t2.large → t2.medium → t2.small

Pros:
  ✅ Simple (no distributed system complexity)
  ✅ Works with stateful applications

Cons:
  ❌ Downtime (must restart instance)
  ❌ Limited scaling (hardware limits)
  ❌ Single point of failure
```

**Auto-scaling typically refers to horizontal scaling.**

---

## 3. Scaling Metrics

### CPU utilization

Most common metric for auto-scaling.

```
Target: 70% CPU utilization

Current: 85% CPU → Scale out (add instances)
Current: 40% CPU → Scale in (remove instances)

Pros:
  ✅ Simple to understand
  ✅ Available on all platforms

Cons:
  ❌ Doesn't reflect actual load (I/O-bound apps)
  ❌ Lags behind actual demand
```

### Request rate (RPS)

Scale based on requests per second.

```
Target: 1000 RPS per instance

Current: 5000 RPS, 3 instances → 1667 RPS/instance → Scale out
Current: 2000 RPS, 5 instances → 400 RPS/instance → Scale in

Pros:
  ✅ Directly reflects load
  ✅ Predictable capacity planning

Cons:
  ❌ Requires application-level metrics
  ❌ Doesn't account for request complexity
```

### Queue depth

Scale based on message queue backlog.

```
Target: 100 messages per worker

Current: 1000 messages, 5 workers → 200 msg/worker → Scale out
Current: 200 messages, 10 workers → 20 msg/worker → Scale in

Pros:
  ✅ Good for async workloads
  ✅ Prevents queue buildup

Cons:
  ❌ Only works for queue-based systems
```

### Custom metrics

Application-specific metrics.

```
Examples:
  - Active connections (WebSocket servers)
  - Database connections
  - Memory usage
  - Response time (p95 latency)
  - Business metrics (orders/sec, signups/sec)

Pros:
  ✅ Tailored to application needs
  ✅ More accurate scaling decisions

Cons:
  ❌ Requires custom instrumentation
  ❌ More complex to configure
```

---

## 4. Scaling Policies

### Target tracking

Maintain a target value for a metric.

```
Target: 70% CPU utilization

Current CPU: 85%
  → Scale out to bring CPU down to 70%

Current CPU: 40%
  → Scale in to bring CPU up to 70%

AWS Auto Scaling:
  TargetValue: 70
  PredefinedMetricType: ASGAverageCPUUtilization
```

### Step scaling

Scale by different amounts based on metric thresholds.

```
CPU < 40%: Remove 1 instance
CPU 40-60%: No change
CPU 60-80%: Add 1 instance
CPU 80-90%: Add 2 instances
CPU > 90%: Add 3 instances

Allows aggressive scaling during high load
```

### Scheduled scaling

Scale based on time of day or day of week.

```
Monday-Friday 9am-5pm: 10 instances (business hours)
Monday-Friday 5pm-9am: 3 instances (off-hours)
Saturday-Sunday: 2 instances (weekend)

Good for predictable traffic patterns
```

### Predictive scaling

Use machine learning to predict future load and scale proactively.

```
Historical data:
  Every Monday 9am: Traffic increases 5x
  Every Friday 5pm: Traffic decreases 3x

Predictive scaling:
  Monday 8:45am: Pre-scale to 10 instances (before traffic spike)
  Friday 4:45pm: Pre-scale down to 3 instances (before traffic drop)

AWS Auto Scaling Predictive Scaling
```

---

## 5. Scaling Constraints

### Minimum and maximum instances

```
Min: 2 instances (always running for availability)
Max: 20 instances (cost limit or quota)

Desired: Calculated by auto-scaling policy
  → Clamped to [min, max]
```

### Cooldown period

Prevent rapid scaling (thrashing).

```
Scale out cooldown: 300 seconds (5 minutes)
  → After scaling out, wait 5 minutes before scaling out again
  → Allows new instances to start and stabilize

Scale in cooldown: 600 seconds (10 minutes)
  → After scaling in, wait 10 minutes before scaling in again
  → Prevents premature scale-in

Without cooldown:
  Scale out → Load drops → Scale in → Load spikes → Scale out → ...
  (Thrashing: constant scaling up and down)
```

### Warm-up time

Time for new instances to become fully operational.

```
Instance launches:
  1. Boot OS (30 seconds)
  2. Start application (60 seconds)
  3. Load data/cache (120 seconds)
  4. Health check passes (30 seconds)
  Total: 240 seconds (4 minutes)

Auto-scaling should account for warm-up time:
  - Don't scale in immediately after scaling out
  - Use warm pool (pre-warmed instances)
```

---

## 6. Cold Start Problem

### Problem

```
Traffic spike:
  100 RPS → 10,000 RPS (100x increase)

Auto-scaling:
  2 instances → 20 instances (need 18 new instances)

Cold start:
  Each instance takes 4 minutes to warm up
  → 4 minutes of degraded performance
  → Some requests fail or timeout
```

### Solutions

#### 1. Warm pool

Pre-launch instances in a stopped state.

```
Warm pool: 10 stopped instances (pre-configured)

Scale out:
  1. Start instance from warm pool (30 seconds)
  2. Instance is ready much faster than cold start

AWS Auto Scaling Warm Pools
```

#### 2. Over-provisioning

Run more instances than needed.

```
Target: 70% CPU utilization
Over-provision: 50% CPU utilization

Trade-off:
  ✅ Faster response to spikes (more headroom)
  ❌ Higher cost (more idle capacity)
```

#### 3. Predictive scaling

Scale proactively before traffic spike.

```
Historical data: Traffic spikes at 9am every day

Predictive scaling:
  8:45am: Pre-scale to 10 instances
  9:00am: Traffic spike arrives, instances already ready
```

---

## 7. Kubernetes Horizontal Pod Autoscaler (HPA)

### How it works

```
1. Metrics server collects pod metrics (CPU, memory)

2. HPA controller queries metrics every 15 seconds

3. Calculate desired replicas:
   desired = current × (current_metric / target_metric)

4. Scale deployment up or down

Example:
  Current: 3 pods, 80% CPU
  Target: 50% CPU
  Desired: 3 × (80 / 50) = 4.8 → 5 pods
```

### Configuration

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
```

### Custom metrics

```
Scale based on application metrics:
  - Prometheus metrics
  - CloudWatch metrics
  - Datadog metrics

Example: Scale based on queue depth
  - Metric: rabbitmq_queue_messages
  - Target: 100 messages per pod
  - If queue has 500 messages and 3 pods → scale to 5 pods
```

---

## 8. AWS Auto Scaling

### Auto Scaling Group (ASG)

```
Configuration:
  - Launch template (AMI, instance type, security groups)
  - Min: 2, Max: 20, Desired: 5
  - Availability zones: us-east-1a, us-east-1b, us-east-1c
  - Load balancer: my-alb
  - Health check: ELB (load balancer health check)

Scaling policy:
  - Target tracking: 70% CPU utilization
  - Scale out cooldown: 300 seconds
  - Scale in cooldown: 600 seconds
```

### Launch template

```
AMI: ami-12345678 (pre-baked image with application)
Instance type: t3.medium
User data: (startup script)
  #!/bin/bash
  systemctl start my-app
  
Security groups: sg-12345678
IAM role: my-app-role
```

### Health checks

```
EC2 health check:
  - Instance is running
  - System status checks pass

ELB health check:
  - HTTP GET /health → 200 OK
  - If unhealthy, terminate and replace instance

Grace period: 300 seconds
  - Don't terminate instance during warm-up
```

---

## 9. Stateless Design for Auto-Scaling

Auto-scaling requires stateless applications.

### Stateless vs stateful

```
Stateless:
  - No local state (session, cache, files)
  - Any instance can handle any request
  - Easy to scale (add/remove instances)

Stateful:
  - Local state (session in memory, files on disk)
  - Requests must go to same instance (sticky sessions)
  - Hard to scale (can't easily remove instances)
```

### Externalize state

```
Session storage:
  ❌ In-memory (local to instance)
  ✅ Redis, DynamoDB (shared across instances)

File storage:
  ❌ Local disk
  ✅ S3, EFS (shared across instances)

Cache:
  ❌ In-memory (local to instance)
  ✅ Redis, Memcached (shared across instances)
```

---

## 10. Common Interview Questions + Answers

### Q: What metrics would you use for auto-scaling a web application?

> "I'd use CPU utilization as the primary metric with a target of 70%. This is simple and works well for most web apps. I'd also monitor request rate (RPS) and p95 latency as secondary metrics. If latency increases above a threshold, scale out even if CPU is below target. For async workloads like background jobs, I'd use queue depth — scale based on messages per worker. The key is to choose metrics that reflect actual user experience, not just resource utilization."

### Q: How do you handle the cold start problem in auto-scaling?

> "Cold start is when new instances take time to warm up, causing degraded performance during traffic spikes. Solutions include: using a warm pool of pre-configured stopped instances that can start quickly, over-provisioning by targeting lower CPU utilization to have more headroom, and predictive scaling to pre-scale before expected traffic spikes. For Kubernetes, you can use cluster autoscaler with node pools to pre-provision nodes. The goal is to reduce the time from scale decision to instance ready."

### Q: What's the difference between scale-out cooldown and scale-in cooldown?

> "Cooldown periods prevent rapid scaling (thrashing). Scale-out cooldown is the time to wait after scaling out before scaling out again — typically 5 minutes. This allows new instances to start, warm up, and begin handling traffic before deciding if more instances are needed. Scale-in cooldown is longer, typically 10 minutes, because you want to be conservative about removing capacity. Scaling in too quickly can cause performance degradation if traffic increases again. The asymmetry reflects that it's safer to have extra capacity than too little."

### Q: Why is stateless design important for auto-scaling?

> "Stateless design means any instance can handle any request, with no local state like sessions or files. This is critical for auto-scaling because you need to add and remove instances dynamically. With stateful design, you'd need sticky sessions to route users to the same instance, which complicates load balancing and prevents you from removing instances that have active sessions. Stateless design externalizes state to shared stores like Redis for sessions and S3 for files, allowing instances to be truly interchangeable."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Know multiple metrics

Don't just say "CPU utilization." Mention RPS, queue depth, latency, and custom metrics. This shows breadth.

### ✅ Trick 2: Discuss cooldown periods

Cooldown periods prevent thrashing. Always mention them when discussing auto-scaling policies.

### ✅ Trick 3: Mention stateless design

Auto-scaling requires stateless applications. Mentioning this shows you understand the architectural requirements.

### ❌ Pitfall 1: Forgetting about warm-up time

New instances take time to become operational. Don't assume they're immediately ready.

### ❌ Pitfall 2: Thinking auto-scaling is instant

Auto-scaling has delays: metric collection, scaling decision, instance launch, warm-up. It's not instant.

### ❌ Pitfall 3: Ignoring cost

Auto-scaling can increase costs significantly. Mention cost optimization strategies like scheduled scaling and right-sizing instances.

---

## 12. Quick Reference

```
What is auto-scaling?
  Automatically adjust compute resources based on demand
  Benefits: Cost optimization, handle spikes, maintain performance

Horizontal vs Vertical:
  Horizontal: Add/remove instances (scale out/in)
  Vertical: Increase/decrease instance size (scale up/down)
  Auto-scaling typically means horizontal

Scaling metrics:
  CPU utilization: 70% target (most common)
  Request rate: RPS per instance
  Queue depth: Messages per worker
  Custom: Latency, connections, business metrics

Scaling policies:
  Target tracking: Maintain target metric value
  Step scaling: Different amounts based on thresholds
  Scheduled: Time-based (business hours vs off-hours)
  Predictive: ML-based proactive scaling

Constraints:
  Min/max instances: [2, 20]
  Cooldown: Wait time between scaling actions (prevent thrashing)
  Warm-up: Time for new instances to become operational

Cold start problem:
  New instances take time to warm up
  Solutions: Warm pool, over-provisioning, predictive scaling

Kubernetes HPA:
  Horizontal Pod Autoscaler
  Metrics: CPU, memory, custom (Prometheus, etc.)
  Formula: desired = current × (current_metric / target_metric)

AWS Auto Scaling:
  Auto Scaling Group (ASG)
  Launch template (AMI, instance type)
  Health checks: EC2, ELB
  Integration: ALB, CloudWatch

Stateless design:
  Required for auto-scaling
  Externalize state: Redis (sessions), S3 (files)
  Any instance can handle any request

Cooldown periods:
  Scale-out: 5 minutes (allow instances to warm up)
  Scale-in: 10 minutes (conservative about removing capacity)
  Prevents thrashing (rapid scaling up and down)

Best practices:
  - Use multiple metrics (CPU + RPS + latency)
  - Set appropriate min/max limits
  - Use warm pools for faster scaling
  - Monitor costs (auto-scaling can be expensive)
  - Test scaling policies under load
```
