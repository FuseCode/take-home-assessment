# Python/FastAPI Developer (Senior) — Practical Exercise (≤ 4 hours)

## Goal

Implement a **Webhook Outbox** with **reliable delivery**: per-aggregate **ordering**, **exponential backoff + jitter**, **DLQ + replay**, and **HMAC** signing. Provide tiny endpoints to **enqueue** webhooks and **inspect/replay** the outbox. Include a minimal **receiver** to simulate success, flakiness, 429 rate limit, and fail-fast 400.

## Core Requirements

### Stack

* **FastAPI** (async) + **httpx** (async client)
* **SQLAlchemy 2.x (async)** + **Alembic** (or a bootstrap SQL if short on time)
* **PostgreSQL** (preferred) — SQLite acceptable if time-constrained (keep types compatible)
* **pytest** + `pytest-asyncio`

### Data Model

`webhooks_outbox`

* `id UUID PK`
* `aggregate_id TEXT` (e.g., `intentId`)
* `seq INT` (delivery order per aggregate; 0,1,2,…) — **UNIQUE (aggregate\_id, seq)**
* `target_url TEXT`
* `payload JSONB` (or JSON)
* `status TEXT CHECK IN ('pending','delivering','delivered','dead')`
* `attempts INT DEFAULT 0`
* `next_attempt_at TIMESTAMPTZ`
* `last_error TEXT NULL`
* `http_code INT NULL`
* `created_at/updated_at TIMESTAMPTZ`

> Optional: a small `hmac_secret` env var; store nothing sensitive in DB.

### Endpoints

1. **POST `/enqueue`**
   Body: `{ "aggregateId": "A-123", "seq": 0, "targetUrl": "http://...", "payload": {...} }`

   * Inserts a row with `status='pending'`, `next_attempt_at=now()`
   * Returns outbox row (id, status)

2. **GET `/outbox`**
   Query: `status=pending|delivering|delivered|dead` (optional, default all), `limit=50`

   * Returns latest rows with minimal fields (id, aggregateId, seq, status, attempts, nextAttemptAt, httpCode)

3. **POST `/outbox/{id}/replay`**

   * Sets `status='pending'`, `attempts=0`, `next_attempt_at=now()` if current status is `dead`
   * Returns updated row

4. **(Receiver) POST `/receiver`** (same service or a sub-app)

   * Behavior switches via header `X-Mode: success|flaky|rate-limit|fail-400` **per aggregate** (use `X-Aggregate-Id`)
   * `success` → 200
   * `flaky` → first **2** requests for that aggregate return **500**, then 200
   * `rate-limit` → **429** + header `Retry-After: 2` seconds
   * `fail-400` → **400** (non-retryable)

### Delivery Worker

* Background task (startup) or CLI (invoked in tests) that:

  1. Fetches rows where `status='pending' AND now() >= next_attempt_at` **ordered by (`aggregate_id`, `seq`)**.
  2. **Ordering rule:** only deliver a row if `(aggregate_id, seq-1)` is either non-existent or already `delivered`.
  3. Marks row `delivering`, attempts delivery with **httpx** (`timeout=5s`).
  4. Sign payload with **HMAC SHA-256** header
     `X-Webhooks-Signature: t=<epoch_ms>, s=<hex(hmac(t + "." + body))>`.
  5. HTTP results:

     * **2xx** → `delivered` (set `http_code`, clear `last_error`)
     * **408/429/5xx/network** → **retry** with exponential backoff + jitter
     * **4xx (except 408/429)** → mark **dead** immediately
  6. **Backoff:** `base=1000ms`, `max=300000ms`, `factor=2.0`, `jitter=±10%`.
  7. **Max attempts:** e.g., `10` (configurable). If exceeded → `dead`.

> Concurrency: single worker is fine. If you add more, use `FOR UPDATE SKIP LOCKED` or equivalent to avoid double-send.

### Config (env)

* `HMAC_SECRET` (default `"dev-secret"`)
* `WEBHOOK_MAX_ATTEMPTS` (default `10`)
* `WEBHOOK_BACKOFF_BASE_MS` (default `1000`)
* `WEBHOOK_BACKOFF_MAX_MS` (default `300000`)

### Observability (minimal)

* One **structured log** per attempt: `{id, aggregateId, seq, attempt, status, httpCode, nextAttemptInMs}`
* Optional: simple in-process counters (delivered / retried / dead)

## Tests (write a few end-to-end with `pytest-asyncio`)

1. **Retry then succeed**

   * Enqueue seq `0` to `/receiver` with `X-Mode: flaky`
   * Worker delivers after 3rd try; row `status=delivered`, `attempts=3`
2. **Fail-fast 400**

   * `X-Mode: fail-400` → row becomes `dead` after first attempt
3. **429 with Retry-After**

   * `X-Mode: rate-limit` → worker waits \~2s (tolerance ok), then eventually delivers (or retries until success if you switch mode mid-test)
4. **Per-aggregate ordering**

   * Enqueue `(aggregateId=A, seq=1)` **before** `(A, seq=0)`
   * Worker must **not** deliver seq 1 until seq 0 is `delivered`
5. **Replay DLQ**

   * Force `dead`, call `/outbox/{id}/replay`, set `X-Mode: success`, ensure delivery on next cycle
6. **HMAC present** (optional)

   * Receiver recomputes the HMAC and asserts header signature is valid

*(Aim for \~6–10 concise tests.)*

## Deliverables

* Repo with:

  * `/app` (FastAPI app: api + worker + receiver)
  * `/db` (SQLAlchemy models + Alembic or `schema.sql`)
  * `/tests` (pytest)
  * `README.md` with run steps (venv/poetry/pip), env vars, and a quick cURL flow
* Optional: `docker-compose.yml` for Postgres

## Timebox Guidance (≤ 4 hours)

Prioritize in this order:

1. Schema + **enqueue** + **worker** happy path (2xx)
2. **Backoff & retry** (500) and **429 Retry-After**
3. **Ordering** enforcement (seq)
4. **DLQ + replay**
5. HMAC + logs + README polish

## Evaluation Rubric (100 pts)

* **Reliability logic (40)** — ordering, retry/backoff with jitter, correct status transitions, Retry-After
* **Data/transactions (20)** — schema, unique constraints, safe state changes
* **Code quality (15)** — structure, typing, readability
* **Testing (15)** — meaningful async tests, stable assertions
* **DX & docs (10)** — easy run, clear README, sensible defaults

### Bonus (not required)

* `FOR UPDATE SKIP LOCKED` style selection or application-level leasing
* Exposing simple `/metrics` (attempts/delivered/dead)
* Small CLI to run a single worker tick for deterministic tests
