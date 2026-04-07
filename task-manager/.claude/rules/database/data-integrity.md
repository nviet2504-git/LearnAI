---
name: database-data-integrity
description: >
  Enforce database data integrity through constraints, normalization, and relationship rules.
  Use this when designing database schemas, defining table structures, setting up foreign keys,
  implementing referential integrity, normalizing data models, preventing data anomalies,
  configuring cascading behaviors, or ensuring data consistency and validity. Applies to schema
  design, migration planning, data modeling, integrity enforcement, constraint definition,
  relationship management, and preventing orphaned records or invalid states.
---

# Database Data Integrity

This skill enforces database integrity through constraints, normalization, and relationship rules that prevent invalid states at the database layer rather than relying solely on application logic.

## Core Principle

Integrity constraints work at the storage layer, catching errors from all entry points—application code, imports, APIs, direct database access, or migration scripts. This defense-in-depth approach prevents corruption regardless of how data enters the system.

**Why database-level enforcement matters:** Application logic can be bypassed, forgotten, or inconsistently implemented across services. Database constraints provide a single, reliable enforcement point that cannot be circumvented.

## Constraint Types

### Entity Integrity (PRIMARY KEY)

Ensures every row is uniquely identifiable. Without this, you cannot reliably update, delete, or reference specific records.

**Natural vs Surrogate Keys:**
- **Surrogate (recommended):** Auto-incrementing integers or UUIDs. Stable, never change, no cascade update complexity
- **Natural:** Real-world identifiers (email, SKU). Use only when truly immutable and universally unique

```sql
CREATE TABLE customers (
  customer_id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(100) NOT NULL
);
```

**Composite keys** combine multiple columns for uniqueness (common in junction tables):

```sql
CREATE TABLE user_roles (
  user_id INT,
  role_id INT,
  PRIMARY KEY (user_id, role_id),
  FOREIGN KEY (user_id) REFERENCES users(id),
  FOREIGN KEY (role_id) REFERENCES roles(id)
);
```

Performance note: Composite keys create larger indexes and slower joins. Use surrogate keys when the composite isn't semantically meaningful.

### Referential Integrity (FOREIGN KEY)

Maintains valid relationships between tables. Prevents orphaned records where child rows reference non-existent parents.

```sql
CREATE TABLE orders (
  order_id INT PRIMARY KEY,
  customer_id INT NOT NULL,
  total DECIMAL(10,2),
  FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
    ON DELETE RESTRICT
    ON UPDATE CASCADE
);
```

**Why this matters:** Without foreign keys, deleting a customer could leave orders pointing to nothing, breaking reports and causing application errors.

### Domain Integrity (CHECK, NOT NULL, DEFAULT)

Enforces valid value ranges and data types at the column level.

```sql
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  price DECIMAL(10,2) CHECK (price > 0),
  stock INT DEFAULT 0 CHECK (stock >= 0),
  status ENUM('active', 'discontinued') DEFAULT 'active'
);
```

**NOT NULL:** Use for fields essential to the entity's meaning. Avoid overuse—nullable columns provide flexibility for incomplete data during workflows.

**CHECK constraints:** Encode business rules that are truly invariant. Don't encode rules that change frequently (use application logic for those).

### Uniqueness (UNIQUE)

Prevents duplicate values where business rules demand it.

```sql
CREATE TABLE users (
  user_id INT PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  username VARCHAR(50) UNIQUE NOT NULL
);
```

UNIQUE constraints create indexes automatically, speeding lookups but adding overhead on inserts/updates. This tradeoff almost always favors the constraint—data corruption is far costlier than marginal write performance.

## Normalization

Organize data to eliminate redundancy and update anomalies. Each normal form addresses specific problems:

**1NF (First Normal Form):** Atomic values only—no arrays, no repeating groups
- ❌ `tags: "red,blue,green"` 
- ✅ Separate `product_tags` junction table

**2NF (Second Normal Form):** No partial dependencies—non-key columns depend on the entire primary key
- Matters only for composite keys
- If `(order_id, product_id)` is the key, `customer_name` shouldn't be in this table (depends only on `order_id`)

**3NF (Third Normal Form):** No transitive dependencies—non-key columns depend only on the primary key, not on other non-key columns
- ❌ Storing both `city` and `state` when `city` determines `state`
- ✅ Separate `cities` table with `city_id`, `city_name`, `state_id`

**When to stop normalizing:**
- **Normalize to 3NF by default** for transactional systems (OLTP)
- **Denormalize selectively** for read-heavy analytics (OLAP) or when join complexity severely impacts performance
- **Never denormalize without measurement**—premature denormalization causes more problems than it solves

**Denormalization tradeoffs:** Faster reads, but increased storage, update complexity, and risk of inconsistency. Prefer materialized views or read replicas over schema denormalization.

## Cascading Behaviors

Define what happens to child records when parent records are modified or deleted.

### CASCADE Options

**ON DELETE CASCADE:** Automatically delete child records when parent is deleted

```sql
FOREIGN KEY (customer_id) REFERENCES customers(id)
  ON DELETE CASCADE
```

**When to use:** Child records are meaningless without the parent (e.g., order line items when the order is deleted).

**ON UPDATE CASCADE:** Automatically update foreign keys when parent primary key changes

```sql
FOREIGN KEY (customer_id) REFERENCES customers(id)
  ON UPDATE CASCADE
```

**When to use:** Natural keys that might change. With surrogate keys, this is rarely needed.

### Restrictive Options

**RESTRICT / NO ACTION:** Prevent parent deletion/update if children exist

```sql
FOREIGN KEY (customer_id) REFERENCES customers(id)
  ON DELETE RESTRICT
```

**When to use (recommended default):** Preserve historical data, prevent accidental mass deletions, maintain audit trails.

**SET NULL:** Set foreign key to NULL when parent is deleted

```sql
FOREIGN KEY (author_id) REFERENCES users(id)
  ON DELETE SET NULL
```

**When to use:** Child records remain meaningful without parent (e.g., blog posts when author account is deleted).

**SET DEFAULT:** Set foreign key to a default value when parent is deleted

### Cascade Safety Guidelines

**Avoid CASCADE in most cases.** Reasons:
1. **Unintended mass deletions:** Deleting one customer could cascade through orders → line_items → shipments → tracking_events, removing thousands of records
2. **Lost audit trails:** Historical data vanishes, violating compliance requirements
3. **Difficult recovery:** Cascaded deletes don't appear in binlogs for some databases, making point-in-time recovery harder
4. **Hidden complexity:** Team members unaware of cascades may trigger unexpected data loss

**Prefer explicit deletion:** Force applications to delete dependent records explicitly, making the impact visible and intentional.

**Soft deletes alternative:** Use `deleted_at` timestamp columns instead of physical deletion:

```sql
CREATE TABLE customers (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  deleted_at TIMESTAMP NULL
);
```

Filter queries with `WHERE deleted_at IS NULL`. Preserves history, enables recovery, supports compliance.

**When CASCADE is acceptable:**
- Tightly coupled data with no independent meaning (e.g., order → line_items in an e-commerce cart before checkout)
- Development/staging environments where data loss is acceptable
- After careful analysis and documentation of the cascade chain

## Forbidden Unsafe Operations

Operations that violate integrity or risk data loss:

### Schema-Level Dangers

**Dropping constraints without replacement:**
```sql
-- DANGEROUS: Removes protection, allows invalid data
ALTER TABLE orders DROP FOREIGN KEY fk_customer;
```

Only drop constraints when replacing with stronger ones or when the relationship genuinely no longer exists.

**Disabling constraint checking:**
```sql
-- DANGEROUS: Bypasses validation during bulk operations
SET FOREIGN_KEY_CHECKS = 0;
-- bulk import
SET FOREIGN_KEY_CHECKS = 1;
```

Use only for migrations with verified data integrity. Never in production application code.

**Nullable foreign keys without business justification:**
```sql
-- QUESTIONABLE: When is an order without a customer valid?
FOREIGN KEY (customer_id) REFERENCES customers(id)
```

Nullable foreign keys allow orphaned relationships. Use only when the relationship is genuinely optional.

### Data-Level Dangers

**Bulk updates without WHERE clause:**
```sql
-- DANGEROUS: Updates all rows
UPDATE customers SET status = 'inactive';
```

Always include WHERE. Wrap in transactions with verification queries.

**Direct primary key modification:**
```sql
-- AVOID: Breaks relationships even with ON UPDATE CASCADE
UPDATE customers SET customer_id = 9999 WHERE customer_id = 1;
```

Surrogate keys should never change. If using natural keys that must change, ensure CASCADE is configured and tested.

**Deleting parent records without checking dependents:**
```sql
-- DANGEROUS: May fail or cascade unexpectedly
DELETE FROM customers WHERE customer_id = 1;
```

Query dependent tables first. Use transactions to verify impact before committing.

## Implementation Checklist

**Schema Design:**
- [ ] Every table has a primary key (prefer surrogate keys)
- [ ] All relationships have foreign key constraints
- [ ] Critical columns have NOT NULL constraints
- [ ] Business-critical uniqueness enforced with UNIQUE constraints
- [ ] Valid value ranges enforced with CHECK constraints or ENUM types
- [ ] Schema normalized to at least 3NF unless denormalization is measured and justified

**Cascading Configuration:**
- [ ] Default to ON DELETE RESTRICT for preservation of data
- [ ] Use ON UPDATE CASCADE only for natural keys
- [ ] Document any CASCADE behaviors with justification
- [ ] Test cascade chains to understand full impact
- [ ] Consider soft deletes instead of physical deletion

**Constraint Naming:**
```sql
-- Good: Descriptive names aid debugging and migrations
CONSTRAINT fk_orders_customer_id FOREIGN KEY (customer_id) REFERENCES customers(id),
CONSTRAINT chk_products_price_positive CHECK (price > 0),
CONSTRAINT uq_users_email UNIQUE (email)
```

**Migration Safety:**
- [ ] Add constraints in transactions with rollback capability
- [ ] Verify existing data satisfies new constraints before adding them
- [ ] Add NOT NULL in two steps: add nullable column, populate, then make NOT NULL
- [ ] Test constraint violations in staging with production-like data

## Common Patterns

**Audit Trail Tables:**
```sql
CREATE TABLE order_history (
  history_id INT PRIMARY KEY AUTO_INCREMENT,
  order_id INT NOT NULL,
  changed_by INT NOT NULL,
  changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  old_status VARCHAR(50),
  new_status VARCHAR(50),
  FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE RESTRICT,
  FOREIGN KEY (changed_by) REFERENCES users(id) ON DELETE RESTRICT
);
```

RESTRICT prevents deletion of orders or users with history, preserving audit trail.

**Self-Referencing Hierarchies:**
```sql
CREATE TABLE categories (
  category_id INT PRIMARY KEY,
  parent_category_id INT NULL,
  name VARCHAR(100) NOT NULL,
  FOREIGN KEY (parent_category_id) REFERENCES categories(category_id)
    ON DELETE SET NULL
);
```

SET NULL allows deleting parent categories while preserving children as top-level categories.

**Many-to-Many with Metadata:**
```sql
CREATE TABLE project_members (
  user_id INT,
  project_id INT,
  role VARCHAR(50) NOT NULL,
  joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (user_id, project_id),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE
);
```

CASCADE is appropriate here—membership records are meaningless if either user or project is deleted.

## Verification Queries

Check for integrity violations in existing databases:

```sql
-- Find orphaned records (missing foreign key constraint)
SELECT o.* FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id
WHERE c.customer_id IS NULL;

-- Find duplicate values (missing UNIQUE constraint)
SELECT email, COUNT(*) FROM users
GROUP BY email HAVING COUNT(*) > 1;

-- Find NULL values in critical columns (missing NOT NULL)
SELECT * FROM products WHERE name IS NULL OR price IS NULL;

-- Find invalid domain values (missing CHECK constraint)
SELECT * FROM products WHERE price <= 0 OR stock < 0;
```

Run these before adding constraints to identify data that must be cleaned first.