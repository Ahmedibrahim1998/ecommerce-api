# 08 · Conventions & Scaffolding

Namespaces, naming conventions, message strings, and a step-by-step recipe for adding a new API feature the way the rest of the codebase is built. Use this whenever you create a model, service, repository, controller, or DTO.

- [1. Namespace Map](#1-namespace-map)
- [2. Naming Conventions](#2-naming-conventions)
- [3. User-Facing Strings](#3-user-facing-strings)
- [4. How to Add a New API Feature](#4-how-to-add-a-new-api-feature)
- [5. New Feature Checklist](#5-new-feature-checklist)

---

## 1. Namespace Map

| Layer | Namespace | Example (this project) |
|-------|-----------|------------------------|
| **Models** | `App\Models` | `App\Models\Product` |
| **Models (sub)** | `App\Models\{Domain}` | `App\Models\Order\OrderItem` |
| **Enums** | `App\Enum\{Domain}` | `App\Enum\Order\OrderStatusEnum` |
| **Repository** | `App\Repositories\{Entity}` | `App\Repositories\Product\ProductRepository` |
| **Repository interface** | `App\Repositories\{Entity}` | `App\Repositories\Product\ProductRepositoryInterface` |
| **Services (API)** | `App\Services\Api\V1\{Feature}` | `App\Services\Api\V1\Order\CheckoutService` |
| **Services (3rd party)** | `App\Services\ThirdParties\{Vendor}` | `App\Services\ThirdParties\Payment\ChargeService` |
| **Controllers (API)** | `App\Http\Controllers\Api\V1\{Feature}` | `App\Http\Controllers\Api\V1\Cart\CartController` |
| **DTOs** | `App\Http\DTOs\Api\V1\{Feature}` | `App\Http\DTOs\Api\V1\Order\CheckoutDTO` |
| **Resources** | `App\Http\Resources\Api\V1` | `App\Http\Resources\Api\V1\ProductResource` |
| **Middleware** | `App\Http\Middleware\Api` | `App\Http\Middleware\Api\ForceJsonResponse` |
| **Foundation** | `App\Foundation\*` | `App\Foundation\Api\Http\Response\ApiResponse` |
| **Exceptions** | `App\Exceptions\{Domain}` | `App\Exceptions\Order\InsufficientStockException` |
| **Policies** | `App\Policies` | `App\Policies\ProductPolicy` |
| **Notifications** | `App\Notifications` | `App\Notifications\LowStockNotification` |
| **Events** | `App\Events` | `App\Events\OrderPlaced` |
| **Jobs** | `App\Jobs` | `App\Jobs\SendOrderConfirmationJob` |
| **Listeners** | `App\Listeners` | `App\Listeners\NotifyAdminsOfLowStock` |
| **Providers** | `App\Providers` | `App\Providers\RepositoryServiceProvider` |

---

## 2. Naming Conventions

### PHP / Laravel
| Item | Convention | Example |
|------|-----------|---------|
| Class | PascalCase | `CheckoutService`, `OrderStatusEnum` |
| Interface | PascalCase + `Interface` | `ProductRepositoryInterface` |
| Method | camelCase | `deductForSale()`, `activeForUser()` |
| Scope | prefix `scope` | `scopeActive`, `scopeInStock` |
| Enum case | UPPER_SNAKE | `PENDING`, `OUT_OF_STOCK` |

### Database & Eloquent
| Item | Convention | Example |
|------|-----------|---------|
| Table | snake_case, **plural** | `products`, `order_items`, `stock_movements` |
| Pivot | snake_case, logical order | `cart_items` (holds data → first-class) |
| Column | snake_case | `stock_quantity`, `low_stock_threshold` |
| Foreign key | `{singular_table}_id` | `product_id`, `category_id`, `order_id` |
| Boolean | prefix `is_` / `has_` | `is_active`, `is_primary` |
| Model | Singular PascalCase | `Product`, `OrderItem` |
| Relation (belongsTo) | singular | `category()`, `product()` |
| Relation (hasMany) | plural | `items()`, `images()` |

### API
| Item | Convention | Example |
|------|-----------|---------|
| Route URI | kebab-case / lowercase | `cart/items`, `products/low-stock` |
| Nested | `resource/{id}/sub` | `orders/{id}/cancel` |
| JSON keys | **snake_case** | `order_number`, `unit_price`, `stock_quantity` |
| Request body | snake_case | `shipping_address_id`, `product_id` |
| Controller | PascalCase + `Controller` | `OrderController` |
| Action method | camelCase, intent | `store()`, `cancel()`, `lowStock()` |

### Other assets
| Item | Convention | Example |
|------|-----------|---------|
| DTO | PascalCase + `DTO`, snake_case props | `CheckoutDTO`, `$shipping_address_id` |
| Service | PascalCase + `Service` | `InventoryService` |
| Repository | PascalCase + `Repository` | `OrderRepository` |
| Exception | PascalCase + `Exception` | `InsufficientStockException` |
| Event | PascalCase, past tense | `OrderPlaced`, `ProductLowStock` |
| Notification | PascalCase + `Notification` | `LowStockNotification` |
| Lang key | dot.lowercase | `order.api.placed_successfully` |
| Config key | snake_case | `default_currency` |
| Env | UPPER_SNAKE | `PAYMENT_API_KEY` |

---

## 3. User-Facing Strings

The store is **single-language (English)**. User-facing messages are centralized in language files (standard Laravel practice) rather than hard-coded in classes.

- **Files:** `lang/en/*.php` (`enum.php`, `order.php`, `cart.php`, `product.php`, `validation.php`).
- **Usage:** `trans('order.api.placed_successfully')` → `lang/en/order.php` → `['api']['placed_successfully']`.
- **Enum labels:** enums return `trans("enum.order_status.{$this->value}")`; keys in `enum.php`.

**To add a key:** add it to the relevant `lang/en/{file}.php`, then use `trans('...')`.

---

## 4. How to Add a New API Feature

Worked example — a new **Coupon** feature (a realistic future addition): list, create, apply.

**Step 1 — Migration & Model**
```bash
php artisan make:migration create_coupons_table
```
Model at `app/Models/Coupon.php` (namespace `App\Models`) with `fillable`, `casts`, relations. Add an enum `App\Enum\Coupon\CouponTypeEnum` if needed, plus `enum.php` labels.

**Step 2 — Repository (interface + impl + binding)**
- `app/Repositories/Coupon/CouponRepositoryInterface.php` extends `App\Foundation\Repositories\RepositoryInterface`; add extras like `findByCode(string $code)`.
- `app/Repositories/Coupon/CouponRepository.php` extends `App\Foundation\Repositories\Repository`; `parent::__construct(new Coupon)`.
- Register in `RepositoryServiceProvider`: `CouponRepositoryInterface::class => CouponRepository::class`.

**Step 3 — Service**
- `app/Services/Api/V1/Coupon/CouponService.php` (namespace `App\Services\Api\V1\Coupon`).
- Inject `CouponRepositoryInterface`; methods return models/primitives — **no HTTP**.

**Step 4 — DTO**
- `app/Http/DTOs/Api/V1/Coupon/ApplyCouponDTO.php` extends `ValidatedDTO`; `#[Rules(['required','string','exists:coupons,code'])]` on `$code`.

**Step 5 — Resource**
- `app/Http/Resources/Api/V1/CouponResource.php` — snake_case keys in `toArray()`.

**Step 6 — Controller (attribute routes)**
- `app/Http/Controllers/Api/V1/Coupon/CouponController.php`, inject `CouponService`.
- `#[Post(uri: 'cart/apply-coupon', middleware: ['auth:sanctum'])]` → returns `ApiResponse::success(data: new CouponResource(...), message: trans('coupon.api.applied'))`.

**Step 7 — Translations**
- Add keys to `lang/en/coupon.php`.

**Step 8 — Policy / Filament (if admin-managed)**
- `ProductPolicy`-style policy if needed; add a Filament Resource so operators can manage coupons.

**Route result:** `uri: 'cart/apply-coupon'` → `POST https://your-domain/api/v1/cart/apply-coupon`.

---

## 5. New Feature Checklist

- [ ] Migration + model (+ enum + lang enum if needed)
- [ ] Repository interface + implementation + binding in `RepositoryServiceProvider`
- [ ] Service in `App\Services\Api\V1\{Feature}\`
- [ ] DTO(s) in `App\Http\DTOs\Api\V1\{Feature}\`
- [ ] Resource(s) in `App\Http\Resources\Api\V1\`
- [ ] Controller in `App\Http\Controllers\Api\V1\` with route attributes
- [ ] Message strings in `lang/en`
- [ ] Policy if authorization is required
- [ ] Filament Resource if operators manage it
- [ ] Feature test(s) covering happy + failure paths
- [ ] Endpoint documented (L5-Swagger annotations)

---

**Previous:** [← 07 · Tech Stack & Code Style](07-tech-stack-and-code-style.md) · **Next:** [09 · Integrations →](09-integrations.md)
