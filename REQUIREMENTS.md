# Multi-tenant e-commerce API: requirements and learning roadmap

## Goal

Build a NestJS + PostgreSQL backend where multiple shops use the same application while keeping their data isolated. You already know basic PostgreSQL and Express; this project will teach NestJS architecture alongside database design, transactions, concurrency, security, and query performance.

We will build one milestone at a time. You implement each small step; I review your code, explain issues, help you test it, and validate the milestone before we continue. This document is the plan, not an instruction to implement the entire app immediately.

## Product scope

A tenant is one shop. Each shop has members, products, stock, customers, carts, and orders. A user can belong to multiple shops with a different role in each.

- **Owner:** manages shop settings, members, and all shop operations.
- **Staff:** manages catalog, inventory, and orders.
- **Customer:** browses published products and manages their own cart and orders in a shop.
- Every tenant endpoint requires a tenant context; authentication and authorization determine whether the caller may access it.
- The first version is an API only, with one currency per shop and one inventory location per shop.
- Payments use a fake provider. No real card data or live charges.
- Shipping integrations, tax engines, discount stacking, subscriptions, multiple warehouses, and a frontend are outside the initial scope.

## Learning and implementation choices

- Keep the existing NestJS application and use feature modules, controllers, providers, DTOs, pipes, guards, and dependency injection.
- Use PostgreSQL with the `pg` driver and versioned SQL migrations so that SQL and transaction behavior remain visible. We can compare an ORM after the core milestones.
- Use parameterized SQL for values and allowlists for dynamic SQL identifiers such as sort columns.
- Use a shared database and shared tables with `tenant_id`, then add PostgreSQL row-level security (RLS) as defense in depth.
- Use real PostgreSQL for database integration tests. Mocks cannot validate constraints, locks, or RLS.
- Store money as integer minor units with a currency code; never calculate prices using floating-point arithmetic. Define bounds and handle PostgreSQL `bigint` values explicitly in Node.js.
- Use UUID identifiers and `timestamptz` for recorded instants. Explain any exceptions.
- Prefer database constraints for invariants that must hold regardless of which code path writes data.

## Core requirements and invariants

1. A tenant cannot read or modify another tenant's records, including by guessing an ID.
2. A user cannot gain access by changing the tenant identifier in a URL or header.
3. Tenant-owned relationships cannot reference records in a different tenant. Use composite foreign keys such as `(tenant_id, product_id)` referencing a unique `(tenant_id, id)` pair.
4. Shop-local identifiers such as SKU are unique within a tenant, not across the whole application.
5. Stock quantities and monetary values stay within their defined bounds. Available stock never becomes negative.
6. Checkout is atomic: either the order, items, stock changes, and event record all commit, or none do.
7. Concurrent purchases cannot sell more stock than exists.
8. Order items preserve the purchased name, SKU, price, and currency even when a product changes later.
9. Repeating a successful checkout request with the same idempotency key does not create another order.
10. Customers can access only their own carts and orders; staff permissions are checked separately from tenant membership.
11. Payment events and inventory restoration are safe to retry without duplicate effects.
12. Query performance improvements must be supported by query plans and measurements on representative data.

## Initial data model

These are starting boundaries; we will write exact columns and constraints before each migration.

| Entity | Purpose | Ownership |
| --- | --- | --- |
| users | Login identity and password hash | Global |
| tenants | Shop name, slug, currency, settings | Tenant root |
| tenant_memberships | User role within a shop | Tenant |
| customers | Shop-specific customer profile linked to a user | Tenant |
| categories | Product classification | Tenant |
| products | Catalog entry, SKU, price, status, attributes | Tenant |
| product_categories | Product/category relationship | Tenant |
| inventory | Current stock per product | Tenant |
| inventory_movements | Append-only stock change history | Tenant |
| carts / cart_items | Customer's intended purchases | Tenant |
| orders / order_items | Purchase and immutable item snapshots | Tenant |
| checkout_requests | Idempotency key, request fingerprint, result | Tenant |
| payments / payment_events | Fake payment state and deduplication | Tenant |
| outbox_events | Events committed with business changes | Tenant |
| audit_logs | Selected administrative changes | Tenant |

## How we work through each milestone

1. Discuss the feature and the database invariant it must preserve.
2. Break it into small implementation tasks and agree on the first task.
3. You implement that task and share your changes or ask for a review in this workspace.
4. I review correctness, NestJS structure, SQL, security, and error handling, with explanations suited to your Express background.
5. We run the relevant checks and fix failures. Database milestones include integration tests against PostgreSQL.
6. You explain the key concept in your own words. We mark the milestone complete only when its acceptance checks pass.

Validation should use actual code and test output. A checklist item is not evidence by itself. We will not jump ahead and generate all modules unless you ask.

## Module-by-module roadmap

### 1. Foundation: AppModule, ConfigModule, DatabaseModule

**Build:** local database setup, validated configuration, a reusable database pool, SQL migration commands, and a health endpoint.

**Steps:**
1. Trace an incoming NestJS request through controller and service; compare it with Express middleware and handlers.
2. Add local PostgreSQL configuration and an `.env.example` with placeholders; keep secrets out of Git.
3. Create database lifecycle management, including pool shutdown, and a database-backed readiness check.
4. Create and apply the first migration; document how to create a clean test database.

**PostgreSQL focus:** connections vs. pools, schemas, migrations, database roles, and transaction boundaries. A transaction must use one checked-out client throughout.

**Validate:** clean database setup succeeds; invalid configuration fails clearly; readiness reports a database outage; the application closes its pool on shutdown; build and existing tests pass.

### 2. Identity: UsersModule and AuthModule

**Build:** registration, login, password hashing, authenticated user context, and a protected profile endpoint. Start with short-lived access tokens; add refresh tokens only as an extension.

**PostgreSQL focus:** primary keys, `NOT NULL`, unique constraints, and case-insensitive email uniqueness using a deliberate normalization/index strategy.

**Validate:** two concurrent registrations for the same normalized email produce one user; invalid credentials fail; password hashes never appear in responses or logs; unauthenticated requests cannot access the profile.

### 3. Tenancy: TenantsModule and MembershipsModule

**Build:** create a shop, list my shops, add existing users as members, assign roles, and resolve tenant context from `/tenants/:tenantId/...` routes.

**PostgreSQL focus:** joins, many-to-many relationships, composite uniqueness and foreign keys, and transactions for creating a tenant and its owner membership together.

**Validate:** membership is unique per user/shop; creating a shop creates its owner atomically; a nonmember cannot access it; changing the tenant ID does not grant access; only owners can manage members. Define and test protection against removing the last owner.

### 4. Database isolation: TenantAccessModule

**Build:** a tenant transaction wrapper and RLS policies on tenant-owned tables. Require later modules to use this wrapper and add their own policies.

**PostgreSQL focus:** `USING`, `WITH CHECK`, transaction-local tenant context, runtime vs. migration roles, and RLS owner/bypass behavior.

The server first verifies membership and authorization, then sets tenant context inside the transaction on the same connection used by its queries. The runtime role must not be a superuser, have `BYPASSRLS`, or accidentally bypass policies through table ownership. RLS supplements application authorization; it does not establish membership by itself.

**Validate:** under the actual runtime role, cross-tenant SELECT/INSERT/UPDATE/DELETE attempts fail or expose no rows; missing context fails closed; a reused pooled connection never inherits the previous request's tenant context. Cross-tenant foreign keys also fail independently of RLS.

### 5. Catalog: ProductsModule and CategoriesModule

**Build:** staff product/category management, published-product browsing, filtering, sorting, cursor pagination, and product archiving.

**PostgreSQL focus:** composite indexes beginning with tenant filters, partial unique indexes for active SKUs if SKU reuse is allowed, many-to-many joins, `CHECK` constraints, and `JSONB` attributes.

**Validate:** tenant-local SKU uniqueness follows the chosen archive policy; invalid prices fail at the database; customers cannot browse draft products; pagination has a deterministic tie-breaker; SQL injection attempts remain data. Explain which fields belong in columns and which belong in JSONB.

### 6. Inventory: InventoryModule

**Build:** stock initialization, staff adjustments with reasons, current availability, and an append-only movement history.

**PostgreSQL focus:** atomic updates, row locks, lost updates, transaction isolation, and keeping a balance and its movement record consistent.

**Validate:** concurrent adjustments preserve the expected total; negative stock is rejected; a failure between balance and movement writes rolls both back; unauthorized changes fail. Choose and explain an atomic conditional update or explicit lock strategy.

### 7. Shopping cart: CartsModule

**Build:** get my cart, add items, change quantity, remove items, and display a current-price estimate.

**PostgreSQL focus:** upserts, composite uniqueness, aggregate queries, and query count control to avoid N+1 reads.

**Validate:** duplicate additions follow a documented quantity rule; quantities must be positive integers; another customer's cart is inaccessible; archived products cannot be newly added. Cart contents do not reserve stock or guarantee price; checkout revalidates both.

### 8. Checkout and orders: CheckoutModule and OrdersModule

**Build:** checkout, customer order history/detail, staff order access, and an explicit order state machine.

**Steps:**
1. Define valid order transitions and checkout request/response contracts.
2. Implement checkout on a single database client in one transaction.
3. Read authoritative prices, validate cart products, lock inventory in a stable order, and deduct stock.
4. Write the order and immutable item snapshots, record inventory movements, and clear the cart.
5. Add an idempotency key scoped to tenant and customer, a request fingerprint, and the stored result.
6. Add bounded retries where appropriate for deadlocks or serialization failures; never blindly repeat external side effects.

**PostgreSQL focus:** ACID, `READ COMMITTED` vs. `SERIALIZABLE`, `SELECT ... FOR UPDATE`, deadlocks, unique constraints as concurrency coordination, and idempotency.

**Validate:** with one unit left, two concurrent purchases produce exactly one successful purchase; failure halfway leaves no partial order; concurrent identical keys create one order and return its result; reusing a key for a different request is rejected; later product edits do not change order history.

### 9. Fake payments and cancellation: PaymentsModule

**Build:** fake payment success/failure events and cancellation rules. Checkout deducts stock; a terminal payment failure or eligible cancellation restores it exactly once. Define allowed transitions before implementation.

**PostgreSQL focus:** event uniqueness, conditional state transitions, row locking, and atomic updates across order, payment, and inventory records.

**Validate:** duplicate events do not repeat effects; cancellation and payment success racing have a defined valid outcome; cancelled orders cannot become paid accidentally; restoration occurs once; wrong-tenant events cannot alter an order.

### 10. Reliable background work: OutboxModule

**Build:** record an order event in the checkout transaction; process it in a worker to simulate a notification. Implement retries, retry limits, and failure visibility.

**PostgreSQL focus:** transactional outbox, `FOR UPDATE SKIP LOCKED`, short claim transactions, claim expiry/recovery, and at-least-once delivery.

**Validate:** rolled-back checkout produces no event; two workers do not concurrently own the same active claim; a crashed worker's event becomes eligible again; duplicate delivery is handled by an idempotent consumer. Do not hold a database transaction open during a network call.

### 11. Reporting and search: ReportsModule and SearchModule

**Build:** revenue by day, top products, customer spending rank, and searchable published products. Define revenue as paid orders and specify the reporting timezone.

**PostgreSQL focus:** CTEs, window functions, full-text search and GIN indexes, `EXPLAIN (ANALYZE, BUFFERS)`, and optional materialized views with a freshness policy.

**Validate:** totals match a hand-calculated fixture; reports remain tenant-scoped; tied rankings follow documented rules; measure representative queries before and after indexing. Use a disposable environment for query-plan experiments that execute writes.

### 12. Operations and final review: AuditModule and database maintenance

**Build:** selected staff-action audit records, structured request logging, migration/recovery notes, and a complete tenant-isolation test suite.

**PostgreSQL focus:** `pg_stat_statements` where available, lock investigation, MVCC, autovacuum, index write costs, safe migration design, and backup restoration practice. Partitioning is an optional experiment only after measurements justify it.

**Validate:** rebuild from migrations and seed data; restore a test backup; inspect an intentionally blocked transaction; document how to diagnose a slow query; pass end-to-end purchase, isolation, and concurrency scenarios. Do not log credentials or tokens.

If we later investigate production behavior, use NestJS Observe traces, errors, logs, and metrics as required by this repository's AGENTS.md. Local learning milestones do not require production telemetry setup.

## API outline

Exact DTOs, status codes, errors, and permissions will be specified before implementing each module.

| Area | Example endpoints |
| --- | --- |
| Authentication | `POST /auth/register`, `POST /auth/login`, `GET /users/me` |
| Shops | `POST /tenants`, `GET /tenants`, `POST /tenants/:tenantId/members` |
| Catalog | `GET/POST /tenants/:tenantId/products`, `PATCH /tenants/:tenantId/products/:id` |
| Inventory | `POST /tenants/:tenantId/inventory/:productId/adjustments` |
| Cart | `GET /tenants/:tenantId/cart`, `POST /tenants/:tenantId/cart/items` |
| Checkout | `POST /tenants/:tenantId/checkout` with `Idempotency-Key` |
| Orders | `GET /tenants/:tenantId/orders`, `POST /tenants/:tenantId/orders/:id/cancel` |
| Fake payment | A development-only event endpoint or test harness; never an unprotected production endpoint |
| Reporting | `GET /tenants/:tenantId/reports/revenue` |

## Progress tracker

- [ ] 1. Foundation and database connection
- [ ] 2. Users and authentication
- [ ] 3. Tenants and memberships
- [ ] 4. Row-level security and tenant transaction context
- [ ] 5. Products and categories
- [ ] 6. Inventory and concurrent updates
- [ ] 7. Shopping cart
- [ ] 8. Atomic checkout, orders, and idempotency
- [ ] 9. Fake payments and cancellation
- [ ] 10. Outbox and workers
- [ ] 11. Reporting, search, and query tuning
- [ ] 12. Operations and final validation

## First implementation task

Start with milestone 1 only: explain the existing `AppModule`, controller, and service request flow, then add PostgreSQL configuration, a connection pool, and a database readiness endpoint. We will review that foundation before creating business tables.
