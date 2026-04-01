## 1. What is Chaos Engineering?

Chaos Engineering is the practice of intentionally injecting failures into a system to test its resilience and identify weaknesses before they cause real outages.

```
Traditional Testing:
  Test expected scenarios
  "Does it work when everything is fine?"

Chaos Engineering:
  Test unexpected failures
  "Does it work when things go wrong?"
  
  Inject failures:
    - Kill servers
    - Introduce latency
    - Corrupt data
    - Network partitions
```

**Key principle:** Break things on purpose to learn how to make them more resilient.

---

## 2. Core Concepts

### Hypothesis

A prediction about system behavior under failure.

```
Hypothesis:
  "If we kill 1 instance of the order service,
   the system will continue processing orders
   with <1% error rate increase"

Test:
  1. Measure baseline (error rate)
  2. Kill 1 instance
  3. Measure impact
  4. Compare to hypothesis

Result:
  ✅ Hypothesis confirmed: System is resilient
  ❌ Hypothesis rejected: System needs improvement
```

### Blast Radius

The scope of impact from a chaos experiment.

```
Small blast radius:
  - Single instance
  - Single availability zone
  - Canary environment
  
  Low risk, safe to start

Large blast radius:
  - All instances
  - All regions
  - Production environment
  
  High risk, only after confidence
```

### Steady State

Normal system behavior (metrics, SLIs).

```
Steady state metrics:
  - Request success rate: 99.9%
  - Latency p99: 200ms
  - Throughput: 1000 req/s

Chaos experiment:
  1. Measure steady state
  2. Inject failure
  3. Measure deviation from steady state
  4. Rollback failure
  5. Verify return to steady state
```

---

## 3. Types of Chaos Experiments

### Resource Failures

```
CPU exhaustion:
  Consume 100% CPU on instance
  Test: Does auto-scaling kick in?

Memory exhaustion:
  Consume all available memory
  Test: Does OOM killer work? Does service restart?

Disk full:
  Fill disk to 100%
  Test: Does service handle gracefully?
```

### Network Failures

```
Latency injection:
  Add 500ms delay to all requests
  Test: Do timeouts work? Does circuit breaker open?

Packet loss:
  Drop 10% of packets
  Test: Do retries work? Is data consistent?

Network partition:
  Isolate service from dependencies
  Test: Does fallback work? Is error handling correct?
```

### Service Failures

```
Instance termination:
  Kill random instances
  Test: Does load balancer route around? Does auto-scaling replace?

Dependency failure:
  Make downstream service return errors
  Test: Does circuit breaker work? Does fallback work?

Database failure:
  Kill database connection
  Test: Does connection pool recover? Is data consistent?
```

### Application Failures

```
Exception injection:
  Throw random exceptions
  Test: Is error handling correct? Are errors logged?

Slow responses:
  Make service respond slowly
  Test: Do timeouts work? Does circuit breaker open?

Invalid responses:
  Return malformed data
  Test: Is input validation correct? Does service handle gracefully?
```

---

## 4. Chaos Engineering Process

### 1. Define Steady State

```
Identify key metrics:
  - Success rate
  - Latency (p50, p95, p99)
  - Throughput
  - Error rate

Measure baseline:
  Success rate: 99.9%
  Latency p99: 200ms
  Throughput: 1000 req/s
```

### 2. Hypothesize

```
Hypothesis:
  "If we kill 1 instance of the API service,
   success rate will remain >99.5%
   and latency p99 will remain <300ms"
```

### 3. Inject Failure

```
Start small (canary):
  - Single instance
  - Single AZ
  - 1% of traffic

Gradually increase:
  - Multiple instances
  - Multiple AZs
  - 10% of traffic
```

### 4. Measure Impact

```
Monitor metrics:
  - Success rate: 99.7% (within threshold)
  - Latency p99: 250ms (within threshold)
  - Throughput: 950 req/s (acceptable)

Result: Hypothesis confirmed ✅
```

### 5. Learn and Improve

```
If hypothesis confirmed:
  - System is resilient
  - Document findings
  - Increase blast radius

If hypothesis rejected:
  - Identify weakness
  - Fix issue
  - Re-test
```

---

## 5. Chaos Engineering Tools

### Netflix Chaos Monkey

```
Randomly terminates instances in production
Part of Simian Army suite

Features:
  - Random instance termination
  - Scheduled chaos (business hours only)
  - Opt-in per service

Use case: Test instance failure resilience
```

### Gremlin

```
Commercial chaos engineering platform
Comprehensive failure injection

Features:
  - Resource attacks (CPU, memory, disk)
  - Network attacks (latency, packet loss)
  - State attacks (kill process, shutdown)
  - Scheduled experiments
  - Blast radius control

Use case: Enterprise chaos engineering
```

### Chaos Mesh

```
Open-source chaos engineering for Kubernetes
Cloud-native

Features:
  - Pod failures
  - Network chaos
  - Stress testing
  - Time chaos
  - Kernel chaos

Use case: Kubernetes environments
```

### Litmus

```
Open-source chaos engineering for Kubernetes
CNCF project

Features:
  - Pre-defined chaos experiments
  - Custom experiments
  - Chaos workflows
  - Observability integration

Use case: Kubernetes, cloud-native
```

---

## 6. Implementation Example

### Chaos Monkey (Python)

```python
import random
import boto3
import time

class ChaosMonkey:
    def __init__(self, region, tag_key, tag_value):
        self.ec2 = boto3.client('ec2', region_name=region)
        self.tag_key = tag_key
        self.tag_value = tag_value
    
    def get_instances(self):
        response = self.ec2.describe_instances(
            Filters=[
                {'Name': f'tag:{self.tag_key}', 'Values': [self.tag_value]},
                {'Name': 'instance-state-name', 'Values': ['running']}
            ]
        )
        
        instances = []
        for reservation in response['Reservations']:
            for instance in reservation['Instances']:
                instances.append(instance['InstanceId'])
        
        return instances
    
    def terminate_random_instance(self):
        instances = self.get_instances()
        
        if not instances:
            print("No instances found")
            return
        
        # Terminate random instance
        victim = random.choice(instances)
        print(f"Terminating instance: {victim}")
        
        self.ec2.terminate_instances(InstanceIds=[victim])
    
    def run(self, interval=3600):
        while True:
            self.terminate_random_instance()
            time.sleep(interval)

# Run Chaos Monkey
monkey = ChaosMonkey(
    region='us-east-1',
    tag_key='chaos-enabled',
    tag_value='true'
)
monkey.run(interval=3600)  # Every hour
```

### Network Latency Injection (Linux)

```bash
# Add 500ms latency to all traffic
tc qdisc add dev eth0 root netem delay 500ms

# Add 500ms latency with 100ms variance
tc qdisc add dev eth0 root netem delay 500ms 100ms

# Add packet loss (10%)
tc qdisc add dev eth0 root netem loss 10%

# Remove latency
tc qdisc del dev eth0 root
```

### Chaos Mesh (Kubernetes)

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-experiment
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - default
    labelSelectors:
      app: my-service
  scheduler:
    cron: "@every 1h"
```

---

## 7. Best Practices

### Start Small

```
1. Start in non-production (staging)
2. Small blast radius (single instance)
3. Business hours only
4. Manual experiments first
5. Gradually automate
6. Gradually increase blast radius
7. Eventually run in production
```

### Define Rollback

```
Always have a rollback plan:
  - How to stop experiment?
  - How to restore service?
  - Who to notify?

Example:
  Experiment: Kill database connection
  Rollback: Restart service, restore connection
  Notify: On-call engineer
```

### Monitor Closely

```
Monitor during experiment:
  - Success rate
  - Latency
  - Error rate
  - Logs
  - Alerts

Set thresholds:
  If success rate < 95%: Stop experiment
  If latency p99 > 1s: Stop experiment
```

### Document Findings

```
Document:
  - Hypothesis
  - Experiment details
  - Results
  - Lessons learned
  - Action items

Example:
  Hypothesis: System handles instance failure
  Result: Success rate dropped to 90% (failed)
  Lesson: Load balancer health check too slow
  Action: Reduce health check interval from 30s to 5s
```

---

## 8. Chaos Engineering in Production

### GameDays

```
Scheduled chaos events:
  - Team participates
  - Inject multiple failures
  - Practice incident response
  - Learn and improve

Example:
  9:00 AM: Start GameDay
  9:15 AM: Kill 2 instances
  9:30 AM: Inject network latency
  9:45 AM: Simulate database failure
  10:00 AM: Debrief and document
```

### Continuous Chaos

```
Automated chaos in production:
  - Random instance termination
  - Scheduled experiments
  - Low blast radius
  - Automatic rollback

Example:
  Every hour:
    - Kill 1 random instance
    - Monitor for 5 minutes
    - If success rate < 99%: Alert and stop
    - Otherwise: Continue
```

### Chaos as Code

```
Define experiments as code:
  - Version controlled
  - Peer reviewed
  - Automated execution
  - Reproducible

Example (Chaos Mesh):
  apiVersion: chaos-mesh.org/v1alpha1
  kind: Workflow
  metadata:
    name: chaos-workflow
  spec:
    entry: pod-kill
    templates:
      - name: pod-kill
        templateType: PodChaos
        deadline: 5m
```

---

## 9. Challenges

### Cultural resistance

```
Problem:
  "Don't break production!"
  Fear of causing outages

Solution:
  - Start in non-production
  - Show value (find issues before customers do)
  - Get buy-in from leadership
  - Celebrate learning, not blame
```

### Blast radius control

```
Problem:
  Experiment impacts too many users

Solution:
  - Start small (single instance)
  - Gradual rollout (1% → 10% → 100%)
  - Automatic rollback (if metrics degrade)
  - Limit to specific environments
```

### Measuring impact

```
Problem:
  Hard to measure impact of experiment

Solution:
  - Define clear metrics (success rate, latency)
  - Measure baseline before experiment
  - Compare to baseline during experiment
  - Use statistical significance
```

---

## 10. Common Interview Questions + Answers

### Q: What is chaos engineering and why is it important?

> "Chaos engineering is the practice of intentionally injecting failures into a system to test its resilience. Instead of waiting for failures to happen in production, you proactively cause them in a controlled way to identify weaknesses. It's important because it helps you find issues before customers do, validates that your resilience mechanisms like circuit breakers and retries actually work, and builds confidence in your system's ability to handle failures. Netflix pioneered this with Chaos Monkey to ensure their services could handle instance failures."

### Q: How do you run a chaos experiment safely?

> "Start small with a clear hypothesis and rollback plan. Begin in non-production environments, then gradually move to production with a small blast radius — maybe just one instance or 1% of traffic. Define your steady state metrics like success rate and latency, measure the baseline, inject the failure, and monitor closely. Set automatic rollback thresholds — if success rate drops below 95%, stop the experiment immediately. Always run during business hours when engineers are available, and document everything you learn."

### Q: What types of failures should you test with chaos engineering?

> "Test the failures most likely to happen in production. Instance failures by randomly terminating servers to validate auto-scaling and load balancing. Network issues like latency, packet loss, and partitions to test timeouts and circuit breakers. Dependency failures by making downstream services unavailable to test fallbacks. Resource exhaustion like CPU or memory spikes to test resource limits. Also test application-level failures like slow responses or invalid data to validate error handling."

### Q: What are the challenges with chaos engineering?

> "First, cultural resistance — teams fear breaking production. You need leadership buy-in and should start in non-production to build confidence. Second, blast radius control — you need to ensure experiments don't impact too many users. Start small and gradually increase scope. Third, measuring impact — you need clear metrics and baselines to determine if an experiment succeeded. Finally, knowing what to test — focus on realistic failures that are likely to happen, not every possible edge case."

---

## 11. Quick Reference

```
What is Chaos Engineering?
  Intentionally inject failures to test resilience
  Find weaknesses before they cause outages
  "Break things on purpose"

Core concepts:
  Hypothesis: Prediction about system behavior
  Blast Radius: Scope of impact
  Steady State: Normal system behavior

Types of experiments:
  - Resource failures (CPU, memory, disk)
  - Network failures (latency, packet loss, partition)
  - Service failures (instance termination, dependency failure)
  - Application failures (exceptions, slow responses)

Process:
  1. Define steady state (metrics)
  2. Hypothesize (prediction)
  3. Inject failure (start small)
  4. Measure impact (compare to baseline)
  5. Learn and improve (fix issues)

Popular tools:
  - Chaos Monkey (Netflix, instance termination)
  - Gremlin (commercial, comprehensive)
  - Chaos Mesh (Kubernetes, open-source)
  - Litmus (Kubernetes, CNCF)

Best practices:
  - Start small (non-production, single instance)
  - Define rollback plan
  - Monitor closely (set thresholds)
  - Document findings
  - Gradually increase blast radius
  - Run during business hours

Production chaos:
  - GameDays (scheduled chaos events)
  - Continuous chaos (automated)
  - Chaos as code (version controlled)

Challenges:
  - Cultural resistance (fear of breaking production)
  - Blast radius control (limit impact)
  - Measuring impact (need clear metrics)

Benefits:
  - Find issues before customers do
  - Validate resilience mechanisms
  - Build confidence in system
  - Improve incident response
  - Document system behavior

When to use:
  ✅ Production systems with high availability requirements
  ✅ After implementing resilience patterns
  ✅ To validate disaster recovery
  ✅ To train teams on incident response
```
