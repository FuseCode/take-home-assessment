# Angular Developer (Senior) — Practical Exercise (≤ 4 hours)

## Goal

Build a small **Tickets** web app in **Angular + TypeScript** showcasing senior-level cross-cutting skills:

* **JWT auth with auto-refresh** (single-flight refresh on 401)
* **Live updates** via **SSE** with **resume token** and backoff (polling fallback allowed)
* **Idempotent POST** actions using an **Idempotency-Key** interceptor
* **ETag / If-None-Match** caching for ticket detail
* **Keyset (cursor) pagination**
* Clean, reactive state (RxJS or Signals) with **OnPush** change detection
* A few targeted tests

Keep it compact and production-minded.

---

## Tech (suggested)

* Angular 16+ (standalone components OK), TypeScript, RxJS
* HttpClient, Router, **OnPush** components
* State: Signals **or** a minimal NgRx feature slice (your choice)
* Tests: **Jest** (or Karma) for unit; optional **Playwright/Cypress** single happy-path e2e
* Styling: anything light
* Mock backend:

  * Option A: **json-server** + tiny middleware for ETag, idempotency, SSE
  * Option B: minimal Node/Express stub included in the repo

Pick the fastest option; don’t over-engineer infra.

---

## API (mockable)

### Model

```ts
type Ticket = {
  id: string;                // uuid
  title: string;
  station: 'grill' | 'fryer' | 'expedite';
  status: 'queued' | 'in_progress' | 'done';
  createdAt: string;         // ISO
  version: number;           // increments on change
};
```

### Endpoints

1. **POST `/auth/token`** → `{ accessToken, refreshToken, expiresIn }`
   **POST `/auth/refresh`** → `{ accessToken, expiresIn }`

2. **GET `/tickets?limit=10&cursor=<opaque>`**
   Sort: `createdAt DESC, id DESC`
   Response: `{ items: Ticket[], nextCursor?: string }`

3. **GET `/tickets/:id`**
   Respond with `ETag` header; if request has `If-None-Match` equal to current ETag → **304** (no body).

4. **POST `/tickets/:id/done`**
   Headers: `Authorization: Bearer <jwt>`, **`Idempotency-Key: <uuid>`**
   Response: updated `Ticket` (status `done`, bumped `version`).
   Same key + same body (15 min) → **200 replay** (identical response).
   Same key + different body → **409** `{ code, message }`.

5. **SSE `/tickets/stream?cursor=<resume>`**
   Emits:

   ```json
   event: ticket.updated
   data: {"ticket": Ticket, "cursor":"abc123"}
   ```

   Client should reconnect with backoff and send the last cursor.

---

## App Requirements

### Pages

* **Tickets List**

  * Search (optional), infinite scroll using **keyset pagination**
  * Rows: title, station, status chip, **Done** button (hidden if already `done`)
  * Loading/empty/error states

* **Ticket Detail**

  * Uses **ETag / If-None-Match** via an **HttpInterceptor** that caches by `id`.
  * On 304, serve cached body to the component.

### Cross-cutting behavior

* **Auth & single-flight refresh**

  * `AuthInterceptor` attaches `Authorization`.
  * On **401**, perform **one** refresh request, queue/retry pending requests when refresh resolves; if refresh fails → sign-out state.

* **Live updates**

  * Subscribe to **SSE**; on event, upsert ticket in local store (newer `version` wins).
  * Reconnect with exponential backoff (cap \~30s) and **resume token** (persist in memory or localStorage).
  * If you prefer **polling fallback**, poll with `cursor` and de-dupe; note the choice in README.

* **Idempotent actions**

  * `IdempotencyKeyInterceptor` generates a UUID header per `POST /done`.
  * Double-clicking “Done” should result in a single logical update (replay OK).

* **State & performance**

  * Use **signals** or RxJS + `AsyncPipe`; components **OnPush**.
  * Avoid manual subscriptions in templates; prefer `*ngIf="vm$ | async as vm"` or signals.
  * Keep data service(s) slim and typed.

* **Errors & UX**

  * `HttpErrorInterceptor` maps non-2xx to `{code,message}` → toast/snackbar.
  * Optimistic UI for “Done”: flip immediately; if final failure, revert and show toast.

---

## Tests (aim for 5–7 concise tests)

1. **Single-flight refresh**

   * Two concurrent requests receive 401; ensure only **one** `/auth/refresh` occurs and both ultimately succeed.

2. **Idempotent action**

   * Two “Done” calls with same Idempotency-Key return the **same** ticket `version/id` (mock replay).

3. **ETag / 304**

   * First detail GET stores ETag; second with `If-None-Match` → treat 304 as cache hit and render cached body.

4. **SSE merge**

   * Simulate a `ticket.updated` with higher `version`; store updates and component reflects the new status.

5. **Pagination (keyset)**

   * Service test: two `loadMore()` calls yield 10 then remaining items, no duplicates.

6. *(Optional)* **Polling fallback**

   * When SSE disabled, periodic fetch with `cursor` merges correctly.

> Use HttpTestingController for unit tests; for SSE you may simulate by injecting a Subject/stream into the service, or mock EventSource.

---

## Deliverables

* Angular project with:

  * `app/` (standalone components), `core/` (interceptors/services/models), `features/tickets/`
  * Interceptors: `Auth`, `IdempotencyKey`, `EtagCache`, `HttpError`
  * A small state service (signals or NgRx) handling list, detail cache, and merges
  * Tests in `src/app/...` and/or `src/test/`
  * `README.md` with:

    * Install & run (app + mock server)
    * How SSE vs polling is toggled
    * Manual verification script (login → list → load more → open detail → click Done twice)

* Mock backend in `mock/` (json-server + middleware or Node script) implementing:

  * ETag/304 for detail
  * Idempotency replay for `POST /done`
  * SSE stream emitting `ticket.updated`

---

## Timebox Guidance (≤ 4 hours)

Prioritize:

1. **List + keyset pagination** and state service
2. **Auth + single-flight refresh**
3. **SSE (or polling) merge**
4. **ETag** interceptor for detail
5. **Idempotency** interceptor for Done
6. Tests (pick at least 4 of the above) + README

---

## Evaluation Rubric (100 pts)

* **Cross-cutting correctness (40)** — single-flight refresh, SSE/polling merge, ETag/304, idempotent action
* **Angular/RxJS quality (25)** — OnPush, clean streams/signals, interceptors, separation of concerns
* **UX & performance (15)** — loading/empty/error states, no duplicate rows, responsive updates
* **Testing (15)** — meaningful unit tests (and optional e2e)
* **DX & docs (5)** — easy run, clear README

### Bonus (not required)

* Persist last list `cursor` and SSE resume token across reloads
* Simple `KeysetPaginator` utility and `RetryBackoff` helper for SSE
* Basic Web Vitals/Lighthouse check and change-detection profiling notes
