## 1. What is Audit Logging?

Audit logging is the practice of recording all security-relevant events in a tamper-evident log for compliance, security analysis, and forensics.

```
Regular Logging:
  - Application errors
  - Performance metrics
  - Debug information

Audit Logging:
  - Who did what, when, where
  - Authentication attempts
  - Authorization decisions
  - Data access
  - Configuration changes
  - Immutable, tamper-evident
```

**Key principle:** Record all sensitive actions for accountability and compliance.

---

## 2. What to Log

### Authentication Events

```
- Login attempts (success/failure)
- Logout
- Password changes
- MFA enrollment
- Session creation/expiration

Example:
  {
    "event": "login_success",
    "user_id": "user_123",
    "ip": "192.168.1.1",
    "timestamp": "2024-01-01T10:00:00Z",
    "mfa_used": true
  }
```

### Authorization Events

```
- Access granted/denied
- Permission changes
- Role assignments
- Policy updates

Example:
  {
    "event": "access_denied",
    "user_id": "user_123",
    "resource": "/api/admin/users",
    "reason": "insufficient_permissions",
    "timestamp": "2024-01-01T10:01:00Z"
  }
```

### Data Access

```
- Read sensitive data
- Create/update/delete records
- Export data
- Bulk operations

Example:
  {
    "event": "data_access",
    "user_id": "user_123",
    "action": "read",
    "resource": "user_profile",
    "resource_id": "user_456",
    "timestamp": "2024-01-01T10:02:00Z"
  }
```

### Configuration Changes

```
- System settings
- Feature flags
- Security policies
- Infrastructure changes

Example:
  {
    "event": "config_change",
    "user_id": "admin_789",
    "setting": "max_login_attempts",
    "old_value": "3",
    "new_value": "5",
    "timestamp": "2024-01-01T10:03:00Z"
  }
```

---

## 3. Audit Log Format

### Standard Fields

```json
{
  "event_id": "evt_abc123",
  "event_type": "data_access",
  "timestamp": "2024-01-01T10:00:00Z",
  "actor": {
    "user_id": "user_123",
    "ip": "192.168.1.1",
    "user_agent": "Mozilla/5.0..."
  },
  "action": "read",
  "resource": {
    "type": "user_profile",
    "id": "user_456"
  },
  "result": "success",
  "metadata": {
    "request_id": "req_xyz789"
  }
}
```

### Who, What, When, Where

```
Who: actor (user_id, service_id, ip)
What: action (read, write, delete)
When: timestamp (ISO 8601)
Where: resource (type, id, path)
Result: success, failure, denied
Why: reason (optional)
```

---

## 4. Implementation Example

### Audit Logger (Python)

```python
import json
import hashlib
from datetime import datetime

class AuditLogger:
    def __init__(self, log_file):
        self.log_file = log_file
        self.previous_hash = None
    
    def log(self, event_type, actor, action, resource, result, metadata=None):
        event = {
            "event_id": self._generate_id(),
            "event_type": event_type,
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "actor": actor,
            "action": action,
            "resource": resource,
            "result": result,
            "metadata": metadata or {},
            "previous_hash": self.previous_hash
        }
        
        # Calculate hash for tamper detection
        event_hash = self._calculate_hash(event)
        event["hash"] = event_hash
        self.previous_hash = event_hash
        
        # Write to log (append-only)
        with open(self.log_file, 'a') as f:
            f.write(json.dumps(event) + '\n')
    
    def _generate_id(self):
        import uuid
        return f"evt_{uuid.uuid4().hex[:12]}"
    
    def _calculate_hash(self, event):
        # Hash event for tamper detection
        event_str = json.dumps(event, sort_keys=True)
        return hashlib.sha256(event_str.encode()).hexdigest()

# Usage
audit = AuditLogger('/var/log/audit.log')

audit.log(
    event_type="data_access",
    actor={"user_id": "user_123", "ip": "192.168.1.1"},
    action="read",
    resource={"type": "user_profile", "id": "user_456"},
    result="success"
)
```


### Middleware for Audit Logging

```python
from flask import Flask, request, g
import time

app = Flask(__name__)
audit = AuditLogger('/var/log/audit.log')

@app.before_request
def before_request():
    g.start_time = time.time()

@app.after_request
def after_request(response):
    # Log all API requests
    duration = time.time() - g.start_time
    
    audit.log(
        event_type="api_request",
        actor={
            "user_id": g.user_id if hasattr(g, 'user_id') else None,
            "ip": request.remote_addr,
            "user_agent": request.user_agent.string
        },
        action=request.method,
        resource={
            "type": "api_endpoint",
            "path": request.path
        },
        result="success" if response.status_code < 400 else "failure",
        metadata={
            "status_code": response.status_code,
            "duration_ms": int(duration * 1000)
        }
    )
    
    return response
```

---

## 5. Tamper-Evident Logging

### Hash Chain

```
Each log entry includes hash of previous entry:

Entry 1: {data, hash: hash(data), prev_hash: null}
Entry 2: {data, hash: hash(data), prev_hash: hash1}
Entry 3: {data, hash: hash(data), prev_hash: hash2}

If entry modified:
  - Hash doesn't match
  - Chain broken
  - Tampering detected
```

### Digital Signatures

```python
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import rsa, padding

# Generate key pair
private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
public_key = private_key.public_key()

def sign_log_entry(entry, private_key):
    entry_bytes = json.dumps(entry, sort_keys=True).encode()
    signature = private_key.sign(
        entry_bytes,
        padding.PSS(
            mgf=padding.MGF1(hashes.SHA256()),
            salt_length=padding.PSS.MAX_LENGTH
        ),
        hashes.SHA256()
    )
    return signature.hex()

def verify_log_entry(entry, signature, public_key):
    entry_bytes = json.dumps(entry, sort_keys=True).encode()
    try:
        public_key.verify(
            bytes.fromhex(signature),
            entry_bytes,
            padding.PSS(
                mgf=padding.MGF1(hashes.SHA256()),
                salt_length=padding.PSS.MAX_LENGTH
            ),
            hashes.SHA256()
        )
        return True
    except:
        return False
```

---

## 6. Storage and Retention

### Append-Only Storage

```
Requirements:
  - Append-only (no updates or deletes)
  - Immutable (cannot modify)
  - Durable (replicated, backed up)

Options:
  - S3 with object lock
  - Append-only database table
  - Write-ahead log (WAL)
  - Blockchain (for high security)
```

### Retention Policies

```
Compliance requirements:
  - SOC2: 1 year
  - HIPAA: 6 years
  - GDPR: Varies by data type
  - PCI DSS: 1 year

Implementation:
  - Archive old logs to cold storage
  - Compress archived logs
  - Encrypt at rest
  - Maintain hash chain for verification
```

---

## 7. Querying and Analysis

### Centralized Logging

```
Send audit logs to central system:
  - Elasticsearch (search and analysis)
  - Splunk (SIEM)
  - CloudWatch Logs (AWS)
  - BigQuery (data warehouse)

Benefits:
  - Search across all services
  - Correlate events
  - Detect patterns
  - Generate reports
```

### Common Queries

```sql
-- Failed login attempts
SELECT user_id, COUNT(*) as attempts
FROM audit_logs
WHERE event_type = 'login_failure'
  AND timestamp > NOW() - INTERVAL '1 hour'
GROUP BY user_id
HAVING COUNT(*) > 5;

-- Privilege escalation
SELECT *
FROM audit_logs
WHERE event_type = 'role_assignment'
  AND resource.role = 'admin'
ORDER BY timestamp DESC;

-- Data exports
SELECT user_id, COUNT(*) as exports
FROM audit_logs
WHERE action = 'export'
  AND timestamp > NOW() - INTERVAL '1 day'
GROUP BY user_id;
```

---

## 8. Compliance Requirements

### SOC 2

```
Requirements:
  - Log all access to sensitive data
  - Retain logs for 1 year
  - Protect logs from tampering
  - Monitor for anomalies

Audit logs must show:
  - Who accessed what data
  - When access occurred
  - Whether access was authorized
```

### GDPR

```
Requirements:
  - Log all personal data access
  - Right to access (show user their audit logs)
  - Right to erasure (delete user's audit logs)
  - Data breach notification (detect via logs)

Audit logs must include:
  - Data subject (user)
  - Purpose of processing
  - Legal basis
```

### HIPAA

```
Requirements:
  - Log all PHI access
  - Retain logs for 6 years
  - Encrypt logs at rest and in transit
  - Regular audits

Audit logs must show:
  - Who accessed patient data
  - What data was accessed
  - When access occurred
  - Why access was needed (optional)
```

---

## 9. Best Practices

### What NOT to Log

```
❌ Passwords (even hashed)
❌ Credit card numbers
❌ Social security numbers
❌ API keys, secrets
❌ Session tokens

Instead:
  ✅ Log that password was changed (not the password)
  ✅ Log last 4 digits of card (not full number)
  ✅ Log that API key was used (not the key)
```

### Performance Considerations

```
Async logging:
  - Don't block request
  - Queue logs
  - Batch writes

Example:
  request_handler():
      process_request()
      audit_queue.put(log_entry)  # Async
      return response
  
  background_worker():
      while True:
          batch = audit_queue.get_batch(100)
          write_logs(batch)
```

### Monitoring

```
Alert on:
  - Failed login attempts (> 5 in 5 minutes)
  - Privilege escalation
  - Unusual data access patterns
  - Bulk data exports
  - Configuration changes

Example:
  if failed_logins > 5:
      alert("Possible brute force attack")
```

---

## 10. Common Interview Questions + Answers

### Q: What is audit logging and why is it important?

> "Audit logging records all security-relevant events like who accessed what data, when, and whether it was authorized. It's critical for compliance requirements like SOC2 and HIPAA, security forensics to investigate breaches, and accountability to track user actions. Unlike regular application logs that focus on errors and performance, audit logs are immutable and tamper-evident, providing a reliable record of all sensitive operations."

### Q: What should you include in audit logs?

> "Include who (user ID, IP address), what (action like read/write/delete), when (timestamp), where (resource type and ID), and the result (success/failure/denied). Also include context like request ID for correlation. Never log sensitive data like passwords, credit cards, or API keys. Each entry should have a unique ID and ideally a hash of the previous entry for tamper detection. The goal is to answer 'who did what to which resource and when' for any security event."

### Q: How do you make audit logs tamper-evident?

> "Use a hash chain where each log entry includes the hash of the previous entry. If someone modifies an entry, the hash won't match and the chain breaks, revealing tampering. For higher security, digitally sign each entry with a private key. Store logs in append-only storage like S3 with object lock or an append-only database table. Replicate logs to multiple locations and use write-once-read-many (WORM) storage. These techniques make it nearly impossible to modify logs without detection."

### Q: What are the challenges with audit logging?

> "First, performance overhead from logging every request. Use async logging and batch writes to minimize impact. Second, storage costs since logs grow quickly and must be retained for years. Archive old logs to cheap storage and compress them. Third, sensitive data exposure — never log passwords or secrets. Fourth, log volume makes analysis hard. Use centralized logging with search capabilities like Elasticsearch. Finally, compliance requirements vary by regulation, so understand what you need to log and for how long."

---

## 11. Quick Reference

```
What is Audit Logging?
  Record all security-relevant events
  Immutable, tamper-evident
  For compliance, security, forensics

What to log:
  - Authentication (login, logout, password change)
  - Authorization (access granted/denied)
  - Data access (read, write, delete)
  - Configuration changes

Log format:
  Who: user_id, ip, user_agent
  What: action (read, write, delete)
  When: timestamp (ISO 8601)
  Where: resource (type, id)
  Result: success, failure, denied

Tamper-evident:
  - Hash chain (each entry hashes previous)
  - Digital signatures
  - Append-only storage
  - Replication

Storage:
  - Append-only (no updates/deletes)
  - Durable (replicated, backed up)
  - Encrypted at rest
  - Retention per compliance (1-6 years)

Compliance:
  - SOC2: 1 year retention
  - HIPAA: 6 years retention
  - GDPR: Right to access, right to erasure
  - PCI DSS: 1 year retention

What NOT to log:
  ❌ Passwords
  ❌ Credit cards
  ❌ API keys
  ❌ Session tokens

Best practices:
  - Async logging (don't block requests)
  - Centralized logging (Elasticsearch, Splunk)
  - Monitor for anomalies
  - Alert on suspicious activity
  - Regular audits
  - Encrypt logs
```
