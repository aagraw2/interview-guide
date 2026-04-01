## 1. What is a DDoS Attack?

**Distributed Denial of Service (DDoS)** attacks overwhelm a system with traffic from many sources, making it unavailable to legitimate users.

```
Normal traffic:
  1,000 requests/sec → Server handles normally

DDoS attack:
  1,000,000 requests/sec from 10,000 bots → Server overwhelmed → Legitimate users can't access
```

**Key characteristic:** Traffic comes from many distributed sources (botnet), making it hard to block by IP address alone.

---

## 2. Types of DDoS Attacks

### Volumetric Attacks (Layer 3/4)

Flood the network with massive amounts of traffic to saturate bandwidth.

```
UDP Flood:
  - Send massive UDP packets to random ports
  - Server tries to respond with ICMP "port unreachable"
  - Bandwidth exhausted

ICMP Flood (Ping Flood):
  - Send massive ICMP echo requests
  - Server responds with echo replies
  - Bandwidth exhausted

DNS Amplification:
  - Attacker sends DNS queries with spoofed source IP (victim's IP)
  - DNS servers send large responses to victim
  - Amplification factor: 50-100x
```

**Goal:** Saturate network bandwidth (Gbps-scale attacks).

---

### Protocol Attacks (Layer 4)

Exploit weaknesses in network protocols to exhaust server resources (CPU, memory, connection tables).

```
SYN Flood:
  - Send massive TCP SYN packets (connection requests)
  - Server allocates resources for half-open connections
  - Connection table fills up → legitimate connections rejected

Ping of Death:
  - Send malformed or oversized ICMP packets
  - Crashes or hangs vulnerable systems

Smurf Attack:
  - Send ICMP requests to broadcast address with spoofed source IP
  - All hosts on network respond to victim
```

**Goal:** Exhaust server resources (connection tables, CPU).

---

### Application Layer Attacks (Layer 7)

Target the application itself with seemingly legitimate requests that are expensive to process.

```
HTTP Flood:
  - Send massive HTTP GET/POST requests
  - Each request looks legitimate
  - Server processes each request → CPU/DB exhausted

Slowloris:
  - Open many connections, send partial HTTP requests slowly
  - Server keeps connections open waiting for complete request
  - Connection pool exhausted → legitimate users can't connect

DNS Query Flood:
  - Send massive DNS queries to authoritative DNS servers
  - DNS server CPU exhausted

API Abuse:
  - Target expensive API endpoints (search, report generation)
  - Each request is valid but resource-intensive
```

**Goal:** Exhaust application resources (CPU, database, memory).

**Hardest to defend:** Requests look legitimate, hard to distinguish from real users.

---

## 3. DDoS Protection Strategies

### 1. Over-Provisioning / Elastic Scaling

Provision enough capacity to absorb attack traffic.

```
Normal capacity:  10,000 RPS
DDoS capacity:    100,000 RPS (10x over-provisioning)

Use auto-scaling to add capacity during attacks
```

**Pros:**

- Simple
- Handles legitimate traffic spikes too

**Cons:**

- Expensive (paying for unused capacity)
- Doesn't work for massive attacks (Tbps-scale)

---

### 2. Rate Limiting

Limit requests per IP, per user, or globally.

```
Per IP:     100 requests/minute per IP
Per user:   1,000 requests/minute per authenticated user
Global:     100,000 requests/second total
```

**Implementation:**

```python
# Token bucket rate limiter
from redis import Redis

redis = Redis()

def is_allowed(ip):
    key = f"rate_limit:{ip}"
    current = redis.get(key)
    
    if current is None:
        redis.setex(key, 60, 1)  # 1 request, expires in 60 seconds
        return True
    
    if int(current) < 100:  # 100 requests per minute
        redis.incr(key)
        return True
    
    return False  # Rate limit exceeded
```

**Pros:**

- Effective against single-source attacks
- Protects backend from overload

**Cons:**

- Distributed attacks use many IPs (hard to rate limit)
- May block legitimate users behind NAT

---

### 3. Anycast + Edge Absorption

Distribute traffic across many edge locations using Anycast routing. Attack traffic is absorbed at the edge, far from the origin server.

```
┌──────────────────────────────────────────────────┐
│  Attacker sends traffic to 1.2.3.4              │
└──────────────┬───────────────────────────────────┘
               │
               │ Anycast routes to nearest edge
               ▼
┌──────────────────────────────────────────────────┐
│  Edge PoPs (Points of Presence)                  │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐          │
│  │ Edge 1  │  │ Edge 2  │  │ Edge 3  │          │
│  │ (US)    │  │ (EU)    │  │ (Asia)  │          │
│  └────┬────┘  └────┬────┘  └────┬────┘          │
│       │            │             │               │
│       └────────────┴─────────────┘               │
│                    │                             │
│         Filter malicious traffic                 │
│                    │                             │
│                    ▼                             │
│         ┌──────────────────┐                     │
│         │  Origin Server   │                     │
│         └──────────────────┘                     │
└──────────────────────────────────────────────────┘
```

**How it works:**

1. Anycast IP (1.2.3.4) is announced from multiple edge locations
2. BGP routes traffic to the nearest edge
3. Edge filters malicious traffic, forwards legitimate traffic to origin
4. Attack traffic is distributed across many edges (diluted)

**Pros:**

- Massive capacity (Tbps-scale)
- Low latency (traffic routed to nearest edge)
- Origin server protected

**Cons:**

- Requires global infrastructure (Cloudflare, AWS Shield, Akamai)
- Expensive

---

### 4. IP Reputation & Blacklists

Block traffic from known malicious IPs.

```
Sources:
  - Threat intelligence feeds (Spamhaus, Talos)
  - Historical attack data
  - Botnet IP lists
```

**Implementation:**

```python
# Check IP against blacklist
def is_blacklisted(ip):
    return redis.sismember("ip_blacklist", ip)

if is_blacklisted(request.ip):
    return 403  # Forbidden
```

**Pros:**

- Blocks known attackers
- Low false positive rate

**Cons:**

- Attackers use new IPs (botnets rotate IPs)
- Legitimate users may share IPs with attackers (NAT, VPN)

---

### 5. CAPTCHAs & Proof of Work

Challenge suspicious traffic to prove it's human.

```
Suspicious request detected
→ Show CAPTCHA ("Select all traffic lights")
→ User solves CAPTCHA
→ Request allowed
```

**Proof of Work (PoW):**

```
Client must solve a computational puzzle before request is processed
→ Slows down bots (can't send millions of requests/sec)
→ Legitimate users unaffected (one puzzle per session)
```

**Pros:**

- Effective against bots
- Doesn't block legitimate users

**Cons:**

- Degrades user experience
- Sophisticated bots can solve CAPTCHAs (CAPTCHA farms, ML)

---

### 6. Geo-Blocking

Block traffic from countries where you don't have users.

```
If user_country not in ["US", "CA", "UK", "DE"]:
    return 403  # Forbidden
```

**Pros:**

- Simple
- Effective if attacks come from specific regions

**Cons:**

- Blocks legitimate users traveling abroad
- Attackers use VPNs to bypass

---

### 7. SYN Cookies (SYN Flood Protection)

Defend against SYN flood attacks without maintaining connection state.

```
Normal TCP handshake:
  Client → SYN → Server (allocates memory for connection)
  Server → SYN-ACK → Client
  Client → ACK → Server (connection established)

SYN Flood:
  Attacker sends millions of SYN packets
  → Server allocates memory for each
  → Memory exhausted

SYN Cookies:
  Server doesn't allocate memory on SYN
  → Encodes connection info in SYN-ACK sequence number
  → Only allocates memory when ACK is received
  → Attacker can't complete handshake → no memory allocated
```

**Pros:**

- Effective against SYN floods
- No memory exhaustion

**Cons:**

- Slight performance overhead
- Some TCP options lost

---

### 8. Web Application Firewall (WAF)

Filter malicious HTTP requests at the application layer.

```
Rules:
  - Block SQL injection patterns
  - Block XSS patterns
  - Block requests with suspicious user agents
  - Block requests with missing headers
  - Rate limit by URL path
```

**Example: AWS WAF rule**

```json
{
  "Name": "BlockSQLInjection",
  "Priority": 1,
  "Statement": {
    "SqliMatchStatement": {
      "FieldToMatch": {
        "QueryString": {}
      }
    }
  },
  "Action": {
    "Block": {}
  }
}
```

**Pros:**

- Protects against application-layer attacks
- Customizable rules

**Cons:**

- False positives (blocks legitimate requests)
- Requires tuning

---

## 4. DDoS Protection Services

### Cloudflare

```
- Global Anycast network (300+ PoPs)
- Automatic DDoS mitigation (up to Tbps-scale)
- WAF, rate limiting, bot detection
- Free tier available
```

---

### AWS Shield

```
AWS Shield Standard:
  - Free, automatic protection for all AWS customers
  - Protects against common Layer 3/4 attacks

AWS Shield Advanced:
  - $3,000/month
  - Enhanced protection, DDoS response team
  - Cost protection (refund for scaling costs during attack)
```

---

### Akamai Prolexic

```
- Enterprise DDoS protection
- Scrubbing centers (filter attack traffic)
- 24/7 SOC (Security Operations Center)
```

---

### Google Cloud Armor

```
- DDoS protection for Google Cloud
- WAF, rate limiting, geo-blocking
- Integrates with Google Cloud Load Balancer
```

---

## 5. Monitoring & Detection

### Metrics to Monitor

```
Traffic volume:
  - Requests per second (RPS)
  - Bandwidth (Gbps)
  - Packets per second (PPS)

Error rates:
  - 5xx errors (server overload)
  - Timeouts
  - Connection failures

Latency:
  - Response time (p50, p99)
  - Time to first byte (TTFB)

Resource utilization:
  - CPU, memory, network
  - Connection pool usage
```

---

### Anomaly Detection

```
Baseline:  Normal traffic = 10,000 RPS
Alert:     Traffic > 50,000 RPS (5x baseline)

Baseline:  Normal error rate = 0.1%
Alert:     Error rate > 5% (50x baseline)
```

**Use machine learning** to detect anomalies (sudden spikes, unusual traffic patterns).

---

### Logging & Forensics

```
Log all requests during attacks:
  - IP address
  - User agent
  - Request path
  - Timestamp
  - Geo-location

Analyze logs to identify attack patterns:
  - Common IPs
  - Common user agents
  - Common request paths
```

---

## 6. Incident Response

### 1. Detect

Monitor metrics, set up alerts for anomalies.

---

### 2. Triage

Determine attack type (volumetric, protocol, application layer).

---

### 3. Mitigate

Apply appropriate defenses:

```
Volumetric:  Enable Anycast, contact ISP for upstream filtering
Protocol:    Enable SYN cookies, increase connection limits
Application: Enable rate limiting, WAF rules, CAPTCHAs
```

---

### 4. Communicate

Notify users of service degradation, provide status updates.

---

### 5. Post-Mortem

Analyze attack, improve defenses, update runbooks.

---

## 7. Common Interview Questions + Answers

### Q: How do you defend against a Layer 7 DDoS attack?

> "Layer 7 attacks target the application with seemingly legitimate requests. I'd use a combination of rate limiting, WAF rules, and CAPTCHAs. Rate limiting restricts requests per IP and per user. WAF rules block suspicious patterns like missing headers or unusual user agents. For persistent attacks, I'd enable CAPTCHAs to challenge suspicious traffic. I'd also monitor application metrics like error rates and latency to detect attacks early. If the attack is large, I'd use a DDoS protection service like Cloudflare to absorb traffic at the edge."

---

### Q: What's the difference between volumetric and application-layer DDoS attacks?

> "Volumetric attacks flood the network with massive traffic to saturate bandwidth — think UDP floods or DNS amplification. They're measured in Gbps or Tbps. Application-layer attacks target the application with requests that look legitimate but are expensive to process — like hitting a search endpoint repeatedly. They're measured in requests per second. Volumetric attacks are easier to detect (huge traffic spikes) but require massive capacity to absorb. Application-layer attacks are harder to detect (requests look normal) and require smarter filtering like rate limiting and WAF rules."

---

### Q: How does Anycast help with DDoS protection?

> "Anycast routes traffic to the nearest edge location by announcing the same IP from multiple locations. During a DDoS attack, traffic is distributed across many edge PoPs instead of hitting a single origin server. Each edge filters malicious traffic and forwards only legitimate requests to the origin. This dilutes the attack — if 1 Tbps of attack traffic is distributed across 100 edges, each edge only sees 10 Gbps. The origin server is protected and only sees clean traffic. Services like Cloudflare and AWS Shield use Anycast for Tbps-scale protection."

---

### Q: How do you distinguish DDoS traffic from legitimate traffic spikes?

> "It's challenging. I'd look at several signals: traffic patterns (DDoS often has uniform request rates, legitimate spikes are more organic), geographic distribution (DDoS may come from unusual regions), user agents (DDoS often uses generic or missing user agents), and behavior (DDoS hits the same endpoints repeatedly, legitimate users browse naturally). I'd also check if the traffic correlates with known events like product launches or marketing campaigns. When in doubt, I'd use CAPTCHAs to challenge suspicious traffic without blocking legitimate users."

---

## 8. Interview Tricks & Pitfalls

### ✅ Trick 1: Know the three attack types

Volumetric (Layer 3/4), protocol (Layer 4), and application-layer (Layer 7). Know examples and defenses for each.

---

### ✅ Trick 2: Mention Anycast for large-scale attacks

For Tbps-scale attacks, Anycast is the only practical defense. Shows you understand real-world DDoS protection.

---

### ✅ Trick 3: Combine multiple defenses

No single defense is perfect. Mention defense in depth: rate limiting + WAF + CAPTCHAs + Anycast.

---

### ❌ Pitfall 1: Blocking all traffic

Don't suggest blocking all traffic during an attack. The goal is to filter malicious traffic while allowing legitimate users through.

---

### ❌ Pitfall 2: Relying only on rate limiting

Rate limiting by IP doesn't work for distributed attacks (many IPs). Need additional defenses like WAF, CAPTCHAs, Anycast.

---

### ❌ Pitfall 3: Ignoring application-layer attacks

Many candidates focus on volumetric attacks. Application-layer attacks are harder to defend and more common. Show you understand both.

---

## 9. Quick Reference

```
DDoS: Distributed Denial of Service (overwhelm system with traffic)

Attack Types:
  Volumetric (Layer 3/4):  Saturate bandwidth (UDP flood, DNS amplification)
  Protocol (Layer 4):      Exhaust resources (SYN flood, connection exhaustion)
  Application (Layer 7):   Expensive requests (HTTP flood, Slowloris, API abuse)

Defenses:
  Over-provisioning:  Scale capacity to absorb attack
  Rate limiting:      Limit requests per IP/user
  Anycast:            Distribute traffic across edge PoPs
  IP reputation:      Block known malicious IPs
  CAPTCHAs:           Challenge suspicious traffic
  Geo-blocking:       Block traffic from specific countries
  SYN cookies:        Defend against SYN floods
  WAF:                Filter malicious HTTP requests

DDoS Protection Services:
  Cloudflare:         Global Anycast, automatic mitigation, free tier
  AWS Shield:         Standard (free), Advanced ($3k/month)
  Akamai Prolexic:    Enterprise, scrubbing centers
  Google Cloud Armor: GCP DDoS protection + WAF

Monitoring:
  Traffic volume:  RPS, bandwidth, PPS
  Error rates:     5xx errors, timeouts
  Latency:         p50, p99, TTFB
  Resources:       CPU, memory, connections

Incident Response:
  1. Detect:    Monitor metrics, set alerts
  2. Triage:    Identify attack type
  3. Mitigate:  Apply defenses (rate limiting, WAF, Anycast)
  4. Communicate: Notify users
  5. Post-mortem: Analyze, improve

Key Concepts:
  Anycast:        Same IP announced from multiple locations
  SYN cookies:    Stateless SYN flood defense
  Amplification:  Attacker uses third party to amplify attack (DNS, NTP)
  Botnet:         Network of compromised machines used for DDoS
```
