---
name: event-driven-systems
description: >
  Design and maintain event-driven architectures. Use this skill when working with message queues, event sourcing, CQRS, pub/sub patterns, event brokers, Kafka, RabbitMQ, event streams, async messaging, distributed systems, microservices communication, event handlers, event buses, idempotency, retry logic, dead-letter queues, exponential backoff, distributed tracing, observability, eventual consistency, event-driven workflows, or any architecture where services communicate via events rather than direct calls.
---

# Event-Driven Systems

## Overview

Event-driven architecture (EDA) is a pattern where services communicate through events—immutable records of state changes—rather than direct synchronous calls. Producers publish events to brokers; consumers react asynchronously. This decouples services, enabling independent scaling, real-time processing, and resilient systems.

Core principle: Producers don't know who consumes their events. Consumers don't know who produced them. The event broker (Kafka, RabbitMQ, AWS SQS, Google Pub/Sub) handles routing.

## Core Patterns

### Pub/Sub (Publish-Subscribe)

Producers publish events to topics. Multiple consumers subscribe to topics and each receives a copy of every event. Use when multiple services need to react to the same event independently.

**Example:** OrderCreated event → inventory service updates stock, email service sends confirmation, analytics service logs metrics.

### Message Queues (Competing Consumers)

Producers send messages to a queue. Consumers compete to process messages—each message is delivered to exactly one consumer. Use for work distribution and load balancing.

**Example:** Image processing queue → multiple workers pull tasks, process images, acknowledge completion.

### Event Streaming

Events are written to an ordered, durable log (Kafka topics). Consumers read from any position and control their own offset. Events are retained (hours to forever) enabling replay. Use for high-throughput data pipelines, audit logs, and time-series data.

## Idempotency

Distributed systems deliver messages at-least-once by default. Network failures, retries, and rebalancing cause duplicates. Idempotency ensures processing the same message multiple times produces the same result as processing it once.

### Why It Matters

Without idempotency: charging a credit card twice, creating duplicate records, sending multiple notifications. With idempotency: retries are safe.

### Implementation Strategies

**Natural idempotency:** Operations that are inherently idempotent (SET user.email = "new@example.com", DELETE where id = 123).

**Idempotency keys:** Assign each event a unique ID (event ID, request ID, or business key like order_id). Before processing:

```python
if idempotency_store.exists(event.id):
    return  # Already processed

process_event(event)
idempotency_store.add(event.id)  # Mark as processed
```

Store idempotency keys in the same database as your business logic, within the same transaction. This prevents partial failures where the event is processed but the key isn't recorded.

**Precondition checks:** Before applying changes, verify expected state. If state doesn't match, skip processing (likely a duplicate).

**Not every consumer needs explicit idempotency.** If your operation is naturally idempotent or duplicates cause no harm, skip the overhead. Apply idempotency where duplicates cause financial loss, data corruption, or user-facing issues.

## Retry Logic and Error Handling

Transient failures (network timeouts, temporary service unavailability) are normal in distributed systems. Retries increase reliability, but naive retries can overwhelm struggling services.

### Exponential Backoff

Increase delay between retries exponentially: 1s, 2s, 4s, 8s. This gives failing services time to recover and prevents retry storms.

```python
delay = base_delay * (2 ** attempt_number)
```

Add jitter (random variance) to prevent synchronized retries from multiple consumers hitting at the same moment:

```python
delay = base_delay * (2 ** attempt_number) * (0.5 + random.random())
```

### Max Retry Limits

Set a maximum retry count (3-5 attempts is common). After exhaustion, move the message to a dead-letter queue for manual inspection or automated handling.

### Dead-Letter Queues (DLQ)

A secondary queue for messages that fail repeatedly. DLQs prevent poison pills (malformed messages) from blocking the main queue. Monitor DLQs actively—they signal systemic issues (schema changes, downstream service failures, bugs).

**Workflow:**
1. Consumer fails to process message
2. Retry with exponential backoff
3. After max retries, publish to DLQ
4. Alert on-call team or trigger automated remediation
5. Investigate, fix root cause, replay messages from DLQ

### Circuit Breakers

If a downstream service is consistently failing, stop sending requests temporarily. After a timeout, try again. This prevents cascading failures and gives failing services breathing room to recover.

## Event Sourcing and CQRS

### Event Sourcing

Store state as a sequence of events rather than current state. Instead of UPDATE users SET balance = 150, store DepositedFunds(amount=50) and WithdrewFunds(amount=100). Replay events to reconstruct current state.

**Benefits:**
- Complete audit trail—every state change is recorded
- Time travel—replay events to any point in time
- Debuggability—see exactly how the system arrived at current state
- New projections—build new read models from existing events

**Tradeoffs:**
- Increased complexity—different programming model
- Storage overhead—every event is kept
- Query complexity—need to replay events to get current state

### CQRS (Command Query Responsibility Segregation)

Separate write operations (commands) from read operations (queries). Commands update the write store (event log). Queries read from optimized read stores (materialized views).

**Why combine with event sourcing:** Event sourcing makes writes efficient (append-only) but queries expensive (replay events). CQRS solves this by maintaining separate read models updated asynchronously from events.

**Workflow:**
1. Command handler validates and publishes event to event store
2. Event handler consumes event, updates read model (denormalized, optimized for queries)
3. Query handler reads from read model directly

**Tradeoff:** Eventual consistency. Read model may lag behind write model by milliseconds to seconds. If strong consistency is required, CQRS adds complexity without benefit.

## Distributed Tracing and Observability

Asynchronous event flows make debugging harder. A single user action can trigger a cascade of events across dozens of services. Distributed tracing connects these flows.

### Correlation IDs

Propagate a unique ID (correlation_id, trace_id) through the entire event chain. Every service logs this ID. When debugging, search logs by correlation ID to see the complete flow.

```python
event = {
    "correlation_id": request.headers.get("X-Correlation-ID", generate_id()),
    "event_type": "OrderCreated",
    "data": {...}
}
```

### Structured Logging

Log events in structured format (JSON) with consistent fields: timestamp, service_name, event_type, correlation_id, duration, status. This enables querying and aggregation.

### Key Metrics

- **Queue depth:** Messages waiting to be processed. Rising depth indicates consumers are falling behind.
- **Processing latency:** Time from event publication to consumption. Tracks end-to-end delay.
- **Error rate:** Failed messages / total messages. Spikes indicate issues.
- **Retry count:** How often messages are retried. High retries signal transient failures.
- **DLQ size:** Messages in dead-letter queue. Non-zero requires investigation.

### Tracing Tools

OpenTelemetry, Jaeger, Zipkin, AWS X-Ray, Google Cloud Trace. These visualize event flows, measure latency at each hop, and identify bottlenecks.

## Message Design

### Event Naming

Use past tense (OrderCreated, PaymentProcessed, InventoryUpdated). Events represent facts that already happened.

### Event Schema

Include:
- **event_id:** Unique identifier (UUID)
- **event_type:** What happened
- **timestamp:** When it happened (ISO 8601)
- **correlation_id:** Trace ID for distributed tracing
- **data:** Event payload (business data)
- **metadata:** Version, producer service, etc.

### Schema Evolution

Consumers and producers evolve independently. Use backward-compatible changes:
- Add optional fields (consumers ignore unknown fields)
- Never remove or rename fields
- Use schema registries (Confluent Schema Registry, AWS Glue) to enforce compatibility

### Event Size

Keep events small (< 1MB). Large payloads slow down brokers and consumers. For large data (images, documents), store in object storage (S3) and include a reference in the event.

## Ordering and Consistency

### Ordering Guarantees

Most brokers guarantee order within a partition/shard, not globally. If order matters, route related events to the same partition using a partition key (user_id, order_id).

**Kafka:** Messages in the same partition are ordered. Use the same key for related events.
**SQS FIFO:** Guarantees order within a message group ID.
**RabbitMQ:** Order is maintained per queue, but competing consumers may process out of order.

### Eventual Consistency

Event-driven systems are eventually consistent. After a write, reads may not immediately reflect the change. This is acceptable for most use cases (social media feeds, analytics). For critical consistency (financial transactions), use synchronous calls or sagas with compensating transactions.

## Anti-Patterns

**Event chain explosion:** One event triggers another, which triggers another, creating deep dependency chains. Limit event chains to 2-3 hops. Beyond that, consider orchestration.

**Shared database:** Services reading/writing the same database defeat the purpose of decoupling. Each service owns its data.

**Synchronous event processing:** Blocking until all consumers finish. Use async processing; producers shouldn't wait for consumers.

**Over-eventing:** Publishing events for every trivial state change. Events should represent meaningful business occurrences.

**Ignoring idempotency:** Assuming exactly-once delivery. Always design for at-least-once.

**No dead-letter queue:** Failed messages disappear silently. Always configure DLQs.

## When to Use EDA

**Good fit:**
- Microservices needing loose coupling
- Real-time data processing (analytics, monitoring)
- High scalability requirements
- Systems requiring audit trails
- IoT and streaming data

**Poor fit:**
- Simple CRUD applications
- Strong consistency requirements (banking transactions without sagas)
- Small teams unfamiliar with async patterns
- Low-latency, synchronous user interactions (use request-response)