## 1. What is DNS?

DNS (Domain Name System) translates human-readable domain names (example.com) into IP addresses (93.184.216.34) that computers use to communicate.

```
User types: www.example.com
DNS resolves: 93.184.216.34
Browser connects to: 93.184.216.34
```

DNS is a distributed, hierarchical database that enables the internet to scale to billions of devices.

---

## 2. DNS Hierarchy

### Root, TLD, and authoritative servers

```
www.example.com
    │
    ├── .com (Top-Level Domain)
    │   └── example.com (Second-Level Domain)
    │       └── www.example.com (Subdomain)
    │
Root servers (.)
  ├── TLD servers (.com, .org, .net)
  │   └── Authoritative servers (example.com)
```

### DNS resolution flow

```
1. User: "What's the IP for www.example.com?"

2. Recursive resolver (ISP or 8.8.8.8):
   "Let me find out for you"

3. Root server:
   "I don't know, but ask the .com TLD server at 192.5.6.30"

4. TLD server (.com):
   "I don't know, but ask example.com's authoritative server at 199.43.135.53"

5. Authoritative server (example.com):
   "www.example.com is 93.184.216.34"

6. Recursive resolver:
   "Here's your answer: 93.184.216.34"
   (Also caches the result for future queries)
```

---

## 3. DNS Record Types

### A record (Address)

Maps domain to IPv4 address.

```
example.com.  A  93.184.216.34

Query: example.com
Answer: 93.184.216.34
```

### AAAA record

Maps domain to IPv6 address.

```
example.com.  AAAA  2606:2800:220:1:248:1893:25c8:1946

Query: example.com
Answer: 2606:2800:220:1:248:1893:25c8:1946
```

### CNAME record (Canonical Name)

Alias one domain to another.

```
www.example.com.  CNAME  example.com.
example.com.      A      93.184.216.34

Query: www.example.com
  → Resolves to example.com (CNAME)
  → Resolves to 93.184.216.34 (A record)
```

**Use case:** Point multiple subdomains to the same server without duplicating IP addresses.

### MX record (Mail Exchange)

Specifies mail servers for a domain.

```
example.com.  MX  10 mail1.example.com.
example.com.  MX  20 mail2.example.com.

Priority: Lower number = higher priority
  → Try mail1 first, fallback to mail2
```

### TXT record

Arbitrary text data. Used for verification, SPF, DKIM.

```
example.com.  TXT  "v=spf1 include:_spf.google.com ~all"

Use cases:
  - Domain verification (Google, AWS)
  - Email authentication (SPF, DKIM, DMARC)
  - Site ownership proof
```

### NS record (Name Server)

Specifies authoritative name servers for a domain.

```
example.com.  NS  ns1.example.com.
example.com.  NS  ns2.example.com.

Delegates DNS queries to these servers
```

### SOA record (Start of Authority)

Metadata about the DNS zone.

```
example.com.  SOA  ns1.example.com. admin.example.com. (
  2024010101  ; Serial number
  3600        ; Refresh (1 hour)
  600         ; Retry (10 minutes)
  86400       ; Expire (1 day)
  300         ; Minimum TTL (5 minutes)
)
```

---

## 4. TTL (Time To Live)

TTL specifies how long a DNS record can be cached.

```
example.com.  300  A  93.184.216.34
              ^^^
              TTL = 300 seconds (5 minutes)

Resolver caches this record for 5 minutes
After 5 minutes, resolver queries again for fresh data
```

### TTL trade-offs

```
Short TTL (e.g., 60 seconds):
  ✅ Changes propagate quickly
  ❌ More DNS queries (higher load, slower)

Long TTL (e.g., 86400 seconds = 1 day):
  ✅ Fewer DNS queries (lower load, faster)
  ❌ Changes take longer to propagate

Typical: 300-3600 seconds (5 minutes to 1 hour)
```

### TTL strategy for migrations

```
Before migration:
  1. Lower TTL to 60 seconds (1 week before)
  2. Wait for old TTL to expire

During migration:
  3. Update DNS record to new IP
  4. Changes propagate in 60 seconds

After migration:
  5. Raise TTL back to 3600 seconds
```

---

## 5. Recursive vs Authoritative Resolvers

### Recursive resolver

Performs the full DNS resolution on behalf of the client.

```
Examples:
  - ISP DNS (provided by your internet provider)
  - Google Public DNS (8.8.8.8, 8.8.4.4)
  - Cloudflare DNS (1.1.1.1, 1.0.0.1)
  - Quad9 (9.9.9.9)

Responsibilities:
  - Query root, TLD, and authoritative servers
  - Cache results
  - Return answer to client
```

### Authoritative resolver

Holds the actual DNS records for a domain.

```
Examples:
  - Route 53 (AWS)
  - Cloud DNS (Google)
  - Cloudflare DNS
  - Your own DNS server

Responsibilities:
  - Store DNS records (A, CNAME, MX, etc.)
  - Answer queries for your domain
  - No caching (always returns fresh data)
```

---

## 6. GeoDNS (Geographic Routing)

Route users to the nearest server based on their location.

```
User in US:
  Query: api.example.com
  Answer: 1.2.3.4 (US server)

User in Europe:
  Query: api.example.com
  Answer: 5.6.7.8 (EU server)

User in Asia:
  Query: api.example.com
  Answer: 9.10.11.12 (Asia server)
```

### How it works

```
DNS resolver detects client's location (by IP address)
Returns the IP of the closest server

Benefits:
  ✅ Lower latency (users connect to nearby servers)
  ✅ Better user experience
  ✅ Load distribution across regions
```

### Route 53 geolocation routing

```
Record 1: api.example.com → 1.2.3.4 (North America)
Record 2: api.example.com → 5.6.7.8 (Europe)
Record 3: api.example.com → 9.10.11.12 (Asia)
Record 4: api.example.com → 13.14.15.16 (Default)

Route 53 returns the appropriate IP based on client location
```

---

## 7. DNS Load Balancing

Return multiple IP addresses for a single domain.

```
example.com.  A  1.2.3.4
example.com.  A  5.6.7.8
example.com.  A  9.10.11.12

Client receives all 3 IPs
Client picks one (usually the first)
```

### Round-robin DNS

```
Query 1: Returns [1.2.3.4, 5.6.7.8, 9.10.11.12]
Query 2: Returns [5.6.7.8, 9.10.11.12, 1.2.3.4]
Query 3: Returns [9.10.11.12, 1.2.3.4, 5.6.7.8]

DNS server rotates the order
Distributes load across servers
```

### Limitations

```
❌ No health checks (returns IPs even if server is down)
❌ Uneven distribution (clients cache DNS, don't query every time)
❌ No session affinity (client may connect to different server on retry)

Better solution: Use a load balancer (L4/L7) instead of DNS load balancing
```

---

## 8. DNS Caching

Caching happens at multiple levels.

```
1. Browser cache (Chrome, Firefox)
   TTL: Typically 60 seconds

2. OS cache (Windows, macOS, Linux)
   TTL: Varies by OS

3. Recursive resolver cache (ISP, 8.8.8.8)
   TTL: Specified in DNS record

4. Authoritative server (no caching)
   Always returns fresh data
```

### Cache poisoning

Attacker injects fake DNS records into a resolver's cache.

```
Attack:
  1. Attacker sends fake DNS response: example.com → 6.6.6.6 (attacker's IP)
  2. Resolver caches the fake record
  3. Users are redirected to attacker's server

Mitigation:
  - DNSSEC (cryptographic signatures)
  - Randomize query IDs and source ports
  - Use trusted resolvers (8.8.8.8, 1.1.1.1)
```

---

## 9. DNSSEC (DNS Security Extensions)

Cryptographically sign DNS records to prevent tampering.

```
Without DNSSEC:
  Resolver: "What's example.com?"
  Attacker: "It's 6.6.6.6" (fake response)
  Resolver: "OK, I'll cache that"

With DNSSEC:
  Resolver: "What's example.com?"
  Attacker: "It's 6.6.6.6" (fake response, no signature)
  Resolver: "Invalid signature, ignoring"
  Authoritative: "It's 93.184.216.34" (signed)
  Resolver: "Valid signature, caching"
```

### How DNSSEC works

```
1. Authoritative server signs DNS records with private key
2. Public key is published in DNS (DNSKEY record)
3. Resolver verifies signature using public key
4. If signature is valid, record is trusted

Chain of trust:
  Root (.) signs .com
  .com signs example.com
  example.com signs www.example.com
```

---

## 10. DNS in System Design

### DNS for service discovery

```
Service: api.example.com
  → Resolves to load balancer IP
  → Load balancer routes to backend servers

Benefits:
  - Clients don't need to know backend IPs
  - Can change backend IPs without updating clients
  - Can use GeoDNS for multi-region routing
```

### DNS failover

```
Primary server: 1.2.3.4 (healthy)
Backup server: 5.6.7.8 (standby)

Route 53 health check:
  - Ping 1.2.3.4 every 30 seconds
  - If unhealthy, update DNS to 5.6.7.8
  - Clients automatically failover to backup

Limitation: TTL delay (clients may cache old IP for TTL duration)
```

### DNS for blue-green deployments

```
Blue environment: blue.example.com → 1.2.3.4
Green environment: green.example.com → 5.6.7.8

Production: www.example.com → CNAME blue.example.com

Deploy to green:
  1. Deploy new version to green environment
  2. Test green.example.com
  3. Update CNAME: www.example.com → green.example.com
  4. Traffic switches to green (after TTL expires)

Rollback:
  Update CNAME back to blue.example.com
```

---

## 11. DNS Performance

### Query latency

```
Typical DNS query latency:
  - Cached: <1ms (browser/OS cache)
  - Recursive resolver: 10-50ms (ISP or 8.8.8.8)
  - Full resolution: 100-200ms (root → TLD → authoritative)

Optimization:
  - Use low TTL only when needed (migrations)
  - Use fast recursive resolvers (1.1.1.1, 8.8.8.8)
  - Pre-resolve DNS in application (DNS prefetching)
```

### DNS prefetching

```html
<link rel="dns-prefetch" href="//api.example.com">
<link rel="dns-prefetch" href="//cdn.example.com">

Browser resolves DNS in the background
Reduces latency when user clicks a link
```

---

## 12. Common Interview Questions + Answers

### Q: How does DNS resolution work?

> "The client queries a recursive resolver, which queries the root server for the TLD server, then queries the TLD server for the authoritative server, then queries the authoritative server for the actual IP address. The recursive resolver caches the result and returns it to the client. Each level of the hierarchy delegates to the next level, creating a distributed system that scales globally."

### Q: What's the difference between a CNAME and an A record?

> "An A record maps a domain directly to an IP address. A CNAME is an alias that points one domain to another domain, which then resolves to an IP via an A record. Use A records for the root domain and CNAME for subdomains. You can't use a CNAME for the root domain because it conflicts with other required records like MX and SOA."

### Q: How would you implement DNS-based failover?

> "Use a DNS provider with health checks like Route 53. Configure health checks to ping your primary server every 30 seconds. If the health check fails, automatically update the DNS record to point to a backup server. The limitation is TTL — clients may cache the old IP for the TTL duration, so use a short TTL (60 seconds) for critical services. For faster failover, use a load balancer with health checks instead of DNS."

### Q: What's the purpose of TTL in DNS?

> "TTL specifies how long a DNS record can be cached by resolvers. Short TTL means changes propagate quickly but increases DNS query load. Long TTL reduces query load but delays propagation of changes. Typical TTL is 5 minutes to 1 hour. Before a migration, lower TTL to 60 seconds, then raise it back after the migration completes."

---

## 13. Interview Tricks & Pitfalls

### ✅ Trick 1: Explain the hierarchy

DNS is hierarchical: root → TLD → authoritative. Mentioning this shows you understand the distributed nature of DNS.

### ✅ Trick 2: Discuss TTL trade-offs

TTL is a common interview topic. Always mention the trade-off between propagation speed and query load.

### ✅ Trick 3: Know GeoDNS

GeoDNS is used for multi-region routing. Mentioning it shows you understand global system design.

### ❌ Pitfall 1: Confusing recursive and authoritative resolvers

Recursive resolvers query on behalf of clients. Authoritative resolvers hold the actual records. Don't mix them up.

### ❌ Pitfall 2: Thinking DNS is instant

DNS has caching at multiple levels (browser, OS, resolver). Changes don't propagate instantly — they depend on TTL.

### ❌ Pitfall 3: Using DNS for load balancing

DNS round-robin is not a good load balancer — no health checks, uneven distribution, no session affinity. Use a real load balancer instead.

---

## 14. Quick Reference

```
What is DNS?
  Translates domain names to IP addresses
  Distributed, hierarchical database
  Enables internet to scale to billions of devices

DNS hierarchy:
  Root (.) → TLD (.com) → Authoritative (example.com)

Resolution flow:
  Client → Recursive resolver → Root → TLD → Authoritative → Answer

Record types:
  A: Domain → IPv4 address
  AAAA: Domain → IPv6 address
  CNAME: Alias one domain to another
  MX: Mail servers (with priority)
  TXT: Arbitrary text (verification, SPF, DKIM)
  NS: Authoritative name servers
  SOA: Zone metadata

TTL (Time To Live):
  How long a record can be cached
  Short TTL: Fast propagation, more queries
  Long TTL: Slow propagation, fewer queries
  Typical: 300-3600 seconds (5 min to 1 hour)

Recursive vs Authoritative:
  Recursive: Queries on behalf of client (8.8.8.8, 1.1.1.1)
  Authoritative: Holds actual records (Route 53, Cloudflare)

GeoDNS:
  Route users to nearest server based on location
  Lower latency, better UX

DNS load balancing:
  Return multiple IPs for one domain
  Round-robin rotation
  Limitations: No health checks, uneven distribution

DNS caching:
  Browser → OS → Recursive resolver → Authoritative
  TTL controls cache duration

DNSSEC:
  Cryptographic signatures to prevent tampering
  Chain of trust: Root → TLD → Domain

Use cases:
  Service discovery (api.example.com → load balancer)
  Failover (health checks + DNS update)
  Blue-green deployments (CNAME switching)

Performance:
  Cached: <1ms
  Recursive resolver: 10-50ms
  Full resolution: 100-200ms
  Optimization: Low TTL only when needed, DNS prefetching
```
