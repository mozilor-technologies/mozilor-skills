---
name: postgres
description: PostgreSQL best practices covering schema design, indexing, query optimization, performance tuning, and production operations.
triggers:
  - "postgres"
  - "postgresql"
  - "database design"
  - "sql optimization"
  - "db performance"
  - "database best practices"
  - "pg tuning"
---

# PostgreSQL — Best Practices & Performance Patterns

## 1) Schema Design Principles

### Data Types

```sql
-- Use appropriate types — not everything is varchar
id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,  -- or ULID for sortability
email       TEXT NOT NULL,                                -- TEXT > VARCHAR in PG
status      TEXT CHECK (status IN ('active', 'inactive')), -- enum alternatives
amount      NUMERIC(10, 2) NOT NULL,                      -- not FLOAT for money
created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),           -- always with timezone
metadata    JSONB,                                         -- JSONB > JSON
tags        TEXT[],                                        -- arrays when appropriate
```

### Normalization Rules

Normalize to 3NF by default. Denormalize only for documented performance reasons:

```sql
-- Normalized: avoid redundancy
CREATE TABLE users (id UUID PRIMARY KEY, name TEXT, email TEXT);
CREATE TABLE orders (id UUID PRIMARY KEY, user_id UUID REFERENCES users(id), total NUMERIC);

-- Denormalized (document why): avoid JOIN on hot read path
CREATE TABLE order_summaries (
    order_id UUID PRIMARY KEY,
    user_id UUID,
    user_email TEXT,   -- denormalized for read performance
    total NUMERIC,
    FOREIGN KEY (order_id) REFERENCES orders(id)
);
```

### Naming Conventions

```sql
-- Tables: snake_case, plural
CREATE TABLE order_items (...);

-- Columns: snake_case
user_id, created_at, is_active

-- Indexes: descriptive name
CREATE INDEX idx_orders_user_id ON orders (user_id);
CREATE INDEX idx_orders_status_created ON orders (status, created_at DESC);

-- Foreign keys: fk_<table>_<referenced_table>
ALTER TABLE orders ADD CONSTRAINT fk_orders_users FOREIGN KEY (user_id) REFERENCES users(id);
```

## 2) Indexing Strategy

### When to Index

```sql
-- B-tree indexes: equality and range queries, ordering
CREATE INDEX idx_orders_user_id ON orders (user_id);
CREATE INDEX idx_products_price ON products (price);

-- Composite index: column order matters (most selective / most filtered first)
CREATE INDEX idx_orders_user_status ON orders (user_id, status);
-- This covers: WHERE user_id = X, WHERE user_id = X AND status = Y
-- Does NOT cover: WHERE status = Y alone

-- Partial index: only index the rows you query
CREATE INDEX idx_orders_pending ON orders (created_at)
WHERE status = 'pending';

-- Covering index: include extra columns to enable index-only scans
CREATE INDEX idx_users_email_inc ON users (email)
INCLUDE (id, name);

-- GIN index for JSONB and full-text search
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);
CREATE INDEX idx_articles_search ON articles USING GIN (to_tsvector('english', title || ' ' || body));
```

### Index Maintenance

```sql
-- Find unused indexes (run after extended operation period)
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexname NOT LIKE 'pk%';

-- Drop unused indexes safely
DROP INDEX CONCURRENTLY idx_old_unused;

-- Rebuild bloated indexes without locking
REINDEX INDEX CONCURRENTLY idx_orders_user_id;

-- Analyze after bulk operations
ANALYZE orders;
```

## 3) Query Optimization

### EXPLAIN ANALYZE

```sql
-- Always use EXPLAIN (ANALYZE, BUFFERS) — not just EXPLAIN
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.status = 'active'
GROUP BY u.id, u.name;

-- Look for:
-- "Seq Scan" on large tables → missing index
-- "Hash Join" on huge datasets → possible index improvement
-- Rows= actual vs estimated large discrepancy → run ANALYZE
-- Buffers: hit vs read → cache effectiveness
```

### Query Patterns

```sql
-- Use CTEs for readability (PG 12+ materializes only when needed)
WITH active_users AS (
    SELECT id, name FROM users WHERE status = 'active'
),
user_orders AS (
    SELECT user_id, COUNT(*) as order_count, SUM(total) as total_spent
    FROM orders
    WHERE created_at > NOW() - INTERVAL '30 days'
    GROUP BY user_id
)
SELECT au.name, uo.order_count, uo.total_spent
FROM active_users au
JOIN user_orders uo ON uo.user_id = au.id
ORDER BY uo.total_spent DESC;

-- Prefer EXISTS over IN for subqueries
-- BAD:
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE status = 'completed');
-- GOOD:
SELECT * FROM users u WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.status = 'completed'
);

-- Use window functions instead of correlated subqueries
SELECT
    id,
    amount,
    SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS running_total
FROM orders;
```

## 4) Performance Tuning (postgresql.conf)

Baseline settings for a 32GB RAM, 8-core production server:

```ini
# Memory
shared_buffers = 8GB               # 25% of RAM
work_mem = 64MB                    # per-sort/per-hash operation
effective_cache_size = 24GB        # 75% of RAM (planner hint)
maintenance_work_mem = 1GB         # VACUUM, CREATE INDEX

# WAL
wal_buffers = 64MB
checkpoint_completion_target = 0.9
max_wal_size = 4GB

# Connections
max_connections = 200              # use PgBouncer for more
# In app: pool to 4× CPU cores

# Query planner
random_page_cost = 1.1             # SSD: reduce from default 4.0
effective_io_concurrency = 200     # SSD: high value

# Logging (tune for your workload)
log_min_duration_statement = 100   # Log queries > 100ms
log_checkpoints = on
log_autovacuum_min_duration = 500
```

## 5) Partitioning

```sql
-- Range partitioning for time-series data
CREATE TABLE events (
    id         BIGSERIAL,
    user_id    UUID NOT NULL,
    event_type TEXT NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL,
    payload    JSONB
) PARTITION BY RANGE (occurred_at);

CREATE TABLE events_2026_q1 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');

CREATE TABLE events_2026_q2 PARTITION OF events
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');

-- Create indexes on partitions
CREATE INDEX ON events_2026_q1 (user_id, occurred_at);

-- Automatic partition management with pg_partman
```

## 6) Connection Pooling (PgBouncer)

```ini
# pgbouncer.ini
[databases]
myapp = host=127.0.0.1 port=5432 dbname=myapp

[pgbouncer]
pool_mode = transaction          # Best for web applications
max_client_conn = 1000
default_pool_size = 20           # = DB max_connections / num_app_instances
reserve_pool_size = 5
server_idle_timeout = 600
```

Monitor connections:

```sql
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
SELECT * FROM pg_stat_activity WHERE state = 'idle in transaction';
```

## 7) Transactions & Locking

```sql
-- Explicit transactions for multi-step operations
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = $1;
  UPDATE accounts SET balance = balance + 100 WHERE id = $2;
  INSERT INTO transfers (from_id, to_id, amount) VALUES ($1, $2, 100);
COMMIT;

-- Use SERIALIZABLE for financial operations
BEGIN ISOLATION LEVEL SERIALIZABLE;
...
COMMIT;

-- SELECT FOR UPDATE with SKIP LOCKED for queue processing
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

## 8) Migrations Best Practices

```sql
-- Always add columns as nullable first, backfill, then add NOT NULL
-- Step 1: Add nullable column
ALTER TABLE users ADD COLUMN phone TEXT;

-- Step 2: Backfill (in batches to avoid locking)
UPDATE users SET phone = '' WHERE phone IS NULL AND id BETWEEN $1 AND $2;

-- Step 3: Add constraint only after backfill complete
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;

-- Add indexes concurrently (no table lock)
CREATE INDEX CONCURRENTLY idx_users_phone ON users (phone);

-- Never do in production:
-- ALTER TABLE large_table ADD COLUMN col TEXT NOT NULL DEFAULT 'value'; -- locks entire table in old PG
-- In PG 11+: adding non-null column with DEFAULT is safe and instant
```

## 9) Monitoring Queries

```sql
-- Top slow queries
SELECT query, calls, total_exec_time / calls AS avg_ms, rows
FROM pg_stat_statements
ORDER BY avg_ms DESC
LIMIT 20;

-- Table bloat
SELECT tablename,
       n_dead_tup,
       n_live_tup,
       round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY dead_pct DESC;

-- Cache hit ratio (should be > 99%)
SELECT
    sum(heap_blks_hit) / sum(heap_blks_hit + heap_blks_read) AS cache_hit_ratio
FROM pg_statio_user_tables;

-- Long-running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '1 minute'
  AND state != 'idle';
```

## 10) Backup & Recovery

```bash
# Continuous archiving with pg_basebackup
pg_basebackup -h localhost -U replication -D /backup/base -P -Xs -R

# Logical backup
pg_dump -Fc myapp > myapp_$(date +%Y%m%d).dump

# Point-in-time recovery with WAL-G or pgBackRest
# Always test restores — untested backups are not backups
```

## 11) Security Checklist

- [ ] Use pg_hba.conf with scram-sha-256 authentication
- [ ] Application uses a dedicated low-privilege role (no superuser)
- [ ] Connection uses SSL/TLS (`sslmode=require`)
- [ ] Enable `pgaudit` for audit logging on sensitive tables
- [ ] Rotate credentials regularly with zero-downtime (connection pooler)
- [ ] Row Level Security (RLS) for multi-tenant data isolation
- [ ] Never use `SUPERUSER` for application accounts

## References

- [PostgreSQL Optimization Guide 2025](https://mediusware.com/blog/postgresql-performance-optimization)
- [PostgreSQL Performance Tuning](https://www.mydbops.com/blog/postgresql-parameter-tuning-best-practices)
- [PostgreSQL Official Docs](https://www.postgresql.org/docs/)
- [Use The Index, Luke](https://use-the-index-luke.com/)
