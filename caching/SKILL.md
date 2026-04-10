---
name: caching
description: Use this skill when implementing caching, choosing cache strategies, configuring Redis, setting TTLs, handling cache invalidation, using HTTP cache headers, or when the user asks about caching layers, cache eviction, cache-aside, write-through, read-through, ETags, CDN caching, or Redis patterns.
---

# Caching — Backend Engineering

## Core Mental Model

Caching trades memory for speed. You store the result of expensive work so you can return it faster on the next request without repeating the work.

**The three questions to answer before caching anything:**
1. Is this data expensive to compute/fetch? (CPU, DB query, external API)
2. Is it read frequently? (high read-to-write ratio)
3. Can I tolerate slightly stale data? (what is acceptable staleness?)

If the answer to all three is yes: cache it.

## Cache Layers (Outside-In)

```
Client Browser
    ↓
CDN (CloudFront, Cloudflare)      ← static assets, public API responses
    ↓
Reverse Proxy / Load Balancer     ← sometimes caches responses
    ↓
Application (in-process memory)   ← hot data (airport codes, config)
    ↓
Distributed Cache (Redis)         ← session data, API results, computed data
    ↓
Database Query Cache              ← query result caching
    ↓
Database
```

## Caching Strategies

### Cache-Aside (Lazy Loading) — The Most Common

```
1. Check cache for key
2a. Cache HIT → return cached value (fast path)
2b. Cache MISS → fetch from database
3. Store result in cache with TTL
4. Return result

Read path: cache → [miss] → DB → cache → client
Write path: DB → [invalidate] cache key
```

```python
def get_user(user_id):
    cache_key = f"user:{user_id}"
    cached = redis.get(cache_key)
    if cached:
        return deserialize(cached)
    
    user = user_repo.find(user_id)  # DB call only on cache miss
    redis.set(cache_key, serialize(user), ex=300)  # TTL: 5 minutes
    return user
```

**On write:** invalidate the cache key. Don't try to update it.
```python
def update_user(user_id, data):
    user = user_repo.update(user_id, data)
    redis.delete(f"user:{user_id}")  # invalidate — next read will repopulate
    return user
```

### Write-Through

```
Write path: DB → [immediately] cache → client
Read path: cache → [miss] → DB → cache → client
```

Write to both DB and cache on every write. Cache is always warm. More write latency, consistent reads.

Use when: read-heavy, writes are infrequent, cache should always reflect current state.

### Write-Behind (Write-Back)

```
Write path: cache → [async] → DB
Read path: cache always (DB may be behind)
```

Write to cache immediately. Async worker flushes to DB. Fastest writes, risk of data loss on crash.

Use for: high-write scenarios where losing a few writes is acceptable (analytics counters, view counts).

### Read-Through

Cache sits in front of DB. Cache handles DB fetching automatically. Application only talks to cache.

Use when: your cache library/framework supports it natively.

## Cache Eviction Policies

| Policy | Behavior | Use When |
|--------|----------|----------|
| TTL | Expires after time period | Most cases — predictable staleness |
| LRU | Evicts Least Recently Used | Fixed memory budget, all data equally important |
| LFU | Evicts Least Frequently Used | Some data is "hot", some rarely accessed |
| FIFO | Evicts oldest entry | Not usually optimal, avoid |

**Default: always set a TTL.** Don't cache forever — data changes.

## TTL Guidelines

| Data Type | Suggested TTL |
|-----------|---------------|
| User profile | 5–15 minutes |
| Product catalog | 30–60 minutes |
| Flight search results | 10 minutes |
| Hotel search results | 10 minutes |
| Exchange rates | 1 hour |
| Session data | 24 hours |
| Static reference data (airports, currencies) | 24 hours |
| Authentication tokens | match token expiry |
| Rate limit counters | rolling window |

## Cache Key Design

```python
# Pattern: service:entity:identifier:variant
"user:profile:usr_123"
"flight:search:JFK-LHR-2025-06-15-2A-economy"
"product:catalog:electronics:page:2"
"user:permissions:usr_123"
"session:sess_abc123"
"rate_limit:ip:192.168.1.1:login"
```

Rules:
- Namespaced with colons
- Include all dimensions that make results different
- Deterministic — same inputs always produce same key
- No special characters that need escaping

## Cache Invalidation — The Hard Part

Three approaches:

**1. TTL Expiry (time-based)**
- Simplest approach
- Acceptable for most data
- Accept that data may be up to TTL seconds stale

**2. Active Invalidation (event-based)**
```python
def update_product(product_id, data):
    product = product_repo.update(product_id, data)
    redis.delete(f"product:{product_id}")           # single item
    redis.delete_pattern(f"product:catalog:*")      # all catalog pages
    return product
```
- More complex but data is immediately consistent
- Use for data where staleness is not acceptable

**3. Versioned Keys**
```python
# Store a version number
version = redis.incr("product:catalog:version")

# Include version in all cache keys
cache_key = f"product:catalog:v{version}:page:2"

# Invalidate all: just increment version
redis.incr("product:catalog:version")
# Old keys become orphaned (cleaned up by TTL or background job)
```

## Redis Patterns

### String (simple key-value)
```python
redis.set("key", value, ex=300)       # set with 5min TTL
value = redis.get("key")
redis.delete("key")
```

### Hash (object fields)
```python
redis.hset("user:123", mapping={"name": "John", "email": "john@example.com"})
name = redis.hget("user:123", "name")
user = redis.hgetall("user:123")
redis.expire("user:123", 300)
```

### List (queue, recent items)
```python
redis.lpush("recent:views", item_id)   # push to head
redis.ltrim("recent:views", 0, 99)     # keep only last 100
items = redis.lrange("recent:views", 0, 9)  # get first 10
```

### Set (unique members)
```python
redis.sadd("user:123:roles", "admin", "editor")
roles = redis.smembers("user:123:roles")
redis.sismember("user:123:roles", "admin")  # check membership
```

### Sorted Set (leaderboard, priority queue)
```python
redis.zadd("leaderboard", {"player1": 500, "player2": 300})
top10 = redis.zrevrange("leaderboard", 0, 9, withscores=True)
```

### Atomic Counter (rate limiting)
```python
current = redis.incr(f"rate_limit:{user_id}:requests")
if current == 1:
    redis.expire(f"rate_limit:{user_id}:requests", 60)  # 1 minute window
if current > 100:
    raise RateLimitError()
```

## HTTP Caching Headers

For API responses that can be cached by browsers and CDNs:

```http
# No caching (sensitive data, user-specific)
Cache-Control: no-store

# Cache but always revalidate
Cache-Control: no-cache

# Cache for 5 minutes, browser only
Cache-Control: private, max-age=300

# Cache for 1 hour, CDN and browser
Cache-Control: public, max-age=3600

# With ETag for conditional requests
ETag: "abc123"
Cache-Control: public, max-age=300

# Client revalidates with:
If-None-Match: "abc123"
# Server responds 304 if unchanged (no body), 200 + new ETag if changed
```

## Caching Do's and Don'ts

**Do:**
- Always set a TTL
- Namespace cache keys
- Log cache hit/miss ratio — target > 80% hit rate
- Handle cache unavailability gracefully (fall through to DB)
- Cache at the right layer (not too granular, not too coarse)

**Don't:**
- Cache passwords, tokens, or sensitive credentials
- Cache user-specific data in shared caches without user-namespacing
- Cache in a way that bypasses authorization checks
- Store large objects (> 1MB) in cache without compression
- Cache error responses — only cache successful results
- Assume the cache always has data — always handle miss gracefully

## Distributed Cache for Multi-Instance Deployments

If you run multiple application servers, use a centralized cache (Redis). In-process caches are per-instance — different instances will have different cached values.

```
Instance 1 → Redis → DB
Instance 2 → Redis → DB   # same cache, consistent data
Instance 3 → Redis → DB
```

In-process memory cache is only safe for truly global, immutable reference data (country codes, airport codes loaded at startup).
