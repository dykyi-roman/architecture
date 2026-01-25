# [Architecture](README.md)

## API Design

API (Application Programming Interface) design is the process of developing interfaces that expose data and application functionality for use by developers and applications. Good API design is crucial for developer experience, maintainability, and system evolution.

## API Styles Comparison

| Aspect | REST | GraphQL | gRPC |
|--------|------|---------|------|
| Protocol | HTTP | HTTP | HTTP/2 |
| Data Format | JSON/XML | JSON | Protocol Buffers |
| Contract | OpenAPI/Swagger | Schema | .proto files |
| Best For | Public APIs, CRUD | Flexible queries, BFF | Microservices, high performance |
| Caching | HTTP caching | Complex | Custom |
| Learning Curve | Low | Medium | High |

## REST API

REST (Representational State Transfer) is an architectural style for distributed hypermedia systems.

### Richardson Maturity Model

**Level 0** - The Swamp of POX (Plain Old XML)
```
POST /api
{ "action": "getUser", "id": 123 }
```

**Level 1** - Resources
```
POST /users/123
{ "action": "get" }
```

**Level 2** - HTTP Verbs
```
GET /users/123
PUT /users/123
DELETE /users/123
```

**Level 3** - Hypermedia Controls (HATEOAS)
```json
{
    "id": 123,
    "name": "John",
    "_links": {
        "self": { "href": "/users/123" },
        "orders": { "href": "/users/123/orders" },
        "update": { "href": "/users/123", "method": "PUT" }
    }
}
```

### REST Best Practices

**Resource Naming**
```
GET    /users              # List users
GET    /users/123          # Get user
POST   /users              # Create user
PUT    /users/123          # Replace user
PATCH  /users/123          # Partial update
DELETE /users/123          # Delete user

GET    /users/123/orders   # User's orders (nested resource)
GET    /orders?user_id=123 # Filter orders (query parameter)
```

**HTTP Status Codes**
```
2xx Success
    200 OK              - Successful GET, PUT, PATCH
    201 Created         - Successful POST
    204 No Content      - Successful DELETE

4xx Client Errors
    400 Bad Request     - Validation errors
    401 Unauthorized    - Authentication required
    403 Forbidden       - Authorization failed
    404 Not Found       - Resource doesn't exist
    409 Conflict        - Resource conflict (duplicate)
    422 Unprocessable   - Semantic errors

5xx Server Errors
    500 Internal Error  - Unexpected server error
    502 Bad Gateway     - Upstream service error
    503 Unavailable     - Service temporarily down
```

**Error Response Format**
```php
final readonly class ApiErrorResponse
{
    public function __construct(
        public string $type,
        public string $title,
        public int $status,
        public string $detail,
        public ?string $instance = null,
        public ?array $errors = null,
    ) {
    }
}

// Response (RFC 7807 Problem Details)
{
    "type": "https://api.example.com/errors/validation",
    "title": "Validation Error",
    "status": 422,
    "detail": "The request contains invalid data",
    "instance": "/users/123",
    "errors": {
        "email": ["Invalid email format"],
        "age": ["Must be at least 18"]
    }
}
```

### API Versioning Strategies

**URL Path Versioning** (Most common)
```
GET /v1/users
GET /v2/users
```

**Query Parameter**
```
GET /users?version=1
GET /users?version=2
```

**Header Versioning**
```
GET /users
Accept: application/vnd.api+json; version=1
```

**Content Negotiation**
```
GET /users
Accept: application/vnd.company.user.v1+json
```

### Pagination

**Offset-based** (Simple but slow for large datasets)
```
GET /users?offset=0&limit=20
GET /users?page=1&per_page=20

{
    "data": [...],
    "pagination": {
        "total": 1000,
        "page": 1,
        "per_page": 20,
        "total_pages": 50
    }
}
```

**Cursor-based** (Better for large/real-time data)
```
GET /users?cursor=abc123&limit=20

{
    "data": [...],
    "pagination": {
        "next_cursor": "def456",
        "has_more": true
    }
}
```

## GraphQL

GraphQL is a query language for APIs that allows clients to request exactly the data they need.

### Schema Definition

```graphql
type User {
    id: ID!
    name: String!
    email: String!
    orders: [Order!]!
}

type Order {
    id: ID!
    total: Money!
    status: OrderStatus!
    items: [OrderItem!]!
}

type Query {
    user(id: ID!): User
    users(filter: UserFilter, pagination: PaginationInput): UserConnection!
}

type Mutation {
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User!
}
```

### When to Use GraphQL

**Advantages:**
* Flexible queries - clients get exactly what they need
* Single endpoint - no over/under-fetching
* Strong typing with introspection
* Great for BFF (Backend for Frontend) pattern

**Disadvantages:**
* Complex caching (no HTTP caching by default)
* N+1 query problems require DataLoader
* Rate limiting is more complex
* File uploads require additional handling

## gRPC

gRPC is a high-performance RPC framework using Protocol Buffers.

### Proto Definition

```protobuf
syntax = "proto3";

package user;

service UserService {
    rpc GetUser(GetUserRequest) returns (User);
    rpc CreateUser(CreateUserRequest) returns (User);
    rpc ListUsers(ListUsersRequest) returns (stream User);
}

message User {
    string id = 1;
    string name = 2;
    string email = 3;
}

message GetUserRequest {
    string id = 1;
}
```

### When to Use gRPC

**Advantages:**
* High performance (binary protocol, HTTP/2)
* Strongly typed contracts
* Bi-directional streaming
* Code generation for multiple languages

**Disadvantages:**
* Not browser-friendly (requires grpc-web)
* Less human-readable (binary format)
* Steeper learning curve
* Limited tooling compared to REST

## API Security

### Authentication Methods

**API Keys** - Simple, for server-to-server
```
Authorization: Api-Key sk_live_xxx
```

**JWT (JSON Web Tokens)** - Stateless authentication
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

**OAuth 2.0** - Third-party authorization
```
Authorization: Bearer access_token_xxx
```

### Rate Limiting

```php
final readonly class RateLimiter
{
    public function __construct(
        private CacheInterface $cache,
        private int $maxRequests = 100,
        private int $windowSeconds = 60,
    ) {
    }

    public function check(string $clientId): RateLimitResult
    {
        $key = "rate_limit:{$clientId}";
        $current = (int) $this->cache->get($key, 0);

        if ($current >= $this->maxRequests) {
            return RateLimitResult::exceeded($this->maxRequests, $this->getResetTime($key));
        }

        $this->cache->increment($key);
        if ($current === 0) {
            $this->cache->expire($key, $this->windowSeconds);
        }

        return RateLimitResult::allowed($this->maxRequests - $current - 1);
    }
}

// Response headers
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1640000000
```

### Input Validation

```php
final readonly class CreateUserRequest
{
    public function __construct(
        #[Assert\NotBlank]
        #[Assert\Email]
        public string $email,

        #[Assert\NotBlank]
        #[Assert\Length(min: 2, max: 100)]
        public string $name,

        #[Assert\NotBlank]
        #[Assert\Length(min: 8)]
        #[Assert\PasswordStrength]
        public string $password,
    ) {
    }
}
```

## API Documentation

### OpenAPI/Swagger

```yaml
openapi: 3.0.0
info:
  title: User API
  version: 1.0.0

paths:
  /users:
    get:
      summary: List users
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
      responses:
        '200':
          description: List of users
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserList'

  /users/{id}:
    get:
      summary: Get user by ID
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
                $ref: '#/components/schemas/User'
        '404':
          description: User not found
```

## Idempotency

Ensure operations can be safely retried.

```php
final readonly class IdempotentApiMiddleware
{
    public function handle(Request $request, callable $next): Response
    {
        $idempotencyKey = $request->headers->get('Idempotency-Key');

        if ($idempotencyKey === null) {
            return $next($request);
        }

        // Check for cached response
        $cachedResponse = $this->cache->get("idempotency:{$idempotencyKey}");
        if ($cachedResponse !== null) {
            return $cachedResponse;
        }

        $response = $next($request);

        // Cache successful responses
        if ($response->isSuccessful()) {
            $this->cache->set(
                "idempotency:{$idempotencyKey}",
                $response,
                ttl: 86400, // 24 hours
            );
        }

        return $response;
    }
}
```

## Summary

* Choose API style based on use case: REST for public APIs, GraphQL for flexible clients, gRPC for microservices
* Follow REST conventions (HTTP methods, status codes, resource naming)
* Version your APIs from the start
* Implement proper security (authentication, rate limiting, input validation)
* Document your APIs with OpenAPI/Swagger
* Make operations idempotent where possible

### Read
* [REST API Design Best Practices](https://stackoverflow.blog/2020/03/02/best-practices-for-rest-api-design/)
* [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
* [GraphQL Best Practices](https://graphql.org/learn/best-practices/)
* [gRPC Documentation](https://grpc.io/docs/)
* [RFC 7807 - Problem Details for HTTP APIs](https://tools.ietf.org/html/rfc7807)
* [OpenAPI Specification](https://swagger.io/specification/)