---
name: server-side-validation
description: >
  Enforce strict server-side validation with schema validation, type safety,
  input sanitization, structured error responses, and mandatory rejection of
  malformed data. Use this for API endpoints, form processing, data ingestion,
  backend services, webhooks, GraphQL resolvers, REST APIs, microservices,
  user input handling, authentication flows, file uploads, database operations,
  or any scenario where untrusted data enters your system. Apply when building
  secure backends, implementing data validation, protecting against injection
  attacks (SQL, XSS, command injection), ensuring data integrity, validating
  request payloads, processing external data sources, or enforcing business
  rules on incoming data.
---

# Server-Side Validation

## Overview

Server-side validation is the non-negotiable security boundary between untrusted input and your application logic. Client-side validation can be bypassed—server-side validation cannot. Every input from external sources (users, APIs, file uploads, webhooks) must be validated and sanitized before processing.

This skill focuses on production-grade validation: schema-based type checking, input sanitization, structured error handling, and fail-secure defaults.

## Core Principles

### Never Trust Client Input

Client-side validation improves UX but provides zero security. Attackers can disable JavaScript, manipulate HTTP requests, or craft custom payloads. Server-side validation is the only security boundary that matters.

**Why this matters:** "Any failure to validate against discrete options is a high security event" (OWASP). Validation failures often indicate tampering attempts.

### Validate Early, Validate Always

Validate at the entry point—before data touches business logic, databases, or downstream services. The earlier you catch invalid data, the smaller your attack surface.

**Pattern:** Validation should be the first operation in any request handler, before authentication checks or business logic.

### Fail Secure

When validation fails, reject the request completely. Never attempt to "fix" malformed data or proceed with partial validation. Ambiguity creates vulnerabilities.

**Default behavior:** Unknown fields → reject. Type mismatch → reject. Out-of-range values → reject. Malformed structure → reject.

## Schema-Based Validation

Schema validation automates type checking, structure validation, and constraint enforcement. Define schemas once, get runtime validation and static types.

### TypeScript/JavaScript: Zod

```typescript
import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email().max(255),
  password: z.string().min(12).max(128),
  age: z.number().int().min(13).max(120),
  role: z.enum(['user', 'admin', 'moderator']),
  metadata: z.record(z.string()).optional(),
});

type CreateUserInput = z.infer<typeof CreateUserSchema>;

// In your request handler
app.post('/users', async (req, res) => {
  const result = CreateUserSchema.safeParse(req.body);
  
  if (!result.success) {
    return res.status(400).json({
      error: 'Validation failed',
      issues: result.error.issues.map(issue => ({
        path: issue.path.join('.'),
        message: issue.message,
      })),
    });
  }
  
  const validData = result.data; // Type-safe, validated data
  // Proceed with business logic
});
```

**Why Zod:** Type inference eliminates duplication. `safeParse` returns structured errors without throwing. Composable schemas scale to complex validation.

### Python: Pydantic

```python
from pydantic import BaseModel, EmailStr, Field, ValidationError
from typing import Optional, Literal

class CreateUserRequest(BaseModel):
    email: EmailStr
    password: str = Field(min_length=12, max_length=128)
    age: int = Field(ge=13, le=120)
    role: Literal['user', 'admin', 'moderator']
    metadata: Optional[dict[str, str]] = None

@app.post('/users')
def create_user(data: dict):
    try:
        validated = CreateUserRequest(**data)
    except ValidationError as e:
        return JSONResponse(
            status_code=400,
            content={
                'error': 'Validation failed',
                'issues': [{
                    'path': '.'.join(map(str, err['loc'])),
                    'message': err['msg'],
                } for err in e.errors()]
            }
        )
    
    # validated is type-safe and guaranteed valid
```

**Why Pydantic:** Native Python type hints. Fast (Rust-based validation). Automatic JSON schema generation.

### Validation Patterns

**Enums for fixed sets:** If input must be one of a discrete set (dropdown, radio buttons), validate against that exact set. Anything else indicates tampering.

```typescript
const StatusSchema = z.enum(['draft', 'published', 'archived']);
// Rejects 'pending', 'deleted', or any unlisted value
```

**Strict object schemas:** By default, reject unknown keys. Attackers often inject extra fields to probe for vulnerabilities.

```typescript
const schema = z.object({ name: z.string() }).strict();
// Rejects { name: 'Alice', admin: true }
```

**Range constraints:** Validate numeric ranges, string lengths, array sizes. Prevents resource exhaustion and overflow attacks.

```python
count: int = Field(ge=1, le=100)  # 1-100 inclusive
tags: list[str] = Field(max_length=10)  # Max 10 tags
```

**Format validation:** Use built-in validators for emails, URLs, UUIDs, dates. Don't write regex for complex formats.

```typescript
z.string().email()  // RFC-compliant email validation
z.string().url()    // Valid URL with protocol
z.string().uuid()   // Valid UUID v4
```

## Input Sanitization

Sanitization removes or escapes dangerous characters. Validation rejects bad input; sanitization cleans acceptable input that contains risky characters.

### Allowlisting Over Denylisting

**Allowlist:** Define what IS allowed; reject everything else. More secure, harder to bypass.

```typescript
const usernameSchema = z.string().regex(/^[a-zA-Z0-9_-]{3,20}$/);
// Only alphanumeric, underscore, hyphen; 3-20 chars
```

**Denylisting** (blocking specific patterns) is weaker—attackers find bypasses. Use only as a secondary defense.

### Context-Specific Escaping

Escape data based on where it's used. SQL context needs SQL escaping; HTML context needs HTML escaping.

**SQL:** Use parameterized queries (prepared statements). Never concatenate user input into SQL strings.

```typescript
// WRONG: SQL injection vulnerability
db.query(`SELECT * FROM users WHERE email = '${email}'`);

// RIGHT: Parameterized query
db.query('SELECT * FROM users WHERE email = ?', [email]);
```

**HTML:** Escape `<`, `>`, `&`, `"`, `'` before rendering user content.

```typescript
import escapeHtml from 'escape-html';
const safe = escapeHtml(userInput);
// '<script>alert(1)</script>' → '&lt;script&gt;alert(1)&lt;/script&gt;'
```

**Command execution:** Never pass user input to shell commands. If unavoidable, use allowlists and escape shell metacharacters.

### Normalization

Canonicalize input before validation. Attackers use encoding tricks to bypass naive validation.

```typescript
// Normalize Unicode before validation
const normalized = input.normalize('NFC');

// Trim whitespace
const trimmed = z.string().trim();

// Lowercase for case-insensitive comparison
const email = z.string().email().toLowerCase();
```

**Why:** Prevents bypasses like `admin@example.com` vs `ADMIN@example.com` or Unicode lookalikes.

## Error Handling

Validation errors must be informative for developers but not leak sensitive implementation details.

### Structured Error Format

```typescript
interface ValidationError {
  error: 'Validation failed';
  issues: Array<{
    path: string;      // 'email', 'address.zip'
    message: string;   // 'Invalid email format'
    code?: string;     // 'invalid_email'
  }>;
}
```

**Return 400 Bad Request** for validation failures. Include field-level errors so clients can highlight specific issues.

### Error Message Guidelines

**Good:** "Email must be a valid email address"
**Bad:** "Regex /^[a-z]+@[a-z]+\.[a-z]+$/ failed"

**Good:** "Password must be at least 12 characters"
**Bad:** "String length 8 is less than minimum 12"

Describe the constraint, not the internal validation logic. Never expose stack traces, SQL errors, or file paths in validation errors.

### Logging Validation Failures

Log validation failures at INFO or WARN level. Log at ERROR if the failure indicates tampering (e.g., enum value not in the UI's dropdown).

```typescript
if (!result.success) {
  logger.warn('Validation failed', {
    path: req.path,
    ip: req.ip,
    issues: result.error.issues,
  });
}
```

High-frequency validation failures from a single IP may indicate an attack. Monitor and rate-limit.

## Common Validation Scenarios

### File Uploads

```typescript
const FileUploadSchema = z.object({
  filename: z.string().regex(/^[a-zA-Z0-9._-]+$/),
  mimetype: z.enum(['image/jpeg', 'image/png', 'application/pdf']),
  size: z.number().max(10 * 1024 * 1024), // 10MB
});

// Validate file metadata, not just extension
// Verify mimetype matches actual file content
```

**Additional checks:** Scan file content (not just extension). Strip EXIF metadata from images. Store uploads outside web root.

### Nested Objects

```typescript
const AddressSchema = z.object({
  street: z.string().min(1).max(100),
  city: z.string().min(1).max(50),
  zip: z.string().regex(/^\d{5}(-\d{4})?$/),
});

const UserSchema = z.object({
  name: z.string(),
  address: AddressSchema,
});
```

Validation errors include full path: `address.zip: Invalid format`.

### Arrays with Constraints

```typescript
const schema = z.object({
  tags: z.array(z.string().min(1).max(20)).min(1).max(10),
  // 1-10 tags, each 1-20 chars
});
```

Prevents resource exhaustion from massive arrays.

### Conditional Validation

```typescript
const schema = z.object({
  type: z.enum(['individual', 'business']),
  taxId: z.string().optional(),
}).refine(
  data => data.type !== 'business' || data.taxId != null,
  { message: 'Tax ID required for business accounts', path: ['taxId'] }
);
```

Cross-field validation for business rules.

## Integration Patterns

### Middleware Pattern

```typescript
function validate<T>(schema: z.ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json(formatErrors(result.error));
    }
    req.body = result.data; // Replace with validated data
    next();
  };
}

app.post('/users', validate(CreateUserSchema), createUserHandler);
```

### Validation at Multiple Layers

1. **API Gateway/Edge:** Rate limiting, basic format checks
2. **Request Handler:** Schema validation (this skill's focus)
3. **Business Logic:** Domain-specific rules (e.g., "username not already taken")
4. **Database:** Constraints as final safety net (NOT NULL, UNIQUE, CHECK)

Each layer serves a different purpose. Schema validation catches malformed data; database constraints catch logic bugs.

## Anti-Patterns

**❌ Skipping validation for "trusted" sources:** Internal APIs, admin panels, and background jobs still need validation. Compromised internal systems can inject bad data.

**❌ Validating after processing:** Validate before touching databases, calling external APIs, or executing business logic.

**❌ Generic "invalid input" errors:** Provide field-specific errors. "Invalid input" frustrates users and wastes debugging time.

**❌ Accepting and ignoring unknown fields:** Unknown fields often indicate API version mismatches or probing attacks. Reject them explicitly.

**❌ Sanitizing instead of validating:** If input should be a number, validate it's a number—don't try to parse strings into numbers. Sanitization is for cleaning valid input, not fixing invalid input.

## Quick Reference

**Validation workflow:**
1. Parse request body/query/params
2. Run schema validation (fail → 400 with structured errors)
3. Sanitize validated data (escape, normalize)
4. Proceed to business logic

**Schema design:**
- Use enums for fixed sets
- Set min/max constraints on strings, numbers, arrays
- Validate formats (email, URL, UUID) with built-in validators
- Reject unknown keys by default
- Compose schemas for nested objects

**Security:**
- Never trust client input
- Validate at the entry point
- Use parameterized queries for SQL
- Escape output based on context (HTML, SQL, shell)
- Log suspicious validation failures

**Error handling:**
- Return 400 for validation failures
- Include field-level errors with paths
- Use human-readable messages
- Don't leak implementation details