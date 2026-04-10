---
name: architecture
description: Use this skill when structuring backend code, deciding where to put business logic, designing the separation between HTTP handling and data access, implementing middleware pipelines, working with request context, or when the user asks about controllers, services, repositories, middleware, layered architecture, MVC, request lifecycle, or dependency injection.
---

# Backend Architecture — Controllers, Services, Repositories, Middleware

## The Three-Layer Architecture

Every backend system must separate concerns into three distinct layers. This is not optional.

```
HTTP Request
     ↓
┌─────────────────────────────┐
│   Controller / Handler      │  ← HTTP layer: parse request, call service, return response
└─────────────────────────────┘
     ↓ calls
┌─────────────────────────────┐
│   Service                   │  ← Business logic: rules, decisions, orchestration
└─────────────────────────────┘
     ↓ calls
┌─────────────────────────────┐
│   Repository                │  ← Data access: all DB queries, nothing else
└─────────────────────────────┘
     ↓
Database / External Store
```

**Dependency direction flows one way:** Controller → Service → Repository. Never reversed. Never circular.

## Layer Responsibilities

### Controller / Handler

**Does:**
- Receive and parse the HTTP request
- Extract path params, query params, headers, body
- Call validation
- Call the appropriate service method
- Serialize the response
- Return the HTTP response with correct status code

**Does NOT:**
- Contain any business logic
- Directly query the database
- Know how data is stored
- Make decisions about business rules

```python
# Controller example
def create_order(request):
    body = parse_body(request)
    validate(body, CreateOrderSchema)          # validate
    order = order_service.create(body, request.user)  # call service
    return response(201, serialize(order))    # return HTTP response
```

### Service

**Does:**
- Contain all business logic
- Enforce business rules
- Orchestrate multiple repository calls
- Call other services
- Call external APIs
- Throw business errors (not HTTP errors)
- Be testable without HTTP or database

**Does NOT:**
- Know about HTTP (no request/response objects)
- Directly construct SQL queries
- Know about serialization formats

```python
# Service example
def create_order(data, user):
    if not user.has_permission("orders:create"):
        raise ForbiddenError("Not allowed to create orders")
    
    product = product_repo.find(data.product_id)
    if not product:
        raise NotFoundError("Product not found")
    
    if product.stock < data.quantity:
        raise BusinessError("Insufficient stock")
    
    order = order_repo.create({
        "user_id": user.id,
        "product_id": product.id,
        "quantity": data.quantity,
        "total": product.price * data.quantity
    })
    
    notification_service.send_order_confirmation(user, order)
    return order
```

### Repository

**Does:**
- All database interactions (SELECT, INSERT, UPDATE, DELETE)
- Build and execute queries
- Map database rows to domain objects
- Handle database-level errors

**Does NOT:**
- Contain business logic
- Know about HTTP
- Call other services
- Make business decisions

```python
# Repository example
def create(data):
    return db.query(
        "INSERT INTO orders (user_id, product_id, quantity, total) VALUES ($1, $2, $3, $4) RETURNING *",
        [data.user_id, data.product_id, data.quantity, data.total]
    ).first()

def find_by_user(user_id, filters):
    return db.query(
        "SELECT * FROM orders WHERE user_id = $1 AND status = ANY($2) ORDER BY created_at DESC LIMIT $3 OFFSET $4",
        [user_id, filters.statuses, filters.limit, filters.offset]
    ).all()
```

## Middleware Pipeline

Middleware runs before and/or after the main handler. Each piece handles one cross-cutting concern.

**Standard pipeline order (top to bottom):**

```
1. Request ID injection          → generate/forward X-Request-ID
2. Security headers              → set HSTS, CSP, X-Frame-Options, etc.
3. CORS                          → handle preflight and set CORS headers
4. Request logging               → log incoming request (method, path, IP)
5. Body parsing                  → parse JSON/multipart body
6. Compression                   → gzip/brotli response
7. Rate limiting                 → check rate limit, reject if exceeded
8. Authentication                → verify token/session, attach user to context
9. Authorization (if global)     → role check for route groups
10. Handler (controller)         → your actual business logic
11. Response logging             → log outgoing response (status, latency)
12. Error handler (catch-all)    → convert errors to HTTP responses
```

**Rules:**
- Error handler middleware must be last — it catches everything above it
- CORS middleware before auth middleware — OPTIONS preflight has no auth
- Never put business logic in middleware
- Keep each middleware focused on one thing

## Request Context

Context is the carrier for per-request data. It flows through the entire lifecycle — middleware → controller → service.

**Context must contain:**
```
request_id:    string    # unique per request, for log correlation and X-Request-ID
trace_id:      string    # distributed trace ID (OpenTelemetry)
user:          object    # populated after auth middleware (id, roles, email)
ip_address:    string    # client IP (from X-Forwarded-For or REMOTE_ADDR)
start_time:    datetime  # when request arrived (for latency calculation)
deadline:      datetime  # when this request must complete (for downstream timeout)
```

**Rules:**
- Context is scoped to the request — never stored in global/class-level state
- Context passed explicitly (function argument) — not via global variables
- Sensitive data not in context unless strictly required
- Context cancelled when client disconnects — propagate to DB queries and external calls
- Use context deadline for all downstream timeouts

## Service-to-Service Communication

When one service calls another:
- Pass the context (with request_id and trace_id) to the called service
- Never start a new context — preserve the chain
- Set a timeout on every call

## Dependency Injection

Services and repositories are injected, not imported directly. This enables:
- Unit testing (inject mock dependencies)
- Swapping implementations (SQL vs NoSQL)
- Cleaner initialization

```python
# Constructor injection
class OrderService:
    def __init__(self, order_repo, product_repo, notification_service):
        self.order_repo = order_repo
        self.product_repo = product_repo
        self.notification_service = notification_service

# Wire up at application start
order_repo = OrderRepository(db_connection)
product_repo = ProductRepository(db_connection)
notification_service = NotificationService(email_client)
order_service = OrderService(order_repo, product_repo, notification_service)
```

## File Structure

```
src/
├── modules/
│   ├── orders/
│   │   ├── orders.controller.py    # HTTP layer
│   │   ├── orders.service.py       # Business logic
│   │   ├── orders.repository.py    # Data access
│   │   ├── orders.schema.py        # Validation schemas
│   │   ├── orders.serializer.py    # Response serialization
│   │   └── orders.routes.py        # Route definitions
│   ├── users/
│   │   └── ...
│   └── auth/
│       └── ...
├── middleware/
│   ├── auth.py
│   ├── rate_limit.py
│   ├── logger.py
│   ├── cors.py
│   └── error_handler.py
├── shared/
│   ├── database.py
│   ├── redis.py
│   ├── context.py
│   └── errors.py
└── app.py                          # Application bootstrap and middleware setup
```

## Error Handling in Layers

Define typed errors. Let each layer convert to the appropriate format:

```
Service throws:  NotFoundError, ForbiddenError, BusinessError, ValidationError
Repository throws: DatabaseError, UniqueConstraintError

Error handler middleware converts:
  NotFoundError      → 404
  ForbiddenError     → 403
  BusinessError      → 400 or 409 depending on type
  ValidationError    → 422
  DatabaseError      → 500 (log internally, return generic message)
  Unhandled anything → 500
```

Services never throw HTTP status codes. Services throw domain errors. The HTTP layer translates.

## Testing Strategy Per Layer

| Layer | Test Type | Depends On |
|-------|-----------|------------|
| Controller | Integration | HTTP client + real service or mock |
| Service | Unit | Mock repositories |
| Repository | Integration | Real test database |
| Middleware | Unit | Mock request/response |
