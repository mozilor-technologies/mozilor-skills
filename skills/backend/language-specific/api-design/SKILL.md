---
name: api-design
description: REST and GraphQL API design principles — resource modeling, versioning, error standards, pagination, and schema-first development.
triggers:
  - "api design"
  - "rest api"
  - "graphql"
  - "api principles"
  - "api standards"
  - "api versioning"
  - "api documentation"
  - "openapi"
---

# API Design Principles — REST & GraphQL

## 1) When to Use Each

| Criterion | REST | GraphQL |
|---|---|---|
| Simple CRUD operations | ✅ Best fit | Overkill |
| Multiple client types (web, mobile, partners) | Consider | ✅ Best fit |
| Deeply nested / relational data | Complex | ✅ Best fit |
| File upload/download | ✅ Native | Not supported |
| Public APIs with external consumers | ✅ Familiar | Possible |
| Real-time (subscriptions) | Via webhooks | ✅ Native |
| Caching (CDN, HTTP cache) | ✅ Simple | Complex (POST) |
| Rapid frontend iteration without backend deploys | Requires versioning | ✅ Best fit |

## 2) REST API Design

### Resource-Oriented URLs

Resources are nouns. HTTP verbs express the action.

```
# BAD: action-based URLs (RPC style)
POST /createUser
GET  /getProduct?id=42
POST /deleteOrder?id=7

# GOOD: resource-based URLs
POST   /users               → create user
GET    /users               → list users
GET    /users/42            → get user
PATCH  /users/42            → partial update
PUT    /users/42            → full replace
DELETE /users/42            → delete user

# Nested resources (sparingly — max 2 levels)
GET  /users/42/orders       → user's orders
POST /users/42/orders       → create order for user

# Actions as sub-resources (when needed)
POST /orders/7/cancel       → cancel order (action)
POST /users/42/verify-email → verify email (action verb ok here)
```

### HTTP Method Semantics

| Method | Idempotent | Safe | Use for |
|---|---|---|---|
| GET | ✅ | ✅ | Read resource(s) |
| POST | ❌ | ❌ | Create resource, non-idempotent actions |
| PUT | ✅ | ❌ | Full replace of a resource |
| PATCH | ❌ | ❌ | Partial update |
| DELETE | ✅ | ❌ | Delete resource |

### Status Code Standards

```
2xx — Success
  200 OK           → successful GET, PATCH, PUT
  201 Created      → successful POST creating a resource (include Location header)
  204 No Content   → successful DELETE or action with no response body
  202 Accepted     → async operation queued

4xx — Client Errors
  400 Bad Request        → validation failure, malformed request
  401 Unauthorized       → missing or invalid authentication
  403 Forbidden          → authenticated but not permitted
  404 Not Found          → resource doesn't exist
  409 Conflict           → duplicate key, state conflict
  422 Unprocessable Entity → validation error with field details
  429 Too Many Requests  → rate limit exceeded

5xx — Server Errors
  500 Internal Server Error → unexpected server failure
  502 Bad Gateway           → upstream dependency failure
  503 Service Unavailable   → intentional downtime / overload
```

### Consistent Response Structure

```json
// Success (single resource)
{
  "data": {
    "id": "usr_123",
    "email": "jane@example.com",
    "createdAt": "2026-01-15T10:30:00Z"
  }
}

// Success (list with pagination)
{
  "data": [...],
  "meta": {
    "total": 1250,
    "page": 1,
    "perPage": 20,
    "totalPages": 63
  },
  "links": {
    "self": "/users?page=1",
    "next": "/users?page=2",
    "prev": null
  }
}

// Error
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      { "field": "email", "message": "Invalid email format" },
      { "field": "name", "message": "Name is required" }
    ]
  }
}
```

### Pagination

Cursor-based for large/real-time datasets; offset-based for simpler needs:

```
# Offset pagination (simple, allows page jumping)
GET /products?page=3&per_page=20

# Cursor pagination (stable for real-time data, infinite scroll)
GET /events?cursor=eyJpZCI6MTAwfQ&limit=20

# Filter and sort
GET /orders?status=pending&user_id=42&sort=-created_at&limit=20
```

### Versioning Strategy

```
# URL path versioning (recommended — explicit, cacheable, easy to deprecate)
/api/v1/users
/api/v2/users

# Header versioning (cleaner URLs, harder to test)
Accept: application/vnd.myapi.v2+json

# Rules:
# - Never break v1 when shipping v2
# - Deprecate old versions with Sunset header
# - Maintain at least 2 versions simultaneously
# - Document migration guides
Sunset: Sat, 31 Dec 2026 23:59:59 GMT
Deprecation: true
Link: </api/v2/users>; rel="successor-version"
```

## 3) GraphQL API Design

### Schema-First Approach

Define the schema in SDL before writing any resolver:

```graphql
# Naming conventions
# Types: PascalCase
# Fields: camelCase
# Enums: ALL_CAPS

type User {
  id: ID!
  email: String!
  name: String!
  role: UserRole!
  createdAt: DateTime!
  "Orders placed by this user"
  orders(
    status: OrderStatus
    first: Int = 20
    after: String
  ): OrderConnection!
}

enum UserRole {
  ADMIN
  USER
  GUEST
}

type Order {
  id: ID!
  status: OrderStatus!
  total: Float!
  items: [OrderItem!]!
  user: User!
  createdAt: DateTime!
}

enum OrderStatus {
  PENDING
  CONFIRMED
  SHIPPED
  DELIVERED
  CANCELLED
}
```

### Pagination — Relay Cursor Specification

```graphql
type OrderConnection {
  edges: [OrderEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type OrderEdge {
  node: Order!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

# Usage
query {
  user(id: "123") {
    orders(first: 10, after: "cursor==") {
      edges {
        node { id status total }
        cursor
      }
      pageInfo { hasNextPage endCursor }
      totalCount
    }
  }
}
```

### Mutations — Clear Naming

```graphql
type Mutation {
  # Verb + Resource + (optional qualifier)
  createUser(input: CreateUserInput!): CreateUserPayload!
  updateUser(id: ID!, input: UpdateUserInput!): UpdateUserPayload!
  deleteUser(id: ID!): DeleteUserPayload!
  cancelOrder(id: ID!, reason: String): CancelOrderPayload!
}

input CreateUserInput {
  email: String!
  name: String!
  role: UserRole = USER
}

type CreateUserPayload {
  user: User
  errors: [UserError!]!
}

type UserError {
  field: String
  message: String!
  code: String!
}
```

### N+1 Problem — DataLoader Pattern

```typescript
// Never: N+1 in resolvers
const resolver = {
  Order: {
    user: (order) => UserRepo.findById(order.userId) // N queries!
  }
};

// Always: DataLoader batching
const userLoader = new DataLoader(async (ids: string[]) => {
  const users = await UserRepo.findByIds(ids); // 1 query for N orders
  return ids.map(id => users.find(u => u.id === id));
});

const resolver = {
  Order: {
    user: (order, _, ctx) => ctx.loaders.user.load(order.userId)
  }
};
```

### Persisted Queries & Security

```graphql
# Enable query depth limiting
# Max depth: 7 levels for most APIs
# Enable complexity analysis: reject expensive queries

# Never expose introspection in production for public APIs
GRAPHQL_INTROSPECTION=false  # production

# Use persisted queries for known clients
# Only allow pre-registered query hashes from trusted clients
```

## 4) Universal API Principles

### Documentation

- Every public API endpoint must have an OpenAPI 3.1 spec
- Every GraphQL API must have schema documentation (docstrings)
- Provide working examples in documentation
- Use Postman collections or Bruno for API examples

```yaml
# openapi.yaml excerpt
/users/{id}:
  get:
    summary: Get user by ID
    description: Returns a single user by their unique identifier.
    parameters:
      - name: id
        in: path
        required: true
        schema:
          type: string
          format: uuid
    responses:
      '200':
        description: User found
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserResponse'
      '404':
        $ref: '#/components/responses/NotFound'
```

### Authentication Standards

```
# REST: Bearer token in Authorization header
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...

# API Keys: X-API-Key header (not query params — they end up in logs)
X-API-Key: sk_live_...

# Never in:
# - URL query parameters (?token=secret)
# - Request body for GET requests
# - Cookies without Secure + HttpOnly + SameSite=Strict
```

### Rate Limiting Headers

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 987
X-RateLimit-Reset: 1717171717
Retry-After: 60
```

### Idempotency for Mutations

```
# Client sends unique key — safe to retry without double-processing
POST /payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

# Server stores result keyed on Idempotency-Key
# Returns same response for duplicate requests
```

## 5) API Design Checklist

- [ ] Resources are nouns, HTTP verbs express actions
- [ ] Consistent response envelope across all endpoints
- [ ] Standard error response shape with machine-readable codes
- [ ] Pagination on all list endpoints
- [ ] API versioned (URL path preferred)
- [ ] All endpoints documented with OpenAPI / GraphQL docstrings
- [ ] Authentication required on all non-public endpoints
- [ ] Rate limiting in place with standard headers
- [ ] Input validation returns 400/422 with field-level errors
- [ ] Secrets never appear in URLs or logs
- [ ] CORS configured with explicit allowed origins (not `*` in production)

## References

- [REST API Design Best Practices](https://zeonedge.com/sw/blog/api-design-best-practices-2026-rest-graphql-grpc)
- [GraphQL Best Practices](https://andrewodendaal.com/graphql-api-design-best-practices/)
- [GraphQL Schema Design Guide](https://www.toolbrew.dev/blog/graphql-api-schema-design-guide)
- [Relay Cursor Connection Spec](https://relay.dev/graphql/connections.htm)
- [OpenAPI 3.1 Spec](https://spec.openapis.org/oas/v3.1.0)
