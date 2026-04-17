---
name: postgres-patterns
description: >-
  PostgreSQL best practices for query performance, connection management,
  security, schema design, concurrency, data access patterns, monitoring,
  and advanced features. Use when writing, reviewing, or optimizing SQL
  queries, database schemas, RLS policies, or Postgres configurations.
---

<!-- See README.md for attribution. -->

# PostgreSQL Patterns

Performance, security, and operational patterns for PostgreSQL. Rules are
organized by priority -- address CRITICAL items before lower tiers.

| Priority | Category                   | Impact      |
| -------- | -------------------------- | ----------- |
| 1        | Query Performance          | CRITICAL    |
| 2        | Connection Management      | CRITICAL    |
| 3        | Security and RLS           | CRITICAL    |
| 4        | Schema Design              | HIGH        |
| 5        | Concurrency and Locking    | MEDIUM-HIGH |
| 6        | Data Access Patterns       | MEDIUM      |
| 7        | Monitoring and Diagnostics | LOW-MEDIUM  |
| 8        | Advanced Features          | LOW         |

---

## 1. Query Performance [CRITICAL]

### Always EXPLAIN before optimizing

Never guess at query performance. Use `EXPLAIN (ANALYZE, BUFFERS)` to see
the actual execution plan.

```sql
-- WRONG: Guessing that an index will help
CREATE INDEX orders_date_idx ON orders (created_at);

-- RIGHT: Check the plan first
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE created_at > now() - interval '7 days';
-- Then decide if an index is warranted based on Seq Scan cost
```

### Index every foreign key column

Unindexed FKs cause sequential scans on JOINs and cascading deletes.

```sql
-- WRONG: FK exists but no index
ALTER TABLE orders ADD CONSTRAINT orders_customer_fk
  FOREIGN KEY (customer_id) REFERENCES customers(id);
-- JOIN on orders.customer_id triggers Seq Scan

-- RIGHT: Always pair FK with index
CREATE INDEX orders_customer_id_idx ON orders (customer_id);
```

### Use covering indexes with INCLUDE

When a query only needs a few columns, a covering index avoids heap fetches
entirely (index-only scan).

```sql
-- Seq Scan or Index Scan + heap fetch
SELECT email FROM users WHERE tenant_id = 42;

-- Index-only scan: no heap access needed
CREATE INDEX users_tenant_id_idx ON users (tenant_id) INCLUDE (email);
```

### CTEs as optimization fences (PG < 12)

In PostgreSQL versions before 12, CTEs are always materialized. They act as
optimization fences that prevent the planner from pushing predicates down.

```sql
-- PG < 12: CTE materializes ALL orders before filtering
WITH recent AS (
  SELECT * FROM orders
)
SELECT * FROM recent WHERE created_at > now() - interval '1 day';

-- Better for PG < 12: Use a subquery instead
SELECT * FROM (
  SELECT * FROM orders
) sub WHERE created_at > now() - interval '1 day';
```

In PG 12+, the planner can inline CTEs automatically. Use `MATERIALIZED`
or `NOT MATERIALIZED` to be explicit when it matters.

---

## 2. Connection Management [CRITICAL]

### Use a connection pooler

PostgreSQL forks a process per connection. Default `max_connections` is 100
and each connection consumes ~10 MB of RAM. Applications that open many
short-lived connections (serverless, microservices) exhaust this fast.

Place a pooler (PgBouncer, pgcat, or cloud-native equivalent) in front of
Postgres. Configure one pool per logical database.

```
Application --> PgBouncer (port 6432) --> PostgreSQL (port 5432)
                transaction mode          max_connections = 100
                pool_size = 20
```

### Set idle transaction timeout

Long-running idle transactions hold locks and prevent autovacuum.

```sql
-- Set at the database or role level
ALTER DATABASE mydb SET idle_in_transaction_session_timeout = '30s';
```

### Connection discipline

- Size connection pools to ~25% of `max_connections` per application
- Use `statement` or `transaction` pooling mode (not `session` for
  serverless)
- Monitor `pg_stat_activity` for idle-in-transaction connections

---

## 3. Security and RLS [CRITICAL]

### Enable RLS on all user-data tables

Any table that stores user-specific data MUST have RLS enabled. Without
it, any authenticated user can read all rows.

```sql
-- WRONG: No RLS -- any authenticated query returns all rows
CREATE TABLE documents (
  id bigint PRIMARY KEY,
  owner_id uuid NOT NULL,
  content text
);

-- RIGHT: Enable RLS and add a policy
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

CREATE POLICY documents_owner_policy ON documents
  USING ((SELECT current_setting('app.user_id', true)::uuid) = owner_id);
-- The application must SET app.user_id = '<uuid>' on each connection before queries.
-- Wrapping in SELECT ensures the setting is evaluated once, not per row.
```

### SECURITY DEFINER bypasses RLS

Functions declared `SECURITY DEFINER` run with the privileges of the
function owner, not the caller. This bypasses RLS.

```sql
-- WARNING: This function bypasses RLS on the documents table
CREATE FUNCTION get_all_documents()
RETURNS SETOF documents
LANGUAGE sql
SECURITY DEFINER
SET search_path = ''
AS $$
  SELECT * FROM public.documents;
$$;
```

Use `SECURITY DEFINER` only for administrative helper functions. Always
set `search_path = ''` to prevent search-path injection. Prefer
`SECURITY INVOKER` (the default) for application functions.

### Row-level policies over app-layer WHERE

RLS policies are enforced by the database engine. Application-layer WHERE
clauses can be forgotten, bypassed, or inconsistently applied.

```sql
-- WRONG: Relying on application code
SELECT * FROM documents WHERE owner_id = $1;
-- What if another endpoint forgets this clause?

-- RIGHT: RLS enforces it regardless of query
-- (policy from above handles it automatically)
SELECT * FROM documents;
```

---

## 4. Schema Design [HIGH]

### gen_random_uuid() over serial for PKs

UUIDs are globally unique regardless of table or shard. Serial IDs leak
row count and ordering information.

```sql
-- WRONG for new tables
CREATE TABLE orders (
  id serial PRIMARY KEY
);

-- RIGHT
CREATE TABLE orders (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid()
);
```

### Always TIMESTAMPTZ, not TIMESTAMP

`TIMESTAMP` (without time zone) silently discards timezone information,
leading to bugs when servers or users cross time zones.

```sql
-- WRONG
CREATE TABLE events (
  created_at timestamp DEFAULT now()
);

-- RIGHT
CREATE TABLE events (
  created_at timestamptz DEFAULT now()
);
```

### Partial indexes for sparse conditions

When a query filters on a rare condition, a partial index is smaller and
faster than a full index.

```sql
-- Full index: includes all rows, most of which are NOT 'pending'
CREATE INDEX orders_status_idx ON orders (status);

-- Partial index: only indexes the rows that matter
CREATE INDEX orders_pending_idx ON orders (created_at)
  WHERE status = 'pending';
```

### Partition when tables exceed 50-100M rows

Declarative partitioning by range (date) or list (tenant) keeps partitions
manageable and enables partition pruning.

```sql
CREATE TABLE events (
  id uuid DEFAULT gen_random_uuid(),
  created_at timestamptz NOT NULL,
  payload jsonb
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2025_q1 PARTITION OF events
  FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');
```

### Enum types for constrained string columns

Enums save storage and enforce valid values at the database level.

```sql
-- WRONG: Storing status as unconstrained text
CREATE TABLE orders (
  status text DEFAULT 'pending'
);

-- RIGHT: Enforce valid values
CREATE TYPE order_status AS ENUM ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled');

CREATE TABLE orders (
  status order_status DEFAULT 'pending'
);
```

---

## 5. Concurrency and Locking [MEDIUM-HIGH]

### FOR UPDATE SKIP LOCKED for queues

Building a job queue with SELECT ... FOR UPDATE causes contention. Add
`SKIP LOCKED` so workers grab different rows concurrently.

```sql
-- WRONG: Workers contend on the same row
SELECT * FROM jobs WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE;

-- RIGHT: Workers skip locked rows and grab the next available
SELECT * FROM jobs WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

### Advisory locks for application-level mutexes

When you need a distributed lock without locking table rows:

```sql
-- Acquire (non-blocking)
SELECT pg_try_advisory_lock(hashtext('invoice-generation'));

-- Release
SELECT pg_advisory_unlock(hashtext('invoice-generation'));
```

### Short transactions

Long transactions hold locks, block autovacuum, and increase transaction
ID wraparound risk. Keep transactions as short as possible.

```sql
-- WRONG: Long transaction with external HTTP call in the middle
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- ... application makes HTTP call to payment gateway ...
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- RIGHT: Do external work OUTSIDE the transaction
-- 1. Call payment gateway
-- 2. On success:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

### NOWAIT to detect contention

Use `NOWAIT` when you prefer to fail fast rather than block indefinitely.

```sql
SELECT * FROM inventory WHERE product_id = 42
FOR UPDATE NOWAIT;
-- Raises ERROR if the row is already locked, instead of waiting
```

---

## 6. Data Access Patterns [MEDIUM]

### Eliminate N+1 with CTEs and lateral joins

Fetching related rows in a loop (one query per parent row) is the most
common performance mistake in application code.

```sql
-- WRONG: N+1 from application code
-- for each order: SELECT * FROM line_items WHERE order_id = ?

-- RIGHT: Single query with JOIN
SELECT o.id, o.total, li.product_id, li.quantity
FROM orders o
JOIN line_items li ON li.order_id = o.id
WHERE o.customer_id = $1;

-- RIGHT: LATERAL join for complex per-row subqueries
SELECT o.id, latest.*
FROM orders o
CROSS JOIN LATERAL (
  SELECT * FROM shipments s
  WHERE s.order_id = o.id
  ORDER BY s.created_at DESC
  LIMIT 1
) latest;
```

### COPY over INSERT for bulk loading

For loading more than a few hundred rows, `COPY` is orders of magnitude
faster than individual INSERT statements.

```sql
-- WRONG: Row-by-row inserts
INSERT INTO events (id, payload) VALUES (...);
INSERT INTO events (id, payload) VALUES (...);
-- ... 10,000 times

-- RIGHT: Bulk load
COPY events (id, payload) FROM '/tmp/events.csv' WITH (FORMAT csv);

-- Or from application code: use the COPY protocol (e.g., pgx.CopyFrom in Go,
-- psycopg.copy_from in Python)
```

### Cursor-based pagination over OFFSET

OFFSET re-scans and discards rows from the start. Cursor-based (keyset)
pagination is stable and constant-time.

```sql
-- WRONG: OFFSET scales linearly
SELECT * FROM products ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
-- Scans and discards 10,000 rows

-- RIGHT: Cursor-based (keyset) pagination
SELECT * FROM products
WHERE created_at < $last_seen_created_at
ORDER BY created_at DESC
LIMIT 20;
```

---

## 7. Monitoring and Diagnostics

### Enable pg_stat_statements

This extension tracks execution statistics for all queries. Essential for
identifying slow and frequently-called queries.

```sql
-- Enable the extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 10 queries by total execution time
SELECT
  calls,
  round(total_exec_time::numeric, 2) AS total_ms,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

### Monitor locks with pg_locks

```sql
SELECT
  blocked.pid AS blocked_pid,
  blocking.pid AS blocking_pid,
  blocked.query AS blocked_query,
  blocking.query AS blocking_query
FROM pg_locks bl
JOIN pg_stat_activity blocked ON bl.pid = blocked.pid
JOIN pg_locks kl ON bl.locktype = kl.locktype
  AND bl.database IS NOT DISTINCT FROM kl.database
  AND bl.relation IS NOT DISTINCT FROM kl.relation
  AND bl.page IS NOT DISTINCT FROM kl.page
  AND bl.tuple IS NOT DISTINCT FROM kl.tuple
  AND bl.transactionid IS NOT DISTINCT FROM kl.transactionid
  AND bl.pid != kl.pid
  AND NOT bl.granted
JOIN pg_stat_activity blocking ON kl.pid = blocking.pid
WHERE kl.granted;
```

### Track autovacuum

Dead tuples trigger autovacuum. If autovacuum falls behind, table bloat
grows and queries slow down.

```sql
SELECT
  schemaname,
  relname,
  n_dead_tup,
  last_autovacuum,
  last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

### Log slow queries

```sql
-- Log queries slower than 500ms
ALTER DATABASE mydb SET log_min_duration_statement = 500;
```

---

## 8. Advanced Features

### GIN indexes for JSONB and full-text search

GIN indexes support containment (`@>`), existence (`?`), and full-text
`@@` operators.

```sql
-- JSONB containment queries
CREATE INDEX events_payload_idx ON events USING gin (payload jsonb_path_ops);
SELECT * FROM events WHERE payload @> '{"type": "purchase"}';

-- Full-text search
CREATE INDEX articles_search_idx ON articles USING gin (to_tsvector('english', title || ' ' || body));
SELECT * FROM articles WHERE to_tsvector('english', title || ' ' || body) @@ to_tsquery('postgres & performance');
```

### JSONB over JSON

`JSONB` is stored in a decomposed binary format. It supports indexing and
efficient operators. `JSON` stores raw text and must be re-parsed on every
access.

```sql
-- WRONG: json type -- no indexing, reparsed every access
CREATE TABLE logs (data json);

-- RIGHT: jsonb type -- indexable, binary storage
CREATE TABLE logs (data jsonb);
```

### Precomputed tsvector columns

For tables with frequent full-text searches, store a precomputed `tsvector`
column instead of computing it at query time.

```sql
ALTER TABLE articles ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (to_tsvector('english', coalesce(title, '') || ' ' || coalesce(body, ''))) STORED;

CREATE INDEX articles_search_vector_idx ON articles USING gin (search_vector);

-- Query uses the precomputed column
SELECT * FROM articles WHERE search_vector @@ to_tsquery('postgres & patterns');
```

### PostGIS: use ST_DWithin, not ST_Distance

`ST_DWithin` uses spatial indexes. `ST_Distance` computes distance for
every row, then filters.

```sql
-- WRONG: Computes distance for all rows, then filters
SELECT * FROM locations
WHERE ST_Distance(geom, ST_MakePoint(-73.99, 40.73)::geography) < 1000;

-- RIGHT: Uses spatial index to find candidates first
SELECT * FROM locations
WHERE ST_DWithin(geom, ST_MakePoint(-73.99, 40.73)::geography, 1000);
```
