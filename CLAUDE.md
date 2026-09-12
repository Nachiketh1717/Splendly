# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Spendly is a Flask-based personal expense tracker built as a guided, step-by-step learning project. Routes and files contain markers like `# Step 1 — Database Setup` or `"coming in Step 3"` — these are intentional placeholders for a student to implement incrementally, not bugs to fix silently. When touching a stubbed route or file, implement it in line with what the marker/comment describes rather than assuming it's dead code.

## Commands

```bash
# Setup
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Run the app (Flask dev server, debug mode, port 5001)
python app.py

# Run tests
pytest
```

There is no lint/format tooling configured in this repo yet.

## Architecture

- **`app.py`** — single-file Flask app with all routes. Implemented: `/`, `/terms`, `/privacy` (GET only, render templates); `/register` and `/login` (GET renders the form, POST validates and processes it — see below). Stubbed/unimplemented: `/logout`, `/profile`, `/expenses/add`, `/expenses/<id>/edit`, `/expenses/<id>/delete` — these currently return plain placeholder strings.
- **`database/db.py`** — implemented per the Step 1 spec: `get_db()` (SQLite connection with `row_factory` and foreign keys enabled), `init_db()` (creates `users`/`expenses` tables with `CREATE TABLE IF NOT EXISTS`), `seed_db()` (inserts one demo user + 8 sample expenses, idempotent). `init_db()`/`seed_db()` run at import time inside `app.app_context()` in `app.py`. DB file: `expense_tracker.db` in the project root (gitignored).
- **`database/__init__.py`** — empty.
- **`templates/`** — Jinja2 templates. `base.html` defines the shared layout (navbar, footer, `{% block content %}`) and is extended by `landing.html`, `login.html`, `register.html`, `terms.html`, `privacy.html`.
- **`static/css/style.css`** and **`static/js/main.js`** — shared frontend assets referenced via `url_for('static', ...)` in `base.html`.

## Notable gaps to be aware of

- `POST /register` and `POST /login` are implemented: server-side validation, duplicate-email check, `werkzeug.security` password hashing/verification, and on success `session["user_id"]`/`session["user_name"]` are set and the request redirects to `/profile`. On failure both re-render their form with the `error` variable. `app.secret_key` is a hardcoded dev value in `app.py` — move it to an env var before deploying.
- `/logout` and `/profile` are still placeholder strings — session is created on login/register but nothing reads or clears it yet. That's the next step.
- No per-request DB connection lifecycle yet (no `flask.g` caching or `app.teardown_appcontext`) — each route that touches the DB opens and closes its own connection via `get_db()`.
