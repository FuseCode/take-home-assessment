# NestJS Developer (Senior) — Practical Exercise (≤ 4 hours)

## Goal

Implement a **Webhook Outbox** in **NestJS** with **reliable delivery**: per-aggregate **ordering**, **exponential backoff + jitter**, **DLQ + replay**, and **HMAC** signing. Include tiny endpoints to **enqueue**, **inspect**, and **replay**. Provide a minimal **receiver** that returns success, flakiness, 429, or fail-fast 400 to exercise your worker.

## Stack

* **NestJS** (latest), **TypeScript**
* **TypeORM** *or* **Prisma** (your choice)
* **PostgreSQL** (preferred; SQLite acceptable if time is tight)
* **Jest** + **Supertest**
* **class-validator / class-transformer**
* Optional but nice: **@nestjs/config**, **@nestjs/schedule**, **pino** (or Nest logger)

## Data Model

`webhooks_outbox`

* `id` UUID PK
* `aggregate_id` TEXT (e.g., `intentId`)
* `seq` INT (monotonic per aggregate) — **UNIQUE (aggregate\_id, seq)**
* `target_url` TEXT
* `payload` JSON/JSONB
* `status` ENUM(`pending`,`delivering`,`delivered`,`dead`)
* `attempts` INT DEFAULT 0
* `next_attempt_at` TIMESTAMPTZ
* `http_code` INT NULL
* `last_error` TEXT NULL
* `created_at` / `updated_at` TIMESTAMPTZ

> If your ORM supports it, use a query that can **lease** rows safely (e.g., `FOR UPDATE SKIP LOCKED`) when picking work.

## Endpoints

1. **POST `/webhooks/enqueue`**
   Body: `{ "aggregateId":"A-1", "seq":0, "targetUrl":"http://...", "payload":{...} }`

   * Insert row with `status='pending'`, `next_attempt_at=now()`
   * Return the row summary

2. **GET `/webhooks/outbox?status=pending|delivering|delivered|dead&limit=50`**

   * Return a list: `{ id, aggregateId, seq, status, attempts, nextAttemptAt, httpCode }`

3. **POST `/webhooks/outbox/:id/replay`**

   * If current status is `dead`, set `status='pending'`, `attempts=0`, `next_attempt_at=now()`; return updated row

4. **(Receiver) POST `/receiver`**

   * Behavior toggled by headers:

     * `X-Mode: success | flaky | rate-limit | fail-400`
     * `X-Aggregate-Id: <id>` (to vary behavior per aggregate)
   * `success` → **200**
   * `flaky` → first **2** hits for that aggregate return **500**, then **200**
   * `rate-limit` → **429** + `Retry-After: 2` seconds
   * `fail-400` → **400** (non-retryable)

## Worker (delivery loop)

* Start a background processor (e.g., via **@nestjs/schedule** interval or an app bootstrap loop):

  1. Select rows where `status='pending' AND now() >= next_attempt_at`
     **order by (`aggregate_id`, `seq`)**
  2. **Ordering rule:** Deliver `(aggregate_id, seq)` only if `(aggregate_id, seq-1)` is **delivered** (or doesn’t exist)
  3. Mark row `delivering`, attempt HTTP POST to `target_url` with `payload`
  4. Add HMAC header:
     `X-Webhooks-Signature: t=<epoch_ms>, s=<hex(hmac_sha256(t + "." + body, HMAC_SECRET))>`
  5. Handle responses:

     * **2xx** → `delivered` (set `http_code`, clear `last_error`)
     * **408/429/5xx/network** → **retry**: set `status='pending'`, increment `attempts`, compute `next_attempt_at = now() + backoff`
     * **4xx (except 408/429)** → mark **dead**
* **Backoff**: exponential with jitter

  * base: 1000ms, factor: 2.0, max: 300000ms, jitter: ±10%
  * **Max attempts**: 10 (then `dead`)
* Log each attempt (structured): `{id, aggregateId, seq, attempt, status, httpCode, nextAttemptInMs}`

## Config (env)

* `HMAC_SECRET=dev-secret`
* `WEBHOOK_MAX_ATTEMPTS=10`
* `WEBHOOK_BACKOFF_BASE_MS=1000`
* `WEBHOOK_BACKOFF_MAX_MS=300000`

## Validation & DX

* DTOs with `class-validator`; global `ValidationPipe`
* Global exception filter mapping errors to `{ code, message }`
* Swagger at `/docs` (basic is fine)
* Use a **ConfigModule** (if you like) to type env vars

## Tests (Jest + Supertest, end-to-end-ish)

Write concise tests (6–10 total):

1. **Retry then succeed**

   * Enqueue `(A, seq=0)` to `/receiver` with `X-Mode: flaky`
   * Worker should mark it `delivered` with `attempts=3`

2. **Fail-fast 400**

   * `X-Mode: fail-400` → row becomes `dead` after first attempt

3. **429 with Retry-After**

   * `X-Mode: rate-limit` → worker respects `Retry-After` (\~2s tolerance), then eventually delivers

4. **Per-aggregate ordering**

   * Enqueue `(A, seq=1)` **before** `(A, seq=0)`
   * Worker must **not** deliver `seq=1` until `seq=0` is `delivered`

5. **Replay DLQ**

   * Force `dead`, call `/webhooks/outbox/:id/replay`, set `X-Mode: success`, verify delivery on next cycle

6. **HMAC present** *(optional)*

   * Receiver recomputes signature; assert the header is valid

> Tests can drive the worker by calling an exported `tick()` or by shortening the schedule to run frequently during tests.

## Deliverables

* Repository with:

  * `src/` (modules: `OutboxModule`, `ReceiverModule`, `AppModule`; services, controllers, entities)
  * Migrations (TypeORM/Prisma) **or** a bootstrap SQL
  * `test/` (integration tests with Supertest)
  * `README.md` (run, migrate, test, and a short cURL flow)
* Optional: `docker-compose.yml` for Postgres

## Timebox Guidance (≤ 4 hours)

Prioritize:

1. Schema + enqueue + single-worker happy path
2. Backoff & retry (500) and `429 Retry-After`
3. Ordering (seq)
4. DLQ + replay
5. HMAC + logs + README polish

## Evaluation Rubric (100 pts)

* **Reliability logic (40)** — ordering, retry/backoff with jitter, Retry-After handling, correct terminal states
* **Data/transactions (20)** — schema, unique constraints, safe state transitions
* **Code quality (15)** — NestJS module boundaries, DI, validation, readability
* **Testing (15)** — meaningful integration tests, stable assertions
* **DX & docs (10)** — easy to run, clear README, sensible defaults

### Bonus (not required)

* Row leasing with `FOR UPDATE SKIP LOCKED` (or equivalent)
* `/metrics` endpoint with delivered/retried/dead counters
* Request-scoped correlation id (interceptor) and log enrichment
