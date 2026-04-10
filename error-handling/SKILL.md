---
name: error-handling
description: Use this skill when implementing error handling, building fault-tolerant systems, designing retry logic, implementing circuit breakers, or when the user asks about error types, global error handlers, graceful degradation, fallback behavior, timeouts, or building resilient backend services.
---

# Error Handling & Fault Tolerance — Backend Engineering

## The Right Mindset

Errors are not exceptional — they are normal. Networks fail, services go down, disks fill up, users send bad data, bugs exist. A system that assumes everything works is a system waiting to fail catastrophically.

Design for failure from the start. Error handling is not an afterthought.

## Error Classification

| Type | Description | Response |
|------|-------------|----------|
| Validation errors | Client sent bad input | 400/422, explain what's wrong |
| Auth errors | Not authenticated or authorized | 401/403 |
| Not found | Resource doesn't exist | 404 |
| Business logic errors | Operation not permitted by rules | 409/422 |
| External service errors | Third-party API failure | Retry, fallback, 503 |
| Database errors | Query failed, connection lost | Retry, 500 |
| Infrastructure errors | OOM, disk full, network | Alert, 503 |
| Logic errors | Bug in your code | 500, fix it |

## Typed Error Hierarchy

Define explicit error types. Never throw raw strings or generic exceptions.

```python
class AppError(Exception):
    """Base for all application errors"""
    status_code: int
    error_code: str
    message: str

class ValidationError(AppError):
    status_code = 422
    error_code = "VALIDATION_ERROR"

class NotFoundError(AppError):
    status_code = 404
    error_code = "NOT_FOUND"

class ForbiddenError(AppError):
    status_code = 403
    error_code = "FORBIDDEN"

class ConflictError(AppError):
    status_code = 409
    error_code = "CONFLICT"

class BusinessError(AppError):
    status_code = 422
    error_code = "BUSINESS_RULE_VIOLATION"

class ExternalServiceError(AppError):
    status_code = 503
    error_code = "EXTERNAL_SERVICE_ERROR"

class InternalError(AppError):
    status_code = 500
    error_code = "INTERNAL_ERROR"
```

## Global Error Handler

The last middleware in the chain. Catches everything that wasn't explicitly handled.

```python
def error_handler(error, request):
    request_id = request.context.request_id
    
    if isinstance(error, AppError):
        # Known application error
        log.warn("Application error", error=error, request_id=request_id)
        return response(error.status_code, {
            "error": {
                "code": error.error_code,
                "message": error.message,  # safe to show
                "request_id": request_id
            }
        })
    
    # Unknown error — bug or infrastructure failure
    log.error("Unhandled error", error=error, stack=traceback(), request_id=request_id)
    alert_on_call(error)  # if error rate is above threshold
    return response(500, {
        "error": {
            "code": "INTERNAL_ERROR",
            "message": "An unexpected error occurred. Please try again later.",
            # DO NOT expose: stack trace, DB error, internal state
            "request_id": request_id  # for support lookup
        }
    })
```

## Fail Fast vs Fail Safe

**Fail Fast:** detect the problem as early as possible and stop. Good for: validation, pre-condition checks.
```python
def process_order(user, product_id, quantity):
    if quantity <= 0:
        raise ValidationError("Quantity must be positive")  # fail immediately
    if not user.is_active:
        raise ForbiddenError("Account is suspended")  # fail immediately
    # proceed with business logic only when inputs are valid
```

**Fail Safe:** when something fails, default to a safe/degraded state. Good for: non-critical features.
```python
def get_product_recommendations(user_id):
    try:
        return recommendation_service.get(user_id)
    except Exception:
        log.warn("Recommendation service failed, returning empty")
        return []  # safe default — page still loads without recommendations
```

## Retry Logic — Transient Failure Recovery

Only retry transient failures (network timeout, temporary service unavailability). Never retry permanent failures (auth error, invalid input, 404).

```python
def call_with_retry(fn, max_retries=3, retryable_codes=(503, 429, 502, 504)):
    for attempt in range(max_retries + 1):
        try:
            return fn()
        except ExternalServiceError as e:
            if e.status_code not in retryable_codes:
                raise  # permanent failure, don't retry
            if attempt == max_retries:
                raise  # exhausted retries
            
            delay = min(2 ** attempt * 1.0, 30)  # exponential backoff, max 30s
            jitter = random.uniform(0, delay * 0.1)  # add jitter to avoid thundering herd
            time.sleep(delay + jitter)
```

Retryable errors: `503`, `429`, `502`, `504`, network timeout, connection reset
Non-retryable errors: `400`, `401`, `403`, `404`, `422`, any 4xx

## Circuit Breaker Pattern

Prevents cascading failures when an external service is down.

```
States:
CLOSED   → normal operation, calls go through
OPEN     → service is down, fail fast without calling
HALF-OPEN → testing if service recovered (allow 1 call through)

Transitions:
CLOSED → OPEN:      after N consecutive failures in time window
OPEN → HALF-OPEN:   after cooldown period (e.g., 60 seconds)
HALF-OPEN → CLOSED: if test call succeeds
HALF-OPEN → OPEN:   if test call fails
```

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, cooldown=60):
        self.state = "CLOSED"
        self.failures = 0
        self.failure_threshold = failure_threshold
        self.opened_at = None
        self.cooldown = cooldown
    
    def call(self, fn):
        if self.state == "OPEN":
            if time.now() - self.opened_at > self.cooldown:
                self.state = "HALF_OPEN"
            else:
                raise ExternalServiceError("Circuit open — service unavailable")
        
        try:
            result = fn()
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise
    
    def on_success(self):
        self.failures = 0
        self.state = "CLOSED"
    
    def on_failure(self):
        self.failures += 1
        if self.failures >= self.failure_threshold:
            self.state = "OPEN"
            self.opened_at = time.now()
```

## Timeout — Never Wait Forever

Every external call must have a timeout. No exceptions.

```python
# Timeout every:
# - HTTP call to external API
# - Database query
# - Redis operation
# - File I/O
# - Background job execution

result = http_client.get(url, timeout=10)  # 10 second timeout

db.query(sql, params, timeout=5)  # 5 second query timeout

redis.get(key, socket_timeout=1)  # 1 second Redis timeout
```

Timeout values by type:
- Redis operations: 1–2 seconds
- Database queries (simple): 5 seconds
- Database queries (complex): 30 seconds
- Internal service calls: 5–10 seconds
- External API calls (Sabre search): 15 seconds
- External API calls (Sabre booking): 30 seconds
- Email sending: 10 seconds

## Graceful Degradation

When a non-critical service fails, the core product still works:

```python
def get_product_page(product_id):
    product = product_repo.find(product_id)  # critical — let it fail if broken
    
    # Non-critical — degrade gracefully
    try:
        reviews = review_service.get(product_id, limit=5)
    except Exception:
        reviews = []  # page loads without reviews
    
    try:
        recommendations = recommendation_service.get(product_id)
    except Exception:
        recommendations = []  # page loads without recommendations
    
    return { "product": product, "reviews": reviews, "recommendations": recommendations }
```

## Error Monitoring — Non-Negotiable

Integrate error tracking from day one:

```python
import sentry_sdk  # or rollbar, bugsnag, datadog

# Capture unhandled errors
sentry_sdk.capture_exception(error)

# Add context
with sentry_sdk.push_scope() as scope:
    scope.set_user({"id": user.id, "email": user.email})
    scope.set_tag("booking_id", booking_id)
    sentry_sdk.capture_exception(error)
```

Set up alerts on:
- Error rate above 1% of requests
- New error type appearing
- Error volume spike
- Specific critical errors (payment failure, booking failure)

## Database Transaction Error Handling

```python
def create_booking(data):
    with db.transaction():
        try:
            booking = booking_repo.create(data)
            inventory_repo.decrement(data.product_id, data.quantity)
            payment_repo.authorize(data.payment_method_id, booking.total)
            return booking
        except Exception:
            # Transaction auto-rolls back
            raise  # re-raise for caller to handle
```

Never catch exceptions inside a transaction unless you can handle them and continue correctly.

## What NOT to Do

- Swallowing errors silently (`except: pass` — the worst pattern)
- Catching all exceptions and returning 200 OK
- Exposing stack traces, database query details, or internal paths to clients
- Retrying non-idempotent operations without idempotency checks
- Setting infinite timeouts on external calls
- Logging errors without actionable context
- Not alerting when errors spike
- Throwing HTTP status codes from service layer (use domain errors)
