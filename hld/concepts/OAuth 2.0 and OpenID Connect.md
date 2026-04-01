## 1. OAuth 2.0 vs OpenID Connect

**OAuth 2.0** is an authorization framework — it lets a user grant a third-party app limited access to their resources without sharing credentials.

**OpenID Connect (OIDC)** is an identity layer built on top of OAuth 2.0 — it adds authentication, letting apps verify who the user is.

```
OAuth 2.0:  "Can this app access my Google Drive?"  → Authorization
OIDC:       "Who is this user?"                     → Authentication

OIDC = OAuth 2.0 + ID Token (JWT with user identity)
```

---

## 2. OAuth 2.0 Core Concepts

### Roles

```
Resource Owner:  The user who owns the data
Client:          The app requesting access (e.g., a mobile app)
Authorization Server: Issues tokens (e.g., Google, Auth0)
Resource Server: Hosts the protected data (e.g., Google Drive API)
```

### Tokens

```
Access Token:  Short-lived token to access protected resources
               Opaque string or JWT, typically expires in 1 hour
               
Refresh Token: Long-lived token to get new access tokens
               Stored securely, can be revoked
               
ID Token (OIDC only): JWT containing user identity claims
                      Signed by authorization server
```

---

## 3. OAuth 2.0 Flows (Grant Types)

### Authorization Code Flow (Most Common)

Used by web apps and mobile apps. Most secure flow.

```
1. User clicks "Login with Google"
2. App redirects to Google authorization page
   https://accounts.google.com/authorize?
     client_id=abc123&
     redirect_uri=https://myapp.com/callback&
     response_type=code&
     scope=profile email

3. User logs in and grants permission
4. Google redirects back with authorization code
   https://myapp.com/callback?code=xyz789

5. App exchanges code for tokens (server-to-server)
   POST https://oauth2.googleapis.com/token
   {
     "code": "xyz789",
     "client_id": "abc123",
     "client_secret": "secret",
     "redirect_uri": "https://myapp.com/callback",
     "grant_type": "authorization_code"
   }

6. Google returns tokens
   {
     "access_token": "ya29.a0...",
     "refresh_token": "1//0g...",
     "expires_in": 3600,
     "token_type": "Bearer"
   }

7. App uses access token to call APIs
   GET https://www.googleapis.com/drive/v3/files
   Authorization: Bearer ya29.a0...
```

**Why the code exchange?** The authorization code is exchanged server-side using the client secret, which never leaves the server. This prevents token theft if the redirect is intercepted.

---

### Authorization Code Flow with PKCE

**PKCE (Proof Key for Code Exchange)** adds security for public clients (mobile apps, SPAs) that can't securely store a client secret.

```
1. App generates random code_verifier (43-128 chars)
   code_verifier = "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"

2. App creates code_challenge = SHA256(code_verifier)
   code_challenge = "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"

3. Authorization request includes code_challenge
   https://accounts.google.com/authorize?
     ...&code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&
     code_challenge_method=S256

4. Token exchange includes code_verifier
   POST /token
   {
     "code": "xyz789",
     "code_verifier": "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk",
     ...
   }

5. Server verifies SHA256(code_verifier) == code_challenge
```

**Why PKCE?** Even if an attacker intercepts the authorization code, they can't exchange it without the original `code_verifier`, which never leaves the client.

**Modern best practice:** Use PKCE for all OAuth flows, even confidential clients.

---

### Client Credentials Flow

Used for server-to-server communication (no user involved).

```
POST /token
{
  "grant_type": "client_credentials",
  "client_id": "service-a",
  "client_secret": "secret",
  "scope": "read:orders"
}

Response:
{
  "access_token": "eyJhbGc...",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

**Use case:** Microservice A needs to call Microservice B's API. No user context.

---

### Implicit Flow (Deprecated)

Tokens returned directly in URL fragment. No longer recommended — use Authorization Code + PKCE instead.

```
https://myapp.com/callback#access_token=ya29...&expires_in=3600
```

**Why deprecated?** Tokens exposed in browser history and referrer headers. No refresh token support.

---

### Resource Owner Password Credentials (Deprecated)

User provides username/password directly to the app. Only for legacy systems or first-party apps.

```
POST /token
{
  "grant_type": "password",
  "username": "user@example.com",
  "password": "secret",
  "client_id": "abc123"
}
```

**Why deprecated?** Defeats the purpose of OAuth — the app sees the user's password.

---

## 4. OpenID Connect (OIDC)

OIDC adds an **ID Token** to OAuth 2.0. The ID Token is a JWT containing user identity claims.

### Authorization Code Flow with OIDC

```
1. Authorization request includes openid scope
   https://accounts.google.com/authorize?
     ...&scope=openid profile email

2. Token response includes ID token
   {
     "access_token": "ya29...",
     "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
     "refresh_token": "1//0g...",
     "expires_in": 3600
   }

3. Decode ID token (JWT)
   {
     "iss": "https://accounts.google.com",
     "sub": "110169484474386276334",
     "aud": "abc123.apps.googleusercontent.com",
     "exp": 1672531200,
     "iat": 1672527600,
     "email": "user@example.com",
     "email_verified": true,
     "name": "John Doe",
     "picture": "https://..."
   }
```

**Key claims:**

```
iss (issuer):   Who issued the token
sub (subject):  Unique user ID
aud (audience): Client ID this token is for
exp (expiry):   Token expiration timestamp
iat (issued at): When token was issued
```

**Validation:** Always verify the signature, issuer, audience, and expiry before trusting the ID token.

---

### UserInfo Endpoint

Alternative to decoding the ID token — call the UserInfo endpoint with the access token.

```
GET https://openidconnect.googleapis.com/v1/userinfo
Authorization: Bearer ya29...

Response:
{
  "sub": "110169484474386276334",
  "email": "user@example.com",
  "name": "John Doe",
  "picture": "https://..."
}
```

---

## 5. Token Refresh Flow

Access tokens are short-lived (1 hour). Use the refresh token to get a new access token without re-authenticating the user.

```
POST /token
{
  "grant_type": "refresh_token",
  "refresh_token": "1//0g...",
  "client_id": "abc123",
  "client_secret": "secret"
}

Response:
{
  "access_token": "new_access_token",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

**Refresh token rotation:** Some providers issue a new refresh token with each refresh. The old refresh token is invalidated. This limits the damage if a refresh token is stolen.

---

## 6. Scopes

Scopes define what the access token can do.

```
scope=openid profile email          → OIDC: get user identity
scope=read:orders write:orders      → OAuth: access order data
scope=https://www.googleapis.com/auth/drive.readonly → Google Drive read-only
```

**Best practice:** Request the minimum scopes needed. Users are more likely to grant narrow permissions.

---

## 7. Token Storage

### Access Token

```
Web app (server-side):  Store in session, never expose to client
SPA (browser):          Store in memory (React state), not localStorage
Mobile app:             Store in secure storage (iOS Keychain, Android Keystore)
```

**Never store access tokens in localStorage** — vulnerable to XSS attacks.

---

### Refresh Token

```
Web app:   Store in httpOnly, secure, SameSite cookie
SPA:       Don't store — use BFF (Backend for Frontend) pattern
Mobile:    Secure storage (Keychain, Keystore)
```

**Refresh tokens are long-lived and powerful** — treat them like passwords.

---

## 8. Security Best Practices

### Always use HTTPS

OAuth tokens in transit must be encrypted. Never use OAuth over HTTP.

---

### Validate redirect_uri

Authorization servers must validate the `redirect_uri` against a whitelist. Prevents authorization code interception attacks.

```
Registered redirect URIs:
  https://myapp.com/callback
  https://myapp.com/oauth/callback

Reject:
  https://evil.com/steal-code
  https://myapp.com.evil.com/callback
```

---

### Validate state parameter

Prevents CSRF attacks. The client generates a random `state` value, includes it in the authorization request, and verifies it matches in the callback.

```
1. App generates state = "random123"
2. Authorization request: ...&state=random123
3. Callback: ?code=xyz&state=random123
4. App verifies state matches → prevents CSRF
```

---

### Short-lived access tokens

Access tokens should expire quickly (15 minutes to 1 hour). Use refresh tokens for long-lived sessions.

---

### Rotate refresh tokens

Issue a new refresh token on each refresh. Invalidate the old one. Limits the window of vulnerability if a token is stolen.

---

## 9. Common Use Cases

### Social Login

"Login with Google/Facebook/GitHub" — uses OIDC to authenticate users without managing passwords.

```
User clicks "Login with Google"
→ Authorization Code Flow with OIDC
→ App receives ID token with user email, name, picture
→ App creates or updates user account
→ User is logged in
```

---

### Third-Party API Access

"Allow Zapier to access your Slack workspace" — uses OAuth 2.0 to grant limited access.

```
User authorizes Zapier
→ Zapier receives access token with scope=channels:read,chat:write
→ Zapier can read channels and send messages, but not delete workspace
```

---

### Microservices Authentication

Service A calls Service B using client credentials flow.

```
Service A → Token endpoint (client credentials)
         → Access token
         → Service B API (Authorization: Bearer token)
```

---

## 10. OAuth 2.0 vs SAML

|Feature|OAuth 2.0 / OIDC|SAML|
|---|---|---|
|Primary use case|Authorization + Authentication|Authentication (SSO)|
|Token format|JWT (OIDC) or opaque|XML|
|Transport|JSON over HTTPS|XML over HTTP POST/Redirect|
|Mobile support|Excellent|Poor|
|Complexity|Moderate|High|
|Enterprise adoption|Growing|Dominant (legacy)|

**Modern systems prefer OIDC.** SAML is still common in enterprise SSO for legacy apps.

---

## 11. Common Interview Questions + Answers

### Q: What's the difference between OAuth 2.0 and OpenID Connect?

> "OAuth 2.0 is for authorization — it lets an app access resources on behalf of a user. OIDC is for authentication — it adds an ID token (JWT) that tells the app who the user is. OIDC is built on top of OAuth 2.0 by adding the `openid` scope and the ID token to the token response."

---

### Q: Why use PKCE?

> "PKCE protects public clients like mobile apps and SPAs that can't securely store a client secret. The client generates a random code verifier, hashes it to create a code challenge, and sends the challenge with the authorization request. When exchanging the code for tokens, the client sends the original verifier. The server verifies the hash matches. This prevents authorization code interception attacks — even if an attacker steals the code, they can't exchange it without the verifier."

---

### Q: How do you handle token refresh in a single-page app?

> "SPAs can't securely store refresh tokens because they're vulnerable to XSS. The best pattern is a Backend for Frontend (BFF) — a thin server-side proxy that holds the refresh token in an httpOnly cookie. The SPA calls the BFF to refresh tokens. The BFF exchanges the refresh token and returns a new access token. This keeps the refresh token out of the browser."

---

### Q: What's the state parameter for?

> "The state parameter prevents CSRF attacks. The client generates a random value, stores it in session, and includes it in the authorization request. When the authorization server redirects back, the client verifies the state matches. This ensures the callback is in response to a request initiated by the client, not an attacker."

---

## 12. Interview Tricks & Pitfalls

### ✅ Trick 1: Know the flows by name

Interviewers expect you to say "Authorization Code Flow with PKCE" not "the one where you get a code first." Knowing the official names signals experience.

---

### ✅ Trick 2: Distinguish authorization vs authentication

OAuth 2.0 alone doesn't tell you who the user is — it only grants access. OIDC adds authentication. This distinction comes up constantly.

---

### ✅ Trick 3: Mention PKCE for mobile/SPA

If designing a mobile app or SPA, always mention PKCE. It's the modern standard and shows you're up to date.

---

### ❌ Pitfall 1: Storing tokens in localStorage

Never store access tokens or refresh tokens in localStorage — they're accessible to any JavaScript on the page, including XSS attacks. Use memory (for access tokens) or httpOnly cookies (for refresh tokens).

---

### ❌ Pitfall 2: Using implicit flow

Implicit flow is deprecated. Always use Authorization Code + PKCE instead, even for SPAs.

---

### ❌ Pitfall 3: Not validating ID tokens

ID tokens are JWTs — you must verify the signature, issuer, audience, and expiry. Don't just decode and trust the claims.

---

## 13. Quick Reference

```
OAuth 2.0:  Authorization framework (access delegation)
OIDC:       Authentication layer on OAuth 2.0 (identity)

Flows:
  Authorization Code + PKCE  → Web, mobile, SPA (most secure)
  Client Credentials         → Service-to-service (no user)
  Implicit (deprecated)      → Use Authorization Code + PKCE instead
  Password (deprecated)      → Use Authorization Code instead

Tokens:
  Access Token   → Short-lived (1 hour), access resources
  Refresh Token  → Long-lived, get new access tokens
  ID Token (OIDC) → JWT with user identity claims

Security:
  ✅ Always HTTPS
  ✅ Validate redirect_uri
  ✅ Use state parameter (CSRF protection)
  ✅ Use PKCE for public clients
  ✅ Short-lived access tokens
  ✅ Rotate refresh tokens
  ❌ Never store tokens in localStorage
  ❌ Never use implicit flow

Token storage:
  Access token:  Memory (SPA), session (server), secure storage (mobile)
  Refresh token: httpOnly cookie (web), secure storage (mobile), BFF (SPA)

OIDC claims:
  iss → issuer
  sub → user ID
  aud → client ID
  exp → expiry
  iat → issued at
```
