---
name: transaction-handling
description: >
  Design and implement database transactions with proper isolation levels,
  locking strategies, ACID guarantees, and conflict resolution. Use this skill
  when working with database transactions, concurrency control, data integrity,
  isolation levels (Read Uncommitted, Read Committed, Repeatable Read, Serializable,
  Snapshot), locking mechanisms (pessimistic, optimistic, MVCC), transaction design,
  avoiding long-running transactions, handling deadlocks, transaction timeouts,
  rollback strategies, distributed transactions, two-phase commit, saga patterns,
  compensating transactions, or any scenario involving concurrent database access,
  data consistency, race conditions, dirty reads, phantom reads, non-repeatable reads,
  transaction log management, or multi-user database systems.
---

# Transaction Handling

## Overview

Transactions ensure data integrity through ACID properties (Atomicity, Consistency, Isolation, Durability). The skill covers choosing isolation levels, designing locking strategies, avoiding long-running transactions, and resolving conflicts—all critical for correctness in concurrent systems.

## Core Decision: Isolation Level Selection

Isolation level determines the trade-off between consistency and concurrency. Choose based on anomaly tolerance and performance requirements.

**Read Uncommitted** — Allows dirty reads. Rarely appropriate; use only for approximate analytics where speed matters more than accuracy (e.g., dashboards showing rough counts).

**Read Committed** — Default in most systems. Prevents dirty reads but allows non-repeatable reads. Good balance for OLTP: web apps, APIs, transactional workloads where individual statements need committed data but cross-statement consistency isn't critical.

**Repeatable Read** — Prevents dirty and non-repeatable reads. Use when a transaction needs a stable view of specific rows: generating reports, multi-step workflows reading the same data multiple times. Still permits phantom reads (new rows appearing).

**Serializable** — Strictest level. Transactions behave as if executed sequentially. Essential for financial operations, inventory management, any scenario where correctness cannot be compromised. Expect serialization errors; implement retry logic.

**Snapshot Isolation** — Uses MVCC to provide consistent point-in-time views without locking readers. Reduces read-write contention. Ideal for reporting, analytics, high-concurrency reads. Default in Azure SQL Database; opt-in elsewhere.

### Choosing the Level

- **Financial transactions, inventory, seat booking** → Serializable (accept retries for correctness)
- **Reports, dashboards, analytics** → Snapshot or Repeatable Read (consistent view without blocking)
- **Web/mobile OLTP, general CRUD** → Read Committed (speed + basic consistency)
- **Approximate analytics** → Read Uncommitted (only when dirty reads acceptable)

Higher isolation = stronger guarantees + more blocking + potential for serialization failures. Lower isolation = higher concurrency + risk of anomalies.

## Locking Strategies

**Pessimistic Locking** — Acquire locks before accessing data. Prevents conflicts by blocking concurrent access. Use when conflicts are likely or when you must guarantee exclusive access.

- Shared locks (read locks): Multiple readers allowed, block writers
- Exclusive locks (write locks): Block all other access
- Reduces concurrency; risk of deadlocks if lock ordering isn't consistent
- Example: `SELECT ... FOR UPDATE` in PostgreSQL/MySQL

**Optimistic Locking** — No locks during read; check for conflicts at commit. Assume conflicts are rare. Use for high-concurrency scenarios with infrequent collisions.

- Track version numbers or timestamps
- At commit, verify no concurrent modifications occurred
- If conflict detected, abort and retry
- Better concurrency but requires application-level retry logic
- Example: Add `version` column, increment on update, check version unchanged before committing

**MVCC (Multi-Version Concurrency Control)** — Readers see snapshots; writers create new versions. Readers never block writers, writers never block readers. Built into PostgreSQL, Oracle, SQL Server (with snapshot isolation). Increases storage (old versions retained) but dramatically reduces contention.

### Lock Granularity

- Row-level locks: Finest granularity, highest concurrency, most overhead
- Page-level locks: Lock groups of rows; balance between concurrency and overhead
- Table-level locks: Coarsest; use for bulk operations or schema changes

Prefer row-level for OLTP; use table-level only when necessary (bulk load, maintenance).

## ACID Properties in Practice

**Atomicity** — All operations in a transaction succeed or all fail. Implemented via transaction logs and rollback. Wrap related operations in `BEGIN`/`COMMIT`; database handles rollback on error.

**Consistency** — Transactions move database from one valid state to another. Enforce via constraints (foreign keys, unique, check constraints). Design transactions to respect business invariants.

**Isolation** — Concurrent transactions don't interfere. Controlled by isolation level (above). Test concurrent scenarios; isolation bugs often appear only under load.

**Durability** — Committed changes survive crashes. Implemented via write-ahead logging (WAL). Once `COMMIT` returns, data is safe. Ensure proper backup/replication for disaster recovery.

## Avoiding Long-Running Transactions

Long-running transactions cause:

- **Blocking**: Locks held longer, blocking other transactions
- **Bloat**: MVCC systems can't clean old row versions, database grows
- **Log growth**: Transaction logs can't truncate, disk fills
- **Wraparound risks**: PostgreSQL transaction ID exhaustion

### Detection

Monitor transaction duration. In OLTP systems, transactions over 30-60 seconds are concerning. Query active transactions:

```sql
-- PostgreSQL
SELECT pid, now() - xact_start AS duration, state, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY xact_start;

-- SQL Server
SELECT session_id, transaction_id, 
       DATEDIFF(s, transaction_begin_time, GETDATE()) AS duration_sec
FROM sys.dm_tran_active_transactions t
JOIN sys.dm_tran_session_transactions st ON t.transaction_id = st.transaction_id;
```

### Prevention

- **Set timeouts**: `statement_timeout` (per query), `idle_in_transaction_session_timeout` (idle time in transaction)
- **Break into smaller transactions**: Process in batches of 100-1000 rows, commit between batches
- **Avoid user interaction in transactions**: Don't hold transaction open waiting for user input
- **Optimize queries**: Slow queries extend transaction duration; index appropriately
- **Separate read-only and write transactions**: Use lower isolation for reads

Example batch processing:

```sql
SET statement_timeout = '30s';

WHILE rows_remaining > 0 LOOP
  BEGIN;
  UPDATE large_table SET processed = true 
  WHERE id IN (SELECT id FROM large_table WHERE NOT processed LIMIT 1000);
  COMMIT;
  PERFORM pg_sleep(0.1); -- Brief pause to reduce contention
END LOOP;
```

## Conflict Resolution

**Deadlocks** — Two transactions wait for each other's locks. Database detects and aborts one (victim). Implement retry logic with exponential backoff. Prevent by acquiring locks in consistent order.

**Serialization Failures** — At Serializable isolation, database aborts transactions that would violate serializability. Expected behavior; retry the transaction.

**Optimistic Concurrency Conflicts** — Version mismatch at commit. Retry with fresh data.

### Retry Pattern

```python
import time
import psycopg2
from psycopg2 import errors

def execute_with_retry(operation, max_attempts=3):
    for attempt in range(max_attempts):
        try:
            conn.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_SERIALIZABLE)
            with conn:
                with conn.cursor() as cur:
                    operation(cur)
            return  # Success
        except errors.SerializationFailure:
            if attempt == max_attempts - 1:
                raise
            time.sleep(0.1 * (2 ** attempt))  # Exponential backoff
        except errors.DeadlockDetected:
            if attempt == max_attempts - 1:
                raise
            time.sleep(0.1 * (2 ** attempt))
```

Always handle `SerializationFailure` and `DeadlockDetected` when using Serializable isolation or pessimistic locking.

## Distributed Transactions

When transactions span multiple databases or services:

**Two-Phase Commit (2PC)** — Coordinator asks all participants to prepare (phase 1), then commits if all agree (phase 2). Provides atomicity across systems but blocks if coordinator fails. Use when strong consistency required across databases.

**Saga Pattern** — Chain of local transactions with compensating actions. Each step commits independently; if later step fails, compensate previous steps. Use for long-running business processes across microservices.

- **Choreography**: Each service publishes events; others react
- **Orchestration**: Central coordinator directs the saga

Compensation is business-specific: canceling a reservation, issuing a refund. Design idempotent compensations.

**Eventual Consistency** — Accept temporary inconsistency; system converges to consistent state. Use when availability matters more than immediate consistency (e.g., social media likes, view counts).

## Practical Guidelines

- **Start with Read Committed**; escalate isolation only when anomalies observed
- **Keep transactions short**: Seconds, not minutes. Batch large operations.
- **Set timeouts globally**: Prevent runaway transactions from degrading system
- **Test under concurrency**: Isolation bugs appear under load; use load testing tools
- **Monitor transaction duration**: Alert on transactions exceeding threshold
- **Index for concurrency**: Ensure queries inside transactions use indexes to minimize lock duration
- **Use optimistic locking for low-contention**: Version columns for entities updated infrequently
- **Use pessimistic locking for high-contention**: `SELECT FOR UPDATE` when conflicts likely
- **Implement retry logic**: Essential for Serializable isolation and distributed systems
- **Avoid SELECT * in transactions**: Fetch only needed columns to reduce lock scope
- **Commit or rollback explicitly**: Don't rely on implicit behavior; be explicit about transaction boundaries

## Common Patterns

**Financial Transfer**

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A' AND balance >= 100;
IF NOT FOUND THEN RAISE EXCEPTION 'Insufficient funds';

UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B';

COMMIT;
```

**Inventory Reservation**

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT quantity FROM inventory WHERE product_id = 123 FOR UPDATE;
-- Check quantity sufficient
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 123;

COMMIT;
-- Handle serialization failure with retry
```

**Report Generation**

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- or SNAPSHOT if available

SELECT /* complex report queries */;
-- All queries see consistent snapshot

COMMIT;
```

**Bulk Update with Batching**

```sql
DO $$
DECLARE
  batch_size INT := 1000;
  updated INT;
BEGIN
  LOOP
    BEGIN;
    UPDATE large_table SET status = 'processed'
    WHERE id IN (
      SELECT id FROM large_table 
      WHERE status = 'pending' 
      LIMIT batch_size
    );
    GET DIAGNOSTICS updated = ROW_COUNT;
    COMMIT;
    
    EXIT WHEN updated = 0;
    PERFORM pg_sleep(0.05);
  END LOOP;
END $$;
```