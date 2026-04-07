---
name: data-flow-synchronization
description: >
  Design data flow and synchronization architectures including read/write flows,
  event propagation, ETL/ELT pipelines, data replication strategies, CDC patterns,
  eventual consistency models, and idempotency. Use this skill when designing or
  evaluating distributed data systems, microservices data integration, real-time
  sync pipelines, bi-directional data flows, event-driven architectures with data
  consistency requirements, database replication strategies, or when working on
  data synchronization between CRMs, ERPs, warehouses, operational databases, or
  any scenario involving data movement, transformation, consistency guarantees,
  conflict resolution, or idempotent operations.
---

# Data Flow & Synchronization Design

## Overview

This skill covers architectural patterns for moving, transforming, and synchronizing data across distributed systems. The core challenge: maintaining consistency and correctness while data flows between systems with different consistency models, latency requirements, and failure modes.

## Core Decision Framework

### Directionality

**One-way sync** (source → destination):
- Analytics pipelines, reporting, data warehouses
- Migrations with cutover windows
- Read replicas for scaling reads
- Audit logs, compliance archives

**Bi-directional sync** (A ↔ B):
- Operational systems requiring shared state (CRM ↔ ERP)
- Multi-master replication
- Offline-first applications with eventual merge
- Cross-region active-active setups

Bi-directional requires conflict resolution strategy: last-write-wins (with vector clocks or timestamps), application-level merge logic, or manual intervention.

### Consistency Model

**Strong consistency** — all replicas see same data immediately:
- Financial transactions, inventory with hard limits
- Requires coordination (2PC, consensus), sacrifices availability during partitions
- High latency for geo-distributed writes

**Eventual consistency** — replicas converge over time:
- User profiles, social feeds, content catalogs
- Favors availability and partition tolerance (AP in CAP)
- Requires idempotent operations and conflict resolution
- Typical convergence: milliseconds to seconds in healthy systems

**Read-your-writes consistency** — users see their own updates immediately:
- Hybrid: route user's reads to same replica that handled their write
- Session stickiness or client-side tracking of write timestamps

Most modern distributed systems default to eventual consistency for scalability, using strong consistency only where business invariants demand it.

### Latency & Throughput

**Real-time (sub-second)**: Event streams (Kafka, Kinesis), CDC with log-based replication, WebSockets
- Use when: operational dashboards, fraud detection, real-time personalization

**Near-real-time (seconds to minutes)**: Polling with delta queries, micro-batch processing
- Use when: analytics refresh, inventory sync with acceptable lag

**Batch (hours to days)**: Scheduled ETL jobs
- Use when: historical reporting, compliance archives, low-change-rate datasets

Latency drives architecture: real-time requires event-driven patterns; batch can use simpler scheduled jobs.

## Read/Write Flow Patterns

### Write Patterns

**Direct write** — application writes to primary, sync replicates:
- Simplest, but couples application to replication lag
- Use with strong consistency requirements

**Write-ahead log (WAL) capture** — CDC from database transaction log:
- Debezium, AWS DMS, logical replication (Postgres)
- Captures every change without application code changes
- Preserves order within a partition/shard

**Outbox pattern** — write business data + event to same database transaction:
- Guarantees atomicity: either both succeed or both fail
- Separate process polls outbox table or uses CDC on it
- Prevents dual-write problem (writing to DB and message queue separately can fail halfway)

**Event sourcing** — store events as source of truth, derive state:
- Append-only event log, rebuild state by replaying events
- Natural audit trail, time-travel queries
- Complexity: event schema evolution, snapshotting for performance

### Read Patterns

**Read from replica** — offload reads to follower databases:
- Reduces load on primary
- Accepts replication lag (eventual consistency for reads)

**CQRS (Command Query Responsibility Segregation)** — separate write and read models:
- Write model optimized for transactions, read model for queries
- Read model is projection/materialized view, updated via events
- Use when read and write access patterns diverge significantly

**Cache-aside** — application checks cache, then database:
- Reduces database load for hot data
- Invalidation strategy critical: TTL, write-through, or event-driven purge

## Event Propagation

### Event Design

**Thin events** (notification-only): `{"orderId": "123", "event": "OrderCreated"}`
- Consumer fetches full data from source
- Lower bandwidth, simpler versioning
- Adds latency (extra query), coupling (consumer needs source access)

**Fat events** (event-carried state transfer): `{"orderId": "123", "customerId": "456", "items": [...], "total": 99.99}`
- Consumer has all data in event
- Higher bandwidth, versioning complexity (schema evolution)
- Decouples consumer from source, lower latency

Default to fat events for operational use cases (consumers need to act on data), thin events for notifications where consumers already have context.

### Ordering & Delivery Guarantees

**At-most-once**: Fire and forget, no retries. Data loss possible. Rarely acceptable.

**At-least-once**: Retry until acknowledged. Duplicates possible. **Requires idempotent consumers.**

**Exactly-once**: Guaranteed single processing. Expensive (distributed transactions or deduplication). Kafka supports exactly-once semantics within a transaction; end-to-end exactly-once across systems is impractical—design for idempotency instead.

**Ordering**: Kafka preserves order within a partition; EventBridge/SQS standard queues offer best-effort ordering. If order matters, use partition keys (Kafka) or FIFO queues (SQS FIFO) with message group IDs. Design consumers to tolerate out-of-order events when possible (use sequence numbers, timestamps).

## ETL/ELT Logic

**ETL (Extract, Transform, Load)** — transform before loading into target:
- Transformation in staging layer or dedicated pipeline (Airflow, dbt)
- Use when: target is optimized for queries (warehouse), transformations are complex
- Tools: Fivetran, Airbyte, Talend, custom Python/Spark jobs

**ELT (Extract, Load, Transform)** — load raw, transform in target:
- Leverages warehouse compute (Snowflake, BigQuery, Databricks)
- Use when: target has powerful query engine, schema-on-read flexibility desired
- Faster initial load, transformations versioned as SQL/dbt models

**Reverse ETL** — push transformed warehouse data back to operational systems:
- Enriched customer segments to CRM, ML features to application DB
- Tools: Census, Hightouch

Modern trend: ELT for analytics (raw data lake → warehouse → dbt transformations), ETL for operational sync (validate/transform before writing to production systems).

## Data Replication Strategies

**Snapshot replication** — full copy at intervals:
- Simple, but doesn't scale for large or frequently changing datasets
- Use for small reference tables, initial seeding

**Incremental replication** — only changed rows:
- Delta queries: `WHERE updated_at > last_sync_timestamp`
- Requires timestamp column, doesn't capture deletes well
- Polling-based, adds load to source

**Log-based replication (CDC)** — tail database transaction log:
- Captures inserts, updates, deletes in real-time
- Minimal source load, preserves order
- Tools: Debezium, AWS DMS, Postgres logical replication, MySQL binlog
- Best for operational replication and event-driven architectures

**Streaming replication** — continuous event flow:
- Kafka, Kinesis, Pulsar
- Producers publish changes as events, consumers process stream
- Decouples producers/consumers, enables fan-out (multiple consumers per stream)

## Eventual Consistency & Conflict Resolution

### Handling Temporary Inconsistency

- **Design UX for lag**: Show "processing" states, avoid immediate reads after writes across systems
- **Compensating transactions**: If downstream processing fails, emit reverse event or manual reconciliation
- **Saga pattern**: Multi-step distributed transaction with rollback logic for each step

### Conflict Resolution (Bi-directional Sync)

**Last-write-wins (LWW)**:
- Use timestamp or version number; highest wins
- Simple but loses concurrent updates
- Requires synchronized clocks (NTP) or logical clocks (vector clocks)

**Application-specific merge**:
- Shopping cart: union of items from both replicas
- Counter: sum increments (CRDTs — Conflict-free Replicated Data Types)
- Requires domain logic, more complex but preserves intent

**Manual resolution**:
- Present conflicts to user or admin
- Use when: critical data (financial records), no safe automatic merge

## Idempotency

**Why it matters**: Network retries, at-least-once delivery, and failure recovery mean operations will be executed multiple times. Idempotency ensures repeated execution has same effect as single execution.

### Implementation Patterns

**Idempotency keys**:
- Client generates unique ID (UUID) per request
- Server stores key + result; if key seen again, return cached result without re-executing
- TTL on keys (e.g., 24 hours) balances storage vs. protection window

**Natural idempotency**:
- `SET status = 'active'` (same result every time)
- `DELETE WHERE id = 123` (deleting again is no-op)
- Prefer idempotent operations in design (set absolute values vs. increment)

**Conditional updates**:
- Optimistic locking: `UPDATE ... WHERE version = expected_version`
- Fails if state changed since read, client retries with fresh read
- Prevents lost updates, ensures idempotency

**Deduplication in consumers**:
- Track processed event IDs in database or cache
- Before processing, check if event ID already handled
- Use event sequence numbers or stream offsets when available

### Database-Specific Techniques

- **Postgres**: `INSERT ... ON CONFLICT DO NOTHING` or `DO UPDATE`
- **MongoDB**: `updateOne` with filter on version field, `$gte` check for event revision
- **DynamoDB**: Conditional writes with `expected` attribute

Idempotency is non-negotiable for at-least-once delivery and retry-heavy distributed systems.

## Common Anti-Patterns

**Dual writes** — writing to database and message queue separately:
- Can fail halfway (DB succeeds, queue fails or vice versa)
- Use outbox pattern or CDC instead

**Polling without backoff**:
- Hammers source system, wastes resources
- Use exponential backoff or switch to event-driven (CDC, webhooks)

**Ignoring replication lag**:
- Reading immediately after write from replica can return stale data
- Use read-your-writes consistency (read from primary or track write timestamp)

**Non-idempotent operations with at-least-once delivery**:
- Leads to duplicate charges, double-counted metrics, corrupted state
- Always design for idempotency when retries are possible

**Fat events without versioning**:
- Breaking schema changes break consumers
- Use schema registry (Avro, Protobuf), version events, support backward compatibility

## Practical Guidance

### Choosing a Pattern

1. **Identify consistency requirements**: Can you tolerate lag? How much?
2. **Determine directionality**: One-way or bi-directional?
3. **Assess latency needs**: Real-time, near-real-time, or batch?
4. **Evaluate source system**: Can you use CDC, or must you poll? Is it operational or analytical?
5. **Plan for failure**: Retries? Idempotency? Conflict resolution?

### Designing for Resilience

- **Idempotency by default**: Assume every operation will be retried
- **Monitoring & alerting**: Track replication lag, event processing latency, error rates
- **Dead-letter queues**: Capture failed events for manual inspection, prevent blocking
- **Circuit breakers**: Stop retrying failing downstream systems to prevent cascading failures
- **Backpressure**: Slow producers if consumers can't keep up (rate limiting, queue depth limits)

### Testing

- **Chaos testing**: Inject failures (network partitions, slow replicas) to verify eventual consistency
- **Duplicate event injection**: Verify idempotency by processing same event multiple times
- **Out-of-order events**: Shuffle event order to test sequence handling
- **Lag simulation**: Delay replication to test UX under eventual consistency

## Example: E-commerce Order Flow

**Write path**:
1. User submits order → API writes to Orders DB + Outbox table (single transaction)
2. CDC process tails Outbox → publishes `OrderCreated` event to Kafka
3. Inventory service consumes event → reserves stock (idempotent: checks order ID before reserving)
4. Payment service consumes event → charges card (idempotency key = order ID)
5. Shipping service consumes event → creates shipment

**Read path**:
- Order status API reads from Orders DB (primary for strong consistency)
- Customer dashboard reads from read replica or CQRS projection (eventual consistency acceptable)
- Analytics warehouse receives orders via CDC → ELT transformations in dbt

**Consistency**:
- Strong: Order creation (user must see order immediately)
- Eventual: Inventory reservation (stock count can lag seconds), analytics (reports can be minutes old)

**Idempotency**:
- Payment service deduplicates by order ID
- Inventory service checks reservation table before decrementing stock
- Outbox processor tracks last processed offset to avoid reprocessing

**Conflict resolution**: N/A (one-way flow from orders to downstream systems)

This architecture handles retries, partial failures, and scales horizontally while maintaining correctness.
