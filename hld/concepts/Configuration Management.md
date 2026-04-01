## 1. What is Configuration Management?

Configuration management is the practice of storing, versioning, and dynamically updating application configuration separate from code.

```
Hardcoded Config (Bad):
  DATABASE_URL = "postgresql://prod-db:5432/mydb"
  API_KEY = "abc123"
  
  Problems:
    - Can't change without redeploying
    - Secrets in code
    - Different per environment

External Config (Good):
  DATABASE_URL = env.get("DATABASE_URL")
  API_KEY = secrets.get("API_KEY")
  
  Benefits:
    - Change without redeploy
    - Secrets separate
    - Environment-specific
```

**Key principle:** Separate configuration from code for flexibility and security.

---

## 2. Types of Configuration

### Environment Variables

```bash
# Set environment variables
export DATABASE_URL="postgresql://localhost:5432/mydb"
export API_KEY="abc123"
export LOG_LEVEL="info"

# Access in code
import os
database_url = os.getenv("DATABASE_URL")
api_key = os.getenv("API_KEY")
```

### Configuration Files

```yaml
# config.yaml
database:
  host: localhost
  port: 5432
  name: mydb

api:
  key: abc123
  timeout: 30

logging:
  level: info
```

### Remote Configuration

```
Centralized config service:
  - AWS AppConfig
  - LaunchDarkly
  - Consul
  - etcd

Benefits:
  - Dynamic updates (no redeploy)
  - Centralized management
  - Audit trail
```

---

## 3. Configuration Hierarchy

### Precedence Order

```
1. Command-line arguments (highest priority)
2. Environment variables
3. Configuration file
4. Default values (lowest priority)

Example:
  Default: LOG_LEVEL = "info"
  Config file: LOG_LEVEL = "debug"
  Env var: LOG_LEVEL = "warn"
  CLI arg: --log-level=error
  
  Result: LOG_LEVEL = "error" (CLI wins)
```

### Environment-Specific Config

```
config/
  default.yaml       # Base config
  development.yaml   # Dev overrides
  staging.yaml       # Staging overrides
  production.yaml    # Prod overrides

Load order:
  1. Load default.yaml
  2. Load environment-specific (e.g., production.yaml)
  3. Merge (environment overrides default)
```

---

## 4. Feature Flags

### What are Feature Flags?

```
Toggle features without deploying:
  if feature_enabled("new_checkout"):
      use_new_checkout()
  else:
      use_old_checkout()

Benefits:
  - Gradual rollout (1% → 10% → 100%)
  - A/B testing
  - Kill switch (disable if broken)
  - Decouple deploy from release
```

### Implementation

```python
class FeatureFlags:
    def __init__(self):
        self.flags = {
            "new_checkout": {"enabled": True, "rollout": 10},  # 10% of users
            "dark_mode": {"enabled": False},
            "premium_features": {"enabled": True, "users": ["user_123"]}
        }
    
    def is_enabled(self, flag_name, user_id=None):
        flag = self.flags.get(flag_name)
        if not flag or not flag.get("enabled"):
            return False
        
        # User-specific
        if "users" in flag:
            return user_id in flag["users"]
        
        # Percentage rollout
        if "rollout" in flag:
            return hash(user_id) % 100 < flag["rollout"]
        
        return True

# Usage
flags = FeatureFlags()
if flags.is_enabled("new_checkout", user_id="user_456"):
    use_new_checkout()
```

---

## 5. Secrets Management

### What are Secrets?

```
Sensitive configuration:
  - Database passwords
  - API keys
  - Encryption keys
  - Certificates

Never:
  ❌ Hardcode in code
  ❌ Commit to git
  ❌ Store in plain text
```

### Secrets Management Tools

```
AWS Secrets Manager:
  - Encrypted storage
  - Automatic rotation
  - Access control (IAM)

HashiCorp Vault:
  - Centralized secrets
  - Dynamic secrets
  - Encryption as a service

Kubernetes Secrets:
  - Base64 encoded
  - Mounted as files or env vars
  - RBAC for access control
```

### Implementation (AWS Secrets Manager)

```python
import boto3
import json

def get_secret(secret_name):
    client = boto3.client('secretsmanager', region_name='us-east-1')
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])

# Usage
db_creds = get_secret('prod/database')
database_url = f"postgresql://{db_creds['username']}:{db_creds['password']}@{db_creds['host']}/mydb"
```

---

## 6. Dynamic Configuration

### Hot Reload

```python
import time
import threading

class DynamicConfig:
    def __init__(self, config_file):
        self.config_file = config_file
        self.config = self.load_config()
        self.start_watcher()
    
    def load_config(self):
        with open(self.config_file) as f:
            return yaml.safe_load(f)
    
    def start_watcher(self):
        def watch():
            last_mtime = os.path.getmtime(self.config_file)
            while True:
                time.sleep(5)
                mtime = os.path.getmtime(self.config_file)
                if mtime > last_mtime:
                    print("Config changed, reloading...")
                    self.config = self.load_config()
                    last_mtime = mtime
        
        thread = threading.Thread(target=watch, daemon=True)
        thread.start()
    
    def get(self, key, default=None):
        return self.config.get(key, default)

# Usage
config = DynamicConfig('config.yaml')
log_level = config.get('log_level', 'info')
```

### Remote Config Service

```python
import requests

class RemoteConfig:
    def __init__(self, config_url):
        self.config_url = config_url
        self.config = {}
        self.refresh()
    
    def refresh(self):
        response = requests.get(self.config_url)
        self.config = response.json()
    
    def get(self, key, default=None):
        return self.config.get(key, default)

# Usage
config = RemoteConfig('https://config-service/api/config')

# Refresh periodically
import schedule
schedule.every(60).seconds.do(config.refresh)
```

---

## 7. Configuration Validation

### Schema Validation

```python
from jsonschema import validate, ValidationError

config_schema = {
    "type": "object",
    "properties": {
        "database": {
            "type": "object",
            "properties": {
                "host": {"type": "string"},
                "port": {"type": "integer", "minimum": 1, "maximum": 65535}
            },
            "required": ["host", "port"]
        },
        "log_level": {
            "type": "string",
            "enum": ["debug", "info", "warn", "error"]
        }
    },
    "required": ["database"]
}

def load_config(config_file):
    with open(config_file) as f:
        config = yaml.safe_load(f)
    
    try:
        validate(instance=config, schema=config_schema)
        return config
    except ValidationError as e:
        print(f"Invalid config: {e.message}")
        sys.exit(1)
```

---

## 8. Best Practices

### 12-Factor App Principles

```
1. Store config in environment
2. Strict separation from code
3. Never commit secrets
4. Environment-specific config
5. Default values for dev

Example:
  # Good
  DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://localhost/dev")
  
  # Bad
  DATABASE_URL = "postgresql://prod-db/mydb"
```

### Configuration as Code

```
Version control configuration:
  - Store in git (except secrets)
  - Review changes (pull requests)
  - Audit trail (git history)
  - Rollback (git revert)

Example:
  config/
    base.yaml
    prod.yaml
    staging.yaml
  
  All in git, reviewed before merge
```

### Immutable Configuration

```
Don't modify config at runtime:
  ❌ config["log_level"] = "debug"
  ✅ Restart with new config

Benefits:
  - Predictable behavior
  - Easier debugging
  - Reproducible
```

---

## 9. Common Interview Questions + Answers

### Q: What is configuration management and why is it important?

> "Configuration management is storing application settings separate from code so you can change them without redeploying. This includes database URLs, API keys, feature flags, and environment-specific settings. It's important because it allows you to deploy the same code to different environments with different configs, change settings dynamically without downtime, keep secrets out of code, and manage configuration centrally. Following the 12-factor app principle, config should be in environment variables or external services, never hardcoded."

### Q: How do you handle secrets in configuration?

> "Never hardcode secrets or commit them to git. Use a secrets management service like AWS Secrets Manager, HashiCorp Vault, or Kubernetes Secrets. These encrypt secrets at rest, provide access control, support automatic rotation, and audit access. In code, fetch secrets at runtime from the secrets service. For local development, use environment variables or a local secrets file that's gitignored. Always use different secrets for each environment and rotate them regularly."

### Q: What are feature flags and when would you use them?

> "Feature flags are toggles that enable or disable features without deploying code. You'd use them for gradual rollouts — enable for 1% of users, monitor, then increase to 100%. For A/B testing to compare different implementations. As kill switches to quickly disable broken features. To decouple deployment from release — deploy code with flag off, enable when ready. And for user-specific features like premium functionality. They're essential for continuous deployment and reducing deployment risk."

### Q: How do you manage configuration across multiple environments?

> "Use a hierarchy with base config and environment-specific overrides. Have a default.yaml with common settings, then production.yaml, staging.yaml, etc. that override specific values. Load base config first, then merge environment-specific config. Use environment variables for secrets and environment-specific values. Consider a remote config service like AWS AppConfig for centralized management. Always validate config on startup to catch errors early. And version control all config files except secrets for audit trail and rollback capability."

---

## 10. Quick Reference

```
What is Configuration Management?
  Store config separate from code
  Version, validate, dynamically update
  Environment-specific settings

Types of config:
  - Environment variables
  - Configuration files (YAML, JSON)
  - Remote config services
  - Feature flags

Hierarchy (precedence):
  1. Command-line args (highest)
  2. Environment variables
  3. Config files
  4. Defaults (lowest)

Feature flags:
  Toggle features without deploy
  Gradual rollout, A/B testing, kill switch

Secrets management:
  - AWS Secrets Manager
  - HashiCorp Vault
  - Kubernetes Secrets
  Never hardcode or commit to git

Dynamic config:
  - Hot reload (watch file changes)
  - Remote config service
  - No redeploy needed

Best practices:
  - 12-factor app (config in environment)
  - Version control (except secrets)
  - Validate on startup
  - Environment-specific overrides
  - Immutable at runtime
  - Default values for dev

Tools:
  - AWS AppConfig
  - LaunchDarkly (feature flags)
  - Consul, etcd (distributed config)
  - Vault (secrets)
```
