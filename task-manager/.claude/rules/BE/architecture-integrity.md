---
name: backend-architecture-integrity
description: >
  Enforce backend architectural integrity with layered architecture principles
  (controller-service-repository pattern), separation of concerns, modularity
  requirements, and detection of architecture smells. Use this when reviewing
  backend code, building APIs, designing server-side systems, working with
  layered architectures, enforcing clean architecture, code review for backend
  services, refactoring backend systems, establishing backend coding standards,
  or whenever architectural quality matters in server-side development. Applies
  to REST APIs, GraphQL backends, microservices, monoliths, Spring Boot, Node.js,
  Django, Rails, ASP.NET, and any backend framework.
---

# Backend Architecture Integrity

This skill enforces architectural integrity in backend systems using layered architecture principles, clear separation of concerns, and detection of common architecture smells.

## Core Architecture: Controller-Service-Repository

The three-layer pattern provides clear boundaries and testability:

**Controller Layer** (Presentation/API)
- Handles HTTP concerns: routing, request/response, status codes
- Validates input format (not business rules)
- Assembles responses from service results
- Thin by design—no business logic, no direct data access
- Example responsibilities: parse request body, call service method, return 200/400/500

**Service Layer** (Business Logic)
- Contains all business rules and domain logic
- Orchestrates workflows across multiple repositories or external services
- Enforces invariants and business constraints
- Transaction boundaries live here
- Reusable across multiple controllers or interfaces
- Example: "Can this user place this order?" logic, calculating totals, coordinating payment + inventory updates

**Repository Layer** (Data Access)
- Abstracts database operations behind interfaces
- Encapsulates queries, ORM usage, and data mapping
- Returns domain objects, not raw database records
- Isolates persistence technology from business logic
- Example: findById, save, findByEmail—no business logic

### Communication Rules

- Controllers call Services (never Repositories directly)
- Services call Repositories and other Services
- Repositories call only database/ORM—no business logic
- Data flows: Request → Controller → Service → Repository → Database
- Response flows: Database → Repository → Service → Controller → Response

### Why This Matters

Layering isolates change. When you swap databases, only repositories change. When business rules evolve, only services change. When you add a GraphQL API alongside REST, controllers change but services stay identical. Each layer is independently testable with mocks.

## Separation of Concerns

### What Belongs Where

**Controllers should:**
- Validate request format (JSON structure, required fields present)
- Map DTOs to domain objects
- Handle authentication/authorization tokens
- Return appropriate HTTP status codes
- Log request/response at API boundary

**Controllers should NOT:**
- Contain if/else business logic
- Directly query databases or call ORMs
- Perform calculations or transformations
- Manage transactions
- Call external APIs for business data

**Services should:**
- Implement business rules ("orders over $100 get free shipping")
- Coordinate multiple repository calls
- Enforce domain invariants
- Handle transaction boundaries (@Transactional)
- Orchestrate external service calls
- Perform domain calculations

**Services should NOT:**
- Parse HTTP requests or build HTTP responses
- Contain SQL queries or ORM code
- Know about JSON, XML, or serialization formats
- Reference framework-specific HTTP objects (HttpServletRequest, etc.)

**Repositories should:**
- Execute queries and return domain objects
- Map database rows to entities
- Handle database-specific error translation
- Provide intent-revealing method names (findActiveUsersByRole, not executeQuery)

**Repositories should NOT:**
- Validate business rules
- Perform calculations
- Call other repositories or services
- Contain complex conditional logic

### The "If Statement" Heuristic

If code contains conditional logic about the domain (not just null checks or format validation), it's probably business logic and belongs in a Service. Controllers and Repositories should be nearly branch-free.

## Modularity Requirements

### Loose Coupling

- Layers depend on interfaces, not concrete implementations
- Use dependency injection—never instantiate dependencies with `new` inside classes
- Services shouldn't know which database or ORM is used
- Controllers shouldn't know how services implement logic
- Repository interfaces belong in the service layer; implementations in infrastructure

### High Cohesion

- Group related operations: UserService handles user domain logic, OrderService handles orders
- Avoid "Manager" or "Helper" classes that do unrelated things
- Each service should have a clear, singular purpose
- If a service grows beyond ~10 methods, consider splitting by subdomain

### Interface Segregation

- Repositories expose only methods clients need
- Don't create generic "CRUD" repositories that expose update() when entities are immutable
- Services should have focused interfaces—split large services into multiple smaller ones

### Dependency Direction

Dependencies flow inward:
```
Controller → Service → Repository → Database
```
Never reverse this. Repositories must not call services. Services must not call controllers.

## Architecture Smells (Forbidden Patterns)

### God Object / Blob
**Symptom:** One service or controller with 20+ methods, 500+ lines, handling multiple unrelated domains.
**Why it's bad:** Becomes a bottleneck for changes, impossible to test in isolation, violates single responsibility.
**Fix:** Split by domain boundaries. UserService, OrderService, PaymentService—not ApplicationService.

### Anemic Domain Model
**Symptom:** Domain objects are just data holders (getters/setters), all logic in services.
**Why it's acceptable in layered architecture:** This pattern is common and often appropriate in transaction scripts. Services coordinate behavior.
**When it's a smell:** If services are just pass-through wrappers with no logic, you're adding layers without value.

### Feature Envy
**Symptom:** A service method spends most of its time calling getters on another domain object and making decisions about it.
**Fix:** Move that logic to the object it operates on, or to a service responsible for that domain.

### Shotgun Surgery
**Symptom:** A single business rule change requires editing 5 controllers, 3 services, 4 repositories.
**Why it's bad:** Fragile, error-prone, indicates duplication or misplaced responsibility.
**Fix:** Centralize the rule in one service method called by all clients.

### Business Logic in Controllers
**Symptom:** Controllers contain if/else based on domain rules, calculations, or workflow orchestration.
**Example:** `if (order.total > 100) { applyDiscount(); }`
**Why it's bad:** Untestable without HTTP mocking, unreusable (can't call from batch jobs), violates separation.
**Fix:** Move to service layer immediately.

### Direct Repository Access from Controllers
**Symptom:** Controller calls `userRepository.findById()` directly.
**Why it's bad:** Bypasses business logic, makes transactions unclear, couples controller to data layer.
**Fix:** Always go through a service, even if it's a thin wrapper initially.

### Repository with Business Logic
**Symptom:** Repository methods like `findEligibleUsersForPromotion()` that encode business rules in queries.
**Why it's nuanced:** Sometimes query optimization requires pushing logic to SQL. Acceptable if the repository method name clearly expresses intent and the service layer calls it.
**When it's a smell:** If the repository is making business decisions (if/else on domain state), extract that to service.

### Leaky Abstractions
**Symptom:** Service layer references JPA entities, SQL exceptions, or HTTP status codes.
**Why it's bad:** Couples business logic to infrastructure, makes testing hard, prevents technology changes.
**Fix:** Use domain objects in services. Map at boundaries (controller and repository edges).

### Circular Dependencies
**Symptom:** ServiceA calls ServiceB, ServiceB calls ServiceA.
**Why it's bad:** Indicates unclear boundaries, makes reasoning about flow impossible, often causes runtime errors.
**Fix:** Extract shared logic to a third service, or rethink domain boundaries.

### Transaction Script Overload
**Symptom:** Every service method is a long procedural script with no reusable pieces.
**Why it's bad:** Duplication, hard to test individual steps, difficult to compose behaviors.
**Fix:** Break scripts into smaller, reusable service methods. Extract common patterns.

### Primitive Obsession in Services
**Symptom:** Passing around strings, ints, and maps instead of domain objects.
**Example:** `createOrder(String userId, String productId, int quantity)` instead of `createOrder(Order order)`
**Why it's bad:** Loses type safety, spreads validation everywhere, error-prone.
**Fix:** Use value objects and domain entities.

### Fat Models, Thin Services (inverted)
**Symptom:** Domain objects contain database access code, HTTP calls, or framework dependencies.
**Why it's bad:** Violates single responsibility, makes domain objects untestable, couples domain to infrastructure.
**Fix:** Keep models pure. Move infrastructure to repositories, logic to services.

## Enforcement Strategy

### Code Review Checklist

- [ ] Does this controller contain any if/else on domain state? → Move to service
- [ ] Does this controller call a repository directly? → Route through service
- [ ] Does this service contain HTTP status codes or request parsing? → Move to controller
- [ ] Does this service directly use ORM annotations or SQL? → Move to repository
- [ ] Does this repository contain business logic? → Move to service
- [ ] Are dependencies injected or hardcoded with `new`? → Inject
- [ ] Does this class have more than one reason to change? → Split
- [ ] Can I test this class without mocking HTTP or database? → If not, refactor

### Refactoring Priorities

1. **Critical:** Business logic in controllers—immediate extraction
2. **High:** Direct repository access from controllers—add service layer
3. **High:** Circular dependencies—redesign boundaries
4. **Medium:** God objects—split incrementally
5. **Medium:** Leaky abstractions—introduce DTOs/mappers
6. **Low:** Anemic models—acceptable if services have clear logic

### Testing as Architecture Validation

If you can't write a unit test for a service without starting a web server or database, your architecture is wrong. Services should be testable with mocked repositories. Controllers should be testable with mocked services. Repositories should be testable with in-memory databases or test containers.

## When to Bend the Rules

### Acceptable Exceptions

- **Simple CRUD endpoints:** For trivial read/write with zero business logic, a thin service that just delegates to repository is fine. Don't add layers for the sake of layers.
- **Read-only queries:** A controller calling a read-only repository for a simple lookup (if no business rules apply) can be acceptable in small apps. Document why.
- **Performance-critical paths:** Sometimes query optimization requires repository methods that embed domain knowledge. Make the trade-off explicit and document it.

### Evolutionary Architecture

Start simple. Small apps don't need every layer. But when you add the service layer, do it properly—don't half-commit. A common path:
1. Start: Controllers + Repositories (no business logic yet)
2. Add: Service layer when first business rule appears
3. Refactor: Extract all business logic to services
4. Maintain: Keep boundaries clean as system grows

The key is recognizing when you've crossed the threshold where layering pays off, then committing fully.

## Framework-Specific Notes

- **Spring Boot:** Use `@RestController`, `@Service`, `@Repository` annotations to mark layers. Leverage constructor injection.
- **Node.js/Express:** Organize by folders: `/controllers`, `/services`, `/repositories`. Use dependency injection libraries (tsyringe, InversifyJS) or manual injection.
- **Django:** Views = Controllers, Services are custom (create `/services`), Models + Managers = Repositories.
- **Rails:** Controllers stay thin, Services are POROs in `/services`, Models handle data access but avoid fat models.
- **ASP.NET Core:** Controllers, Services (in `/Services`), Repositories (with interfaces in domain layer, implementations in infrastructure).

Regardless of framework, the principles remain: Controllers handle HTTP, Services handle business logic, Repositories handle data. Enforce these boundaries ruthlessly.