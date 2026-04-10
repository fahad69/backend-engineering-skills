---
name: backend-security
description: Use this skill when implementing security measures, preventing SQL injection, XSS, CSRF, setting security headers, handling rate limiting, securing APIs, or when the user asks about OWASP, injection attacks, access control, data protection, dependency vulnerabilities, API security, or how to harden a backend system.
---

# Backend Security — Backend Engineering

## Security is a Default, Not a Feature

Security is not something you add at the end. It is a requirement woven into every design decision from the start. Every time you write code that touches user input, network traffic, authentication, or data storage — think about security.

## OWASP Top 10 — Know These

1. Broken Access Control
2. Cryptographic Failures
3. Injection (SQL, NoSQL, Command, LDAP)
4. Insecure Design
5. Security Misconfiguration
6. Vulnerable and Outdated Components
7. Identification and Authentication Failures
8. Software and Data Integrity Failures
9. Security Logging and Monitoring Failures
10. Server-Side Request Forgery (SSRF)

## Injection Attacks

### SQL Injection — Never Use String Interpolation in Queries

```python
# WRONG — critically vulnerable
query = f"SELECT * FROM users WHERE email = '{email}'"
# Attacker sends: ' OR '1'='1
# Becomes: SELECT * FROM users WHERE email = '' OR '1'='1'
# Returns all users

# CORRECT — parameterized query
query = "SELECT * FROM users WHERE email = $1"
db.execute(query, [email])
```

This is non-negotiable. Every query uses parameterized statements. No exceptions.

### NoSQL Injection

```javascript
// Wrong — MongoDB injection
db.users.find({ email: req.body.email })
// Attacker sends: { "$ne": null }
// Returns all users

// Correct — validate type before use
if (typeof req.body.email !== 'string') {
  throw new ValidationError("email must be a string")
}
db.users.find({ email: req.body.email })
```

### Command Injection

```python
# WRONG — never pass user input to shell
os.system(f"convert {filename} output.jpg")
# Attacker sends: file.jpg; rm -rf /

# CORRECT — use API, not shell
from PIL import Image
img = Image.open(filename)
img.save("output.jpg")

# If shell is unavoidable — whitelist inputs
allowed_formats = ["jpg", "png", "webp"]
if extension not in allowed_formats:
    raise ValidationError("Unsupported format")
subprocess.run(["convert", filename, "output.jpg"])  # array form, not string
```

## Cross-Site Scripting (XSS)

XSS happens when user input is rendered in HTML without escaping.

```python
# Wrong — renders user input as HTML
return f"<div>Hello {user_name}</div>"
# Attacker name: <script>document.cookie</script>

# Correct — escape output for HTML context
return f"<div>Hello {html.escape(user_name)}</div>"
```

Context-specific escaping:
- HTML body: escape `<`, `>`, `&`, `"`, `'`
- HTML attribute: escape `"`, `'`, `>`
- JavaScript context: JSON encode
- URL context: percent-encode

Content Security Policy as defense-in-depth:
```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'
```

## CSRF — Cross-Site Request Forgery

CSRF tricks a user's browser into making requests to your API on behalf of the attacker.

### Defense 1: SameSite Cookie Attribute
```
Set-Cookie: session=abc; SameSite=Strict; Secure; HttpOnly
```
`SameSite=Strict`: cookie not sent on cross-site requests (strongest protection).
`SameSite=Lax`: cookie sent on top-level navigation, not on POST.

### Defense 2: CSRF Token (Double Submit)
```python
# On page load — generate token
csrf_token = secrets.token_hex(32)
session['csrf_token'] = csrf_token

# On form submit — validate
if request.form.get('csrf_token') != session.get('csrf_token'):
    raise ForbiddenError("CSRF validation failed")
```

### Defense 3: Check Origin/Referer Headers
```python
allowed_origins = ["https://yourapp.com", "https://www.yourapp.com"]
if request.headers.get('Origin') not in allowed_origins:
    raise ForbiddenError("Invalid origin")
```

## Security Headers — Set on Every Response

```
# Mandatory
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'; img-src 'self' data:; style-src 'self' 'unsafe-inline'
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=()

# Remove these (never reveal your stack)
X-Powered-By: (remove it)
Server: (remove or set generic value)
```

## Rate Limiting

Every public endpoint needs rate limiting. Auth endpoints need aggressive limits.

```
Global API:           100 requests per user per minute
Search endpoints:     30 requests per user per minute
Login:                10 attempts per IP per 15 minutes
Registration:         5 per IP per hour
Password reset:       3 per email per hour
OTP verification:     5 attempts per session
Payment endpoints:    5 per user per minute
```

Response on limit exceeded:
```
HTTP 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1686001200
```

Store rate limit state in Redis (not in-process memory — useless with multiple instances).

## Access Control

### Principle of Least Privilege

Every component gets minimum permissions:
- DB user: only SELECT, INSERT, UPDATE, DELETE — not DROP, ALTER
- API keys: scoped to specific operations only
- Service accounts: only what they need
- Users: only their own resources

### Server-Side Authorization Always

Client-side hiding is not access control. A direct API call bypasses all UI restrictions.

```python
# Always check on every request
def get_order(request, order_id):
    order = order_repo.find(order_id)
    
    # Check ownership AND role
    if order.user_id != request.user.id and not request.user.has_role("admin"):
        raise ForbiddenError()
    
    return order
```

### IDOR — Insecure Direct Object Reference

```
# Vulnerable
GET /api/orders/12345
# Any authenticated user can see any order by guessing IDs

# Protected — check ownership
GET /api/orders/12345
→ verify order belongs to requesting user before returning
```

Use UUIDs instead of sequential IDs to make ID guessing harder (but still enforce authorization).

## Sensitive Data Protection

### Encryption at Rest
```python
# Passport numbers, SSN, financial data — encrypt before storing
from cryptography.fernet import Fernet

key = config.field_encryption_key  # from secrets manager, never hardcoded
cipher = Fernet(key)

# Encrypt before storing
encrypted_passport = cipher.encrypt(passport_number.encode())
db.execute("UPDATE users SET passport_number = $1", [encrypted_passport])

# Decrypt when needed
raw_passport = cipher.decrypt(stored_encrypted_value).decode()
```

### PII in APIs — Return Minimal Data
```json
// Wrong — full passport number in response
{ "passport_number": "P12345678" }

// Correct — masked
{ "passport_number": "P*****678" }
```

### HTTPS — Non-Negotiable
- TLS 1.2 minimum, TLS 1.3 preferred
- All HTTP redirects to HTTPS (301)
- HSTS enforced
- Certificate expiry monitored (alert 30 days before)

## Dependency Security

```bash
# Check for vulnerable dependencies
npm audit
pip-audit
bundle audit  # Ruby
govulncheck   # Go

# Run in CI pipeline — fail build on high/critical vulnerabilities
npm audit --audit-level=high

# Keep dependencies updated
npm outdated
pip list --outdated
```

Never use `latest` version pins in production. Pin exact versions in lockfiles.

## Security Logging

Log all security-relevant events:

```python
# Log these:
log.info("Authentication", event="login_success", user_id=user.id, ip=ip)
log.warn("Authentication", event="login_failure", email=email, ip=ip, attempt=count)
log.warn("Authentication", event="account_locked", user_id=user.id, ip=ip)
log.warn("Authorization", event="permission_denied", user_id=user.id, resource=resource)
log.warn("Rate limit", event="limit_exceeded", user_id=user_id, endpoint=endpoint)
log.info("Password", event="password_changed", user_id=user.id)
log.info("MFA", event="mfa_enabled", user_id=user.id)
log.warn("Suspicious", event="multiple_failed_payments", user_id=user.id)
```

Never log:
- Passwords or hashes
- Tokens (JWT, session, API key)
- Card numbers, CVV
- Passport numbers, SSN

## JWT Security Checklist

- [ ] Always verify signature before trusting any claim
- [ ] Never use `alg: none`
- [ ] Explicitly whitelist allowed algorithms (`RS256` or `HS256`)
- [ ] Always check `exp` claim — reject expired tokens
- [ ] Keep access tokens short-lived (15 minutes)
- [ ] Refresh tokens stored hashed in database
- [ ] Refresh token rotation on every use
- [ ] JWT payload contains no sensitive data (it's base64, not encrypted)

## API Key Security Checklist

- [ ] Generated with cryptographic randomness (not UUID or timestamp)
- [ ] Stored as hash (bcrypt or SHA-256 + salt) — never plaintext
- [ ] Only transmitted over HTTPS
- [ ] Scoped to minimum required permissions
- [ ] Expiry supported (or manual rotation enforced)
- [ ] Multiple active keys supported (for zero-downtime rotation)
- [ ] Revocation instant
- [ ] Usage logged with key ID (not the key value)
- [ ] Key shown only once at creation — never again
