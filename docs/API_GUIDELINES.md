# API Guidelines — vsnvi-food

> Drafted from defaults — review and adjust.

## 1. URL rules

- All endpoints are prefixed with **`/api/v1`**.
- Resources are **plural, kebab-case nouns**: `/restaurants`, `/menu-items`, `/delivery-tasks`.
- Nest at most **two levels** deep: `/restaurants/{restaurantId}/menu-items`. Go deeper only through a
  top-level resource (`/menu-items/{id}`).
- Path variables are UUIDs: `/orders/3f2c…`.
- **State-changing actions** that aren't plain CRUD are written as sub-resources with `POST`:
  `POST /orders/{id}/cancel`, `POST /orders/{id}/accept`.
- "Current user" resources use `/me`: `GET /users/me`, `GET /users/me/orders`.
- Query parameters are camelCase: `?cuisine=north-indian&minRating=4&page=0&size=20`.
- No verbs in paths (except action sub-resources), no file extensions, and no trailing slashes.

## 2. Request and response format

- JSON only (`application/json`). SSE endpoints use `text/event-stream`.
- Field names are **camelCase**.
- Timestamps are **ISO-8601 in UTC**: `"2026-10-25T14:03:00Z"`.
- IDs are UUID strings.
- Money is always an object with a string amount, so no precision is lost:
  ```json
  { "amount": "249.00", "currency": "INR" }
  ```
- Enums are UPPER_SNAKE_CASE strings: `"READY_FOR_PICKUP"`.
- Omit null fields from responses.
- Successful responses return the resource directly. List responses use the pagination envelope below.

## 3. HTTP methods and status codes

| Method   | Use                                   | Success code           |
|----------|---------------------------------------|------------------------|
| `GET`    | Read                                  | `200 OK`               |
| `POST`   | Create / action                       | `201 Created` (with `Location` header) / `200 OK` for actions |
| `PUT`    | Full replace                          | `200 OK`               |
| `PATCH`  | Partial update                        | `200 OK`               |
| `DELETE` | Delete                                | `204 No Content`       |

| Error code | When                                                                 |
|------------|----------------------------------------------------------------------|
| `400`      | Malformed request or validation failure                              |
| `401`      | Missing, invalid or expired token                                    |
| `403`      | Authenticated but not allowed (wrong role or not the owner)          |
| `404`      | Resource not found (also used instead of 403 to hide other users' resources) |
| `409`      | Conflict: invalid state transition, duplicate, optimistic lock failure, idempotency key reused with a different body |
| `422`      | Business rule violated (restaurant closed, item unavailable, outside delivery area) |
| `429`      | Rate limited                                                         |
| `500`      | Unexpected server error                                              |

## 4. Error format

All errors use **RFC 9457 Problem Details** (Spring `ProblemDetail`), with these extensions:
`code`, `traceId` and, for validation errors, `errors`.

```json
{
  "type": "https://api.vsnvi.com/errors/validation-failed",
  "title": "Validation failed",
  "status": 400,
  "detail": "Request has 2 invalid fields",
  "instance": "/api/v1/orders",
  "code": "VALIDATION_FAILED",
  "traceId": "a1b2c3d4e5f6",
  "timestamp": "2026-10-25T14:03:00Z",
  "errors": [
    { "field": "items[0].quantity", "message": "must be greater than 0" },
    { "field": "addressId", "message": "must not be null" }
  ]
}
```

### Error code catalog (initial)

| Code                        | HTTP | Meaning                                         |
|-----------------------------|------|-------------------------------------------------|
| `VALIDATION_FAILED`         | 400  | Bean validation errors                          |
| `MALFORMED_REQUEST`         | 400  | Unreadable JSON or wrong types                  |
| `UNAUTHORIZED`              | 401  | Missing or invalid token                        |
| `TOKEN_EXPIRED`             | 401  | Access token expired, so refresh it             |
| `FORBIDDEN`                 | 403  | Not allowed                                     |
| `RESOURCE_NOT_FOUND`        | 404  | Entity not found                                |
| `EMAIL_ALREADY_REGISTERED`  | 409  | Duplicate registration                          |
| `INVALID_ORDER_STATE`       | 409  | Illegal order status transition                 |
| `CONCURRENT_MODIFICATION`   | 409  | Optimistic lock failure, so retry               |
| `IDEMPOTENCY_KEY_CONFLICT`  | 409  | Same key, different payload                     |
| `CART_RESTAURANT_MISMATCH`  | 422  | Adding an item from a different restaurant      |
| `ITEM_UNAVAILABLE`          | 422  | Menu item out of stock or disabled              |
| `RESTAURANT_CLOSED`         | 422  | Outside opening hours                           |
| `OUT_OF_DELIVERY_AREA`      | 422  | Address outside the serviceable radius          |
| `PAYMENT_FAILED`            | 422  | Payment declined or failed                      |
| `RATE_LIMITED`              | 429  | Too many requests                               |
| `INTERNAL_ERROR`            | 500  | Unexpected error (no internals leaked)          |

## 5. Pagination, sorting and filtering

**Offset pagination** (default, for most listings):

- Request: `?page=0&size=20&sort=createdAt,desc`
- Defaults: `page=0`, `size=20`. Maximum `size=100`.
- Response envelope:

```json
{
  "content": [ { "...": "..." } ],
  "page": { "number": 0, "size": 20, "totalElements": 134, "totalPages": 7 }
}
```

**Cursor pagination** (for feeds that grow quickly, like order history and location history):

- Request: `?limit=20&cursor=eyJjcmVhdGVkQXQiOi4uLn0`
- Response: `{ "content": [...], "nextCursor": "…" }`. `nextCursor` is absent on the last page.

**Filtering** uses plain query parameters (`?status=DELIVERED&from=2026-10-01T00:00:00Z`).
Only documented filter and sort fields are accepted. Unknown values return `400`.

## 6. Idempotency

- The `Idempotency-Key: <uuid>` header is **required** on `POST /orders` and on `POST /payments/**`
  (except the webhook).
- The server stores the key, request hash and response for 24 hours:
  - The same key with the same body returns the stored response.
  - The same key with a different body returns `409 IDEMPOTENCY_KEY_CONFLICT`.
- Webhooks are deduplicated by the gateway's event ID.

## 7. Authentication

- `Authorization: Bearer <accessToken>` (JWT, about 15 minutes).
- Refresh with `POST /api/v1/auth/refresh` using the refresh token (about 30 days, rotated on use).
- Public endpoints: auth register/login/refresh, restaurant and menu browsing, the payment webhook,
  and actuator health.
- Authorization is by role and ownership. Accessing another user's resource returns `404`.

## 8. Versioning

- The major version is in the URL (`/api/v1`).
- Additive changes (new fields, new endpoints) don't need a new version. Clients must ignore unknown fields.
- Breaking changes go into `/api/v2`. `v1` stays supported for at least one release cycle and is marked
  with the `Deprecation` and `Sunset` headers.

## 9. Endpoint list

Roles: **C** = CUSTOMER, **O** = RESTAURANT_OWNER, **D** = DELIVERY_PARTNER, **A** = ADMIN, **Public** = no auth.

### Auth and identity (`identity`)

| Method | Path                              | Role     | Description                     |
|--------|-----------------------------------|----------|---------------------------------|
| POST   | `/api/v1/auth/register`           | Public   | Register a customer, owner or partner |
| POST   | `/api/v1/auth/login`              | Public   | Get access and refresh tokens   |
| POST   | `/api/v1/auth/refresh`            | Public   | Rotate tokens                   |
| POST   | `/api/v1/auth/logout`             | Any      | Revoke refresh token            |
| GET    | `/api/v1/users/me`                | Any      | Current profile                 |
| PATCH  | `/api/v1/users/me`                | Any      | Update profile                  |
| GET    | `/api/v1/users/me/addresses`      | C        | List addresses                  |
| POST   | `/api/v1/users/me/addresses`      | C        | Add address                     |
| PUT    | `/api/v1/users/me/addresses/{id}` | C        | Update address                  |
| DELETE | `/api/v1/users/me/addresses/{id}` | C        | Delete address                  |

### Restaurants (`restaurant`)

| Method | Path                                         | Role     | Description                                  |
|--------|----------------------------------------------|----------|----------------------------------------------|
| GET    | `/api/v1/restaurants`                        | Public   | Search (`lat`, `lng`, `cuisine`, `minRating`, `isOpen`, `q`), paginated |
| GET    | `/api/v1/restaurants/{id}`                   | Public   | Restaurant details                           |
| POST   | `/api/v1/restaurants`                        | O        | Create restaurant (pending approval)         |
| PATCH  | `/api/v1/restaurants/{id}`                   | O        | Update own restaurant                        |
| PUT    | `/api/v1/restaurants/{id}/opening-hours`     | O        | Set opening hours                            |
| POST   | `/api/v1/restaurants/{id}/toggle-open`       | O        | Temporarily open or close                    |
| GET    | `/api/v1/restaurants/me`                     | O        | List own restaurants                         |

### Menu (`catalog`)

| Method | Path                                             | Role   | Description                       |
|--------|--------------------------------------------------|--------|-----------------------------------|
| GET    | `/api/v1/restaurants/{id}/menu`                  | Public | Full menu grouped by category     |
| POST   | `/api/v1/restaurants/{id}/menu-categories`       | O      | Create category                   |
| PATCH  | `/api/v1/menu-categories/{id}`                   | O      | Update category                   |
| DELETE | `/api/v1/menu-categories/{id}`                   | O      | Delete category                   |
| POST   | `/api/v1/restaurants/{id}/menu-items`            | O      | Create item (variants, add-ons)   |
| GET    | `/api/v1/menu-items/{id}`                        | Public | Item details                      |
| PATCH  | `/api/v1/menu-items/{id}`                        | O      | Update item                       |
| POST   | `/api/v1/menu-items/{id}/availability`           | O      | Mark in or out of stock           |
| DELETE | `/api/v1/menu-items/{id}`                        | O      | Delete item                       |

### Cart (`cart`)

| Method | Path                              | Role | Description                                      |
|--------|-----------------------------------|------|--------------------------------------------------|
| GET    | `/api/v1/cart`                    | C    | Current cart with computed totals                |
| POST   | `/api/v1/cart/items`              | C    | Add item (`422` if it's from a different restaurant) |
| PATCH  | `/api/v1/cart/items/{itemId}`     | C    | Change quantity or options                       |
| DELETE | `/api/v1/cart/items/{itemId}`     | C    | Remove item                                      |
| DELETE | `/api/v1/cart`                    | C    | Clear cart                                       |

### Orders (`order`)

| Method | Path                                  | Role  | Description                                         |
|--------|---------------------------------------|-------|-----------------------------------------------------|
| POST   | `/api/v1/orders`                      | C     | Checkout cart (`Idempotency-Key`), returns order and payment intent |
| GET    | `/api/v1/orders/{id}`                 | C/O/D/A | Order details (ownership enforced)                |
| GET    | `/api/v1/users/me/orders`             | C     | Order history (cursor pagination)                   |
| POST   | `/api/v1/orders/{id}/cancel`          | C/A   | Cancel (customer only while `PLACED`)               |
| GET    | `/api/v1/restaurants/{id}/orders`     | O     | Incoming or active orders (`status` filter)         |
| POST   | `/api/v1/orders/{id}/accept`          | O     | Accept with `prepTimeMinutes`                       |
| POST   | `/api/v1/orders/{id}/reject`          | O     | Reject with reason                                  |
| POST   | `/api/v1/orders/{id}/preparing`       | O     | Mark preparing                                      |
| POST   | `/api/v1/orders/{id}/ready`           | O     | Mark ready for pickup                               |
| GET    | `/api/v1/orders/{id}/tracking`        | C     | **SSE** stream: `location` and `status` events      |

### Payments (`payment`)

| Method | Path                                  | Role   | Description                                   |
|--------|---------------------------------------|--------|-----------------------------------------------|
| POST   | `/api/v1/payments/intents`            | C      | (Re)create a payment intent for a pending order (`Idempotency-Key`) |
| GET    | `/api/v1/payments/{id}`               | C/A    | Payment status                                |
| POST   | `/api/v1/payments/webhook`            | Public | Gateway webhook (signature verified)          |
| POST   | `/api/v1/payments/{id}/refund`        | A      | Manual refund                                 |

### Delivery (`delivery`)

| Method | Path                                          | Role | Description                              |
|--------|-----------------------------------------------|------|------------------------------------------|
| POST   | `/api/v1/delivery/availability`               | D    | Go online or offline                     |
| POST   | `/api/v1/delivery/location`                   | D    | Location ping (`lat`, `lng`, `heading`, `speed`) |
| GET    | `/api/v1/delivery-tasks/offers`               | D    | Pending task offers                      |
| POST   | `/api/v1/delivery-tasks/{id}/accept`          | D    | Accept offer                             |
| POST   | `/api/v1/delivery-tasks/{id}/decline`         | D    | Decline offer                            |
| POST   | `/api/v1/delivery-tasks/{id}/pickup`          | D    | Confirm pickup, giving `OrderPickedUp`   |
| POST   | `/api/v1/delivery-tasks/{id}/deliver`         | D    | Confirm delivery (OTP), giving `OrderDelivered` |
| GET    | `/api/v1/delivery-tasks/me`                   | D    | Active and past tasks                    |

### Reviews (`review`)

| Method | Path                                  | Role   | Description                              |
|--------|---------------------------------------|--------|------------------------------------------|
| POST   | `/api/v1/orders/{id}/reviews`         | C      | Rate restaurant and partner (after `DELIVERED`) |
| GET    | `/api/v1/restaurants/{id}/reviews`    | Public | Restaurant reviews (paginated)           |

### Admin

| Method | Path                                          | Role | Description                         |
|--------|-----------------------------------------------|------|-------------------------------------|
| GET    | `/api/v1/admin/restaurants`                   | A    | List restaurants (`status` filter)  |
| POST   | `/api/v1/admin/restaurants/{id}/approve`      | A    | Approve restaurant                  |
| POST   | `/api/v1/admin/restaurants/{id}/suspend`      | A    | Suspend restaurant                  |
| GET    | `/api/v1/admin/users`                         | A    | List users                          |
| POST   | `/api/v1/admin/users/{id}/block`              | A    | Block user                          |
| GET    | `/api/v1/admin/orders`                        | A    | Search all orders                   |

### Operations

| Method | Path                     | Role   | Description        |
|--------|--------------------------|--------|--------------------|
| GET    | `/actuator/health`       | Public | Health check       |
| GET    | `/actuator/info`         | A      | Build info         |
| GET    | `/actuator/metrics`      | A      | Metrics            |
