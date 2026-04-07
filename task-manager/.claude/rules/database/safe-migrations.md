---
name: safe-database-migrations
description: >
  Guide database schema changes with zero-downtime requirements, backwards
  compatibility patterns, rollback safety, and data verification. Use this for
  any database migration, schema change, ALTER TABLE, column rename, index addition,
  table modification, production deployment involving databases, or when discussing
  migration safety, deployment risk, breaking changes, downtime prevention, rollback
  strategies, or data integrity during schema updates. Applies to SQL databases
  (Postgres, MySQL, SQL Server), NoSQL migrations, cloud database changes, microservice
  database updates, and any scenario where schema and code must evolve together
  without service interruption.
---

# Safe Database Migrations

## Core Principle

The fundamental challenge: application code deploys in seconds, but database migrations can take minutes to hours. During that window, old and new code may run against a schema in flux. Every migration must be backward compatible with currently running application code.

**Why this matters:** Schema changes are stateful and expensive to reverse. A failed migration can corrupt data, lock tables for hours, or break production. The goal is to make schema evolution routine rather than risky.

## The Expand-Contract Pattern

The safest approach for complex schema changes. Break dangerous migrations into safe, incremental steps across multiple deployments.

### Three-Phase Process

1. **Expand** (Deployment N-1): Add new structures alongside old ones
   - Add new columns, tables, or indexes
   - Both old and new code can operate
   - Database supports both versions simultaneously

2. **Migrate** (Deployment N): Transition to new structures
   - Deploy code that writes to both old and new structures
   - Backfill data from old to new structures
   - Verify data integrity and performance
   - Switch reads to new structures

3. **Contract** (Deployment N+1): Remove old structures
   - Deploy code that only uses new structures
   - Drop old columns, tables, or indexes
   - Clean up migration code

**Why three deployments:** Each step is independently reversible. If issues arise, roll back the application—the database remains compatible.

## Safe vs. Dangerous Operations

### Always Safe (Additive Changes)

- Add nullable column with default value
- Add new table not yet referenced
- Add index with CONCURRENTLY (Postgres) or online DDL (MySQL)
- Create view or stored procedure
- Add enum value at end of list

These can be deployed before application code. Old code ignores new structures.

### Requires Expand-Contract (Breaking Changes)

- Rename column or table
- Change column type
- Add NOT NULL constraint
- Remove column or table
- Split column into multiple columns
- Merge tables

**Why these are dangerous:** Old code expects structures that no longer exist or behave differently. A direct change causes runtime errors.

### Example: Renaming a Column

**Bad approach (causes downtime):**
```sql
ALTER TABLE users RENAME COLUMN last_name TO surname;
```
Old code breaks immediately—it references `last_name`.

**Safe approach:**

*Deployment 1 (Expand):*
```sql
ALTER TABLE users ADD COLUMN surname VARCHAR(255);
UPDATE users SET surname = last_name WHERE surname IS NULL;
```
Code writes to both columns. Old code still works.

*Deployment 2 (Migrate):*
Deploy code that reads from `surname`, writes to both. Verify in production.

*Deployment 3 (Contract):*
```sql
ALTER TABLE users DROP COLUMN last_name;
```
Remove old column and dual-write logic.

## Rollback Strategy

### Prefer Rolling Forward

Database rollbacks are expensive and risky. Rolling back schema changes means physically altering all rows. Instead:

- **Fix forward:** Deploy a new migration to correct the issue
- **Feature flags:** Hide new behavior until verified safe
- **Backwards compatibility:** Design so old code continues working

**Why avoid rollback:** Reversing a migration on a large table can take hours and may lose data written by new code.

### When Rollback is Necessary

- Test rollback scripts in staging against production-sized data
- Measure rollback duration—if it exceeds acceptable downtime, reconsider approach
- Never delete data immediately—rename columns/tables with deprecation markers (e.g., `zzz_deprecated_last_name_20260407`)
- Keep deprecated structures for 2-3 deployment cycles before dropping

### Rollback-Safe Practices

- Deploy schema changes separately from feature releases
- Use feature flags to decouple code deployment from feature activation
- Always have a backout plan documented before production deployment
- Maintain backwards compatibility for at least one version

## Data Verification

### Pre-Migration Validation

- **Schema analysis:** Identify dependencies (foreign keys, indexes, views, stored procedures)
- **Data profiling:** Understand volume, distribution, and edge cases
- **Performance testing:** Run migration against production-sized dataset in staging
- **Lock analysis:** Identify operations that lock tables (avoid during peak hours)

### During Migration

- **Dual writes:** Write to both old and new structures during transition
- **Consistency checks:** Verify old and new data match during backfill
- **Monitoring:** Track migration progress, lock wait times, replication lag

### Post-Migration Validation

- **Row counts:** Verify source and target match
- **Data sampling:** Compare random samples between old and new structures
- **Application health:** Monitor error rates, latency, and query performance
- **Rollback readiness:** Confirm ability to revert before dropping old structures

## Zero-Downtime Techniques

### Change Data Capture (CDC)

For large-scale migrations, use CDC tools to replicate changes in real-time:

1. Perform initial bulk copy
2. CDC continuously captures inserts/updates/deletes
3. Keep old and new systems in sync
4. Switch traffic with near-zero interruption

**Use when:** Migrating between database engines, moving to cloud, or restructuring large tables.

### Online Schema Change Tools

- **Postgres:** Use `CONCURRENTLY` for index creation, `pg_repack` for table rewrites
- **MySQL:** `pt-online-schema-change` (Percona), `gh-ost` (GitHub)
- **SQL Server:** Online index operations (Enterprise edition)

These tools avoid long-running locks by creating shadow tables and replaying changes.

### Blue-Green Database Pattern

Maintain two identical database environments:

1. Apply migration to inactive (green) environment
2. Replicate data from active (blue) to green
3. Test application against green
4. Switch traffic from blue to green
5. Keep blue as instant rollback option

**Trade-off:** Doubles infrastructure cost, but provides instant rollback.

## Prohibited Destructive Changes

### Never Do These Directly

- **DROP COLUMN** on active table—old code breaks immediately
- **RENAME COLUMN/TABLE** without backwards-compatible transition
- **ALTER COLUMN TYPE** that narrows precision (e.g., VARCHAR(100) → VARCHAR(50))
- **ADD NOT NULL** without default or backfill
- **DROP TABLE** still referenced by any code version

### Instead

- Mark for deprecation, remove in later deployment
- Use expand-contract pattern
- Add new column with new type, migrate data, drop old column
- Backfill data first, then add constraint
- Verify no references in all deployed code versions

## Migration Deployment Pipeline

### Recommended Order

1. **Pre-deployment migrations** (expand phase): Add new structures
2. **Deploy application code**: Works with both old and new schema
3. **Verify deployment**: Confirm application health
4. **Post-deployment migrations** (contract phase): Remove old structures

### Separation of Concerns

- Decouple migration execution from application startup
- Run migrations as separate CI/CD step with dedicated monitoring
- Use migration tools (Flyway, Liquibase, Alembic) for version control
- Never auto-run migrations on app boot in production

**Why separate:** Gives control over timing. If migration takes 20 minutes, application deployment isn't blocked.

## Version Compatibility Matrix

### Three-Version Strategy

Design each schema change considering three application versions:

- **vN-2:** May still be running during rollout
- **vN-1:** Prepares for new schema (reads new structures, writes to both)
- **vN:** Uses new schema exclusively

**Backwards compatibility:** vN-1 must read data written by vN
**Forwards compatibility:** vN-1 must not break when vN writes data

### Coordination Pattern

1. vN-1 deploys: Can read new schema, writes to old (or both)
2. Migration runs: Adds new structures, backfills data
3. vN deploys: Reads and writes to new schema
4. vN+1 deploys: Removes old schema support

## Testing Requirements

### Staging Must Mirror Production

- Same database engine version
- Production-sized dataset (not a 100-row sample)
- Realistic query load during migration
- Test on copy of production data, not synthetic data

### What to Test

- Migration duration and resource usage
- Lock contention and replication lag
- Application behavior with both old and new schema
- Rollback procedure end-to-end
- Edge cases: null values, large text fields, concurrent updates

### Load Testing During Migration

Run application load tests while migration executes. This reveals:

- Query performance degradation
- Lock wait timeouts
- Replication lag spikes
- Memory or disk pressure

## Common Pitfalls

**Underestimating migration time:** A change that takes 5 seconds on 1000 rows may take 3 hours on 100M rows.

**Ignoring indexes:** Adding an index locks the table unless using CONCURRENTLY. Dropping an index is fast but can devastate query performance.

**Forgetting replication lag:** Migration completes on primary, but replicas lag by 30 minutes. Reads hit stale schema.

**Tight coupling:** Deploying schema change and code change together. If either fails, both must roll back.

**Inadequate monitoring:** Migration runs, appears to succeed, but silently corrupted data. Verify with row counts and sampling.

**Assuming small tables are safe:** Even small tables can cause issues if frequently locked or part of complex joins.

## Framework-Specific Guidance

### Django

- Use `RunPython` for data migrations separate from schema migrations
- Set `db_index=True` with caution—generates CONCURRENTLY on Postgres if configured
- Avoid `null=False` without `default` on existing tables

### Rails

- Use `safety_assured` only when you've verified safety
- Strong Migrations gem catches dangerous operations
- Use `add_column` with `default: value` carefully (pre-5.0 locks table)

### Flyway/Liquibase

- Separate versioned migrations (expand) from repeatable migrations (contract)
- Use `validate` before `migrate` in CI
- Tag migrations with deployment version for tracking

## Decision Framework

**Can old code tolerate new schema?** → Deploy schema first, then code

**Can new code tolerate old schema?** → Deploy code first, then schema

**Neither?** → Use expand-contract pattern across multiple deployments

**Is table small (<10k rows) and low-traffic?** → Direct change may be acceptable

**Is table large or high-traffic?** → Use online schema change tools or expand-contract

**Is rollback critical?** → Maintain backwards compatibility for 2+ versions

**Is data loss unacceptable?** → Never drop columns directly—deprecate first
