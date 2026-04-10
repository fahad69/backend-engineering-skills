---
name: config-management
description: Use this skill when setting up configuration management, managing environment variables, working with secrets, feature flags, or when the user asks about separating config from code, managing different environments, secrets management, configuration validation, hot reloading config, or production-safe configuration practices.
---

# Configuration Management — Backend Engineering

## Core Rule: Separate Config from Code

Config is anything that changes between environments (dev, staging, production). Code is everything that doesn't.

The test: can you open-source your codebase right now without exposing any secrets? If no, your config is in your code. Fix it.

## What Is Configuration

Not just database passwords. Configuration includes:
- Database connection strings and credentials
- External API keys and secrets (Sabre, Stripe, SendGrid, etc.)
- Service endpoints (different per environment)
- Feature flags (enable/disable features without deploying)
- Rate limit thresholds (configurable per environment)
- Timeout values (lower in dev, higher in prod)
- Log levels (DEBUG in dev, WARN in prod)
- Cache TTL values
- Connection pool sizes
- Email sender addresses
- Base URLs and allowed origins

## Configuration Sources (Priority Order, Highest First)

```
1. Environment variables        ← runtime, highest priority, overrides all
2. Secrets manager (Vault, AWS Secrets Manager)  ← for sensitive values
3. Config files (.env.local, config.yaml)  ← per-environment, not committed for secrets
4. Default values in code       ← lowest priority, safe defaults only
```

At startup: merge all sources, highest priority wins.

## Never Commit Secrets to Version Control

```bash
# .gitignore — always include:
.env
.env.local
.env.production
*.pem
*.key
secrets.yaml
config.production.json
```

Use `.env.example` (committed) to document what variables are required:
```bash
# .env.example — committed to repo, no real values
DATABASE_URL=postgresql://user:password@host:5432/dbname
SABRE_CLIENT_ID=your_sabre_client_id_here
SABRE_CLIENT_SECRET=your_sabre_client_secret_here
JWT_SECRET=your_jwt_secret_here
STRIPE_SECRET_KEY=sk_live_your_key_here
```

## Configuration Validation at Startup

Fail fast on missing or invalid config. Do not start the server with broken config.

```python
class Config:
    def __init__(self):
        self.database_url = self.require("DATABASE_URL")
        self.jwt_secret = self.require("JWT_SECRET")
        self.sabre_client_id = self.require("SABRE_CLIENT_ID")
        self.sabre_client_secret = self.require("SABRE_CLIENT_SECRET")
        self.port = self.require_int("PORT", default=8000)
        self.log_level = self.require_enum("LOG_LEVEL", ["DEBUG","INFO","WARN","ERROR"], default="INFO")
        self.redis_url = self.require("REDIS_URL")
        self.environment = self.require_enum("APP_ENV", ["development","staging","production"])
    
    def require(self, key):
        value = os.getenv(key)
        if not value:
            raise ConfigError(f"Required environment variable '{key}' is missing")
        return value
    
    def require_int(self, key, default=None):
        value = os.getenv(key, str(default) if default is not None else None)
        if value is None:
            raise ConfigError(f"Required environment variable '{key}' is missing")
        try:
            return int(value)
        except ValueError:
            raise ConfigError(f"Environment variable '{key}' must be an integer, got: {value}")

# Initialize config at startup — crashes immediately if invalid
config = Config()
```

## Environment Separation

Never use `if env == 'production':` scattered across your codebase. All environment differences go through config.

```
Config for development:
  LOG_LEVEL=DEBUG
  SABRE_ENVIRONMENT=test
  STRIPE_SECRET_KEY=sk_test_...
  SEND_REAL_EMAILS=false
  RATE_LIMIT_REQUESTS=1000  # relaxed in dev

Config for production:
  LOG_LEVEL=WARN
  SABRE_ENVIRONMENT=production
  STRIPE_SECRET_KEY=sk_live_...
  SEND_REAL_EMAILS=true
  RATE_LIMIT_REQUESTS=100
```

## Secrets Management (Production)

Never put production secrets in environment files. Use a secrets manager:

| Tool | Best For |
|------|----------|
| AWS Secrets Manager | AWS deployments |
| HashiCorp Vault | Multi-cloud, self-hosted |
| GCP Secret Manager | GCP deployments |
| Azure Key Vault | Azure deployments |
| Doppler | Simple, developer-friendly |

```python
# Fetch secret at startup (not on every request)
import boto3

def get_secret(name):
    client = boto3.client('secretsmanager', region_name='us-east-1')
    response = client.get_secret_value(SecretId=name)
    return json.loads(response['SecretString'])

secrets = get_secret('prod/myapp/database')
database_url = secrets['url']
```

## Feature Flags

Feature flags let you enable/disable features without deploying code.

```python
class FeatureFlags:
    def __init__(self, config):
        self.new_payment_ui = config.get_bool("FEATURE_NEW_PAYMENT_UI", default=False)
        self.vacation_rentals = config.get_bool("FEATURE_VACATION_RENTALS", default=True)
        self.ai_recommendations = config.get_bool("FEATURE_AI_RECOMMENDATIONS", default=False)

# Usage
if feature_flags.vacation_rentals:
    include_vacation_rentals()
```

For advanced feature flags (percentage rollouts, user targeting): use LaunchDarkly, Unleash, or Flipt.

```python
# Percentage rollout: 10% of users get the new feature
def is_new_feature_enabled(user_id):
    return int(hashlib.md5(user_id.encode()).hexdigest(), 16) % 100 < 10
```

## Configuration Never Logged

```python
# Wrong — logs the secret
log.info(f"Connecting to database: {config.database_url}")

# Correct — log that config loaded, not the values
log.info("Database config loaded", host=config.db_host, name=config.db_name)
# Never log password, connection string, API key, or JWT secret
```

## Dynamic Configuration (Hot Reload)

For configs that change without restart:
```python
# Poll remote config every 60 seconds
def config_refresh_worker():
    while True:
        new_rate_limit = fetch_remote_config("rate_limit")
        if new_rate_limit != current_config.rate_limit:
            current_config.rate_limit = new_rate_limit
            log.info("Rate limit config updated", new_value=new_rate_limit)
        time.sleep(60)
```

Use remote config for:
- Rate limits
- Feature flags
- Timeout values
- Log levels (change verbosity in production without restart)

Do NOT use remote config for:
- Database credentials (require restart to rotate safely)
- JWT secrets (require restart to take effect)

## Configuration Documentation

Every config variable must be documented:

```
DATABASE_URL
  Type:     string (PostgreSQL connection string)
  Required: yes
  Example:  postgresql://user:password@localhost:5432/myapp
  Notes:    Must include SSL mode in production

SABRE_SESSION_POOL_SIZE
  Type:     integer
  Required: no
  Default:  10
  Range:    1-50
  Notes:    Higher values use more memory but handle more concurrent searches

FEATURE_VACATION_RENTALS
  Type:     boolean (true/false)
  Required: no
  Default:  false
  Notes:    Enable vacation rental vertical. Requires ES_URL to be configured.
```

Document in `.env.example` with comments and in your README.
