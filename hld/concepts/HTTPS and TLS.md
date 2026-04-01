## 1. What Is HTTPS?

HTTPS (HTTP Secure) is HTTP over TLS/SSL. It encrypts data in transit between client and server.

```
HTTP (unencrypted):
  Client → "username=alice&password=secret123" → Server
  Anyone on network can read this

HTTPS (encrypted):
  Client → [encrypted data] → Server
  Only client and server can decrypt
```

**TLS (Transport Layer Security)** is the protocol that provides encryption. SSL (Secure Sockets Layer) is the older, deprecated version.

---

## 2. Why HTTPS?

### 1. Confidentiality

```
Encrypt data in transit
Prevents eavesdropping
Passwords, credit cards, personal data protected
```

---

### 2. Integrity

```
Detect tampering
If data modified in transit → detected
Prevents man-in-the-middle attacks
```

---

### 3. Authentication

```
Verify server identity
Certificate proves server is who it claims to be
Prevents impersonation
```

---

## 3. TLS Handshake

How client and server establish encrypted connection.

### TLS 1.2 Handshake

```
1. Client Hello
   Client → Server
   - Supported TLS versions
   - Supported cipher suites
   - Random number (client random)

2. Server Hello
   Server → Client
   - Chosen TLS version
   - Chosen cipher suite
   - Random number (server random)
   - Server certificate (contains public key)

3. Client verifies certificate
   - Check certificate is signed by trusted CA
   - Check certificate is valid (not expired)
   - Check certificate matches domain

4. Client Key Exchange
   Client → Server
   - Generate pre-master secret
   - Encrypt with server's public key
   - Send to server

5. Both sides derive session keys
   - Client and server independently compute:
     session_key = function(client_random, server_random, pre_master_secret)

6. Finished messages
   Client → Server: "Finished" (encrypted with session key)
   Server → Client: "Finished" (encrypted with session key)

7. Application data
   All subsequent data encrypted with session key
```

**Total: 2 round trips (4 messages) before application data.**

---

### TLS 1.3 Handshake (faster)

```
1. Client Hello
   Client → Server
   - Supported TLS versions
   - Supported cipher suites
   - Key share (Diffie-Hellman public key)

2. Server Hello + Application Data
   Server → Client
   - Chosen cipher suite
   - Key share (Diffie-Hellman public key)
   - Server certificate
   - Finished message
   - Can immediately send application data

3. Client Finished + Application Data
   Client → Server
   - Finished message
   - Can immediately send application data

Total: 1 round trip (2 messages) before application data
```

**TLS 1.3 is faster and more secure.**

---

## 4. Certificates

### What is a certificate?

```
Digital document that proves identity
Contains:
  - Domain name (example.com)
  - Public key
  - Issuer (Certificate Authority)
  - Validity period (not before, not after)
  - Signature (CA's signature)
```

---

### Certificate chain

```
Root CA (trusted by browsers)
  ↓ signs
Intermediate CA
  ↓ signs
Server Certificate (example.com)

Browser trusts Root CA
→ trusts Intermediate CA (signed by Root)
→ trusts Server Certificate (signed by Intermediate)
```

**Why intermediate CA?** Root CA private key kept offline for security. Intermediate CA used for day-to-day signing.

---

### Certificate validation

```
Browser checks:
  1. Certificate signed by trusted CA
  2. Certificate not expired
  3. Certificate domain matches requested domain
  4. Certificate not revoked (CRL or OCSP)

If all checks pass → trust the server
If any check fails → show warning
```

---

## 5. Certificate Authorities (CAs)

### Trusted CAs

```
Pre-installed in browsers/OS:
  - DigiCert
  - Let's Encrypt
  - GlobalSign
  - Comodo
  - GeoTrust

Browsers trust ~100-200 root CAs
```

---

### Let's Encrypt (free, automated)

```
Automated certificate issuance:
  1. Install certbot on server
  2. Run: certbot --nginx -d example.com
  3. Certbot proves you control domain (HTTP challenge)
  4. Let's Encrypt issues certificate
  5. Certbot installs certificate
  6. Auto-renewal every 90 days

Free, widely trusted, automated
```

---

### Self-signed certificates

```
Generate your own certificate (not signed by CA)

Use for:
  - Development/testing
  - Internal services

Don't use for:
  - Public websites (browsers show warning)
  - Production (users won't trust)
```

---

## 6. Cipher Suites

Combination of algorithms for encryption, authentication, and key exchange.

### Format

```
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
 ↑    ↑     ↑        ↑       ↑    ↑
 |    |     |        |       |    Hash (SHA256)
 |    |     |        |       Mode (GCM)
 |    |     |        Encryption (AES 128-bit)
 |    |     Authentication (RSA)
 |    Key Exchange (ECDHE - Elliptic Curve Diffie-Hellman Ephemeral)
 Protocol (TLS)
```

---

### Modern cipher suites (TLS 1.3)

```
TLS_AES_128_GCM_SHA256
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256

Simpler, more secure
Forward secrecy by default
```

---

### Forward secrecy

```
Even if server's private key is compromised later,
past sessions can't be decrypted

Achieved with ephemeral key exchange (DHE, ECDHE)
Each session uses unique keys
Keys discarded after session
```

---

## 7. HTTPS Best Practices

### 1. Use TLS 1.2 or 1.3 only

```
Disable:
  - SSL 2.0 (broken)
  - SSL 3.0 (broken)
  - TLS 1.0 (weak)
  - TLS 1.1 (weak)

Enable:
  - TLS 1.2 (minimum)
  - TLS 1.3 (preferred)
```

---

### 2. Use strong cipher suites

```
Prefer:
  - ECDHE (forward secrecy)
  - AES-GCM (authenticated encryption)
  - SHA256 or SHA384 (strong hash)

Avoid:
  - RC4 (broken)
  - 3DES (weak)
  - MD5 (broken)
  - Export ciphers (weak)
```

---

### 3. HSTS (HTTP Strict Transport Security)

```
Response header:
  Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

Tells browser:
  - Always use HTTPS for this domain
  - For 1 year (31536000 seconds)
  - Include all subdomains
  - Add to browser's preload list

Prevents:
  - Downgrade attacks (force HTTP)
  - SSL stripping
```

---

### 4. Certificate pinning

```
App hardcodes expected certificate or public key

If server presents different certificate → reject

Prevents:
  - Compromised CA issuing fake certificate
  - Man-in-the-middle with rogue certificate

Use for:
  - Mobile apps
  - High-security applications

Tradeoff:
  - Hard to update (app update required)
  - Breaks if certificate rotated
```

---

### 5. OCSP stapling

```
Server fetches certificate revocation status
Includes it in TLS handshake

Benefits:
  - Faster (client doesn't query OCSP server)
  - Privacy (client doesn't reveal which sites they visit)
  - Reliability (doesn't depend on OCSP server availability)
```

---

## 8. SSL/TLS Termination

### At load balancer

```
Client ──HTTPS──▶ Load Balancer ──HTTP──▶ Backend Servers
                   (decrypt)              (plaintext)
```

**Pros:**
- Backend servers don't need certificates
- Reduced compute on backend (no encryption)
- Centralized certificate management

**Cons:**
- Traffic between LB and backend is unencrypted
- LB is single point of failure for security

**Use when:** Backend is on trusted internal network.

---

### End-to-end encryption

```
Client ──HTTPS──▶ Load Balancer ──HTTPS──▶ Backend Servers
                   (decrypt + re-encrypt)   (decrypt)
```

**Pros:**
- Traffic encrypted all the way
- Defense in depth

**Cons:**
- More compute overhead
- Certificate management on all servers

**Use when:** Compliance requires end-to-end encryption.

---

## 9. Common Attacks and Mitigations

### Man-in-the-Middle (MITM)

```
Attack:
  Client ──▶ Attacker ──▶ Server
  Attacker intercepts and reads/modifies traffic

Mitigation:
  - HTTPS (attacker can't decrypt)
  - Certificate validation (detect fake certificates)
  - HSTS (prevent downgrade to HTTP)
  - Certificate pinning (reject unexpected certificates)
```

---

### SSL Stripping

```
Attack:
  Client requests http://example.com
  Attacker intercepts, proxies to https://example.com
  Client ← HTTP ← Attacker ← HTTPS ← Server
  Client thinks it's HTTP, attacker reads traffic

Mitigation:
  - HSTS (browser always uses HTTPS)
  - HTTPS redirects (server redirects HTTP to HTTPS)
  - HSTS preload list (browser knows to use HTTPS before first visit)
```

---

### Downgrade attacks

```
Attack:
  Attacker forces use of weak TLS version or cipher

Mitigation:
  - Disable weak TLS versions (SSL 3.0, TLS 1.0, TLS 1.1)
  - Disable weak ciphers
  - TLS 1.3 (prevents downgrade by design)
```

---

## 10. Performance Optimization

### 1. TLS session resumption

```
First connection: Full handshake (expensive)
Subsequent connections: Resume session (fast)

Session ID:
  Server stores session state
  Client sends session ID
  Server resumes session (no full handshake)

Session tickets:
  Server encrypts session state, sends to client
  Client sends ticket back
  Server decrypts, resumes session
  No server-side storage needed
```

---

### 2. HTTP/2

```
HTTPS + HTTP/2:
  - Multiplexing (multiple requests over one connection)
  - Header compression
  - Server push

Faster than HTTP/1.1
Requires HTTPS (browsers only support HTTP/2 over HTTPS)
```

---

### 3. OCSP stapling

```
Reduces handshake time
Client doesn't need to query OCSP server
```

---

### 4. TLS 1.3

```
Faster handshake (1-RTT vs 2-RTT)
0-RTT resumption (even faster for repeat connections)
Stronger security by default
```

---

## 11. Common Interview Questions + Answers

### Q: How does HTTPS work?

> "HTTPS is HTTP over TLS, which encrypts data in transit. When a client connects, it performs a TLS handshake with the server. The client sends supported cipher suites, the server chooses one and sends its certificate containing its public key. The client verifies the certificate is signed by a trusted CA and matches the domain. They then perform a key exchange (typically Diffie-Hellman) to establish a shared session key. All subsequent data is encrypted with this session key. This provides confidentiality (encryption), integrity (tampering detection), and authentication (certificate proves server identity)."

### Q: What's the difference between TLS 1.2 and TLS 1.3?

> "TLS 1.3 is faster and more secure. The handshake is 1 round trip instead of 2, reducing latency. It removes weak cipher suites and algorithms, supporting only modern, secure options. Forward secrecy is mandatory (ephemeral key exchange). It supports 0-RTT resumption for even faster repeat connections. And it encrypts more of the handshake, improving privacy. TLS 1.2 is still widely used and secure if configured properly, but TLS 1.3 is the preferred version."

### Q: What is certificate pinning and when would you use it?

> "Certificate pinning is when an application hardcodes the expected certificate or public key for a server. If the server presents a different certificate, the connection is rejected. This prevents man-in-the-middle attacks even if a CA is compromised and issues a fake certificate. I'd use it for high-security mobile apps or APIs where we control both client and server. The tradeoff is operational complexity — if you rotate certificates, you must update the app. For most web applications, proper certificate validation without pinning is sufficient."

### Q: How do you terminate SSL/TLS in a load-balanced environment?

> "I'd terminate TLS at the load balancer. The load balancer decrypts HTTPS traffic and forwards HTTP to backend servers over the internal network. This simplifies certificate management (one place), reduces compute on backend servers, and allows the load balancer to inspect traffic for routing decisions. If compliance requires end-to-end encryption, I'd re-encrypt between the load balancer and backends, but this adds overhead. For most applications, terminating at the load balancer with a trusted internal network is sufficient."

---

## 12. Interview Tricks & Pitfalls

### ✅ Trick 1: Mention the handshake

When discussing HTTPS:

> "The TLS handshake establishes the encrypted connection. The client and server negotiate a cipher suite, exchange keys, and verify the server's certificate. In TLS 1.3, this takes 1 round trip. In TLS 1.2, it takes 2 round trips."

### ✅ Trick 2: Discuss certificate validation

Show you understand the trust model:

> "The browser validates the certificate by checking it's signed by a trusted CA, not expired, matches the domain, and not revoked. The certificate chain goes from the server certificate to an intermediate CA to a root CA that's pre-installed in the browser."

### ✅ Trick 3: Know the performance optimizations

HTTPS doesn't have to be slow:

> "To optimize HTTPS performance, I'd use TLS 1.3 for faster handshakes, enable session resumption so repeat connections don't need a full handshake, use HTTP/2 for multiplexing, and enable OCSP stapling to reduce handshake time."

### ❌ Pitfall 1: Confusing SSL and TLS

SSL is deprecated:

> "TLS is the modern protocol. SSL 2.0 and 3.0 are broken and should be disabled. When people say 'SSL,' they usually mean TLS. The current versions are TLS 1.2 and TLS 1.3."

### ❌ Pitfall 2: Not mentioning HSTS

HSTS is critical for security:

> "I'd enable HSTS with a long max-age and includeSubDomains. This tells browsers to always use HTTPS, preventing downgrade attacks and SSL stripping. I'd also add the domain to the HSTS preload list."

### ❌ Pitfall 3: Forgetting about Let's Encrypt

Show you know modern certificate management:

> "For certificates, I'd use Let's Encrypt. It's free, automated, and widely trusted. Certbot handles issuance and auto-renewal every 90 days. This eliminates the operational overhead of manual certificate management."

---

## 13. Quick Reference

```
HTTPS = HTTP over TLS (Transport Layer Security)

Benefits:
  - Confidentiality (encryption)
  - Integrity (tampering detection)
  - Authentication (certificate proves identity)

TLS Handshake:
  TLS 1.2: 2 round trips (4 messages)
  TLS 1.3: 1 round trip (2 messages, faster)

Steps:
  1. Client Hello (supported ciphers)
  2. Server Hello (chosen cipher, certificate)
  3. Client verifies certificate
  4. Key exchange (establish session key)
  5. Application data (encrypted)

Certificates:
  - Issued by Certificate Authority (CA)
  - Contains domain, public key, validity period
  - Chain: Root CA → Intermediate CA → Server Certificate
  - Validation: signed by trusted CA, not expired, matches domain, not revoked

Certificate Authorities:
  - Let's Encrypt (free, automated, 90-day validity)
  - DigiCert, GlobalSign, Comodo (paid)
  - Self-signed (development only)

Cipher suites:
  Modern: TLS_AES_128_GCM_SHA256 (TLS 1.3)
  Legacy: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (TLS 1.2)
  Components: Key exchange, authentication, encryption, hash

Best practices:
  ✅ TLS 1.2 or 1.3 only (disable SSL, TLS 1.0, TLS 1.1)
  ✅ Strong cipher suites (ECDHE, AES-GCM, SHA256)
  ✅ HSTS (force HTTPS, prevent downgrade)
  ✅ Certificate pinning (mobile apps, high security)
  ✅ OCSP stapling (faster, more private)

SSL/TLS termination:
  At load balancer:  Decrypt at LB, HTTP to backend (common)
  End-to-end:        HTTPS all the way (compliance)

Attacks and mitigations:
  MITM:        HTTPS, certificate validation, HSTS
  SSL stripping: HSTS, HTTPS redirects, preload list
  Downgrade:   Disable weak versions/ciphers, TLS 1.3

Performance:
  - Session resumption (avoid full handshake)
  - HTTP/2 (multiplexing, requires HTTPS)
  - OCSP stapling (faster handshake)
  - TLS 1.3 (1-RTT handshake, 0-RTT resumption)

Key insight: HTTPS encrypts data in transit, preventing eavesdropping and tampering
```
