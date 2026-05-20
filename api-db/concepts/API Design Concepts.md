# API Design Concepts

## REST Principles

REST (Representational State Transfer) is an architectural style for distributed hypermedia systems. Key constraints:

- **Stateless**: Each request contains all information needed to process it. No server-side session state.
- **Uniform Interface**: Resources identified by URIs; manipulation through representations; self-descriptive messages.
- **Client-Server**: Separation of concerns between UI and data storage.
- **Cacheable**: Responses must define themselves as cacheable or non-cacheable.
- **Layered System**: Client cannot tell whether it is connected directly to the end server.

## HTTP Methods

| Method | Semantics | Idempotent | Safe |
|--------|-----------|-----------|------|
| GET | Retrieve resource | Yes | Yes |
| POST | Create resource | No | No |
| PUT | Replace resource | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Remove resource | Yes | No |

## Resource Naming

- Use nouns, not verbs: `/users` not `/getUsers`
- Use plural nouns: `/users/{id}` not `/user/{id}`
- Nest for relationships: `/users/{id}/orders`
- Keep nesting shallow (max 2 levels): avoid `/users/{id}/orders/{id}/items/{id}`

## Request and Response Design

### Request Body
- Use JSON for structured data
- Validate all fields server-side (type, length, format)
- Use explicit field names — avoid ambiguous abbreviations

### Response Body
- Always return a consistent envelope: `{ "data": ..., "error": null }`
- Or use HTTP status codes directly with a body only for errors
- Include pagination metadata for list endpoints: `{ "items": [...], "total": 100, "page": 1, "pageSize": 20 }`

### HTTP Status Codes
| Code | Meaning |
|------|---------|
| 200 | OK — successful GET, PUT, PATCH |
| 201 | Created — successful POST |
| 204 | No Content — successful DELETE |
| 400 | Bad Request — invalid input |
| 401 | Unauthorized — missing/invalid auth |
| 403 | Forbidden — authenticated but not authorized |
| 404 | Not Found |
| 409 | Conflict — duplicate resource, optimistic lock failure |
| 422 | Unprocessable Entity — validation error |
| 429 | Too Many Requests — rate limited |
| 500 | Internal Server Error |

## Pagination

### Offset-Based
```
GET /items?page=2&pageSize=20
```
- Simple to implement
- Inconsistent results if items are inserted/deleted between pages
- Poor performance on large offsets (DB must scan all preceding rows)

### Cursor-Based
```
GET /items?cursor=eyJpZCI6MTAwfQ&limit=20
```
- Consistent results regardless of concurrent writes
- Better performance — uses indexed cursor field
- Cannot jump to arbitrary pages
- Best for infinite scroll / real-time feeds

## API Versioning Strategies

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| URL path | `/v1/users` | Explicit, easy to route | URL pollution |
| Header | `Accept: application/vnd.api+json;version=1` | Clean URLs | Less visible |
| Query param | `/users?version=1` | Easy to test | Caching issues |

URL path versioning is most common and easiest to reason about.

## Authentication Patterns

### API Keys
- Simple string passed in header: `Authorization: ApiKey abc123`
- Good for server-to-server communication
- No expiry by default — must be rotated manually

### JWT (JSON Web Tokens)
- Self-contained token with claims (user ID, roles, expiry)
- Stateless — server validates signature without DB lookup
- Short-lived access tokens (15 min) + long-lived refresh tokens (7 days)
- Cannot be revoked before expiry without a blocklist

### OAuth 2.0
- Delegated authorization — user grants app access to their data
- Access token + refresh token flow
- Best for third-party integrations

## Rate Limiting

- Protect APIs from abuse and ensure fair usage
- Common strategies: fixed window, sliding window, token bucket
- Return `429 Too Many Requests` with `Retry-After` header
- Apply at API gateway level for consistency

## GraphQL vs REST

| Aspect | REST | GraphQL |
|--------|------|---------|
| Data fetching | Fixed endpoints, fixed response shape | Client specifies exact fields needed |
| Over/under-fetching | Common problem | Solved by design |
| Versioning | URL or header versioning | Schema evolution with deprecation |
| Caching | HTTP caching works naturally | Requires custom caching (persisted queries) |
| Tooling | Mature, universal | Growing ecosystem |
| Best for | Simple CRUD, public APIs | Complex data graphs, mobile clients |

## Idempotency Keys

For non-idempotent operations (POST), clients can send an idempotency key:
```
POST /payments
Idempotency-Key: uuid-v4-here
```
Server stores the result keyed by the idempotency key. Duplicate requests return the cached result. Prevents double-charges on network retries.
