# Angular Developer (Mid-Level) — Practical Exercise (≤ 4 hours)

## Goal

Build a small **Catalog** web app in **Angular + TypeScript** that shows solid fundamentals:

* **List & search** products with **keyset (cursor) pagination**
* **Product detail** with **ETag / If-None-Match** caching
* **Create product** with **Idempotency-Key** header
* Clean state with **RxJS** streams and **OnPush** components
* A few targeted tests

Keep it compact, production-minded, and easy to run.

---

## Tech (suggested)

* **Angular 16+** (standalone components are fine)
* **TypeScript**, **RxJS**
* **Angular Router**, **HttpClient**
* **State**: simple service with `BehaviorSubject`/`signal`, or a minimal **NgRx** slice if you prefer
* **Testing**: **Jest** (or Karma) for unit; **Cypress** or **Playwright** for one happy-path e2e
* **Styling**: anything light (CSS/SCSS/Tailwind)
* **Mock backend**:

  * Option A (fast): **json-server** with a tiny middleware for ETag & idempotency
  * Option B: a minimal **Node/Express** stub you include in the repo

> Choose the fastest option; don’t overbuild infra.

---

## API (mockable)

### Models

```ts
type Product = {
  id: string;           // uuid
  sku: string;
  name: string;
  priceCents: number;
  createdAt: string;    // ISO
  updatedAt: string;    // ISO
  version: number;      // int
};
```

### Endpoints

1. **GET `/products?search=&limit=10&cursor=<opaque>`**

   * Sort: `createdAt DESC, id DESC`
   * Response: `{ items: Product[], nextCursor?: string }`
   * `cursor` encodes the last `(createdAt, id)` (opaque string is fine)

2. **GET `/products/:id`**

   * Returns `Product` and **`ETag`** header
   * If request has **`If-None-Match`** equal to current ETag → **304** (no body)

3. **POST `/products`**

   * Headers: **`Idempotency-Key: <uuid>`** (required)
   * Body: `{ sku, name, priceCents }`
   * On same key + same body (within 15 min) → **200 replay** (identical response)
   * On same key + different body → **409** `{ code, message }`

> Your mock can keep idempotency state in memory.

---

## App Requirements

### Pages & Features

1. **Products List**

   * Search box (client sends `?search=...`)
   * List results with **infinite scroll** or “Load more” using **keyset pagination**
   * Skeleton/loading states, empty state
   * Rows show `sku`, `name`, formatted price

2. **Product Detail**

   * Navigates to `/products/:id`
   * Uses **ETag / If-None-Match** via an **HttpInterceptor**:

     * Store ETag + body per `id` in a small in-memory cache
     * On 304, serve cached body to the component
   * Show `version` and timestamps

3. **Create Product**

   * Modal or page with simple form: `sku`, `name`, `priceCents`
   * On submit, generate a **UUID Idempotency-Key** in an **HttpInterceptor** (or in the service)
   * If the user clicks submit twice quickly, expect **200 replay** (same product id)

### Architecture & Quality

* Components use **OnPush**; data via **observables/signals**; no manual `subscribe` in templates
* A dedicated **DataService** encapsulates HTTP calls and pagination cursors
* Basic error handling: toasts/snackbars from an **HttpErrorInterceptor** mapping `{code,message}`
* Keep it small and idiomatic; avoid over-engineering

---

## Tests (aim for 4–6 concise tests)

1. **List pagination (keyset)**

   * Service test: calling `loadMore()` twice yields distinct items, no duplicates; last call contains `nextCursor` from the server

2. **ETag / 304**

   * Interceptor test: first `GET /products/:id` stores ETag; second request with `If-None-Match` → treat 304 as cache hit and emit cached body

3. **Idempotent create**

   * Service test: two `create()` calls with the same `Idempotency-Key` return the **same** `id` (mock backend replay)

4. **Component smoke**

   * List component renders initial items and appends on “Load more”; uses **OnPush** (no stale UI)

> If you add an e2e: one happy-path that searches, loads, opens detail, and creates a product.

---

## Deliverables

* Angular project with:

  * `app/` standalone or module-based structure
  * `core/` (services, interceptors, models)
  * `features/` (`products-list`, `product-detail`, `product-create`)
  * Tests in `src/app/...` and/or `src/test/`
  * `README.md` with:

    * **Install & run** (app + mock server)
    * Notes on ETag caching & Idempotency-Key
    * A short manual test script (search, load more, open detail, create twice)

* Mock backend:

  * `mock/` folder with **json-server** config + middleware for ETag & idempotency
    **or** a tiny `node mock/server.js`

---

## Timebox Guidance (≤ 4 hours)

Prioritize:

1. **List + pagination** and **DataService**
2. **ETag interceptor** + detail page
3. **Create with Idempotency-Key** (interceptor or service)
4. **Tests** + README polish

---

## Evaluation Rubric (100 pts)

* **HTTP & state correctness (35)** — stable keyset pagination, correct 304 cache handling, idempotent create
* **Angular & RxJS quality (25)** — OnPush, clean streams/signals, interceptors, separation of concerns
* **UI/UX basics (15)** — loading/empty/error states, accessible labels, formatting
* **Testing (15)** — meaningful unit tests (and optional e2e) with clear assertions
* **DX & docs (10)** — easy run, clear README, sensible defaults

### Bonus (not required)

* Persist last list `cursor` in `sessionStorage` to keep place on refresh
* Reusable `KeysetPaginator` utility (observable of pages + `loadMore()`)
* Simple **retry/backoff** for transient 5xx on list/detail
