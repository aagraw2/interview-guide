## 1. What is Zero Trust Architecture?

Zero Trust is a security model that assumes no user or service should be trusted by default, regardless of whether they're inside or outside the network perimeter.

```
Traditional (Perimeter Security):
  Outside network: Untrusted
  Inside network: Trusted
  
  Problem: Once inside, full access

Zero Trust:
  Everything: Untrusted by default
  Every request: Authenticated and authorized
  
  Principle: "Never trust, always verify"
```

**Key principle:** Trust nothing, verify everything, regardless of location.

---

## 2. Core Principles

### Never Trust, Always Verify

```
Every request must be:
  1. Authenticated (who are you?)
  2. Authorized (what can you do?)
  3. Encrypted (secure communication)

No implicit trust based on:
  - Network location
  - Previous authentication
  - Device ownership
```

### Least Privilege Access

```
Grant minimum permissions needed:
  - User: Access only their data
  - Service: Access only required APIs
  - Time-bound: Expire after use

Example:
  ❌ Admin access to all databases
  ✅ Read access to specific table for 1 hour
```

### Assume Breach

```
Design as if already compromised:
  - Segment network (limit blast radius)
  - Monitor all activity (detect anomalies)
  - Encrypt everything (data at rest and in transit)
```

---

## 3. Zero Trust Components

### Identity and Access Management (IAM)

```
Every user and service has identity:
  - Users: SSO, MFA
  - Services: Service accounts, certificates

Authentication:
  - Multi-factor authentication (MFA)
  - Certificate-based (mTLS)
  - Token-based (JWT, OAuth)

Authorization:
  - Role-based access control (RBAC)
  - Attribute-based access control (ABAC)
  - Policy-based (OPA)
```

### Micro-Segmentation

```
Divide network into small segments:
  - Each service in own segment
  - Firewall rules between segments
  - Limit lateral movement

Example:
  Web tier: Can access API tier only
  API tier: Can access database tier only
  Database tier: No outbound access
```

### Continuous Verification

```
Verify on every request:
  - Check authentication token
  - Validate authorization
  - Assess device posture
  - Evaluate risk score

Not just at login:
  - Re-authenticate periodically
  - Re-authorize on sensitive operations
  - Revoke access on anomalies
```

---

## 4. Implementation Example

### Service-to-Service Authentication (mTLS)

```python
import requests

# Client certificate
cert = ('/path/to/client.crt', '/path/to/client.key')

# Server CA certificate
verify = '/path/to/ca.crt'

# Make request with mTLS
response = requests.get(
    'https://api.example.com/users',
    cert=cert,
    verify=verify
)

# Server verifies client certificate
# Client verifies server certificate
# Mutual authentication
```


### Policy-Based Authorization

```python
# Open Policy Agent (OPA) policy
package authz

default allow = false

# Allow if user has required role
allow {
    input.user.roles[_] == "admin"
}

# Allow if user owns resource
allow {
    input.user.id == input.resource.owner_id
}

# Check policy
import requests

policy_check = {
    "input": {
        "user": {"id": 123, "roles": ["user"]},
        "resource": {"owner_id": 123},
        "action": "read"
    }
}

response = requests.post(
    'http://opa:8181/v1/data/authz/allow',
    json=policy_check
)

if response.json()['result']:
    # Authorized
    process_request()
else:
    # Denied
    return 403
```

### Service Mesh (Istio)

```yaml
# Mutual TLS for all services
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT

# Authorization policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-policy
spec:
  selector:
    matchLabels:
      app: api
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/web"]
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]
```

---

## 5. Zero Trust for Users

### Multi-Factor Authentication (MFA)

```
Something you know: Password
Something you have: Phone, hardware token
Something you are: Biometric

Example flow:
  1. User enters password
  2. System sends code to phone
  3. User enters code
  4. Access granted

Prevents: Password theft, phishing
```

### Conditional Access

```
Grant access based on context:
  - Device: Managed, compliant
  - Location: Office, VPN
  - Risk: Low, medium, high
  - Time: Business hours

Example:
  If (device.managed AND location.office):
      grant_access()
  Else:
      require_mfa()
```

### Just-In-Time Access

```
Grant temporary access:
  - Request access for specific resource
  - Approve for limited time (1 hour)
  - Auto-revoke after expiry

Example:
  User requests database access
  Manager approves for 2 hours
  Access auto-revoked after 2 hours
```

---

## 6. Zero Trust for Services

### Service Identity

```
Every service has identity:
  - Kubernetes: Service account
  - AWS: IAM role
  - Certificate: X.509 certificate

Example (Kubernetes):
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: api-service
  
  Service runs as api-service
  Other services verify identity
```

### Mutual TLS (mTLS)

```
Both client and server authenticate:
  1. Client connects to server
  2. Server sends certificate
  3. Client verifies server certificate
  4. Client sends certificate
  5. Server verifies client certificate
  6. Encrypted communication

Benefits:
  - Mutual authentication
  - Encrypted traffic
  - No shared secrets
```

### API Gateway with Authentication

```
All requests through gateway:
  1. Client sends request to gateway
  2. Gateway validates JWT token
  3. Gateway checks authorization
  4. Gateway forwards to service
  5. Service trusts gateway

Gateway enforces:
  - Authentication
  - Authorization
  - Rate limiting
  - Logging
```

---

## 7. Monitoring and Detection

### Continuous Monitoring

```
Monitor all activity:
  - Authentication attempts
  - Authorization decisions
  - API calls
  - Data access

Detect anomalies:
  - Unusual access patterns
  - Failed authentication
  - Privilege escalation
  - Data exfiltration
```

### Security Information and Event Management (SIEM)

```
Centralized logging and analysis:
  - Collect logs from all services
  - Correlate events
  - Detect threats
  - Alert on anomalies

Example:
  User logs in from US
  5 minutes later, login from China
  Alert: Impossible travel
```

### Zero Trust Network Access (ZTNA)

```
Replace VPN with zero trust:
  - No network access by default
  - Authenticate per application
  - Authorize per request
  - Encrypt all traffic

Example:
  User accesses app1:
    Authenticate → Authorize → Grant access to app1 only
  User accesses app2:
    Re-authenticate → Re-authorize → Grant access to app2 only
```

---

## 8. Challenges

### Complexity

```
Problem:
  Zero trust is complex to implement
  Requires identity management, policies, monitoring

Solutions:
  - Start small (one service at a time)
  - Use managed services (Istio, AWS IAM)
  - Automate policy enforcement
```

### Performance overhead

```
Problem:
  Authentication and authorization on every request
  Adds latency

Solutions:
  - Cache authentication tokens
  - Use fast authorization (OPA)
  - Optimize mTLS handshake
```

### Legacy systems

```
Problem:
  Old systems don't support modern authentication

Solutions:
  - Proxy with authentication (sidecar)
  - Gradual migration
  - Network segmentation
```

---

## 9. When to Use Zero Trust

### Good fit

```
✅ Cloud-native applications
✅ Microservices architecture
✅ Remote workforce
✅ High security requirements
✅ Compliance (SOC2, HIPAA)

Examples:
  - Financial services
  - Healthcare
  - Government
  - SaaS applications
```

### Poor fit

```
❌ Simple applications
❌ Small teams (overhead too high)
❌ Legacy systems (hard to retrofit)

Examples:
  - Internal tools
  - Prototypes
  - Monolithic applications
```

---

## 10. Common Interview Questions + Answers

### Q: What is zero trust architecture?

> "Zero trust is a security model that assumes no user or service should be trusted by default, regardless of network location. Instead of trusting everything inside the network perimeter, zero trust requires authentication and authorization for every request. The principle is 'never trust, always verify.' This is critical in modern cloud environments where the traditional network perimeter doesn't exist and services communicate across the internet."

### Q: How does zero trust differ from traditional security?

> "Traditional security uses perimeter-based defense — once you're inside the network, you're trusted. Zero trust assumes breach and verifies every request regardless of location. Traditional security uses VPNs for remote access, giving broad network access. Zero trust uses application-level access, granting access only to specific resources. Traditional security authenticates once at login. Zero trust continuously verifies on every request. This prevents lateral movement if an attacker compromises one service."

### Q: What are the key components of zero trust?

> "First, strong identity and access management with MFA and certificate-based authentication. Second, micro-segmentation to limit lateral movement by isolating services. Third, least privilege access, granting only minimum required permissions. Fourth, continuous verification on every request, not just at login. Fifth, encryption everywhere with mTLS for service-to-service communication. Finally, comprehensive monitoring to detect anomalies and respond to threats. These work together to ensure nothing is trusted by default."

### Q: What are the challenges with implementing zero trust?

> "First, complexity — it requires identity management, policy enforcement, and monitoring across all services. Second, performance overhead from authenticating and authorizing every request, though caching helps. Third, legacy systems that don't support modern authentication need proxies or gradual migration. Fourth, organizational change — teams need to adopt new security practices. Finally, initial setup cost is high. Start small with one service, use managed solutions like service mesh, and gradually expand coverage."

---

## 11. Quick Reference

```
What is Zero Trust?
  Never trust, always verify
  Authenticate and authorize every request
  No implicit trust based on location

Core principles:
  - Never trust, always verify
  - Least privilege access
  - Assume breach

Components:
  - Identity and Access Management (IAM)
  - Micro-segmentation
  - Continuous verification
  - Encryption (mTLS)
  - Monitoring and detection

For users:
  - Multi-factor authentication (MFA)
  - Conditional access (device, location, risk)
  - Just-in-time access (temporary)

For services:
  - Service identity (service accounts, certificates)
  - Mutual TLS (mTLS)
  - API gateway with authentication
  - Policy-based authorization (OPA)

Implementation:
  - Service mesh (Istio, Linkerd)
  - Identity provider (Okta, Auth0)
  - Policy engine (OPA)
  - SIEM (Splunk, ELK)

Challenges:
  - Complexity (start small, use managed services)
  - Performance (cache tokens, optimize mTLS)
  - Legacy systems (proxy, gradual migration)

When to use:
  ✅ Cloud-native applications
  ✅ Microservices
  ✅ Remote workforce
  ✅ High security requirements
  
  ❌ Simple applications
  ❌ Small teams
  ❌ Legacy systems

Best practices:
  - Start with one service
  - Use service mesh for mTLS
  - Implement MFA for users
  - Monitor all activity
  - Automate policy enforcement
  - Encrypt everything
```
