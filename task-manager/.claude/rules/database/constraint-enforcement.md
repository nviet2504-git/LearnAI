---
name: database-constraint-enforcement
description: >
  Enforce data integrity through database constraints including primary keys,
  foreign keys, unique constraints, check constraints, and not-null rules. Use this
  for database schema design, data modeling, table creation, migration writing, or whenever
  designing tables, defining columns, establishing relationships, or ensuring data validity.
  Applies to discussions about referential integrity, data quality, constraint strategy,
  when to use DB constraints vs application validation, constraint performance tradeoffs,
  and best practices for relational database design in PostgreSQL, MySQL, SQL Server,
  or any SQL database.
---

# Database Constraint Enforcement

## Overview

Database constraints are rules enforced at the database level that guarantee data integrity regardless of how data enters the system—through application code, direct SQL, migrations, imports, or third-party integrations. Constraints prevent invalid data from being persisted and serve as the last line of defense against corruption.

This skill focuses on when and how to use each constraint type, balancing data integrity with performance, and understanding the strategic difference between database-level enforcement and application-level validation.

## Core Constraint Types

### Primary Keys

**Purpose:** Uniquely identify each row. Foundation for relationships and data traceability.

**Rules:**
- Every table needs one (with rare exceptions like pure junction tables)
- Must be NOT NULL and UNIQUE by definition
- Immutable—never change a primary key value after creation

**Implementation patterns:**

```sql
-- Auto-incrementing integer (most common, best join performance)
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  email TEXT NOT NULL
);

-- UUID (distributed systems, no sequential exposure)
CREATE TABLE events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_type TEXT NOT NULL
);

-- Composite key (junction tables, natural uniqueness)
CREATE TABLE user_roles (
  user_id BIGINT NOT NULL,
  role_id BIGINT NOT NULL,
  PRIMARY KEY (user_id, role_id)
);
```

**When to use surrogate vs natural keys:**
- **Surrogate (auto-increment/UUID):** Default choice. Stable, no business meaning, efficient joins
- **Natural (email, username):** Only when truly immutable and universally unique. Rare in practice

### Foreign Keys

**Purpose:** Enforce referential integrity between tables. Prevent orphaned records.

**When mandatory:**
- Parent-child relationships where orphans are logically invalid (orders → customers)
- Data must remain consistent across tables
- Single-database OLTP systems with moderate write volume

**When to reconsider:**
- High-concurrency write workloads (foreign keys add lock contention)
- Distributed systems where parent/child live in different databases
- Data warehouses or analytical systems (enforce in ETL instead)
- Very large bulk imports (disable during load, re-enable with CHECK after)

**Implementation:**

```sql
CREATE TABLE orders (
  id BIGSERIAL PRIMARY KEY,
  customer_id BIGINT NOT NULL,
  order_date DATE NOT NULL,
  FOREIGN KEY (customer_id) REFERENCES customers(id)
    ON DELETE RESTRICT  -- Prevent deletion if orders exist
    ON UPDATE CASCADE   -- Propagate ID changes (rare)
);
```

**Cascading actions:**
- `RESTRICT` (default): Block parent deletion if children exist. Safest choice
- `CASCADE`: Auto-delete children when parent deleted. Use only for truly dependent data
- `SET NULL`: Nullify foreign key when parent deleted. Rare; requires nullable column
- `NO ACTION`: Like RESTRICT but deferred until transaction end

**Performance considerations:**
- Always index foreign key columns (databases don't auto-index them)
- Foreign keys add overhead to INSERT/UPDATE/DELETE on child tables
- Trusted constraints (created with CHECK) enable query optimizer improvements
- Untrusted constraints (created with NOCHECK or disabled) don't help performance

```sql
-- Index the foreign key for join performance and constraint checking
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
```

### Unique Constraints

**Purpose:** Enforce business-level uniqueness beyond the primary key.

**When mandatory:**
- Email addresses, usernames, external IDs—anything users expect to be unique
- Business keys that aren't the primary key (invoice numbers, SKUs)
- Composite uniqueness (one role per user per project)

**Why at database level:** Application-level checks have race conditions. Two requests can check "email doesn't exist" simultaneously and both insert. Database unique constraints are atomic.

```sql
-- Single-column uniqueness
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  email TEXT NOT NULL UNIQUE
);

-- Composite uniqueness
CREATE TABLE project_memberships (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL,
  project_id BIGINT NOT NULL,
  UNIQUE (user_id, project_id)
);

-- Partial unique index (conditional uniqueness)
CREATE UNIQUE INDEX idx_active_subscriptions 
  ON subscriptions(user_id) 
  WHERE status = 'active';
```

### NOT NULL Constraints

**Purpose:** Require a value. Prevent missing data.

**When mandatory:**
- Any column essential to the entity's identity or core function
- Foreign keys in required relationships (child must have parent)
- Timestamps like `created_at`

**When to allow NULL:**
- Truly optional data (middle name, phone number)
- Columns that will be populated later in a workflow
- Avoid NULL when you can use a sensible DEFAULT instead

**Performance benefit:** NOT NULL on foreign keys enables query optimizer to push limits below joins, significantly improving query performance.

```sql
CREATE TABLE employees (
  id BIGSERIAL PRIMARY KEY,
  first_name TEXT NOT NULL,
  last_name TEXT NOT NULL,
  middle_name TEXT,  -- Optional
  department_id BIGINT NOT NULL,  -- Required relationship
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Check Constraints

**Purpose:** Enforce domain-specific rules and business logic at the data layer.

**When to use:**
- Value ranges (age >= 18, price > 0, quantity >= 0)
- Enumerated values when you can't use a foreign key to a lookup table
- Cross-column validation (end_date > start_date)
- Format requirements (email contains '@', phone matches pattern)

**When NOT to use:**
- Complex business logic better suited to application layer
- Rules requiring joins or subqueries (most databases don't support this well)
- Frequently changing business rules (constraints require migrations to change)

```sql
CREATE TABLE products (
  id BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  price DECIMAL(10,2) NOT NULL CHECK (price > 0),
  discount_pct DECIMAL(5,2) CHECK (discount_pct BETWEEN 0 AND 100)
);

CREATE TABLE bookings (
  id BIGSERIAL PRIMARY KEY,
  start_date DATE NOT NULL,
  end_date DATE NOT NULL,
  CHECK (end_date > start_date)
);

-- Complex check with CASE expression
CREATE TABLE employees (
  id BIGSERIAL PRIMARY KEY,
  employment_type TEXT NOT NULL CHECK (employment_type IN ('FULL_TIME', 'PART_TIME', 'CONTRACT')),
  salary DECIMAL(10,2),
  hourly_rate DECIMAL(8,2),
  CHECK (
    CASE 
      WHEN employment_type = 'FULL_TIME' THEN salary > 0
      WHEN employment_type IN ('PART_TIME', 'CONTRACT') THEN hourly_rate > 0
      ELSE FALSE
    END
  )
);
```

## Constraints vs Application Validation

### Use Database Constraints When:
- The rule is universally true for your domain ("price must be positive")
- Data integrity is non-negotiable
- Multiple applications or services access the database
- You need protection against bugs, race conditions, or direct SQL access
- The rule is simple and stable

### Use Application Validation When:
- You need rich, user-friendly error messages
- The rule is context-specific to one workflow
- The rule requires external API calls or complex computation
- The rule changes frequently based on business logic
- You're validating presentation concerns (password confirmation match)

### Best Practice: Use Both

**Layer 1 (Application):** Validate early for great UX. Show helpful messages. Handle workflow-specific rules.

**Layer 2 (Database):** Enforce universal truths. Catch bugs and race conditions. Protect against all entry points.

Example: Email uniqueness
- Application checks uniqueness before form submission → friendly "Email already taken" message
- Database UNIQUE constraint → prevents race condition where two signups happen simultaneously

## Performance Considerations

### Foreign Keys
- Add overhead to writes (must verify parent exists)
- Require indexes on FK columns for acceptable performance
- Can cause lock contention in high-concurrency scenarios
- Trusted FKs enable optimizer to eliminate unnecessary joins
- For bulk loads: disable, load data, re-enable WITH CHECK

### Check Constraints
- Minimal overhead—evaluated only on INSERT/UPDATE
- Modern databases optimize constraint checking efficiently
- Avoid subqueries or complex expressions that execute slowly

### Unique Constraints
- Implemented as unique indexes—minimal overhead
- Actually improve read performance for queries filtering on unique columns

## Constraint Naming Convention

Name constraints explicitly for easier debugging and migrations:

```sql
CREATE TABLE orders (
  id BIGSERIAL PRIMARY KEY,
  customer_id BIGINT NOT NULL,
  total DECIMAL(10,2) NOT NULL,
  order_date DATE NOT NULL,
  ship_date DATE,
  
  CONSTRAINT fk_orders_customer 
    FOREIGN KEY (customer_id) REFERENCES customers(id),
  
  CONSTRAINT chk_orders_total_positive 
    CHECK (total > 0),
  
  CONSTRAINT chk_orders_ship_after_order 
    CHECK (ship_date IS NULL OR ship_date >= order_date)
);
```

Pattern: `{type}_{table}_{column(s)}_{description}`
- `fk_` for foreign keys
- `chk_` for check constraints
- `uq_` for unique constraints

## Common Mistakes to Avoid

1. **Not indexing foreign keys** → Slow joins and slow constraint verification
2. **Over-relying on application validation alone** → Race conditions and data corruption from bugs
3. **Using check constraints for frequently changing business rules** → Requires schema migrations
4. **Forgetting ON DELETE/ON UPDATE behavior** → Unexpected cascade deletes or blocked operations
5. **Creating untrusted constraints** → No optimizer benefit, still pay write-time cost
6. **Nullable foreign keys without thought** → Decide: is the relationship optional or required?

## Migration Strategy

Adding constraints to existing tables:

```sql
-- Check data first
SELECT COUNT(*) FROM orders WHERE customer_id IS NULL;
SELECT COUNT(*) FROM orders WHERE total <= 0;

-- Fix invalid data
UPDATE orders SET total = 0.01 WHERE total <= 0;

-- Add constraint
ALTER TABLE orders ADD CONSTRAINT chk_orders_total_positive CHECK (total > 0);

-- For large tables, add NOT VALID first, then validate separately (PostgreSQL)
ALTER TABLE orders ADD CONSTRAINT chk_orders_total_positive 
  CHECK (total > 0) NOT VALID;
-- Existing rows not checked, new rows are
ALTER TABLE orders VALIDATE CONSTRAINT chk_orders_total_positive;
-- Now validates existing rows without blocking writes
```

## When Constraints Are Not Enough

Some rules require triggers or application logic:
- Auditing (track who changed what when)
- Complex multi-table validations
- Calculations depending on aggregations
- Time-based rules ("can't modify orders older than 30 days")
- External system checks

Constraints enforce data-level integrity. Triggers and application code handle workflow-level complexity.