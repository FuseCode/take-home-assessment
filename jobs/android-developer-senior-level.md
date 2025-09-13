# Android Developer (Senior) — Practical Exercise (≤ 4 hours)

## Goal

Build a small **Tickets** app in **Kotlin + Jetpack Compose** that demonstrates senior-level cross-cutting skills:

* **JWT auth with auto-refresh** (single-flight refresh on 401)
* **Live updates** via **SSE** with **resume token** and backoff
* **Idempotent POST actions** with a client-side **replay cache**
* **ETag / If-None-Match** caching for ticket detail
* **Offline queue & retry** (WorkManager)
* **Room** cache, clean architecture, and targeted tests

Keep scope tight and production-minded.

---

## Stack (suggested)

* Kotlin, Jetpack Compose, Coroutines/Flow
* DI: **Hilt** (or Koin)
* Networking: **OkHttp/Retrofit** (+ **okhttp-sse**), **kotlinx.serialization**/**Moshi**
* Persistence: **Room**, **DataStore** (for tokens/cursors)
* Background: **WorkManager**
* Tests: JUnit, MockK (or Mockito), **MockWebServer**, Compose UI test / Robolectric

---

## Mock API (use MockWebServer; hardcode a tenant)

### Auth

* **POST `/auth/token`** → `{ "accessToken": "jwt", "refreshToken": "r1", "expiresIn": 30 }`
* **POST `/auth/refresh`** (with `refreshToken`) → new `accessToken` (simulate expiry with 401)

### Tickets

1. **GET `/tickets?limit=20&cursor=<opaque>`**
   → `{ "items":[ Ticket... ], "nextCursor":"..." }`
   Sort by `createdAt DESC, id DESC`.

2. **GET `/tickets/{id}`**
   Returns ticket + **ETag**. If request has **If-None-Match** equal to current ETag → **304**.

3. **POST `/tickets/{id}/done`**
   Headers: `Authorization: Bearer <jwt>`, **`Idempotency-Key: <uuid>`**
   → Updated ticket (status `done`, new `version`).
   If server receives same key + same body within 15 min → **200 replay** (same response).
   If same key + different body → **409**.

4. **SSE `/tickets/stream?cursor=<resume>`**
   Events:

   ```json
   event: ticket.updated
   data: {"ticket": Ticket, "cursor":"abc123"}
   ```

   Client should reconnect with backoff and resume from last cursor.

    **Ticket model**

    ```json
    {
    "id": "uuid",
    "title": "Grill Burger x2",
    "station": "grill",
    "status": "queued|in_progress|done",
    "createdAt": "2024-01-01T12:00:00Z",
    "version": 3
    }
    ```

---

## App Requirements

### Architecture

* Modules (optional if time allows): `core-network`, `core-db`, `feature-tickets`.
* Layers: **UI (Compose)** → **ViewModel** → **Repository** → **API/DB**.
* **Repository** merges Room cache, paged fetches, and SSE updates (newer `version` wins).

### Auth (senior focus)

* Store tokens in **DataStore** (or EncryptedSharedPrefs if you prefer).
* **Auth interceptor** attaches `Authorization`.
* On **401**, trigger a **single-flight** refresh (only one refresh request even if multiple calls fail). Pending requests should wait and retry with the new token. If refresh fails → sign out state.

### Live updates

* **SSE** client with exponential backoff (e.g., 1s, 2s, 4s… capping at \~30s) and **resume token** from the last seen event (persisted).
* On event, upsert ticket in Room and update UI.

### Actions

* **Done** action: send POST with client-generated **Idempotency-Key**.
* If network fails or offline → place in a local **outbox** table and enqueue **WorkManager** job with exponential backoff until success (or max attempts).
* **Optimistic UI**: mark as done immediately; if final failure, revert and show a snackbar.

### Caching

* **ETag / If-None-Match** on `GET /tickets/{id}`: cache last ETag + body. On 304, serve cached body.

### UI

* Tickets list with **infinite scroll** (cursor pagination) and pull-to-refresh.
* Each item: title, station, status chip, **Done** button (hidden when already `done`).
* A simple detail screen that uses **ETag** caching on open.

### Logging (lightweight)

* Log a line per network call containing: endpoint, http code, **trace id** (generate per request), retry count. (Console is fine.)

---

## Tests (aim 5–7 concise tests)

1. **Token refresh (single-flight)**

   * Simulate 401 for two concurrent API calls; ensure only **one** `/auth/refresh` occurs; both calls eventually succeed.

2. **Idempotent POST**

   * Same `Idempotency-Key` twice → same response (replay) and only one logical state change.

3. **SSE merge**

   * Emit `ticket.updated` with higher `version`; repository updates Room and UI observer sees the change.

4. **Offline queue**

   * Make `done` fail (server down) → action enqueued; when server becomes available, WorkManager retry applies update.

5. **ETag / 304**

   * First detail fetch returns body + ETag. Second fetch with `If-None-Match` → 304; repository serves cached body.

6. **Pagination stability** (optional)

   * Two pages (20 + rest) without duplicates/missing items.

> Use **MockWebServer**; for SSE you can simulate by emitting chunked lines to the connection or by invoking a repository stream helper.

---

## Deliverables

* Android Studio project with:

  * Compose screens, ViewModels, Repository, API, DB
  * Interceptors: **Auth**, **Idempotency-Key**, (optional) **ETag** helper
  * Room entities/DAO, outbox table, DataStore for tokens/cursor
  * Tests in `test/` and/or `androidTest/` using MockWebServer
  * **README.md**: run steps, mock setup, flows (refresh, SSE, idempotent done), and what you’d do next if given more time

---

## Timebox Guidance (≤ 4 hours)

Prioritize:

1. **Auth + single-flight refresh**
2. **SSE (or polling fallback) + Room merge**
3. **Idempotent done + outbox retry**
4. **ETag** on detail
5. Tests & README

---

## Evaluation Rubric (100 pts)

* **Cross-cutting correctness (40)** — single-flight refresh, SSE resume, idempotent POST + retries, ETag/304
* **Architecture & code quality (25)** — clear layers, DI, flows, testability, readability
* **Offline & live sync (20)** — Room-first render, merge logic, backoff/reconnect
* **Testing (10)** — meaningful tests with MockWebServer and concurrency case
* **DX & docs (5)** — easy to run, clear README

### Bonus (not required)

* Network **tracing IDs** propagated via header and logged
* Simple **Benchmark** (Macrobenchmark or JUnit) for list render
* Accessibility check (content descriptions) and performance-friendly Compose (stable models, keys in LazyColumn)
