---
name: graceful-shutdown
description: Use this skill when implementing server shutdown handling, zero-downtime deployments, signal handling, or when the user asks about graceful shutdown, SIGTERM handling, in-flight request draining, deployment strategies, or how to restart a server without dropping active connections.
---

# Graceful Shutdown — Backend Engineering

## Why It Matters

Imagine a user is in the middle of a payment checkout. Your deployment pipeline restarts the server. Without graceful shutdown: their request is killed mid-way, the charge may be captured but booking not confirmed, data is inconsistent.

With graceful shutdown: the server stops accepting new requests, finishes all in-flight work, then exits cleanly.

## OS Signals

| Signal | Meaning | Handleable? |
|--------|---------|------------|
| SIGTERM | Polite termination request (Kubernetes, systemd) | Yes |
| SIGINT | Ctrl+C interrupt | Yes |
| SIGKILL | Immediate kill (no recovery) | No — cannot catch |
| SIGHUP | Hang up (terminal closed, reload config) | Yes |

Handle `SIGTERM` and `SIGINT`. Both should trigger graceful shutdown. Never catch `SIGKILL` (can't).

## Graceful Shutdown Sequence

```
Step 1: Receive SIGTERM or SIGINT
Step 2: Stop accepting new incoming requests
         → Load balancer health check returns unhealthy (503)
         → Load balancer stops routing new traffic here
         → Close server listener (no new connections accepted)
Step 3: Wait for in-flight requests to complete
         → Give them a deadline (e.g., 30 seconds)
         → Requests that complete before deadline: ✓
         → Requests still running at deadline: force-close + log
Step 4: Drain background job workers
         → Workers finish current job, do not pick up new ones
Step 5: Close database connections
         → Allow current transactions to complete or rollback
         → Close connection pool
Step 6: Close external service connections (Redis, message broker, etc.)
Step 7: Flush pending logs and metrics
Step 8: Exit cleanly with code 0
```

## Implementation

```python
import signal
import threading
import time

class GracefulServer:
    def __init__(self):
        self.shutdown_event = threading.Event()
        self.active_requests = 0
        self.active_requests_lock = threading.Lock()
        self.shutdown_timeout = 30  # seconds
        
        # Register signal handlers
        signal.signal(signal.SIGTERM, self._handle_shutdown)
        signal.signal(signal.SIGINT, self._handle_shutdown)
    
    def _handle_shutdown(self, signum, frame):
        log.info("Shutdown signal received", signal=signum)
        self.shutdown_event.set()
    
    def handle_request(self, request):
        if self.shutdown_event.is_set():
            # Reject new requests during shutdown
            return response(503, {"error": "Server shutting down"})
        
        with self.active_requests_lock:
            self.active_requests += 1
        
        try:
            return process_request(request)
        finally:
            with self.active_requests_lock:
                self.active_requests -= 1
    
    def shutdown(self):
        log.info("Starting graceful shutdown")
        
        # Stop accepting new requests (already handled by shutdown_event)
        
        # Wait for in-flight requests
        deadline = time.time() + self.shutdown_timeout
        while time.time() < deadline:
            with self.active_requests_lock:
                if self.active_requests == 0:
                    break
            log.info("Waiting for requests to complete", count=self.active_requests)
            time.sleep(0.5)
        
        if self.active_requests > 0:
            log.warn("Force closing with active requests", count=self.active_requests)
        
        # Cleanup
        db_pool.close_all()
        redis_client.close()
        background_worker.stop()
        log.info("Graceful shutdown complete")
```

## Health Check During Shutdown

The health check endpoint must return unhealthy during shutdown so the load balancer stops routing:

```python
def health_check():
    if server.shutdown_event.is_set():
        return response(503, {"status": "shutting_down"})
    
    db_ok = check_db_connection()
    redis_ok = check_redis_connection()
    
    if db_ok and redis_ok:
        return response(200, {"status": "healthy"})
    else:
        return response(503, {"status": "unhealthy", "db": db_ok, "redis": redis_ok})
```

Kubernetes liveness and readiness probes:
```yaml
livenessProbe:
  httpGet:
    path: /health/live    # is process alive?
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready   # is process ready to receive traffic?
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 5

terminationGracePeriodSeconds: 60  # give 60s before SIGKILL
```

Set `terminationGracePeriodSeconds` > your shutdown timeout + buffer.

## Kubernetes Deployment Lifecycle

```
Pod receives SIGTERM
    ↓
preStop hook runs (optional delay to let load balancer drain)
    ↓
App receives SIGTERM → graceful shutdown starts
    ↓
terminationGracePeriodSeconds timer starts
    ↓
App finishes in-flight requests, closes connections
    ↓
App exits with code 0
    ↓ (or if not done in time)
SIGKILL sent by Kubernetes → forced termination
```

Recommended `preStop` hook to give load balancer time to deregister:
```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "5"]   # wait 5s before sending SIGTERM to app
```

## Zero-Downtime Deployment Strategies

### Rolling Deployment
- Replace pods one at a time (or in small batches)
- Always have N-1 healthy pods running
- New pod passes health check → old pod receives SIGTERM → old pod drains

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0     # never reduce below current replica count
    maxSurge: 1           # allow 1 extra pod during update
```

### Blue-Green Deployment
- Run two identical environments (blue = current, green = new)
- Deploy new version to green
- Test green thoroughly
- Switch load balancer from blue to green
- Blue remains as instant rollback
- Terminate blue after confidence period

### Canary Deployment
- Route small % of traffic to new version (1%, 5%, 10%)
- Monitor error rates and latency on canary
- Gradually increase traffic if metrics are healthy
- Rollback by routing all traffic back to stable version

## Background Job Workers During Shutdown

```python
class Worker:
    def __init__(self):
        self.running = True
    
    def start(self):
        while self.running:
            job = queue.fetch(timeout=5)  # blocking poll with timeout
            if job:
                self.process(job)   # finish current job completely
        log.info("Worker stopped cleanly")
    
    def stop(self):
        self.running = False  # will stop after current job completes
        # Do NOT interrupt a running job
```

## Connection Draining Checklist

On shutdown, close in this order:
1. Stop accepting new connections (HTTP listener)
2. Finish in-flight HTTP requests
3. Signal background workers to stop
4. Wait for workers to finish current job
5. Close message broker connections (dequeue no more)
6. Close Redis connections
7. Close database connection pool (waits for active queries)
8. Flush buffered logs
9. Exit

## Testing Graceful Shutdown

```bash
# Start server
./server &
PID=$!

# Send traffic
ab -n 1000 -c 10 http://localhost:8000/api/health &

# Trigger graceful shutdown
kill -SIGTERM $PID

# Verify:
# 1. In-flight requests completed (not 502/503 in ab results)
# 2. Server exited with code 0
# 3. No "connection reset by peer" errors in client logs
# 4. No partial database writes
```
