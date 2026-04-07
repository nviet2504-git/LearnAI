# OpenAPI Specification Patterns

Advanced patterns for OpenAPI 3.1+ specifications.

## Reusable Components

Define once, reference everywhere:

```yaml
components:
  schemas:
    Error:
      type: object
      required: [code, message]
      properties:
        code:
          type: string
          example: VALIDATION_ERROR
        message:
          type: string
        details:
          type: array
          items:
            $ref: '#/components/schemas/ErrorDetail'
        trace_id:
          type: string
          format: uuid
    
    ErrorDetail:
      type: object
      properties:
        field:
          type: string
        message:
          type: string
    
    PaginationMeta:
      type: object
      properties:
        next_cursor:
          type: string
          nullable: true
        has_more:
          type: boolean
        total:
          type: integer
          format: int64
  
  parameters:
    LimitParam:
      name: limit
      in: query
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20
    
    CursorParam:
      name: cursor
      in: query
      schema:
        type: string
  
  responses:
    BadRequest:
      description: Invalid request parameters
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    
    Unauthorized:
      description: Authentication required
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
```

Reference in paths:
```yaml
paths:
  /users:
    get:
      parameters:
        - $ref: '#/components/parameters/LimitParam'
        - $ref: '#/components/parameters/CursorParam'
      responses:
        '200':
          description: Success
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
```

## Polymorphic Responses (oneOf, anyOf, allOf)

**oneOf** (exactly one schema matches):
```yaml
PaymentMethod:
  oneOf:
    - $ref: '#/components/schemas/CreditCard'
    - $ref: '#/components/schemas/BankAccount'
  discriminator:
    propertyName: type
    mapping:
      credit_card: '#/components/schemas/CreditCard'
      bank_account: '#/components/schemas/BankAccount'
```

**anyOf** (one or more schemas match):
```yaml
SearchResult:
  anyOf:
    - $ref: '#/components/schemas/User'
    - $ref: '#/components/schemas/Organization'
```

**allOf** (compose schemas):
```yaml
AdminUser:
  allOf:
    - $ref: '#/components/schemas/User'
    - type: object
      properties:
        admin_level:
          type: integer
        permissions:
          type: array
          items:
            type: string
```

## Webhooks (OpenAPI 3.1+)

Document event-driven callbacks:

```yaml
webhooks:
  orderCreated:
    post:
      summary: Order created event
      description: Triggered when new order is created
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                event:
                  type: string
                  enum: [order.created]
                data:
                  $ref: '#/components/schemas/Order'
                timestamp:
                  type: string
                  format: date-time
      responses:
        '200':
          description: Webhook received
```

## Conditional Schemas

Use `if/then/else` for conditional validation:

```yaml
Shipment:
  type: object
  properties:
    type:
      type: string
      enum: [standard, express]
    delivery_date:
      type: string
      format: date
  if:
    properties:
      type:
        const: express
  then:
    required: [delivery_date]
```

## Examples for AI Agents

Provide multiple examples per endpoint:

```yaml
paths:
  /users:
    post:
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
            examples:
              basic:
                summary: Basic user creation
                value:
                  email: user@example.com
                  name: Jane Doe
              withOptionals:
                summary: User with optional fields
                value:
                  email: admin@example.com
                  name: Admin User
                  role: admin
                  phone: +1234567890
      responses:
        '201':
          content:
            application/json:
              examples:
                success:
                  value:
                    id: 550e8400-e29b-41d4-a716-446655440000
                    email: user@example.com
                    name: Jane Doe
                    created_at: '2026-04-07T10:30:00Z'
```

## Multi-File Specs

Split large specs across files:

```yaml
# openapi.yaml (entry point)
openapi: 3.1.0
info:
  title: My API
  version: 1.0.0
paths:
  $ref: './paths/index.yaml'
components:
  schemas:
    $ref: './schemas/index.yaml'

# paths/index.yaml
/users:
  $ref: './users.yaml'
/orders:
  $ref: './orders.yaml'

# paths/users.yaml
get:
  summary: List users
  responses:
    '200':
      $ref: '../responses/users.yaml#/UserList'
```

## Deprecation Headers

Document deprecation in responses:

```yaml
responses:
  '200':
    description: Success (deprecated endpoint)
    headers:
      Deprecation:
        schema:
          type: string
          example: 'true'
      Sunset:
        schema:
          type: string
          format: http-date
          example: 'Sat, 01 Jun 2026 00:00:00 GMT'
      Link:
        schema:
          type: string
          example: '</v2/users>; rel="alternate"'
```