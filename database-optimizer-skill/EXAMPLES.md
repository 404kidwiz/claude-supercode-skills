# Database Optimizer - Examples & Patterns

## Anti-Patterns

### Anti-Pattern: Over-Indexing

**What it looks like:**

```sql
-- Creating too many indexes "just in case"
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_name ON users (name);
CREATE INDEX idx_users_status ON users (status);
CREATE INDEX idx_users_created ON users (created_at);
CREATE INDEX idx_users_updated ON users (updated_at);
CREATE INDEX idx_users_email_name ON users (email, name);
CREATE INDEX idx_users_email_status ON users (email, status);
CREATE INDEX idx_users_name_status ON users (name, status);
-- ... 15 more indexes on same table
```

**Why it fails:**
- **Write penalty**: Every INSERT/UPDATE/DELETE must update ALL indexes (10x slower writes)
- **Storage bloat**: Indexes consume disk space (can exceed table size)
- **Planner confusion**: Too many options can cause suboptimal index selection
- **Maintenance overhead**: VACUUM, ANALYZE, REINDEX take longer

**Correct approach:**

```sql
-- Analyze actual query patterns first
SELECT 
  queryid,
  calls,
  mean_exec_time,
  query
FROM pg_stat_statements
WHERE query LIKE '%users%'
ORDER BY calls DESC
LIMIT 10;

-- Create only necessary indexes based on query frequency
CREATE INDEX idx_users_email ON users (email);  -- Login queries (high frequency)
CREATE INDEX idx_users_status_created ON users (status, created_at) 
  WHERE status = 'active';  -- Partial index for active user queries

-- Monitor index usage and remove unused ones
SELECT 
  schemaname,
  tablename,
  indexname,
  idx_scan
FROM pg_stat_user_indexes
WHERE schemaname = 'public' AND idx_scan < 100  -- Low usage
ORDER BY idx_scan;

-- Remove unused index
DROP INDEX IF EXISTS idx_users_updated;  -- Never used
```

**Rule of thumb**: Max 5-7 indexes per table for OLTP workloads.

---

### Anti-Pattern: Premature Denormalization

**What it looks like:**

```sql
-- "Joins are slow, let's denormalize everything!"
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INTEGER,
  -- Duplicated user data (violates normalization)
  user_email VARCHAR(255),
  user_name VARCHAR(255),
  user_address TEXT,
  product_id INTEGER,
  -- Duplicated product data
  product_name VARCHAR(255),
  product_price DECIMAL(10,2),
  product_category VARCHAR(100)
);
```

**Why it fails:**
- **Data inconsistency**: User email changes, but denormalized copies remain stale
- **Update anomalies**: Must update multiple rows when product price changes
- **Storage waste**: Redundant data increases table size and I/O
- **Premature optimization**: Joins are fast with proper indexes

**Correct approach:**

```sql
-- Start with normalized design (3NF)
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(255),
  address TEXT
);

CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  price DECIMAL(10,2) NOT NULL,
  category_id INTEGER REFERENCES categories(id)
);

CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  product_id INTEGER REFERENCES products(id),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Optimize with strategic indexes (joins are fast!)
CREATE INDEX idx_orders_user_id ON orders (user_id);
CREATE INDEX idx_orders_product_id ON orders (product_id);

-- Query with JOIN (fast with proper indexes)
EXPLAIN ANALYZE
SELECT 
  o.id,
  u.email,
  u.name,
  p.name as product_name,
  p.price
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN products p ON o.product_id = p.id
WHERE o.created_at >= NOW() - INTERVAL '7 days';

/*
With proper indexes, execution time: 12ms for 10K rows
*/

-- ONLY denormalize IF:
-- 1. Proven performance bottleneck (not assumption)
-- 2. Read-heavy workload (10:1 read:write ratio)
-- 3. Data rarely changes (product price vs stock level)

-- Selective denormalization example (materialized view)
CREATE MATERIALIZED VIEW mv_order_details AS
SELECT 
  o.id as order_id,
  u.email,
  p.name as product_name,
  p.price,
  o.created_at
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN products p ON o.product_id = p.id;

-- Refresh periodically (acceptable staleness)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_order_details;
```

**Rule**: Normalize first, denormalize only when measurements prove it necessary.

---

### Anti-Pattern: Missing Connection Pooling

**What it looks like:**

```python
# Opening new connection for every request
def get_user(user_id):
    conn = psycopg2.connect(DATABASE_URL)  # New connection every time!
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    result = cursor.fetchone()
    conn.close()
    return result
```

**Why it fails:**
- Connection overhead: 50-100ms per new connection
- Resource exhaustion: max_connections limit reached
- Database overload: Too many connection processes

**Correct approach:**

```python
# Use connection pool
from psycopg2 import pool

connection_pool = pool.ThreadedConnectionPool(
    minconn=5,
    maxconn=20,
    dsn=DATABASE_URL
)

def get_user(user_id):
    conn = connection_pool.getconn()
    try:
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
        return cursor.fetchone()
    finally:
        connection_pool.putconn(conn)
```

---

## Quality Checklist

Before marking database optimization complete, verify:

### Performance Metrics
- [ ] P50 query latency <50ms for OLTP workloads
- [ ] P95 query latency <100ms for OLTP workloads
- [ ] P99 query latency <200ms for OLTP workloads
- [ ] Analytical queries <5s for standard reports
- [ ] Cache hit ratio >90% (shared_buffers + OS cache)
- [ ] Index usage >95% for all created indexes

### Index Strategy
- [ ] All foreign keys have indexes
- [ ] All WHERE/JOIN columns covered by indexes
- [ ] No unused indexes (idx_scan = 0 in pg_stat_user_indexes)
- [ ] Covering indexes eliminate heap fetches for hot queries
- [ ] Partial indexes used for selective queries
- [ ] Max 5-7 indexes per table (OLTP workloads)

### Configuration
- [ ] shared_buffers = 25% of RAM
- [ ] effective_cache_size = 75% of RAM
- [ ] work_mem appropriate for workload (no temp file spills)
- [ ] max_connections matches application connection pool
- [ ] Autovacuum configured to prevent bloat
- [ ] Checkpoint tuning reduces I/O spikes

### Verification
- [ ] EXPLAIN ANALYZE executed for all optimized queries
- [ ] Before/after metrics documented with %improvement
- [ ] Slow query log reviewed (no queries >200ms)
- [ ] Load testing performed (2x expected traffic)
- [ ] Rollback plan documented for configuration changes

### Documentation
- [ ] Index strategy documented with rationale
- [ ] Configuration changes documented with reasoning
- [ ] Monitoring queries provided for ongoing tracking
- [ ] Maintenance procedures documented (VACUUM, ANALYZE, REINDEX)

### Monitoring
- [ ] Query performance dashboard created (Grafana/Datadog)
- [ ] Alerts configured for P95 latency >100ms
- [ ] Alerts configured for cache hit ratio <90%
- [ ] Alerts configured for replication lag >1s
- [ ] Alerts configured for connection pool exhaustion

---

## Common Index Patterns

### Covering Index (Index-Only Scan)

```sql
-- Query pattern
SELECT id, email FROM users WHERE status = 'active';

-- Covering index (no heap access needed)
CREATE INDEX idx_users_status_covering 
  ON users (status) 
  INCLUDE (id, email);
```

### Partial Index (Selective Queries)

```sql
-- Query pattern (only queries active users)
SELECT * FROM users WHERE status = 'active' AND created_at > '2024-01-01';

-- Partial index (smaller, faster)
CREATE INDEX idx_users_active_created 
  ON users (created_at) 
  WHERE status = 'active';
```

### Expression Index

```sql
-- Query pattern (case-insensitive search)
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';

-- Expression index
CREATE INDEX idx_users_email_lower ON users (LOWER(email));
```

### BRIN Index (Time-Series Data)

```sql
-- Large table with natural ordering (timestamps)
CREATE INDEX idx_events_created_brin 
  ON events 
  USING BRIN (created_at);

-- Much smaller than B-tree, effective for range queries
```
