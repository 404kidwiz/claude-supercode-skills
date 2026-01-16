# SQL Pro - Code Examples & Patterns

## Recursive CTE for Hierarchical Data

**Scenario**: Organizational hierarchy - find all employees under a manager

```sql
-- Create employees table
CREATE TABLE employees (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  manager_id INTEGER REFERENCES employees(id),
  title VARCHAR(100),
  salary DECIMAL(10,2)
);

-- Find all employees under VP Bob (id=2), including indirect reports
WITH RECURSIVE org_tree AS (
  -- Anchor: Starting point (VP Bob himself)
  SELECT 
    id,
    name,
    manager_id,
    title,
    salary,
    0 as depth,  -- Track hierarchy depth
    ARRAY[id] as path,  -- Track path from root (prevents cycles)
    name::TEXT as hierarchy  -- Visual representation
  FROM employees
  WHERE id = 2  -- Starting point: VP Bob
  
  UNION ALL
  
  -- Recursive: Find direct reports of current level
  SELECT 
    e.id,
    e.name,
    e.manager_id,
    e.title,
    e.salary,
    ot.depth + 1,
    ot.path || e.id,  -- Append to path
    ot.hierarchy || ' > ' || e.name  -- Build visual hierarchy
  FROM employees e
  INNER JOIN org_tree ot ON e.manager_id = ot.id
  WHERE NOT (e.id = ANY(ot.path))  -- Prevent infinite loops (cycle detection)
)

SELECT 
  id,
  REPEAT('  ', depth) || name as indented_name,  -- Visual indentation
  title,
  salary,
  depth,
  hierarchy
FROM org_tree
ORDER BY path;  -- Depth-first traversal
```

## Pagination with Total Count

```sql
-- Template: Window function for pagination + total count in single query
WITH paginated_users AS (
  SELECT 
    id,
    name,
    email,
    created_at,
    COUNT(*) OVER () as total_count  -- Total count without separate query
  FROM users
  WHERE status = 'active'
  ORDER BY created_at DESC
  LIMIT 20 OFFSET 40  -- Page 3 (20 per page)
)
SELECT * FROM paginated_users;

/*
Output includes total_count in every row (no separate COUNT query needed):
id  | name  | email       | created_at | total_count
----|-------|-------------|------------|------------
123 | Alice | a@email.com | 2024-03-15 | 5432
124 | Bob   | b@email.com | 2024-03-14 | 5432
*/
```

## Conditional Aggregation (Pivot Table)

```sql
-- Pivot without PIVOT syntax (works across all databases)
SELECT 
  EXTRACT(YEAR FROM sale_date) as year,
  SUM(CASE WHEN EXTRACT(QUARTER FROM sale_date) = 1 THEN amount ELSE 0 END) as q1_sales,
  SUM(CASE WHEN EXTRACT(QUARTER FROM sale_date) = 2 THEN amount ELSE 0 END) as q2_sales,
  SUM(CASE WHEN EXTRACT(QUARTER FROM sale_date) = 3 THEN amount ELSE 0 END) as q3_sales,
  SUM(CASE WHEN EXTRACT(QUARTER FROM sale_date) = 4 THEN amount ELSE 0 END) as q4_sales,
  SUM(amount) as total_sales
FROM sales
WHERE sale_date >= '2024-01-01'
GROUP BY EXTRACT(YEAR FROM sale_date);

/*
Output:
year | q1_sales | q2_sales | q3_sales | q4_sales | total_sales
-----|----------|----------|----------|----------|------------
2024 | 125000   | 145000   | 167000   | 189000   | 626000
*/
```

## Index Coverage Check

```sql
-- Template: Check if query can use index-only scan
-- 1. Identify columns in WHERE, JOIN, ORDER BY
-- 2. Identify columns in SELECT
-- 3. Create covering index

-- Example query:
SELECT user_id, created_at, total 
FROM orders 
WHERE status = 'completed' AND created_at >= '2024-01-01'
ORDER BY created_at DESC;

-- Covering index design:
CREATE INDEX idx_orders_status_created_covering
  ON orders (status, created_at DESC)  -- Filter + sort columns
  INCLUDE (user_id, total);  -- SELECT columns (PostgreSQL)

-- MySQL equivalent:
CREATE INDEX idx_orders_status_created_covering
  ON orders (status, created_at DESC, user_id, total);

-- Verify with EXPLAIN: look for "Index Only Scan" (PostgreSQL) or "Using index" (MySQL)
```

## Anti-Patterns & Fixes

### Anti-Pattern: SELECT * in Production Code

**BAD:**
```sql
-- Selecting all columns (even if only need 3)
SELECT * FROM users WHERE id = 123;

-- Joining with SELECT *
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending';
```

**Why it fails:**
- **Performance**: Fetches unnecessary data (network transfer, memory overhead)
- **Index coverage**: Cannot use index-only scans (must access heap)
- **Breaking changes**: Schema changes break application code
- **Ambiguity**: Column name conflicts in joins (which 'id' column?)

**GOOD:**
```sql
-- Select only needed columns
SELECT id, email, name, created_at 
FROM users 
WHERE id = 123;

-- Explicit columns in joins (qualify with table alias)
SELECT 
  o.id as order_id,
  o.total,
  o.status,
  u.email,
  u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending';

-- Bonus: Can use covering index
CREATE INDEX idx_users_id_covering 
  ON users (id) 
  INCLUDE (email, name, created_at);  -- Index-only scan possible
```

### Anti-Pattern: Implicit Type Conversion in WHERE Clause

**BAD:**
```sql
-- user_id is INTEGER, but comparing with string
SELECT * FROM orders WHERE user_id = '12345';

-- created_at is TIMESTAMP, but comparing with string
SELECT * FROM orders WHERE created_at = '2024-03-15';

-- phone_number is VARCHAR, but comparing with number
SELECT * FROM users WHERE phone_number = 1234567890;
```

**Why it fails:**
- **Index not used**: Type mismatch prevents index usage (full table scan)
- **Implicit conversion overhead**: Database converts every row's value (slow)
- **Inconsistent behavior**: Different databases handle conversions differently

**GOOD:**
```sql
-- Use correct data types
SELECT * FROM orders WHERE user_id = 12345;  -- INTEGER

-- Explicit type casting when necessary
SELECT * FROM orders WHERE created_at = '2024-03-15'::DATE;

-- Or use proper timestamp
SELECT * FROM orders WHERE created_at >= '2024-03-15 00:00:00'::TIMESTAMP
  AND created_at < '2024-03-16 00:00:00'::TIMESTAMP;

-- String comparison for VARCHAR
SELECT * FROM users WHERE phone_number = '1234567890';

-- Verify index usage with EXPLAIN
EXPLAIN SELECT * FROM orders WHERE user_id = 12345;
-- Should show "Index Scan" or "Index Seek", NOT "Seq Scan"
```

## Query Optimization Example

```sql
-- Before: Multiple scans, inefficient joins
SELECT o.order_id, c.customer_name, SUM(oi.quantity * oi.price) as total
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_date >= '2024-01-01'
GROUP BY o.order_id, c.customer_name;

-- After: Optimized with covering index and CTE
WITH recent_orders AS (
  SELECT order_id, customer_id, order_date
  FROM orders
  WHERE order_date >= '2024-01-01'
)
SELECT ro.order_id, c.customer_name, SUM(oi.quantity * oi.price) as total
FROM recent_orders ro
JOIN customers c ON ro.customer_id = c.customer_id
JOIN order_items oi ON ro.order_id = oi.order_id
GROUP BY ro.order_id, c.customer_name;

-- Covering index
CREATE INDEX idx_orders_date_customer ON orders(order_date, customer_id) INCLUDE (order_id);
```
