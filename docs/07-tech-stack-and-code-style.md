# 07 · Tech Stack, Code Style & Design Patterns

The engineering handbook for this project. It mirrors the conventions of our existing Laravel backends so any developer on the team can contribute consistently from day one.

- [1. Tech Stack](#1-tech-stack)
- [2. Code Style](#2-code-style)
- [3. Design Patterns](#3-design-patterns)
- [4. Library Usage Summary](#4-library-usage-summary)
- [5. File Layout](#5-file-layout-high-level)

---

## 1. Tech Stack

### Core
| Layer | Technology |
|-------|-----------|
| **Runtime** | PHP ^8.2 |
| **Framework** | Laravel ^11.0 |
| **API Auth** | Laravel Sanctum ^4.0 |
| **API Versioning** | URL prefix `api/v1` |

### API & Data
| Package | Purpose |
|---------|---------|
| **Spatie Route Attributes** | Attribute-based API routing (`#[Get]`, `#[Post]`, ...) |
| **WendellAdriel Validated DTO** | Request validation & typed DTOs |
| **Laravel API Resources** | JSON response shaping |
| **DarkaOnline L5-Swagger** | OpenAPI 3.0 / Swagger UI (served at `/api/documentation`) |

### Admin
| Package | Purpose |
|---------|---------|
| **Filament** ^3.x | Admin panel UI (products, categories, orders, inventory) |
| **Filament Shield** | Roles & permissions |

### Media & Storage
| Package | Purpose |
|---------|---------|
| **Spatie Media Library** | Product images & media collections |
| **League Flysystem AWS S3** | Object storage for media |

### Dev & Quality
| Package | Purpose |
|---------|---------|
| **Laravel Pint** | PHP code style (PSR-12) |
| **PHPUnit** | Tests |
| **Laravel Telescope** | Local debugging / request inspection |
| **Doctrine DBAL** | Schema changes in migrations |

---

## 2. Code Style

### EditorConfig (`.editorconfig`)
- **Charset:** UTF-8
- **Indent:** 4 spaces
- **Line ending:** LF
- **Insert final newline:** yes
- **Trim trailing whitespace:** yes
- **YAML:** indent 2

### PHP Style (Laravel Pint)
- PSR-12 base.
- Run: `composer lint`, `composer test:lint`.

### Conventions in Code
- **Type hints** for parameters and return types wherever applicable.
- **Readonly:** Services are `final readonly class` with constructor property promotion.
- **Docblocks** for `@property` / `@var` / `@return` on models and key classes.
- **Strict types** where practical.

---

## 3. Design Patterns

### 3.1 Repository Pattern
- **Location:** `App\Repositories\{Entity}\`
- **Interface:** `{Entity}RepositoryInterface` extends `App\Foundation\Repositories\RepositoryInterface`
- **Implementation:** `{Entity}Repository` extends `App\Foundation\Repositories\Repository`
- **Binding:** in `App\Providers\RepositoryServiceProvider` (interface → implementation)
- **Usage:** injected into Services; controllers never touch repositories directly.
- **Base contract:** `all()`, `create()`, `update()`, `delete()`, `find()`, `with()`, `paginate()`, `getModel()`.

### 3.2 Service Layer
- **Location:** `App\Services\Api\V1\{Feature}\` (e.g. `Order\CheckoutService`).
- **Role:** orchestrate repositories and domain logic; **no HTTP** in services.
- **Return:** models or primitives; controllers turn them into responses.
- **Third-party:** `App\Services\ThirdParties\{Vendor}\` (e.g. payment gateway).

### 3.3 DTO (Validated)
- **Package:** `wendelladriel/laravel-validated-dto`.
- **Location:** `App\Http\DTOs\Api\V1\{Feature}\`.
- **Role:** validate incoming request data and expose typed properties.
- **Usage:** controllers type-hint the DTO; validation runs before the method body.

### 3.4 API Response Wrapper
- **Class:** `App\Foundation\Api\Http\Response\ApiResponse`.
- **Methods:** `success($data, $message, $code)`, `error($message, $errors, $code)`, `responseWithError(\Throwable $e)`.
- **Shape:** `{ "message": "...", "data": ... }` or `{ "message": "...", "errors": [...] }`.

### 3.5 API Routing (Attribute-Based)
- **Package:** Spatie Route Attributes.
- **Config:** `config/route-attributes.php` — scans `App\Http\Controllers\Api\V1` with prefix `api/v1`, middleware `api`.
- **Attributes:** `#[Get(uri, middleware)]`, `#[Post]`, `#[Patch]`, `#[Delete]`.

### 3.6 Resources
- **Location:** `App\Http\Resources\Api\V1\`.
- **Usage:** returned inside `ApiResponse::success(data: ...)`; **snake_case** JSON keys.

### 3.7 Enums
- **Location:** `App\Enum\{Domain}\` (e.g. `Order\OrderStatusEnum`).
- **Backed:** `enum XxxEnum: string`, cases in `UPPER_SNAKE`.
- **Labels:** implement `HasLabel::getLabel()` via `trans('enum...')` for Filament & API.

### 3.8 Exceptions
- **Location:** `App\Exceptions\{Domain}\`.
- **Usage:** thrown from services; mapped to `ApiResponse::error` centrally.

### 3.9 Policies
- **Location:** `App\Policies\` — one per model.
- **Used by:** Filament and API authorization.

---

## 4. Library Usage Summary

| Need | Library / Pattern |
|------|-------------------|
| Auth (API) | Sanctum + `auth:sanctum` |
| Validation | Validated DTOs |
| API routes | Spatie Route Attributes |
| JSON shape | API Resources + `ApiResponse` |
| Persistence | Eloquent + Repository over models |
| Business logic | Services using Repositories |
| Admin panel | Filament + Filament Shield |
| Media / images | Spatie Media Library + S3 |
| Enums & labels | Backed enums + `lang/en/enum.php` |
| User-facing strings | `lang/en/*.php` + `trans()` (single language) |
| API docs | L5-Swagger at `/api/documentation` |

---

## 5. File Layout (High Level)

```
app/
├── Enum/                      # Domain enums
├── Events/                    # Domain events
├── Exceptions/                # Custom exceptions (by domain)
├── Filament/                  # Admin panel
├── Foundation/                # ApiResponse, Repository base, Enum base
├── Http/
│   ├── Controllers/Api/V1/    # API controllers (attribute routes)
│   ├── DTOs/Api/V1/           # Request DTOs
│   ├── Middleware/Api/        # API middleware
│   └── Resources/Api/V1/      # API resources
├── Jobs/                      # Queued jobs
├── Listeners/                 # Event listeners
├── Models/                    # Eloquent models (+ sub-namespaces)
├── Notifications/             # Notifications
├── Policies/                  # Authorization policies
├── Providers/                 # App, Repository, Filament
├── Repositories/              # Interface + implementation per entity
└── Services/
    ├── Api/V1/                # Feature services
    └── ThirdParties/          # Payment gateway, etc.
```

Read alongside [08 · Conventions & Scaffolding](08-conventions-and-scaffolding.md) when adding new pieces.

---

**Previous:** [← 06 · Implementation Plan](06-implementation-plan.md) · **Next:** [08 · Conventions & Scaffolding →](08-conventions-and-scaffolding.md)
