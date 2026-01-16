# PostgreSQL Professional - Technical Reference

## What This Skill Does

The postgres-pro skill provides comprehensive PostgreSQL database administration capabilities. It handles end-to-end PostgreSQL optimization from configuration tuning through replication setup to backup automation. The skill ensures solutions achieve high query performance (< 50ms), minimal replication lag (< 500ms), excellent uptime (> 99.95%), and fast recovery (RTO < 1 hour).

## PostgreSQL Architecture

### Process Architecture
- Postmaster (main process)
- Backend processes (one per connection)
- Background workers (autovacuum, WAL writer, checkpointer)
- Auxiliary processes (logger, archiver, stats collector)

### Memory Configuration
- **shared_buffers**: 25% of RAM (main cache for table/index data)
- **effective_cache_size**: 75% of RAM (tells planner about OS cache)
- **work_mem**: Per-operation memory for sorts and hash tables
- **maintenance_work_mem**: Memory for VACUUM, CREATE INDEX

### WAL Mechanics
- Write-Ahead Logging for durability
- WAL segments (16MB default)
- Checkpoint tuning (checkpoint_completion_target)
- WAL archiving for PITR

### MVCC Implementation
- Multi-Version Concurrency Control
- Transaction IDs (XIDs)
- Tuple visibility rules
- Vacuum for dead tuple cleanup

## Performance Tuning

### Configuration Optimization

```ini
# postgresql.conf - Production settings

# Memory
shared_buffers = '8GB'              # 25% of RAM
effective_cache_size = '24GB'       # 75% of RAM
work_mem = '64MB'                   # Per-operation memory
maintenance_work_mem = '2GB'        # For VACUUM, CREATE INDEX

# WAL
wal_level = replica
max_wal_size = '4GB'
min_wal_size = '1GB'
checkpoint_completion_target = 0.9

# Query Planning
random_page_cost = 1.1              # SSD storage
effective_io_concurrency = 200      # SSD storage
default_statistics_target = 100     # Statistics accuracy

# Autovacuum
autovacuum_vacuum_scale_factor = 0.05   # More aggressive
autovacuum_analyze_scale_factor = 0.02
autovacuum_vacuum_cost_delay = 2ms
autovacuum_max_workers = 4

# Connections
max_connections = 200               # Or use PgBouncer
```

### Vacuum Tuning

Autovacuum triggers when:
- Dead tuples > autovacuum_vacuum_threshold + scale_factor * table_size
- Default: 50 + 0.2 * rows = 20% dead tuples triggers vacuum

For large tables (>1M rows):
```sql
ALTER TABLE large_table SET (
  autovacuum_vacuum_scale_factor = 0.01,  -- 1% instead of 20%
  autovacuum_analyze_scale_factor = 0.005,
  autovacuum_vacuum_threshold = 1000
);
```

## Partitioning Design

### Range Partitioning (Time-series)
```sql
CREATE TABLE events (
  id SERIAL,
  created_at TIMESTAMP NOT NULL,
  data JSONB
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE events_2024_01 PARTITION OF events
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE events_2024_02 PARTITION OF events
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

### List Partitioning (Categorical)
```sql
CREATE TABLE orders (
  id SERIAL,
  region TEXT NOT NULL,
  total DECIMAL
) PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders
  FOR VALUES IN ('US', 'CA');
CREATE TABLE orders_eu PARTITION OF orders
  FOR VALUES IN ('UK', 'DE', 'FR');
```

### Partition Maintenance
```sql
-- Detach old partition (for archiving)
ALTER TABLE events DETACH PARTITION events_2023_01;

-- Create future partitions with pg_partman
CREATE EXTENSION pg_partman;
SELECT partman.create_parent(
  p_parent_table => 'public.events',
  p_control => 'created_at',
  p_type => 'native',
  p_interval => 'monthly'
);
```

## High Availability Setup

### Streaming Replication Configuration

**Primary Server (postgresql.conf):**
```ini
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 1GB
hot_standby = on
archive_mode = on
archive_command = 'test ! -f /mnt/wal_archive/%f && cp %p /mnt/wal_archive/%f'
```

**Primary Server (pg_hba.conf):**
```
host    replication     replicator      192.168.1.0/24          scram-sha-256
```

**Replica Setup:**
```bash
# Stop PostgreSQL, clear data directory
sudo systemctl stop postgresql
sudo rm -rf /var/lib/postgresql/14/main/*

# Base backup from primary
sudo -u postgres pg_basebackup \
  -h primary_host \
  -D /var/lib/postgresql/14/main \
  -U replicator \
  -P -v -R -X stream -C -S replica_slot

# Start replica
sudo systemctl start postgresql
```

### Monitoring Replication

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

### Failover Procedure

```bash
# On REPLICA (promote to primary)
sudo -u postgres pg_ctl promote -D /var/lib/postgresql/14/main

# Or using SQL
sudo -u postgres psql -c "SELECT pg_promote();"

# Verify new primary
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"  -- Should return 'f'
```

## Connection Pooling

### PgBouncer Configuration

```ini
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction  # or session
max_client_conn = 1000
default_pool_size = 50
min_pool_size = 10
reserve_pool_size = 5
```

**Performance improvement**: 20-50ms connection time → <1ms (50x faster)

## Monitoring Queries

### Cache Hit Ratio
```sql
SELECT 
  sum(heap_blks_hit) / nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) as cache_hit_ratio
FROM pg_statio_user_tables;
-- Target: > 0.95 (95%)
```

### Table Bloat
```sql
SELECT 
  schemaname, tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size,
  n_dead_tup,
  n_live_tup,
  round(n_dead_tup::numeric / nullif(n_live_tup, 0) * 100, 2) as dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

### Slow Queries (pg_stat_statements)
```sql
SELECT 
  calls,
  round(total_exec_time::numeric / 1000, 2) as total_time_sec,
  round(mean_exec_time::numeric, 2) as mean_time_ms,
  query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

## Integration with Other Skills

- **database-optimizer**: Collaborates on general database optimization techniques
- **backend-developer**: Supports with PostgreSQL-specific query patterns and optimizations
- **data-engineer**: Works together on ETL processes and data pipeline integration
- **devops-engineer**: Guides on PostgreSQL deployment and infrastructure automation
- **sre-engineer**: Helps with reliability, monitoring, and incident response
- **cloud-architect**: Assists with cloud PostgreSQL (RDS, Cloud SQL, etc.)
- **security-auditor**: Partners on PostgreSQL security and compliance
- **performance-engineer**: Coordinates on system-level performance tuning
