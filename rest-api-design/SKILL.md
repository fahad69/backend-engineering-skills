---
name: rest-api-design
description: Use this skill when designing or reviewing REST APIs, deciding between PUT vs PATCH, designing request/response bodies, handling pagination, API versioning, designing error responses, or when the user asks about REST principles, resource design, CRUD mapping, API consistency, backward compatibility, or OpenAPI specification.
---

# REST API Design — Backend Engineering

## REST in One Paragraph

REST (Representational State Transfer) is an architectural style where resources are identified by URLs, and you interact with them using standard HTTP methods. The URL identifies the resource. The HTTP method describes the operation. The status code describes the outcome. The body carries the data.

## Resource Design

Everything is a resource. Resources are nouns, never verbs.

```
# Resources
/users
/orders
/products
/invoices
/payment-methods

# Not resources (wrong)
/getUser
/createOrder
/processPayment
```

### CRUD to HTTP Mapping

| Operation | HTTP Method | URL | Success Code |
|-----------|-------------|-----|--------------|
| List collection | GET | /resources | 200 |
| Get single | GET | /resources/:id | 200 |
| Create | POST | /resources | 201 |
| Full replace | PUT | /resources/:id | 200 |
| Partial update | PATCH | /resources/:id | 200 |
| Delete | DELETE | /resources/:id | 204 |
| Non-CRUD action | POST | /resources/:id/verb | 200 |

## PUT vs PATCH — The Clear Rule

**PUT** = replace the entire resource. You send the complete representation. Missing fields are cleared/reset to defaults.

**PATCH** = update only the fields you send. Everything else stays the same.

```json
// Current resource: { "name": "John", "email": "john@example.com", "age": 30 }

// PUT — sends full resource
PUT /users/123
{ "name": "John Updated", "email": "john@example.com", "age": 30 }
// Result: all three fields updated. Send all or lose the ones you omit.

// PATCH — sends only what changes
PATCH /users/123
{ "name": "John Updated" }
// Result: { "name": "John Updated", "email": "john@example.com", "age": 30 }
// email and age unchanged
```

When in doubt: use **PATCH**. It's safer and what most users expect from an "update" operation.

## Pagination

Always paginate list endpoints. Never return unbounded collections.

### Offset/Page-Based Pagination
```
GET /products?page=2&limit=20

Response:
{
  "data": [...],
  "meta": {
    "total": 347,
    "page": 2,
    "limit": 20,
    "total_pages": 18,
    "has_next": true,
    "has_prev": true
  }
}
```

Good for: small-to-medium datasets, UIs that need page numbers.
Bad for: large datasets where `OFFSET` becomes slow.

### Cursor-Based Pagination
```
GET /feed?limit=20
GET /feed?cursor=eyJpZCI6MTIzfQ&limit=20

Response:
{
  "data": [...],
  "meta": {
    "next_cursor": "eyJpZCI6MTQzfQ",
    "has_next": true
  }
}
```

Good for: real-time feeds, large datasets, avoiding duplicates as data changes.
Cursor is an opaque base64-encoded reference to the last item.

**Always set:**
- Default limit: 20
- Maximum limit: 100 (enforce this even if client sends more)
- Never return more than the max regardless of what client requests

## Filtering and Sorting

```
GET /products?category=electronics&status=active&min_price=100&max_price=500
GET /orders?user_id=123&created_after=2025-01-01&status=confirmed
GET /users?sort=created_at&order=desc
GET /products?sort=price&order=asc
```

Sort field validation: only allow sorting on indexed columns. Reject unknown sort fields.

## Search

```
GET /products?q=laptop+gaming
GET /users?q=john
```

For full-text search needs — pass to search engine, not a SQL LIKE query.

## Response Envelope — Consistent Shape for Everything

Every response follows the same envelope:

```json
// Success with data
{
  "data": { ... },
  "meta": {
    "request_id": "req_abc123",
    "timestamp": "2025-06-01T12:00:00Z"
  }
}

// Success list
{
  "data": [ ... ],
  "meta": {
    "total": 100,
    "page": 1,
    "limit": 20
  }
}

// Error
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "The requested user does not exist",
    "details": [],
    "request_id": "req_abc123"
  }
}
```

Never mix these structures. An endpoint that returns data should always have it in `data`. An error should always be in `error`.

## Error Design

Every error has a machine-readable `code` field. Status code alone is not enough.

```json
{
  "error": {
    "code": "VALIDATION_ERROR",           // machine-readable, SCREAMING_SNAKE_CASE
    "message": "Request validation failed", // human-readable
    "details": [                           // field-level details for validation
      { "field": "email", "message": "Must be a valid email" }
    ],
    "request_id": "req_abc123"
  }
}
```

Error code taxonomy:
```
VALIDATION_ERROR         → 422
AUTHENTICATION_REQUIRED  → 401
PERMISSION_DENIED        → 403
RESOURCE_NOT_FOUND       → 404
RESOURCE_CONFLICT        → 409
RATE_LIMIT_EXCEEDED      → 429
INTERNAL_ERROR           → 500 (no details in message — just the request_id)
```

## API Versioning

Version from day one. Use URI versioning (`/v1/`).

```
/api/v1/users
/api/v2/users
```

**What requires a new version (breaking changes):**
- Removing a field from response
- Changing a field type (string → integer)
- Renaming a field
- Changing the meaning of a field
- Removing an endpoint
- Changing required/optional status of a field

**What does NOT require a new version:**
- Adding a new optional field to response
- Adding a new optional query parameter
- Adding a new endpoint
- Bug fixes that don't change the contract

**Deprecation headers:**
```
Deprecation: true
Sunset: Sat, 01 Jan 2026 00:00:00 GMT
Link: <https://api.example.com/v2/users>; rel="successor-version"
```

Give at minimum 6 months notice before removing an old version.

## Naming Conventions

Pick one, apply everywhere, never mix:

```json
// snake_case (preferred for REST APIs)
{
  "user_id": "123",
  "first_name": "John",
  "created_at": "2025-01-01T00:00:00Z",
  "is_active": true,
  "order_items": []
}
```

## Field Naming Guidelines

- Boolean fields: `is_active`, `has_children`, `requires_approval` — always a clear yes/no question
- Timestamps: `created_at`, `updated_at`, `deleted_at`, `expires_at` — always `_at` suffix
- IDs: `user_id`, `order_id` — always `_id` suffix for foreign keys
- Counts: `item_count`, `review_count` — always `_count` suffix

## Partial Response (Field Selection)

For bandwidth-sensitive clients, allow field selection:

```
GET /users/123?fields=id,email,name
```

Return only the requested fields. Cache the full object, project at the response layer.

## Idempotency

For operations that should not be duplicated (payment, booking creation):

```
POST /orders
Idempotency-Key: client-generated-uuid

# Second request with same key → return same response, no duplicate processing
```

Server stores: `hash(idempotency_key) → response` for 24 hours.

## Webhooks (Outbound Events)

When your system needs to notify external consumers of events:

```json
// POST to customer's registered endpoint
{
  "event": "order.completed",
  "event_id": "evt_abc123",
  "timestamp": "2025-06-01T12:00:00Z",
  "data": {
    "order_id": "ord_xyz",
    "status": "completed"
  }
}
```

Always:
- Include `event_id` for deduplication
- Sign the payload (`X-Signature: hmac-sha256=...`)
- Retry with exponential backoff on failure
- Let consumers verify the signature

## OpenAPI / Swagger Specification

Every public API must have an OpenAPI 3.x spec. This is not optional.

Minimum documentation per endpoint:
- Description of what it does
- All parameters (path, query, header) with types and constraints
- Request body schema with examples
- All possible response schemas with examples
- Authentication requirement
- All possible error codes

Keep documentation in sync with code — generate it from code annotations where possible. Stale documentation is worse than no documentation.
