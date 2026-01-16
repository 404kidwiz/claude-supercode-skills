# PostgreSQL Professional - Code Examples & Patterns

## JSONB Indexing and Querying

### Design JSONB Schema

```sql
-- Create table with JSONB column
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(255),
  metadata JSONB NOT NULL DEFAULT '{}',
  created_at TIMESTAMP DEFAULT NOW()
);

-- Sample JSONB data structure
INSERT INTO users (email, name, metadata) VALUES
  ('alice@example.com', 'Alice', '{
    "preferences": {
      "theme": "dark",
      "language": "en",
      "notifications": {
        "email": true,
        "push": false
      }
    },
    "profile": {
      "age": 28,
      "city": "San Francisco",
      "interests": ["tech", "hiking", "photography"]
    },
    "subscription": {
      "plan": "premium",
      "expires_at": "2025-12-31"
    }
  }');
```

### Create GIN Indexes

```sql
-- Default GIN index (supports all JSONB operators)
CREATE INDEX idx_users_metadata_gin ON users USING GIN (metadata);

-- GIN index with jsonb_path_ops (faster, containment only)
CREATE INDEX idx_users_metadata_path_ops ON users USING GIN (metadata jsonb_path_ops);

-- Partial GIN index (for specific keys)
CREATE INDEX idx_users_subscription ON users USING GIN ((metadata->'subscription'))
  WHERE metadata ? 'subscription';

-- Expression index for frequently queried path
CREATE INDEX idx_users_theme ON users ((metadata->'preferences'->>'theme'))
  WHERE metadata->'preferences' ? 'theme';
```

### Query Patterns with Optimal Index Usage

```sql
-- Containment query (uses GIN index)
SELECT * FROM users
WHERE metadata @> '{"preferences": {"theme": "dark"}}';

-- Key existence check
SELECT * FROM users
WHERE metadata ? 'subscription';

-- Path extraction (uses expression index if exists)
SELECT email, metadata->'preferences'->>'theme' as theme
FROM users
WHERE metadata->'preferences'->>'theme' = 'dark';

-- Array containment
SELECT * FROM users
WHERE metadata->'profile'->'interests' @> '["tech"]';

-- Complex nested query
SELECT email, metadata
FROM users
WHERE metadata->'subscription'->>'plan' = 'premium'
  AND (metadata->'subscription'->>'expires_at')::DATE > CURRENT_DATE;

-- Aggregation on JSONB field
SELECT 
  metadata->'preferences'->>'theme' as theme,
  COUNT(*) as user_count
FROM users
WHERE metadata->'preferences' ? 'theme'
GROUP BY metadata->'preferences'->>'theme'
ORDER BY user_count DESC;
```

### Verify Index Usage

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users
WHERE metadata @> '{"preferences": {"theme": "dark"}}';

/*
Expected output with GIN index:
Bitmap Heap Scan on users  (cost=... actual time=0.234..1.456 rows=234 loops=1)
  Recheck Cond: (metadata @> '{"preferences": {"theme": "dark"}}'::jsonb)
  Heap Blocks: exact=123
  Buffers: shared hit=126
  ->  Bitmap Index Scan on idx_users_metadata_gin  (cost=... actual time=0.123 rows=234 loops=1)
        Index Cond: (metadata @> '{"preferences": {"theme": "dark"}}'::jsonb)
        Buffers: shared hit=3
*/
```

## Materialized View with Concurrent Refresh

```sql
-- Create materialized view
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT 
  DATE(created_at) as sale_date,
  product_id,
  COUNT(*) as order_count,
  SUM(total) as total_revenue,
  AVG(total) as avg_order_value
FROM orders
WHERE status = 'completed'
GROUP BY DATE(created_at), product_id
WITH DATA;

-- Unique index required for REFRESH CONCURRENTLY
CREATE UNIQUE INDEX mv_daily_sales_pkey ON mv_daily_sales (sale_date, product_id);

-- Refresh without blocking reads
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales;

-- Schedule refresh with pg_cron
CREATE EXTENSION IF NOT EXISTS pg_cron;

SELECT cron.schedule(
  'refresh-daily-sales',
  '0 2 * * *',  -- Every day at 2 AM
  'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales'
);
```

## Anti-Patterns & Fixes

### Anti-Pattern: Not Using Connection Pooling

**BAD:**
```python
# Opening new connection for every request (NO connection pool)
def handle_request():
    conn = psycopg2.connect("dbname=mydb user=postgres")
    cur = conn.cursor()
    cur.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    result = cur.fetchall()
    conn.close()  # Close connection after each request
    return result
```

**Why it fails:**
- **Connection overhead**: Creating connection takes 20-50ms (vs <1ms from pool)
- **Resource exhaustion**: PostgreSQL max_connections limit (200-400 typically)
- **Performance degradation**: Connection setup overhead dominates query time

**GOOD:**
```python
# Use connection pool
from psycopg2 import pool

# Create connection pool (application startup)
connection_pool = pool.SimpleConnectionPool(
    minconn=10,
    maxconn=50,
    dbname="mydb",
    user="postgres",
    host="localhost"
)

def handle_request():
    conn = connection_pool.getconn()  # Get from pool (reuse existing)
    try:
        cur = conn.cursor()
        cur.execute("SELECT * FROM users WHERE id = %s", (user_id,))
        result = cur.fetchall()
        return result
    finally:
        connection_pool.putconn(conn)  # Return to pool (don't close)
```

**Performance improvement**: 20-50ms → <1ms per request (50x faster).

### Anti-Pattern: Using TEXT for Enumerated Values

**BAD:**
```sql
-- Storing status as unbounded TEXT
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  status TEXT,  -- Could be anything: 'pending', 'Pending', 'PENDING', 'pnding' (typo)
  total DECIMAL(10,2)
);
```

**Why it fails:**
- **Data integrity**: No database-level validation (typos, invalid values)
- **Storage waste**: TEXT uses more space than ENUM or SMALLINT
- **Query performance**: Cannot optimize with CHECK constraint or ENUM

**GOOD:**
```sql
-- Option 1: ENUM type (PostgreSQL-specific, best for few values)
CREATE TYPE order_status AS ENUM ('pending', 'processing', 'completed', 'canceled');

CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  status order_status NOT NULL DEFAULT 'pending',  -- Type-safe!
  total DECIMAL(10,2)
);

-- Typos are caught at database level
INSERT INTO orders (status, total) VALUES ('complted', 100.00);
-- ERROR: invalid input value for enum order_status: "complted"

-- Option 2: CHECK constraint (portable across databases)
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  status VARCHAR(20) NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'processing', 'completed', 'canceled')),
  total DECIMAL(10,2)
);

-- Option 3: Foreign key to lookup table (best for many values)
CREATE TABLE order_statuses (
  id SMALLINT PRIMARY KEY,
  name VARCHAR(20) UNIQUE NOT NULL
);

CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  status_id SMALLINT NOT NULL REFERENCES order_statuses(id) DEFAULT 1,
  total DECIMAL(10,2)
);
```

## Streaming Replication Setup

### Configure Primary Server

```bash
# Edit postgresql.conf on PRIMARY
sudo vim /etc/postgresql/14/main/postgresql.conf

# Replication settings
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 1GB
hot_standby = on
archive_mode = on
archive_command = 'test ! -f /mnt/wal_archive/%f && cp %p /mnt/wal_archive/%f'
```

### Create Replication User and Slot

```sql
-- On PRIMARY, create replication user
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'secure_password';

-- Create replication slot (prevents WAL deletion)
SELECT * FROM pg_create_physical_replication_slot('replica_1_slot');

-- Verify slots created
SELECT slot_name, slot_type, active FROM pg_replication_slots;
```

### Set Up Replica Server

```bash
# Stop PostgreSQL on replica
sudo systemctl stop postgresql

# Remove existing data directory
sudo rm -rf /var/lib/postgresql/14/main/*

# Base backup from primary
sudo -u postgres pg_basebackup -h primary_host -D /var/lib/postgresql/14/main -U replicator -P -v -R -X stream -C -S replica_1_slot

# Start replica
sudo systemctl start postgresql

# Verify replication status
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"  -- Should return 't'
```

### Monitor Replication Lag

```sql
-- On PRIMARY: Check connected replicas
SELECT 
  client_addr,
  state,
  sync_state,
  pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag_bytes
FROM pg_stat_replication;

-- On REPLICA: Check lag in seconds
SELECT 
  NOW() - pg_last_xact_replay_timestamp() AS replication_lag_seconds;
```
