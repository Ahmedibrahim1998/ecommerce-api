# 10 · Twelve-Factor Compliance

This project follows **[The Twelve-Factor App](https://12factor.net)** methodology. This document turns each factor into **concrete, checkable rules for this codebase**. It is binding during implementation: a pull request that breaks one of these rules is not done.

- [Summary](#summary)
- [I. Codebase](#i-codebase)
- [II. Dependencies](#ii-dependencies)
- [III. Config](#iii-config)
- [IV. Backing Services](#iv-backing-services)
- [V. Build, Release, Run](#v-build-release-run)
- [VI. Processes](#vi-processes)
- [VII. Port Binding](#vii-port-binding)
- [VIII. Concurrency](#viii-concurrency)
- [IX. Disposability](#ix-disposability)
- [X. Dev/Prod Parity](#x-devprod-parity)
- [XI. Logs](#xi-logs)
- [XII. Admin Processes](#xii-admin-processes)
- [Pull Request Checklist](#pull-request-checklist)

---

## Summary

| # | Factor | Rule in this project | Status |
|:-:|--------|----------------------|:------:|
| I | Codebase | One repo (`ecommerce-api`) for API + Filament; every environment deploys from it | ✅ |
| II | Dependencies | Everything (incl. PHP extensions) in `composer.json`; `composer.lock` committed | ⏳ with first code |
| III | Config | Environment variables only; complete `.env.example`; `config()` in code, never `env()` | ⏳ with first code |
| IV | Backing Services | MySQL, Redis, S3, mail, payment are swappable via config; third parties behind interfaces | ✅ by design |
| V | Build, Release, Run | CI pipeline; deploys from Git tags; releases are immutable and can be rolled back | ⏳ Milestone 1 |
| VI | Processes | Stateless: Sanctum tokens, Redis cache/queue/locks, S3 media — nothing on local disk | ✅ by design (NFR-5) |
| VII | Port Binding | `php artisan serve` in dev; container or Octane in production | ⚠️ open decision |
| VIII | Concurrency | `web` / `worker` / `scheduler` process types; stock locking in the database | ✅ by design |
| IX | Disposability | Transactions, `Idempotency-Key`, idempotent jobs, graceful worker restarts | ✅ by design (NFR-2) |
| X | Dev/Prod Parity | Same MySQL + Redis versions everywhere; concurrency test on real MySQL | ⚠️ Redis missing locally |
| XI | Logs | `stderr` in production; structured log events; no secrets in logs | ⏳ (NFR-7) |
| XII | Admin Processes | One-off tasks are Artisan commands in the repo; no manual production DB edits | ✅ by design |

---

## I. Codebase

> One codebase tracked in Git, many deploys.

- The API and the Filament admin panel live in **one repository**, because they share the same Services and Models.
- Every environment (local, staging, production) is deployed from a known commit or tag of this repo.
- **Never** edit files directly on a server.
- Code shared with another project becomes a Composer package, not a copy-paste.

## II. Dependencies

> Explicitly declare and isolate dependencies.

- Every package **and every PHP extension** the code uses is declared in `composer.json` (`ext-pdo`, `ext-intl`, `ext-gd`, and `ext-redis` or `predis/predis`).
- `composer.lock` is **committed**. `/vendor` is **not** (already in `.gitignore`).
- Servers and CI run `composer install` (never `composer update`).
- Production installs without dev tools: `composer install --no-dev --optimize-autoloader`.

## III. Config

> Store config in the environment.

- Anything that differs between environments (credentials, hosts, API keys, `APP_DEBUG`, `APP_URL`, currency) is an environment variable.
- `.env` is never committed (already in `.gitignore`). **`.env.example` is committed and always lists every key** used by the app, without secret values.
- `env()` is called **only inside `config/*.php`**. Application code reads settings with `config('...')`, because `php artisan config:cache` makes `env()` return `null` elsewhere.
- Project-specific settings live in `config/shop.php` (e.g. `config('shop.default_currency')` from `DEFAULT_CURRENCY`).
- **Test:** the repo could be made public right now without leaking a single credential.

```php
// ✅ config/services.php
'payment' => ['key' => env('PAYMENT_API_KEY')],

// ✅ application code
$key = config('services.payment.key');

// ❌ application code
$key = env('PAYMENT_API_KEY');
```

## IV. Backing Services

> Treat backing services as attached resources.

| Service | Selected / located by |
|---------|-----------------------|
| MySQL 8 | `DB_CONNECTION`, `DB_HOST`, `DB_DATABASE`, ... |
| Redis (cache, queue, locks) | `REDIS_HOST`, `CACHE_STORE`, `QUEUE_CONNECTION` |
| S3 (product media) | `FILESYSTEM_DISK`, `AWS_*` |
| Mail | `MAIL_MAILER`, `MAIL_HOST`, ... |
| Payment gateway | `PAYMENT_*` + `PaymentGatewayInterface` (FR-6.2) |

- Swapping any of these (local → managed cloud, stub → live gateway) must require **config changes only, no code changes**.
- Third parties are reached through interfaces bound in a service provider (see [09 · Integrations](09-integrations.md)).

## V. Build, Release, Run

> Strictly separate build and run stages.

| Stage | Steps in this project |
|-------|-----------------------|
| **Build** | `composer install --no-dev --optimize-autoloader` · `composer lint` · `php artisan test` (CI) |
| **Release** | build + environment `.env` · `php artisan config:cache` · `php artisan route:cache` · `php artisan migrate --force` |
| **Run** | web process + `queue:work` + `schedule:work` · `php artisan queue:restart` after each deploy |

- Deploys are made from **Git tags** (e.g. `v0.1.0`). Each release is immutable and identifiable.
- Rolling back = switching to the previous release, not hot-fixing on the server.

## VI. Processes

> Execute the app as one or more stateless processes.

This is **NFR-5** (stateless, horizontally scalable API).

| Concern | ❌ Not allowed | ✅ Required |
|---------|---------------|------------|
| Auth | file sessions | Sanctum bearer tokens (stored in DB) |
| Cache | `CACHE_STORE=file` | `redis` |
| Uploaded files | `storage/app` on the server | S3 via Spatie Media Library |
| Locks | static variables / in-memory flags | database row locks or `Cache::lock()` on Redis |
| Queue (production) | `sync` | `redis` |
| Cart | session | `carts` / `cart_items` tables |

- No `static` properties or singletons that hold per-user or per-request data (this also keeps the door open for Octane).

## VII. Port Binding

> Export services via port binding.

- Development: `php artisan serve` → `http://localhost:8000` (see [README](../README.md#-getting-started)).
- Production: **open decision** between Nginx + PHP-FPM, a Docker container exposing one port, or Laravel Octane. A container is preferred because it also serves factor X.
- The port is configuration, not code.

## VIII. Concurrency

> Scale out via the process model.

| Process type | Command | Responsibility | Scaled when |
|--------------|---------|----------------|-------------|
| `web` | PHP-FPM / Octane | API + Filament requests | traffic grows |
| `worker` | `php artisan queue:work redis --tries=3` | emails, low-stock alerts | queue backlog grows |
| `scheduler` | `php artisan schedule:work` | periodic tasks | always exactly **one** |

- Processes are managed by a process manager (Supervisor, systemd, Docker); the app never daemonizes itself.
- Because many `web` processes run at once, **overselling protection must live in the database** (`DB::transaction` + `lockForUpdate()`), never in PHP memory. See [05 · Inventory & Concurrency](05-inventory-and-concurrency.md).

## IX. Disposability

> Fast startup and graceful shutdown.

- Checkout, cancellation and restock run inside `DB::transaction`, so a process killed mid-way leaves no partial state.
- `POST /orders` honours the `Idempotency-Key` header (NFR-2), so client retries never create duplicate orders.
- Queued listeners/jobs are **idempotent**: running one twice has the same effect as running it once (e.g. check `confirmation_sent_at` before sending the email).
- Workers run with `--max-time` / `--max-jobs` and are restarted gracefully with `php artisan queue:restart` on deploy.

## X. Dev/Prod Parity

> Keep development, staging, and production as similar as possible.

- Same database engine and major version everywhere: **MySQL 8**. SQLite is **not** used as a substitute for it.
- **The mandatory overselling concurrency test runs against real MySQL.** SQLite does not support row-level `SELECT … FOR UPDATE`, so a test there can pass while the code is wrong.
- Redis runs locally (Laragon or Docker) because cache, queue and locks depend on it.
- Same PHP and Laravel versions in every environment.
- Production runs Linux (case-sensitive): file names must match class names exactly.
- Deploy small changes often to keep the time gap short.

## XI. Logs

> Treat logs as event streams.

- Production: `LOG_CHANNEL=stderr`; the environment collects and ships logs. Development may use `stack`/`single`.
- Log **structured events** (event name + context), not free-form sentences (NFR-7):

```php
Log::info('order.placed', ['order_id' => $order->id, 'user_id' => $user->id, 'total' => $order->total]);
Log::warning('checkout.insufficient_stock', ['product_id' => $id, 'requested' => $qty, 'available' => $available]);
```

- Standard events: `order.placed`, `order.cancelled`, `checkout.insufficient_stock`, `payment.failed`.
- **Never** log passwords, tokens, card data, or API keys.
- `stock_movements` is business data (audit trail), not a log; it stays in the database.
- Telescope is for local development only.

## XII. Admin Processes

> Run admin/management tasks as one-off processes.

- One-off tasks are **Artisan commands** in `app/Console/Commands`, committed and reviewed, and run from the current release with the same `.env`.
- Schema changes only through migrations; reference data through seeders.
- Stock is never edited directly in the database. Manual restock/adjustment goes through `InventoryService` (from Filament, FR-5.4) so every change is recorded as a stock movement.
- Planned command: `php artisan inventory:reconcile` — verifies each product's `stock_quantity` matches its latest `stock_movements.resulting_quantity`.

---

## Pull Request Checklist

- [ ] No secrets or environment-specific values in code; new env keys added to `.env.example`.
- [ ] `env()` used only in `config/*.php`.
- [ ] New packages/extensions declared in `composer.json`; `composer.lock` updated.
- [ ] No state kept on local disk or in process memory.
- [ ] Multi-step writes wrapped in a transaction; queued jobs are idempotent.
- [ ] Tests that depend on locking run on MySQL.
- [ ] Important events logged as structured entries; nothing sensitive logged.
- [ ] One-off data changes delivered as Artisan commands or migrations.

---

**Previous:** [← 09 · Integrations](09-integrations.md) · **Back to:** [README](../README.md)
