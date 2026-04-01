## 1. Authentication vs Authorization

**Authentication (AuthN):** Who are you?

```
User provides credentials → System verifies identity
"Prove you are Alice"
```

**Authorization (AuthZ):** What can you do?

```
System checks permissions → Grants or denies access
"Alice can read orders, but not delete them"
```

**Example:**

```
Authentication: Login with username/password → "You are Alice"
Authorization: Try to delete order → Check if Alice has delete permission → Deny
```

---

## 2. Authentication Methods

### Username & Password

```
Client → POST /login {username: "alice", password: "secret123"}
Server → Verify credentials → Return session token
Client → Stores token → Sends with future requests
```

**Pros:** Simple, familiar. **Cons:** Passwords can be weak, stolen, or phished.

**Best practices:**
- Hash passwords (bcrypt, Argon2)
- Enforce strong password policy
- Rate limit login attempts
- Multi-factor authentication

---

### Session-Based Authentication

```
1. User logs in → Server creates session
2. Server stores session in memory/database
3. Server returns session ID in cookie
4. Client sends cookie with each request
5. Server validates session ID

Session store:
  session_id: "abc123"
  user_id: 456
  expires: 2024-01-15T12:00:00Z
```

**Pros:** Server controls sessions (can revoke). **Cons:** Doesn't scale horizontally (need shared session store).

---

### Token-Based Authentication (JWT)

```
1. User logs in → Server creates JWT
2. Server signs JWT with secret key
3. Server returns JWT to client
4. Client stores JWT (localStorage, cookie)
5. Client sends JWT in Authorization header
6. Server verifies JWT signature

JWT structure:
  Header:  {alg: "HS256", typ: "JWT"}
  Payload: {user_id: 456, role: "admin", exp: 1700000000}
  Signature: HMACSHA256(header + payload, secret)
```

**Pros:** Stateless (no server-side storage), scales horizontally. **Cons:** Can't revoke (until expiry), larger than session ID.

---

### OAuth 2.0

Delegate authentication to third party.

```
User → "Login with Google"
App → Redirects to Google
User → Logs in to Google
Google → Redirects back with authorization code
App → Exchanges code for access token
App → Uses token to access Google APIs
```

**Use cases:** "Login with Google/Facebook/GitHub", third-party API access.

**Roles:**
- Resource Owner: User
- Client: Your app
- Authorization Server: Google/Facebook
- Resource Server: Google/Facebook APIs

---

### Multi-Factor Authentication (MFA)

Require multiple forms of verification.

```
Factors:
  1. Something you know (password)
  2. Something you have (phone, hardware token)
  3. Something you are (fingerprint, face)

Example:
  1. User enters password
  2. Server sends code to phone
  3. User enters code
  4. Server grants access
```

**Types:**
- SMS code (least secure, but convenient)
- TOTP (Time-based One-Time Password, like Google Authenticator)
- Hardware token (YubiKey)
- Biometric (fingerprint, face ID)

---

### API Keys

Simple token for API access.

```
Client → GET /api/users
Headers: X-API-Key: <example-api-key-not-real>

Server → Validates API key → Returns data
```

**Pros:** Simple. **Cons:** No user context, hard to rotate, often long-lived.

**Use for:** Server-to-server communication, public APIs.

---

## 3. Authorization Models

### Role-Based Access Control (RBAC)

Users assigned roles, roles have permissions.

```
Roles:
  Admin:    can read, write, delete
  Editor:   can read, write
  Viewer:   can read

User Alice → Role: Editor → Permissions: read, write

Alice tries to delete → Check role → Editor can't delete → Deny
```

**Pros:** Simple, easy to understand. **Cons:** Role explosion (too many roles), inflexible.

---

### Attribute-Based Access Control (ABAC)

Access based on attributes (user, resource, environment).

```
Policy:
  Allow if:
    user.department == resource.department
    AND user.clearance_level >= resource.classification
    AND time.hour >= 9 AND time.hour <= 17

Example:
  Alice (department: Sales, clearance: 3)
  tries to access Document (department: Sales, classification: 2)
  at 10:00 AM
  → All conditions met → Allow
```

**Pros:** Flexible, fine-grained. **Cons:** Complex, harder to debug.

---

### Access Control Lists (ACL)

List of permissions per resource.

```
Document 123:
  Alice: read, write
  Bob: read
  Charlie: read, write, delete

Alice tries to delete Document 123 → Check ACL → Alice doesn't have delete → Deny
```

**Pros:** Fine-grained per resource. **Cons:** Hard to manage at scale.

---

### Policy-Based Access Control

Centralized policy engine evaluates access.

```
Policy (OPA - Open Policy Agent):
  allow {
    input.user.role == "admin"
  }
  
  allow {
    input.user.id == input.resource.owner_id
  }

Request:
  user: {id: 456, role: "editor"}
  resource: {id: 789, owner_id: 456}
  action: "delete"

Policy engine → Evaluates → user.id == resource.owner_id → Allow
```

**Pros:** Centralized, flexible, auditable. **Cons:** Additional component, learning curve.

---

## 4. JWT Deep Dive

### Structure

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjo0NTYsInJvbGUiOiJhZG1pbiIsImV4cCI6MTcwMDAwMDAwMH0.signature

Header (base64):
  {
    "alg": "HS256",
    "typ": "JWT"
  }

Payload (base64):
  {
    "user_id": 456,
    "role": "admin",
    "exp": 1700000000,
    "iat": 1699990000
  }

Signature:
  HMACSHA256(
    base64(header) + "." + base64(payload),
    secret_key
  )
```

---

### Validation

```python
def validate_jwt(token, secret):
    try:
        # Split token
        header, payload, signature = token.split('.')
        
        # Verify signature
        expected_signature = hmac_sha256(header + '.' + payload, secret)
        if signature != expected_signature:
            return None  # invalid signature
        
        # Decode payload
        payload_data = base64_decode(payload)
        
        # Check expiration
        if payload_data['exp'] < current_time():
            return None  # expired
        
        return payload_data
    except:
        return None  # invalid token
```

---

### Claims

```
Standard claims:
  iss (issuer):    Who created the token
  sub (subject):   User ID
  aud (audience):  Who the token is for
  exp (expiration): When token expires
  iat (issued at):  When token was created
  nbf (not before): Token not valid before this time

Custom claims:
  user_id: 456
  role: "admin"
  permissions: ["read", "write"]
```

---

### Refresh tokens

```
Access token:  Short-lived (15 minutes), used for API requests
Refresh token: Long-lived (7 days), used to get new access token

Flow:
  1. User logs in → Get access token + refresh token
  2. Use access token for API requests
  3. Access token expires → Use refresh token to get new access token
  4. Refresh token expires → User must log in again

Benefits:
  - Short-lived access tokens (less risk if stolen)
  - Long-lived refresh tokens (better UX, no frequent logins)
  - Can revoke refresh tokens (stored server-side)
```

---

## 5. OAuth 2.0 Flows

### Authorization Code Flow (most secure)

```
1. User → "Login with Google"
2. App → Redirect to Google with client_id
3. User → Logs in to Google
4. Google → Redirect to app with authorization code
5. App → Exchange code for access token (server-side, with client_secret)
6. Google → Return access token
7. App → Use access token to call Google APIs

Security: client_secret never exposed to browser
```

---

### Implicit Flow (deprecated)

```
1. User → "Login with Google"
2. App → Redirect to Google
3. User → Logs in
4. Google → Redirect with access token in URL fragment

Security: Token exposed in browser, no client_secret
Deprecated: Use Authorization Code Flow with PKCE instead
```

---

### Client Credentials Flow

```
Server-to-server authentication (no user involved)

1. App → POST /token with client_id + client_secret
2. Auth server → Return access token
3. App → Use token to call APIs

Use for: Backend services, cron jobs, microservices
```

---

### PKCE (Proof Key for Code Exchange)

```
Adds security to Authorization Code Flow for mobile/SPA

1. App generates code_verifier (random string)
2. App generates code_challenge = SHA256(code_verifier)
3. App → Redirect to auth server with code_challenge
4. User logs in
5. Auth server → Redirect with authorization code
6. App → Exchange code + code_verifier for token
7. Auth server → Verify SHA256(code_verifier) == code_challenge → Return token

Prevents: Authorization code interception attacks
```

---

## 6. Single Sign-On (SSO)

One login for multiple applications.

### SAML (Security Assertion Markup Language)

```
Enterprise SSO (older, XML-based)

1. User → Access App
2. App → Redirect to Identity Provider (IdP)
3. User → Logs in to IdP
4. IdP → Return SAML assertion (XML)
5. App → Validate assertion → Grant access

Use for: Enterprise applications, corporate SSO
```

---

### OpenID Connect (OIDC)

```
Modern SSO (built on OAuth 2.0, JSON-based)

1. User → "Login with Google"
2. App → OAuth 2.0 flow
3. Google → Return access token + ID token (JWT)
4. App → Validate ID token → Extract user info → Grant access

ID token contains:
  {
    "sub": "google_user_123",
    "email": "alice@example.com",
    "name": "Alice",
    "picture": "https://..."
  }

Use for: Modern web/mobile apps, consumer SSO
```

---

## 7. Security Best Practices

### Password storage

```
Never store plaintext passwords

Bad:
  password: "secret123"

Good:
  password_hash: bcrypt("secret123", salt)
  
Verify:
  bcrypt(input_password, salt) == stored_hash
```

**Use bcrypt, Argon2, or scrypt** (slow hashing algorithms designed for passwords).

---

### Token storage

```
Web:
  - HttpOnly cookie (can't be accessed by JavaScript, prevents XSS)
  - Secure flag (only sent over HTTPS)
  - SameSite flag (prevents CSRF)

Mobile:
  - Secure storage (Keychain on iOS, Keystore on Android)

Never:
  - localStorage (vulnerable to XSS)
  - URL parameters (logged, cached)
```

---

### HTTPS everywhere

```
All authentication traffic over HTTPS
Prevents: Man-in-the-middle attacks, credential theft
```

---

### Rate limiting

```
Prevent brute force attacks:
  - Max 5 login attempts per minute per IP
  - Max 10 login attempts per hour per account
  - Exponential backoff after failures
```

---

### Token expiration

```
Access tokens: Short-lived (15 minutes)
Refresh tokens: Long-lived (7 days)
Session tokens: Medium-lived (24 hours)

Expired tokens must be rejected
```

---

## 8. Common Interview Questions + Answers

### Q: What's the difference between authentication and authorization?

> "Authentication is verifying who you are — like logging in with a username and password. Authorization is determining what you can do — like checking if you have permission to delete a resource. Authentication happens first, then authorization. For example, you authenticate as Alice, then the system checks if Alice is authorized to access the admin panel. You can be authenticated but not authorized for a specific action."

### Q: What's the difference between session-based and token-based authentication?

> "Session-based stores session state on the server — the client gets a session ID cookie, and the server looks up the session in memory or a database. Token-based is stateless — the client gets a JWT containing all necessary information, and the server validates the signature without storing anything. Session-based allows easy revocation but doesn't scale horizontally without a shared session store. Token-based scales easily but can't be revoked until expiry. I'd use sessions for traditional web apps and JWTs for APIs and microservices."

### Q: How does OAuth 2.0 work?

> "OAuth 2.0 is a delegation protocol that lets users grant third-party apps access to their resources without sharing passwords. The most common flow is Authorization Code: the user clicks 'Login with Google,' gets redirected to Google, logs in, and Google redirects back with an authorization code. The app exchanges this code for an access token using its client secret (server-side). The app then uses the access token to call Google APIs on behalf of the user. This is secure because the client secret never leaves the server."

### Q: What's the difference between RBAC and ABAC?

> "RBAC (Role-Based Access Control) assigns users to roles, and roles have permissions. It's simple — Alice is an Editor, so she can read and write. ABAC (Attribute-Based Access Control) makes decisions based on attributes of the user, resource, and environment. For example, 'allow if user.department == resource.department AND time.hour >= 9.' RBAC is simpler and works well for most applications. ABAC is more flexible and fine-grained but more complex. I'd start with RBAC and move to ABAC only if I need dynamic, context-aware access control."

---

## 9. Interview Tricks & Pitfalls

### ✅ Trick 1: Distinguish AuthN from AuthZ

Always clarify which you're discussing:

> "For authentication, I'd use JWT tokens. For authorization, I'd implement RBAC with roles like Admin, Editor, and Viewer. The JWT contains the user's role, and each endpoint checks if the role has the required permission."

### ✅ Trick 2: Mention refresh tokens

When discussing JWTs:

> "I'd use short-lived access tokens (15 minutes) with long-lived refresh tokens (7 days). This balances security (stolen access tokens expire quickly) with UX (users don't need to log in frequently). Refresh tokens are stored server-side so they can be revoked."

### ✅ Trick 3: Address token storage

Show you understand security:

> "I'd store JWTs in HttpOnly cookies with Secure and SameSite flags. This prevents XSS attacks (JavaScript can't access the cookie) and CSRF attacks (SameSite prevents cross-site requests). Never store tokens in localStorage — it's vulnerable to XSS."

### ❌ Pitfall 1: Confusing OAuth with authentication

OAuth is for authorization (delegation), not authentication:

> "OAuth 2.0 is for authorization — granting access to resources. For authentication, you'd use OpenID Connect, which is built on top of OAuth 2.0 and adds an ID token that contains user information."

### ❌ Pitfall 2: Not mentioning password hashing

If you discuss passwords, mention hashing:

> "I'd hash passwords with bcrypt before storing them. Never store plaintext passwords. Bcrypt is designed to be slow, which makes brute force attacks impractical."

### ❌ Pitfall 3: Forgetting about rate limiting

Authentication endpoints need protection:

> "I'd rate limit login attempts — max 5 per minute per IP and 10 per hour per account. This prevents brute force attacks. After repeated failures, I'd add exponential backoff or require CAPTCHA."

---

## 10. Quick Reference

```
Authentication (AuthN) = Who are you?
Authorization (AuthZ) = What can you do?

Authentication methods:
  Username/Password:  Simple, hash with bcrypt
  Session-based:      Server stores session, client gets session ID
  Token-based (JWT):  Stateless, client stores token
  OAuth 2.0:          Delegate to third party (Google, Facebook)
  MFA:                Multiple factors (password + phone)
  API Keys:           Simple token for APIs

Authorization models:
  RBAC:   Roles have permissions (simple)
  ABAC:   Attribute-based (flexible, complex)
  ACL:    Per-resource permissions
  Policy: Centralized policy engine (OPA)

JWT:
  Structure: Header.Payload.Signature
  Claims: iss, sub, aud, exp, iat
  Validation: Verify signature + check expiration
  Refresh tokens: Long-lived, get new access token

OAuth 2.0 flows:
  Authorization Code:  Most secure, for web apps
  Client Credentials:  Server-to-server
  PKCE:                Mobile/SPA security

SSO:
  SAML:  Enterprise, XML-based
  OIDC:  Modern, built on OAuth 2.0, JSON-based

Security best practices:
  - Hash passwords (bcrypt, Argon2)
  - Store tokens in HttpOnly cookies
  - HTTPS everywhere
  - Rate limit login attempts
  - Short-lived access tokens + long-lived refresh tokens
  - Never store tokens in localStorage

Key insight: Authentication verifies identity, authorization controls access
```
