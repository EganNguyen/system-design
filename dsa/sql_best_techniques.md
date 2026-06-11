# SQL Database — Requirements & Best Techniques

> A complete reference guide mapping common database requirements to their optimal implementation strategies.

---

## Quick Reference Table

| # | Requirement | Best Technique |
|---|-------------|----------------|
| 1 | Lookup by ID | Index Seek |
| 2 | Check existence | EXISTS |
| 3 | Latest row per group | CROSS APPLY + TOP(1) |
| 4 | Top N ranking | ROW_NUMBER() |
| 5 | Heavy aggregation | Summary Table |
| 6 | Dashboard | Materialized View / Summary Table |
| 7 | Billion-row history | Partitioning |
| 8 | Analytics | Columnstore Index |
| 9 | Repeated joins | Denormalization |
| 10 | Real-time reads | Master Table |
| 11 | Huge reporting query | Temp Tables + Pre-aggregation |
| 12 | Very high traffic | Cache + Read Replica |
| 13 | Enterprise scale | CQRS |
| 14 | Fuzzy / full-text search | Full-Text Index / FTS |
| 15 | Range queries (dates, prices) | Clustered Index on range column |
| 16 | Duplicate detection | DISTINCT + CTE |
| 17 | Hierarchical data | Recursive CTE / HierarchyID |
| 18 | Running totals / cumulative | Window Functions (SUM OVER) |
| 19 | Pagination (large dataset) | Keyset Pagination |
| 20 | Upsert (insert or update) | MERGE / INSERT ON CONFLICT |
| 21 | Conditional aggregation | CASE inside aggregate |
| 22 | Multi-tenant isolation | Row-Level Security (RLS) |
| 23 | Soft deletes | Filtered Index on IsDeleted |
| 24 | Audit trail | Temporal Tables / CDC |
| 25 | JSON / semi-structured data | JSON columns + JSON_VALUE index |
| 26 | Graph / network traversal | Recursive CTE / Graph Tables |
| 27 | Bulk inserts | Bulk Insert / COPY / Batch Insert |
| 28 | Schema changes with no downtime | Online Index Rebuild |
| 29 | Cross-database reporting | Linked Server / Federated Query |
| 30 | Preventing dirty reads | Snapshot Isolation (RCSI) |

---

## Detailed Breakdown

---

### 1. Lookup by ID — Index Seek

**When to use:** Point lookups on primary keys or unique columns.

```sql
-- Ensure a clustered or covering index exists
CREATE UNIQUE INDEX idx_users_id ON users (user_id);

-- Query that triggers Index Seek (not Scan)
SELECT name, email
FROM users
WHERE user_id = 42;
```

**Key points:**
- Always use a clustered primary key for the main lookup column
- Avoid functions on indexed columns: `WHERE YEAR(created_at) = 2024` disables seek — use range instead
- Include frequently selected columns in a covering index to avoid key lookups

---

### 2. Check Existence — EXISTS

**When to use:** You only need to know *if* rows exist, not retrieve them.

```sql
-- ✅ Efficient — stops at first match
IF EXISTS (SELECT 1 FROM orders WHERE customer_id = 101 AND status = 'pending')
    PRINT 'Has pending orders';

-- ❌ Avoid COUNT(*) for existence checks
IF (SELECT COUNT(*) FROM orders WHERE customer_id = 101) > 0 ...
```

**Key points:**
- `EXISTS` short-circuits on the first matching row
- `NOT EXISTS` is generally faster than `NOT IN` when nulls may be present
- Use `EXISTS` in WHERE clauses for correlated subqueries

---

### 3. Latest Row per Group — CROSS APPLY + TOP(1)

**When to use:** Retrieve the most recent (or any single) row per group efficiently.

```sql
-- Latest order per customer
SELECT c.customer_id, c.name, o.order_date, o.total
FROM customers c
CROSS APPLY (
    SELECT TOP(1) order_date, total
    FROM orders
    WHERE customer_id = c.customer_id
    ORDER BY order_date DESC
) o;

-- Alternative with ROW_NUMBER (less efficient for large datasets)
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
) t WHERE rn = 1;
```

**Key points:**
- `CROSS APPLY + TOP(1)` leverages the index seek per row — very fast
- Use `OUTER APPLY` to include customers with no orders (like LEFT JOIN)
- Requires an index on `(customer_id, order_date DESC)`

---

### 4. Top N Ranking — ROW_NUMBER()

**When to use:** Return top N rows per group, rank results, or assign sequential numbers.

```sql
-- Top 3 products by sales per category
SELECT category, product_name, sales
FROM (
    SELECT
        category,
        product_name,
        sales,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS rn
    FROM products
) ranked
WHERE rn <= 3;
```

**Window functions comparison:**

| Function | Behavior on Ties |
|----------|-----------------|
| `ROW_NUMBER()` | Unique rank — arbitrary tiebreak |
| `RANK()` | Gaps after ties (1, 1, 3) |
| `DENSE_RANK()` | No gaps (1, 1, 2) |
| `NTILE(n)` | Divide into N buckets |

---

### 5. Heavy Aggregation — Summary Table

**When to use:** Aggregations over millions of rows that are too slow to compute on the fly.

```sql
-- Pre-aggregated summary table (refreshed by job/trigger)
CREATE TABLE sales_daily_summary (
    summary_date DATE NOT NULL,
    region_id INT NOT NULL,
    total_revenue DECIMAL(18,2),
    order_count INT,
    PRIMARY KEY (summary_date, region_id)
);

-- Populate/refresh
INSERT INTO sales_daily_summary
SELECT
    CAST(order_date AS DATE),
    region_id,
    SUM(amount),
    COUNT(*)
FROM orders
WHERE order_date >= DATEADD(DAY, -1, GETDATE())
GROUP BY CAST(order_date AS DATE), region_id
ON CONFLICT DO UPDATE SET ...;
```

**Key points:**
- Refresh on a schedule (cron, SQL Agent) or via triggers
- Pair with a materialized view where the database supports it natively
- Keep grain (level of detail) consistent with query needs

---

### 6. Dashboard — Materialized View / Summary Table

**When to use:** Dashboards with frequent reads, slow underlying queries, acceptable staleness.

```sql
-- PostgreSQL materialized view
CREATE MATERIALIZED VIEW dashboard_kpis AS
SELECT
    DATE_TRUNC('day', created_at) AS day,
    COUNT(*) AS new_users,
    SUM(revenue) AS daily_revenue
FROM events
GROUP BY 1;

-- Refresh (can be scheduled)
REFRESH MATERIALIZED VIEW CONCURRENTLY dashboard_kpis;

-- SQL Server indexed view equivalent
CREATE VIEW dbo.vw_sales_summary WITH SCHEMABINDING AS
SELECT region_id, COUNT_BIG(*) AS cnt, SUM(amount) AS total
FROM dbo.orders
GROUP BY region_id;

CREATE UNIQUE CLUSTERED INDEX idx_vw ON dbo.vw_sales_summary(region_id);
```

**Key points:**
- `CONCURRENTLY` (PostgreSQL) allows refresh without locking reads
- Materialized views trade freshness for speed — ideal for hourly/daily dashboards
- In SQL Server, indexed views auto-update but have strict restrictions

---

### 7. Billion-Row History — Partitioning

**When to use:** Tables with massive historical data where queries are typically scoped to a time range.

```sql
-- SQL Server: partition by year
CREATE PARTITION FUNCTION pf_orders_year (DATE)
AS RANGE RIGHT FOR VALUES ('2022-01-01', '2023-01-01', '2024-01-01');

CREATE PARTITION SCHEME ps_orders
AS PARTITION pf_orders_year ALL TO ([PRIMARY]);

CREATE TABLE orders (
    order_id BIGINT,
    order_date DATE NOT NULL,
    amount DECIMAL(18,2)
) ON ps_orders(order_date);

-- PostgreSQL: declarative partitioning
CREATE TABLE orders (
    order_id BIGINT,
    order_date DATE NOT NULL
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2024
    PARTITION OF orders FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

**Key points:**
- Partition elimination makes queries touch only relevant partitions
- Archive / drop old partitions instantly by switching out a partition
- Combine with columnstore indexes on old partitions for analytics
- Keep partition count reasonable (< 1000 for SQL Server)

---

### 8. Analytics — Columnstore Index

**When to use:** Analytical queries scanning large volumes of data, aggregations, OLAP workloads.

```sql
-- Add non-clustered columnstore for analytics on an OLTP table
CREATE NONCLUSTERED COLUMNSTORE INDEX ncci_orders_analytics
ON orders (order_date, region_id, product_id, amount);

-- Clustered columnstore for a dedicated analytics/fact table
CREATE CLUSTERED COLUMNSTORE INDEX cci_fact_sales
ON fact_sales;
```

**Key points:**
- Columnstore compresses data 10x and uses batch execution (128 rows at a time)
- Ideal for queries touching many rows but few columns
- In SQL Server, updatable non-clustered columnstore is supported via delta store
- PostgreSQL equivalent: use `cstore_fdw` extension or switch to a columnar engine (DuckDB, Redshift, BigQuery)

---

### 9. Repeated Joins — Denormalization

**When to use:** Queries that repeatedly join the same tables for a hot path, and join cost is measurable.

```sql
-- Instead of joining orders → customers every query:
ALTER TABLE orders ADD COLUMN customer_name VARCHAR(200);
ALTER TABLE orders ADD COLUMN customer_email VARCHAR(200);

-- Keep in sync via trigger or application logic
CREATE TRIGGER trg_sync_customer_name
AFTER UPDATE ON customers
FOR EACH ROW
UPDATE orders SET customer_name = NEW.name WHERE customer_id = NEW.customer_id;
```

**When NOT to denormalize:**
- Data changes frequently (sync cost exceeds join cost)
- Storage is constrained
- Correctness is harder to maintain

---

### 10. Real-Time Reads — Master Table

**When to use:** A single authoritative table is required for real-time, point-in-time consistent reads.

```sql
-- Master table acts as the single source of truth
CREATE TABLE product_master (
    product_id INT PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(10,2),
    stock INT,
    last_updated DATETIME2 DEFAULT SYSUTCDATETIME()
);

-- All real-time reads go here; analytics reads go to replicas or summary tables
SELECT price, stock FROM product_master WHERE product_id = 99;
```

**Key points:**
- Master table + read replicas pattern: writes to master, reads from replica
- Use optimistic concurrency (`rowversion` / `etag`) to prevent stale overwrites
- Combine with Redis cache in front for sub-millisecond reads

---

### 11. Huge Reporting Query — Temp Tables + Pre-aggregation

**When to use:** Complex reports with multiple steps that benefit from intermediate materialization.

```sql
-- Step 1: Pre-aggregate into temp table
DROP TABLE IF EXISTS #daily_sales;
SELECT
    CAST(order_date AS DATE) AS day,
    region_id,
    SUM(amount) AS revenue
INTO #daily_sales
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY CAST(order_date AS DATE), region_id;

CREATE INDEX idx_tmp ON #daily_sales (day, region_id);

-- Step 2: Join temp table (fast — small result set)
SELECT r.region_name, d.day, d.revenue
FROM #daily_sales d
JOIN regions r ON r.region_id = d.region_id
ORDER BY d.day;
```

**Key points:**
- Temp tables allow the optimizer to gather statistics mid-query
- Break one massive query into stages — easier to debug and tune
- Use `#temp` tables (SQL Server) or `CREATE TEMP TABLE` (PostgreSQL) — scoped to session
- CTEs do NOT materialize by default; temp tables do

---

### 12. Very High Traffic — Cache + Read Replica

**When to use:** Read-heavy workloads exceeding what a single DB can handle, or sub-millisecond latency required.

```
Architecture:

  [Application]
       |
  [Redis Cache] ←── Cache miss ──→ [Read Replica 1]
                                   [Read Replica 2]
                                         ↑
                                   [Primary DB] ←── Writes
```

```sql
-- Application-level cache-aside pattern (pseudo-code)
value = redis.get("product:99")
IF value IS NULL:
    value = db.query("SELECT * FROM products WHERE product_id = 99")
    redis.set("product:99", value, TTL=300)
RETURN value
```

**Key points:**
- Read replicas handle SELECT load; primary handles writes only
- Redis TTL controls freshness — tune per data volatility
- Watch for replication lag: read replicas may be milliseconds behind
- Use connection pooling (PgBouncer, ProxySQL) to limit DB connections

---

### 13. Enterprise Scale — CQRS

**When to use:** Systems where read and write models have fundamentally different requirements.

```
CQRS Pattern:

  [Write Side]                    [Read Side]
  Command → Handler → Domain      Query → Read Model (optimized view)
                 ↓                              ↑
           [Event Store]  ──→  [Projection]  ──→ [Read DB]
```

```sql
-- Write model: normalized, transactional
INSERT INTO order_events (order_id, event_type, payload, created_at)
VALUES (1, 'OrderPlaced', '{"items": [...]}', NOW());

-- Read model: denormalized, optimized for queries
CREATE TABLE order_read_model (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(200),
    total DECIMAL(18,2),
    status VARCHAR(50),
    item_count INT,
    last_updated TIMESTAMP
);
```

**Key points:**
- Separate databases for reads and writes are common
- Event sourcing often pairs with CQRS
- Eventual consistency is accepted for the read side
- Complexity is high — reserve for enterprise-scale systems

---

### 14. Fuzzy / Full-Text Search — Full-Text Index

**When to use:** Searching text content (articles, descriptions, comments) where LIKE is too slow.

```sql
-- SQL Server
CREATE FULLTEXT INDEX ON products(description) KEY INDEX pk_products;
SELECT * FROM products WHERE CONTAINS(description, '"wireless" AND "noise cancelling"');

-- PostgreSQL
CREATE INDEX idx_fts_products ON products USING GIN(to_tsvector('english', description));
SELECT * FROM products WHERE to_tsvector('english', description) @@ plainto_tsquery('wireless headphones');
```

---

### 15. Range Queries — Clustered Index on Range Column

**When to use:** Date ranges, price filters, sequential ID scans.

```sql
-- Date-ranged clustered index
CREATE CLUSTERED INDEX idx_orders_date ON orders (order_date, order_id);

-- Efficient range query
SELECT * FROM orders WHERE order_date BETWEEN '2024-06-01' AND '2024-06-30';
```

---

### 16. Duplicate Detection — DISTINCT + CTE

```sql
-- Find and remove duplicates, keeping lowest ID
WITH dupes AS (
    SELECT
        email,
        MIN(user_id) AS keep_id
    FROM users
    GROUP BY email
    HAVING COUNT(*) > 1
)
DELETE FROM users
WHERE email IN (SELECT email FROM dupes)
  AND user_id NOT IN (SELECT keep_id FROM dupes);
```

---

### 17. Hierarchical Data — Recursive CTE / HierarchyID

```sql
-- Recursive CTE for org chart
WITH org_tree AS (
    SELECT employee_id, name, manager_id, 0 AS level
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.employee_id, e.name, e.manager_id, t.level + 1
    FROM employees e
    INNER JOIN org_tree t ON e.manager_id = t.employee_id
)
SELECT * FROM org_tree ORDER BY level, name;
```

---

### 18. Running Totals — Window Functions

```sql
-- Cumulative revenue by day
SELECT
    order_date,
    daily_revenue,
    SUM(daily_revenue) OVER (ORDER BY order_date ROWS UNBOUNDED PRECEDING) AS cumulative_revenue,
    AVG(daily_revenue) OVER (ORDER BY order_date ROWS 6 PRECEDING) AS rolling_7day_avg
FROM daily_sales;
```

---

### 19. Pagination (Large Dataset) — Keyset Pagination

```sql
-- ❌ Offset pagination — slow at high offsets
SELECT * FROM orders ORDER BY order_id OFFSET 100000 ROWS FETCH NEXT 20 ROWS ONLY;

-- ✅ Keyset pagination — O(log n) always
SELECT * FROM orders
WHERE order_id > :last_seen_id   -- cursor from previous page
ORDER BY order_id
FETCH FIRST 20 ROWS ONLY;
```

---

### 20. Upsert — MERGE / INSERT ON CONFLICT

```sql
-- SQL Server MERGE
MERGE target_table AS t
USING source_table AS s ON t.id = s.id
WHEN MATCHED THEN UPDATE SET t.value = s.value
WHEN NOT MATCHED THEN INSERT (id, value) VALUES (s.id, s.value);

-- PostgreSQL INSERT ON CONFLICT
INSERT INTO settings (key, value)
VALUES ('theme', 'dark')
ON CONFLICT (key) DO UPDATE SET value = EXCLUDED.value;
```

---

### 21. Conditional Aggregation — CASE inside Aggregate

```sql
SELECT
    region_id,
    COUNT(*) AS total_orders,
    SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END) AS completed,
    SUM(CASE WHEN status = 'refunded' THEN amount ELSE 0 END) AS refunded_revenue
FROM orders
GROUP BY region_id;
```

---

### 22. Multi-Tenant Isolation — Row-Level Security (RLS)

```sql
-- PostgreSQL RLS
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant')::INT);
```

---

### 23. Soft Deletes — Filtered Index

```sql
ALTER TABLE users ADD COLUMN is_deleted BIT DEFAULT 0;

-- Filtered index ignores deleted rows — most queries stay fast
CREATE INDEX idx_active_users ON users (email) WHERE is_deleted = 0;

-- Query automatically uses the filtered index
SELECT * FROM users WHERE email = 'alice@example.com' AND is_deleted = 0;
```

---

### 24. Audit Trail — Temporal Tables / CDC

```sql
-- SQL Server temporal table (auto-versioning)
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    price DECIMAL(10,2),
    valid_from DATETIME2 GENERATED ALWAYS AS ROW START,
    valid_to   DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (valid_from, valid_to)
) WITH (SYSTEM_VERSIONING = ON);

-- Query history
SELECT * FROM products FOR SYSTEM_TIME AS OF '2024-01-15 12:00:00';
```

---

### 25. JSON / Semi-Structured Data — JSON Columns + Index

```sql
-- PostgreSQL JSONB with index
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    payload JSONB
);
CREATE INDEX idx_events_type ON events ((payload->>'event_type'));

SELECT * FROM events WHERE payload->>'event_type' = 'purchase';
```

---

### 26. Bulk Inserts — BULK INSERT / COPY / Batch

```sql
-- SQL Server bulk insert
BULK INSERT orders FROM 'C:\data\orders.csv'
WITH (FIELDTERMINATOR = ',', ROWTERMINATOR = '\n', FIRSTROW = 2);

-- PostgreSQL COPY (fastest native method)
COPY orders FROM '/data/orders.csv' DELIMITER ',' CSV HEADER;

-- Application-level batch insert (avoid row-by-row)
INSERT INTO orders (order_id, amount) VALUES
    (1, 99.99), (2, 149.50), (3, 20.00);  -- batch of N rows
```

---

### 27. Preventing Dirty Reads — Snapshot Isolation

```sql
-- SQL Server: enable RCSI (Read Committed Snapshot Isolation)
ALTER DATABASE MyDB SET READ_COMMITTED_SNAPSHOT ON;

-- PostgreSQL: uses MVCC by default; use REPEATABLE READ for consistency
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE account_id = 1;
-- No dirty reads; snapshot taken at transaction start
COMMIT;
```

---

## Technique Selection Cheat Sheet

```
Is your problem about...

  SPEED of point lookups?       → Index Seek, Covering Index
  EXISTENCE checks?             → EXISTS
  LATEST row per group?         → CROSS APPLY + TOP(1)
  RANKING rows?                 → ROW_NUMBER / RANK / DENSE_RANK
  SLOW aggregation?             → Summary Table, Materialized View
  HISTORICAL / archive data?    → Partitioning
  ANALYTICAL queries?           → Columnstore Index
  REPEATED expensive joins?     → Denormalization
  HIGH READ traffic?            → Cache (Redis) + Read Replica
  COMPLEX reporting?            → Temp Tables + Pre-aggregation
  ENTERPRISE read/write split?  → CQRS
  TEXT search?                  → Full-Text Index / FTS
  PAGINATION at scale?          → Keyset Pagination
  AUDIT / history?              → Temporal Tables / CDC
  MULTI-TENANT data?            → Row-Level Security
```

---

*Last updated: 2024 | Covers SQL Server, PostgreSQL, MySQL/MariaDB patterns*
