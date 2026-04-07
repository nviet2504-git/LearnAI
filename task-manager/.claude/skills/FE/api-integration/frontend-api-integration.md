---
name: frontend-api-integration
description: >
  Generate TypeScript service layers for frontend API integration. Use this when building API clients, service layers, HTTP request handlers, data fetching logic, or any task involving calling external APIs from the frontend. Covers error handling, retry logic with exponential backoff, pagination strategies, token refresh flows, request/response interceptors, TypeScript schema generation from OpenAPI specs using Zod, API client architecture, and resilient API communication patterns. Apply this for REST API integration, GraphQL clients, API wrappers, data fetching utilities, or any frontend code that communicates with backend services.
---

# Frontend API Integration

Generate production-ready TypeScript service layers for frontend applications that communicate with backend APIs. This skill focuses on clean architecture, type safety, resilience, and maintainability.

## Core Architecture

### Service Layer Pattern

Structure API logic in three layers:

- **HTTP Client** — Base client with interceptors, auth, retry logic
- **Service Layer** — Domain-specific methods (UserService, OrderService)
- **Type Layer** — Request/response types, Zod schemas for runtime validation

Keep HTTP concerns (headers, retries, errors) in the client. Keep business logic (data transformation, caching) in services. Keep validation schemas separate and reusable.

### Base HTTP Client

Build a configurable base client that handles:

- Request/response interceptors for auth tokens
- Automatic retry with exponential backoff for transient failures
- Centralized error handling and normalization
- Request cancellation via AbortController
- Type-safe request/response handling

Example structure:

```typescript
class ApiClient {
  private baseURL: string;
  private getAuthToken: () => Promise<string | null>;
  private refreshToken: () => Promise<string>;

  async request<T>(config: RequestConfig): Promise<T> {
    // Add auth headers via interceptor
    // Execute with retry logic
    // Handle token refresh on 401
    // Validate response with Zod schema if provided
  }
}
```

Pass dependencies (auth functions, base URL) rather than hardcoding. This makes testing and multi-environment support straightforward.

## Error Handling

### Error Classification

Distinguish error types to handle them appropriately:

- **Network errors** (no response) — Retry with backoff
- **4xx Client errors** — Don't retry (except 408 timeout, 429 rate limit)
- **5xx Server errors** — Retry with backoff
- **Validation errors** — Surface immediately, don't retry

Create typed error classes:

```typescript
class ApiError extends Error {
  constructor(
    public statusCode: number,
    public code: string,
    message: string,
    public details?: unknown
  ) {
    super(message);
  }

  isRetryable(): boolean {
    return (
      this.statusCode >= 500 ||
      this.statusCode === 408 ||
      this.statusCode === 429
    );
  }
}
```

Normalize backend error responses into a consistent shape. Different services return errors differently—your client should present a unified interface.

### Token Refresh Flow

Handle 401 responses by refreshing the token and retrying the original request:

1. Request fails with 401
2. Pause all pending requests
3. Refresh token (only once, even if multiple requests fail)
4. Retry all paused requests with new token
5. If refresh fails, clear auth state and redirect to login

Use a token refresh queue to prevent race conditions when multiple requests fail simultaneously.

## Retry Logic with Exponential Backoff

### When to Retry

Retry only transient failures:

- Network timeouts
- 5xx server errors
- 408 Request Timeout
- 429 Too Many Requests (respect `Retry-After` header)
- Connection refused/reset

Never retry:

- 4xx errors (except 408, 429) — these are client mistakes
- Non-idempotent operations (POST, PATCH) unless explicitly safe
- Validation failures

### Exponential Backoff with Jitter

Double wait time between retries, add randomness to prevent thundering herd:

```typescript
function calculateBackoff(attempt: number, baseDelay = 500): number {
  const exponentialDelay = baseDelay * Math.pow(2, attempt);
  const jitter = Math.random() * baseDelay;
  const maxDelay = 30000; // Cap at 30s
  return Math.min(exponentialDelay + jitter, maxDelay);
}
```

Typical config: 3-5 retries, 500ms base delay, 30s max delay.

Respect `Retry-After` headers for 429/503 responses—use that value instead of calculated backoff.

## Pagination Strategies

Support common pagination patterns:

### Offset-Based

```typescript
interface OffsetPaginatedResponse<T> {
  data: T[];
  total: number;
  offset: number;
  limit: number;
}

async function fetchPaginated<T>(
  endpoint: string,
  limit: number,
  offset = 0
): Promise<OffsetPaginatedResponse<T>> {
  return client.get(endpoint, { params: { limit, offset } });
}
```

### Cursor-Based

```typescript
interface CursorPaginatedResponse<T> {
  data: T[];
  nextCursor: string | null;
  hasMore: boolean;
}

async function* fetchAllPages<T>(
  endpoint: string,
  pageSize = 50
): AsyncGenerator<T[], void, undefined> {
  let cursor: string | null = null;
  do {
    const response: CursorPaginatedResponse<T> = await client.get(endpoint, {
      params: { cursor, limit: pageSize },
    });
    yield response.data;
    cursor = response.nextCursor;
  } while (cursor);
}
```

Cursor-based is more resilient to data changes during iteration. Offset-based is simpler for random access.

## TypeScript Schema Generation

### From OpenAPI Specs

Generate Zod schemas and TypeScript types from OpenAPI specs using tools like `openapi-zod-client`, `@hey-api/openapi-ts`, or `orval`.

Typical workflow:

1. Obtain OpenAPI spec (JSON/YAML) from backend team or docs
2. Configure generator (e.g., `orval.config.ts`)
3. Run generator to produce schemas + types
4. Import schemas in service layer for runtime validation

Example `orval.config.ts`:

```typescript
export default {
  api: {
    input: './openapi.json',
    output: {
      target: './src/generated/api.ts',
      client: 'fetch',
      mode: 'tags-split',
      schemas: './src/generated/schemas',
    },
    hooks: {
      afterAllFilesWrite: 'prettier --write',
    },
  },
};
```

This generates type-safe API client methods and Zod schemas for validation.

### Runtime Validation with Zod

Validate API responses to catch breaking changes early:

```typescript
async function fetchUser(id: string): Promise<User> {
  const response = await client.get(`/users/${id}`);
  return UserSchema.parse(response); // Throws if shape mismatches
}
```

Validation catches:

- Backend breaking changes
- Unexpected null/undefined values
- Type mismatches (string vs number)
- Missing required fields

In production, consider logging validation errors to monitoring rather than throwing—prevents user-facing crashes from backend changes.

### Manual Schema Definition

When OpenAPI specs aren't available, define Zod schemas manually:

```typescript
import { z } from 'zod';

export const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string().min(1),
  role: z.enum(['admin', 'user', 'guest']),
  createdAt: z.string().datetime(),
  metadata: z.record(z.string()).optional(),
});

export type User = z.infer<typeof UserSchema>;
```

Define schemas alongside service files, not scattered through components.

## Service Layer Examples

### User Service

```typescript
export class UserService {
  constructor(private client: ApiClient) {}

  async getUser(id: string): Promise<User> {
    const data = await this.client.get(`/users/${id}`);
    return UserSchema.parse(data);
  }

  async updateUser(id: string, updates: Partial<User>): Promise<User> {
    const data = await this.client.patch(`/users/${id}`, updates);
    return UserSchema.parse(data);
  }

  async listUsers(filters?: UserFilters): Promise<PaginatedResponse<User>> {
    const data = await this.client.get('/users', { params: filters });
    return PaginatedUserSchema.parse(data);
  }
}
```

### Request Cancellation

Support cancellation for long-running requests:

```typescript
class ApiClient {
  async get<T>(
    url: string,
    config?: RequestConfig & { signal?: AbortSignal }
  ): Promise<T> {
    const controller = new AbortController();
    const signal = config?.signal || controller.signal;

    // Pass signal to fetch
    const response = await fetch(url, { ...config, signal });
    return response.json();
  }
}

// Usage in React
const controller = new AbortController();
useEffect(() => {
  userService.getUser(id, { signal: controller.signal });
  return () => controller.abort();
}, [id]);
```

## Best Practices

### Separation of Concerns

- **Client layer** handles HTTP mechanics (auth, retries, errors)
- **Service layer** handles business logic (data transformation, caching)
- **Component layer** handles UI state and user interaction

Don't mix these. Services shouldn't know about React hooks. Components shouldn't construct HTTP headers.

### Type Safety

- Generate types from OpenAPI when possible—single source of truth
- Use Zod for runtime validation at service boundaries
- Avoid `any`—use `unknown` and validate/narrow
- Prefer branded types for IDs: `type UserId = string & { __brand: 'UserId' }`

### Testing

- Mock the HTTP client, not individual services
- Test retry logic with controlled failures
- Test token refresh flow with expired tokens
- Validate error handling for each error class

### Performance

- Batch requests when possible (GraphQL, REST batch endpoints)
- Cache responses at service layer with TTL
- Debounce rapid successive requests
- Use `AbortController` to cancel stale requests

### Observability

- Log failed requests with context (endpoint, status, attempt count)
- Track retry metrics (success rate after N retries)
- Monitor token refresh frequency (high frequency = short expiry or leaks)
- Alert on validation failures (indicates backend breaking changes)

## Common Pitfalls

- **Retrying non-idempotent operations** — POST/PATCH may duplicate data
- **Infinite retry loops** — Always cap max retries
- **Ignoring `Retry-After` headers** — Causes rate limiting to worsen
- **Thundering herd on token refresh** — Use a queue/lock to refresh once
- **Swallowing errors** — Always surface or log, never silent catch
- **Tight coupling to fetch/axios** — Abstract behind interface for flexibility

## Framework Integration

### React

Use hooks for data fetching, but keep API logic in services:

```typescript
function useUser(id: string) {
  return useQuery(['user', id], () => userService.getUser(id));
}
```

Services remain framework-agnostic. Hooks handle React-specific concerns (caching, suspense, invalidation).

### Vue

Similar pattern with composables:

```typescript
function useUser(id: Ref<string>) {
  return useQuery(['user', id], () => userService.getUser(id.value));
}
```

### Svelte

Use stores for reactive state:

```typescript
const user = writable<User | null>(null);
userService.getUser(id).then(user.set);
```

In all cases, services are plain TypeScript classes/functions—no framework coupling.

## Decision Guide

**When to generate schemas from OpenAPI:**

- Backend provides up-to-date OpenAPI spec
- Multiple services share types
- Frequent API changes

**When to write schemas manually:**

- No OpenAPI spec available
- Simple, stable API
- Need custom validation logic beyond OpenAPI

**When to retry:**

- Idempotent operations (GET, PUT, DELETE)
- Transient errors (5xx, timeouts, network failures)
- Rate limiting with `Retry-After`

**When NOT to retry:**

- 4xx errors (except 408, 429)
- Non-idempotent operations (POST, PATCH) unless backend guarantees idempotency
- Validation errors

**Pagination strategy:**

- **Cursor-based** for real-time data, infinite scroll, large datasets
- **Offset-based** for random access, page numbers, small datasets

## Output Structure

Generate code with this structure:

```
src/
  api/
    client.ts          # Base HTTP client with retry, auth, errors
    errors.ts          # Error classes and utilities
    types.ts           # Shared types (RequestConfig, PaginatedResponse)
  services/
    user.service.ts    # UserService with domain methods
    order.service.ts   # OrderService
  schemas/
    user.schema.ts     # Zod schemas for User domain
    order.schema.ts    # Zod schemas for Order domain
  generated/           # Auto-generated from OpenAPI (if used)
    api.ts
    schemas/
```

Keep generated code separate from hand-written code. Never edit generated files manually.
