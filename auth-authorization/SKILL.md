---
name: auth-authorization
description: Use this skill when implementing login, registration, JWT tokens, sessions, OAuth, API keys, role-based access control, permissions, password hashing, MFA, or when the user asks about authentication, authorization, securing endpoints, RBAC, session management, token refresh, cookie security, or preventing auth-related attacks.
---

# Authentication & Authorization — Backend Engineering

## The Two-Sentence Definitions

**Authentication** = answering "who are you?" — verifying the identity of a subject.
**Authorization** = answering "what can you do?" — checking what an authenticated identity is permitted to do.

They are separate concerns. Authenticate first. Authorize second. Never conflate them.

## Authentication Mechanisms

### 1. JWT (JSON Web Token) — Stateless

Structure: `header.payload.signature` (base64url encoded, dot-separated)

```json
// Header
{ "alg": "HS256", "typ": "JWT" }

// Payload
{
  "sub": "user_id_here",
  "iat": 1686000000,
  "exp": 1686000900,
  "roles": ["user"]
}
```

**Rules:**
- Always verify signature on every request — never trust unverified JWT
- Always check `exp` claim — reject expired tokens
- Never use `alg: none` — explicitly whitelist allowed algorithms
- Short expiry for access tokens: 15 minutes
- Separate refresh tokens with longer expiry: 7–30 days
- Never store sensitive data in JWT payload — it's base64 encoded, not encrypted, anyone can read it
- Use RS256 (asymmetric) if multiple services need to verify tokens
- Use HS256 (symmetric) only if a single service both issues and verifies

**Refresh Token Pattern:**
```
1. Access token expires (15 min)
2. Client sends refresh token to /auth/refresh
3. Server validates refresh token from database
4. Server issues new access token + new refresh token (rotation)
5. Old refresh token is invalidated immediately
6. If old refresh token is used after rotation → security incident → revoke entire session
```

### 2. Session-Based — Stateful

```
1. User logs in → server creates session in DB/Redis
2. Server sends session ID in cookie (Secure, HttpOnly, SameSite=Strict)
3. Client sends cookie on every request
4. Server looks up session by ID to identify user
5. Logout → delete session from server
```

Session storage: Redis is preferred (fast, TTL support). Database for persistence.

**Cookie attributes — all required:**
```
Set-Cookie: session_id=abc; Secure; HttpOnly; SameSite=Strict; Path=/; Max-Age=86400
```
- `Secure` — only sent over HTTPS
- `HttpOnly` — inaccessible to JavaScript (prevents XSS token theft)
- `SameSite=Strict` — not sent on cross-site requests (CSRF protection)

### 3. API Keys — Machine-to-Machine

```
Authorization: ApiKey your-api-key-here
# OR
X-API-Key: your-api-key-here
```

**Implementation:**
- Generate: cryptographically random 32+ bytes, base64url encoded
- Store: hash of the key (bcrypt or SHA-256 with salt) — never plaintext
- Transmit: only over HTTPS
- Scope: each key has defined permissions
- Rotation: allow multiple active keys so rotation doesn't cause downtime
- Audit: log every request with key ID (not the key itself)

### 4. OAuth 2.0 — Delegated Authorization

Four flows:

| Flow | Use Case |
|------|----------|
| Authorization Code + PKCE | Web apps, mobile apps (current standard) |
| Client Credentials | Server-to-server, no user involved |
| Refresh Token | Obtaining new access token |
| ~~Implicit~~ | Deprecated — do not use |

**Authorization Code + PKCE flow:**
```
1. Client generates code_verifier (random), code_challenge = SHA256(code_verifier)
2. Redirect user to /oauth/authorize?code_challenge=...&state=random_csrf_value
3. User authenticates and consents
4. Auth server redirects back with code
5. Client verifies state matches (CSRF check)
6. Client exchanges code + code_verifier for tokens at /oauth/token
```

Always validate `state` parameter to prevent CSRF attacks on OAuth.

## Password Security

### Hashing — The Only Correct Approach

Never store plaintext passwords. Never use MD5, SHA-1, SHA-256 alone — they're too fast.

Use one of:
- **bcrypt**: cost factor 12 (auto-upgrades with hardware)
- **argon2id**: memory-hard, current best practice
- **scrypt**: memory-hard, acceptable

```
# bcrypt with cost 12
hash = bcrypt.hash(password, rounds=12)
verify = bcrypt.verify(input_password, stored_hash)
```

### Password Requirements
- Minimum 8 characters (NIST recommends 12+)
- Maximum 128 characters (prevent DoS on bcrypt)
- No complexity requirements are mandatory (NIST 2017) — length > complexity
- Check against HaveIBeenPwned API for known breached passwords
- Trim whitespace: NO. Preserve the user's password as-is.

### Timing-Safe Comparison

Always use constant-time comparison for secrets — prevents timing attacks:

```python
# Wrong — timing attack possible
if stored_hash == computed_hash:

# Correct — constant time
import hmac
if hmac.compare_digest(stored_hash, computed_hash):
```

## Account Security

### Rate Limiting Auth Endpoints
- Login: 10 attempts per IP per 15 minutes
- Registration: 5 per IP per hour
- Password reset: 3 per email per hour
- OTP verification: 5 attempts before invalidating OTP

### Account Lockout After Failed Attempts
- Lock after 5 consecutive failures within 15 minutes
- Lockout duration: 30 minutes (progressive: 30min → 2h → 24h)
- Unlock via email link or time expiry
- Alert user when account is locked

### Error Messages — Never Distinguish
```
# Wrong — reveals whether email exists
"No account found with this email"
"Incorrect password"

# Correct — same message for both
"Invalid email or password"
```

## Authorization Patterns

### RBAC — Role-Based Access Control

Assign roles to users. Assign permissions to roles.

```
User → has Role → Role has Permissions
admin → ["users:read", "users:write", "users:delete", "orders:read", "orders:write"]
user  → ["orders:read", "profile:read", "profile:write"]
```

Check in service layer — not just at route level:
```python
def update_order(user, order_id, data):
    if not user.has_permission("orders:write"):
        raise ForbiddenError()
    order = order_repo.find(order_id)
    if order.user_id != user.id and not user.has_role("admin"):
        raise ForbiddenError()   # horizontal check
```

### ABAC — Attribute-Based Access Control

More granular: permissions based on user attributes + resource attributes + environment.

```
user.department == resource.department AND action == "read"
user.clearance_level >= resource.classification_level
time_of_day BETWEEN 09:00 AND 17:00
```

Use ABAC when RBAC becomes too complex to manage (too many role combinations).

### Horizontal Authorization (Ownership Check)

Always verify the user owns the resource they're trying to access — not just that they're authenticated.

```python
order = order_repo.find(order_id)

# WRONG — only checks authentication, not ownership
return order

# CORRECT — checks ownership
if order.user_id != current_user.id and not current_user.is_admin():
    raise ForbiddenError("You don't have access to this order")
return order
```

## Multi-Factor Authentication (MFA)

Implementation steps:
1. User enables MFA → generate TOTP secret, return QR code
2. User scans with authenticator app
3. User verifies with first code → MFA activated
4. On login: after password success → require TOTP code
5. TOTP code valid for 30 seconds ± 1 window (allow 1 window drift for clock skew)
6. Backup codes: generate 10 one-time codes at MFA setup

Libraries: use standard TOTP (RFC 6238). Do not build your own.

## Audit Logging — Mandatory

Log every:
- Successful login (user_id, IP, device, timestamp)
- Failed login attempt (email attempted, IP, timestamp)
- Logout
- Password change
- MFA enable/disable
- Permission change
- Token refresh
- Account lockout

Never log passwords, tokens, or secrets. Log the event, not the credential.

## Common Vulnerabilities and Mitigations

| Attack | Mitigation |
|--------|------------|
| Brute force | Rate limiting + account lockout |
| Credential stuffing | Rate limiting + breach password check + MFA |
| JWT forgery | Always verify signature, whitelist algorithms |
| Session fixation | Regenerate session ID after login |
| CSRF | SameSite cookies + CSRF tokens |
| XSS token theft | HttpOnly cookies for session, short-lived JWTs |
| Timing attack | Constant-time comparison |
| Privilege escalation | Server-side authorization checks on every operation |
| Replay attack | Short token TTL + nonce for sensitive operations |
