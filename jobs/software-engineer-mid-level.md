# Software Engineer (Mid-Level) — Practical Exercise (≤ 4 hours)

## Goal

Build a small **Orders API** with production-grade patterns:

* **Idempotent create** via `Idempotency-Key`
* **Optimistic locking** on update via `If-Match: <version>`
* **Keyset (cursor) pagination** for listing
* **Transactional outbox** row when closing an order
* **Tenant scoping** (simple header)

Keep it compact, clear, and testable. **Choose your preferred language and framework.**

---

## Tech Stack (Your Choice)

* **Language:** Any modern language (Java, Python, TypeScript/JavaScript, Go, C#, Kotlin, etc.)
* **Framework:** Any web framework appropriate for your language (Spring Boot, FastAPI, Express, NestJS, Gin, ASP.NET, Ktor, etc.)
* **Database:** **PostgreSQL** (preferred). If tight on time, you may use **SQLite**/**H2** locally but keep SQL portable.
* **ORM/Query Builder:** Your choice (JPA, SQLAlchemy, TypeORM, Prisma, Dapper, etc.)
* **Build Tool:** Standard for your language (Maven/Gradle, pip/poetry, npm/yarn, go mod, etc.)
* **Tests:** Standard testing framework for your language (JUnit, pytest, Jest, Go testing, etc.)

---

## Domain & Schema

### `orders`

* `id` UUID (PK)
* `tenant_id` TEXT/VARCHAR (scopes every query)
* `status` ENUM/VARCHAR: `draft | confirmed | closed`
* `version` INT (starts at 1; bump on each write)
* `total_cents` INT (nullable until confirmed)
* `created_at`, `updated_at` TIMESTAMPTZ

### `outbox`

* `id` UUID (PK)
* `event_type` TEXT/VARCHAR (e.g., `orders.closed`)
* `order_id` UUID
* `tenant_id` TEXT/VARCHAR
* `payload` JSON/JSONB
* `published_at` TIMESTAMPTZ NULL

### `idempotency_keys`

* `tenant_id` TEXT/VARCHAR
* `key` TEXT/VARCHAR
* `response_json` JSON/JSONB
* `created_at` TIMESTAMPTZ
  **Unique:** `(tenant_id, key)`

Indexes for pagination (e.g., `(tenant_id, created_at DESC, id DESC)`) are a plus.

---

## Endpoints

1. **POST `/orders`** — create draft (**idempotent**)

   * **Headers:**

     * `X-Tenant-Id: <tenant>` (required; simple dev auth)
     * `Idempotency-Key: <string>` (required)
   * **Body:** `{ }` (no fields needed beyond tenant)
   * **Behavior:**

     * Create a **draft** order; set `version=1`.
     * Same `Idempotency-Key` + **same body** within 1h → **200 replay** with identical response.
     * Same key + **different body** → **409** `{ "code":"conflict", "message":"..." }`.
   * **Response:** `{ "id", "tenantId", "status", "version", "createdAt" }`

2. **PATCH `/orders/{id}/confirm`** — set total (**optimistic locking**)

   * **Headers:** `If-Match: "<version>"` (required)
   * **Body:** `{ "totalCents": 1234 }`
   * Allowed transition: `draft → confirmed`; bump `version`.
   * On stale `If-Match` → **409** `{ "code":"conflict", "message":"stale version" }`.
   * **Response:** `{ "id", "status":"confirmed", "version", "totalCents" }`

3. **POST `/orders/{id}/close`** — close & **write outbox** (same transaction)

   * Preconditions: `status = confirmed`
   * In **one DB transaction**:

     * Update order → `closed`, bump `version`
     * Insert **one** outbox row with `event_type="orders.closed"` and payload `{ "orderId", "tenantId", "totalCents", "closedAt" }`
   * **Response:** `{ "id", "status":"closed", "version" }`

4. **GET `/orders?limit=10&cursor=<opaque>`** — **keyset pagination**

   * Sort: `created_at DESC, id DESC`
   * `cursor` encodes the last `(created_at, id)` as an opaque Base64 JSON (e.g., `{"ts": "...", "id": "..."}`)
   * **Response:** `{ "items":[...], "nextCursor":"..." }`
   * Must be **stable**: no duplicates/omissions across pages

---

## Cross-cutting

* **Validation:** Use appropriate validation for your framework. Clear error messages.
* **Error model:** Non-2xx → `{ "code", "message" }` (+ optional `"details"`).
* **API Documentation:** OpenAPI/Swagger or framework-native docs. Fields must match responses.
* **Time/IDs:** Use UTC and UUIDs; server generates IDs/timestamps.
* **Tenancy:** Every query filtered by `tenant_id` from header.

---

## Tests (write a few meaningful integration tests)

Use Testcontainers or similar for Postgres if you can; otherwise an in-memory DB is acceptable with a note.

1. **Idempotency**

   * Same `Idempotency-Key` + same body → same `order.id` (replay)
   * Same key + different body → **409**

2. **Optimistic locking**

   * Confirm with correct `If-Match` succeeds and bumps `version`
   * Stale `If-Match` → **409**

3. **Close + outbox (transactional)**

   * After closing, order is `closed` and exactly **one** outbox row exists for that order

4. **Pagination**

   * Create ≥15 orders; fetch with `limit=10` → 10 then 5 items, **no duplicates**

*(Aim for \~6–10 concise tests.)*

---

## Deliverables

* Repository with:

  * Build configuration files (pom.xml, build.gradle, package.json, go.mod, etc.)
  * Source code (controllers/routes, services, repositories, models/entities, DTOs)
  * Database migrations (Flyway, Liquibase, Alembic, etc.) **or** a `schema.sql`
  * Test files (integration tests)
  * `README.md` with instructions to run, migrate, test, and a short cURL sequence
* Optional: `docker-compose.yml` for Postgres

---

## Timebox Guidance (≤ 4 hours)

Prioritize in this order:

1. Create (idempotent)
2. Confirm (optimistic lock)
3. Close + outbox (same transaction)
4. Pagination
5. Tests & README polish

---

## Evaluation Rubric (100 pts)

* **Correctness & rules (35)** — idempotency, optimistic locking, transactional outbox, stable pagination
* **Code quality (20)** — structure, idiomatic code for your language, proper patterns (DI, separation of concerns), readability
* **Testing (20)** — meaningful integration tests, clear assertions
* **Data & SQL (15)** — schema choices, indexes, migrations or clear bootstrap
* **DX & Docs (10)** — easy run, clear README, sensible defaults

### Bonus (not required)

* Outbox publisher **interface** (no need to implement a real broker)
* Correlation ID in logs; structured logging
* Basic observability (OpenTelemetry spans or similar)
* Docker setup for easy local development

