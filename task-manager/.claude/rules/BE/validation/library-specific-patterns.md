# Library-Specific Validation Patterns

## Zod (TypeScript/JavaScript)

### Transformations

Zod can transform validated data:

```typescript
const schema = z.object({
  email: z.string().email().toLowerCase().transform(s => s.trim()),
  createdAt: z.string().datetime().transform(s => new Date(s)),
});

const result = schema.parse({
  email: '  USER@EXAMPLE.COM  ',
  createdAt: '2026-04-07T12:00:00Z',
});
// result.email === 'user@example.com'
// result.createdAt instanceof Date
```

**Use case:** Normalize data during validation. Ensures consistent format before business logic.

### Async Validation

For validations requiring I/O (database checks, external API calls):

```typescript
const schema = z.object({
  username: z.string().refine(
    async (username) => {
      const exists = await db.users.exists({ username });
      return !exists;
    },
    { message: 'Username already taken' }
  ),
});

const result = await schema.safeParseAsync(data);
```

**Caution:** Async validation is slower. Use sparingly and consider moving to business logic layer for complex checks.

### Union Types

Validate against multiple possible schemas:

```typescript
const PaymentSchema = z.discriminatedUnion('method', [
  z.object({
    method: z.literal('card'),
    cardNumber: z.string().regex(/^\d{16}$/),
    cvv: z.string().regex(/^\d{3}$/),
  }),
  z.object({
    method: z.literal('bank'),
    accountNumber: z.string(),
    routingNumber: z.string(),
  }),
]);
```

**Use case:** Polymorphic payloads where structure depends on a discriminator field.

### Custom Error Messages

```typescript
const schema = z.string().min(8, { message: 'Password must be at least 8 characters' });
```

Override default messages for better UX.

## Pydantic (Python)

### Field Validators

```python
from pydantic import BaseModel, field_validator

class User(BaseModel):
    username: str
    email: str
    
    @field_validator('username')
    @classmethod
    def username_alphanumeric(cls, v: str) -> str:
        if not v.isalnum():
            raise ValueError('Username must be alphanumeric')
        return v.lower()  # Normalize to lowercase
```

**Use case:** Custom validation logic beyond built-in constraints.

### Model Validators

Cross-field validation:

```python
from pydantic import BaseModel, model_validator

class PasswordReset(BaseModel):
    password: str
    confirm_password: str
    
    @model_validator(mode='after')
    def passwords_match(self):
        if self.password != self.confirm_password:
            raise ValueError('Passwords do not match')
        return self
```

### Computed Fields

```python
from pydantic import BaseModel, computed_field

class User(BaseModel):
    first_name: str
    last_name: str
    
    @computed_field
    @property
    def full_name(self) -> str:
        return f'{self.first_name} {self.last_name}'
```

**Use case:** Derive fields from validated input without storing redundant data.

### Strict Mode

```python
class StrictUser(BaseModel):
    model_config = {'strict': True}
    age: int  # Rejects '25' (string), requires int
```

By default, Pydantic coerces types (`'25'` → `25`). Strict mode rejects type mismatches.

## Joi (Node.js)

Joi predates Zod but remains popular in legacy codebases.

```javascript
const Joi = require('joi');

const schema = Joi.object({
  email: Joi.string().email().required(),
  age: Joi.number().integer().min(13).max(120).required(),
  role: Joi.string().valid('user', 'admin', 'moderator').required(),
});

const { error, value } = schema.validate(data, { abortEarly: false });
if (error) {
  // error.details contains all validation issues
}
```

**Zod vs Joi:** Zod has better TypeScript integration and type inference. Use Zod for new projects; Joi is fine for existing codebases.

## JSON Schema

Language-agnostic schema definition. Useful for API documentation (OpenAPI) and cross-language validation.

```json
{
  "type": "object",
  "properties": {
    "email": { "type": "string", "format": "email" },
    "age": { "type": "integer", "minimum": 13, "maximum": 120 }
  },
  "required": ["email", "age"],
  "additionalProperties": false
}
```

**TypeScript:** Use `ajv` for JSON Schema validation. Zod can generate JSON Schema via `zod-to-json-schema`.

**Python:** Use `jsonschema` library or generate JSON Schema from Pydantic models.

**Trade-off:** JSON Schema is verbose and lacks type inference. Use language-native libraries (Zod, Pydantic) for application code; JSON Schema for API contracts.

## Express-Validator (Node.js)

Middleware-based validation for Express:

```javascript
const { body, validationResult } = require('express-validator');

app.post('/users',
  body('email').isEmail().normalizeEmail(),
  body('password').isLength({ min: 12 }),
  (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    // Proceed
  }
);
```

**Zod vs Express-Validator:** Zod provides type inference and composable schemas. Express-Validator is simpler for basic validation but lacks type safety.

## FastAPI (Python)

FastAPI uses Pydantic natively:

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, ValidationError

app = FastAPI()

class CreateUserRequest(BaseModel):
    email: str
    password: str

@app.post('/users')
def create_user(user: CreateUserRequest):
    # user is automatically validated
    return {'email': user.email}
```

FastAPI automatically returns 422 Unprocessable Entity with validation errors. No manual error handling needed.

## Django REST Framework (Python)

```python
from rest_framework import serializers

class UserSerializer(serializers.Serializer):
    email = serializers.EmailField()
    age = serializers.IntegerField(min_value=13, max_value=120)
    
    def validate_email(self, value):
        if User.objects.filter(email=value).exists():
            raise serializers.ValidationError('Email already exists')
        return value
```

DRF serializers handle validation and deserialization. Similar to Pydantic but tightly coupled to Django.

## Library Selection Guide

**TypeScript/JavaScript:**
- **Zod:** Best all-around. Type inference, composable, excellent DX.
- **Valibot:** Smaller bundle size, modular. Use for bundle-size-critical applications.
- **Joi:** Legacy codebases. Avoid for new projects.

**Python:**
- **Pydantic:** Modern, fast, type-hint-based. Default choice.
- **Marshmallow:** Older, more verbose. Use if already in codebase.
- **Django REST Framework serializers:** Django projects only.

**Cross-language:**
- **JSON Schema:** API contracts, OpenAPI specs. Not for application code.
- **Protobuf:** High-performance, strongly typed. Use for microservices communication.

**Selection criteria:**
- Type inference (Zod, Pydantic) eliminates duplication
- Composable schemas scale to complex validation
- Framework integration (FastAPI + Pydantic, tRPC + Zod)
- Ecosystem maturity and community support