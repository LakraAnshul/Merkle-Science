# Sanctum Sanctorum Bookstore — Project Notes

## 1. Live Deployment & Database Setup

- **Live Deployment URL**: https://sanctum-sanctorum-29ww.onrender.com
- **Seeded Member IDs for Testing**:
  - `ID: 1` — **Wong Li** (`wong@example.com`, Tier: `supreme`)
  - `ID: 2` — **Christine Palmer** (`christine@example.com`, Tier: `master`)
  - `ID: 3` — **Jonathan Pangborn** (`jonathan@example.com`, Tier: `adept`)
  - `ID: 4` — **Sara Lin** (`sara@example.com`, Tier: `apprentice`)
- **Database Architecture**:
  - **Local Development & Pytest**: Uses SQLite (in-memory `sqlite://` for tests, local file `sqlite:///./sanctum.db` for local dev).
  - **Production / Supabase**: Configured via the `SANCTUM_DATABASE_URL` environment variable. In `app/db.py`, `connect_args={"check_same_thread": False}` is applied conditionally only when the database URL scheme begins with `sqlite`, allowing PostgreSQL URLs (e.g. `postgresql://postgres:[PASSWORD]@db.fgfabitsyagixberteae.supabase.co:5432/postgres`) to connect smoothly. Environment variables are loaded automatically from `.env` via `python-dotenv`.

---

## 2. What Was Completed

All 5 modules specified in `SPEC.md` are completely implemented, fully compliant with requirements, and passing 100% of the acceptance criteria:

- [x] **Books Module**
  - Validation: title/author whitespace trimming with length bounds (1–200) verified after stripping.
  - ISBN-13 checksum verification (alternating weights 1, 3 with check digit calculation `(10 - sum%10) % 10`) and hyphen/space normalization.
  - Conflict detection: HTTP 409 on duplicate normalized ISBN.
  - Catalogue query (`GET /books`): filters (`q` substring on title/author, `restricted`, inclusive `min_price`/`max_price`), sorting (`title`, `-title`, `price`, `-price` with secondary `id` asc tie-breaking), and pagination (`limit`, `offset`, accurate pre-pagination `total`).
  - Partial update (`PATCH /books/{id}`): updates provided fields, ignores `isbn` and unknown fields without error.

- [x] **Members Module**
  - Validation: name length and whitespace stripping, case-insensitive email normalization with regex validation.
  - Conflict detection: HTTP 409 on duplicate email (case-insensitive).
  - Tier access control: corrected `tier_at_least` comparison (`>=` instead of `>`) so `master` and `supreme` members can access restricted books.
  - Member activity statistics (`GET /members/{id}/stats`): aggregates `orders_paid`, `total_spent_cents`, `active_loans`, `overdue_loans` (with strict boundary: active at exactly `due_at`), and `late_fees_cents` for returned loans.

- [x] **Orders Module**
  - Validation: rejects empty items list, negative or zero quantities, and duplicate book entries with HTTP 422.
  - Strict check order: 404 (missing member/book) $\rightarrow$ 403 (restricted book below master) $\rightarrow$ 409 (insufficient stock).
  - Data integrity: all-or-nothing stock reservation guarantees no partial stock mutation on failure.
  - Pricing & discounts: unit prices frozen at order placement time; tier discounts combined with bulk discount (+5% for total copies $\ge 10$), rounded down via floor division.
  - Order lifecycle: `POST /orders/{id}/pay` moves `pending` $\rightarrow$ `paid`; `POST /orders/{id}/cancel` restores reserved stock for all items and marks `cancelled`.

- [x] **Loans Module**
  - ORM model completion: added `due_at`, `returned_at`, and `late_fee_cents` to `Loan`.
  - Borrowing rules & tiered limits: apprentice (1), adept (3), master (5), supreme (unlimited).
  - Checks in order: 404 $\rightarrow$ 403 $\rightarrow$ 409 overdue loans $\rightarrow$ 409 duplicate unreturned copy $\rightarrow$ 409 tier limit $\rightarrow$ 409 out of stock.
  - Dynamic status: computed at read time (`returned`, `overdue`, or `active`).
  - Returns & late fees: restores book stock; calculates late fee at 25 cents per started day late (partial days rounded up via `math.ceil`), capped at book's return-time `price_cents`.
  - Member loans filtering: `GET /members/{id}/loans?status=active|overdue|returned`.

- [x] **Reports Module**
  - Best-selling books (`GET /reports/top-books`): aggregates quantities across paid orders only, excludes zero-sale books, sorts by `copies_sold` descending then `title` ascending, with configurable `limit` (1..50).

- [x] **Test Suite**: **202 / 202 passing** (`uv run pytest` output: `202 passed`).

---

## 3. Architecture & Design Decisions

1. **Strict Layered Separation**:
   - **Routers (`app/routers/`)**: Kept as thin HTTP adapters responsible solely for request parsing, dependency injection (`Session`, `clock`), calling the domain service, and serializing responses.
   - **Services (`app/services/`)**: Encapsulate all business logic, invariant enforcement, domain calculations, and database transactions.
   - **Schemas (`app/schemas.py`)**: Handle boundary validation (e.g. ISBN shape and checksum, item uniqueness, email format, string trimming) so invalid requests fail fast with 422 before touching the database.

2. **Data Integrity & Concurrency**:
   - All state mutations involving stock (order placement, order cancellation, borrowing, returning) are designed transactionally. In `create_order`, stock availability across all requested items is verified prior to any mutations, ensuring that partial checkouts never leave stock in an inconsistent state.
   - Price snapshotting: `OrderItem.unit_price_cents` captures book price at order placement so subsequent catalog price updates never alter past orders.

3. **Time Injection (`app.clock`)**:
   - Every operation requiring timestamps utilizes FastAPI's dependency injection (`Depends(get_now)`), ensuring that business logic never directly calls non-deterministic system clocks and allowing tests to fast-forward time with precision.

---

## 4. Database & Platform Trade-offs

- **SQLite for Testing & Local Development**:
  - SQLite provides zero-latency in-memory databases during automated testing (`pytest`), isolating tests completely and running all 202 tests in seconds without external network dependencies.
- **PostgreSQL / Supabase for Production**:
  - Serverless or cloud application hosts have ephemeral filesystems where a SQLite file resets on redeployment or idle container recycling. Migrating to Supabase PostgreSQL solves persistence and supports concurrent connections.
  - To support both seamlessly, `app/db.py` inspects the URL scheme and only passes SQLite-specific connection arguments (`check_same_thread: False`) when connected to SQLite.

---

## 5. Specification Inconsistencies & Bug Fixes

- **`tier_at_least` Bug**: In `app/services/members.py`, the starter code implemented `tier_at_least` using strict inequality (`>`), causing members with tier `master` to fail the check against `RESTRICTED_MIN_TIER` (`master`). Corrected to `>=`.
- **`cancel_order` Stock Restoration**: The starter code updated order status to `cancelled` but never restored book stock. Implemented item-by-item stock replenishment.
- **Collation Differences**: Notice was taken of database differences in case ordering between SQLite and PostgreSQL when ordering titles; queries rely on standard ASCII/lexicographic sorting with primary ordering on quantities/prices.

---

## 6. AI Usage

- **Tools Used**:
  - Antigravity AI assistant for codebase analysis, test suite diagnostics, scaffolding service functions, and reviewing boundary conditions.
- **How They Were Used**:
  - Investigated test failures in the initial repository state (73 passing, 125 failing, 4 errors).
  - Drafted ISBN-13 checksum algorithm and mathematical calculation of partial-day late fees.
  - Structured service functions with clean, explicit validation ordering according to `SPEC.md`.
- **Critical Review / Where the AI Was Overridden**:
  - In an initial suggestion for `create_order`, the AI proposed decrementing stock for each item inside a single combined loop that checked and updated stock sequentially. This violated the requirement for **all-or-nothing** stock updates: if the 3rd item in an order had insufficient stock, the first 2 items would have already had their database stock decremented. We overrode this by splitting order creation into distinct phases:
    1. Validate member and all books exist.
    2. Validate tier permissions on all books.
    3. Validate stock sufficiency across all items.
    4. Only upon 100% validation pass, decrement stock and commit the order.
