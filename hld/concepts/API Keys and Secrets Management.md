## 1. What Are API Keys?

API keys are simple tokens used to authenticate and authorize API requests.

```
Client → GET /api/users
Headers: X-API-Key: <example-api-key-not-real>

Server → Validates key → Returns data
```

**Purpose:**
- Identify the calling application
- Track usage and enforce rate limits
- Control access to APIs

---

## 2. API Key Types

### Public keys (publishable)

```
pk_<example-publishable-key-not-real>

Safe to expose in client-side code
Limited permissions (read-only, specific endpoints)
Used in: Frontend JavaScript, mobile apps
```

**Example:** Stripe publishable key for creating payment tokens (can't charge money).

---

### Secret keys (private)

```
sk_<example-secret-key-not-real>

Must be kept secret
Full permissions
Used in: Backend servers only
```

**Example:** Stripe secret key for charging payments.

---

### Scoped keys

```
Key with specific permissions:
  - Read users only
  - Write to specific database
  - Access specific resources

Example:
  key_read_users_only: Can only GET /users
  key_full_access: Can do anything
```

---

## 3. API Key Format

### Best practices

```
Prefix + Random string:
  sk_<prefix>_<random-string>
  ↑   ↑    ↑
  |   |    Random cryptographic string
  |   Environment (live, test)
  Type (secret key)

Benefits:
  - Easy to identify key type
  - Easy to rotate by environment
  - Can detect leaked keys in code repos
```

---

### Generation

```python
import secrets

def generate_api_key(prefix="sk"):
    random_part = secrets.token_urlsafe(32)  # 32 bytes = 43 chars
    return f"{prefix}_{random_part}"

# Example output shape: prefix + "_" + random chars (never commit real keys to git)
```

**Use cryptographically secure random generator** (not `random.random()`).

---

## 4. Storing API Keys

### Server-side storage

```
Database table:
  id:         1
  key_hash:   bcrypt("<hashed-api-key>")
  user_id:    456
  permissions: ["read", "write"]
  created_at: 2024-01-15
  last_used:  2024-01-20

Never store plaintext keys
Store hash (like passwords)
```

**Validation:**

```python
def validate_api_key(provided_key):
    # Hash the provided key
    key_hash = bcrypt.hash(provided_key)
    
    # Look up in database
    record = db.query("SELECT * FROM api_keys WHERE key_hash = ?", key_hash)
    
    if record:
        # Update last_used timestamp
        db.execute("UPDATE api_keys SET last_used = NOW() WHERE id = ?", record.id)
        return record
    
    return None  # invalid key
```

---

### Client-side storage

```
Backend:
  - Environment variables (never in code)
  - Secrets manager (AWS Secrets Manager, HashiCorp Vault)
  - Configuration files (not in version control)

Frontend (public keys only):
  - Environment variables (injected at build time)
  - Configuration files
  - Never hardcode in source code
```

---

## 5. Secrets Management

### The problem

```
Bad:
  # config.py
  DATABASE_URL = "postgresql://user:password@host/db"
  API_KEY = "<example-not-real>"
  
  # Committed to Git → exposed to anyone with repo access
```

---

### Environment variables

```
# .env file (not in Git)
DATABASE_URL=postgresql://user:password@host/db
API_KEY=<example-not-real>

# Application reads from environment
import os
database_url = os.getenv("DATABASE_URL")
api_key = os.getenv("API_KEY")
```

**Pros:** Simple, widely supported. **Cons:** Can leak in logs, process listings, error messages.

---

### Secrets managers

**AWS Secrets Manager:**

```python
import boto3

client = boto3.client('secretsmanager')

# Store secret
client.create_secret(
    Name='prod/database/password',
    SecretString='my_secure_password'
)

# Retrieve secret
response = client.get_secret_value(SecretId='prod/database/password')
password = response['SecretString']
```

**Benefits:**
- Encrypted at rest
- Access control (IAM policies)
- Automatic rotation
- Audit logging

---

**HashiCorp Vault:**

```bash
# Store secret
vault kv put secret/database password=my_secure_password

# Retrieve secret
vault kv get secret/database
```

**Benefits:**
- Dynamic secrets (generate on-demand)
- Lease management (auto-expire)
- Encryption as a service
- Multi-cloud support

---

### Kubernetes Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
data:
  username: YWRtaW4=  # base64 encoded
  password: cGFzc3dvcmQ=

---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: database-credentials
          key: username
```

**Note:** Kubernetes secrets are base64 encoded, not encrypted by default. Use encryption at rest or external secrets manager.

---

## 6. API Key Rotation

### Why rotate?

```
Reasons:
  - Key leaked (committed to Git, exposed in logs)
  - Employee left company
  - Periodic security practice (every 90 days)
  - Suspected compromise
```

---

### Rotation strategy

```
1. Generate new key
2. Distribute new key to clients
3. Support both old and new keys (grace period)
4. After grace period, revoke old key

Timeline:
  Day 0:  Generate new key, send to clients
  Day 1-7: Both keys work (grace period)
  Day 7:  Revoke old key
```

**Implementation:**

```python
def validate_api_key(provided_key):
    # Check if key is valid (not revoked)
    record = db.query("""
        SELECT * FROM api_keys 
        WHERE key_hash = ? 
        AND revoked_at IS NULL
    """, bcrypt.hash(provided_key))
    
    return record
```

---

### Automatic rotation

```
AWS Secrets Manager:
  - Configure rotation schedule (e.g., every 30 days)
  - Lambda function generates new secret
  - Updates database password
  - Updates secret in Secrets Manager

Benefits:
  - No manual intervention
  - Reduces risk of forgotten rotation
  - Enforces security policy
```

---

## 7. API Key Security Best Practices

### 1. Never commit to version control

```
# .gitignore
.env
secrets.yaml
config/production.yml
*.key
*.pem
```

**Use pre-commit hooks to detect secrets:**

```bash
# Install git-secrets
git secrets --install
git secrets --register-aws

# Scans commits for secrets
```

---

### 2. Use different keys per environment

```
Development:  sk_dev_<example>
Staging:      sk_staging_<example>
Production:   sk_<prod-example-not-real>

Benefits:
  - Leaked dev key doesn't compromise production
  - Easy to identify which environment
  - Can rotate independently
```

---

### 3. Principle of least privilege

```
Key for analytics service:
  - Read access to analytics database only
  - No write access
  - No access to user database

Key for admin panel:
  - Full access to all resources
  - Only used by admin users
```

---

### 4. Rate limiting per key

```
Free tier:     1,000 requests/day
Pro tier:      100,000 requests/day
Enterprise:    Unlimited

Track usage per API key
Enforce limits
Alert on unusual activity
```

---

### 5. Monitor and alert

```
Alerts:
  - API key used from new IP address
  - API key used from new country
  - Unusual spike in requests
  - Failed authentication attempts
  - Key approaching rate limit
```

---

### 6. Expiration

```
Short-lived keys:
  - Expire after 24 hours
  - Must be refreshed
  - Reduces risk if leaked

Long-lived keys:
  - Expire after 1 year
  - Require manual rotation
  - Convenient but riskier
```

---

## 8. Detecting Leaked Secrets

### GitHub secret scanning

```
GitHub automatically scans commits for known secret patterns:
  - AWS keys
  - Stripe keys
  - Google API keys
  - Private keys

If detected:
  - Alert repository owner
  - Notify secret provider (e.g., AWS)
  - Provider may revoke key automatically
```

---

### Tools

**TruffleHog:**

```bash
# Scan Git history for secrets
trufflehog git https://github.com/user/repo

# Finds:
  - API keys
  - Passwords
  - Private keys
  - Tokens
```

**git-secrets:**

```bash
# Prevent committing secrets
git secrets --scan

# Blocks commit if secrets detected
```

---

## 9. Secrets in CI/CD

### GitHub Actions

```yaml
# Store secrets in GitHub Settings → Secrets

name: Deploy
on: push
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Deploy
        env:
          API_KEY: ${{ secrets.API_KEY }}
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: ./deploy.sh
```

**Secrets are encrypted and only exposed to workflows.**

---

### Environment-specific secrets

```yaml
# Different secrets per environment

jobs:
  deploy-staging:
    environment: staging
    env:
      API_KEY: ${{ secrets.STAGING_API_KEY }}
  
  deploy-production:
    environment: production
    env:
      API_KEY: ${{ secrets.PRODUCTION_API_KEY }}
```

---

## 10. Common Interview Questions + Answers

### Q: How do you securely store API keys?

> "I'd never store API keys in plaintext. On the server, I'd hash them with bcrypt before storing in the database, just like passwords. When validating, I hash the provided key and compare with the stored hash. For secrets my application needs (like database passwords or third-party API keys), I'd use a secrets manager like AWS Secrets Manager or HashiCorp Vault. These encrypt secrets at rest, provide access control, support automatic rotation, and audit all access. For local development, I'd use environment variables in a .env file that's not committed to Git."

### Q: What's the difference between API keys and JWT tokens?

> "API keys are long-lived, simple tokens that identify an application or service. They're typically used for server-to-server communication and don't contain user context. JWT tokens are short-lived, contain user information and permissions in the payload, and are used for user authentication. API keys are stateless but can't be easily revoked (must be rotated). JWTs are also stateless but expire quickly, reducing risk. I'd use API keys for third-party integrations and internal service-to-service calls, and JWTs for user authentication in web and mobile apps."

### Q: How do you handle API key rotation?

> "I'd implement a grace period strategy. First, generate a new key and distribute it to clients. For a grace period (typically 7 days), both the old and new keys work. This gives clients time to update. After the grace period, I revoke the old key. In the database, I'd have a revoked_at timestamp — keys with a non-null revoked_at are rejected. For critical secrets like database passwords, I'd use automatic rotation with AWS Secrets Manager, which generates new credentials, updates the database, and updates the secret without manual intervention."

### Q: How do you prevent API keys from being committed to Git?

> "I'd use multiple layers of protection. First, add common secret file patterns to .gitignore (.env, secrets.yaml, *.key). Second, use pre-commit hooks with tools like git-secrets to scan commits for secret patterns and block them. Third, enable GitHub secret scanning, which automatically detects and alerts on committed secrets. Fourth, use environment variables or secrets managers instead of config files. And finally, educate the team on security practices and conduct regular audits of the repository history."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Mention hashing

When discussing API key storage:

> "I'd hash API keys with bcrypt before storing them in the database, just like passwords. This way, even if the database is compromised, the attacker can't use the keys. When validating, I hash the provided key and compare with the stored hash."

### ✅ Trick 2: Discuss rotation

Show you understand operational security:

> "I'd implement API key rotation with a grace period. Generate a new key, support both old and new keys for 7 days, then revoke the old key. This allows clients to update without downtime. For critical secrets, I'd use automatic rotation with AWS Secrets Manager."

### ✅ Trick 3: Differentiate key types

Shows you understand the nuances:

> "I'd use different key types: public keys for frontend (limited permissions, safe to expose) and secret keys for backend (full permissions, must be kept secret). Each key would be scoped to specific permissions following the principle of least privilege."

### ❌ Pitfall 1: Storing keys in plaintext

Never store secrets in plaintext:

> "I'd never store API keys in plaintext in the database or in code. Keys should be hashed in the database and stored in environment variables or a secrets manager in the application."

### ❌ Pitfall 2: Not mentioning secrets managers

For production systems, environment variables aren't enough:

> "For production, I'd use a secrets manager like AWS Secrets Manager or HashiCorp Vault. These provide encryption at rest, access control, automatic rotation, and audit logging. Environment variables are fine for development but not secure enough for production."

### ❌ Pitfall 3: Forgetting about monitoring

API keys need visibility:

> "I'd monitor API key usage — track requests per key, alert on unusual activity (new IP, new country, spike in requests), and log all authentication failures. This helps detect compromised keys quickly."

---

## 12. Quick Reference

```
API Keys = simple tokens for API authentication

Types:
  Public (publishable):  Safe to expose, limited permissions
  Secret (private):      Must be kept secret, full permissions
  Scoped:                Specific permissions per key

Format:
  Prefix + Environment + Random string
  Example: prefix + environment + random string (never use real keys in repos)

Storage:
  Server-side:  Hash with bcrypt (like passwords)
  Client-side:  Environment variables, secrets manager
  Never:        Plaintext in code, committed to Git

Secrets management:
  Environment variables:  Simple, can leak in logs
  AWS Secrets Manager:    Encrypted, access control, auto-rotation
  HashiCorp Vault:        Dynamic secrets, multi-cloud
  Kubernetes Secrets:     Base64 encoded (not encrypted by default)

Rotation:
  1. Generate new key
  2. Distribute to clients
  3. Grace period (both keys work)
  4. Revoke old key

Security best practices:
  ✅ Never commit to Git (.gitignore, pre-commit hooks)
  ✅ Different keys per environment (dev, staging, prod)
  ✅ Principle of least privilege (scope permissions)
  ✅ Rate limiting per key
  ✅ Monitor and alert (unusual activity)
  ✅ Expiration (short-lived or periodic rotation)

Detecting leaks:
  - GitHub secret scanning (automatic)
  - TruffleHog (scan Git history)
  - git-secrets (pre-commit hooks)

CI/CD:
  - Store secrets in CI/CD platform (GitHub Secrets, GitLab CI/CD variables)
  - Environment-specific secrets
  - Never log secrets

API keys vs JWT:
  API keys:  Long-lived, identify application, server-to-server
  JWT:       Short-lived, contain user info, user authentication

Key insight: Treat API keys like passwords — hash, rotate, monitor
```
