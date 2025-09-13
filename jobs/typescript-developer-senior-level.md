# TypeScript Developer (Senior) — Practical Exercise (≤ 4 hours)

## Goal

Build a small **Webhook Outbox** in **TypeScript (Node.js)** with production-minded reliability:

* Per-aggregate **ordering** (seq 0…N must send in order)
* **Exponential backoff + jitter** and **Retry-After** handling
* **DLQ + replay**
* **HMAC** signing of requests
* Minimal **inspect** endpoints and concise tests

Keep the code compact and readable.

## Stack (suggested)

* Node.js 18+ with **TypeScript**
* Web: **Fastify** (or Express)
* ORM: **Prisma / Drizzle / TypeORM** (your choice)
* DB: **PostgreSQL** (preferred) — SQLite allowed if time is tight
* Validation: **zod** or **class-validator**
* Tests: **Jest** or **Vitest** + **Supertest**
* Logging: **pino** (or console)

## Data Model

`webhooks_outbox`

* `id` UUID PK
* `aggregate_id` TEXT (e.g., `orderId` or `intentId`)
* `seq` INT (0,1,2,…) **UNIQUE (aggregate\_id, seq)**
* `target_url` TEXT
* `payload` JSON/JSONB
* `status` ENUM(`pending`,`delivering`,`delivered`,`dead`)
* `attempts` INT DEFAULT 0
* `next_attempt_at` TIMESTAMPTZ
* `http_code` INT NULL
* `last_error` TEXT NULL
* `created_at` / `updated_at` TIMESTAMPTZ

*(Indexes on `(status, next_attempt_at)` and `(aggregate_id, seq)` are a plus.)*

## Endpoints

1. **POST `/webhooks`** — enqueue
   Body:

   ```json
   { "aggregateId": "A-1", "seq": 0, "targetUrl": "http://localhost:3000/receiver", "payload": { "hello": "world" } }
   ```

   * Insert row with `status='pending'`, `next_attempt_at=now()`.
   * Return the row summary.

2. **GET `/outbox?status=pending|delivering|delivered|dead&limit=50`** — inspect

   * Return `{ items:[{ id, aggregateId, seq, status, attempts, nextAttemptAt, httpCode }...] }`.

3. **POST `/outbox/:id/replay`** — DLQ replay

   * If current status is `dead`, set `status='pending'`, `attempts=0`, `next_attempt_at=now()` and return the updated row.

4. **POST `/receiver`** — simulated destination

   * Behavior selected by headers:

     * `X-Mode: success | flaky | rate-limit | fail-400`
     * `X-Aggregate-Id: <id>` (to vary behavior per aggregate)
   * Semantics:

     * `success` → **200**
     * `flaky` → first **2** hits for that aggregate return **500**, then **200**
     * `rate-limit` → **429** + `Retry-After: 2` seconds
     * `fail-400` → **400** (non-retryable)

## Worker (delivery loop)

Implement a background loop or an exported `tick()` function the tests can call:

1. Select rows where `status='pending' AND now() >= next_attempt_at` ordered by **(`aggregate_id`, `seq`)**.
2. **Ordering rule:** Only deliver `(aggregate_id, seq)` if `(aggregate_id, seq-1)` is `delivered` (or doesn’t exist).
3. Mark row `delivering`, POST `payload` to `target_url` with headers:

   * `Content-Type: application/json`
   * `X-Webhooks-Signature: t=<epoch_ms>, s=<hex(hmac_sha256(t + "." + body, HMAC_SECRET))>`
4. Handle result:

   * **2xx** → `delivered`, set `http_code`, clear `last_error`.
   * **408/429/5xx/network** → **retry**: increment `attempts`, set `status='pending'`, compute `next_attempt_at = now() + backoff`, honoring `Retry-After` if present.
   * **4xx (except 408/429)** → **dead** immediately.
5. **Backoff:** exponential with jitter

   * base: 1000ms, factor: 2.0, max: 300000ms, jitter ±10%
   * **Max attempts:** 10 → then `dead`.

> If you support concurrency, use SQL row leasing (`FOR UPDATE SKIP LOCKED`) or an application-level lease to avoid double-sends.

## Config (env)

* `HMAC_SECRET=dev-secret`
* `WEBHOOK_MAX_ATTEMPTS=10`
* `WEBHOOK_BACKOFF_BASE_MS=1000`
* `WEBHOOK_BACKOFF_MAX_MS=300000`

## Observability (minimal)

* Log one structured line per attempt: `{id, aggregateId, seq, attempt, status, httpCode, nextAttemptInMs}`.
* Optional: simple in-memory counters for delivered/retried/dead.

## Tests (6–10 concise tests)

Use Jest/Vitest + Supertest; you may drive the worker with `tick()` for determinism.

1. **Retry then succeed**

   * Enqueue `(A,0)` to `/receiver` with `X-Mode: flaky` → ends **delivered** with `attempts=3`.

2. **Fail-fast 400**

   * `X-Mode: fail-400` → row becomes **dead** on the first attempt.

3. **429 with Retry-After**

   * `X-Mode: rate-limit` → worker respects `Retry-After` (\~2s tolerance) and eventually delivers.

4. **Per-aggregate ordering**

   * Enqueue `(A,1)` **before** `(A,0)` → worker must not deliver `seq=1` until `seq=0` is `delivered`.

5. **Replay DLQ**

   * Force **dead**, call `/outbox/:id/replay`, set receiver to success, confirm delivery next cycle.

6. **HMAC present & valid** *(optional)*

   * Receiver recomputes signature using `HMAC_SECRET` and asserts correctness.

## Deliverables

* Repo with:

  * `src/` (server, routes, worker, db models)
  * `tests/` (integration tests)
  * Migrations or `schema.sql`
  * `README.md` (run, migrate, test, sample cURL)
* Optional: `docker-compose.yml` for Postgres.

## Timebox guidance (≤ 4 hours)

Prioritize in this order:

1. Schema + enqueue + happy-path delivery
2. Backoff/retry (500) + `429 Retry-After`
3. Ordering enforcement
4. DLQ + replay
5. HMAC + logs + README polish

## Evaluation rubric (100 pts)

* **Reliability logic (40)** — ordering, retry/backoff with jitter, Retry-After, terminal states
* **Data/transactions (20)** — schema, constraints, safe status transitions
* **Code quality (15)** — structure, typing, validation, readability
* **Testing (15)** — meaningful, stable tests
* **DX & docs (10)** — easy run, clear README, sensible defaults

### Bonus (not required)

* Prometheus-style `/metrics` with attempt counters
* Row leasing (`FOR UPDATE SKIP LOCKED`) or app-level lease
* Correlation id middleware that enriches logs
