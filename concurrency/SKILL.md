---
name: concurrency
description: Use this skill when implementing concurrent request handling, async I/O, thread pools, worker processes, or when the user asks about concurrency vs parallelism, I/O-bound vs CPU-bound operations, event loops, async/await, race conditions, deadlocks, mutex, connection pooling, or handling multiple operations simultaneously.
---

# Concurrency & Parallelism — Backend Engineering

## Concurrency vs Parallelism

**Concurrency:** handling multiple tasks by interleaving their execution. One worker, many tasks in progress simultaneously. Task A starts, while Task A waits for I/O, Task B runs.

**Parallelism:** executing multiple tasks simultaneously. Multiple workers, each processing a separate task at the same time. Requires multiple CPU cores.

```
Concurrency (1 core, 3 tasks interleaved):
─────────────────────────────────────────
Task A: ████░░░░░░████░░░░████
Task B: ░░░░████░░░░████░░░░░
Task C: ░░░░░░░░████░░░░████░
         ████ = running   ░░░ = waiting for I/O

Parallelism (3 cores, 3 tasks running simultaneously):
─────────────────────────────────────────
Core 1: ████████████████████████
Core 2: ████████████████████████
Core 3: ████████████████████████
```

A backend server needs both: concurrency (handle many requests at once) and parallelism (use all CPU cores).

## I/O-Bound vs CPU-Bound

**I/O-Bound:** time is spent waiting for external things.
- Network requests (database, external API, Redis)
- File system reads/writes
- Most backend web requests

**CPU-Bound:** time is spent computing.
- Image processing, video encoding
- Data aggregation, heavy calculations
- Cryptography, compression

**Why it matters:** the right concurrency strategy depends on whether your workload is I/O-bound or CPU-bound.

## Runtime Concurrency Models

### Node.js — Single-Threaded Event Loop

Node.js runs JavaScript on a single thread. It handles concurrency by never blocking — all I/O is async.

```javascript
// WRONG — blocks the event loop, kills throughput
const data = fs.readFileSync('large-file.txt')  // blocking!

// CORRECT — non-blocking I/O
const data = await fs.promises.readFile('large-file.txt')
// While waiting for disk, event loop processes other requests
```

Event loop handles I/O-bound work efficiently. For CPU-bound work: use Worker Threads or a separate process.

### Python — GIL and Async

Python has the Global Interpreter Lock (GIL) — only one Python thread runs at a time.
- Use `asyncio` for I/O-bound concurrency
- Use `multiprocessing` for CPU-bound parallelism
- Use threads for I/O-bound with blocking libraries

```python
# Async I/O (correct for I/O-bound)
async def get_user(user_id):
    return await db.fetch_one("SELECT * FROM users WHERE id = $1", user_id)

# Multiprocessing for CPU-bound
from multiprocessing import Pool
with Pool(processes=4) as pool:
    results = pool.map(process_image, image_list)
```

### Go — Goroutines

Go goroutines are lightweight, cheap to create (a few KB stack). The Go runtime multiplexes them across OS threads automatically.

```go
// Spawn a goroutine — lightweight, thousands can run simultaneously
go func() {
    result := callExternalAPI()
    ch <- result
}()

// Channels for communication between goroutines
ch := make(chan Result)
go worker(ch)
result := <-ch
```

Go is designed for concurrent servers. Goroutines are the right model for both I/O and CPU-bound work.

### Java — Thread Pool

Java uses OS threads. Thread pools manage a fixed number of threads and queue work:

```java
ExecutorService pool = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors() * 2);

CompletableFuture<User> future = CompletableFuture.supplyAsync(() -> {
    return userRepository.findById(id);
}, pool);
```

## Connection Pooling — Every External Connection

Creating a new database connection for every request is expensive (TCP handshake, authentication, TLS). Use a pool.

```python
# Pool configuration
pool = create_pool(
    min_size=5,      # always keep 5 connections warm
    max_size=20,     # never exceed 20 connections
    timeout=5,       # wait max 5s to acquire connection from pool
    max_lifetime=1800,  # recycle connections every 30 minutes
)

# Pool acquires connection from pool, returns it after request
async with pool.acquire() as conn:
    result = await conn.fetch("SELECT * FROM users")
# Connection returned to pool automatically
```

Same pattern for Redis, HTTP clients, external API clients.

## Async/Await — The Pattern for I/O-Bound Backends

```python
# Concurrent I/O — do both calls simultaneously, not sequentially
async def get_order_page(order_id, user_id):
    # Sequential (slow): total time = query1 + query2
    order = await order_repo.find(order_id)
    user = await user_repo.find(user_id)
    
    # Concurrent (fast): total time = max(query1, query2)
    order, user = await asyncio.gather(
        order_repo.find(order_id),
        user_repo.find(user_id)
    )
    return { "order": order, "user": user }
```

Fan-out pattern — launch multiple I/O operations, collect results:
```python
async def search_all_verticals(query, dates):
    flights, hotels, cars = await asyncio.gather(
        search_flights(query, dates),
        search_hotels(query, dates),
        search_cars(query, dates),
    )
    return { "flights": flights, "hotels": hotels, "cars": cars }
```

## Race Conditions

A race condition occurs when two concurrent operations read-then-write shared state, and the result depends on timing.

### Database-Level Race Condition

```python
# WRONG — race condition on inventory
def reserve_seat(seat_id, user_id):
    seat = db.query("SELECT * FROM seats WHERE id = $1", seat_id)
    if seat.status == 'available':
        # Between reading and writing, another request could read too
        db.query("UPDATE seats SET status='reserved', user_id=$2 WHERE id=$1", seat_id, user_id)

# CORRECT — atomic conditional update
def reserve_seat(seat_id, user_id):
    result = db.query("""
        UPDATE seats 
        SET status='reserved', user_id=$2 
        WHERE id=$1 AND status='available'
        RETURNING id
    """, seat_id, user_id)
    
    if not result:
        raise ConflictError("Seat already taken")  # someone else got it first
```

### Optimistic Locking

```python
# Read with version
order = db.query("SELECT *, version FROM orders WHERE id = $1", order_id)

# Write only if version hasn't changed
result = db.query("""
    UPDATE orders 
    SET status='processing', version=version+1
    WHERE id=$1 AND version=$2
""", order_id, order.version)

if result.rowcount == 0:
    raise ConflictError("Order was modified by another process, please retry")
```

### Distributed Locking (Redis)

```python
# Acquire lock for a critical operation across multiple instances
def reserve_booking(property_id, dates, user_id):
    lock_key = f"lock:booking:{property_id}:{dates.check_in}:{dates.check_out}"
    
    # SET key value NX EX (set only if not exists, with TTL)
    acquired = redis.set(lock_key, instance_id, nx=True, ex=30)
    
    if not acquired:
        raise ConflictError("Booking in progress, please retry")
    
    try:
        # Safe to proceed — only one process holds this lock
        create_booking(property_id, dates, user_id)
    finally:
        # Release lock only if we own it
        if redis.get(lock_key) == instance_id:
            redis.delete(lock_key)
```

## Deadlocks

A deadlock occurs when two processes each hold a lock the other needs.

```
Process A holds lock on Table1, waiting for lock on Table2
Process B holds lock on Table2, waiting for lock on Table1
→ Neither can proceed → Deadlock
```

Prevention:
1. **Consistent lock ordering**: always acquire locks in the same order across all code paths
2. **Short lock scopes**: acquire lock → do minimal work → release lock
3. **Timeout on lock acquisition**: if can't acquire in N seconds, fail rather than wait forever
4. **Retry on deadlock**: database detects deadlocks and returns an error — retry the transaction

```python
# Always lock tables in alphabetical order — prevents ordering deadlocks
def transfer_funds(from_account, to_account, amount):
    # Sort accounts to ensure consistent lock order
    first, second = sorted([from_account, to_account])
    
    with db.transaction():
        lock_account(first)
        lock_account(second)
        
        debit(from_account, amount)
        credit(to_account, amount)
```

## Backpressure

When producers send work faster than consumers can process:

```python
class BoundedQueue:
    def __init__(self, max_size=1000):
        self.queue = asyncio.Queue(maxsize=max_size)
    
    async def enqueue(self, item):
        try:
            await asyncio.wait_for(
                self.queue.put(item),
                timeout=1.0  # wait max 1s for space in queue
            )
        except asyncio.TimeoutError:
            # Return 429 — we're overloaded
            raise RateLimitError("Server too busy, try again")
    
    async def process(self):
        while True:
            item = await self.queue.get()
            await handle(item)
```

## Thread Pool Sizing

For CPU-bound work:
```
optimal_threads ≈ number of CPU cores
# More threads than cores → context switching overhead exceeds benefit
```

For I/O-bound work:
```
optimal_threads ≈ CPU cores × (1 + wait_time / compute_time)
# Example: if each request spends 90% waiting and 10% computing:
# optimal = 4 cores × (1 + 9) = 40 threads
```

For async I/O (Node.js, Python asyncio, Go goroutines): don't think in threads — just don't block.

## Monitoring Concurrency

Metrics to watch:
```
active_connections          # current open connections
connection_pool_active      # connections in use from pool
connection_pool_waiting     # requests waiting for a connection
goroutines / threads        # current goroutine/thread count
event_loop_lag_ms           # (Node.js) how long events wait
queue_depth                 # background job queue depth
async_task_count            # current concurrent async tasks
```

Alert on:
- Connection pool near full (> 90% active)
- Queue depth growing (consumers can't keep up)
- Event loop lag > 100ms (something is blocking)
