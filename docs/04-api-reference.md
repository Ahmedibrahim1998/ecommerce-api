# 04 · API Reference

RESTful, versioned under `/api/v1`. All requests and responses are JSON. Authentication uses Sanctum bearer tokens.

- [1. Conventions](#1-conventions)
- [2. Authentication](#2-authentication)
- [3. Errors](#3-errors)
- [4. Endpoints](#4-endpoints)

---

## 1. Conventions

| Concern | Convention |
|---------|-----------|
| Base URL | `/api/v1` |
| Format | JSON request & response; `Accept: application/json` |
| Auth | `Authorization: Bearer <token>` |
| Pagination | `?page=` & `?per_page=`; responses include `data`, `links`, `meta` |
| Filtering | `?filter[category]=`, `?filter[price_min]=`, `?filter[in_stock]=1` |
| Sorting | `?sort=-created_at,price` (prefix `-` = descending) |
| Idempotency | `Idempotency-Key: <uuid>` header on `POST /orders` |
| Rate limiting | `throttle` middleware; stricter on auth & checkout |
| Versioning | URI-based (`/v1`); breaking changes ship under `/v2` |

**Legend:** 🔓 public · 🔐 authenticated · 👑 admin only.

---

## 2. Authentication

| Method | Endpoint | Access | Description |
|:------:|----------|:------:|-------------|
| POST | `/auth/register` | 🔓 | Register a customer; returns user + token |
| POST | `/auth/login` | 🔓 | Log in; returns a bearer token |
| POST | `/auth/logout` | 🔐 | Revoke the current token |
| GET | `/auth/me` | 🔐 | Current authenticated user |

**Example — login**

```http
POST /api/v1/auth/login
Content-Type: application/json

{ "email": "sara@example.com", "password": "secret123" }
```

```json
{
  "data": {
    "user": { "id": 1, "name": "Sara", "email": "sara@example.com", "role": "customer" },
    "token": "12|Xy9...token"
  }
}
```

---

## 3. Response Envelope & Errors

Every response is wrapped by `App\Foundation\Api\Http\Response\ApiResponse` (see [Architecture §3](02-architecture.md#3-the-layers)).

**Success**
```json
{ "message": "Order placed successfully.", "data": { "...": "..." } }
```

**Error**
```json
{
  "message": "The given data was invalid.",
  "errors": {
    "shipping_address_id": ["The selected shipping address is invalid."]
  }
}
```

| Status | Meaning | Example |
|:------:|---------|---------|
| 200 | OK | Successful read |
| 201 | Created | Order / resource created |
| 204 | No Content | Item removed |
| 401 | Unauthenticated | Missing/invalid token |
| 403 | Forbidden | Authorization (policy) failure |
| 404 | Not Found | Unknown resource |
| 409 | Conflict | Invalid state transition |
| 422 | Unprocessable | Validation failure / **insufficient stock** |
| 429 | Too Many Requests | Rate limit exceeded |

---

## 4. Endpoints

### Catalog (customer-facing)

| Method | Endpoint | Access | Description |
|:------:|----------|:------:|-------------|
| GET | `/categories` | 🔓 | List categories (tree) |
| GET | `/categories/{slug}/products` | 🔓 | Products in a category |
| GET | `/products` | 🔓 | List products (paginate / filter / sort / search) |
| GET | `/products/{slug}` | 🔓 | Product detail with images & availability |

> **Admin operations** — creating/editing products & categories, restocking, adjusting inventory, and viewing the low-stock report are done in the **Filament admin panel**, not through the API. They are backed by the same Services the API uses. See [Architecture §7](02-architecture.md#7-admin-filament-not-api).

### Cart 🔐

| Method | Endpoint | Description |
|:------:|----------|-------------|
| GET | `/cart` | View the active cart |
| POST | `/cart/items` | Add item `{ product_id, quantity }` |
| PATCH | `/cart/items/{id}` | Update item quantity |
| DELETE | `/cart/items/{id}` | Remove an item |
| DELETE | `/cart` | Clear the cart |

### Orders 🔐

| Method | Endpoint | Description |
|:------:|----------|-------------|
| POST | `/orders` | Checkout — create an order from the cart |
| GET | `/orders` | List the customer's orders |
| GET | `/orders/{id}` | Order detail |
| POST | `/orders/{id}/cancel` | Cancel (restocks items) |

> Order **status transitions** by operators (e.g. mark shipped/delivered) happen in the Filament admin panel.

**Example — checkout**

```http
POST /api/v1/orders
Authorization: Bearer 12|Xy9...token
Idempotency-Key: 5f1c2b7e-8a3d-4e6f-9c0a-1b2c3d4e5f60
Content-Type: application/json

{ "shipping_address_id": 7 }
```

```json
// 201 Created
{
  "data": {
    "id": 1001,
    "order_number": "ORD-2026-0001",
    "status": "pending",
    "total": "149.90",
    "items": [
      { "product_name": "Wireless Mouse", "unit_price": "74.95", "quantity": 2, "line_total": "149.90" }
    ],
    "created_at": "2026-07-19T10:15:00Z"
  }
}
```

```json
// 422 Unprocessable — insufficient stock
{
  "message": "Insufficient stock for 'Wireless Mouse'.",
  "errors": { "product_id": [12], "available": [1], "requested": [2] }
}
```

---

**Previous:** [← 03 · Data Model](03-data-model.md) · **Next:** [05 · Inventory & Concurrency →](05-inventory-and-concurrency.md)
