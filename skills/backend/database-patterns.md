# Database Patterns — Schema Design & Query Reference

| Field          | Value                                                  |
|----------------|--------------------------------------------------------|
| Version        | 1.1                                                    |
| Last Updated   | 2026-06-04                                             |
| Applicability  | PostgreSQL 15+, applicable to most relational databases |
| Dependencies   | PostgreSQL; optionally Alembic, Prisma, or Supabase CLI for migrations |

---

## Normalization vs. Denormalization

### When to Normalize (3NF+)

- Transactional systems (OLTP) where data integrity matters.
- When write frequency is high relative to reads.
- When storage cost matters more than query latency.

```sql
-- Normalized: separate tables, foreign keys
CREATE TABLE authors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL
);

CREATE TABLE books (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title TEXT NOT NULL,
    author_id UUID NOT NULL REFERENCES authors(id)
);
```

### When to Denormalize

- Read-heavy dashboards, analytics, search results.
- When JOIN performance is measurably unacceptable.
- Caching layers (materialized views, read replicas).

```sql
-- Denormalized: embed author name for fast reads
CREATE TABLE books_denormalized (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title TEXT NOT NULL,
    author_id UUID NOT NULL,
    author_name TEXT NOT NULL  -- duplicated, must be kept in sync
);
```

**Rules:**
- Start normalized. Denormalize only when you have measured performance data.
- If you denormalize, document the sync mechanism (trigger, application code, event).
- Materialized views are a safe middle ground — denormalized reads without data duplication in base tables.

---

## Indexing Strategy

### Index Types

| Type        | Use Case                                   | Syntax                                      |
|-------------|-------------------------------------------|----------------------------------------------|
| B-tree      | Equality, range, sorting (default)         | `CREATE INDEX idx ON t(col)`                 |
| Hash        | Equality only, no range                    | `CREATE INDEX idx ON t USING hash(col)`      |
| GIN         | Full-text search, JSONB, arrays            | `CREATE INDEX idx ON t USING gin(col)`       |
| GiST        | Geometric, range types, proximity          | `CREATE INDEX idx ON t USING gist(col)`      |
| BRIN        | Large tables with naturally ordered data   | `CREATE INDEX idx ON t USING brin(col)`      |

### Index Design Rules

```sql
-- Composite index: column order matters (leftmost prefix rule)
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);
-- This index supports:
--   WHERE user_id = X
--   WHERE user_id = X AND created_at > Y
--   WHERE user_id = X ORDER BY created_at DESC
-- It does NOT efficiently support:
--   WHERE created_at > Y  (without user_id)

-- Partial index: index only rows that matter
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';

-- Expression index
CREATE INDEX idx_users_email_lower ON users(lower(email));

-- Covering index (include columns to enable index-only scans)
CREATE INDEX idx_orders_user_covering ON orders(user_id)
INCLUDE (total_amount, status);
```

**Rules:**
- Index columns that appear in `WHERE`, `JOIN ON`, and `ORDER BY`.
- Avoid indexing low-cardinality columns alone (e.g., boolean flags) unless combined with a partial index.
- Monitor unused indexes: `pg_stat_user_indexes` where `idx_scan = 0`.
- Every index slows down writes. Benchmark before adding.

---

## Migration Management

### Principles

1. **Migrations are append-only.** Never edit a migration that has been applied to any shared environment.
2. **Each migration does one thing.** Separate schema changes from data migrations.
3. **Migrations must be reversible** when possible (include `down` / rollback SQL).
4. **Use transactions.** Wrap DDL in a transaction (PostgreSQL supports transactional DDL).

### File Naming Convention

```
migrations/
├── 20260301_001_create_users_table.sql
├── 20260301_002_create_orders_table.sql
├── 20260305_001_add_orders_status_index.sql
└── 20260306_001_add_users_avatar_url.sql
```

### Safe Migration Patterns

```sql
-- Adding a column (safe, no lock issues)
ALTER TABLE users ADD COLUMN avatar_url TEXT;

-- Adding a NOT NULL column to an existing table (safe pattern)
ALTER TABLE users ADD COLUMN role TEXT DEFAULT 'member';
-- Then later, if removing the default:
ALTER TABLE users ALTER COLUMN role SET NOT NULL;
ALTER TABLE users ALTER COLUMN role DROP DEFAULT;

-- Renaming a column (requires coordinated deploy)
-- Step 1: Add new column
ALTER TABLE users ADD COLUMN display_name TEXT;
-- Step 2: Backfill
UPDATE users SET display_name = name WHERE display_name IS NULL;
-- Step 3: Application reads from both, writes to both
-- Step 4: Drop old column after cutover
ALTER TABLE users DROP COLUMN name;

-- Creating an index without blocking writes
CREATE INDEX CONCURRENTLY idx_orders_user ON orders(user_id);
-- NOTE: Cannot run inside a transaction. Must be a standalone migration.
```

### Dangerous Operations

| Operation                     | Risk                  | Mitigation                                |
|-------------------------------|-----------------------|-------------------------------------------|
| `DROP COLUMN`                 | Data loss             | Backup first, deploy in stages            |
| `ALTER COLUMN TYPE`           | Full table rewrite    | Add new column, migrate, drop old         |
| `ADD COLUMN ... NOT NULL`     | Fails if rows exist   | Add with DEFAULT, then set NOT NULL       |
| `CREATE INDEX` (non-concurrent) | Table lock          | Use `CONCURRENTLY`                        |
| `TRUNCATE` / `DROP TABLE`    | Data loss             | Soft-delete or archive first              |

---

## Query Optimization

### Reading EXPLAIN Output

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = $1 AND status = 'pending';
```

**Key things to look for:**
- `Seq Scan` on large tables — missing index.
- `Nested Loop` with high row estimates — consider a hash or merge join.
- `Sort` — can often be eliminated with an index on the ORDER BY columns.
- `Rows Removed by Filter` — index is not selective enough.
- `Buffers: shared hit` vs `shared read` — cache hit ratio.

### Common Optimizations

```sql
-- Pagination: keyset (cursor) pagination, NOT OFFSET
-- Bad (slow on deep pages):
SELECT * FROM items ORDER BY created_at DESC LIMIT 20 OFFSET 10000;

-- Good (fast regardless of page depth):
SELECT * FROM items
WHERE created_at < $last_seen_created_at
ORDER BY created_at DESC
LIMIT 20;

-- Count optimization: use an estimate for UI "total" counts
SELECT reltuples::bigint AS estimate
FROM pg_class
WHERE relname = 'items';

-- Batch inserts
INSERT INTO items (name, price)
VALUES
    ('a', 10),
    ('b', 20),
    ('c', 30);
-- Or use COPY for bulk loads.

-- Avoid SELECT *
SELECT id, name, price FROM items WHERE ...;
```

---

## Connection Pooling

### Why It Matters

PostgreSQL forks a process per connection. Without pooling:
- 100 connections = 100 OS processes.
- Default `max_connections` is 100.
- Each connection uses ~5-10 MB RAM.

### Options

| Tool       | Where It Runs        | Mode           |
|------------|---------------------|----------------|
| PgBouncer  | Separate process     | Transaction (recommended) or Session |
| Supavisor  | Supabase managed     | Transaction    |
| App-level  | In application       | asyncpg pool, SQLAlchemy pool |

### Application-Level Pool (asyncpg)

```python
import asyncpg

pool = await asyncpg.create_pool(
    dsn="postgresql://...",
    min_size=5,
    max_size=20,
    max_inactive_connection_lifetime=300,  # seconds
    command_timeout=30,
)

async with pool.acquire() as conn:
    rows = await conn.fetch("SELECT * FROM users WHERE id = $1", user_id)
```

**Rules:**
- Pool size = `(2 * CPU cores) + effective_spindle_count`. Start with 10-20.
- Always release connections (use `async with` or try/finally).
- Set statement and connection timeouts to prevent hanging.
- With Supabase, prefer their Supavisor connection string (port 6543) for pooled connections.

---

## Transaction Patterns

```sql
-- Basic transaction
BEGIN;
    INSERT INTO orders (user_id, total) VALUES ($1, $2) RETURNING id;
    INSERT INTO order_items (order_id, product_id, qty) VALUES (...);
COMMIT;

-- Savepoints for partial rollback
BEGIN;
    INSERT INTO orders (...) VALUES (...);
    SAVEPOINT before_notification;
    INSERT INTO notifications (...) VALUES (...);  -- if this fails...
    ROLLBACK TO before_notification;                -- ...rollback only this part
COMMIT;
```

### Application-Level Pattern

```python
async with pool.acquire() as conn:
    async with conn.transaction():
        order_id = await conn.fetchval(
            "INSERT INTO orders (user_id, total) VALUES ($1, $2) RETURNING id",
            user_id, total,
        )
        await conn.execute(
            "INSERT INTO order_items (order_id, product_id, qty) VALUES ($1, $2, $3)",
            order_id, product_id, qty,
        )
        # Transaction auto-commits on exit, auto-rolls-back on exception
```

**Rules:**
- Keep transactions short. Never hold a transaction open during user input or external API calls.
- Use `SELECT ... FOR UPDATE` sparingly — prefer optimistic concurrency with version columns.
- For idempotency, use `INSERT ... ON CONFLICT` (upsert).

---

## Full-Text Search

```sql
-- Add a tsvector column
ALTER TABLE articles ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(body, '')), 'B')
    ) STORED;

-- Index it
CREATE INDEX idx_articles_search ON articles USING gin(search_vector);

-- Query
SELECT id, title, ts_rank(search_vector, query) AS rank
FROM articles, to_tsquery('english', 'postgres & performance') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;

-- Phrase search
SELECT * FROM articles
WHERE search_vector @@ phraseto_tsquery('english', 'connection pooling');

-- Autocomplete with prefix matching
SELECT * FROM articles
WHERE search_vector @@ to_tsquery('english', 'conn:*');
```

**When to use Postgres FTS vs. external search:**
- Postgres FTS: Good enough for most apps up to ~10M rows. Zero infrastructure overhead.
- External (Elasticsearch, Typesense, Meilisearch): When you need fuzzy matching, typo tolerance, faceted search, or sub-50ms latency at scale.

---

## JSON / JSONB Columns

```sql
-- Use JSONB, not JSON (JSONB is binary, indexable, deduplicates keys)
ALTER TABLE products ADD COLUMN metadata JSONB DEFAULT '{}';

-- Query JSONB
SELECT * FROM products WHERE metadata->>'color' = 'red';
SELECT * FROM products WHERE metadata @> '{"tags": ["sale"]}';

-- Index JSONB
CREATE INDEX idx_products_metadata ON products USING gin(metadata);
-- Or index a specific path:
CREATE INDEX idx_products_color ON products ((metadata->>'color'));

-- Update nested values
UPDATE products
SET metadata = jsonb_set(metadata, '{dimensions,weight}', '"2kg"')
WHERE id = $1;
```

**Rules:**
- Use JSONB for flexible/semi-structured data (metadata, preferences, feature flags).
- Do NOT use JSONB to avoid schema design. If you query a field frequently, it should be a column.
- JSONB lacks type enforcement — validate in the application layer or use CHECK constraints.

```sql
-- CHECK constraint on JSONB
ALTER TABLE products ADD CONSTRAINT valid_metadata
CHECK (metadata ? 'type' AND metadata->>'type' IN ('physical', 'digital'));
```

---

## Audit Trail Design

### Approach 1: Audit Table with Triggers

```sql
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    table_name TEXT NOT NULL,
    row_id UUID NOT NULL,
    action TEXT NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    old_data JSONB,
    new_data JSONB,
    changed_by UUID,       -- user ID, nullable for system changes
    changed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_table_row ON audit_log(table_name, row_id);
CREATE INDEX idx_audit_changed_at ON audit_log(changed_at);

-- Generic trigger function
CREATE OR REPLACE FUNCTION audit_trigger_fn()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, row_id, action, new_data, changed_by)
        VALUES (TG_TABLE_NAME, NEW.id, 'INSERT', to_jsonb(NEW),
                current_setting('app.current_user_id', true)::UUID);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, row_id, action, old_data, new_data, changed_by)
        VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW),
                current_setting('app.current_user_id', true)::UUID);
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, row_id, action, old_data, changed_by)
        VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', to_jsonb(OLD),
                current_setting('app.current_user_id', true)::UUID);
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Apply to a table
CREATE TRIGGER audit_users
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_fn();
```

### Approach 2: Soft Deletes + Updated-By Columns

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;
ALTER TABLE users ADD COLUMN updated_by UUID;

-- Query active records
CREATE VIEW active_users AS
SELECT * FROM users WHERE deleted_at IS NULL;

-- "Delete" = set timestamp
UPDATE users SET deleted_at = now(), updated_by = $1 WHERE id = $2;
```

### Passing User Context to Triggers

```python
# Set the session variable before queries
await conn.execute("SET LOCAL app.current_user_id = $1", str(user_id))
# Then the trigger reads it via current_setting('app.current_user_id', true)
```

**Rules:**
- Use the trigger approach for compliance-grade audit trails.
- Use soft deletes for simple "undo" functionality.
- Partition or archive the audit_log table if it grows large (by `changed_at`).
- Never let the audit mechanism block or slow down the primary write path in a way that degrades UX.

---

## List Endpoints with Per-Row Rollups: Batch, Never N+1

A list endpoint that shows summary counts from related tables (e.g. "recent
parent records, each with counts from three child tables") must not issue one
query per row. Two acceptable shapes:

1. **SQL-side aggregation:** a view or JOIN with GROUP BY that returns the
   rollups directly. Best when the aggregation is reused or windowed.
2. **Batched IN-queries:** fetch the page of parent rows first, collect the
   page's ids, then issue ONE query per related table using `WHERE parent_id
   IN (page_ids)`, and aggregate in application code over the bounded page.
   Query count is fixed (1 + number of related tables) regardless of page
   size. Best when no migration is wanted and the page is small (<=100 rows).

Either way, ensure every related table is indexed on the foreign key being
filtered. The anti-pattern to watch for: a loop over list results that calls
a per-id repository method (it often hides inside "get detail for each row").

---

## Quick Reference Checklist

- [ ] Start normalized; denormalize with evidence
- [ ] Index columns in WHERE, JOIN, ORDER BY
- [ ] Use `CREATE INDEX CONCURRENTLY` on live tables
- [ ] Keyset (cursor) pagination, never deep OFFSET
- [ ] Connection pool sized at 10-20, with timeouts
- [ ] Transactions are short, no external calls inside
- [ ] JSONB for semi-structured data, not as a schema escape hatch
- [ ] Generated tsvector column + GIN index for full-text search
- [ ] Audit trail via triggers or soft deletes depending on requirements
- [ ] Migrations are append-only, one change per file, reversible
