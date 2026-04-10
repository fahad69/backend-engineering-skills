---
name: api-routing
description: Use this skill when designing URL structures, defining routes, working with path parameters, query parameters, route versioning, nested resources, or when the user asks about routing patterns, URL design, API path structure, route grouping, or how requests are mapped to handlers.
---

# API Routing — Backend Engineering

## Core Mental Model

A route is the combination of **HTTP method + URL path**. That pair is the unique identifier for a backend operation. Two routes with the same path but different methods are completely different routes.

```
GET  /users        → list users
POST /users        → create user
GET  /users/:id    → get one user
PUT  /users/:id    → replace user
PATCH /users/:id   → update user
DELETE /users/:id  → delete user
```

## Path Design Rules

### Use Nouns, Not Verbs

```
# Wrong
GET /getUsers
POST /createUser
DELETE /deleteUser/:id

# Correct
GET /users
POST /users
DELETE /users/:id
```

### Use Plural Nouns for Collections

```
/users            # correct
/user             # wrong
/products         # correct
/product-list     # wrong
```

### Use Lowercase and Hyphens

```
/user-profiles    # correct
/userProfiles     # wrong (camelCase in URLs)
/user_profiles    # avoid (underscores)
```

## Path Parameters vs Query Parameters

### Path Parameters — Identity
Use for the identity of a specific resource. Required. Part of the resource address.

```
GET /users/:id
GET /orders/:orderId
GET /products/:productId/reviews/:reviewId
```

### Query Parameters — Modifiers
Use for filtering, sorting, pagination, search. Optional. Do not change what resource you're looking at.

```
GET /users?role=admin&status=active
GET /products?category=electronics&min_price=100&max_price=500
GET /orders?sort=created_at&order=desc&page=2&limit=20
```

### Standard Pagination Query Params

```
?page=1          # page number, 1-indexed
?limit=20        # items per page
?offset=0        # alternative to page — skip N items
?cursor=xyz      # cursor-based pagination for large datasets
?sort=field      # field to sort by
?order=asc|desc  # sort direction
```

Always:
- Set a default limit (e.g., 20)
- Enforce a maximum limit (e.g., 100)
- Return total count in response for page-based pagination
- Return next/prev cursors for cursor-based pagination

## Nested Routes — Hierarchical Resources

Use nesting to express ownership or containment. But limit to 2 levels deep.

```
# One level of nesting (good)
/users/:userId/posts
/users/:userId/posts/:postId
/orders/:orderId/items

# Too deep (avoid)
/users/:userId/posts/:postId/comments/:commentId/likes
# Better: /comments/:commentId/likes
```

When a sub-resource can exist independently, prefer flat routes with query params:
```
# Instead of: /users/:userId/orders/:orderId
# Use flat:   /orders/:orderId          (order knows its user)
# Or filter:  /orders?user_id=123
```

## Route Versioning — Mandatory from Day One

Never ship an unversioned API. Breaking changes without versioning break consumers.

### URI Versioning (Recommended)
```
/api/v1/users
/api/v2/users
```

### Header Versioning (Alternative)
```
GET /api/users
API-Version: 2
```

### Versioning Rules
- Start at `v1` — never `v0`
- New version only for breaking changes (field removal, type change, semantics change)
- Additive changes (new optional fields, new endpoints) do NOT require new version
- Run multiple versions simultaneously during transition period
- Announce deprecation with `Deprecation` and `Sunset` response headers
- Give consumers at least 6 months before removing old version
- `Sunset: Sat, 01 Jan 2026 00:00:00 GMT` header tells when old version shuts down

## Route Grouping

Group routes by domain or feature. Never put all routes in one file.

```
/api/v1/auth/...         → auth routes group
/api/v1/users/...        → user routes group
/api/v1/products/...     → product routes group
/api/v1/orders/...       → order routes group
/api/v1/admin/...        → admin routes group (separate auth middleware)
```

Apply middleware at group level, not per-route:
- Auth middleware on all `/api/v1/` except `/api/v1/auth/`
- Admin middleware on all `/api/v1/admin/`
- Rate limiting per group with different limits

## Non-CRUD Actions

When an action doesn't map cleanly to CRUD, use POST with a verb in the path:

```
POST /orders/:id/cancel
POST /users/:id/suspend
POST /invoices/:id/send
POST /accounts/:id/verify
POST /payments/:id/refund
```

Do NOT abuse GET for state-changing operations.
Do NOT invent verbs for things that are actually CRUD.

```
# Wrong — this is just a PATCH
POST /orders/:id/updateStatus

# Correct — PATCH is right here
PATCH /orders/:id    { "status": "processing" }

# Correct — this is genuinely an action
POST /orders/:id/cancel
```

## Route-Level Validation

Validate path parameters at the route level before they reach business logic:
- UUID format validation for ID params
- Integer validation for numeric IDs
- Enum validation for type params in paths

Return `400 Bad Request` for malformed path params, not `404`.

## 404 Catch-All

Always define a catch-all route at the end of your router:

```
# Catches any undefined route
* → 404 Not Found
{ "error": { "code": "ROUTE_NOT_FOUND", "message": "The requested endpoint does not exist" } }
```

## Route Documentation Annotations

Every route must be documented at definition:
```
method: GET
path: /users/:id
description: Retrieve a single user by ID
auth: required
params:
  id: UUID — user identifier
responses:
  200: User object
  401: Not authenticated
  403: Not authorized
  404: User not found
```

## Search Routes

```
# Collection-level search
GET /products/search?q=laptop&category=electronics

# OR use query params on the collection
GET /products?q=laptop

# Never POST for search (GET is correct — no side effects, bookmarkable)
# Exception: complex search criteria that exceed URL length limits → POST /search
```

## Common Mistakes

- Using verbs in path names (`/getUser`, `/createProduct`)
- Singular nouns for collections (`/user` instead of `/users`)
- Missing versioning (`/api/users` — breaks when you need to change it)
- Over-nesting routes more than 2 levels deep
- Putting filters in path instead of query params (`/users/active` instead of `/users?status=active`)
- Using GET for operations with side effects
- Inconsistent naming (mixing camelCase and snake_case in paths)
