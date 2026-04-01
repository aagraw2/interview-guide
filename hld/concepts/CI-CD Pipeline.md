## 1. What is CI/CD?

**Continuous Integration (CI):** Automatically build and test code on every commit.

**Continuous Delivery (CD):** Automatically deploy code to staging/production after passing tests.

```
Without CI/CD:
  Developer commits code
  → Manual build
  → Manual testing
  → Manual deployment
  → Slow, error-prone, infrequent releases

With CI/CD:
  Developer commits code
  → Automated build
  → Automated tests
  → Automated deployment
  → Fast, reliable, frequent releases
```

---

## 2. CI/CD Pipeline Stages

```
┌──────────────────────────────────────────────────────────────┐
│                      CI/CD Pipeline                          │
└──────────────────────────────────────────────────────────────┘

1. Source
   ├─ Developer commits code to Git
   └─ Webhook triggers pipeline

2. Build
   ├─ Checkout code
   ├─ Install dependencies
   ├─ Compile code
   └─ Build Docker image

3. Test
   ├─ Unit tests
   ├─ Integration tests
   ├─ Linting / static analysis
   └─ Security scanning

4. Package
   ├─ Create artifacts (JAR, Docker image)
   └─ Push to artifact registry (Docker Hub, ECR, Nexus)

5. Deploy to Staging
   ├─ Deploy to staging environment
   ├─ Run smoke tests
   └─ Run end-to-end tests

6. Deploy to Production
   ├─ Manual approval (optional)
   ├─ Deploy to production
   ├─ Run smoke tests
   └─ Monitor metrics

7. Monitor
   ├─ Track error rates, latency
   ├─ Alert on anomalies
   └─ Rollback if needed
```

---

## 3. CI: Continuous Integration

### Goal

Catch bugs early by building and testing on every commit.

---

### Example: GitHub Actions CI Pipeline

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linter
      run: npm run lint
    
    - name: Run unit tests
      run: npm test
    
    - name: Run integration tests
      run: npm run test:integration
    
    - name: Build Docker image
      run: docker build -t myapp:${{ github.sha }} .
    
    - name: Push Docker image
      run: |
        echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
        docker push myapp:${{ github.sha }}
```

---

### Key Practices

```
✅ Run on every commit (main branch and PRs)
✅ Fast feedback (pipeline completes in < 10 minutes)
✅ Fail fast (stop on first failure)
✅ Isolated builds (each build starts from clean state)
✅ Automated tests (unit, integration, linting)
```

---

## 4. CD: Continuous Delivery

### Goal

Automatically deploy code to staging/production after passing tests.

---

### Continuous Delivery vs Continuous Deployment

```
Continuous Delivery:
  Code is automatically deployed to staging
  Manual approval required for production
  
Continuous Deployment:
  Code is automatically deployed to production
  No manual approval (fully automated)
```

Most teams use Continuous Delivery (manual approval for production).

---

### Example: GitHub Actions CD Pipeline

```yaml
name: CD Pipeline

on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Deploy to staging
      run: |
        kubectl set image deployment/myapp myapp=myapp:${{ github.sha }} -n staging
        kubectl rollout status deployment/myapp -n staging
    
    - name: Run smoke tests
      run: npm run test:smoke -- --env=staging
  
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    steps:
    - name: Deploy to production
      run: |
        kubectl set image deployment/myapp myapp=myapp:${{ github.sha }} -n production
        kubectl rollout status deployment/myapp -n production
    
    - name: Run smoke tests
      run: npm run test:smoke -- --env=production
```

**Manual approval:** The `environment: production` requires manual approval in GitHub Actions settings.

---

## 5. Deployment Strategies

### 1. Rolling Deployment

Gradually replace old instances with new ones.

```
Step 1: [v1] [v1] [v1] [v1]
Step 2: [v2] [v1] [v1] [v1]  ← Deploy v2 to 1 instance
Step 3: [v2] [v2] [v1] [v1]  ← Deploy v2 to 2 instances
Step 4: [v2] [v2] [v2] [v1]  ← Deploy v2 to 3 instances
Step 5: [v2] [v2] [v2] [v2]  ← All instances running v2
```

**Pros:** Zero downtime, simple. **Cons:** Both versions running simultaneously (compatibility issues).

---

### 2. Blue-Green Deployment

Run two identical environments (blue = current, green = new). Switch traffic all at once.

```
Blue (v1):  [v1] [v1] [v1] [v1]  ← 100% traffic
Green (v2): [v2] [v2] [v2] [v2]  ← 0% traffic (idle)

Deploy v2 to green, run tests
Switch load balancer to green

Blue (v1):  [v1] [v1] [v1] [v1]  ← 0% traffic (idle, ready for rollback)
Green (v2): [v2] [v2] [v2] [v2]  ← 100% traffic
```

**Pros:** Instant rollback (switch back to blue). **Cons:** Requires 2x infrastructure.

---

### 3. Canary Deployment

Deploy new version to a small percentage of users first. Gradually increase if no issues.

```
Step 1: [v1] [v1] [v1] [v1]      ← 100% traffic
Step 2: [v2] [v1] [v1] [v1]      ← 10% traffic to v2 (canary)
        Monitor metrics (error rate, latency)
Step 3: [v2] [v2] [v1] [v1]      ← 50% traffic to v2
        Monitor metrics
Step 4: [v2] [v2] [v2] [v2]      ← 100% traffic to v2
```

**Pros:** Low risk (catch issues early with small user base). **Cons:** More complex, requires traffic splitting.

---

### 4. Feature Flags

Deploy code to production but keep features disabled. Enable for specific users or gradually.

```
Code deployed to production:
  if (featureFlag.isEnabled('new-checkout')) {
    return newCheckout();
  } else {
    return oldCheckout();
  }

Enable for 10% of users → monitor → enable for 50% → enable for 100%
```

**Pros:** Decouple deployment from release, instant rollback (disable flag). **Cons:** Code complexity, flag management overhead.

---

## 6. Testing in CI/CD

### Test Pyramid

```
        ┌─────────────┐
        │     E2E     │  ← Few, slow, expensive
        │   (UI tests)│
        ├─────────────┤
        │ Integration │  ← Moderate, test service interactions
        │    Tests    │
        ├─────────────┤
        │    Unit     │  ← Many, fast, cheap
        │    Tests    │
        └─────────────┘
```

**Best practice:** Many unit tests, some integration tests, few E2E tests.

---

### Types of Tests

```
Unit Tests:
  - Test individual functions/classes
  - Fast (milliseconds)
  - Run on every commit

Integration Tests:
  - Test service interactions (API, database)
  - Slower (seconds)
  - Run on every commit

End-to-End Tests:
  - Test full user flows (UI, backend, database)
  - Slow (minutes)
  - Run before deployment

Smoke Tests:
  - Quick sanity checks after deployment
  - Verify critical paths work
  - Run after deployment

Load Tests:
  - Test performance under load
  - Run periodically (nightly, weekly)
```

---

## 7. Artifact Management

### Docker Registry

Store Docker images.

```
Examples:
  - Docker Hub (public)
  - AWS ECR (Elastic Container Registry)
  - Google Container Registry (GCR)
  - Azure Container Registry (ACR)
  - Harbor (self-hosted)
```

**Tagging strategy:**

```
myapp:latest           ← Latest build (not recommended for production)
myapp:1.2.3            ← Semantic version
myapp:abc123           ← Git commit SHA (recommended)
myapp:main-abc123      ← Branch + commit SHA
```

---

### Artifact Repository

Store build artifacts (JARs, WARs, npm packages).

```
Examples:
  - Nexus
  - Artifactory
  - AWS CodeArtifact
  - npm registry (for npm packages)
```

---

## 8. CI/CD Tools

### GitHub Actions

```
Pros:
  ✅ Integrated with GitHub
  ✅ Free for public repos, generous free tier for private
  ✅ Easy to set up (YAML in repo)
  ✅ Large marketplace of actions

Cons:
  ❌ GitHub-only
  ❌ Limited self-hosted runner options
```

---

### GitLab CI/CD

```
Pros:
  ✅ Integrated with GitLab
  ✅ Free tier, self-hosted option
  ✅ Built-in container registry
  ✅ Auto DevOps (automatic CI/CD setup)

Cons:
  ❌ GitLab-only
```

---

### Jenkins

```
Pros:
  ✅ Open-source, self-hosted
  ✅ Highly customizable (plugins)
  ✅ Works with any Git provider

Cons:
  ❌ Complex to set up and maintain
  ❌ Requires dedicated infrastructure
  ❌ UI feels dated
```

---

### CircleCI

```
Pros:
  ✅ Fast builds (parallelization)
  ✅ Works with GitHub, Bitbucket
  ✅ Good free tier

Cons:
  ❌ Expensive for large teams
```

---

### AWS CodePipeline

```
Pros:
  ✅ Integrates with AWS services
  ✅ Managed service (no infrastructure)

Cons:
  ❌ AWS-only
  ❌ Limited features compared to others
```

---

## 9. Security in CI/CD

### Secrets Management

Never hardcode secrets in code or CI/CD config.

```
Bad:
  docker login -u myuser -p mypassword

Good:
  docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD
  (secrets stored in CI/CD tool's secret manager)
```

**Secret storage:**

```
- GitHub Actions Secrets
- GitLab CI/CD Variables
- AWS Secrets Manager
- HashiCorp Vault
```

---

### Dependency Scanning

Scan dependencies for known vulnerabilities.

```
Tools:
  - Snyk
  - Dependabot (GitHub)
  - npm audit
  - OWASP Dependency-Check
```

---

### Container Scanning

Scan Docker images for vulnerabilities.

```
Tools:
  - Trivy
  - Clair
  - Snyk Container
  - AWS ECR image scanning
```

---

### Static Application Security Testing (SAST)

Analyze source code for security vulnerabilities.

```
Tools:
  - SonarQube
  - Checkmarx
  - Veracode
```

---

## 10. Monitoring and Rollback

### Post-Deployment Monitoring

```
Metrics to monitor:
  - Error rate (5xx errors)
  - Latency (p50, p95, p99)
  - Request rate (RPS)
  - CPU, memory usage

Alert if:
  - Error rate > 1% (baseline: 0.1%)
  - p99 latency > 1s (baseline: 500ms)
  - CPU > 80%
```

---

### Automated Rollback

Automatically rollback if metrics degrade.

```
Deploy v2
→ Monitor for 5 minutes
→ If error rate > 1%, rollback to v1
→ If latency > 1s, rollback to v1
```

**Tools:** Flagger (Kubernetes), AWS CodeDeploy, Spinnaker.

---

## 11. Common Interview Questions + Answers

### Q: What's the difference between CI and CD?

> "CI is Continuous Integration — automatically building and testing code on every commit to catch bugs early. CD is Continuous Delivery — automatically deploying code to staging or production after passing tests. CI focuses on the build and test stages, CD focuses on the deployment stage. Together, they enable fast, reliable releases."

---

### Q: What's the difference between blue-green and canary deployments?

> "Blue-green deployment runs two identical environments. You deploy the new version to the green environment, test it, then switch all traffic at once. It allows instant rollback by switching back to blue. Canary deployment gradually routes a small percentage of traffic to the new version, monitors metrics, and increases traffic if there are no issues. Blue-green is all-or-nothing, canary is gradual. Canary is lower risk because you catch issues with a small user base first."

---

### Q: How do you handle secrets in CI/CD?

> "Never hardcode secrets in code or CI/CD config. Store secrets in the CI/CD tool's secret manager like GitHub Actions Secrets or AWS Secrets Manager. Reference them as environment variables in the pipeline. For production, use a dedicated secret manager like HashiCorp Vault or AWS Secrets Manager, and have the application fetch secrets at runtime. Rotate secrets regularly and use least-privilege access policies."

---

### Q: How do you ensure fast CI/CD pipelines?

> "Parallelize tests — run unit tests, integration tests, and linting in parallel. Cache dependencies to avoid re-downloading on every build. Use incremental builds — only rebuild what changed. Keep the test suite fast by focusing on unit tests and limiting slow E2E tests. Use fast CI/CD runners with sufficient resources. Fail fast — stop the pipeline on the first failure. Aim for pipelines under 10 minutes for fast feedback."

---

## 12. Interview Tricks & Pitfalls

### ✅ Trick 1: Know deployment strategies

Blue-green, canary, and rolling deployments are common interview topics. Know the trade-offs.

---

### ✅ Trick 2: Mention the test pyramid

Shows you understand testing strategy — many unit tests, few E2E tests.

---

### ✅ Trick 3: Talk about monitoring and rollback

Deployment isn't done when code is deployed. Mention monitoring metrics and automated rollback.

---

### ❌ Pitfall 1: Confusing CI and CD

CI is build + test. CD is deploy. Don't conflate them.

---

### ❌ Pitfall 2: Ignoring secrets management

Never hardcode secrets. Always mention using a secret manager.

---

### ❌ Pitfall 3: Not testing before production

Always deploy to staging first, run tests, then deploy to production. Never deploy directly to production.

---

## 13. Quick Reference

```
CI/CD: Continuous Integration / Continuous Delivery

CI (Continuous Integration):
  - Build and test on every commit
  - Catch bugs early
  - Fast feedback (< 10 minutes)

CD (Continuous Delivery):
  - Automatically deploy to staging/production
  - Manual approval for production (optional)

Pipeline Stages:
  1. Source:   Commit triggers pipeline
  2. Build:    Compile, build Docker image
  3. Test:     Unit, integration, linting, security
  4. Package:  Push artifacts to registry
  5. Deploy:   Deploy to staging, then production
  6. Monitor:  Track metrics, rollback if needed

Deployment Strategies:
  Rolling:     Gradually replace old with new (zero downtime)
  Blue-Green:  Two environments, switch all at once (instant rollback)
  Canary:      Deploy to small % first, gradually increase (low risk)
  Feature Flags: Deploy code, enable features gradually

Testing:
  Unit:        Fast, many (test functions/classes)
  Integration: Moderate, some (test service interactions)
  E2E:         Slow, few (test full user flows)
  Smoke:       Quick sanity checks after deployment

Artifact Management:
  Docker Registry: Docker Hub, ECR, GCR, Harbor
  Artifact Repo:   Nexus, Artifactory, CodeArtifact

CI/CD Tools:
  GitHub Actions: Integrated with GitHub, easy setup
  GitLab CI/CD:   Integrated with GitLab, self-hosted option
  Jenkins:        Open-source, highly customizable, complex
  CircleCI:       Fast, parallelization
  AWS CodePipeline: AWS-only, managed service

Security:
  Secrets:     Store in secret manager (never hardcode)
  Dependency:  Scan for vulnerabilities (Snyk, Dependabot)
  Container:   Scan Docker images (Trivy, Clair)
  SAST:        Static code analysis (SonarQube)

Monitoring:
  Metrics:     Error rate, latency, RPS, CPU/memory
  Rollback:    Automatic if metrics degrade

Best Practices:
  ✅ Run CI on every commit
  ✅ Deploy to staging first
  ✅ Automate tests
  ✅ Fast pipelines (< 10 minutes)
  ✅ Monitor after deployment
  ✅ Automated rollback
  ✅ Use feature flags for gradual rollout
```
