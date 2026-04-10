---
name: observability
description: Use this skill when implementing logging, metrics, monitoring, distributed tracing, setting up alerts, or when the user asks about structured logging, log levels, Prometheus metrics, Grafana dashboards, distributed tracing, OpenTelemetry, observability pillars, or understanding what's happening inside a running system.
---

# Logging, Monitoring & Observability — Backend Engineering

## The Three Pillars

| Pillar | Answers | Tools |
|--------|---------|-------|
| **Logs** | What happened? | ELK, Loki, CloudWatch |
| **Metrics** | How is the system performing? | Prometheus, Datadog, CloudWatch |
| **Traces** | Where did a request go and how long did each step take? | Jaeger, Zipkin, Datadog APM |

You need all three. Logs alone can't answer performance questions. Metrics alone can't explain why a specific request failed. Traces alone don't give you historical trends.

## Structured Logging — The Only Acceptable Format

Never log plain text strings in production. Log structured JSON. Machines parse logs, not humans.

```json
// Bad — impossible to query or parse programmatically
"2025-06-01 10:30:15 ERROR Failed to process order 123 for user john@example.com"

// Good — structured, queryable, filterable
{
  "timestamp": "2025-06-01T10:30:15.123Z",
  "level": "error",
  "message": "Order processing failed",
  "request_id": "req_abc123",
  "trace_id": "trace_xyz789",
  "user_id": "usr_456",
  "order_id": "ord_123",
  "error_code": "PAYMENT_DECLINED",
  "service": "order-service",
  "environment": "production",
  "duration_ms": 1250
}
```

Every log entry must have:
- `timestamp`: ISO 8601 UTC
- `level`: debug | info | warn | error | fatal
- `message`: human-readable description
- `request_id`: correlation ID for the request
- `service`: which service emitted the log

## Log Levels — Use Them Correctly

| Level | When to Use | Production Volume |
|-------|-------------|-------------------|
| DEBUG | Detailed developer info (DB query params, variable values) | Disabled |
| INFO | Normal events (request received, job started, user logged in) | Low |
| WARN | Unexpected but recoverable (retrying, fallback used, deprecated endpoint hit) | Low |
| ERROR | Failure that needs attention (payment failed, external service error, DB error) | Alert-worthy |
| FATAL | Application cannot continue (startup failure, unrecoverable state) | Restart server |

```python
# INFO — normal events
log.info("User logged in", user_id=user.id, ip=request.ip)
log.info("Order created", order_id=order.id, amount=order.total)

# WARN — unexpected but handled
log.warn("Cache miss on critical key", key=cache_key, fallback="database")
log.warn("Payment retry attempt", order_id=order_id, attempt=2)

# ERROR — action required
log.error("Payment failed", order_id=order.id, error=str(e), gateway_code=e.code)
log.error("Sabre API timeout", endpoint="BFM", timeout_ms=15000)

# DEBUG — only in development
log.debug("SQL query", query=sql, params=params, duration_ms=12)
```

## What to Log

### Always Log
- Every incoming request: method, path, status code, duration, request_id
- Every outgoing response: same fields
- Authentication events: login, logout, failed login, MFA, token refresh
- Authorization failures
- Every payment event (no card details — just outcome and amounts)
- Every booking event
- Errors with full context
- Background job start, completion, failure
- External service calls: endpoint, latency, status

### Never Log (Security / Privacy)
- Passwords (any form)
- Tokens (JWT, session, API key, OAuth tokens)
- Credit card numbers or CVV
- PII in raw form (passport numbers, SSN, bank accounts)
- Secrets and private keys
- Full request body if it contains sensitive fields
- Database connection strings

## Request Logging Pattern

```python
# Log at entry (request received)
log.info("Request received",
    method=request.method,
    path=request.path,
    ip=request.ip,
    user_agent=request.user_agent,
    request_id=request.context.request_id,
    user_id=request.context.user_id  # None if not authenticated
)

# Log at exit (response sent)
log.info("Request completed",
    method=request.method,
    path=request.path,
    status_code=response.status_code,
    duration_ms=elapsed_time_ms,
    request_id=request.context.request_id
)
```

## Core Metrics to Track

### The Four Golden Signals (Google SRE)

1. **Latency**: time to serve a request
   - p50 (median), p95, p99 percentiles
   - Separate success vs error latency
   
2. **Traffic**: requests per second (RPS)
   - By endpoint, by method, by status code
   
3. **Errors**: rate of failed requests
   - Error % = 5xx / total requests
   - Separate 4xx (client) from 5xx (server)
   
4. **Saturation**: how full is your system?
   - CPU %, memory %, connection pool usage %, queue depth

### Application Metrics
```
http_requests_total{method, path, status}
http_request_duration_seconds{method, path, quantile}
active_connections
db_connection_pool_active
db_connection_pool_idle
db_query_duration_seconds{operation}
cache_hits_total
cache_misses_total
background_jobs_queued
background_jobs_processing
background_jobs_failed_total
external_api_calls_total{service, endpoint, status}
external_api_duration_seconds{service, endpoint}
```

### Business Metrics (Custom)
```
orders_created_total
bookings_created_total{type: flight|hotel|car|vacation}
payments_processed_total{status: success|failed}
search_requests_total{vertical}
users_registered_total
tickets_issued_total
```

## Prometheus + Grafana Setup

```python
# Expose /metrics endpoint
from prometheus_client import Counter, Histogram, Gauge

request_counter = Counter('http_requests_total', 'Total HTTP requests',
    ['method', 'path', 'status'])

request_duration = Histogram('http_request_duration_seconds', 'Request duration',
    ['method', 'path'],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0])

# Record in middleware
@app.middleware
def metrics_middleware(request, next):
    start = time.time()
    response = next(request)
    duration = time.time() - start
    
    request_counter.labels(request.method, request.path, response.status_code).inc()
    request_duration.labels(request.method, request.path).observe(duration)
    return response
```

## Alerting Rules

Configure alerts based on meaningful thresholds:

```yaml
# Error rate > 1% over 5 minutes
alert: HighErrorRate
expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.01
for: 5m
severity: critical

# p99 latency > 2 seconds
alert: HighLatency
expr: histogram_quantile(0.99, http_request_duration_seconds) > 2
for: 5m
severity: warning

# Queue depth growing
alert: BacklogBuilding
expr: background_jobs_queued > 1000
for: 10m
severity: warning

# DB connection pool nearly full
alert: DBPoolSaturation
expr: db_connection_pool_active / db_connection_pool_max > 0.9
for: 5m
severity: critical
```

**Alert fatigue prevention:**
- Only alert on things that require human action
- Alert on symptoms (error rate up), not causes (specific service down)
- Group related alerts
- Calibrate thresholds — too sensitive and people ignore alerts

## Distributed Tracing

In microservices, a single request touches multiple services. Tracing lets you follow it.

```python
# Every request gets a trace_id
# Every operation within the request is a "span"

tracer = opentelemetry.trace.get_tracer("order-service")

def create_order(request):
    with tracer.start_as_current_span("create_order") as span:
        span.set_attribute("user.id", request.user.id)
        
        with tracer.start_as_current_span("validate_input"):
            validate(request.body)
        
        with tracer.start_as_current_span("db.insert_order"):
            order = order_repo.create(request.body)
        
        with tracer.start_as_current_span("external.sabre_api"):
            sabre_result = call_sabre(order)
        
        return order
```

Trace propagation — pass trace context to all downstream calls:
```
X-Trace-Id: trace_xyz789
X-Span-Id: span_abc123
```

## Log Aggregation

Send all logs to a central system:
- **ELK Stack**: Elasticsearch + Logstash + Kibana (self-hosted)
- **Loki + Grafana**: lightweight, queries like Prometheus (self-hosted)
- **Datadog**: fully managed, expensive
- **CloudWatch**: if you're on AWS

Log retention policy:
- Debug/Info: 7–14 days
- Warnings: 30 days
- Errors: 90 days
- Security/audit logs: 1–7 years (compliance requirement)
