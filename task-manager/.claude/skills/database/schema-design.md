---
name: database-schema-design
description: >
  Design database schemas including ERD creation, table modeling, relationship mapping,
  normalization reasoning, SQL generation, and capacity planning. Use this for database
  design, data modeling, schema architecture, creating entity-relationship diagrams,
  defining tables and relationships, database normalization or denormalization decisions,
  foreign key constraints, index strategy, SQL DDL generation, schema migration planning,
  or any task involving relational database structure. Helpful for PostgreSQL, MySQL,
  SQL Server, Oracle, and other RDBMS schema design.
---

# Database Schema Design

Design relational database schemas from requirements through to production-ready SQL. This skill covers ERD creation, table modeling, relationship mapping, normalization reasoning, SQL generation, and capacity planning.

## Core Workflow

### 1. Understand Requirements

Start by identifying:

- **Entities**: What are the core concepts? (users, orders, products, etc.)
- **Attributes**: What information does each entity hold?
- **Relationships**: How do entities relate? (one-to-one, one-to-many, many-to-many)
- **Access patterns**: Which queries will run frequently? Which are rare?
- **Write vs. read ratio**: OLTP (transactional) or OLAP (analytical)?
- **Scale**: Expected row counts, growth rate, concurrency

Ask clarifying questions when requirements are ambiguous. The schema must match how the data will actually be used.

### 2. Create the ERD

Visualize the schema before writing SQL. An ERD shows:

- **Entities** as rectangles (tables)
- **Attributes** inside or listed beside entities
- **Relationships** as lines with cardinality markers (crow's foot notation is common)
- **Primary keys** (often marked with PK or asterisk)
- **Foreign keys** (often marked with FK)

Label all entities and relationships clearly. Use singular nouns for entity names (e.g., `User`, not `Users`). The ERD is the blueprint—get this right and SQL generation becomes straightforward.

### 3. Model Tables and Relationships

**Table naming**: Use `snake_case` consistently. Singular nouns (e.g., `customer`, not `customers`). Avoid reserved words, hyphens, spaces, special characters.

**Primary keys**: Every table needs one. Options:
- **Auto-increment integers** (`SERIAL`, `BIGSERIAL`, `AUTO_INCREMENT`): Simple, compact, clustered-index friendly.
- **UUIDs**: Globally unique, good for distributed systems or public-facing APIs, but larger index size and slightly worse insert performance. Use sequential UUIDs (e.g., `uuid_generate_v7()` in PostgreSQL 17+) when possible to preserve insert order.

**Foreign keys**: Always define `FOREIGN KEY` constraints at the database level—never rely solely on application logic. Choose referential actions explicitly:
- `RESTRICT` (safe default): Prevents deletion if children exist.
- `CASCADE`: Deletes or updates children automatically (use for truly dependent data).
- `SET NULL`: Child can exist without parent.

Every foreign key should have an index for join performance.

**Relationship patterns**:

- **One-to-many**: FK in the "many" side points to PK of the "one" side. Example: `order.customer_id` → `customer.id`.
- **Many-to-many**: Requires a junction table with FKs to both sides. Example: `student_course` with `student_id` + `course_id`. Name junction tables by combining entity names.
- **One-to-one**: Rare. Use when splitting a large entity or when a subset of rows has optional extended data. FK can be on either side, often with `UNIQUE` constraint.

### 4. Normalization Reasoning

Normalization reduces redundancy and prevents anomalies. Most production schemas aim for **3NF** or **BCNF**:

- **1NF**: Atomic values, no repeating groups.
- **2NF**: 1NF + no partial dependency on composite keys.
- **3NF**: 2NF + no transitive dependency (non-key columns depend only on the primary key).
- **BCNF**: 3NF + every determinant is a candidate key.

**When to normalize**:
- OLTP systems (frequent inserts, updates, deletes)
- Data integrity is critical (banking, ERP, CRM)
- Schema needs to evolve easily
- Storage efficiency matters

**When to denormalize**:
- OLAP systems (analytics, reporting, dashboards)
- Read-heavy workloads where joins become bottlenecks
- Frequently accessed data that rarely changes
- Real-time dashboards needing sub-second response

**Denormalization techniques**:
- Duplicate columns across tables to avoid joins
- Flatten fact and dimension tables (star schema)
- Materialized views for expensive aggregations
- Add computed/derived columns

Denormalization trades write complexity and storage for read speed. Document the reasoning: "Denormalized `customer_name` into `order` table because 80% of queries need customer name with order details, and customer names rarely change."

### 5. Define Constraints and Indexes

**Constraints enforce rules at the database level**:

- `NOT NULL`: Flag nullable columns carefully. If a column should always have a value, make it `NOT NULL`.
- `UNIQUE`: Enforce uniqueness for business keys (e.g., email, username, SKU).
- `CHECK`: Validate data ranges, formats, or business rules (e.g., `CHECK (price >= 0)`).
- `DEFAULT`: Provide sensible defaults (e.g., `created_at DEFAULT NOW()`).

**Indexes speed up queries but slow down writes**:

- Index all foreign keys (for joins).
- Index columns used in `WHERE`, `ORDER BY`, `GROUP BY`.
- Composite indexes for multi-column filters (order matters: most selective first).
- Avoid over-indexing—every index adds write overhead.
- Use partial indexes for queries filtering on specific values.
- Consider covering indexes for read-heavy queries.

### 6. Add Metadata Columns

Include standard metadata on most tables:

- `created_at TIMESTAMP NOT NULL DEFAULT NOW()`
- `updated_at TIMESTAMP NOT NULL DEFAULT NOW()` (with trigger or application update)
- `deleted_at TIMESTAMP` (for soft deletes, if applicable)
- `version INT` or `etag` (for optimistic locking, if needed)

These columns are cheap to add and invaluable for debugging, auditing, and data lineage.

### 7. Generate SQL DDL

Translate the ERD into executable SQL. Structure:

```sql
-- Create tables in dependency order (parent tables first)
CREATE TABLE customer (
  id BIGSERIAL PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  name VARCHAR(255) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE order (
  id BIGSERIAL PRIMARY KEY,
  customer_id BIGINT NOT NULL REFERENCES customer(id) ON DELETE RESTRICT,
  total_amount DECIMAL(10,2) NOT NULL CHECK (total_amount >= 0),
  status VARCHAR(50) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Create indexes
CREATE INDEX idx_order_customer_id ON order(customer_id);
CREATE INDEX idx_order_status ON order(status);
CREATE INDEX idx_order_created_at ON order(created_at DESC);
```

**Data type selection**:
- Use appropriate sizes: `SMALLINT`, `INT`, `BIGINT` based on range.
- `VARCHAR(n)` for variable-length strings; avoid `VARCHAR(MAX)` or overly large limits.
- `DECIMAL(p,s)` for money (never `FLOAT` or `DOUBLE`).
- `TIMESTAMP` or `TIMESTAMPTZ` for dates with time; `DATE` for date-only.
- `BOOLEAN` for true/false flags.
- `JSONB` (PostgreSQL) for semi-structured data, but normalize when structure is stable.

### 8. Capacity Planning

Estimate storage and performance:

- **Row size**: Sum of column sizes + overhead (typically 24-40 bytes per row).
- **Table size**: Row size × expected row count.
- **Index size**: Typically 30-50% of table size per index.
- **Growth rate**: Estimate annual growth; plan for 3-5 years.
- **Query load**: Estimate QPS (queries per second) for common queries.

Identify hotspots early:
- Large tables (> 10M rows) needing partitioning
- High-cardinality columns needing better indexing
- Expensive joins suggesting denormalization

## Design Principles

**Design for queries, not just entities**: The schema must support actual access patterns efficiently. If 80% of queries need data from three tables joined together, consider denormalizing or creating a materialized view.

**Enforce integrity at the database level**: Foreign keys, `NOT NULL`, `UNIQUE`, `CHECK` constraints. Don't rely on application code alone—databases outlive applications, and multiple apps may access the same database.

**Use consistent conventions**: Pick a naming style (`snake_case` recommended), FK naming pattern (`<table>_id`), and stick to it across the entire schema.

**Document non-obvious decisions**: Use column comments, schema README, or ERD annotations to explain why a column is nullable, what valid enum values are, why a particular index exists, why denormalization was chosen.

**Plan for change**: Schemas evolve. Use migrations (not direct DDL changes). Version control your schema. Design backward-compatible changes when possible (add columns with defaults, avoid renaming). Test migrations against production-scale data.

**Avoid premature optimization**: Start normalized. Denormalize only when you have evidence (via query profiling) that joins are a bottleneck. Measure before optimizing.

## Common Patterns

**Soft deletes**: Add `deleted_at TIMESTAMP` and filter `WHERE deleted_at IS NULL` in queries. Useful for audit trails and accidental deletion recovery, but complicates unique constraints.

**Polymorphic associations**: Avoid storing `(entity_type, entity_id)` pairs—they break referential integrity. Use separate FKs or separate junction tables instead.

**Audit logs**: Separate table capturing `(table_name, row_id, action, old_value, new_value, user_id, timestamp)`. Alternatively, use database triggers or change data capture (CDC).

**Lookup tables**: Small, static reference data (countries, statuses, categories). Often loaded into application memory. Keep simple: `(id, code, name)`.

**Time-series data**: Partition by time (monthly or yearly). Index on timestamp. Consider retention policies.

**Multi-tenancy**: Options include separate schemas per tenant, tenant_id column in every table (with row-level security), or separate databases. Trade-offs in isolation, complexity, and cost.

## Example: E-Commerce Schema

**Entities**: Customer, Product, Order, OrderItem, Category

**ERD summary**:
- Customer (1) → (many) Order
- Order (1) → (many) OrderItem
- Product (1) → (many) OrderItem
- Category (1) → (many) Product

**Normalization decisions**:
- Normalized: Customer address in separate `address` table (customers can have multiple addresses).
- Denormalized: `product_name` and `product_price` copied into `order_item` (captures price at time of order, even if product price changes later).

**Indexes**:
- `order.customer_id`, `order.status`, `order.created_at`
- `order_item.order_id`, `order_item.product_id`
- `product.category_id`, `product.sku` (unique)

**Constraints**:
- `order.total_amount >= 0`
- `order_item.quantity > 0`
- `product.price >= 0`
- `customer.email` unique

This schema supports common queries (customer order history, product sales, revenue by category) efficiently while maintaining data integrity.

## Migrations and Versioning

Never alter production schemas directly. Use migration tools:
- **Flyway**, **Liquibase** (Java ecosystem)
- **Alembic** (Python)
- **ActiveRecord Migrations** (Ruby)
- **Prisma Migrate** (Node.js)

Each migration is versioned and idempotent. Migrations run in order, applied once, and tracked in a `schema_migrations` table.

**Safe migration patterns**:
- Add columns with defaults (non-blocking)
- Create indexes concurrently (PostgreSQL: `CREATE INDEX CONCURRENTLY`)
- Avoid renaming columns—add new, backfill, deprecate old
- Test migrations on production-scale data in staging
- Have rollback scripts ready for high-risk changes

## Tools and Visualization

ERD and schema design tools:
- **DbSchema**, **MySQL Workbench**, **pgAdmin**, **DBeaver**: Reverse-engineer existing databases, design visually, generate SQL.
- **Lucidchart**, **draw.io**, **Mermaid**, **ERDPlus**: Lightweight ERD drawing.
- **dbdiagram.io**: Define schemas in code, visualize automatically.

Use these tools to collaborate with stakeholders—visual diagrams communicate better than SQL.
