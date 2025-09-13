# NestJS Developer (Mid-Level) — Practical Exercise (≤ 4 hours)

## Goal

Build a tiny **Orders API** in **NestJS** with:

* **Idempotent create** (via `Idempotency-Key`)
* **Optimistic locking** on update (`If-Match` version)
* **Keyset pagination** for listing
* A minimal **transactional outbox** row on close

Keep it production-minded but compact.

---

## Stack (pick your preferred ORM)

* **NestJS** (latest) with **TypeScript**
* **TypeORM** *or* **Prisma** (your choice)
* **PostgreSQL** (preferred) — SQLite acceptable if time is tight
* **Jest** + **Supertest** for tests
* **class-validator** / **class-transformer**

> If Postgres setup will exceed the timebox, use SQLite but keep schema compatible.

---

## Domain & Schema

### `orders`

* `id` UUID (PK)
* `tenant_id` TEXT
* `status` ENUM: `draft | confirmed | closed`
* `version` INT (start at 1; increment on each mutation)
* `total_cents` INT (nullable until confirmed)
* `created_at`, `updated_at` TIMESTAMPTZ

### `outbox`

* `id` UUID (PK)
* `event_type` TEXT (e.g., `orders.closed`)
* `order_id` UUID
* `tenant_id` TEXT
* `payload` JSON
* `published_at` TIMESTAMPTZ NULL

### `idempotency_keys`

* `tenant_id` TEXT
* `key` TEXT
* `response_json` JSON
* `created_at` TIMESTAMPTZ
  **PK/Unique:** `(tenant_id, key)`

Indexes you add for pagination/lookup are a plus.

---

## Endpoints

1. **POST `/orders`** — create draft (idempotent)

   * **Headers:**

     * `Authorization: Bearer <anything>` *or* `X-Tenant-Id: <tenant>` (pick one; document choice)
     * `Idempotency-Key: <string>` (required)
   * **Body:** `{ }` (no fields needed beyond tenant)
   * **Behavior:**

     * Creates a **draft** order for the current tenant.
     * Same `Idempotency-Key` + **same body** within 1h → replay the original response (**200**).
     * Same key + **different body** → **409** `{ code:"conflict", message:"..." }`.
   * **Response:** `{ id, tenantId, status, version, createdAt }`

2. **PATCH `/orders/:id/confirm`** — set total (optimistic lock)

   * **Headers:** `If-Match: "<version>"` (required)
   * **Body:** `{ "totalCents": 1234 }`
   * Allowed transition: `draft → confirmed`
   * On stale `If-Match` → **409** `{ code:"conflict", message:"stale version" }`
   * **Response:** `{ id, status:"confirmed", version, totalCents }`

3. **POST `/orders/:id/close`** — close & write **outbox** (one row)

   * Preconditions: status is `confirmed`
   * **In the same DB transaction**:

     * Update order to `closed`, bump `version`
     * Insert into `outbox` with `event_type="orders.closed"` and payload `{ orderId, tenantId, totalCents, closedAt }`
   * **Response:** `{ id, status:"closed", version }`

4. **GET `/orders?limit=10&cursor=<opaque>`** — **keyset pagination**

   * Sort by `created_at DESC, id DESC`
   * **Response:** `{ items:[...], nextCursor:"..." }` where cursor encodes last `(created_at,id)`
   * Must be **stable**: no dupes or skips across pages

---

## Cross-Cutting Requirements

* **Validation:** DTOs with `class-validator`; global ValidationPipe.
* **Error shape:** non-2xx returns `{ code, message }` (and optional `details`).
* **Tenant scoping:** All queries **must** filter by tenant (from `Authorization` or `X-Tenant-Id`—your choice).
* **OpenAPI:** Expose Swagger at `/docs` (basic schema is fine).
* **Project layout:** `AppModule` + `OrdersModule` (+ `Outbox`/`Common` if you like).

---

## Tests (Jest + Supertest)

Write concise integration tests (can use an in-memory/SQLite DB if needed):

1. **Idempotency**

   * Same `Idempotency-Key` + same body → same order `id` returned (replay)
   * Same key + different body → **409**

2. **Optimistic locking**

   * Confirm with correct `If-Match` succeeds and bumps `version`
   * Stale `If-Match` → **409**

3. **Close + outbox (transactional)**

   * After closing, order is `closed` and outbox has exactly **one** row linked to that order

4. **Pagination**

   * Create ≥15 orders, list with `limit=10` → pages of 10 then 5, **no duplicates**

*(Aim for \~6–10 tests total.)*

---

## Deliverables

* Repo with:

  * `src/` (Nest app, modules, controllers, services, entities/ORM models)
  * `test/` (integration tests)
  * Migrations (`typeorm` or `prisma migrate`) **or** a bootstrap SQL
  * `README.md` — how to run/migrate/test and a short cURL sequence
* Optional: `docker-compose.yml` for Postgres

---

## Timebox Guidance (≤ 4 hours)

Prioritize:

1. Create (idempotent)
2. Confirm (optimistic locking)
3. Close + outbox (same transaction)
4. Pagination
5. Tests & README

---

## Evaluation Rubric (100 pts)

* **Correctness & rules (35)** — idempotency, optimistic lock, transactional outbox, stable pagination
* **Code quality (20)** — NestJS module structure, DI, DTOs, validation, readability
* **Testing (20)** — useful integration tests, clear assertions
* **Data & SQL (15)** — schema choices, indexes, migrations or clear bootstrap
* **DX & Docs (10)** — easy run, clear README, sensible defaults

### Bonus (not required)

* Global exception filter mapping to `{code,message}`
* Basic logger (pino) or request-scoped correlation id
* Simple OpenTelemetry span around each handler
