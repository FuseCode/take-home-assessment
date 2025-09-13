# Kotlin Developer (Senior) — Practical Exercise (≤ 4 hours)

## Goal

Implement a **Webhook Outbox** in **Kotlin** with production-minded reliability:

* Per-aggregate **ordering** (seq 0…N must deliver in order)
* **Exponential backoff + jitter** and **Retry-After** handling
* **DLQ + replay**
* **HMAC** signing
* Minimal **enqueue/inspect/replay** endpoints and concise tests

Keep the solution compact, readable, and testable.

---

## Stack (choose your flavor)

* **Kotlin 1.9+**
* Web: **Spring Boot 3** (WebFlux or MVC) *or* **Ktor**
* Persistence: **jOOQ** or **Spring Data JPA/Hibernate** (your choice)
* DB: **PostgreSQL** (preferred). If time is tight, H2/SQLite is OK—keep SQL portable.
* HTTP client: **WebClient** (Spring) or **Ktor HttpClient**
* Build: **Gradle (Kotlin DSL)**
* Tests: **JUnit 5**, **AssertJ**, **MockK**, **Testcontainers** (optional but nice)

---

## Data Model

`webhooks_outbox`

* `id` **UUID PK**
* `aggregate_id` **TEXT** (e.g., `intentId`)
* `seq` **INT** (monotonic per aggregate) — **UNIQUE (aggregate\_id, seq)**
* `target_url` **TEXT**
* `payload` **JSON/JSONB** (or TEXT)
* `status` **ENUM**(`pending`,`delivering`,`delivered`,`dead`)
* `attempts` **INT** DEFAULT 0
* `next_attempt_at` **TIMESTAMPTZ**
* `http_code` **INT** NULL
* `last_error` **TEXT** NULL
* `created_at`, `updated_at` **TIMESTAMPTZ**

> Helpful indexes: `(status, next_attempt_at)`, `(aggregate_id, seq)`.

---

## Endpoints

1. **POST `/webhooks/enqueue`**
   Body:

   ```json
   { "aggregateId": "A-1", "seq": 0, "targetUrl": "http://localhost:8080/receiver", "payload": { "hello": "world" } }
   ```

   * Insert row with `status='pending'`, `next_attempt_at=now()`.
   * Return summary `{id, aggregateId, seq, status}`.

2. **GET `/webhooks/outbox?status=pending|delivering|delivered|dead&limit=50`**

   * Return `{ items:[{ id, aggregateId, seq, status, attempts, nextAttemptAt, httpCode }] }`.

3. **POST `/webhooks/outbox/{id}/replay`**

   * If current status is `dead`, set `status='pending'`, `attempts=0`, `next_attempt_at=now()`; return updated row.

4. **(Receiver) POST `/receiver`** *(same app, minimal)*

   * Behavior via headers:

     * `X-Mode: success | flaky | rate-limit | fail-400`
     * `X-Aggregate-Id: <id>` (to vary behavior per aggregate)
   * Semantics:

     * `success` → **200**
     * `flaky` → first **2** requests for that aggregate return **500**, then **200**
     * `rate-limit` → **429** with `Retry-After: 2` seconds
     * `fail-400` → **400** (non-retryable)

---

## Worker (delivery loop)

Implement as a scheduled task, coroutine loop, or an exported `tick()` for tests:

1. **Select due work:** rows where `status='pending' AND now() >= next_attempt_at`, **ordered by (`aggregate_id`, `seq`)**.
2. **Ordering rule:** deliver `(aggregate, seq)` **only if** `(aggregate, seq-1)` is `delivered` (or doesn’t exist).
3. **Lease & send:** mark row `delivering` (transaction), POST `payload` to `target_url` with headers:

   * `Content-Type: application/json`
   * `X-Webhooks-Signature: t=<epoch_ms>, s=<hex(hmac_sha256(t + "." + body, HMAC_SECRET))>`
4. **Handle result:**

   * **2xx** → set `delivered`, store `http_code`, clear `last_error`
   * **408/429/5xx/network** → **retry**: increment `attempts`, set `status='pending'`, set `next_attempt_at = now() + backoff`, honoring **`Retry-After`** if present
   * **4xx (except 408/429)** → mark **dead**
5. **Backoff:** exponential with jitter

   * base **1000ms**, factor **2.0**, max **300000ms**, jitter **±10%**
   * **Max attempts:** **10** → then `dead`

> If you support concurrency, use `FOR UPDATE SKIP LOCKED` (jOOQ) or optimistic row leasing to avoid double-sends.

---

## Config (env or `application.yml`)

* `HMAC_SECRET=dev-secret`
* `WEBHOOK_MAX_ATTEMPTS=10`
* `WEBHOOK_BACKOFF_BASE_MS=1000`
* `WEBHOOK_BACKOFF_MAX_MS=300000`

---

## Observability (minimal)

* One structured **log** per attempt: `{id, aggregateId, seq, attempt, status, httpCode, nextAttemptInMs}`.
* Optional: Micrometer counters (delivered/retried/dead) or a tiny `/metrics` endpoint.

---

## Tests (6–10 concise, integration-oriented)

Use **JUnit 5**; you can drive the worker via `tick()` for determinism.

1. **Retry then succeed**

   * Enqueue `(A,0)` to `/receiver` with `X-Mode: flaky` → ends **delivered** with `attempts=3`.

2. **Fail-fast 400**

   * `X-Mode: fail-400` → row becomes **dead** on first attempt.

3. **429 with Retry-After**

   * `X-Mode: rate-limit` → worker respects \~2s backoff from header, then delivers.

4. **Per-aggregate ordering**

   * Enqueue `(A,1)` **before** `(A,0)` → `seq=1` must wait until `seq=0` is **delivered**.

5. **Replay DLQ**

   * Force **dead**, call `/webhooks/outbox/{id}/replay`, set receiver to success, confirm delivery next cycle.

6. **HMAC present (optional)**

   * Receiver recomputes HMAC using `HMAC_SECRET` and asserts signature.

> Using Testcontainers Postgres is great; if you use H2, note any behavior differences.

---

## Deliverables

* Repository with:

  * `build.gradle.kts`, `settings.gradle.kts`
  * `src/main/kotlin` (modules, controllers/routes, services, repo/DAO)
  * Migrations (`Flyway`/`Liquibase`) **or** `schema.sql`
  * `src/test/kotlin` tests (JUnit 5; worker `tick()` or fast schedule)
  * `README.md` with run/migrate/test and a short cURL flow
* Optional: `docker-compose.yml` for Postgres

---

## Timebox Guidance (≤ 4 hours)

Prioritize:

1. Schema + **enqueue** + happy-path delivery
2. **Backoff & retry** (500) + **429 Retry-After**
3. **Ordering** enforcement
4. **DLQ + replay**
5. HMAC + logs + README polish

---

## Evaluation Rubric (100 pts)

* **Reliability logic (40)** — ordering, retry/backoff with jitter, Retry-After, correct terminal states
* **Data/transactions (20)** — schema, constraints, safe state transitions
* **Code quality (15)** — idiomatic Kotlin, structure, DI, readability
* **Testing (15)** — meaningful integration tests, stable assertions
* **DX & docs (10)** — easy run, clear README, sensible defaults

### Bonus (not required)

* Row leasing via `FOR UPDATE SKIP LOCKED` (or app-level lease)
* Micrometer metrics exposed at `/actuator/prometheus` (Spring) or simple `/metrics`
* Correlation ID in MDC and log enrichment
