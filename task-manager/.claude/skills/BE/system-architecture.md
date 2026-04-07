---
name: backend-system-architecture
description: >
  Design backend architectures including monolithic, modular monolith, and microservices patterns. Use this for backend architecture design, system architecture planning, service decomposition, choosing between monolithic vs microservices, event-driven architecture, message queues, queue systems (Kafka, RabbitMQ), scalability planning, horizontal vs vertical scaling, distributed systems, API design, service boundaries, CQRS, event sourcing, pub-sub patterns, system design interviews, architecture tradeoffs, or whenever discussing how to structure backend systems, scale services, or decompose applications into components.
---

# Backend System Architecture

## Core Architectural Patterns

### Monolithic Architecture

All functionality in a single deployable unit—one codebase, one database, one deployment.

**When it makes sense:**
- Early-stage products where speed to market matters most
- Small teams (< 10 engineers) without distributed systems expertise
- Simple domains without clear service boundaries
- Applications where ACID transactions across the entire system are critical

**Why it works:** Lower operational overhead, simpler debugging, faster development cycles, easier testing. Most startups should start here—you can extract services later when domain boundaries become clear.

**Where it breaks:** Scaling requires replicating the entire stack (resource-heavy), changes require full redeployment, technology lock-in to a single stack, and growing codebases become harder to reason about.

### Modular Monolith

Structured like microservices (clear module boundaries, defined interfaces) but deployed as one unit.

**Why it's underrated:** You get 80% of microservices benefits—clean boundaries, independent development within modules, easier testing of isolated components—with 20% of the operational cost. No distributed system complexity, no network calls between modules, simpler deployment pipeline.

**Implementation pattern:** Organize by domain modules with explicit interfaces. Each module owns its data access layer. Enforce boundaries through architecture tests or linting rules. Modules communicate through well-defined APIs, not direct database access.

**Transition path:** When a module needs independent scaling or deployment, extract it as a microservice. The clear boundaries make this extraction far less risky than splitting a tangled monolith.

### Microservices Architecture

Independent services, each owning its data, communicating via APIs or asynchronous messages.

**When it makes sense:**
- Teams > 20 engineers where coordination overhead exceeds distribution overhead
- Clear service boundaries aligned with business capabilities
- Different scaling requirements per service (e.g., search needs 10x capacity of checkout)
- Organizational need for independent deployment and technology choices

**Core tradeoff:** You trade code complexity for operational complexity. Debugging spans multiple services, data consistency becomes eventual, network failures are routine, and you need sophisticated observability.

**Critical success factors:**
- Strong DevOps culture and tooling (CI/CD, containerization, orchestration)
- Service discovery, API gateways, and circuit breakers in place
- Distributed tracing and centralized logging from day one
- Clear ownership model—each service has a team responsible for it

**Common failure mode:** Distributed monolith—services that share databases, require synchronous cross-service calls for every operation, or need coordinated deployments. This gives you all the complexity of microservices with none of the benefits.

## Event-Driven Architecture

Services communicate by publishing events rather than direct calls. Decouples producers from consumers—producers don't know who (if anyone) will react to an event.

**Why it matters:** Services can evolve independently, new consumers can be added without modifying producers, and systems become more resilient (if a consumer is down, events queue up rather than causing cascading failures).

**Two primary models:**

**Pub-Sub (Publish-Subscribe):** Events are broadcast to all subscribers. Each subscriber gets a copy. Messages don't persist after delivery. Good for notifications, real-time updates, fan-out patterns. Tools: Redis Pub/Sub, AWS SNS, Azure Event Grid.

**Event Streaming:** Events are written to a durable, ordered log. Consumers read from any position in the log. Events persist and can be replayed. Good for event sourcing, audit trails, reprocessing historical data. Tools: Kafka, AWS Kinesis, Pulsar.

**Tradeoffs:**
- **Debugging complexity:** Following a request across async events requires correlation IDs and distributed tracing. Build this in from the start—retrofitting observability is painful.
- **Eventual consistency:** You can't guarantee immediate consistency across services. Design for it: idempotent consumers, compensating transactions, saga patterns.
- **Ordering challenges:** Events may arrive out of order, especially across partitions. Use sequence numbers, timestamps, or design to be order-independent when possible.

**When NOT to use:** Simple CRUD applications, workflows requiring immediate consistency, or when your team lacks experience with distributed systems. Synchronous APIs are often simpler and sufficient.

## Queue Systems

### Message Queues (RabbitMQ, AWS SQS, Azure Service Bus)

**Pattern:** Producer sends message to queue → Consumer pulls message → Queue deletes message after acknowledgment.

**Characteristics:**
- Messages consumed once (competing consumers pattern)
- Flexible routing (topic exchanges, direct routing, fanout)
- Delivery guarantees (at-least-once, exactly-once with deduplication)
- Good for task distribution, background jobs, decoupling services

**Example use case:** E-commerce order processing—order service publishes to queue, payment service consumes, sends payment event, inventory service consumes and reserves stock. Each service scales independently based on queue depth.

### Event Streaming (Kafka, Pulsar)

**Pattern:** Producer writes to topic partition → Multiple consumers read independently → Events persist for configured retention.

**Characteristics:**
- Partitioning for parallelism (scale consumers by partition count)
- Consumer groups for load distribution
- Replayability (consumers can reprocess historical events)
- High throughput (millions of events/second)
- Good for event sourcing, real-time analytics, change data capture

**Kafka-specific insights:**
- Topics split into partitions, each partition is an ordered log
- Consumer groups: each partition assigned to one consumer in the group (parallel processing)
- Replication for fault tolerance (partition replicas across brokers)
- Compacted topics for maintaining latest state per key

**When to choose streaming over queuing:** You need event replay, multiple independent consumers of the same events, high throughput (> 100k events/sec), or event sourcing patterns.

## Scalability Approaches

### Horizontal vs Vertical Scaling

**Vertical (scale up):** Add more CPU/RAM to existing machines. Simple but has limits—eventually you hit hardware ceilings, and single points of failure remain.

**Horizontal (scale out):** Add more machines. Requires designing for distribution (stateless services, shared-nothing architecture, load balancing). More complex but scales indefinitely and improves fault tolerance.

**Modern approach:** Horizontal scaling for application tier (stateless services), vertical for databases until sharding becomes necessary. Use managed services (RDS, DynamoDB) to defer database scaling complexity.

### Scaling Patterns

**Database scaling:**
- Read replicas for read-heavy workloads (route reads to replicas, writes to primary)
- Caching layer (Redis, Memcached) to reduce database load
- Connection pooling to limit database connections
- Sharding when single-database limits are reached (partition data across multiple databases)

**Service scaling:**
- Auto-scaling based on metrics (CPU, memory, request rate, queue depth)
- Predictive scaling for known traffic patterns (scale up before load arrives)
- Independent scaling per service in microservices (only scale what needs it)

**Bottleneck identification:** Profile before optimizing. Common bottlenecks: database queries (add indexes, caching), synchronous API calls (make async, add circuit breakers), CPU-bound processing (horizontal scaling, optimize algorithms), network I/O (batching, compression).

## Service Decomposition

### Finding Service Boundaries

**Domain-Driven Design approach:** Identify bounded contexts—areas with distinct business language and rules. Each bounded context is a candidate service. Example: in e-commerce, "Product Catalog," "Order Management," "Inventory," and "Shipping" are distinct contexts.

**Heuristics for decomposition:**
- Different scaling needs (search vs checkout)
- Different data models or storage requirements (SQL vs graph database)
- Different change frequencies (pricing rules change often, user profiles don't)
- Team boundaries (services owned by different teams)
- Independent deployment value (can release this feature without coordinating with others)

**Warning signs of wrong boundaries:**
- Services that always deploy together
- Frequent cross-service joins or distributed transactions
- Services that share the same database tables
- Chatty communication (hundreds of calls per user request)

**Start coarse, refine later:** Begin with fewer, larger services. Split when you have evidence (scaling needs, team growth, deployment bottlenecks). Premature decomposition creates distributed monoliths.

## Advanced Patterns

### CQRS (Command Query Responsibility Segregation)

Separate models for writes (commands) and reads (queries). Write model optimized for consistency and business rules. Read model optimized for query performance.

**When it helps:** Complex domains with different read/write patterns, high read-to-write ratios, or when you need multiple read representations of the same data (e.g., relational for reports, document for search).

**Tradeoff:** Increased complexity—you maintain two models and synchronize them (usually via events). Only use when query optimization or domain complexity justifies it.

### Event Sourcing

Store state as a sequence of events rather than current state. Rebuild current state by replaying events.

**Benefits:** Complete audit trail, temporal queries ("what was the state on date X?"), replay events to fix bugs or generate new projections.

**Challenges:** Event schema evolution (old events must remain processable), increased storage, complexity in querying current state (need to replay or maintain projections).

**Good fit:** Financial systems (audit requirements), domains where history matters (medical records, legal), or when you need to support multiple downstream views of the same data.

## Architecture Decision Framework

**Questions to ask:**

1. **Team size and expertise:** < 10 engineers with limited distributed systems experience? Start with monolith or modular monolith.

2. **Domain clarity:** Can you clearly define service boundaries? If not, don't split yet—boundaries will become clearer as the domain matures.

3. **Scaling requirements:** Do different parts of the system have vastly different scaling needs? This justifies microservices. Otherwise, scale the monolith.

4. **Deployment independence:** Do you need to deploy parts of the system independently? If yes, microservices or modular monolith with feature flags.

5. **Consistency requirements:** Do you need immediate consistency across the entire system? Monolith with ACID transactions is simpler. Can you tolerate eventual consistency? Event-driven microservices become viable.

6. **Operational maturity:** Do you have CI/CD, containerization, orchestration (Kubernetes), distributed tracing, centralized logging? If not, microservices will be painful.

**Evolution path:** Monolith → Modular Monolith → Extract high-value services → Full microservices (if justified). Each step should be driven by concrete needs, not trends.

## Common Pitfalls

**Microservices too early:** Premature distribution adds complexity without benefits. Most systems don't need microservices until they have 20+ engineers or clear independent scaling needs.

**Distributed monolith:** Services that share databases or require synchronous calls for every operation. You get microservices complexity without the benefits. Fix: proper service boundaries, async communication, each service owns its data.

**Ignoring network realities:** Network calls fail, have latency, and have bandwidth limits. Design for failure: timeouts, retries with exponential backoff, circuit breakers, bulkheads.

**Event-driven without observability:** You can't debug async systems without correlation IDs, distributed tracing, and centralized logging. Build this in from day one.

**Scaling without profiling:** Adding servers won't help if the database is the bottleneck. Measure, identify bottlenecks, then optimize the constraint.

**Over-engineering for hypothetical scale:** Design for 10x your current scale, not 1000x. You'll learn and refactor along the way. Premature optimization wastes time and creates complexity you don't need yet.