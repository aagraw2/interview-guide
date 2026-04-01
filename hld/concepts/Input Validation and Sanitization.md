## 1. Why Input Validation Matters

**Never trust user input.** Every input from users, APIs, or external systems is a potential attack vector.

```
Common attacks from untrusted input:
  - SQL Injection
  - Cross-Site Scripting (XSS)
  - Command Injection
  - Path Traversal
  - XML External Entity (XXE)
  - Server-Side Request Forgery (SSRF)
  - Buffer Overflow
```

**Principle:** Validate at the boundary. Don't trust data from clients, even if you control the client (mobile apps can be reverse-engineered).

---

## 2. Validation vs Sanitization

### Validation

Check if input meets expected format. Reject if invalid.

```python
# Validate email format
import re

def is_valid_email(email):
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return re.match(pattern, email) is not None

email = request.form['email']
if not is_valid_email(email):
    return 400, "Invalid email format"
```

---

### Sanitization

Clean input by removing or escaping dangerous characters.

```python
# Sanitize HTML input (remove script tags)
from bleach import clean

user_input = "<script>alert('XSS')</script>Hello"
sanitized = clean(user_input, tags=[], strip=True)
# Result: "Hello"
```

**Best practice:** Validate first (reject bad input), sanitize as a fallback (clean input that's mostly valid).

---

## 3. SQL Injection

### The Attack

Attacker injects SQL code into input fields.

```python
# VULNERABLE CODE
username = request.form['username']
password = request.form['password']

query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"
db.execute(query)
```

**Attack:**

```
username: admin' --
password: anything

Resulting query:
SELECT * FROM users WHERE username = 'admin' -- ' AND password = 'anything'

The -- comments out the rest of the query
→ Logs in as admin without password
```

---

### Defense: Parameterized Queries

Use placeholders instead of string concatenation.

```python
# SAFE CODE
query = "SELECT * FROM users WHERE username = ? AND password = ?"
db.execute(query, (username, password))
```

**How it works:** The database treats `?` as data, not code. Even if `username` contains SQL syntax, it's treated as a literal string.

---

### Defense: ORM

Use an ORM (Object-Relational Mapper) that handles parameterization automatically.

```python
# Django ORM (safe)
user = User.objects.filter(username=username, password=password).first()

# SQLAlchemy (safe)
user = session.query(User).filter_by(username=username, password=password).first()
```

---

### Defense: Input Validation

Validate input format before querying.

```python
# Only allow alphanumeric usernames
if not username.isalnum():
    return 400, "Invalid username"
```

**Note:** Validation alone is not enough. Always use parameterized queries.

---

## 4. Cross-Site Scripting (XSS)

### The Attack

Attacker injects JavaScript into web pages viewed by other users.

```html
<!-- VULNERABLE CODE -->
<div>Welcome, {{ username }}</div>

<!-- If username = "<script>alert('XSS')</script>" -->
<div>Welcome, <script>alert('XSS')</script></div>
<!-- Script executes in victim's browser -->
```

**Impact:** Steal cookies, session tokens, redirect to phishing sites, deface pages.

---

### Types of XSS

```
Stored XSS:
  - Attacker stores malicious script in database (e.g., comment, profile)
  - Victim views page → script executes
  - Most dangerous (persistent)

Reflected XSS:
  - Attacker sends malicious link with script in URL
  - Victim clicks link → server reflects script in response → script executes
  - Example: https://example.com/search?q=<script>alert('XSS')</script>

DOM-based XSS:
  - Client-side JavaScript reads untrusted data (URL, localStorage)
  - Inserts data into DOM without sanitization
  - Example: document.write(location.hash)
```

---

### Defense: Output Encoding

Escape HTML special characters before rendering.

```python
# Python (Jinja2 auto-escapes by default)
<div>Welcome, {{ username }}</div>
<!-- If username = "<script>alert('XSS')</script>" -->
<!-- Rendered as: Welcome, &lt;script&gt;alert('XSS')&lt;/script&gt; -->
<!-- Browser displays: <script>alert('XSS')</script> (as text, not code) -->
```

**HTML entities:**

```
<  → &lt;
>  → &gt;
"  → &quot;
'  → &#x27;
&  → &amp;
```

---

### Defense: Content Security Policy (CSP)

HTTP header that restricts where scripts can be loaded from.

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.cdn.com
```

**Effect:** Blocks inline scripts and scripts from untrusted domains.

```html
<!-- Blocked by CSP -->
<script>alert('XSS')</script>

<!-- Allowed (from same origin) -->
<script src="/app.js"></script>

<!-- Allowed (from trusted CDN) -->
<script src="https://trusted.cdn.com/lib.js"></script>
```

---

### Defense: Sanitize HTML Input

If you must allow HTML (e.g., rich text editor), sanitize it.

```python
from bleach import clean

user_input = "<script>alert('XSS')</script><p>Hello</p>"
sanitized = clean(user_input, tags=['p', 'b', 'i', 'a'], attributes={'a': ['href']})
# Result: "<p>Hello</p>"
```

**Allowlist approach:** Only allow safe tags and attributes.

---

## 5. Command Injection

### The Attack

Attacker injects shell commands into input that's passed to the OS.

```python
# VULNERABLE CODE
filename = request.form['filename']
os.system(f"cat {filename}")
```

**Attack:**

```
filename: file.txt; rm -rf /

Resulting command:
cat file.txt; rm -rf /
→ Deletes all files on the system
```

---

### Defense: Avoid Shell Commands

Use language-native APIs instead of shell commands.

```python
# SAFE CODE
with open(filename, 'r') as f:
    content = f.read()
```

---

### Defense: Parameterized Commands

If you must use shell commands, use parameterized APIs.

```python
# SAFE CODE
import subprocess

subprocess.run(['cat', filename])  # filename is passed as argument, not interpolated
```

---

### Defense: Input Validation

Validate input format.

```python
# Only allow alphanumeric filenames
if not re.match(r'^[a-zA-Z0-9_.-]+$', filename):
    return 400, "Invalid filename"
```

---

## 6. Path Traversal

### The Attack

Attacker uses `../` to access files outside the intended directory.

```python
# VULNERABLE CODE
filename = request.args['file']
with open(f"/var/www/uploads/{filename}", 'r') as f:
    return f.read()
```

**Attack:**

```
file: ../../../../etc/passwd

Resulting path:
/var/www/uploads/../../../../etc/passwd
→ /etc/passwd
→ Reads system password file
```

---

### Defense: Validate Path

Ensure the resolved path is within the allowed directory.

```python
import os

base_dir = "/var/www/uploads"
filename = request.args['file']
filepath = os.path.join(base_dir, filename)

# Resolve to absolute path
filepath = os.path.abspath(filepath)

# Check if path is within base_dir
if not filepath.startswith(base_dir):
    return 403, "Access denied"

with open(filepath, 'r') as f:
    return f.read()
```

---

### Defense: Allowlist Filenames

Only allow specific filenames.

```python
allowed_files = ['report.pdf', 'invoice.pdf']
if filename not in allowed_files:
    return 403, "Access denied"
```

---

## 7. Server-Side Request Forgery (SSRF)

### The Attack

Attacker tricks the server into making requests to internal systems.

```python
# VULNERABLE CODE
url = request.form['url']
response = requests.get(url)
return response.content
```

**Attack:**

```
url: http://169.254.169.254/latest/meta-data/iam/security-credentials/

→ Server fetches AWS metadata (credentials)
→ Attacker steals credentials
```

---

### Defense: Allowlist URLs

Only allow requests to trusted domains.

```python
from urllib.parse import urlparse

allowed_domains = ['api.example.com', 'cdn.example.com']
parsed = urlparse(url)

if parsed.hostname not in allowed_domains:
    return 403, "Access denied"

response = requests.get(url)
```

---

### Defense: Block Internal IPs

Prevent requests to private IP ranges.

```python
import ipaddress

def is_internal_ip(hostname):
    try:
        ip = ipaddress.ip_address(hostname)
        return ip.is_private or ip.is_loopback
    except ValueError:
        return False

if is_internal_ip(parsed.hostname):
    return 403, "Access denied"
```

**Private IP ranges:**

```
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
127.0.0.0/8 (loopback)
169.254.0.0/16 (link-local, AWS metadata)
```

---

## 8. XML External Entity (XXE)

### The Attack

Attacker injects malicious XML to read files or make requests.

```xml
<!-- VULNERABLE CODE -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<user>
  <name>&xxe;</name>
</user>
```

**Result:** Server reads `/etc/passwd` and includes it in the response.

---

### Defense: Disable External Entities

Configure XML parser to reject external entities.

```python
import xml.etree.ElementTree as ET

# SAFE CODE
parser = ET.XMLParser()
parser.entity = {}  # Disable entity expansion
tree = ET.fromstring(xml_data, parser=parser)
```

**Better:** Use JSON instead of XML (no XXE risk).

---

## 9. Input Validation Best Practices

### 1. Validate at the Boundary

Validate input as soon as it enters the system (API gateway, controller).

```python
@app.route('/user', methods=['POST'])
def create_user():
    # Validate immediately
    email = request.json.get('email')
    if not is_valid_email(email):
        return 400, "Invalid email"
    
    # Proceed with business logic
    user = User(email=email)
    db.save(user)
```

---

### 2. Allowlist, Not Blocklist

Define what's allowed, not what's forbidden.

```python
# BAD: Blocklist (easy to bypass)
if '<script>' in user_input or 'DROP TABLE' in user_input:
    return 400, "Invalid input"

# GOOD: Allowlist (only allow expected format)
if not re.match(r'^[a-zA-Z0-9]+$', user_input):
    return 400, "Invalid input"
```

---

### 3. Validate Type, Length, Format, Range

```python
# Type
age = request.json.get('age')
if not isinstance(age, int):
    return 400, "Age must be an integer"

# Range
if age < 0 or age > 150:
    return 400, "Age must be between 0 and 150"

# Length
username = request.json.get('username')
if len(username) < 3 or len(username) > 20:
    return 400, "Username must be 3-20 characters"

# Format
if not re.match(r'^[a-zA-Z0-9_]+$', username):
    return 400, "Username must be alphanumeric"
```

---

### 4. Validate on Server, Not Just Client

Client-side validation improves UX but doesn't provide security (can be bypassed).

```javascript
// Client-side validation (UX only)
if (!email.includes('@')) {
    alert('Invalid email');
    return;
}

// MUST also validate on server
```

---

### 5. Use Schema Validation

Define expected input structure and validate against it.

```python
from jsonschema import validate, ValidationError

schema = {
    "type": "object",
    "properties": {
        "email": {"type": "string", "format": "email"},
        "age": {"type": "integer", "minimum": 0, "maximum": 150}
    },
    "required": ["email", "age"]
}

try:
    validate(instance=request.json, schema=schema)
except ValidationError as e:
    return 400, str(e)
```

---

## 10. Common Interview Questions + Answers

### Q: How do you prevent SQL injection?

> "Use parameterized queries or prepared statements. Never concatenate user input into SQL strings. The database treats parameters as data, not code, so even if the input contains SQL syntax, it's treated as a literal string. I'd also validate input format — for example, if expecting a numeric ID, ensure it's actually a number. Using an ORM like Django or SQLAlchemy also helps because they handle parameterization automatically."

---

### Q: What's the difference between validation and sanitization?

> "Validation checks if input meets expected format and rejects it if invalid. Sanitization cleans input by removing or escaping dangerous characters. Best practice is to validate first — reject input that doesn't match the expected format. Sanitize as a fallback for input that's mostly valid but contains some unsafe characters. For example, validate that an email matches the email format, but sanitize HTML input by removing script tags."

---

### Q: How do you prevent XSS?

> "Output encoding is the primary defense — escape HTML special characters before rendering user input. Most modern frameworks like React and Jinja2 do this automatically. I'd also use Content Security Policy headers to block inline scripts and restrict script sources. If the app allows HTML input, like a rich text editor, I'd sanitize it with an allowlist approach — only allow safe tags like p, b, i, and strip everything else. Finally, validate input on the server, not just the client."

---

### Q: What's SSRF and how do you prevent it?

> "Server-Side Request Forgery is when an attacker tricks the server into making requests to internal systems, like AWS metadata endpoints or internal APIs. To prevent it, I'd allowlist the domains the server is allowed to request. I'd also block requests to private IP ranges like 10.0.0.0/8, 192.168.0.0/16, and 169.254.169.254 (AWS metadata). If possible, I'd use a separate service or proxy for external requests with strict network policies."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Always mention parameterized queries for SQL injection

This is the gold standard defense. Mentioning it immediately shows you know the right approach.

---

### ✅ Trick 2: Distinguish validation vs sanitization

Many candidates conflate these. Clearly state: validate first (reject bad input), sanitize as fallback.

---

### ✅ Trick 3: Mention defense in depth

No single defense is perfect. Combine validation, sanitization, output encoding, CSP, etc.

---

### ❌ Pitfall 1: Relying on client-side validation

Client-side validation is for UX, not security. Always validate on the server.

---

### ❌ Pitfall 2: Using blocklists

Blocklists are easy to bypass. Always use allowlists (define what's allowed, not what's forbidden).

---

### ❌ Pitfall 3: Forgetting output encoding

Validating input is not enough for XSS. You must also encode output when rendering.

---

## 12. Quick Reference

```
Input Validation: Check if input meets expected format, reject if invalid
Sanitization: Clean input by removing/escaping dangerous characters

Common Attacks:
  SQL Injection:     Inject SQL code into queries
  XSS:               Inject JavaScript into web pages
  Command Injection: Inject shell commands
  Path Traversal:    Access files outside intended directory (../)
  SSRF:              Trick server into requesting internal systems
  XXE:               Inject malicious XML to read files

Defenses:

SQL Injection:
  ✅ Parameterized queries (?, placeholders)
  ✅ ORM (Django, SQLAlchemy)
  ✅ Input validation (type, format)

XSS:
  ✅ Output encoding (escape HTML: <, >, ", ', &)
  ✅ Content Security Policy (CSP)
  ✅ Sanitize HTML input (allowlist tags)

Command Injection:
  ✅ Avoid shell commands (use native APIs)
  ✅ Parameterized commands (subprocess.run(['cmd', arg]))
  ✅ Input validation (alphanumeric only)

Path Traversal:
  ✅ Validate resolved path is within allowed directory
  ✅ Allowlist filenames

SSRF:
  ✅ Allowlist domains
  ✅ Block private IPs (10.0.0.0/8, 192.168.0.0/16, 169.254.169.254)

XXE:
  ✅ Disable external entities in XML parser
  ✅ Use JSON instead of XML

Best Practices:
  ✅ Validate at the boundary (API gateway, controller)
  ✅ Allowlist, not blocklist
  ✅ Validate type, length, format, range
  ✅ Validate on server, not just client
  ✅ Use schema validation (JSON Schema)
  ✅ Defense in depth (multiple layers)

Validation Checklist:
  - Type (string, int, bool)
  - Length (min, max)
  - Format (regex, email, URL)
  - Range (min, max for numbers)
  - Allowlist (only expected values)
```
