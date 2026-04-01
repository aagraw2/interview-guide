## 1. What is SSO?

**Single Sign-On (SSO)** lets users authenticate once and access multiple applications without re-entering credentials.

```
Without SSO:
  User logs into Gmail     → username + password
  User logs into Drive     → username + password again
  User logs into Calendar  → username + password again

With SSO:
  User logs into Identity Provider (IdP) once
  → Automatically logged into Gmail, Drive, Calendar, etc.
```

SSO improves user experience and centralizes authentication, making it easier to enforce security policies (MFA, password rotation, account lockout).

---

## 2. SSO Architecture

```
┌─────────┐
│  User   │
└────┬────┘
     │
     │ 1. Access app
     ▼
┌─────────────────┐
│  Application    │ (Service Provider - SP)
│  (Gmail, Slack) │
└────┬────────────┘
     │
     │ 2. Redirect to IdP (not authenticated)
     ▼
┌──────────────────┐
│ Identity Provider│ (IdP - Okta, Auth0, Azure AD)
│  (Okta, Auth0)   │
└────┬─────────────┘
     │
     │ 3. User logs in (if not already)
     │ 4. IdP issues token/assertion
     ▼
┌─────────────────┐
│  Application    │
│  (Gmail, Slack) │ 5. Validates token → user logged in
└─────────────────┘
```

**Key components:**

```
Identity Provider (IdP):  Centralized authentication service
                          Examples: Okta, Auth0, Azure AD, Google Workspace

Service Provider (SP):    The application the user wants to access
                          Examples: Salesforce, Slack, internal apps

Trust relationship:       SP trusts IdP to authenticate users
                          Established via metadata exchange or configuration
```

---

## 3. SSO Protocols

### SAML 2.0 (Security Assertion Markup Language)

XML-based protocol. Dominant in enterprise SSO.

```
1. User accesses app (SP)
2. SP generates SAML AuthnRequest (XML), redirects to IdP
3. User logs into IdP (if not already authenticated)
4. IdP generates SAML Response (XML assertion), redirects back to SP
5. SP validates assertion → user logged in
```

**SAML Assertion (simplified):**

```xml
<saml:Assertion>
  <saml:Subject>
    <saml:NameID>user@example.com</saml:NameID>
  </saml:Subject>
  <saml:Conditions NotBefore="..." NotOnOrAfter="..."/>
  <saml:AttributeStatement>
    <saml:Attribute Name="email">
      <saml:AttributeValue>user@example.com</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute Name="role">
      <saml:AttributeValue>admin</saml:AttributeValue>
    </saml:Attribute>
  </saml:AttributeStatement>
  <Signature>...</Signature>
</saml:Assertion>
```

**Pros:**

- Mature, widely supported in enterprise apps
- Strong security (XML signatures)
- Supports attribute mapping (roles, groups)

**Cons:**

- Complex (XML, signatures, metadata)
- Poor mobile support
- Verbose

---

### OpenID Connect (OIDC)

Modern, JSON-based protocol built on OAuth 2.0. Preferred for new systems.

```
1. User accesses app (SP)
2. SP redirects to IdP with OIDC authorization request
3. User logs into IdP (if not already authenticated)
4. IdP redirects back with authorization code
5. SP exchanges code for ID token (JWT)
6. SP validates ID token → user logged in
```

**ID Token (JWT):**

```json
{
  "iss": "https://idp.example.com",
  "sub": "user123",
  "aud": "app-client-id",
  "exp": 1672531200,
  "iat": 1672527600,
  "email": "user@example.com",
  "name": "John Doe",
  "groups": ["admin", "developers"]
}
```

**Pros:**

- Simple (JSON, JWTs)
- Excellent mobile support
- Built on OAuth 2.0 (familiar to developers)
- Growing adoption

**Cons:**

- Less mature than SAML in enterprise
- Some legacy apps don't support it

---

### CAS (Central Authentication Service)

Open-source SSO protocol. Less common than SAML/OIDC.

```
1. User accesses app
2. App redirects to CAS server
3. User logs in, CAS issues ticket
4. App validates ticket with CAS server
5. User logged in
```

Used in academic institutions and some legacy systems.

---

## 4. SSO Flows

### SP-Initiated Flow

User starts at the application (Service Provider).

```
1. User visits https://app.example.com
2. App detects user not authenticated
3. App redirects to IdP: https://idp.example.com/sso?SAMLRequest=...
4. User logs into IdP
5. IdP redirects back to app with assertion
6. App validates assertion → user logged in
```

**Most common flow.** User bookmarks the app URL and goes directly there.

---

### IdP-Initiated Flow

User starts at the Identity Provider (e.g., Okta dashboard).

```
1. User logs into IdP dashboard
2. User clicks "Salesforce" tile
3. IdP generates assertion, redirects to Salesforce
4. Salesforce validates assertion → user logged in
```

**Less common.** Convenient for users who access multiple apps from a central dashboard.

**Security concern:** IdP-initiated flow is more vulnerable to CSRF attacks. SP-initiated is preferred.

---

## 5. Session Management

### IdP Session

When a user logs into the IdP, the IdP creates a session (typically a cookie).

```
User logs into Okta
→ Okta sets session cookie (okta.com domain)
→ Session lasts 8 hours (configurable)
```

As long as the IdP session is active, the user can access any SP without re-authenticating.

---

### SP Session

Each application maintains its own session after SSO login.

```
User SSO into Salesforce
→ Salesforce creates session (salesforce.com domain)
→ Session lasts 2 hours (configurable)
```

**Key point:** IdP session and SP sessions are independent. If the IdP session expires, the user must re-authenticate at the IdP, but existing SP sessions remain active until they expire.

---

### Single Logout (SLO)

When a user logs out of one app, should they be logged out of all apps?

```
User logs out of Salesforce
→ Salesforce sends logout request to IdP
→ IdP sends logout requests to all active SPs
→ All sessions terminated
```

**Challenges:**

- Not all apps support SLO
- If user has multiple browser tabs open, logout may not propagate
- Complex to implement reliably

**Common approach:** Logout only terminates the IdP session. SP sessions remain active until they expire naturally. This is simpler and more reliable.

---

## 6. Just-In-Time (JIT) Provisioning

Automatically create user accounts in the SP when they first log in via SSO.

```
1. User logs into IdP
2. User accesses Salesforce for the first time
3. Salesforce receives SAML assertion with user attributes
4. Salesforce checks: user doesn't exist
5. Salesforce creates user account using attributes from assertion
   - Email: user@example.com
   - Name: John Doe
   - Role: admin (from SAML attribute)
6. User logged in
```

**Benefits:**

- No manual user provisioning
- Users get access immediately
- Attributes (roles, groups) synced from IdP

**Alternative:** SCIM (System for Cross-domain Identity Management) for automated provisioning and deprovisioning.

---

## 7. Multi-Factor Authentication (MFA)

SSO centralizes authentication, making it easier to enforce MFA.

```
User logs into IdP
→ Enters username + password
→ IdP requires MFA (SMS, TOTP, push notification)
→ User completes MFA
→ IdP issues assertion
→ User accesses all apps without MFA again (until IdP session expires)
```

**MFA is enforced once at the IdP** — all downstream apps benefit without implementing MFA themselves.

---

## 8. SSO in Microservices

In a microservices architecture, SSO is typically implemented at the API Gateway.

```
┌──────┐
│ User │
└──┬───┘
   │
   │ 1. Access app
   ▼
┌────────────────┐
│  API Gateway   │
└──┬─────────────┘
   │
   │ 2. Redirect to IdP (if not authenticated)
   │ 3. Validate token from IdP
   │ 4. Forward request with user context
   ▼
┌────────────────┐
│ Microservices  │
│ (Service A, B) │
└────────────────┘
```

**Pattern:**

1. API Gateway handles SSO (OIDC or SAML)
2. Gateway validates tokens from IdP
3. Gateway adds user context to requests (e.g., `X-User-ID`, `X-User-Email` headers)
4. Microservices trust the gateway and use the user context

**Alternative:** Each microservice validates tokens independently (decentralized). More resilient but duplicates logic.

---

## 9. SSO Security Considerations

### Validate assertions/tokens

Always verify:

- Signature (SAML) or JWT signature (OIDC)
- Issuer (iss claim)
- Audience (aud claim)
- Expiry (exp claim)
- Replay protection (SAML: AssertionID, OIDC: nonce)

---

### Use HTTPS everywhere

SSO tokens in transit must be encrypted. Never use SSO over HTTP.

---

### Short-lived assertions

SAML assertions and OIDC ID tokens should expire quickly (5-15 minutes). Reduces the window of vulnerability if a token is stolen.

---

### Protect against replay attacks

**SAML:** Track AssertionID to ensure each assertion is used only once.

**OIDC:** Use nonce parameter to prevent token replay.

---

### Limit session duration

IdP sessions should expire after a reasonable period (e.g., 8 hours). Require re-authentication for sensitive operations.

---

## 10. SSO vs Federation

**SSO:** Single login across multiple apps within the same organization.

```
Example: Employee logs into Okta, accesses Salesforce, Slack, Jira
         All apps belong to the same company
```

**Federation:** Single login across multiple organizations.

```
Example: University student logs into their university IdP,
         accesses research portal hosted by another university
         Trust relationship between two organizations
```

Federation is SSO across organizational boundaries. SAML is commonly used for federation.

---

## 11. Common SSO Providers

### Enterprise IdPs

```
Okta:        Leading enterprise IdP, supports SAML, OIDC, SCIM
Azure AD:    Microsoft's IdP, integrated with Office 365
Ping Identity: Enterprise SSO, strong in financial services
OneLogin:    Cloud-based IdP, supports SAML, OIDC
```

### Developer-Focused IdPs

```
Auth0:       Developer-friendly, supports OIDC, SAML, social login
Keycloak:    Open-source, self-hosted, supports OIDC, SAML
Google Workspace: SSO for Google apps, supports SAML, OIDC
```

---

## 12. Common Interview Questions + Answers

### Q: What's the difference between SSO and OAuth 2.0?

> "SSO is about authentication — logging into multiple apps with one set of credentials. OAuth 2.0 is about authorization — granting an app access to resources without sharing credentials. SSO uses protocols like SAML or OIDC. OIDC is built on OAuth 2.0, so there's overlap, but the goals are different. SSO is 'who are you?' and OAuth is 'what can you access?'"

---

### Q: How does SSO work in a microservices architecture?

> "Typically, the API Gateway handles SSO. When a user accesses the system, the gateway redirects to the IdP for authentication. The IdP returns a token (SAML assertion or OIDC ID token). The gateway validates the token and forwards requests to microservices with user context in headers like X-User-ID. Microservices trust the gateway and don't need to validate tokens themselves. Alternatively, each microservice can validate tokens independently for better resilience, but that duplicates logic."

---

### Q: What's the difference between SP-initiated and IdP-initiated SSO?

> "SP-initiated: the user starts at the application, which redirects to the IdP for authentication. This is more secure because the SP generates a request with a unique ID, preventing CSRF attacks. IdP-initiated: the user starts at the IdP dashboard and clicks an app tile. The IdP generates an assertion and redirects to the app. This is more convenient but less secure — it's vulnerable to CSRF. SP-initiated is the recommended flow."

---

### Q: How do you handle logout in SSO?

> "There are two approaches. Single Logout (SLO): when a user logs out of one app, the IdP sends logout requests to all apps the user accessed, terminating all sessions. This is complex and not all apps support it. The simpler approach: logout only terminates the IdP session. Individual app sessions remain active until they expire naturally. This is more reliable and commonly used in practice."

---

## 13. Interview Tricks & Pitfalls

### ✅ Trick 1: Know SAML vs OIDC trade-offs

SAML is dominant in enterprise but complex. OIDC is modern and simpler. If designing a new system, recommend OIDC. If integrating with enterprise apps, expect SAML.

---

### ✅ Trick 2: Mention JIT provisioning

When discussing SSO, mention Just-In-Time provisioning to show you understand the full user lifecycle, not just authentication.

---

### ✅ Trick 3: Distinguish SSO from OAuth

Interviewers often conflate these. Clearly state: SSO is authentication, OAuth is authorization. OIDC bridges both.

---

### ❌ Pitfall 1: Assuming SSO means no passwords

SSO doesn't eliminate passwords — users still authenticate at the IdP with a password (or MFA). SSO just means they authenticate once, not per app.

---

### ❌ Pitfall 2: Not validating tokens

Always validate SAML assertions and OIDC ID tokens. Verify signature, issuer, audience, expiry. Don't just decode and trust.

---

### ❌ Pitfall 3: Ignoring session management

SSO creates sessions at both the IdP and each SP. These are independent. Understand how session expiry and logout work across both layers.

---

## 14. Quick Reference

```
SSO: Single login across multiple apps

Protocols:
  SAML 2.0:  XML-based, enterprise standard, complex
  OIDC:      JSON/JWT-based, modern, simpler
  CAS:       Open-source, less common

Flows:
  SP-initiated:  User starts at app → redirects to IdP (more secure)
  IdP-initiated: User starts at IdP dashboard → clicks app tile

Components:
  IdP (Identity Provider):  Centralized authentication (Okta, Auth0)
  SP (Service Provider):    Application (Salesforce, Slack)

Sessions:
  IdP session:  Created when user logs into IdP
  SP session:   Created when user accesses each app
  Independent:  IdP session expiry doesn't terminate SP sessions

Single Logout (SLO):
  Logout from one app → IdP notifies all apps → all sessions terminated
  Complex, not always supported
  Alternative: Logout only terminates IdP session

JIT Provisioning:
  Automatically create user accounts in SP on first SSO login
  Uses attributes from SAML assertion or OIDC ID token

Security:
  ✅ Validate signatures, issuer, audience, expiry
  ✅ Use HTTPS everywhere
  ✅ Short-lived assertions (5-15 minutes)
  ✅ Replay protection (AssertionID, nonce)
  ✅ MFA at IdP (enforced once, benefits all apps)

SSO vs OAuth:
  SSO:   Authentication (who are you?)
  OAuth: Authorization (what can you access?)
  OIDC:  Both (authentication layer on OAuth 2.0)
```
