# Advanced Query Optimization

<div align="center">

![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![Database](https://img.shields.io/badge/Database-Query_Optimization-blue?style=for-the-badge)

*PostgreSQL optimization techniques for high-performance applications.*

![PostgreSQL](https://upload.wikimedia.org/wikipedia/commons/2/29/Postgresql_elephant.svg)

</div>

## EXPLAIN ANALYZE Deep Dive

```sql
-- Understanding execution plans
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, o.total, c.name, p.title
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN LATERAL (
  SELECT oi.product_id, p.title
  FROM order_items oi
  JOIN products p ON oi.product_id = p.id
  WHERE oi.order_id = o.id
  LIMIT 5
) p ON true
WHERE o.created_at > NOW() - INTERVAL '30 days'
  AND o.status = 'completed'
ORDER BY o.total DESC
LIMIT 100;

/*
Key metrics to analyze:
- actual time: Real execution time per node
- rows: Actual vs estimated rows (large differences = stale statistics)
- loops: Number of times node executed
- buffers: shared hit (cache) vs read (disk)
- I/O timing: Time spent on disk I/O (enable track_io_timing)

Red flags:
- Seq Scan on large tables
- Nested Loop with high loop count
- Sort with external merge (memory exceeded)
- Hash/Merge joins with high bucket count
*/
```

## Index Strategies

```sql
-- Partial indexes for hot data
CREATE INDEX CONCURRENTLY idx_orders_pending
ON orders (created_at DESC)
WHERE status = 'pending';

-- Covering indexes to avoid table lookups
CREATE INDEX CONCURRENTLY idx_orders_covering
ON orders (customer_id, status)
INCLUDE (total, created_at);

-- Expression indexes
CREATE INDEX CONCURRENTLY idx_users_email_lower
ON users (LOWER(email));

-- GIN for JSONB and arrays
CREATE INDEX CONCURRENTLY idx_products_tags
ON products USING GIN (tags);

CREATE INDEX CONCURRENTLY idx_events_metadata
ON events USING GIN (metadata jsonb_path_ops);

-- BRIN for time-series data (much smaller than B-tree)
CREATE INDEX CONCURRENTLY idx_logs_created_brin
ON logs USING BRIN (created_at)
WITH (pages_per_range = 32);

-- Index for pattern matching
CREATE INDEX CONCURRENTLY idx_products_name_trgm
ON products USING GIN (name gin_trgm_ops);

-- Validate index usage
SELECT
  schemaname,
  relname,
  indexrelname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch,
  pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

## Advanced Query Patterns

```sql
-- Window functions for running calculations
WITH order_metrics AS (
  SELECT
    customer_id,
    created_at::date as order_date,
    total,
    -- Running total
    SUM(total) OVER (
      PARTITION BY customer_id
      ORDER BY created_at
      ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) as running_total,
    -- Moving average
    AVG(total) OVER (
      PARTITION BY customer_id
      ORDER BY created_at
      ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as moving_avg_7d,
    -- Rank within customer
    ROW_NUMBER() OVER (
      PARTITION BY customer_id
      ORDER BY total DESC
    ) as order_rank,
    -- Gap analysis
    created_at - LAG(created_at) OVER (
      PARTITION BY customer_id
      ORDER BY created_at
    ) as days_since_last_order
  FROM orders
  WHERE status = 'completed'
)
SELECT * FROM order_metrics WHERE order_rank <= 3;

-- Recursive CTE for hierarchies
WITH RECURSIVE org_tree AS (
  -- Base case
  SELECT id, name, manager_id, 1 as level, ARRAY[id] as path
  FROM employees
  WHERE manager_id IS NULL

  UNION ALL

  -- Recursive case
  SELECT e.id, e.name, e.manager_id, t.level + 1, t.path || e.id
  FROM employees e
  JOIN org_tree t ON e.manager_id = t.id
  WHERE NOT e.id = ANY(t.path) -- Prevent cycles
)
SELECT
  REPEAT('  ', level - 1) || name as org_chart,
  level,
  array_to_string(path, ' -> ') as reporting_chain
FROM org_tree
ORDER BY path;

-- Lateral joins for top-N per group
SELECT c.id, c.name, recent_orders.*
FROM customers c
CROSS JOIN LATERAL (
  SELECT o.id, o.total, o.created_at
  FROM orders o
  WHERE o.customer_id = c.id
    AND o.status = 'completed'
  ORDER BY o.created_at DESC
  LIMIT 3
) recent_orders
WHERE c.tier = 'premium';
```

## Query Optimization Techniques

```sql
-- Avoid SELECT *
-- Bad
SELECT * FROM orders WHERE customer_id = 123;

-- Good (only needed columns)
SELECT id, total, status, created_at FROM orders WHERE customer_id = 123;

-- Use EXISTS instead of IN for subqueries
-- Slower
SELECT * FROM customers
WHERE id IN (SELECT customer_id FROM orders WHERE total > 1000);

-- Faster
SELECT * FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders o
  WHERE o.customer_id = c.id AND o.total > 1000
);

-- Batch operations to avoid N+1
-- N+1 queries
FOR customer IN SELECT * FROM customers LOOP
  SELECT COUNT(*) FROM orders WHERE customer_id = customer.id;
END LOOP;

-- Single query with aggregation
SELECT c.*, COALESCE(order_counts.count, 0) as order_count
FROM customers c
LEFT JOIN (
  SELECT customer_id, COUNT(*) as count
  FROM orders
  GROUP BY customer_id
) order_counts ON c.id = order_counts.customer_id;

-- Materialized views for complex aggregations
CREATE MATERIALIZED VIEW mv_customer_stats AS
SELECT
  c.id,
  c.name,
  COUNT(o.id) as total_orders,
  SUM(o.total) as lifetime_value,
  MAX(o.created_at) as last_order_date,
  AVG(o.total) as avg_order_value
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id AND o.status = 'completed'
GROUP BY c.id, c.name;

CREATE UNIQUE INDEX ON mv_customer_stats (id);

-- Refresh concurrently (no locks)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_customer_stats;
```

## Connection Pooling & Prepared Statements

```typescript
// PgBouncer configuration for connection pooling
/*
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction  # transaction pooling
max_client_conn = 1000
default_pool_size = 20
reserve_pool_size = 5
reserve_pool_timeout = 3
server_idle_timeout = 600
*/

// Prepared statements in application code
import { Pool } from 'pg';

const pool = new Pool({
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Named prepared statement (cached per connection)
async function getOrdersByCustomer(customerId: string) {
  const query = {
    name: 'get-orders-by-customer',
    text: `
      SELECT id, total, status, created_at
      FROM orders
      WHERE customer_id = $1
        AND created_at > NOW() - INTERVAL '1 year'
      ORDER BY created_at DESC
      LIMIT 100
    `,
    values: [customerId],
  };

  return pool.query(query);
}

// Batch insert with UNNEST
async function batchInsertOrders(orders: Order[]) {
  const query = `
    INSERT INTO orders (customer_id, total, status, items)
    SELECT * FROM UNNEST(
      $1::uuid[],
      $2::numeric[],
      $3::text[],
      $4::jsonb[]
    )
  `;

  await pool.query(query, [
    orders.map(o => o.customerId),
    orders.map(o => o.total),
    orders.map(o => o.status),
    orders.map(o => JSON.stringify(o.items)),
  ]);
}
```

## Statistics & Maintenance

```sql
-- Update statistics for better query plans
ANALYZE orders;
ANALYZE VERBOSE orders; -- With progress

-- Increase statistics target for skewed columns
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;

-- Extended statistics for correlated columns
CREATE STATISTICS orders_customer_status (dependencies)
ON customer_id, status FROM orders;

-- Identify bloat
SELECT
  schemaname || '.' || relname as table,
  pg_size_pretty(pg_total_relation_size(relid)) as total_size,
  pg_size_pretty(pg_relation_size(relid)) as table_size,
  pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid)) as index_size,
  n_dead_tup as dead_tuples,
  n_live_tup as live_tuples,
  ROUND(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) as dead_ratio
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;

-- Vacuum and reindex
VACUUM (VERBOSE, ANALYZE) orders;
REINDEX INDEX CONCURRENTLY idx_orders_customer_id;
```

---

*Learned: December 20, 2025*
*Tags: PostgreSQL, Database, Query Optimization, Performance, SQL*
