# Database Administrator - Code Examples & Patterns

## Anti-Pattern: Ignoring Query Plan Regression

### What it looks like (BAD):

```sql
-- Query works fine in development (1000 rows)
SELECT * FROM orders 
WHERE user_id = 123 
ORDER BY created_at DESC 
LIMIT 10;

-- Execution time: 5ms (great!)

-- Same query in production (10M rows)
-- Execution time: 45 seconds (disaster!)
```

### Why it fails:
- Development data is tiny compared to production
- Query planner chooses different strategies at scale
- Indexes not created in production
- Table statistics stale (autovacuum not running)

### Correct approach:

```sql
-- 1. Always EXPLAIN ANALYZE in production-like data
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM orders 
WHERE user_id = 123 
ORDER BY created_at DESC 
LIMIT 10;

-- Output analysis:
-- Seq Scan on orders  (cost=0.00..250000.00 rows=10000000 width=100)
--   Filter: (user_id = 123)
-- Execution Time: 45234.567 ms  -- PROBLEM!

-- 2. Create appropriate index
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);

-- 3. Re-analyze table statistics
ANALYZE orders;

-- 4. Verify improvement
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders 
WHERE user_id = 123 
ORDER BY created_at DESC 
LIMIT 10;

-- Output after index:
-- Index Scan using idx_orders_user_created on orders
--   (cost=0.43..12.56 rows=10 width=100)
--   Index Cond: (user_id = 123)
-- Execution Time: 0.123 ms  -- 45s → 0.1ms (450,000x faster!)

-- 5. Set up query plan regression monitoring
CREATE EXTENSION pg_stat_statements;

-- Monitor slow queries daily
SELECT 
    query,
    calls,
    mean_exec_time,
    max_exec_time,
    total_exec_time
FROM pg_stat_statements
WHERE mean_exec_time > 1000  -- Queries averaging >1s
ORDER BY total_exec_time DESC
LIMIT 20;
```

## MongoDB Sharding Deployment

```bash
# 1. Deploy Config Server Replica Set
mongod --configsvr --replSet configReplSet --dbpath /data/configdb --port 27019

# Initialize config replica set
mongo --port 27019
rs.initiate({
  _id: "configReplSet",
  configsvr: true,
  members: [
    { _id: 0, host: "10.0.1.10:27019" },
    { _id: 1, host: "10.0.1.11:27019" },
    { _id: 2, host: "10.0.1.12:27019" }
  ]
})

# 2. Deploy Each Shard Replica Set
mongod --shardsvr --replSet shard1-rs --dbpath /data/shard1 --port 27018

# Initialize shard replica set
mongo --port 27018
rs.initiate({
  _id: "shard1-rs",
  members: [
    { _id: 0, host: "10.0.3.10:27018" },
    { _id: 1, host: "10.0.3.11:27018" },
    { _id: 2, host: "10.0.3.12:27018" }
  ]
})

# 3. Deploy mongos Query Routers
mongos --configdb configReplSet/10.0.1.10:27019,10.0.1.11:27019,10.0.1.12:27019 --port 27017

# 4. Add Shards to Cluster
mongo --port 27017
sh.addShard("shard1-rs/10.0.3.10:27018,10.0.3.11:27018,10.0.3.12:27018")
sh.addShard("shard2-rs/10.0.4.10:27018,10.0.4.11:27018,10.0.4.12:27018")
sh.addShard("shard3-rs/10.0.5.10:27018,10.0.5.11:27018,10.0.5.12:27018")

# 5. Enable Sharding
sh.enableSharding("ecommerce")
sh.shardCollection("ecommerce.products", { category_id: 1, product_id: 1 })

# 6. Verify
sh.status()
```

## MongoDB Application Connection

```javascript
const { MongoClient } = require('mongodb');

const uri = "mongodb://mongos-1:27017,mongos-2:27017/ecommerce?replicaSet=configReplSet";
const client = new MongoClient(uri, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
  readPreference: 'secondaryPreferred',  // Distribute reads
  w: 'majority',  // Write concern
  retryWrites: true
});

// Query automatically routed to correct shard
async function getProductsByCategory(categoryId) {
  const products = await client.db('ecommerce')
    .collection('products')
    .find({ category_id: categoryId })  // Uses shard key
    .limit(100)
    .toArray();
  
  return products;
}
```

## PostgreSQL Performance Monitoring Queries

```sql
-- Find slow queries
SELECT 
    query,
    calls,
    total_exec_time / 1000 as total_seconds,
    mean_exec_time / 1000 as avg_seconds,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Find missing indexes (sequential scans on large tables)
SELECT 
    schemaname,
    relname,
    seq_scan,
    seq_tup_read,
    idx_scan,
    n_live_tup
FROM pg_stat_user_tables
WHERE seq_scan > 0 
  AND n_live_tup > 10000
ORDER BY seq_tup_read DESC
LIMIT 10;

-- Check index usage
SELECT 
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Find bloated tables (need VACUUM)
SELECT 
    relname,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup * 100.0 / nullif(n_live_tup, 0), 2) as dead_pct,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- Check replication lag
SELECT 
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) as lag_bytes
FROM pg_stat_replication;

-- Check locks
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity 
    ON blocked_activity.pid = blocked_locks.pid
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
JOIN pg_catalog.pg_stat_activity blocking_activity 
    ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

## Backup Verification Script

```bash
#!/bin/bash
# Verify backup can be restored

BACKUP_PATH=$1
TEST_PORT=5433
TEST_DIR="/tmp/pg_restore_test_$$"

echo "Testing restore of $BACKUP_PATH..."

# Extract backup
mkdir -p $TEST_DIR
tar -xzf $BACKUP_PATH -C $TEST_DIR

# Start PostgreSQL on test port
pg_ctl -D $TEST_DIR -o "-p $TEST_PORT" -l $TEST_DIR/logfile start

# Wait for startup
sleep 5

# Run test queries
psql -p $TEST_PORT -c "SELECT COUNT(*) FROM users;" postgres
psql -p $TEST_PORT -c "SELECT COUNT(*) FROM orders;" postgres

# Cleanup
pg_ctl -D $TEST_DIR stop
rm -rf $TEST_DIR

echo "Backup verification complete!"
```
