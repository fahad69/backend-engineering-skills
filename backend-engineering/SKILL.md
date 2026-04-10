---
name: backend-engineering
description: "Use this skill for any backend engineering task — building APIs, designing systems, implementing authentication, databases, caching, security, scaling, or when the user asks how to build a backend system correctly from first principles. This is the comprehensive backend engineering guide covering all topics: HTTP, routing, serialization, auth, validation, architecture, REST API, databases, caching, task queues, search, error handling, config management, observability, graceful shutdown, security, performance, scaling, and concurrency."
---

# Backend Engineering — Complete Guide from First Principles

> This skill is the complete reference. For deep dives, see individual skills: `http-protocol`, `api-routing`, `serialization`, `auth-authorization`, `validation`, `architecture`, `rest-api-design`, `databases`, `caching`, `task-queues`, `full-text-search`, `error-handling`, `config-management`, `observability`, `graceful-shutdown`, `backend-security`, `scaling-performance`, `concurrency`.

---

## What a Backend Is

A backend is a server that listens on a network port, receives requests, processes them, and returns responses. Its core responsibility is: receive data, process data, persist data, return data.

All backends — regardless of language — share the same fundamental components.

---

## 1. HTTP — The Protocol

All web backends communicate over HTTP. Master the fundamentals.

**Methods:** `GET` (read, idempotent, no side effects) | `POST` (create/action) | `PUT` (full replace, idempotent) | `PATCH` (partial update) | `DELETE` (remove, idempotent)

**Status codes:**
- `200` OK | `201` Created | `204` No Content
- `400` Bad Request | `401` Unauthorized | `403` Forbidden | `404` Not Found | `409` Conflict | `422` Unprocessable | `429` Rate Limited
- `500` Server Error | `503` Service Unavailable

**Security headers — always set:**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'
Referrer-Policy: strict-origin-when-cross-origin
```

**CORS:** explicit origin whitelist. Never `*` for credentialed requests. Handle `OPTIONS` before auth.

**Caching headers:** `Cache-Control`, `ETag`, `Last-Modified` for cacheable responses. `no-store` for sensitive data.

---

## 2. Routing

Route = HTTP method + URL path. Unique pair = unique operation.

- Nouns, plural, lowercase, hyphens: `/users`, `/order-items`
- Path params for identity: `/users/:id`
- Query params for filters, pagination, sort: `?status=active&page=2&limit=20`
- Nested routes max 2 levels: `/users/:id/orders`
- Version from day one: `/api/v1/`
- Non-CRUD actions: `POST /orders/:id/cancel`
- Catch-all 404 at the end

---

## 3. Serialization / Deserialization

- JSON is the default format for REST APIs
- Always use ISO 8601 UTC for dates: `"2025-06-15T10:30:00Z"`
- Never serialize internal models directly — use response DTOs
- Never serialize: passwords, tokens, internal flags, third-party IDs
- Consistent naming: snake_case throughout
- Handle missing fields: defaults for optional, error for required
- Set max body size (e.g., 10MB) to prevent DoS

---

## 4. Authentication

**Authentication** = who are you? **Authorization** = what can you do? Always separate.

**JWT:** verify signature always. Whitelist algorithms. Short-lived access tokens (15 min). Refresh token rotation. Payload is not encrypted — no sensitive data in it.

**Passwords:** bcrypt (cost 12), argon2id, or scrypt. Never MD5/SHA-1 alone. Always salted. Timing-safe comparison.

**Cookies:** `Secure; HttpOnly; SameSite=Strict`

**Account lockout:** 5 failures → 30-minute lockout. Same error message for wrong email or wrong password.

**Rate limit auth endpoints:** 10 login attempts per IP per 15 minutes.

---

## 5. Authorization

- Server-side checks on every request — never rely on client-side hiding
- Horizontal check: does this user own this resource?
- Vertical check: does this user have this role/permission?
- RBAC: roles → permissions. Check in service layer, not just route.
- Return `403 Forbidden` for authorization failures — `401` for auth failures
- Log all authorization failures

---

## 6. Input Validation

Validate at entry point. Reject before reaching business logic.

Types: presence, type, format (email, UUID, E.164, ISO date), range (min/max), semantic (date in past, end after start), enum, relational (field A required if field B set).

**Return all validation errors at once** — not just the first.

Transform after validating: normalize email to lowercase, trim whitespace, coerce query param strings to types, convert to UTC.

Server-side validation is mandatory — client-side is UX only.

---

## 7. Architecture Layers

Three layers. Strict separation. One-way dependency.

```
Controller → Service → Repository
```

**Controller:** HTTP in, HTTP out. No business logic. Parse, validate, call service, serialize response.

**Service:** all business logic. No HTTP awareness. Testable without a web framework.

**Repository:** all database access. No business logic. Returns domain objects.

**Middleware pipeline order:**
1. Request ID → Security Headers → CORS → Logging → Body Parser → Rate Limit → Auth → Handler → Error Handler (last)

**Request context:** request_id, trace_id, user, ip, start_time, deadline — per-request, never global state.

---

## 8. REST API Design

- Resource-oriented, noun-based URLs
- CRUD mapped to correct HTTP methods
- Paginate all list endpoints (default 20, max 100)
- Consistent response envelope: `{ data: ..., meta: ... }`
- Consistent error shape: `{ error: { code, message, details, request_id } }`
- Error codes are machine-readable: `VALIDATION_ERROR`, `NOT_FOUND`, `RATE_LIMIT_EXCEEDED`
- Version from day one: `/v1/`
- Breaking change = new version. Additive change = no new version.
- Deprecation headers: `Deprecation: true`, `Sunset: <date>`
- OpenAPI 3.x spec for all public endpoints

---

## 9. Databases

**ACID:** use transactions for multi-step operations. Either all succeed or all roll back.

**Schema design:**
- Foreign keys always defined — referential integrity at DB level
- Constraints: NOT NULL, UNIQUE, CHECK at DB level — not just application level
- Indexes on: every FK, every WHERE column, every JOIN column
- Always: `created_at`, `updated_at` timestamps
- Use UUIDs for IDs in distributed systems

**Queries:**
- Parameterized queries always — never string interpolation in SQL
- `EXPLAIN ANALYZE` every query that's slow
- Eliminate N+1: use JOINs or batch fetching
- Connection pool always: min 5, max 20 (tune per load)

**Migrations:** versioned files, reversible, run before deploying code that needs them.

**Application DB user:** SELECT, INSERT, UPDATE, DELETE only. No DDL.

---

## 10. Caching

Cache = expensive computation or fetch → stored for fast retrieval.

**Cache-aside:** check cache → miss → fetch DB → store in cache → return. Invalidate on write.

**TTL always set.** No permanent cache entries — data changes.

**Key design:** `service:entity:id` — deterministic, namespaced.

**Invalidation:** TTL (simple), event-driven (accurate), versioned keys (bulk invalidation).

**Never cache:** passwords, tokens, PII, user-specific data in shared caches without namespacing.

**HTTP cache headers:** `Cache-Control`, `ETag`, `Cache-Control: no-store` for sensitive responses.

**Multi-instance:** use Redis (shared) not in-process memory.

---

## 11. Task Queues & Background Jobs

If it doesn't need to be in the response → background job.

**Use for:** emails, notifications, image processing, external API calls, PDF generation, webhooks.

**Design:** jobs must be idempotent. Store IDs not data. Chain jobs for pipelines. Separate queues by priority.

**Retry:** exponential backoff with jitter. Max retries (e.g., 5). Dead letter queue after exhaustion.

**Workers:** finish current job on shutdown. Don't interrupt.

**Cron jobs:** use distributed locking to prevent duplicate runs on multi-instance.

---

## 12. Full-Text Search

`LIKE '%term%'` is slow at scale — full table scan, no relevance, no typo tolerance.

**Solution:** inverted index. Each term → list of documents containing it. O(1) lookup.

**Tools:**
- PostgreSQL full-text: good up to ~100K rows
- Elasticsearch/OpenSearch/Typesense: production-grade, large scale

**Key features:** relevance scoring, field boosting (title^3 > description^1), fuzzy matching, autocomplete.

**Sync:** keep search index in sync with database via async background job on writes.

**Never:** wildcard at start of term — defeats the inverted index.

---

## 13. Error Handling

**Mindset:** errors are normal. Design for failure.

- Define typed error hierarchy: `ValidationError`, `NotFoundError`, `ForbiddenError`, `ExternalServiceError`
- Global error handler: catches everything, returns correct status code
- Never expose stack traces, DB errors, or internal paths to clients
- Retry only transient errors — never retry 4xx errors
- Circuit breaker on external service calls: CLOSED → OPEN → HALF-OPEN
- Timeout on every external call — never infinite wait
- Graceful degradation: non-critical features fail silently, core product works
- Error tracking: Sentry or equivalent. Alert on error rate spikes.

---

## 14. Configuration Management

**Never hardcode config. Never commit secrets.**

Sources (priority order): environment variables → secrets manager → config files → defaults.

**Validate all config at startup** — fail fast if required variables missing.

**Environment separation:** development / staging / production have different config values. Never `if env == 'production'` in code — use config.

**Secrets:** never in `.env` files committed to version control. Use AWS Secrets Manager / HashiCorp Vault.

**Feature flags:** enable/disable features without deploying code.

**Never log config values** — especially secrets.

---

## 15. Observability

**Three pillars:** Logs (what happened) + Metrics (performance) + Traces (request path).

**Structured logging:** JSON format, every log has timestamp, level, request_id, service, message. Never plain text strings.

**Log levels:** DEBUG (dev only), INFO (normal events), WARN (unexpected but handled), ERROR (needs attention), FATAL (fatal).

**Never log:** passwords, tokens, PII, card numbers.

**The 4 Golden Signals:** latency (p50/p95/p99), traffic (RPS), errors (5xx%), saturation (CPU/memory/pool%).

**Alerts:** error rate > 1%, p99 latency > 2s, queue depth growing, DB pool > 90%.

**Distributed tracing:** trace_id propagated through all services and calls. Every external call is a span.

---

## 16. Graceful Shutdown

Handle `SIGTERM` and `SIGINT`. Sequence:

1. Stop accepting new requests → health check returns 503
2. Wait for in-flight requests (deadline: 30s)
3. Drain background workers (finish current job)
4. Close DB connections
5. Close Redis, message broker
6. Flush logs
7. Exit 0

In Kubernetes: `terminationGracePeriodSeconds` > shutdown timeout + buffer. Readiness probe returns unhealthy immediately on shutdown signal.

---

## 17. Security

**HTTPS:** always. TLS 1.2 min, 1.3 preferred. HSTS.

**SQL injection:** parameterized queries everywhere, always.

**XSS:** escape output for context. CSP header.

**CSRF:** SameSite=Strict cookie + CSRF tokens.

**Rate limiting:** every public endpoint. Aggressive on auth endpoints.

**Access control:** server-side, every request, check ownership AND role.

**Dependency vulnerabilities:** scan in CI. Patch critical vulnerabilities immediately.

**Audit logs:** log all auth events, permission changes, payment events.

---

## 18. Performance & Scaling

**Measure first.** Never optimize based on intuition.

**Find bottlenecks:** slow SQL queries (`EXPLAIN ANALYZE`), missing indexes, N+1, connection pool exhaustion, blocking I/O.

**Horizontal scale requires statelessness.** External store for all shared state (Redis for sessions, object storage for files).

**Load balancing:** health check endpoint. Route by round-robin or least connections.

**Database scaling:** read replicas for reads. PgBouncer for connection pooling at scale. Sharding only as last resort.

**CDN:** static assets, public cacheable API responses. Long TTL + content-hash filenames.

**Deployment:** rolling updates (`maxUnavailable: 0`), blue-green, canary — all for zero-downtime.

---

## 19. Concurrency

**I/O-bound:** async/await, event loop, goroutines — never block on I/O.

**CPU-bound:** separate workers, thread pools, worker processes — never on main event loop.

**Connection pooling:** every external connection (DB, Redis, HTTP) — never new connection per request.

**Race conditions:** atomic DB updates, optimistic locking (version field), distributed Redis locks for cross-instance critical sections.

**Deadlocks:** consistent lock ordering, short lock scopes, timeout on acquisition.

**Backpressure:** bounded queues, return 429 when overloaded rather than accepting unbounded work.

---

## 10 Guiding Principles

1. **Security by default.** Every endpoint requires auth unless explicitly public. Every query is parameterized.
2. **Fail fast on bad input. Fail safe on unexpected errors.**
3. **Separate concerns.** HTTP logic ≠ business logic ≠ data access.
4. **Measure before optimizing.** Know your bottleneck.
5. **Statelessness enables scale.** Any instance can serve any request.
6. **Secrets are infrastructure, not code.** Never commit them.
7. **Observability from day one.** Logs + metrics + traces.
8. **Every error is handled.** No silent swallowing. No unhandled rejections.
9. **Backward compatibility is a contract.** Never break consumers.
10. **Simple → Correct → Fast.** In that order.
