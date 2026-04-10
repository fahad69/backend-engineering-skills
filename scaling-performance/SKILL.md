---
name: scaling-performance
description: Use this skill when optimizing backend performance, designing for horizontal scale, implementing load balancing, database replication, sharding, stateless architecture, deployment strategies, or when the user asks about latency, throughput, bottlenecks, vertical vs horizontal scaling, CDNs, connection pooling, or how to handle more traffic.
---

# Scaling & Performance Engineering — Backend Engineering

## Measure Before You Optimize

Never optimize based on intuition. Measure first. Find the actual bottleneck. Fix it. Measure again.

```
Profile → Identify bottleneck → Optimize → Measure → Repeat
```

Premature optimization creates complex code that solves the wrong problem.

## Performance Metrics

| Metric | Definition | Target |
|--------|------------|--------|
| p50 latency | Median response time | < 100ms |
| p95 latency | 95th percentile | < 500ms |
| p99 latency | 99th percentile | < 1s |
| Throughput | Requests per second (RPS) | Depends on load |
| Error rate | % of 5xx responses | < 0.1% |
| TTFB | Time To First Byte | < 200ms |

Always use percentiles — averages hide slow outliers. A p50 of 50ms with p99 of 5s is a bad system.

## Identifying Bottlenecks

Common bottleneck types:
- **Database**: slow queries, missing indexes, N+1, connection pool exhaustion
- **CPU-bound**: heavy computation on the main thread, blocking the event loop
- **I/O-bound**: waiting for external APIs, slow disk, network
- **Memory**: memory leaks, garbage collection pressure, caches not bounded
- **Network**: large payloads, no compression, no keep-alive, many round trips

Profiling tools by language:
- Node.js: `clinic.js`, `--prof` flag, `0x`
- Python: `py-spy`, `cProfile`, `Yappi`
- Go: `pprof`
- Java: JProfiler, async-profiler

**Identify slow database queries:**
```sql
-- PostgreSQL: find slow queries
SELECT query, mean_exec_time, calls, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

## Database Performance

### Indexing (Biggest Lever)
A missing index on a filtered column turns a 5ms query into a 5-second one at 1M rows.

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = $1;
-- "Seq Scan" → missing index → add it
-- "Index Scan" → good

CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
-- CONCURRENTLY = doesn't lock the table during creation
```

### Query Optimization
```sql
-- Select only needed columns, not *
SELECT id, email, name FROM users WHERE status = 'active';

-- Use LIMIT on all queries
SELECT * FROM products ORDER BY created_at DESC LIMIT 20;

-- Use JOIN instead of subqueries where possible
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;
```

### Connection Pool Sizing
```
Pool size formula: (2 × num_cores) + num_disk_spindles
Practical: start at 10, increase if you see "waiting for connection" in metrics

Never open a connection per request.
Use PgBouncer in transaction mode for very high concurrency.
```

## Vertical vs Horizontal Scaling

**Vertical scaling (Scale Up):** bigger machine — more CPU, more RAM.
- Simple — no code changes
- Has a ceiling — can't add infinite RAM
- Single point of failure
- Expensive at high end

**Horizontal scaling (Scale Out):** more machines.
- Unlimited scale in theory
- Requires stateless application
- Requires load balancing
- More complex infrastructure

**Use vertical first** until you hit cost or ceiling limits. Then scale horizontally.

## Statelessness — Required for Horizontal Scaling

A stateless application keeps no request-specific data in memory between requests. Any instance can handle any request.

```
# Stateful (breaks horizontal scale):
user_sessions = {}  # stored in process memory
# If request goes to a different instance: session not found

# Stateless (horizontal scale works):
# Store session in Redis (shared across all instances)
session = redis.get(f"session:{session_id}")
```

Everything that must persist across requests goes to an external store:
- User sessions → Redis
- File uploads → S3/object storage
- Caches → Redis
- Rate limit counters → Redis
- Background job state → Queue / Database

## Load Balancing

```
Clients
  ↓
Load Balancer
  ↙     ↓     ↘
App1   App2   App3
  ↘     ↓     ↙
     Database
```

**Algorithms:**
| Algorithm | Use When |
|-----------|----------|
| Round Robin | Requests are similar in cost (most common) |
| Least Connections | Requests vary significantly in duration |
| IP Hash | You need same user → same server (sticky sessions without shared store) |
| Weighted | Servers have different capacities |

**Health checks:** load balancer polls `/health` — removes unhealthy instances automatically.

**Avoid single point of failure in LB:** run 2+ load balancers with a virtual IP (Keepalived, AWS ALB).

## Database Scaling

### Read Replicas

```
Write → Primary DB
Read  → Replica 1 | Replica 2 | Replica 3
```

Route all writes to primary. Route reads (reports, search, analytics) to replicas. Typical replication lag: < 100ms.

```python
# Route by operation type
def find_users(filters):
    return read_db.query("SELECT ...", filters)  # replica

def create_user(data):
    return write_db.query("INSERT ...", data)  # primary
```

### Sharding (Horizontal Partitioning)

Split a single large table across multiple databases:

```
# Hash-based sharding
shard = hash(user_id) % num_shards

User 1 → shard 0 → DB-0
User 2 → shard 1 → DB-1
User 3 → shard 0 → DB-0
```

Sharding is complex. Only use it when you've exhausted all other options (indexes, query optimization, caching, read replicas). Most applications never need it.

## Caching for Performance

Cache the results of expensive operations:

```
Response time without cache:
  DB query: 50ms + external API: 200ms + compute: 20ms = 270ms

Response time with cache hit:
  Redis lookup: 1ms = 1ms

Cache hit rate target: > 80%
```

See the `caching` skill for full implementation details.

## CDN for Static Assets

```
User (London) → CDN Edge (London) → Origin Server (US East)
                    ↓ cache hit
User (Paris) → CDN Edge (Frankfurt) [served from cache, no origin call]
```

Put behind CDN:
- Images, videos, documents
- JavaScript and CSS bundles (with content-hash filenames for cache busting)
- Rarely-changing API responses (public data)

Never put behind CDN:
- User-specific data
- Authentication endpoints
- Write operations

## Payload Optimization

```
# Compression: gzip saves 60-80% on JSON
Content-Encoding: gzip
Accept-Encoding: gzip, deflate, br

# Return only fields needed
GET /users?fields=id,email,name

# Pagination — never return unbounded collections
GET /products?limit=20&page=1
```

## Deployment Strategies for Zero Downtime

### Rolling Update
```yaml
# Kubernetes
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0   # never less than current count
    maxSurge: 1         # allow 1 extra pod during update
```

### Blue-Green
1. Prod traffic → Blue (current)
2. Deploy → Green (new version)
3. Test Green
4. Switch LB → Green
5. Blue stays for rollback
6. Delete Blue after confidence

### Canary
1. 95% traffic → stable
2. 5% traffic → canary (new version)
3. Monitor error rate, latency
4. Increase canary to 10%, 25%, 50%, 100%
5. Rollback: route all back to stable

## Performance Testing Before Production

```bash
# Load test with wrk
wrk -t12 -c400 -d30s http://localhost:8000/api/products

# Load test with k6
k6 run --vus 100 --duration 30s load-test.js

# Always load test:
# 1. Before going to production
# 2. Before expected high-traffic events
# 3. After significant architecture changes
```

Test scenarios:
- Normal load (average RPS)
- Peak load (2-3x average)
- Stress test (find breaking point)
- Soak test (sustained load for hours — find memory leaks)
