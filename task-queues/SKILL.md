---
name: task-queues
description: Use this skill when implementing background jobs, async task processing, email sending, scheduled tasks, cron jobs, or when the user asks about task queues, job workers, message queues, async processing, retries, dead letter queues, job scheduling, or processing work outside the request lifecycle.
---

# Task Queues & Background Jobs — Backend Engineering

## Core Mental Model

A background job is any work that does not need to complete before you respond to the client. If it takes more than a few hundred milliseconds, or if it doesn't need to be in the response — move it to a background job.

**Request-Response (synchronous):** Client waits → Server does work → Client gets result
**Background Job (async):** Client gets immediate acknowledgment → Server does work later → Client notified separately (webhook, email, websocket)

## When to Use Background Jobs

| Use Case | Why |
|----------|-----|
| Email/SMS sending | External service — variable latency |
| Image/video processing | CPU/IO intensive |
| PDF generation | CPU intensive |
| Third-party API calls | Unpredictable latency, failure handling |
| Payment processing | Retry logic, reconciliation |
| Webhook delivery | Must retry on failure |
| Data exports | Large, long-running |
| Search index updates | Eventual consistency acceptable |
| Notification sending | Not blocking to user |
| Data aggregation | Batch processing |
| Database cleanup | Maintenance work |

## Architecture Components

```
Producer (your app)
    ↓ enqueue task
Queue/Broker (Redis, RabbitMQ, SQS)
    ↓ delivers task
Consumer / Worker (separate process)
    ↓ on failure
Dead Letter Queue (for permanently failed jobs)
```

**Producer:** the part of your application that creates tasks. Can be your HTTP server, a cron scheduler, or another worker.

**Queue/Broker:** persistent storage for tasks. Holds jobs until a worker picks them up. Survives process restarts.

**Consumer/Worker:** a separate process that picks up tasks and executes them. Can run multiple instances for parallelism.

**Dead Letter Queue (DLQ):** where jobs go after exhausting all retry attempts. Requires manual investigation.

## Task Design Principles

### Idempotency — Critical

A task must produce the same result if run multiple times with the same input. Workers can fail mid-task and retry — you cannot allow double-processing.

```python
# Wrong: sends duplicate emails if retried
def send_welcome_email(user_id):
    user = get_user(user_id)
    send_email(user.email, "Welcome!")  # sends twice if retried

# Correct: check-then-act with idempotency tracking
def send_welcome_email(user_id):
    if email_already_sent(user_id, "welcome"):
        return  # already done, skip
    user = get_user(user_id)
    send_email(user.email, "Welcome!")
    mark_email_sent(user_id, "welcome")
```

### Store References, Not Data

Store IDs in job payloads, not copies of data. Data changes between enqueue and execution.

```python
# Wrong: data may be stale by execution time
task = {
    "type": "send_invoice",
    "user_email": "john@example.com",  # what if email changed?
    "order_total": 150.00              # what if order was updated?
}

# Correct: store IDs, fetch fresh data at execution time
task = {
    "type": "send_invoice",
    "order_id": "ord_123"  # fetch order and user at execution
}
```

### Keep Tasks Small and Focused

One task = one unit of work. Do not build monolithic tasks that do 10 things.

If you need sequential steps: chain tasks. If parallel: fan out to multiple tasks.

## Retry Strategy

```python
# Exponential backoff with jitter
retry_delay(attempt):
    base = 60  # seconds
    max_delay = 3600  # 1 hour max
    delay = min(base * 2^attempt, max_delay)
    jitter = random(0, delay * 0.1)
    return delay + jitter

# Attempt 1: immediate or short delay
# Attempt 2: ~2 minutes
# Attempt 3: ~4 minutes
# Attempt 4: ~8 minutes
# Attempt 5: ~16 minutes → then → Dead Letter Queue
```

Configure per job type:
- Email: max 5 retries, backoff
- Webhook: max 10 retries, longer backoff
- Payment capture: max 3 retries (careful — check for duplicates first)
- Data sync: max 3 retries

## Dead Letter Queue

After max retries, job moves to DLQ. DLQ requires:
- Alerting: someone must be notified when jobs hit DLQ
- Visibility: admin dashboard showing DLQ contents
- Requeue: ability to move jobs back to main queue after fixing the issue
- Expiry: DLQ jobs should not live forever — set a retention (e.g., 7 days)

## Job Priorities

Not all jobs are equal. Use separate queues per priority:

```
critical queue    → payment confirmations, booking completions
high queue        → email confirmations, OTP codes
default queue     → notifications, reminders
low queue         → data exports, reports, analytics
maintenance queue → cleanup, archive jobs
```

Workers poll higher-priority queues first. Critical queue always processes before low.

## Scheduled Jobs (Cron)

Do not put cron logic in code. Manage schedules in config.

```yaml
jobs:
  - name: exchange_rate_refresh
    schedule: "0 * * * *"      # every hour
    handler: refresh_exchange_rates

  - name: ticket_time_limit_alerts
    schedule: "*/15 * * * *"   # every 15 minutes
    handler: check_expiring_pnrs

  - name: booking_expiry_cleanup
    schedule: "*/30 * * * *"   # every 30 minutes
    handler: expire_old_bookings

  - name: review_reminders
    schedule: "0 10 * * *"     # daily at 10:00 UTC
    handler: send_review_reminders
```

Cron expression format: `minute hour day month weekday`

## Preventing Duplicate Scheduled Job Runs

In multi-instance deployments, multiple instances can trigger the same cron job. Use distributed locking:

```python
def run_scheduled_job(job_name, handler):
    lock_key = f"cron:lock:{job_name}"
    lock_ttl = 55  # seconds (less than job interval)
    
    acquired = redis.set(lock_key, instance_id, ex=lock_ttl, nx=True)
    if not acquired:
        return  # another instance is running this job
    
    try:
        handler()
    finally:
        # Only release if we own the lock
        if redis.get(lock_key) == instance_id:
            redis.delete(lock_key)
```

## Worker Lifecycle

```python
# Worker should handle shutdown gracefully
def worker_loop():
    while not shutdown_signal:
        job = queue.fetch(timeout=5)  # blocking fetch with timeout
        if job:
            try:
                execute(job)
                job.acknowledge()
            except RetryableError:
                job.retry()
            except PermanentError:
                job.move_to_dlq()
            except Exception:
                log_error()
                job.retry()

# On SIGTERM:
# 1. Stop fetching new jobs
# 2. Finish current job
# 3. Exit cleanly
```

## Job Monitoring

Track per-job-type metrics:
- Queue depth (jobs waiting)
- Processing rate (jobs per second)
- Processing latency (time from enqueue to completion)
- Error rate
- Retry rate
- DLQ depth

Alerts:
- Queue depth exceeds threshold (backlog building)
- DLQ depth > 0 (requires investigation)
- Job error rate above 5%
- Job processing time > expected (worker might be stuck)

## Common Patterns

### Fan-Out
```python
# Process a bulk import by splitting into per-record jobs
def process_import(import_id):
    records = get_import_records(import_id)
    for record in records:
        enqueue("process_record", {"import_id": import_id, "record_id": record.id})
```

### Task Chaining (Pipeline)
```python
# Step 1 → on success → Step 2 → on success → Step 3
def process_payment(payment_id):
    payment = capture_payment(payment_id)
    if payment.success:
        enqueue("issue_ticket", {"booking_id": payment.booking_id})

def issue_ticket(booking_id):
    ticket = call_sabre_ticketing(booking_id)
    if ticket.success:
        enqueue("send_eticket_email", {"booking_id": booking_id})
```

### Batch Processing
```python
# Collect N items then process in batch
def batch_send_notifications(user_ids):
    # Process in chunks of 100 to avoid memory issues
    for chunk in chunks(user_ids, 100):
        for user_id in chunk:
            send_notification(user_id)
        sleep(0.1)  # rate limiting for external service
```
