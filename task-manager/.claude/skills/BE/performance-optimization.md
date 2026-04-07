---
name: backend-performance-optimization
description: >
  Optimize backend systems for speed, scalability, and reliability. Use this skill when working on API performance, database bottlenecks, slow queries, high latency, caching strategies (Redis, Memcached), connection pooling, query optimization, indexing, asynchronous processing, background jobs, batching, rate limiting, monitoring performance metrics (P50, P95, P99 latency, throughput), reducing response times, improving database performance, implementing caching layers, optimizing SQL queries, profiling backend code, or any task involving backend speed improvements, scalability tuning, or performance analysis. Apply this whenever performance, latency, throughput, or resource efficiency is mentioned.
---

# Backend Performance Optimization

## Overview

Backend performance optimization transforms slow, resource-heavy systems into fast, efficient engines. The core principle: eliminate redundant work. Every query to the database, every computation repeated, every blocking operation costs time and resources. Performance optimization identifies where systems waste effort and applies targeted strategies to eliminate bottlenecks.

This skill covers the essential patterns that matter in production: caching to avoid repeated queries, query optimization to reduce database load, connection pooling to reuse resources, async processing to prevent blocking, batching to reduce overhead, and monitoring to measure what actually matters.

## Decision Tree: Where to Start

1. **Measure first** — Profile the system to identify actual bottlenecks. Guessing wastes time.
2. **Database slow?** → Query optimization + indexing + caching
3. **API endpoints slow?** → Async processing + caching + batching
4. **High resource usage?** → Connection pooling + async jobs + rate limiting
5. **Unknown bottleneck?** → Add monitoring and metrics first

Always optimize the biggest bottleneck first. A 10x improvement on the slowest component beats ten 10% improvements elsewhere.

## Caching Strategies

Caching stores frequently accessed data in fast storage (memory) to avoid slow operations (database queries, API calls, computations). Memory access is ~100,000x faster than disk.

### When to Cache

- Read-heavy workloads where the same data is requested repeatedly
- Expensive computations that produce the same result for the same input
- Data that changes infrequently (user profiles, product catalogs, configuration)
- Protection against traffic spikes that would overwhelm the database

### Cache-Aside Pattern (Most Common)

The application manages the cache. On read: check cache first, if miss then query database and populate cache. On write: update database, then invalidate or update cache.

```python
def get_user(user_id):
    cache_key = f"user:{user_id}"
    
    # Check cache first
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)  # Cache hit
    
    # Cache miss: query database
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # Store in cache with TTL
    redis.setex(cache_key, 300, json.dumps(user))  # 5 min TTL
    return user
```

### Write-Through vs Write-Behind

**Write-through**: Write to cache and database simultaneously. Ensures consistency but adds write latency. Use when consistency is critical.

**Write-behind**: Write to cache immediately, flush to database asynchronously. Fastest writes but risks data loss if cache fails. Use for high-write workloads where eventual consistency is acceptable.

### Cache Invalidation

The hardest problem in caching. Three approaches:

- **TTL-based**: Set expiration time. Accept that data may be stale for up to TTL duration. Simplest and works for most cases.
- **Event-driven**: Invalidate cache when underlying data changes. More complex but provides fresher data.
- **Version-based**: Include version number in cache key. When data changes, increment version. Old entries expire naturally.

### Redis vs Memcached

**Redis**: Rich data structures (lists, sets, sorted sets), persistence options, pub/sub, atomic operations. Use when you need more than simple key-value.

**Memcached**: Simple key-value only, no persistence, slightly lower latency for basic operations. Use for pure ephemeral caching.

Default to Redis unless you have a specific reason for Memcached's simplicity.

## Query Optimization

Database queries are often the biggest bottleneck. Optimizing queries can yield 10-100x improvements.

### Core Principles

**Fetch only what you need**: Avoid `SELECT *`. Specify columns. Fewer columns = less data transfer, less memory.

**Filter early**: Push WHERE clauses as close to the data as possible. Reduce rows before joins or aggregations.

**Index wisely**: Indexes speed up reads but slow down writes. Index columns used in WHERE, JOIN, ORDER BY clauses.

### Indexing Strategy

**B-tree indexes** (default): Fast for equality and range queries. Use for most cases.

**Covering indexes**: Include all columns needed by a query in the index. Database can satisfy query from index alone without touching the table.

**Composite indexes**: Index multiple columns together. Order matters—put highest-selectivity column first (the column that filters out the most rows).

**Partial indexes**: Index only a subset of rows (e.g., WHERE active = true). Smaller index, faster queries on that subset.

```sql
-- Before: Full table scan
SELECT name, email FROM users WHERE status = 'active';

-- After: Create index on status, include name and email
CREATE INDEX idx_active_users ON users(status) INCLUDE (name, email) WHERE status = 'active';
```

### Query Patterns to Avoid

**N+1 queries**: Fetching related data in a loop. One query to get parent records, then N queries for children. Solution: Use JOIN or batch fetch.

**Unbounded queries**: No LIMIT clause. Can return millions of rows. Always paginate.

**Functions on indexed columns**: `WHERE YEAR(created_at) = 2024` prevents index use. Rewrite as range: `WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'`.

### Analyze Query Plans

Use `EXPLAIN` or `EXPLAIN ANALYZE` to see how the database executes a query. Look for:

- **Seq Scan / Table Scan**: Reading entire table. Usually bad. Add index.
- **Index Scan**: Using index. Good.
- **Nested Loop**: Joining rows one by one. Can be slow for large datasets. Consider hash join.
- **High cost estimates**: Indicates expensive operation.

## Connection Pooling

Opening a new database connection for every request is expensive (TCP handshake, authentication, resource allocation). Connection pooling maintains a pool of open connections that can be reused.

**Pool size**: Too small = requests wait for connections. Too large = database overwhelmed. Start with: `pool_size = (core_count * 2) + effective_spindle_count`. For cloud databases, 10-20 connections per application instance is typical.

**Idle timeout**: Close connections idle for too long to free resources.

**Max lifetime**: Recycle connections periodically to avoid stale connections or memory leaks.

```python
# Example with SQLAlchemy
engine = create_engine(
    'postgresql://...',
    pool_size=10,           # Base pool size
    max_overflow=20,        # Additional connections under load
    pool_timeout=30,        # Wait time for connection
    pool_recycle=3600       # Recycle after 1 hour
)
```

## Asynchronous Processing

Blocking operations (file I/O, external API calls, email sending) waste server resources waiting. Async processing prevents blocking the main request thread.

### When to Use Async

- Long-running tasks that don't need immediate response (report generation, email sending)
- External API calls with unpredictable latency
- I/O-heavy operations (file uploads, image processing)
- Tasks that can be retried on failure

### Background Jobs

Offload work to background workers using job queues (Celery, Sidekiq, Bull, AWS SQS).

```python
# Synchronous: User waits for email to send
def signup(user_data):
    user = create_user(user_data)
    send_welcome_email(user)  # Blocks for 2-3 seconds
    return user

# Asynchronous: Email sent in background
def signup(user_data):
    user = create_user(user_data)
    send_welcome_email.delay(user.id)  # Returns immediately
    return user
```

### Async I/O

For I/O-bound operations within a single request, use async/await to handle multiple operations concurrently.

```python
# Synchronous: 3 seconds total
def get_dashboard_data(user_id):
    user = fetch_user(user_id)        # 1s
    orders = fetch_orders(user_id)    # 1s
    stats = fetch_stats(user_id)      # 1s
    return {"user": user, "orders": orders, "stats": stats}

# Asynchronous: ~1 second total (parallel)
async def get_dashboard_data(user_id):
    user, orders, stats = await asyncio.gather(
        fetch_user(user_id),
        fetch_orders(user_id),
        fetch_stats(user_id)
    )
    return {"user": user, "orders": orders, "stats": stats}
```

## Batching

Batching combines multiple small operations into fewer large ones, reducing overhead from network round-trips, transaction commits, and function calls.

**Database inserts**: Batch 100-1000 rows per INSERT instead of one at a time. Use multi-row INSERT or bulk import tools.

**API calls**: Batch multiple requests into one if the API supports it. Reduce HTTP overhead.

**Cache operations**: Use MGET/MSET in Redis to fetch/set multiple keys in one round-trip.

```python
# Inefficient: N round-trips
for user_id in user_ids:
    user = redis.get(f"user:{user_id}")

# Efficient: 1 round-trip
keys = [f"user:{uid}" for uid in user_ids]
users = redis.mget(keys)
```

## Monitoring and Metrics

You can't optimize what you don't measure. Track metrics that reveal user-facing performance and system health.

### Key Metrics

**Latency percentiles**: P50 (median), P95, P99. P99 shows worst-case experience. A P99 of 500ms means 99% of requests complete in under 500ms.

**Throughput**: Requests per second. Measures capacity.

**Error rate**: Percentage of failed requests. Spikes indicate problems.

**Database query time**: Slow query log. Identify expensive queries.

**Cache hit rate**: Percentage of cache hits. Low hit rate = cache not effective.

**Resource utilization**: CPU, memory, disk I/O, network. High utilization indicates capacity limits.

### Why P99 Matters More Than Average

Average latency hides outliers. If 99% of requests take 100ms but 1% take 10 seconds, average might be 200ms—looks fine, but 1 in 100 users has a terrible experience. P99 reveals this.

### Monitoring Tools

- **APM tools**: Datadog, New Relic, Sentry — track request traces, slow queries, errors
- **Database monitoring**: pg_stat_statements (Postgres), slow query log (MySQL)
- **Profiling**: py-spy (Python), pprof (Go), perf (Linux) — identify hot code paths

### Alerting Thresholds

Set alerts on P99 latency, error rate, and resource usage. Example thresholds:

- P99 latency > 1 second
- Error rate > 1%
- CPU usage > 80% for 5 minutes
- Cache hit rate < 70%

## Common Patterns

### Thundering Herd Prevention

When a popular cache entry expires, many requests simultaneously try to regenerate it, all hitting the database. Use cache stampede protection: first request acquires a lock, regenerates the value, others wait for the result.

### Rate Limiting

Protect backend from abuse and overload. Limit requests per user/IP using token bucket or sliding window algorithms. Redis is ideal for distributed rate limiting.

### Denormalization

Store redundant data to avoid expensive JOINs. Trades write complexity and storage for read speed. Use when read performance is critical and data changes infrequently.

### Read Replicas

Separate read and write workloads. Write to primary database, read from replicas. Reduces load on primary, scales read capacity horizontally.

## Performance Optimization Workflow

1. **Measure**: Use APM and profiling to identify the slowest components
2. **Prioritize**: Focus on the biggest bottleneck (highest impact)
3. **Optimize**: Apply targeted strategy (caching, indexing, async, etc.)
4. **Measure again**: Verify improvement with metrics
5. **Repeat**: Move to next bottleneck

Avoid premature optimization. Optimize based on data, not assumptions. A 10x improvement on a component that takes 1% of total time only improves overall performance by 10%. Focus on what matters.
