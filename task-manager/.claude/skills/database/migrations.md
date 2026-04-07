---
name: safe-migration-design
description: >
  Design and execute safe database and schema migrations with forward/backward
  compatibility, data transformation, rollback scripts, and zero-downtime strategies.
  Use this whenever the user mentions database migrations, schema changes, deployments
  without downtime, migration scripts, rollback plans, backward compatibility,
  expand-contract patterns, data migrations, blue-green deployments for databases,
  dual-write strategies, migration safety, avoiding breaking changes, or any scenario
  involving evolving database schemas in production while maintaining service availability.
---

# Safe Migration Design

Migrations are production incidents waiting to happen. The core challenge: evolving database schemas while both old and new application versions run simultaneously. This skill focuses on designing migrations that are reversible, observable, and safe under live production load.

## Core Principle: Backward Compatibility

Every migration must support **both the previous and current application version** running concurrently. This is non-negotiable for zero-downtime deployments.

Why this matters: During rolling deployments, half your instances run v1, half run v2. A migration that breaks v1 causes immediate production outages. Reversibility is what turns deployments into controlled experiments rather than irreversible events.

## The Expand-Migrate-Contract Pattern

This is the fundamental technique for non-breaking changes. Split every breaking change into three backward-compatible phases:

### Phase 1: Expand (Add New Structure)

Add new columns, tables, or indexes alongside existing ones. The schema supports both old and new application code.

```sql
-- Safe: adds new column, doesn't touch existing
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);
CREATE INDEX CONCURRENTLY idx_users_full_name ON users(full_name);
```

Deploy this migration **before** any application code changes. Old code ignores the new column. Zero risk.

### Phase 2: Migrate (Dual-Write + Backfill)

Deploy application code that writes to **both** old and new structures. This ensures new schema receives data while maintaining backward compatibility.

```python
# Application writes to both columns
def update_user(user_id, name):
    db.execute(
        "UPDATE users SET name = %s, full_name = %s WHERE id = %s",
        (name, name, user_id)  # Dual-write
    )
```

Then backfill historical data:

```sql
-- Backfill in batches to avoid locks
UPDATE users 
SET full_name = name 
WHERE full_name IS NULL 
LIMIT 10000;
```

Run backfill during off-peak hours or use batched updates. At this point, you can still rollback application code safely—both columns exist.

### Phase 3: Contract (Remove Old Structure)

Once all application instances read from the new schema and data is fully migrated, remove the old structure.

```sql
-- Deploy app code that only uses full_name first
-- Then run this migration
ALTER TABLE users DROP COLUMN name;
```

This takes **three deployments** instead of one. The cost is complexity. The benefit is zero downtime and safe rollback at every step.

## Safe vs. Breaking Changes

### Always Safe
- Adding nullable columns (with or without defaults)
- Adding new tables
- Adding indexes with `CREATE INDEX CONCURRENTLY` (PostgreSQL) or `ONLINE` (MySQL 5.6+)
- Creating views
- Adding optional fields to JSON/JSONB columns

### Requires Expand-Migrate-Contract
- Renaming columns or tables
- Changing column types
- Adding NOT NULL constraints
- Removing columns or tables
- Changing foreign key relationships
- Splitting or merging tables

### Never Do in One Step
- Direct column renames: `ALTER TABLE users RENAME COLUMN name TO full_name` (breaks v1 instantly)
- Dropping columns still in use
- Changing types without compatibility layer

## Zero-Downtime Deployment Strategies

### Rolling Deployments

Replace instances incrementally. Simple but requires strict backward compatibility.

**Migration timing:**
1. Run expand migrations **before** deploying new code
2. Deploy application with dual-write logic
3. Verify data migration complete
4. Deploy application that uses only new schema
5. Run contract migrations to clean up old structures

### Blue-Green with Databases

Two identical environments, instant traffic switch. Works well for infrastructure changes.

**Challenge:** Databases are stateful. You can't simply clone production data without synchronization.

**Solution:** Use dual-write pattern during transition:
1. Blue environment runs old schema
2. Deploy green environment with new schema
3. Application writes to **both** databases (or use database-level replication with schema transformation)
4. Validate green environment with subset of traffic
5. Switch all traffic to green
6. Decommission blue after validation period

### Canary Deployments for Migrations

Gradually increase traffic to new version (1% → 10% → 50% → 100%).

**Migration approach:**
- Expand phase: deploy to all instances
- Migrate phase: canary rollout with dual-write
- Monitor error rates, latency, data integrity at each stage
- Contract phase: only after 100% rollout validated

Best for catching issues affecting specific user segments before full rollout.

## Data Transformation Patterns

### In-Place Transformation

Transform data within the same table. Use for simple changes.

```sql
-- Batch updates to avoid long locks
UPDATE orders 
SET status = CASE 
    WHEN legacy_status = 'P' THEN 'pending'
    WHEN legacy_status = 'C' THEN 'completed'
    ELSE 'unknown'
END
WHERE id BETWEEN 1000000 AND 1100000;
```

Run in batches during off-peak. Monitor replication lag if using replicas.

### Dual-Write Transformation

Transform data at write time. Application handles conversion logic.

```python
def create_order(data):
    # Write transformed data to new column
    transformed = transform_order_data(data)
    db.execute(
        "INSERT INTO orders (legacy_data, new_data) VALUES (%s, %s)",
        (data, transformed)
    )
```

Benefit: No bulk backfill needed for new data. Backfill only handles historical records.

### Separate Transformation Jobs

For complex transformations, run dedicated migration jobs.

```python
# Migration script with progress tracking
def migrate_user_data(batch_size=1000):
    last_id = get_last_migrated_id()
    while True:
        batch = fetch_users(last_id, batch_size)
        if not batch:
            break
        
        for user in batch:
            transformed = complex_transformation(user)
            update_user_new_schema(user.id, transformed)
        
        last_id = batch[-1].id
        record_progress(last_id)
        sleep(0.1)  # Throttle to avoid overwhelming DB
```

Include progress tracking, error handling, and ability to resume. Test on production-like data volumes.

## Rollback Scripts

### Forward and Backward Migrations

Every migration should have a reverse operation.

```sql
-- Forward migration (up)
ALTER TABLE users ADD COLUMN email_verified BOOLEAN DEFAULT FALSE;

-- Reverse migration (down)
ALTER TABLE users DROP COLUMN email_verified;
```

Most frameworks (Flyway, Liquibase, Alembic, Django migrations) support this pattern.

### Rollback Safety Rules

1. **Additive changes are safe to rollback** (just drop the addition)
2. **Destructive changes require data preservation**
   - Before dropping a column, export data if rollback might be needed
   - Keep backups for at least one deployment cycle
3. **Test rollback in staging** with production-like data
4. **Document rollback procedures** for each migration phase

### When Rollback Isn't Possible

Some migrations can't be reversed without data loss:
- Dropping columns with unique data
- Type changes that lose precision (INTEGER → BOOLEAN)
- Complex data transformations

For these:
- Take database snapshots before migration
- Test extensively in staging
- Plan for forward fixes rather than rollback
- Consider feature flags to disable new code paths without reverting schema

## Change Data Capture (CDC) for Zero Downtime

CDC replicates changes in real-time, enabling live migrations.

**Pattern:**
1. Initial bulk copy: snapshot source database to target
2. CDC tool captures ongoing changes (inserts, updates, deletes)
3. Apply changes to target in real-time
4. Both databases stay synchronized
5. Switch application traffic when ready
6. Decommission old database

**Tools:** Debezium, AWS DMS, Google Datastream, Striim

**Use when:** Migrating between database engines, cloud migrations, or any scenario requiring prolonged dual-operation.

## Testing Migrations

### Pre-Production Validation

1. **Schema compatibility tests:** Verify v1 code works with expanded schema
2. **Data integrity tests:** Check constraints, foreign keys, data types after migration
3. **Performance tests:** Measure query performance on new schema with production-like data volumes
4. **Rollback tests:** Execute down migration and verify application still works

### Production Monitoring

During migration:
- Error rates (failed queries, constraint violations)
- Latency (query performance, replication lag)
- Data consistency (row counts, checksums between old/new structures)
- Application metrics (request failures, timeouts)

Set up alerts for anomalies. Have rollback plan ready.

## Common Pitfalls

### Long-Running Migrations

Adding indexes or altering large tables can lock tables for minutes or hours.

**Solution:**
- Use `CREATE INDEX CONCURRENTLY` (PostgreSQL) or `ALGORITHM=INPLACE, LOCK=NONE` (MySQL)
- For massive tables, consider pt-online-schema-change (Percona Toolkit) or gh-ost (GitHub)
- Schedule during maintenance windows if truly unavoidable

### Forgetting Application Compatibility

Migration succeeds but application code expects old schema.

**Solution:**
- Deploy schema changes **before** application code for additive changes
- Deploy application code **before** schema cleanup for removals
- Use feature flags to decouple code deployment from schema usage

### Insufficient Backfill Testing

Backfill works in staging but times out or locks production.

**Solution:**
- Test with production data volumes
- Batch updates (1000-10000 rows at a time)
- Monitor replication lag and throttle if needed
- Run during off-peak hours

### Ignoring Replication Lag

Migration completes on primary but replicas lag behind, causing read inconsistencies.

**Solution:**
- Monitor replication lag during migration
- Throttle batch operations if lag exceeds threshold
- Consider temporarily routing reads to primary for critical operations

## Migration Workflow Checklist

**Planning:**
- [ ] Identify breaking vs. safe changes
- [ ] Design expand-migrate-contract phases if needed
- [ ] Write forward and backward migration scripts
- [ ] Document rollback procedure
- [ ] Estimate migration duration and lock times

**Pre-Deployment:**
- [ ] Test migration on production-like dataset
- [ ] Test rollback procedure
- [ ] Verify application compatibility with expanded schema
- [ ] Set up monitoring and alerts
- [ ] Schedule migration (off-peak if needed)

**Execution:**
- [ ] Take database snapshot/backup
- [ ] Run expand migration
- [ ] Deploy dual-write application code
- [ ] Execute data backfill (if needed)
- [ ] Verify data integrity and consistency
- [ ] Deploy application using new schema
- [ ] Monitor error rates and performance
- [ ] Run contract migration after validation period

**Post-Migration:**
- [ ] Verify all application instances using new schema
- [ ] Monitor for 24-48 hours
- [ ] Document lessons learned
- [ ] Remove old migration code and feature flags

## Framework-Specific Notes

**Django:** Migrations auto-generate but review for safety. Use `RunPython` for data migrations. `--fake` flag for manual schema changes.

**Rails:** `change` method for reversible migrations. Use `up`/`down` for complex logic. `disable_ddl_transaction!` for concurrent indexes.

**Flyway/Liquibase:** Version-based migrations. Use `beforeMigrate`/`afterMigrate` callbacks. Checksum validation prevents accidental changes.

**Alembic (Python):** `upgrade`/`downgrade` functions. `batch_alter_table` for SQLite compatibility. Autogenerate with manual review.

Regardless of framework: **always review auto-generated migrations** for backward compatibility before deploying.

## When to Accept Downtime

Sometimes zero-downtime isn't worth the complexity:
- Low-traffic applications with flexible SLAs
- Migrations requiring fundamental restructuring (rare)
- Startup/development environments
- Cost of dual-write complexity exceeds cost of brief downtime

If accepting downtime:
- Schedule maintenance window
- Notify users in advance
- Test migration thoroughly
- Have rollback plan ready
- Monitor closely during and after

But for production systems with availability requirements, invest in backward-compatible migrations. The discipline pays dividends in operational stability.