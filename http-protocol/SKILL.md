---
name: http-protocol
description: Use this skill when building any backend HTTP layer, designing APIs, setting status codes, working with headers, handling CORS, implementing caching headers, or when the user asks about HTTP methods, status codes, request/response structure, HTTP versions, or HTTPS. Triggers on questions about HTTP, REST transport, headers, CORS, SSL/TLS, keep-alive, compression, content negotiation.
---

# HTTP Protocol — Backend Engineering

## Core Mental Model

HTTP is stateless. Every request is independent. The server holds no memory of previous requests unless you explicitly build that (sessions, tokens). This is a feature — it enables horizontal scaling.

A request is: **Method + URL + Headers + Body**
A response is: **Status Code + Headers + Body**

## HTTP Methods — Use Them Correctly

| Method | Semantics | Idempotent | Safe | Has Body |
|--------|-----------|------------|------|----------|
| GET | Read resource | Yes | Yes | No |
| POST | Create / trigger action | No | No | Yes |
| PUT | Full replace | Yes | No | Yes |
| PATCH | Partial update | No | No | Yes |
| DELETE | Remove | Yes | No | Optional |
| HEAD | Like GET but no body | Yes | Yes | No |
| OPTIONS | CORS preflight / capabilities | Yes | Yes | No |

**Rules:**
- GET must never mutate state. No side effects.
- PUT replaces the entire resource. If you send partial data, missing fields are cleared.
- PATCH updates only what you send.
- POST is the right choice for actions that don't map to CRUD (e.g., `/orders/:id/cancel`).

## Status Codes — The Contract

### 2xx — Success
- `200 OK` — successful GET, PATCH, PUT
- `201 Created` — POST that created a resource. Include `Location: /resources/{id}` header.
- `202 Accepted` — request accepted, processing async
- `204 No Content` — success, no body (DELETE, actions with no return value)

### 3xx — Redirect
- `301 Moved Permanently` — permanent redirect, clients cache it
- `302 Found` — temporary redirect
- `304 Not Modified` — cached response is still valid (with ETags)

### 4xx — Client Error (their fault)
- `400 Bad Request` — malformed request, invalid JSON, missing required field
- `401 Unauthorized` — not authenticated (no token or invalid token)
- `403 Forbidden` — authenticated but not authorized
- `404 Not Found` — resource does not exist
- `405 Method Not Allowed` — wrong HTTP method for this route
- `409 Conflict` — state conflict (duplicate, already exists, wrong state)
- `410 Gone` — resource permanently deleted (stronger than 404)
- `422 Unprocessable Entity` — syntactically valid but semantically wrong (validation failure)
- `429 Too Many Requests` — rate limit hit. Include `Retry-After` header.

### 5xx — Server Error (your fault)
- `500 Internal Server Error` — unhandled exception, bug
- `502 Bad Gateway` — upstream service returned invalid response
- `503 Service Unavailable` — server overloaded or down for maintenance
- `504 Gateway Timeout` — upstream service timed out

**Critical rules:**
- Never return `200` for an error
- Never return `500` for a client mistake
- `401` = not authenticated. `403` = authenticated but no permission.

## Request Headers — What to Expect and Validate

```
Content-Type: application/json          # what format the body is in
Accept: application/json                # what format the client wants back
Authorization: Bearer <token>           # auth token
X-Request-ID: uuid                      # correlation ID for tracing
User-Agent: MyApp/1.0                   # client identifier
Accept-Language: en-US,en;q=0.9        # language preference
Accept-Encoding: gzip, deflate, br      # compression support
Cache-Control: no-cache                 # cache directive
```

## Response Headers — Always Set These

```
Content-Type: application/json; charset=utf-8
X-Request-ID: {echo back the request ID}
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1686000000
```

## Security Headers — Mandatory on Every Response

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=()
```

Remove `X-Powered-By` and `Server` headers — never expose framework/runtime.

## CORS — Cross-Origin Resource Sharing

CORS is enforced by browsers, not servers. The server just sends headers — the browser enforces them.

**Preflight (OPTIONS) request happens when:**
- Method is not GET/POST/HEAD
- Custom headers are present
- Content-Type is not one of the simple types

**CORS response headers:**
```
Access-Control-Allow-Origin: https://yourdomain.com    # NEVER use * for credentialed requests
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization, X-Request-ID
Access-Control-Allow-Credentials: true                  # only if sending cookies/auth
Access-Control-Max-Age: 86400                           # cache preflight for 24h
```

**Rules:**
- Never use `*` as `Allow-Origin` if the request includes credentials (cookies, Authorization header)
- Whitelist allowed origins explicitly — load from config, not hardcoded
- Handle OPTIONS requests before auth middleware (preflight has no auth)

## HTTP Caching

### ETag (Entity Tag)
```
# Server sends:
ETag: "abc123"

# Client sends on next request:
If-None-Match: "abc123"

# Server responds with 304 if unchanged, 200 + new ETag if changed
```

### Cache-Control
```
Cache-Control: no-store                    # never cache (sensitive data)
Cache-Control: no-cache                    # cache but must revalidate
Cache-Control: max-age=3600               # cache for 1 hour
Cache-Control: public, max-age=86400      # CDN-cacheable for 1 day
Cache-Control: private, max-age=300       # user-specific, browser only
```

### Last-Modified
```
Last-Modified: Wed, 21 Oct 2025 07:28:00 GMT
# Client sends: If-Modified-Since: Wed, 21 Oct 2025 07:28:00 GMT
```

## HTTP Compression

Always enable for text responses (JSON, HTML, XML). Never for already-compressed formats (images, videos).

```
# Client announces support:
Accept-Encoding: gzip, deflate, br

# Server responds with compressed body:
Content-Encoding: gzip
```

Priority: **Brotli (br) > gzip > deflate**

## HTTP Versions

| Version | Key Feature | Transport |
|---------|-------------|-----------|
| HTTP/1.1 | Keep-alive, pipelining | TCP |
| HTTP/2 | Multiplexing, header compression, server push | TCP + TLS |
| HTTP/3 | Built on QUIC (UDP), no head-of-line blocking | UDP |

**What this means for you:**
- HTTP/2: multiple requests over single connection simultaneously
- HTTP/1.1: head-of-line blocking — one slow request blocks the queue
- Use TLS (HTTPS) — HTTP/2 requires it in practice

## HTTPS / TLS

- TLS 1.2 minimum in production. TLS 1.3 preferred.
- Certificates from Let's Encrypt (free) or CA for enterprise
- HSTS tells browsers to always use HTTPS: `Strict-Transport-Security: max-age=31536000`
- Redirect all HTTP traffic to HTTPS with `301`
- Monitor certificate expiry — alert 30 days before

## Content Negotiation

```
# Client requests specific format:
Accept: application/json, application/xml;q=0.9

# Server responds with:
Content-Type: application/json

# If server can't satisfy Accept:
# Return 406 Not Acceptable
```

## Keep-Alive / Persistent Connections

```
Connection: keep-alive
Keep-Alive: timeout=5, max=100
```

Reuse TCP connections for multiple requests — avoid TCP handshake overhead on every request. This is default in HTTP/1.1+.

## Idempotency

Build idempotent APIs for retry safety:
- Accept `Idempotency-Key` header on POST/PATCH
- Cache response keyed by `Idempotency-Key` for 24 hours
- Return same response on re-submission without re-executing

## Common Mistakes to Avoid

- Returning `200` with `{ success: false }` in the body — use correct status codes
- Not setting `Content-Type` on responses
- Using `GET` for state-changing operations
- Exposing stack traces in error responses
- Using wildcard CORS in production with credentials
- Not handling `OPTIONS` requests (breaks CORS preflight)
- Forgetting `charset=utf-8` in `Content-Type` for text responses
