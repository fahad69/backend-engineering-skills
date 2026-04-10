---
name: databases
description: Use this skill when designing database schemas, writing SQL queries, setting up indexes, working with transactions, handling database relationships, connection pooling, migrations, or when the user asks about ACID properties, CAP theorem, normalization, query optimization, N+1 problems, ORMs, database locking, isolation levels, or PostgreSQL.
---

# Databases — Backend Engineering

## Why Databases

Databases persist state across requests, restarts, and deployments. Without them, every server restart loses all data. They also handle concurrent access correctly — something files cannot do safely.

## Relational Databases — The Default Choice

Use a relational database (PostgreSQL, MySQL) unless you have a measured, specific reason not to. They give you ACID, relationships, and a decades-old ecosystem.

**PostgreSQL is the preferred default.** It has better standards compliance, superior JSON support, and more advanced features than MySQL.

## ACID Properties — The Guarantee

Every database transaction either fully completes or fully rolls back. There is no partial success.

| Property | Meaning |
|----------|---------|
| **Atomicity** | All operations in a transaction succeed, or none do |
| **Consistency** | Data always moves from one valid state to another |
| **Isolation** | Concurrent transactions don't interfere with each other |
| **Durability** | Once committed, data survives crashes |

## Schema Design Rules

### Primary Keys
```sql
-- Prefer UUID for distributed systems (no coordination needed)
id UUID PRIMARY KEY DEFAULT gen_random_uuid()

-- Auto-increment for simple internal tables (simpler, faster)
id SERIAL PRIMARY KEY
```

### Foreign Keys — Always Define Them
```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    -- ON DELETE RESTRICT: prevents deleting user if orders exist
    -- ON DELETE CASCADE: deletes orders when user deleted
    -- ON DELETE SET NULL: sets user_id to NULL when user deleted
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Constraints — Enforce Invariants at DB Level
```sql
email VARCHAR(255) NOT NULL UNIQUE
age INTEGER CHECK (age >= 0 AND age <= 120)
status VARCHAR(20) NOT NULL CHECK (status IN ('active', 'inactive', 'suspended'))
price NUMERIC(10,2) NOT NULL CHECK (price >= 0)
quantity INTEGER NOT NULL DEFAULT 0 CHECK (quantity >= 0)
```

### Always Include These Timestamp Columns
```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
-- deleted_at for soft deletes:
deleted_at TIMESTAMPTZ
```

### Soft Delete Pattern
```sql
-- Don't DELETE rows. Set deleted_at:
UPDATE users SET deleted_at = NOW() WHERE id = $1

-- Filter in every query:
SELECT * FROM users WHERE deleted_at IS NULL

-- Or use a view:
CREATE VIEW active_users AS SELECT * FROM users WHERE deleted_at IS NULL
```

## Normalization

Normalization removes data redundancy. Store each piece of information in exactly one place.

```sql
-- Wrong: storing city in every order
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    user_id UUID,
    user_city VARCHAR(100),    -- duplicated from users table
    ...
);

-- Correct: reference the user
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id),
    -- get city by joining users
    ...
);
```

Only denormalize when you have measured performance problems and joining is the bottleneck.

## Indexing — The Most Impactful Performance Lever

```sql
-- Index foreign keys (always)
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Index columns in WHERE clauses
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_users_email ON users(email);

-- Composite index (order matters — most selective first)
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Partial index (index only a subset)
CREATE INDEX idx_active_users ON users(email) WHERE deleted_at IS NULL;

-- Unique index
CREATE UNIQUE INDEX idx_users_email_unique ON users(email) WHERE deleted_at IS NULL;
```

**Index every:**
- Primary key (automatic)
- Foreign key column
- Column in every WHERE clause you use in queries
- Column in ORDER BY if frequently sorted
- Column in JOIN conditions

**Do NOT index:**
- Columns rarely used in queries
- Low-cardinality columns (boolean, status with 2 values) unless combined with other columns
- Every column "just in case" — indexes slow down writes

## Query Analysis

```sql
-- Use EXPLAIN ANALYZE to understand query execution
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = $1 AND status = 'active';

-- Look for:
-- "Seq Scan" → full table scan, missing index
-- "Index Scan" → using index (good)
-- "Hash Join" vs "Nested Loop" → different join strategies
-- Actual rows vs estimated rows → statistics may be stale (run ANALYZE)
```

## Transactions — When to Use

Use transactions when multiple operations must all succeed or all fail:

```sql
BEGIN;
    INSERT INTO orders (user_id, total) VALUES ($1, $2) RETURNING id;
    UPDATE inventory SET stock = stock - $3 WHERE product_id = $4;
    INSERT INTO order_items (order_id, product_id, quantity) VALUES ($5, $4, $3);
COMMIT;
-- If any step fails, ROLLBACK automatically undoes all changes
```

Use transactions for:
- Creating related records together
- Updating multiple tables atomically
- Read-then-write operations (optimistic locking)
- Any operation that would leave data inconsistent if partially applied

## Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|---------------------|--------------|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | ✓ Protected | Possible | Possible |
| Repeatable Read | ✓ Protected | ✓ Protected | Possible |
| Serializable | ✓ Protected | ✓ Protected | ✓ Protected |

PostgreSQL default: **Read Committed**. This is correct for most use cases.

Use **Serializable** for financial transactions where consistency is critical.

## Connection Pooling — Always Required

Never open a new database connection per request. Connections are expensive.

```
# Configuration
pool_min:     5    # minimum connections to maintain
pool_max:     20   # maximum connections (match DB max_connections - headroom)
pool_idle_timeout: 10m
pool_connection_timeout: 5s
```

PostgreSQL has a hard limit on concurrent connections (`max_connections`, default 100). A single server with max=100 cannot handle 1000 concurrent requests without a pool. Use PgBouncer for large-scale pooling.

## N+1 Problem — The Silent Killer

```python
# WRONG: N+1 — one query for orders, then one per order for user
orders = order_repo.find_all()           # 1 query
for order in orders:
    order.user = user_repo.find(order.user_id)  # N queries

# CORRECT: eager load with JOIN
orders = order_repo.find_all_with_users()  # 1 query with JOIN
# SELECT orders.*, users.* FROM orders JOIN users ON orders.user_id = users.id
```

Detect N+1: log all queries in development. Any query that repeats N times in a single request is N+1.

## Database Migrations

Every schema change goes through a migration file. Never change the schema manually in production.

```
migrations/
├── 001_create_users_table.sql
├── 002_add_email_index_to_users.sql
├── 003_create_orders_table.sql
└── 004_add_status_to_orders.sql
```

Rules:
- Migrations are append-only — never edit an existing migration
- Migrations must be reversible — always write `down` migration
- Run migrations before deploying new code
- Never deploy code that requires a migration before running the migration
- Zero-downtime migrations: add columns as nullable → backfill → add NOT NULL constraint

## Common SQL Patterns

### Safe Upsert
```sql
INSERT INTO user_settings (user_id, key, value)
VALUES ($1, $2, $3)
ON CONFLICT (user_id, key) DO UPDATE SET value = EXCLUDED.value, updated_at = NOW();
```

### Pagination with Total Count
```sql
SELECT *, COUNT(*) OVER() AS total_count
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT $2 OFFSET $3;
```

### Safe Concurrent Update (Optimistic Locking)
```sql
UPDATE orders
SET status = 'processing', version = version + 1, updated_at = NOW()
WHERE id = $1 AND version = $2;
-- Check affected rows: if 0, another process updated first → retry
```

## Database Security

- Application user has only necessary permissions: `SELECT`, `INSERT`, `UPDATE`, `DELETE`
- Application user does NOT have: `DROP`, `ALTER`, `TRUNCATE`, `CREATE`
- Never use superuser in application code
- Migrations run with a separate migration user (has DDL permissions)
- All queries use parameterized statements — never string interpolation
- Database port not exposed publicly — only accessible from application servers
