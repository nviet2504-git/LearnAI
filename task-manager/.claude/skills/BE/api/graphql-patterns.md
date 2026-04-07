# GraphQL Schema Patterns

Advanced patterns for production GraphQL schemas.

## Connection Pattern (Relay-style Pagination)

Standard pattern for cursor-based pagination:

```graphql
type Query {
  users(
    first: Int
    after: String
    last: Int
    before: String
  ): UserConnection!
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int
}

type UserEdge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

Forward pagination:
```graphql
query {
  users(first: 10, after: "cursor123") {
    edges {
      node { id name }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

Backward pagination:
```graphql
query {
  users(last: 10, before: "cursor456") {
    edges {
      node { id name }
    }
    pageInfo {
      hasPreviousPage
      startCursor
    }
  }
}
```

## Input Object Pattern

Single input argument for mutations:

```graphql
type Mutation {
  createOrder(input: CreateOrderInput!): CreateOrderPayload!
  updateOrder(input: UpdateOrderInput!): UpdateOrderPayload!
}

input CreateOrderInput {
  userId: ID!
  items: [OrderItemInput!]!
  shippingAddress: AddressInput!
}

input OrderItemInput {
  productId: ID!
  quantity: Int!
}

input AddressInput {
  street: String!
  city: String!
  postalCode: String!
  country: String!
}

type CreateOrderPayload {
  order: Order
  errors: [UserError!]
}

type UserError {
  message: String!
  field: String
  code: ErrorCode!
}

enum ErrorCode {
  VALIDATION_ERROR
  INSUFFICIENT_STOCK
  PAYMENT_FAILED
}
```

## Interface Pattern

Shared fields across types:

```graphql
interface Node {
  id: ID!
}

interface Timestamped {
  createdAt: DateTime!
  updatedAt: DateTime!
}

type User implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  email: String!
  name: String!
}

type Organization implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  name: String!
  members: [User!]!
}

type Query {
  node(id: ID!): Node
}
```

Query with inline fragments:
```graphql
query {
  node(id: "123") {
    id
    ... on User {
      email
      name
    }
    ... on Organization {
      name
      members { name }
    }
  }
}
```

## Union Pattern

Multiple possible return types:

```graphql
union SearchResult = User | Organization | Post

type Query {
  search(query: String!): [SearchResult!]!
}

type User {
  id: ID!
  name: String!
}

type Organization {
  id: ID!
  name: String!
}

type Post {
  id: ID!
  title: String!
  author: User!
}
```

Query:
```graphql
query {
  search(query: "graphql") {
    __typename
    ... on User {
      id
      name
    }
    ... on Organization {
      id
      name
    }
    ... on Post {
      id
      title
      author { name }
    }
  }
}
```

## Error Handling Patterns

**Pattern 1: Errors in payload**
```graphql
type CreateUserPayload {
  user: User
  errors: [UserError!]
}
```

**Pattern 2: Union with error types**
```graphql
union CreateUserResult = CreateUserSuccess | ValidationError | DuplicateEmailError

type CreateUserSuccess {
  user: User!
}

type ValidationError {
  message: String!
  fields: [String!]!
}

type DuplicateEmailError {
  message: String!
  existingUserId: ID!
}
```

## Custom Scalars

Define domain-specific types:

```graphql
scalar DateTime
scalar Email
scalar URL
scalar JSON
scalar UUID

type User {
  id: UUID!
  email: Email!
  website: URL
  createdAt: DateTime!
  metadata: JSON
}
```

Implement validation in resolvers. Clients see semantic types, not generic strings.

## Filtering and Sorting

```graphql
type Query {
  products(
    filter: ProductFilter
    sort: ProductSort
    first: Int
    after: String
  ): ProductConnection!
}

input ProductFilter {
  category: String
  minPrice: Float
  maxPrice: Float
  inStock: Boolean
  tags: [String!]
}

input ProductSort {
  field: ProductSortField!
  direction: SortDirection!
}

enum ProductSortField {
  NAME
  PRICE
  CREATED_AT
  POPULARITY
}

enum SortDirection {
  ASC
  DESC
}
```

## Batching and DataLoader

Avoid N+1 queries by batching:

```graphql
type Order {
  id: ID!
  userId: ID!
  user: User!  # Batched via DataLoader
  items: [OrderItem!]!
}

type OrderItem {
  productId: ID!
  product: Product!  # Batched via DataLoader
  quantity: Int!
}
```

Resolver uses DataLoader to batch `user` and `product` fetches:
```javascript
const userLoader = new DataLoader(async (userIds) => {
  const users = await db.users.findMany({ where: { id: { in: userIds } } });
  return userIds.map(id => users.find(u => u.id === id));
});

const resolvers = {
  Order: {
    user: (order, _, { loaders }) => loaders.user.load(order.userId)
  },
  OrderItem: {
    product: (item, _, { loaders }) => loaders.product.load(item.productId)
  }
};
```

## Deprecation Strategy

```graphql
type User {
  id: ID!
  email: String!
  
  # Old field - kept for compatibility
  username: String @deprecated(reason: "Use 'email' instead. Removal: 2026-08-01")
  
  # Renamed field
  fullName: String!
  name: String @deprecated(reason: "Use 'fullName'. Removal: 2026-08-01")
  
  # Type change via new field
  createdAt: DateTime!
  created: String @deprecated(reason: "Use 'createdAt' (DateTime). Removal: 2026-08-01")
}
```

Monitor deprecated field usage via introspection or analytics. Remove after grace period.

## Naming Conventions

- **Types**: `PascalCase` (User, OrderConnection, CreateUserInput)
- **Fields**: `camelCase` (userId, createdAt, orderHistory)
- **Enums**: `SCREAMING_SNAKE_CASE` (ORDER_STATUS_PENDING, SORT_DIRECTION_ASC)
- **Input types**: `{Action}{Type}Input` (CreateUserInput, UpdateOrderInput)
- **Payload types**: `{Action}{Type}Payload` (CreateUserPayload, UpdateOrderPayload)
- **Connection types**: `{Type}Connection` (UserConnection, OrderConnection)
- **Edge types**: `{Type}Edge` (UserEdge, OrderEdge)

## Schema Documentation

Document everything:

```graphql
"""
Represents a user account in the system.
Users can create orders, write reviews, and manage their profile.
"""
type User implements Node {
  """Unique identifier for the user."""
  id: ID!
  
  """
  Primary email address. Must be unique across all users.
  Used for authentication and notifications.
  """
  email: String!
  
  """
  Orders placed by this user, sorted by creation date (newest first).
  Returns empty list if user has no orders.
  """
  orders(
    "Maximum number of orders to return (default: 20, max: 100)"
    first: Int
    "Cursor for pagination"
    after: String
  ): OrderConnection!
}
```

Descriptions appear in GraphQL IDEs and generated docs. AI agents rely on them to understand intent.