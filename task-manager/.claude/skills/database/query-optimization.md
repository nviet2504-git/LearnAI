---
name: sql-query-optimization
description: >
  Optimize SQL query performance through execution plan analysis, strategic indexing,
  query rewriting, and caching. Use this skill when analyzing slow queries, improving
  database performance, reducing query execution time, avoiding table scans, tuning
  indexes, optimizing joins, rewriting subqueries, analyzing execution plans, reducing
  I/O costs, preventing full scans, implementing query caching, or whenever performance
  bottlenecks, high CPU usage, memory pressure, or slow response times are mentioned.
  Also applies to database tuning, cost analysis, cardinality issues, missing indexes,
  or any discussion of SELECT optimization, WHERE clause efficiency, or resource consumption.
---

# SQL Query Optimization

## Overview

Query optimization transforms slow, resource-intensive queries into fast, efficient ones. The core insight: databases provide detailed visibility into *how* they execute queries, and most performance problems stem from a handful of fixable patterns—missing indexes, inefficient joins, non-sargable predicates, or excessive data retrieval.

Optimization is iterative: measure → analyze → fix → verify. Start with execution plans to identify the highest-cost operations, then apply targeted fixes. A 10x speedup often comes from fixing 2-3 critical issues, not rewriting everything.

## Decision Tree: Where to Start

1. **Is the query already slow in production?**
   - Capture actual execution plan and current metrics (duration, reads, CPU)
   - Jump to Execution Plan Analysis

2. **Building a new query or feature?**
   - Start with estimated execution plan
   - Check for scans, high-cost operations, or missing index warnings
   - If clean, proceed; if not, fix before deployment

3. **Performance degraded recently?**
   - Check statistics freshness (stale stats cause bad plans)
   - Compare current vs. historical execution plans
   - Look for parameter sniffing or plan cache issues

4. **General database slowness?**
   - Identify top resource consumers (CPU, I/O, duration) via DMVs or query stores
   - Optimize the top 5-10 queries—these often account for 80%+ of load

## Execution Plan Analysis

### Reading Plans

Execution plans flow **right-to-left, bottom-to-top**. Each operator shows:
- **Operation type**: Scan, Seek, Join method, Sort, etc.
- **Estimated vs. Actual rows**: Large mismatches signal stale statistics or cardinality issues
- **Cost percentage**: Focus on operators consuming >20% of total cost

### Key Red Flags

**Table Scans / Clustered Index Scans**
Reading every row when you need a subset. Causes: missing indexes, non-sargable WHERE clauses.

**Key Lookups (RID/Key Lookups)**
Index found the rows, but must jump back to table for missing columns. Fix: covering indexes (INCLUDE clause).

**Hash Match / Sort operators at high cost**
Often indicates missing indexes on JOIN or ORDER BY columns, or memory pressure.

**Nested Loop joins on large tables**
Efficient for small datasets; catastrophic for large ones without proper indexes. Consider hash or merge joins.

**Cardinality estimate mismatches**
Estimated rows: 100, Actual rows: 1,000,000 → optimizer chose wrong plan. Update statistics or investigate parameter sniffing.

### Tools by Platform

- **SQL Server**: `SET STATISTICS IO ON; SET STATISTICS TIME ON;` + graphical plan (Ctrl+M in SSMS)
- **PostgreSQL**: `EXPLAIN ANALYZE`
- **MySQL**: `EXPLAIN` or `EXPLAIN ANALYZE`
- **Oracle**: `EXPLAIN PLAN` + `DBMS_XPLAN.DISPLAY`

Always use **actual** execution plans for production issues (they include real row counts). Use **estimated** plans during development to avoid running expensive queries.

## Indexing Strategy

### When to Index

Index columns that appear in:
- **WHERE clauses** (especially equality and range filters)
- **JOIN conditions**
- **ORDER BY / GROUP BY** (can eliminate sorts)

Don't index:
- Low-cardinality columns in OLTP (e.g., boolean flags)—exception: bitmap indexes in data warehouses
- Columns that change frequently in write-heavy tables
- Tables with <1000 rows (scans are faster)

### Index Types

**B-Tree (default in most systems)**
Best for: equality, range queries, sorting. Covers 90% of use cases.

**Covering Indexes**
Include all columns needed by the query in the index itself—eliminates lookups.
```sql
CREATE INDEX idx_orders_covering 
ON orders (customer_id, order_date) 
INCLUDE (total_amount, status);
```

**Composite Indexes**
Order matters: most selective column first, then filter columns, then sort columns.
```sql
-- Good for: WHERE customer_id = X AND order_date > Y ORDER BY order_date
CREATE INDEX idx_orders_composite 
ON orders (customer_id, order_date);
```

**Filtered Indexes**
Index a subset of rows—useful for queries that filter on specific values.
```sql
CREATE INDEX idx_active_orders 
ON orders (order_date) 
WHERE status = 'active';
```

### Index Maintenance

- **Update statistics regularly**: Stale stats → bad plans. Most systems auto-update, but verify on large tables.
- **Monitor index usage**: Remove unused indexes—they slow writes and waste space.
- **Rebuild fragmented indexes**: Fragmentation >30% hurts scan performance.

### Over-Indexing

Each index costs:
- Disk space
- Write overhead (INSERT/UPDATE/DELETE must update all indexes)
- Plan compilation time

Aim for 3-5 indexes per table in OLTP; more acceptable in read-heavy warehouses.

## Rewriting Slow Queries

### SELECT Only What You Need

**Bad:**
```sql
SELECT * FROM orders WHERE customer_id = 123;
```

**Good:**
```sql
SELECT order_id, order_date, total_amount 
FROM orders 
WHERE customer_id = 123;
```

Why: Reduces I/O, memory, network transfer. Enables covering indexes.

### Make Predicates Sargable

"Sargable" = Search ARGument ABLE = can use indexes.

**Non-sargable (forces scan):**
```sql
WHERE YEAR(order_date) = 2023
WHERE UPPER(last_name) = 'SMITH'
WHERE salary * 1.1 > 50000
```

**Sargable (uses index):**
```sql
WHERE order_date >= '2023-01-01' AND order_date < '2024-01-01'
WHERE last_name = 'Smith'  -- store data in proper case or use case-insensitive collation
WHERE salary > 50000 / 1.1
```

Rule: Don't wrap indexed columns in functions or expressions.

### Optimize Joins

**Index both sides of the join:**
```sql
SELECT o.order_id, c.name
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date > '2023-01-01';
```
Needs indexes on `orders(order_date)` and `customers(customer_id)`.

**Filter early:**
Apply WHERE conditions before joins when possible—reduces rows entering the join.

**Join order:**
Modern optimizers handle this, but if you see inefficiency: join smallest result set first.

**Avoid Cartesian products:**
Always include join conditions. Missing ON clause = every row × every row.

### Subqueries vs. JOINs vs. CTEs

**Correlated subqueries** (run once per outer row) are often slow:
```sql
-- Slow
SELECT name 
FROM customers c
WHERE (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id) > 5;
```

**Rewrite as JOIN:**
```sql
SELECT c.name
FROM customers c
INNER JOIN (
  SELECT customer_id 
  FROM orders 
  GROUP BY customer_id 
  HAVING COUNT(*) > 5
) o ON c.customer_id = o.customer_id;
```

**Or use EXISTS** (often faster than IN for large subqueries):
```sql
SELECT name 
FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders o 
  WHERE o.customer_id = c.customer_id 
  HAVING COUNT(*) > 5
);
```

**CTEs** improve readability but don't always optimize well—some systems materialize them. Test both CTE and subquery versions.

### UNION vs. UNION ALL

**UNION** deduplicates (requires sort/hash) → slower.
**UNION ALL** stacks results → faster.

Use UNION ALL unless duplicates are a problem.

### Limit Result Sets

**Use LIMIT / TOP / FETCH FIRST:**
```sql
SELECT TOP 100 order_id, order_date 
FROM orders 
WHERE customer_id = 123 
ORDER BY order_date DESC;
```

Prevents accidentally returning millions of rows. Essential for pagination.

## Avoiding Full Scans

Full scans aren't always bad—on small tables (<1000 rows), scans are often faster than index seeks due to overhead. But on large tables:

### Common Causes

1. **No index on filter column** → add index
2. **Non-sargable predicate** → rewrite (see above)
3. **Selecting most rows anyway** → scan is correct choice
4. **Stale statistics** → update stats
5. **Type mismatch** (e.g., joining INT to VARCHAR) → fix schema or cast explicitly

### Force Index Usage (last resort)

Most systems have hints to force index usage, but use sparingly—optimizer is usually right.
```sql
-- SQL Server
SELECT * FROM orders WITH (INDEX(idx_customer_id)) WHERE customer_id = 123;
```

Only use when you've verified the optimizer is wrong (rare).

## Caching Mechanisms

### Query Result Caching

Cache results of expensive, frequently-run queries that don't change often.

**Application-level caching** (Redis, Memcached):
- Full control over invalidation
- Works across database restarts
- Best for: dashboards, reports, aggregations

**Database query caches** (MySQL query cache, SQL Server plan cache):
- Automatic, but limited control
- Invalidated on any table change (can be too aggressive)

**Materialized views:**
Pre-computed query results stored as a table. Refresh periodically or on-demand.
```sql
CREATE MATERIALIZED VIEW sales_summary AS
SELECT product_id, SUM(quantity) as total_sold
FROM orders
GROUP BY product_id;
```

Best for: complex aggregations queried often, updated infrequently.

### Plan Caching

Databases cache execution plans to avoid recompilation. Issues:

**Parameter sniffing**: Plan optimized for first parameter value may be wrong for others.
Fix: Use `OPTION (RECOMPILE)` for queries with highly variable parameters, or use plan guides.

**Plan cache bloat**: Thousands of ad-hoc queries with different literals.
Fix: Use parameterized queries or stored procedures.

## Verification Workflow

1. **Baseline**: Capture current metrics (duration, CPU, reads, writes)
2. **Change**: Apply one optimization at a time
3. **Measure**: Re-run with same data, compare metrics
4. **Validate**: Check execution plan—verify expected behavior (seek vs. scan, etc.)
5. **Test edge cases**: Empty tables, max rows, NULL values

Don't trust intuition—measure. A "better" query that's actually slower happens often.

## Common Pitfalls

- **Premature optimization**: Don't optimize queries that run once a day and take 2 seconds. Focus on high-frequency or user-facing queries.
- **Ignoring statistics**: 80% of "optimizer chose wrong plan" issues are stale statistics.
- **Over-indexing**: More indexes ≠ faster. Balance read vs. write performance.
- **Assuming newest features are faster**: Test. Sometimes older patterns (JOINs) beat newer ones (window functions) depending on data.
- **Optimizing in isolation**: A query might be slow because the database is under load, not because the query is bad.

## Platform-Specific Notes

### SQL Server
- Use Query Store for historical plan analysis
- `SET STATISTICS IO ON` shows logical reads (key metric)
- Missing index DMVs suggest indexes (but verify before creating)

### PostgreSQL
- `EXPLAIN (ANALYZE, BUFFERS)` shows I/O details
- `auto_explain` extension logs slow query plans automatically
- Vacuum regularly—bloat kills performance

### MySQL
- InnoDB uses clustered indexes (primary key = data order)
- `SHOW PROFILE` for detailed execution breakdown
- Avoid `SELECT *` especially with large TEXT/BLOB columns

### Oracle
- `DBMS_XPLAN.DISPLAY_CURSOR` for actual plans
- Bind variable peeking similar to SQL Server parameter sniffing
- Partitioning and parallel query powerful for large tables

## When to Escalate

- Query is optimized (good plan, proper indexes) but still slow → hardware or workload issue
- Cardinality estimates consistently wrong → statistics issue, consider histograms or manual stats
- Locking/blocking issues → transaction or isolation level problem, not query optimization
- Entire database slow → look at server resources (CPU, memory, disk I/O), not individual queries