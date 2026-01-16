# Database Optimizer - Technical Reference

## Database Configuration Tuning (PostgreSQL)

**Scenario**: Production database with high CPU and slow queries

### Step 1: Analyze current configuration and workload

```bash
# Check current PostgreSQL configuration
psql -c "SHOW ALL;" | grep -E "(shared_buffers|effective_cache_size|work_mem|maintenance_work_mem|max_connections)"

# Analyze system resources
free -h  # Available RAM
cat /proc/cpuinfo | grep processor | wc -l  # CPU count
df -h /var/lib/postgresql/  # Disk space

# Check database size and activity
psql -c "SELECT pg_size_pretty(pg_database_size('production'));"
psql -c "SELECT count(*) FROM pg_stat_activity WHERE state = 'active';"
```

### Step 2: Calculate optimal settings (for 32GB RAM, 8 CPUs, SSD)

```conf
# postgresql.conf optimization

# MEMORY SETTINGS
# Rule of thumb: shared_buffers = 25% of RAM (8GB for 32GB system)
shared_buffers = 8GB

# effective_cache_size = 75% of RAM (OS cache + shared_buffers)
effective_cache_size = 24GB

# work_mem: per-operation memory (total_ram / max_connections / 4)
# 32GB / 200 connections / 4 = 40MB
work_mem = 40MB

# maintenance_work_mem: for VACUUM, CREATE INDEX (10% of RAM)
maintenance_work_mem = 2GB

# CONNECTION SETTINGS
max_connections = 200
superuser_reserved_connections = 3

# CHECKPOINT SETTINGS (reduce I/O spikes)
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9
max_wal_size = 4GB
min_wal_size = 1GB

# QUERY PLANNER SETTINGS
random_page_cost = 1.1  # SSD (lower than default 4.0 for HDD)
effective_io_concurrency = 200  # SSD can handle parallel I/O
default_statistics_target = 100  # More accurate query plans

# PARALLEL QUERY SETTINGS
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_worker_processes = 8

# LOGGING FOR PERFORMANCE ANALYSIS
log_min_duration_statement = 200  # Log queries >200ms
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on

# AUTOVACUUM TUNING (prevent bloat)
autovacuum_max_workers = 3
autovacuum_naptime = 20s  # More frequent for high-write workload
autovacuum_vacuum_scale_factor = 0.05  # Vacuum at 5% dead tuples (vs default 20%)
autovacuum_analyze_scale_factor = 0.025
```

### Step 3: Apply configuration incrementally

```bash
# Backup current configuration
cp /etc/postgresql/14/main/postgresql.conf /etc/postgresql/14/main/postgresql.conf.backup

# Apply new configuration
vim /etc/postgresql/14/main/postgresql.conf

# Validate configuration syntax
/usr/lib/postgresql/14/bin/postgres -C config_file

# Apply changes (requires restart for shared_buffers)
sudo systemctl restart postgresql

# Verify applied settings
psql -c "SHOW shared_buffers; SHOW work_mem; SHOW effective_cache_size;"
```

### Step 4: Monitor impact

```sql
-- Monitor query performance improvement
SELECT 
  queryid,
  calls,
  mean_exec_time,
  total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Check cache hit ratio (target: >90%)
SELECT 
  sum(heap_blks_read) as heap_read,
  sum(heap_blks_hit) as heap_hit,
  sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as cache_hit_ratio
FROM pg_statio_user_tables;

-- Monitor connection usage
SELECT count(*), state FROM pg_stat_activity GROUP BY state;

-- Check autovacuum effectiveness
SELECT 
  schemaname,
  relname,
  n_dead_tup,
  n_live_tup,
  last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

**Expected outcome**:
- Query performance improved by 40-60%
- Cache hit ratio increased to >95%
- Checkpoint I/O spikes reduced
- Autovacuum preventing table bloat
- Connection pool stable and efficient

---

## Partitioning by Time-Series Data

**Use case**: Optimize queries on append-only time-series tables (logs, events)

```sql
-- Template: Range partitioning by month
CREATE TABLE {table}_partitioned (
  id BIGSERIAL,
  created_at TIMESTAMP NOT NULL,
  -- other columns
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Create partitions (automate with script)
CREATE TABLE {table}_y2024m01 PARTITION OF {table}_partitioned
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE {table}_y2024m02 PARTITION OF {table}_partitioned
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Indexes on each partition (created automatically)
CREATE INDEX ON {table}_partitioned (created_at);
CREATE INDEX ON {table}_partitioned (user_id);

-- Example: Events table partitioned by month
CREATE TABLE events_partitioned (
  id BIGSERIAL,
  user_id INTEGER NOT NULL,
  event_type VARCHAR(50) NOT NULL,
  payload JSONB,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Monthly partitions
CREATE TABLE events_y2024m01 PARTITION OF events_partitioned
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Maintenance: Archive old partitions
DETACH PARTITION events_y2023m01;  -- Detach old data
DROP TABLE events_y2023m01;  -- Or move to archive storage
```

**Customization points**:
- Partition interval: daily (high volume), monthly (medium), yearly (low)
- Retention policy: Automate partition creation and archival
- Indexes: Create relevant indexes on partition key and query columns

---

## Advanced Monitoring Queries

### Find Missing Indexes

```sql
-- Tables with sequential scans (potential missing indexes)
SELECT 
  schemaname,
  relname,
  seq_scan,
  seq_tup_read,
  idx_scan,
  idx_tup_fetch,
  ROUND(seq_tup_read::numeric / NULLIF(seq_scan, 0), 2) as avg_seq_tup
FROM pg_stat_user_tables
WHERE seq_scan > 100
ORDER BY seq_tup_read DESC
LIMIT 20;
```

### Find Unused Indexes

```sql
-- Indexes with zero scans (candidates for removal)
SELECT 
  schemaname,
  tablename,
  indexname,
  idx_scan,
  pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%pkey%'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Query Performance Statistics

```sql
-- Top queries by total time
SELECT 
  queryid,
  calls,
  ROUND(total_exec_time::numeric, 2) as total_ms,
  ROUND(mean_exec_time::numeric, 2) as mean_ms,
  ROUND((100 * total_exec_time / sum(total_exec_time) OVER())::numeric, 2) as pct,
  LEFT(query, 100) as query_preview
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

### Lock Monitoring

```sql
-- Current blocking locks
SELECT 
  blocked_locks.pid AS blocked_pid,
  blocked_activity.usename AS blocked_user,
  blocking_locks.pid AS blocking_pid,
  blocking_activity.usename AS blocking_user,
  blocked_activity.query AS blocked_query,
  blocking_activity.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
  ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
  AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
  AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
  AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
  AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
  AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
  AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
  AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
  AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
  AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

---

## MySQL Configuration Reference

### InnoDB Optimization (for 32GB RAM)

```ini
[mysqld]
# Buffer pool = 70% of RAM for dedicated DB server
innodb_buffer_pool_size = 22G
innodb_buffer_pool_instances = 8

# Log file size (larger = better write performance, slower recovery)
innodb_log_file_size = 2G
innodb_log_buffer_size = 64M

# Flush settings
innodb_flush_log_at_trx_commit = 1  # ACID compliance
innodb_flush_method = O_DIRECT

# Thread concurrency
innodb_thread_concurrency = 0  # Auto-detect
innodb_read_io_threads = 8
innodb_write_io_threads = 8

# Query cache (disable in MySQL 8+, deprecated)
query_cache_type = 0
query_cache_size = 0

# Slow query log
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 0.2
```

---

## MongoDB Optimization Reference

### Index Strategies

```javascript
// Compound index for common query pattern
db.orders.createIndex(
  { userId: 1, createdAt: -1 },
  { background: true }
);

// Partial index for active records only
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { status: "active" } }
);

// TTL index for automatic expiration
db.sessions.createIndex(
  { lastAccess: 1 },
  { expireAfterSeconds: 3600 }
);

// Text index for search
db.products.createIndex(
  { name: "text", description: "text" },
  { weights: { name: 10, description: 5 } }
);
```

### Aggregation Pipeline Optimization

```javascript
// Use $match early to reduce documents
db.orders.aggregate([
  { $match: { status: "completed", createdAt: { $gte: ISODate("2024-01-01") } } },
  { $group: { _id: "$userId", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } },
  { $limit: 100 }
]);

// Explain aggregation
db.orders.aggregate([...]).explain("executionStats");
```
