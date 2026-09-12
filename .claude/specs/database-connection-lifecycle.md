# Spec Document

## 1. Overview

Replace the current per-call connection pattern in `database/db.py` with a **per-request cached connection** using Flask's `flask.g` and `app.teardown_appcontext`.

Today, every route/function that touches the database calls `get_db()` and opens (and sometimes closes) its own `sqlite3.connect()`. This step introduces a single connection per request, reused across the request via `g`, and guarantees it is closed when the request ends — regardless of success or error.

---

## 2. Depends on

[[Database-setup]] — `get_db()`, `init_db()`, `seed_db()` must already exist and work as implemented.

---

## 3. Routes

- No new routes
- No behavior change to any existing route (`/`, `/terms`, `/privacy`, `/register`, `/login`)
- Stubbed routes (`/logout`, `/profile`, `/expenses/*`) remain untouched by this step

---

## 4. Functions to Change/Add (`database/db.py`)

---

### A. `get_db()` (modify)

- If `flask.g` already has a `db` attribute for this request, return it
- Otherwise, open a new connection (same setup as today: `row_factory = sqlite3.Row`, `PRAGMA foreign_keys = ON`), store it on `g.db`, and return it
- Must be called within an active Flask app/request context (`current_app` or request context available)

---

### B. `close_db(e=None)` (new)

- Pops `db` off `g` if present and closes it
- Accepts an optional `e` (exception) argument, since Flask passes one to teardown functions
- Safe to call even if no connection was opened for the request (no-op)

---

## 5. Changes to `app.py`

- Import `close_db` alongside the existing `get_db`, `init_db`, `seed_db` imports
- Register it once at startup:
    
    ```python
    app.teardown_appcontext(close_db)
    ```
    
- `init_db()` and `seed_db()` startup calls inside `app.app_context()` remain unchanged — they can keep using `get_db()`/closing directly, or adopt the new cached pattern; either is acceptable since they run once at import time, outside a request

---

## 6. Files to Change

- `database/db.py` → modify `get_db()`, add `close_db()`
- `app.py` → import `close_db`, register `app.teardown_appcontext(close_db)`

---

## 7. Files to Create

- None

---

## 8. Rules for Implementation

- Do not change the schema, `init_db()` table creation, or `seed_db()` logic
- `PRAGMA foreign_keys = ON` must still be set whenever a new connection is actually opened
- Do not introduce a global/module-level connection — it must stay request-scoped via `g`
- Existing routes that call `get_db()` should not need any code changes — the caching is transparent to callers
- No ORM, no connection pooling library — plain `sqlite3` + `flask.g` only

---

## 9. Expected Behavior

- Within a single request, multiple calls to `get_db()` return the **same** connection object
- After the request ends, the connection is closed (verify via `sqlite3.Connection` no longer usable, or by tracking open connections in a test)
- An exception raised inside a route still results in the connection being closed (teardown runs on error too)
- Requests remain isolated from each other — no connection is reused across requests

---

## 10. Error Handling Expectations

- If `close_db` is called with no `g.db` set, it must not raise
- If a route raises an exception mid-request, `close_db` still runs (Flask guarantees teardown functions run even on unhandled exceptions) and the connection is closed cleanly

---

## 11. Definition of Done

- [ ]  `get_db()` reuses a cached connection from `g` within a request
- [ ]  A new connection is only opened once per request
- [ ]  `close_db()` is registered via `app.teardown_appcontext`
- [ ]  Connection is closed after every request, including ones that raise an exception
- [ ]  No behavior change to `/`, `/terms`, `/privacy`, `/register`, `/login`
- [ ]  `init_db()`/`seed_db()` startup flow still works unchanged
