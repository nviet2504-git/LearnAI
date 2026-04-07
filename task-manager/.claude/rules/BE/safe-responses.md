---
name: api-response-standards
description: >
  Enforce safe, consistent API responses with mandatory rules for error handling,
  HTTP status codes, information disclosure prevention, and structured formatting.
  Use this skill when designing API endpoints, defining response schemas, building
  error handling systems, creating API standards, reviewing API security, writing
  API documentation, implementing response middleware, or any task involving REST
  API design, API error models, HTTP status code usage, API security hardening,
  response consistency, or preventing data leaks in API responses.
---

# API Response Standards

This skill defines mandatory rules for safe, consistent API responses. These are not suggestions—they exist because violating them creates security vulnerabilities, breaks client integrations, and makes APIs unmaintainable.

## Core Principles

**HTTP status codes carry semantic meaning.** They're not decorative. Clients, proxies, monitoring systems, and caching layers all react to status codes. Returning 200 with an error payload breaks every HTTP-aware tool in the ecosystem.

**Never leak internal state.** Stack traces, database schema details, file paths, framework versions, and configuration values are reconnaissance gifts to attackers. Generic error messages externally, detailed logging internally.

**Consistency enables automation.** Clients should parse responses mechanically. Every endpoint returning errors in different shapes forces brittle, endpoint-specific error handling.

**Security through minimal disclosure.** Return only what the client needs. Extra fields in responses increase attack surface and risk accidental exposure of sensitive data.

## HTTP Status Code Usage

### Success (2xx)

- **200 OK**: Request succeeded, response body contains the result. Use for GET (resource retrieved), PUT (resource updated), PATCH (resource partially updated).
- **201 Created**: Resource created successfully. MUST include `Location` header with URI of new resource. Use for POST when creating resources.
- **202 Accepted**: Request accepted for async processing. Include status-check endpoint or polling guidance in response.
- **204 No Content**: Success with no response body. Common for DELETE operations or updates where returning the resource adds no value.

### Client Errors (4xx)

These indicate the client sent invalid input. The request should NOT be retried without modification.

- **400 Bad Request**: Malformed syntax, invalid JSON, missing required fields. Generic client error when nothing more specific applies.
- **401 Unauthorized**: Authentication required or credentials invalid. Include `WWW-Authenticate` header specifying the auth scheme.
- **403 Forbidden**: Authenticated but lacks permission. User identity is known, authorization failed.
- **404 Not Found**: Resource does not exist at this URI. Also used to hide existence of resources from unauthorized users (return 404 instead of 403 to avoid enumeration).
- **409 Conflict**: Request conflicts with current resource state (e.g., duplicate creation, version mismatch, state transition violation).
- **422 Unprocessable Entity**: Syntax valid but semantic validation failed. Use for business logic validation errors (e.g., email format valid but domain blacklisted, age value negative).
- **429 Too Many Requests**: Rate limit exceeded. Include `Retry-After` header with seconds until retry allowed.

### Server Errors (5xx)

These indicate server-side failure. The same request might succeed if retried later.

- **500 Internal Server Error**: Unexpected server condition. Catch-all for unhandled exceptions. NEVER expose stack traces or internal details to clients.
- **502 Bad Gateway**: Upstream service returned invalid response. Use when proxying or calling dependent services.
- **503 Service Unavailable**: Temporarily unable to handle request (maintenance, overload). Include `Retry-After` header when possible.
- **504 Gateway Timeout**: Upstream service did not respond in time.

**Never return 200 with error payload.** This breaks HTTP semantics, confuses monitoring, prevents proper caching, and makes client error handling unreliable.

## Structured Error Response Format

Adopt RFC 9457 (Problem Details for HTTP APIs) for consistency and tooling compatibility:

```json
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 422,
  "detail": "The email field must be a valid email address.",
  "instance": "/users/create",
  "errors": [
    {
      "field": "email",
      "message": "Invalid email format",
      "code": "INVALID_EMAIL_FORMAT"
    }
  ],
  "request_id": "req_7f3d9a2b"
}
```

**Required fields:**
- `type`: URI identifying the error type (should be documentation link or unique identifier)
- `title`: Short, human-readable summary (constant for this error type)
- `status`: HTTP status code (must match response status)
- `detail`: Human-readable explanation specific to this occurrence

**Recommended fields:**
- `instance`: URI of the specific request that caused the error
- `errors`: Array of field-level validation errors (for 400/422 responses)
- `request_id`: Unique identifier for tracing in logs

**Security constraint:** Never include stack traces, internal paths, database details, or framework-specific error messages in any field sent to clients.

## Success Response Structure

Maintain consistent top-level structure across all endpoints:

```json
{
  "data": { /* actual resource or result */ },
  "meta": {
    "request_id": "req_7f3d9a2b",
    "timestamp": "2026-04-07T14:30:00Z"
  }
}
```

For collections:

```json
{
  "data": [ /* array of resources */ ],
  "meta": {
    "total": 247,
    "page": 2,
    "per_page": 20,
    "total_pages": 13
  },
  "links": {
    "self": "/users?page=2",
    "first": "/users?page=1",
    "prev": "/users?page=1",
    "next": "/users?page=3",
    "last": "/users?page=13"
  }
}
```

**Rationale:** Wrapping data in a `data` field allows adding metadata without colliding with resource properties. Clients can reliably find the payload.

## Preventing Information Disclosure

### What NOT to expose

- **Stack traces**: Log server-side, never return to client
- **Database schema**: Table names, column names, SQL queries
- **File paths**: Absolute paths, directory structure
- **Framework details**: Library versions, internal class names
- **Configuration values**: Connection strings, internal URLs, feature flags
- **Existence of resources**: Use 404 instead of 403 to prevent enumeration of protected resources
- **Excessive data**: Don't return full objects when client needs subset (use field filtering or dedicated DTOs)

### Error message patterns

**Bad (leaks internals):**
```json
{
  "error": "PDOException: SQLSTATE[23000]: Integrity constraint violation: 1062 Duplicate entry 'user@example.com' for key 'users.email'"
}
```

**Good (generic, actionable):**
```json
{
  "type": "https://api.example.com/errors/duplicate-resource",
  "title": "Duplicate Resource",
  "status": 409,
  "detail": "A user with this email address already exists.",
  "field": "email"
}
```

### Headers to remove in production

- `X-Powered-By` (reveals framework/server)
- `Server` (or replace with generic value)
- Any custom debug headers

### Logging vs. client responses

**Log internally (with request_id):**
- Full exception details and stack traces
- Database query text and parameters
- Internal state and variable values
- User IDs and correlation data

**Return to client:**
- Generic error category
- Actionable guidance (what to fix)
- Request ID for support correlation
- No internal implementation details

## Field Filtering and Data Minimization

Allow clients to request only needed fields:

```
GET /users/123?fields=id,name,email
```

Response:
```json
{
  "data": {
    "id": 123,
    "name": "Alice",
    "email": "alice@example.com"
  }
}
```

**Why:** Reduces bandwidth, prevents accidental exposure of sensitive fields, improves performance.

**Implementation:** Use DTOs or serialization groups—never rely on client-side filtering. The API must enforce what data leaves the server.

## Validation Error Details

For 400 or 422 responses, provide field-level errors:

```json
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 422,
  "detail": "One or more fields failed validation.",
  "errors": [
    {
      "field": "email",
      "message": "Must be a valid email address",
      "code": "INVALID_FORMAT"
    },
    {
      "field": "age",
      "message": "Must be between 18 and 120",
      "code": "OUT_OF_RANGE",
      "constraints": { "min": 18, "max": 120 }
    }
  ]
}
```

**Each error object:**
- `field`: Name of the invalid field
- `message`: Human-readable explanation
- `code`: Machine-readable error code (enables client-side logic)
- `constraints` (optional): Limits or rules that were violated

## Response Headers

**Always include:**
- `Content-Type: application/json; charset=utf-8`
- `X-Request-ID`: Unique identifier matching `request_id` in body (for log correlation)

**For errors:**
- `Retry-After`: For 429 (rate limit) and 503 (service unavailable)
- `WWW-Authenticate`: For 401 (authentication required)

**For created resources (201):**
- `Location`: URI of the newly created resource

## Centralized Error Handling

Implement middleware or global exception handlers to enforce these standards:

```javascript
// Express.js example
app.use((err, req, res, next) => {
  const requestId = req.id || generateRequestId();
  
  // Log full error internally
  logger.error({
    request_id: requestId,
    error: err.stack,
    path: req.path,
    method: req.method,
    user_id: req.user?.id
  });
  
  // Map to appropriate status code
  const status = err.statusCode || 500;
  
  // Return safe, structured response
  res.status(status).json({
    type: err.type || `https://api.example.com/errors/internal-error`,
    title: err.title || 'Internal Server Error',
    status: status,
    detail: status === 500 
      ? 'An unexpected error occurred. Please contact support.' 
      : err.message,
    request_id: requestId
  });
});
```

**Why centralized:** Ensures every error follows the same format, prevents accidental leaks, simplifies maintenance.

## Testing Response Standards

**Automated checks:**
- All error responses match RFC 9457 schema
- All 4xx/5xx responses include `request_id`
- No stack traces or internal paths in any response body
- Status code matches `status` field in error body
- All 201 responses include `Location` header
- All 401 responses include `WWW-Authenticate` header

**Security review:**
- Trigger all error paths and inspect responses for leaks
- Check headers for framework/version disclosure
- Verify generic messages for database, auth, and validation errors

## Summary Checklist

- [ ] Use semantically correct HTTP status codes
- [ ] Never return 200 with error payload
- [ ] Follow RFC 9457 for error responses
- [ ] Include `request_id` in all responses
- [ ] Log detailed errors server-side, return generic messages to clients
- [ ] Remove stack traces, paths, and framework details from responses
- [ ] Use 404 instead of 403 to prevent resource enumeration
- [ ] Return only necessary fields (support field filtering)
- [ ] Provide field-level validation errors for 400/422
- [ ] Include appropriate headers (`Location`, `Retry-After`, `WWW-Authenticate`)
- [ ] Implement centralized error handling middleware
- [ ] Automate response format validation in tests
