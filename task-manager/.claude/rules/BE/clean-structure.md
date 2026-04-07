---
name: clean-backend-structure
description: >
  Design and maintain clean, scalable backend folder structures with consistent
  naming conventions, clear module organization, comprehensive documentation
  requirements, and standardized API response formats. Use this skill whenever
  working on backend architecture, organizing backend code, structuring
  Node.js/Express/NestJS/Django/FastAPI projects, refactoring backend services,
  setting up new backend projects, reviewing backend code organization,
  implementing API standards, creating REST APIs, designing microservices, or
  when the user mentions backend folder structure, code organization, project
  structure, maintainability, scalability, separation of concerns, clean
  architecture, layered architecture, MVC, controllers, services, repositories,
  models, routes, middleware, utilities, API responses, error handling, or
  backend best practices.
---

# Clean Backend Structure

## Overview

A well-organized backend structure serves one purpose: make it obvious where code lives and why. When folder structure mirrors intent, developers find what they need in seconds, add features without breaking existing code, and onboard new team members in hours instead of weeks.

This skill focuses on four pillars: folder organization that scales, naming conventions that communicate purpose, documentation that stays current, and API responses that clients can rely on.

## Core Principles

### Separation of Concerns

Each layer has one job. Controllers handle HTTP—they parse requests, validate shape, and return responses. Services contain business logic—pricing calculations, permission checks, workflow orchestration. Repositories abstract data access—queries, transactions, caching. Models define structure and invariants.

When a pricing rule lives in a controller, it's invisible to anyone calling that logic from a background job. When validation lives in a repository, you can't reuse it across different data sources. Keep layers focused.

### Read-Optimized Structure

Code is read far more often than written. Optimize for the question "where does X live?" not "how quickly can I add this file?" A feature-based structure (`/users`, `/orders`) beats a type-based structure (`/controllers`, `/services`) when projects grow beyond 10 endpoints because developers think in features, not file types.

For smaller projects (< 20 files), type-based works fine. For larger systems, group by feature or domain.

### Dependencies Flow Inward

Outer layers depend on inner layers, never the reverse. Controllers depend on services. Services depend on repositories. Domain models depend on nothing. This keeps business logic portable—you can swap Express for Fastify, Postgres for MongoDB, without rewriting core rules.

Use dependency injection to invert infrastructure dependencies. Services declare interfaces; repositories implement them.

## Folder Structure Patterns

### Small Projects (< 20 files): Type-Based

```
src/
├── controllers/     # HTTP handlers
│   ├── user.controller.js
│   └── order.controller.js
├── services/        # Business logic
│   ├── user.service.js
│   └── order.service.js
├── repositories/    # Data access
│   ├── user.repository.js
│   └── order.repository.js
├── models/          # Data structures, schemas
│   ├── user.model.js
│   └── order.model.js
├── middleware/      # Auth, validation, logging
│   ├── auth.middleware.js
│   └── error.middleware.js
├── routes/          # Route definitions
│   ├── user.routes.js
│   └── index.js
├── utils/           # Pure helpers
│   └── date.utils.js
├── config/          # Environment, DB config
│   └── database.js
└── app.js           # Entry point
```

Works when the entire team can hold the domain model in their heads.

### Medium-Large Projects: Feature-Based

```
src/
├── users/
│   ├── user.controller.js
│   ├── user.service.js
│   ├── user.repository.js
│   ├── user.model.js
│   ├── user.routes.js
│   └── dto/
│       ├── create-user.dto.js
│       └── update-user.dto.js
├── orders/
│   ├── order.controller.js
│   ├── order.service.js
│   ├── order.repository.js
│   ├── order.model.js
│   └── order.routes.js
├── shared/
│   ├── middleware/
│   ├── utils/
│   └── interfaces/
├── config/
└── app.js
```

Scales to hundreds of files. Each feature is self-contained. New developers can contribute to one feature without understanding the entire codebase.

### Clean Architecture (Complex Domains)

```
src/
├── domain/              # Core business logic, zero dependencies
│   ├── entities/
│   │   ├── user.entity.js
│   │   └── order.entity.js
│   ├── value-objects/
│   └── interfaces/
│       └── repositories/
├── application/         # Use cases, orchestration
│   ├── use-cases/
│   │   ├── create-user.usecase.js
│   │   └── place-order.usecase.js
│   └── dto/
├── infrastructure/      # External concerns
│   ├── database/
│   │   └── repositories/
│   ├── http/
│   │   └── controllers/
│   └── external-services/
└── main.js
```

Use when business rules are complex, change frequently, or must survive technology shifts. Overkill for CRUD-heavy APIs.

## Naming Conventions

### Files

- **Controllers**: `resource.controller.js` — `user.controller.js`, `order.controller.js`
- **Services**: `resource.service.js` — `user.service.js`, `payment.service.js`
- **Repositories**: `resource.repository.js` — `user.repository.js`
- **Models**: `resource.model.js` — `user.model.js`
- **Routes**: `resource.routes.js` — `user.routes.js`
- **Middleware**: `purpose.middleware.js` — `auth.middleware.js`, `validate.middleware.js`
- **Utils**: `domain.utils.js` — `date.utils.js`, `string.utils.js`
- **DTOs**: `action-resource.dto.js` — `create-user.dto.js`, `update-order.dto.js`

Kebab-case for files, PascalCase for classes, camelCase for functions and variables.

### Routes and Endpoints

- **Resource names**: Plural nouns — `/users`, `/orders`, `/products`
- **No verbs in URLs**: HTTP methods are the verbs — `POST /users`, not `/createUser`
- **Hierarchical relationships**: `/users/123/orders`, `/orders/456/items`
- **Query params for filters**: `/users?role=admin&active=true`
- **Versioning**: `/v1/users`, `/v2/users` or `api.example.com/v1/users`

### Code Identifiers

- **Boolean variables**: `isActive`, `hasPermission`, `canEdit`
- **Collections**: Plural — `users`, `orders`, `items`
- **Single items**: Singular — `user`, `order`, `item`
- **Constants**: `UPPER_SNAKE_CASE` — `MAX_LOGIN_ATTEMPTS`, `DEFAULT_PAGE_SIZE`

## Documentation Requirements

### README.md (Root)

- **Project purpose**: One-sentence description
- **Setup instructions**: Dependencies, environment variables, database setup
- **Running locally**: Development server, tests, linting
- **Project structure**: Brief overview of folder organization
- **Key conventions**: Naming, branching, commit messages

### Inline Comments

Comment the **why**, not the **what**. Code shows what it does; comments explain why it does it that way.

```javascript
// Good: Explains non-obvious reasoning
// Retry logic needed because payment gateway returns 500 on timeout
// but actually processes the transaction
await retryWithIdempotency(processPayment, orderId);

// Bad: Restates the code
// Call the process payment function
await processPayment(orderId);
```

### API Documentation

Document every public endpoint with:

- **Method and path**: `POST /v1/users`
- **Description**: What it does, when to use it
- **Auth requirements**: Public, API key, OAuth scope
- **Request body**: Example with all fields, types, constraints
- **Response**: Success example (200, 201) and common errors (400, 404, 500)
- **Status codes**: What each means in this context

Use OpenAPI/Swagger for auto-generated docs when possible. Keep examples realistic—actual data from staging, not `foo`/`bar`.

### Module Documentation

Each major module or feature should have a README explaining:

- **Purpose**: What problem this solves
- **Key concepts**: Domain terms, workflows
- **Entry points**: Where to start reading
- **Dependencies**: What it relies on, what relies on it

## API Response Standards

### Success Responses

**Single Resource (200 OK)**

```json
{
  "id": 123,
  "email": "user@example.com",
  "name": "Jane Doe",
  "createdAt": "2026-04-07T10:30:00Z"
}
```

**Collection (200 OK)**

```json
{
  "data": [
    { "id": 1, "name": "Item 1" },
    { "id": 2, "name": "Item 2" }
  ],
  "pagination": {
    "page": 2,
    "pageSize": 20,
    "totalItems": 150,
    "totalPages": 8
  }
}
```

**Creation (201 Created)**

```json
// Response includes Location header: /v1/users/123
{
  "id": 123,
  "email": "user@example.com",
  "createdAt": "2026-04-07T10:30:00Z"
}
```

**Update (200 OK or 204 No Content)**

Return updated resource (200) or nothing (204) depending on client needs.

**Deletion (204 No Content)**

No response body. Status code confirms success.

### Error Responses

Use RFC 9457 Problem Details format for consistency:

```json
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 400,
  "detail": "One or more fields failed validation",
  "instance": "/v1/users",
  "errors": [
    {
      "field": "email",
      "message": "Email address is required",
      "code": "REQUIRED_FIELD"
    },
    {
      "field": "password",
      "message": "Password must be at least 8 characters",
      "code": "MIN_LENGTH"
    }
  ]
}
```

**Common Status Codes**

- **200 OK**: Successful GET, PUT, PATCH
- **201 Created**: Successful POST creating a resource
- **204 No Content**: Successful DELETE or update with no response body
- **400 Bad Request**: Invalid input, validation failure
- **401 Unauthorized**: Missing or invalid authentication
- **403 Forbidden**: Authenticated but lacks permission
- **404 Not Found**: Resource doesn't exist
- **409 Conflict**: Request conflicts with current state (duplicate, version mismatch)
- **422 Unprocessable Entity**: Syntactically valid but semantically invalid
- **429 Too Many Requests**: Rate limit exceeded
- **500 Internal Server Error**: Unexpected server failure
- **503 Service Unavailable**: Temporary overload or maintenance

### Headers

**Response Headers**

- `Content-Type: application/json` — Always set for JSON responses
- `Location: /v1/users/123` — On 201 Created, points to new resource
- `Cache-Control: public, max-age=3600` — Enable caching where appropriate
- `X-RateLimit-Limit: 1000` — Rate limit ceiling
- `X-RateLimit-Remaining: 847` — Requests left
- `X-RateLimit-Reset: 1640995200` — Unix timestamp when limit resets

**Request Headers**

- `Content-Type: application/json` — Required for POST/PUT/PATCH with body
- `Accept: application/json` — Preferred response format
- `Authorization: Bearer <token>` — Authentication

### Field Naming

Choose **camelCase** or **snake_case** and use it everywhere. JavaScript ecosystems prefer camelCase; Python prefers snake_case. Never mix.

```json
// camelCase (JavaScript, TypeScript)
{ "userId": 123, "createdAt": "2026-04-07T10:30:00Z" }

// snake_case (Python, Ruby)
{ "user_id": 123, "created_at": "2026-04-07T10:30:00Z" }
```

### Dates and Times

Always use ISO 8601 with UTC timezone: `2026-04-07T10:30:00Z`. Never use Unix timestamps in JSON—they're ambiguous (seconds vs milliseconds) and harder for humans to read.

## Key Decisions

### When to Use Feature-Based Structure

- **More than 20 files**: Type-based becomes hard to navigate
- **Multiple developers**: Features can be worked on independently
- **Clear domain boundaries**: Users, orders, products are distinct

### When to Use Clean Architecture

- **Complex business rules**: Domain logic changes frequently
- **Long-term project**: Will outlive current technology stack
- **Multiple interfaces**: Same logic serves REST API, GraphQL, CLI, background jobs

Don't use it for simple CRUD APIs—the overhead isn't worth it.

### When to Add a Service Layer

If a controller is doing more than:
1. Validate request shape
2. Call one method
3. Return response

...then extract the logic into a service. Services coordinate between repositories, apply business rules, and handle transactions.

### When to Create a Repository

When you need to:
- Swap databases (SQL to NoSQL)
- Mock data access in tests
- Centralize query logic (avoid duplicating complex queries)

For tiny projects with one database and no testing, direct ORM calls from services are fine.

## Common Mistakes

### Fat Controllers

Controllers that contain business logic, validation, and data access. Impossible to test without HTTP. Impossible to reuse from background jobs. Move logic to services.

### Anemic Services

Services that just pass data from controller to repository with no transformation or logic. If the service doesn't add value, skip it—controllers can call repositories directly.

### God Services

A single `UserService` with 50 methods handling authentication, profile updates, permissions, notifications, and analytics. Split by responsibility: `AuthService`, `UserProfileService`, `PermissionService`.

### Inconsistent Naming

Mixing `getUser`, `fetchOrder`, `retrieveProduct` for the same operation. Pick one verb per operation type and use it everywhere: `get`, `create`, `update`, `delete`.

### Missing Error Context

Returning `{ "error": "Bad request" }` with no details. Clients can't fix the problem. Always include field-level errors, codes, and actionable messages.

### Leaking Implementation Details

Exposing database column names (`user_id`) instead of domain terms (`userId`). Returning internal error stack traces to clients. API responses should reflect the domain model, not the database schema.

## Testing Considerations

- **Controllers**: Test HTTP concerns—status codes, headers, request parsing. Mock services.
- **Services**: Test business logic with real domain objects. Mock repositories.
- **Repositories**: Integration tests with real database (or in-memory equivalent).
- **Models**: Unit tests for validation, invariants, computed properties.

Structure tests to mirror source structure: `tests/users/user.service.test.js` for `src/users/user.service.js`.

## Examples

### Controller (Express)

```javascript
// src/users/user.controller.js
const { createUserSchema } = require('./dto/create-user.dto');
const UserService = require('./user.service');

class UserController {
  constructor(userService) {
    this.userService = userService;
  }

  async createUser(req, res, next) {
    try {
      const { error, value } = createUserSchema.validate(req.body);
      if (error) {
        return res.status(400).json({
          type: 'validation-error',
          errors: error.details.map(d => ({
            field: d.path.join('.'),
            message: d.message
          }))
        });
      }

      const user = await this.userService.createUser(value);
      res.status(201)
         .location(`/v1/users/${user.id}`)
         .json(user);
    } catch (err) {
      next(err);
    }
  }
}

module.exports = UserController;
```

### Service

```javascript
// src/users/user.service.js
class UserService {
  constructor(userRepository, emailService) {
    this.userRepository = userRepository;
    this.emailService = emailService;
  }

  async createUser(data) {
    // Business logic: check for duplicates
    const existing = await this.userRepository.findByEmail(data.email);
    if (existing) {
      throw new ConflictError('User with this email already exists');
    }

    // Hash password, create user
    const hashedPassword = await bcrypt.hash(data.password, 10);
    const user = await this.userRepository.create({
      ...data,
      password: hashedPassword
    });

    // Side effect: send welcome email
    await this.emailService.sendWelcome(user.email);

    return user;
  }
}

module.exports = UserService;
```

### Repository

```javascript
// src/users/user.repository.js
class UserRepository {
  constructor(db) {
    this.db = db;
  }

  async findByEmail(email) {
    return this.db('users').where({ email }).first();
  }

  async create(userData) {
    const [id] = await this.db('users').insert(userData).returning('id');
    return this.findById(id);
  }

  async findById(id) {
    return this.db('users').where({ id }).first();
  }
}

module.exports = UserRepository;
```

## Summary

- **Structure**: Feature-based for medium+ projects, type-based for small ones, clean architecture for complex domains
- **Naming**: Consistent conventions—`resource.type.js`, plural nouns for routes, camelCase or snake_case for fields
- **Documentation**: README for setup, inline comments for why, OpenAPI for endpoints, module READMEs for complex features
- **API Responses**: RFC 9457 for errors, consistent field naming, ISO 8601 dates, meaningful status codes, pagination for collections
- **Layers**: Controllers handle HTTP, services handle logic, repositories handle data, models define structure
- **Dependencies**: Flow inward—outer layers depend on inner, never reverse

A clean structure isn't about perfection—it's about consistency. Pick conventions, document them, and enforce them in code review. The goal is for any developer to find what they need in under 30 seconds.
