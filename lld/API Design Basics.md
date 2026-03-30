
This guide gives a practical framework for designing APIs in interviews and real projects.

---

## 1) API design mindset

Before writing endpoints, answer:
- Who are the clients? (web app, mobile app, internal service)
- What workflows must they complete?
- What are read-heavy vs write-heavy paths?
- What consistency, latency, and error expectations exist?

Interview line:
- "I design APIs around client workflows and resource boundaries, then optimize with pagination, idempotency, and clear error contracts."

---

## 2) API-first process (easy template)

1. Define resources (nouns): `users`, `orders`, `payments`.
2. Define operations per resource (CRUD + domain actions).
3. Design request/response contracts.
4. Add validation and error model.
5. Add pagination/filter/sort for list endpoints.
6. Add auth + authorization.
7. Add idempotency and concurrency controls for critical writes.
8. Version and document.

---

## 3) Resource modeling (most common mistake area)

Use nouns for paths, verbs for HTTP methods.

Good:
- `GET /v1/orders/123`
- `POST /v1/orders`
- `PATCH /v1/orders/123`

Avoid:
- `POST /createOrder`
- `GET /getOrderById`

### Nested resources

Use nested paths only when relationship is clear and stable:
- `GET /v1/orders/123/items`

Avoid deep nesting:
- `/users/{id}/orders/{id}/items/{id}/...`

---

## 4) HTTP method semantics

- `GET`: read (safe, idempotent)
- `POST`: create/process non-idempotent action
- `PUT`: full replace (idempotent)
- `PATCH`: partial update
- `DELETE`: remove/deactivate

### Example: Order APIs

```http
POST /v1/orders
GET /v1/orders/{orderId}
GET /v1/orders?user_id=U123&status=CREATED&limit=20&cursor=abc
PATCH /v1/orders/{orderId}
DELETE /v1/orders/{orderId}
```

---

## 5) Request and response design

Principles:
- Keep field names consistent (`snake_case` or `camelCase`, pick one).
- Prefer explicit types and enums.
- Include metadata where needed (`created_at`, `updated_at`).
- Return stable IDs and canonical URLs when useful.

### Create order example

Request:

```json
{
  "user_id": "U123",
  "items": [
    { "product_id": "P1", "quantity": 2 },
    { "product_id": "P2", "quantity": 1 }
  ],
  "currency": "INR"
}
```

Response (`201 Created`):

```json
{
  "id": "O987",
  "user_id": "U123",
  "status": "CREATED",
  "amount": 1499.00,
  "currency": "INR",
  "created_at": "2026-03-30T10:12:00Z"
}
```

---

## 6) Status codes (practical set)

- `200 OK`: successful read/update
- `201 Created`: successful create
- `202 Accepted`: async accepted
- `204 No Content`: successful delete
- `400 Bad Request`: malformed request
- `401 Unauthorized`: missing/invalid auth
- `403 Forbidden`: authenticated but not allowed
- `404 Not Found`: resource missing
- `409 Conflict`: state/version conflict, duplicate logical action
- `422 Unprocessable Entity`: semantic validation failed
- `429 Too Many Requests`: rate-limited
- `500 Internal Server Error`: server fault

---

## 7) Error response contract (important)

Use one consistent error schema across all endpoints.

```json
{
  "error": {
    "code": "INVALID_QUANTITY",
    "message": "quantity must be >= 1",
    "field": "items[0].quantity",
    "request_id": "req_7f93"
  }
}
```

Why this helps:
- Easier client handling
- Better debugging
- Better observability with `request_id`

---

## 8) Pagination, filtering, sorting

### Filtering

```http
GET /v1/orders?user_id=U123&status=CONFIRMED
```

### Sorting

```http
GET /v1/orders?sort_by=created_at&sort_order=desc
```

### Pagination styles

#### Offset pagination (simple)
- `?limit=20&offset=40`
- Easy to implement; can be inefficient for very large offsets.

#### Cursor pagination (preferred at scale)
- `?limit=20&cursor=eyJpZCI6Ik85ODcifQ==`
- Stable and better for large datasets.

Cursor response example:

```json
{
  "items": [{ "id": "O987" }, { "id": "O988" }],
  "next_cursor": "eyJpZCI6Ik85ODgifQ==",
  "has_more": true
}
```

---

## 9) Idempotency and retries (critical for payments/orders)

For operations that clients may retry, support idempotency keys.

```http
POST /v1/payments
Idempotency-Key: 9f8bcd72-7a17-4a74-9ff3-8f8d20b6db77
```

Server behavior:
- First request creates result.
- Repeated same key + same payload returns original result (not duplicate charge/order).

Use when:
- network timeouts
- client retries
- at-least-once delivery systems

---

## 10) Concurrency control in APIs

Avoid lost updates with optimistic concurrency.

### ETag / version example

1. Client reads resource with `ETag: "v5"`.
2. Client updates with:

```http
PATCH /v1/orders/O987
If-Match: "v5"
```

If current version changed, return `409 Conflict` or `412 Precondition Failed`.

---

## 11) Synchronous vs asynchronous APIs

### Synchronous
- Client waits for completion.
- Good for short operations.

### Asynchronous
- Return `202 Accepted` + job ID when work is long-running.

```http
POST /v1/reports
-> 202 Accepted
{
  "job_id": "J123",
  "status_url": "/v1/jobs/J123"
}
```

Then:
- `GET /v1/jobs/J123` to poll status.

---

## 12) Authentication and authorization

### Authentication (who are you?)
- API key, OAuth2/JWT, session token.

### Authorization (what can you do?)
- Role-based checks (`admin`, `user`)
- Resource-level checks (`user can access only own order`)

Interview line:
- "AuthN verifies identity; AuthZ enforces permissions per endpoint and resource."

---

## 13) Versioning strategy

Common approach:
- URI versioning: `/v1/...`, `/v2/...`

Rules:
- Avoid breaking existing clients in same major version.
- Prefer additive changes (new optional fields).
- Deprecate with timeline + migration guide.

---

## 14) API contract quality checklist

- Names are resource-based and consistent.
- Request/response schemas are explicit and stable.
- Validation errors are actionable.
- Status codes are semantically correct.
- List APIs support filter/sort/pagination.
- Write APIs handle retries safely (idempotency).
- Critical updates have concurrency control.
- Auth + rate limits + observability included.

---

## 15) End-to-end example: order + payment flow

### Step 1: Create order

```http
POST /v1/orders
```

Response:
- `201 Created` with `order_id`.

### Step 2: Initiate payment (idempotent)

```http
POST /v1/payments
Idempotency-Key: d51f1...
```

Body:

```json
{
  "order_id": "O987",
  "amount": 1499.00,
  "method": "UPI"
}
```

### Step 3: Fetch order

```http
GET /v1/orders/O987
```

Returns order state:
- `CREATED` -> `PAYMENT_PENDING` -> `CONFIRMED`

Why this design is interview-friendly:
- clear resource boundaries
- idempotent payment API
- predictable status transitions
- easy client integration

---

## Common pitfalls to avoid

- Verb-heavy endpoint names (`/createX`, `/doY`)
- Inconsistent field naming across endpoints
- Missing pagination on list endpoints
- Using `200` for every response
- No standard error format
- No idempotency for retried critical writes
- Breaking existing contract without versioning

---

## Summary

- Design APIs around resources and client workflows.
- Keep contracts consistent, explicit, and versioned.
- Use proper HTTP semantics and error/status code discipline.
- Add pagination, idempotency, and concurrency controls early.
- In interviews, explain trade-offs and failure behavior, not just endpoint names.
