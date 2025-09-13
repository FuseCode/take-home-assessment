# Android Developer (Mid-Level) — Practical Exercise (≤ 4 hours)

## Goal

Build a small **Tickets** app in **Kotlin** with **Jetpack Compose** that lists work tickets and handles live updates + resilient actions:

* **List & paginate** tickets (cursor-style pagination)
* **Live updates** via **SSE** (or a simple polling fallback) with **auto-reconnect** and **resume token**
* **Optimistic actions**: “Bump” and “Done” with **offline queue & retry** (WorkManager)
* **Local cache** (Room) so the list shows instantly on relaunch
* Clean architecture, type-safe models, and a few targeted tests

Keep it compact and production-minded.

---

## Tech (suggested)

* Kotlin, **Jetpack Compose**, **Coroutines/Flow**
* DI: **Hilt** (or Koin)
* Networking: **OkHttp/Retrofit** (+ **okhttp-sse** for SSE) and **kotlinx.serialization**/**Moshi**
* Persistence: **Room**
* Background work: **WorkManager**
* Tests: JUnit4/5 + MockK (or Mockito), **Robolectric**/**Compose UI** tests, **MockWebServer** for API

> You may use **MockWebServer** as the backend during dev/tests. If you prefer, a tiny Ktor/Node stub is fine—keep setup minimal.

---

## Domain & API (mockable)

### Ticket model

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

### Endpoints (all tenant-scoped; for the exercise you can hardcode a tenant)

1. **GET `/tickets?limit=20&cursor=<opaque>`**

   * Sort: `createdAt DESC, id DESC`
   * Response: `{ "items": [Ticket...], "nextCursor": "..." }`

2. **POST `/tickets/{id}/bump`**

   * Body: `{ "idempotencyKey": "<uuid>" }` (client-generated)
   * Response: updated `Ticket` (status may become `in_progress`), includes new `version`.
   * Server may sporadically fail (simulate 500/timeout) to exercise retries.

3. **POST `/tickets/{id}/done`**

   * Preconditions: `status != done`
   * Response: updated `Ticket` (`done`) + new `version`.

4. **SSE `/tickets/stream?cursor=<resumeToken>`** *(optional but preferred)*

   * Emits events:

     * `ticket.updated` → `{ "ticket": Ticket, "cursor": "<resumeToken>" }`
   * Client should reconnect on drop and **resume** using the latest `cursor` it has seen.
   * If you choose **polling fallback**, call `GET /tickets?cursor=lastSeen` every N seconds and de-dupe by `id` + `version`.

> Provide fixtures under `app/src/test/resources/` (or in code) and wire MockWebServer responses/sequences. For SSE, you can emit lines like:

```json
event: ticket.updated
data: {"ticket": {...}, "cursor":"abc123"}

```

---

## App Requirements

### UI/UX

* **Home screen**: list of tickets (group by `station` optional), pull-to-refresh, infinite scroll (cursor).
* Each item: title, station, status chip, and two actions:

  * **Bump** (only if `status=queued`)
  * **Done** (if not already `done`)
* **Optimistic UI**: immediately reflect user intent (e.g., queued → in\_progress; in\_progress → done).

  * If the action ultimately fails after retries, revert state and show a non-blocking error snackbar.

### Data & Sync

* **Local cache** first: show Room contents instantly, then merge server results.
* **SSE (preferred)**: subscribe; on each event, upsert the ticket in Room and update UI.

  * Auto-reconnect with backoff and **resume via last cursor** (persisted in DataStore/Room).
* **Actions**:

  * Post actions with an **idempotency key** (UUID).
  * If offline or 5xx/timeout, enqueue a **WorkManager** job with exponential backoff; keep trying until success (or max attempts).
  * Persist minimal outbox row locally (`ticketId`, `op`, `idempotencyKey`, `payload`, `attempts`).

### Architecture

* Use a simple **Clean** approach: `ui` (Compose) → `viewmodel` → `repository` → `api/db`.
* State as **immutable UI models** (e.g., `TicketUi`), exposed via **Flow/StateFlow**.
* Error model: map to user-friendly toasts/snackbars; log details.

### Optional niceties (only if time permits)

* **ETag / If-None-Match** on a ticket detail fetch.
* Simple **retry-after** handling for SSE reconnects.

---

## Minimum Tests (aim for 3–6 concise tests)

* **Repository merge logic**: incoming SSE/poll events upsert correctly; newer `version` wins.
* **Action queue**: enqueue on failure/offline; WorkManager retry updates ticket on success; stale version triggers UI revert.
* **Pagination stability**: two pages (20 + remaining) with no duplicates when cursor is used.
* **UI smoke** (Compose): list renders items from Room and reacts to a mocked update.

> Use **MockWebServer** to simulate REST and (if you like) SSE; for SSE tests, you can simulate by invoking the repository stream.

---

## Deliverables

* Android Studio project with:

  * `app/` (Compose screens, ViewModels, DI, Repository, API, DB)
  * Tests (`androidTest/` and/or `test/`) with MockWebServer + Robolectric/Instrumented where needed
  * A small **README.md**: how to run, toggle SSE vs polling, sample flows (bump/done), and what’s left if time ran out
* Optional: a debug build variant that spins an in-app MockWebServer

---

## Timebox Guidance (≤ 4 hours)

Prioritize:

1. List + Room cache + pagination
2. SSE **or** polling with de-dupe + merge
3. Actions with optimistic update + WorkManager retry
4. 3–6 tests + README

---

## Evaluation Rubric (100 pts)

* **Correctness & UX (35)** — pagination stability, optimistic actions that reconcile correctly, durable retries
* **Architecture & code quality (25)** — layers, DI, flows, testability, readability
* **Offline & live updates (20)** — Room-first render, SSE/polling merge, resume token or sensible fallback
* **Testing (10)** — meaningful tests for merge/action queue/pagination
* **DX & docs (10)** — easy to build/run, clear README, sensible defaults

### Bonus (not required)

* Backoff and **resume token** persisted across app restarts
* Simple **Design System** tokens (colors/typography) and accessibility (content descriptions)
* Lightweight performance guard (no excessive recomposition)
