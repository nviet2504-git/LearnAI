---
name: database-indexing-standards
description: >
  Define standards for proper database indexing across SQL and NoSQL systems. Use this when creating indexes, designing composite/compound indexes, optimizing query performance, preventing index bloat, aligning indexes with query patterns, reviewing index strategies, refactoring database schemas, or whenever database performance, indexing anti-patterns, index maintenance, or query optimization is mentioned. Covers index creation rules, composite key design (ESR rule), cardinality considerations, avoiding redundant indexes, monitoring index usage, and forbidden inefficient practices.
---

# Database Indexing Standards

## Core Principle

Indexes exist to serve queries, not the other way around. Every index must justify its existence through measurable query performance improvement. Indexes speed reads at the cost of write performance and storage—this tradeoff guides all decisions.

## Index Creation Standards

### Query-Driven Design

Create indexes based on actual or expected query patterns, never speculatively. Before creating an index:

1. Identify the query workload (frequency, importance, current performance)
2. Analyze execution plans to confirm the index will be used
3. Measure the performance impact
4. Document why the index exists

Indexes should target:
- Columns in WHERE clauses with high selectivity
- JOIN conditions, especially foreign keys
- ORDER BY and GROUP BY columns in frequent queries
- Columns with high cardinality (many distinct values)

### Avoid Redundant Indexes

Primary keys and unique constraints automatically create indexes—don't duplicate them. A composite index on (A, B, C) already covers queries on A alone and (A, B) due to index prefixes. Don't create separate indexes that are redundant subsets.

### Cardinality Matters

Low-cardinality columns (boolean flags, status fields with 2-3 values) rarely benefit from standalone indexes. The database may ignore them and scan the table instead. Focus on high-cardinality columns or use composite indexes where low-cardinality columns filter first in combination with others.

### Index Naming Convention

Use descriptive names: `idx_<table>_<col1>_<col2>` or `idx_<table>_<purpose>`. This makes index purpose clear during maintenance and prevents accidental duplication.

## Composite Index Design

### The ESR Rule (Equality, Sort, Range)

When designing composite indexes, order columns by query usage pattern:

1. **Equality** — Columns with exact match conditions (WHERE user_id = 123)
2. **Sort** — Columns used in ORDER BY
3. **Range** — Columns with range conditions (WHERE price > 50 AND price < 200)

Example:
```sql
-- Query pattern
SELECT * FROM orders 
WHERE user_id = 'user123' 
  AND price BETWEEN 50 AND 200 
ORDER BY created_at DESC;

-- Optimal index following ESR
CREATE INDEX idx_orders_user_created_price 
ON orders(user_id, created_at DESC, price);
```

Why ESR works: Equality conditions filter the dataset most aggressively, sort operations work efficiently on the reduced set, and range scans operate on the smallest remaining subset.

### Composite Index Limits

Most filtering happens in the first 1-3 columns of a composite index. Beyond 3 attributes, indexes contribute more to storage overhead than query performance. Keep composite indexes focused—if you need more than 3 columns, reconsider your query design or split into multiple targeted indexes.

### Left-Prefix Rule

Composite index on (A, B, C) can satisfy queries on:
- A alone
- A and B together  
- A, B, and C together

But NOT:
- B alone
- C alone
- B and C together

Columns must be used left-to-right without gaps. Design accordingly.

## Query Pattern Alignment

### Index to Query Mapping

For each index, document which queries it serves. For each slow query, verify an appropriate index exists. This bidirectional mapping prevents both missing indexes and index bloat.

Use database-specific tools:
- PostgreSQL: `pg_stat_statements`, `EXPLAIN ANALYZE`
- MySQL: Slow query log, `EXPLAIN`
- SQL Server: Query Store, execution plans
- MongoDB: `explain()`, profiler

### Covering Indexes

A covering index includes all columns needed to satisfy a query, eliminating table lookups. This is powerful but increases index size. Use for high-frequency queries where the performance gain justifies the storage cost.

Example:
```sql
-- Query needs only these three columns
SELECT name, email FROM users WHERE user_id = 123;

-- Covering index includes all needed columns
CREATE INDEX idx_users_id_name_email 
ON users(user_id, name, email);
```

### Filtered/Partial Indexes

When queries consistently filter on a subset of data, use filtered indexes (SQL Server) or partial indexes (PostgreSQL) to index only relevant rows. This reduces index size and improves performance.

```sql
-- PostgreSQL partial index
CREATE INDEX idx_orders_active 
ON orders(created_at) 
WHERE status = 'active';
```

## Preventing Index Bloat

Index bloat wastes storage, slows writes, and degrades read performance as indexes grow inefficiently.

### Monitor Index Usage

Regularly audit indexes to identify unused ones:

```sql
-- PostgreSQL: Find unused indexes
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE 'pg_toast%';
```

Unused indexes provide zero benefit but add overhead to every write operation. Remove them.

### Index Maintenance

- **Rebuild fragmented indexes** — B-tree indexes fragment over time with updates/deletes. Rebuild periodically (REINDEX in PostgreSQL, ALTER INDEX REBUILD in SQL Server).
- **Update statistics** — Query planners rely on statistics. Stale stats lead to poor index selection. Run ANALYZE/UPDATE STATISTICS regularly.
- **Monitor size growth** — Track index size over time. Unexpected growth signals bloat or design issues.

### Avoid Over-Indexing

Every index adds cost to INSERT, UPDATE, and DELETE operations. More indexes ≠ better performance. Target 3-5 indexes per table for most use cases. If you have 10+ indexes on a single table, audit whether they're all necessary.

### Wide Column Indexes

Avoid indexing very large text columns (VARCHAR(MAX), TEXT, BLOB). Index size explodes and performance degrades. For text search, use full-text indexes or dedicated search engines (Elasticsearch, Solr). For large binary data, don't index at all.

## Forbidden Inefficient Practices

### Never Do This

1. **Don't index every column** — Speculative indexing without query analysis wastes resources and slows writes.

2. **Don't create duplicate indexes** — Check existing indexes before creating new ones. Duplicates provide zero value.

3. **Don't ignore execution plans** — Creating an index doesn't guarantee the query optimizer will use it. Always verify with EXPLAIN.

4. **Don't use functions on indexed columns in WHERE clauses** — `WHERE YEAR(created_at) = 2026` prevents index usage. Use `WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'` instead, or create a computed/generated column.

5. **Don't index columns that change frequently** — High-churn columns cause constant index updates. Evaluate whether the read benefit outweighs the write cost.

6. **Don't forget foreign keys** — Foreign key columns used in JOINs almost always benefit from indexes. Most databases don't auto-index foreign keys.

7. **Don't create indexes during peak hours** — Index creation can lock tables or consume significant resources. Schedule during maintenance windows.

### Anti-Patterns

**One-size-fits-all indexing**: Applying the same index strategy to all tables ignores workload differences. OLTP systems (many small transactions) need different indexes than OLAP systems (large analytical queries).

**Ignoring write workload**: If your application is write-heavy, every index is expensive. Be more conservative. If read-heavy, you can afford more indexes.

**Composite indexes in wrong order**: Column order in composite indexes is critical. (A, B) ≠ (B, A) in terms of query coverage. Order matters.

**Indexing low-selectivity columns alone**: An index on `is_active` (true/false) in a table where 95% of rows are true provides minimal benefit.

## Verification Checklist

Before deploying an index:

- [ ] Documented which queries this index serves
- [ ] Verified with execution plan that the index will be used
- [ ] Confirmed no existing index already covers this use case
- [ ] Evaluated write performance impact
- [ ] Composite index follows ESR rule (if applicable)
- [ ] Cardinality is sufficient to justify the index
- [ ] Index name is descriptive and follows convention
- [ ] Tested in staging environment with production-like data volume

## NoSQL Considerations

### MongoDB

- Compound indexes follow similar left-prefix rules
- Use ESR ordering (equality, sort, range)
- Limit compound indexes to 3-4 fields
- Monitor index size with `db.collection.stats()`
- Use covered queries where possible

### DynamoDB

- Partition key selection is critical for distribution
- Sort keys enable range queries
- Global Secondary Indexes (GSI) and Local Secondary Indexes (LSI) have different tradeoffs
- Sparse indexes automatically exclude items without the indexed attribute

### Cassandra

- Primary key = partition key + clustering columns
- Queries must include partition key
- Secondary indexes are expensive; prefer denormalization
- Materialized views for different query patterns

Each NoSQL system has unique indexing semantics, but the core principles remain: index to serve queries, avoid bloat, measure impact, and maintain regularly.