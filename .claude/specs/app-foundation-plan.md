# App Foundation — Implementation Plan

This document is the step-by-step build plan for the **App Foundation** feature: a FastAPI backend, a static HTML/CSS frontend, and a SQLite database that together implement an intentionally vulnerable authentication flow. The plan is derived from `docs/PRD.md`, `docs/TDD.md`, and `.claude/specs/app-foundation.md`.

**Read before implementing:** this plan produces code with 8 *intentional* vulnerabilities for security education. The vulnerable code paths (string-concatenated SQL, unescaped HTML, hardcoded session secret, MD5, unauthenticated DB download, no rate limiting, no CSRF tokens) are the **deliverable**. They must NOT be "fixed" during implementation.

---

## Phase 1 — Project Structure

### 1.1 Backend layout

Create the following file tree under `backend/`:

```
backend/
├── pyproject.toml
└── app/
    ├── __init__.py              (empty)
    ├── main.py
    ├── core/
    │   ├── __init__.py          (empty)
    │   └── security.py
    ├── db/
    │   ├── __init__.py          (empty)
    │   └── session.py
    ├── services/
    │   ├── __init__.py          (empty)
    │   └── auth_service.py
    └── api/
        ├── __init__.py          (empty)
        └── routes/
            ├── __init__.py      (empty)
            └── auth.py
```

Every `__init__.py` is empty (no package metadata). They exist only so Python treats the directories as packages.

### 1.2 Backend `pyproject.toml`

A **separate** `pyproject.toml` lives at `backend/pyproject.toml`. (The repo-root `pyproject.toml` from the initial scaffold stays for the workspace; the backend one describes the backend package itself.)

Build system: **hatchling**.

Runtime dependencies:
- `fastapi>=0.109.0`
- `uvicorn>=0.27.0`
- `python-multipart>=0.0.6`
- `itsdangerous>=2.0.0`

Optional dev dependency:
- `pytest` (declared under `[project.optional-dependencies]`; do not gate the runtime on it)

### 1.3 Frontend layout

The frontend lives at the repo root under `frontend/` (already created):

```
frontend/
├── templates/
│   ├── login.html
│   ├── signup.html
│   └── dashboard.html
└── static/
    ├── css/
    │   └── styles.css
    └── images/
        ├── PUCIT_Logo.png        (already present)
        ├── blue-logo-scl2.png    (placeholder / source asset)
        └── excaliat-logo.png     (placeholder / source asset)
```

Templates are read from disk on every request (no engine cache) — this is intentional and is part of the runtime contract in `app-foundation.md §2`.

---

## Phase 2 — Database Layer (`backend/app/db/session.py`)

**Files to create:**
- `backend/app/db/session.py`

**Implementation:**

- Database file path: `vulnerable_app.db` at the **project root** (one level above `backend/`). Resolve the path from `backend/app/db/session.py` by walking up two parents from `Path(__file__).parent` (i.e., `Path(__file__).resolve().parents[2] / "vulnerable_app.db"`).
- `get_db()`: open `sqlite3.connect(DB_PATH)`, set `conn.row_factory = sqlite3.Row`, and return the connection. Use `check_same_thread=False` to allow the connection to be shared across threads (FastAPI may call endpoints from different worker threads).
- `init_db()`: execute `CREATE TABLE IF NOT EXISTS users (...)` with the exact schema below. Wrap the `CREATE TABLE` in a transaction (`BEGIN` / `COMMIT`) so a missing-file startup reliably produces an empty, schema-ready DB.

**Schema (exact):**

```sql
CREATE TABLE IF NOT EXISTS users (
    id       INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE,
    email    TEXT,
    password TEXT
)
```

Notes:
- The `UNIQUE` constraint on `username` is the **only** mechanism that enforces uniqueness — do not add application-side dedup (this is `BR-6` in the spec).
- `init_db()` MUST be safe to call on every startup. `IF NOT EXISTS` makes it idempotent.

---

## Phase 3 — Security Utilities (`backend/app/core/security.py`)

**Files to create:**
- `backend/app/core/security.py`

**Functions:**

- `hash_password(password: str) -> str`
  - Returns `hashlib.md5(password.encode()).hexdigest()`.
  - **No salt.** **No pepper.** **No KDF.** This is intentional (Vulnerability #5).
- `verify_password(plain: str, hashed: str) -> str`
  - Returns `hash_password(plain) == hashed`.
  - Defined for completeness; `auth_service.login()` will use a direct SQL compare, but exposing the verify helper keeps the security module self-contained.

Both functions are intentionally trivial. Do not "improve" them.

---

## Phase 4 — Business Logic (`backend/app/services/auth_service.py`)

**Files to create:**
- `backend/app/services/auth_service.py`

**Critical constraint (read first):** Both `signup` and `login` build SQL via **string concatenation**. Do NOT switch to parameterized queries. The concatenated SQL is Vulnerability #1.

### 4.1 `signup(username, email, password) -> Response`

- `username`, `email`, `password` arrive as plain `str` from FastAPI `Form()` params (the route handler is responsible for accepting them as form fields).
- **Validation:** all three values must be present and non-empty after a strip. If any is missing, return an `HTMLResponse` containing a basic error page (or a re-rendered signup template with the error — keep it simple; a small inline error HTML string is acceptable).
- Hashing: `hashed = hash_password(password)`.
- **SQL — string concatenation:**
  ```python
  query = (
      "INSERT INTO users (username, email, password) "
      "VALUES ('" + username + "', '" + email + "', '" + hashed + "')"
  )
  ```
  Execute it via `cursor.execute(query)` on a connection obtained from `get_db()`. Commit.
- On `sqlite3.IntegrityError` (the `UNIQUE` constraint triggered), return an error indicating the username is taken.
- On success: return `RedirectResponse(url="/login", status_code=302)`.

### 4.2 `login(request, username, password) -> Response`

- `request` is the FastAPI/Starlette `Request` (needed for `request.session`).
- `username`, `password` arrive as `str` from `Form()`.
- **Validation:** both fields must be present and non-empty.
- Hashing: `hashed = hash_password(password)`.
- **SQL — string concatenation:**
  ```python
  query = (
      "SELECT * FROM users WHERE username = '" + username
      + "' AND password = '" + hashed + "'"
  )
  ```
- Execute via `cursor.execute(query)` on a connection from `get_db()`. Use `fetchone()`.
- **On match:** write into the session:
  ```python
  request.session["user_id"]  = row["id"]
  request.session["username"] = row["username"]
  request.session["email"]    = row["email"]
  ```
  Return `JSONResponse({"success": True, "redirect": "/welcome"})` — the login page's JS will read this and call `window.location.href = data.redirect`.
- **On failure (no row, missing fields, exception):** return `JSONResponse({"success": False, "error": "Invalid username or password"}, status_code=401)`. The login page JS reads `data.error` and displays it inline.

Returning JSON from login (and HTMLResponse/RedirectResponse from signup) is what makes the two flows use "different response formats" (`BR-4`). Do not unify them.

---

## Phase 5 — Route Handlers (`backend/app/api/routes/auth.py`)

**Files to create:**
- `backend/app/api/routes/auth.py`

All routes are registered on a single `APIRouter`. Import `APIRouter` from `fastapi`. Import `Request`, `Form`, `FileResponse`, `HTMLResponse`, `RedirectResponse` from `fastapi`/`starlette` as needed. Import `auth_service` from the sibling services package.

The route table (exact):

| Method | Path | Handler | Purpose | Auth |
|--------|------|---------|---------|------|
| GET | `/` | `index()` | `RedirectResponse("/signup", 302)` | — |
| GET | `/signup` | `signup_page()` | Read `frontend/templates/signup.html`, return `HTMLResponse` | — |
| POST | `/signup` | `signup_post()` | `await auth_service.signup(username, email, password)` | — |
| GET | `/login` | `login_page()` | Read `frontend/templates/login.html`, return `HTMLResponse` | — |
| POST | `/login` | `login_post()` | `await auth_service.login(request, username, password)` | — |
| GET | `/download/db` | `download_db()` | `FileResponse("vulnerable_app.db", filename="vulnerable_app.db", media_type="application/octet-stream")` — **no auth check** (Vulnerability #6) | — |
| GET | `/search` | `search_user(q)` | String-concatenated SQL with `LIKE '%' + q + '%'`, raw `q` rendered into HTML | — |
| GET | `/welcome` | `welcome_page(request)` | Gate on `user_id`; `html.replace('{{username}}', username)` | session |
| GET | `/logout` | `logout(request)` | `request.session.clear()`; `RedirectResponse("/login", 302)` | — |

### 5.1 Per-route notes

- **`signup_page` / `login_page`:** read template text from `frontend/templates/<name>.html` on every request. Use a path resolved from `Path(__file__).parents[3]` (from `auth.py`, walk up to repo root) plus `"frontend/templates/<name>.html"`. Open in UTF-8.
- **`signup_post`:** parameters `username: str = Form(...)`, `email: str = Form(...)`, `password: str = Form(...)`. Pass through to `auth_service.signup`.
- **`login_post`:** parameters `request: Request`, `username: str = Form(...)`, `password: str = Form(...)`. Pass through to `auth_service.login`.
- **`download_db`:** resolve the same DB path used in `session.py` (or import a shared constant if you prefer), return `FileResponse`. **Do NOT check the session.** This is the entire point.
- **`search_user(q)`:** parameter `q: str` (FastAPI extracts it from the query string).
  - **Vulnerability #3 (Reflected XSS):** `q` is interpolated into the HTML response verbatim.
  - **Vulnerability #1 (also applies here):** the SELECT query is built via string concatenation:
    ```python
    query = "SELECT username, email FROM users WHERE username LIKE '%" + q + "%' OR email LIKE '%" + q + "%'"
    ```
  - Render rows with an f-string: `f"<li>{row['username']} ({row['email']})</li>"` — no escaping.
  - The response includes the raw `q` (e.g. `f"<p>Search results for: {q}</p>"`).
  - On exception, return a plain-text body containing `str(e)` (information leakage is acceptable here).
- **`welcome_page(request)`:** if `"user_id" not in request.session`, return `RedirectResponse("/login", 302)`. Otherwise, read `dashboard.html`, perform literal `html.replace("{{username}}", request.session["username"])`, return `HTMLResponse(result)`. **Do not escape** the username — that is Vulnerability #2 (Stored XSS).
- **`logout(request)`:** `request.session.clear()`; return `RedirectResponse("/login", 302)`.

---

## Phase 6 — Application Entry Point (`backend/app/main.py`)

**Files to create:**
- `backend/app/main.py`

### 6.1 `sys.path` bootstrap (do this first thing in the file)

Add the `backend/` directory to `sys.path` **before any `from app...` imports**, so the script runs correctly regardless of launch context (e.g. `uv run backend/app/main.py` from repo root, or `python app/main.py` from inside `backend/`).

```python
import sys
from pathlib import Path

_BACKEND_DIR = Path(__file__).resolve().parent          # .../backend/app
_PROJECT_ROOT = _BACKEND_DIR.parent                    # .../backend  (for sibling imports)
_BACKEND_PKG = _BACKEND_DIR.parent                     # .../backend  (this is what gets added to sys.path)
sys.path.insert(0, str(_BACKEND_PKG))
```

Concretely: prepend the `backend/` directory (i.e., the parent of `app/`) to `sys.path`. After this, `from app.api.routes.auth import router as auth_router` resolves regardless of cwd.

### 6.2 Application setup

```python
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from starlette.middleware.sessions import SessionMiddleware
from pathlib import Path
import os
import uvicorn

from app.api.routes.auth import router as auth_router
from app.db.session import init_db

app = FastAPI()

# Vulnerability #4: hardcoded weak secret.
app.add_middleware(
    SessionMiddleware,
    secret_key="super-secret-key-12345",
    session_cookie="session",
)

# Static mounts — repo-root-relative.
_PROJECT_ROOT = Path(__file__).resolve().parents[2]   # repo root
app.mount("/static/css",    StaticFiles(directory=str(_PROJECT_ROOT / "frontend" / "static" / "css")),    name="css")
app.mount("/static/images", StaticFiles(directory=str(_PROJECT_ROOT / "frontend" / "static" / "images")), name="images")

app.include_router(auth_router)

# Auto-init DB at module load (runs once when uvicorn imports this module).
init_db()

if __name__ == "__main__":
    port = int(os.environ.get("PORT", "3001"))
    uvicorn.run("app.main:app", host="0.0.0.0", port=port)
```

Notes:
- `init_db()` at module load (not inside an event handler) is the simplest way to satisfy "automatic database initialization on startup" (`app-foundation.md §2`). Because the SQL uses `IF NOT EXISTS`, calling it on every import is safe.
- Path resolution uses `Path(__file__).resolve().parents[2]` from `backend/app/main.py` to reach the repo root — verify this on Windows (use forward slashes or `os.fspath` if needed; `pathlib.Path` strings are fine for `StaticFiles`).
- `0.0.0.0:3001` matches the PRD's stated port; the `PORT` env var override is per `TDD.md §6.3`.

---

## Phase 7 — Frontend Templates

**Files to create:**
- `frontend/templates/login.html`
- `frontend/templates/signup.html`
- `frontend/templates/dashboard.html`

All three templates share the fixed header (`70px`, white background, app title on the left, three logos on the right) and link `../static/css/styles.css` (the actual URL is `/static/css/styles.css` from the browser).

### 7.1 `login.html`

- **Layout:** two-column `50% / 50%` split-screen (`display: grid; grid-template-columns: 1fr 1fr;`).
- **Left panel:** gradient background `#0d1b5e → #1a237e → #283593`. Contains a small uppercase badge, a welcome heading, a description paragraph, and a bullet list. Two semi-transparent white circles (~7% opacity) overlaid as decoration.
- **Right panel:** white. Form container `max-width: 400px`, vertically centered. Fields: username, password. Hidden error-message `<div>` with id like `login-error`. Full-width submit button (`#1a237e` background, white text). "Don't have an account? Sign up" link.
- **JS behavior:**
  - On submit, prevent default.
  - `const formData = new FormData(form);`
  - `const resp = await fetch("/login", { method: "POST", body: formData });`
  - `const data = await resp.json();`
  - If `data.success` → `window.location.href = data.redirect;`
  - Else → show `#login-error` with `data.error` text; do not reload.
- **No CSRF token.** (Vulnerability #8.)

### 7.2 `signup.html`

- **Layout:** identical to login (same gradient, same circles).
- **Form:** standard HTML `<form action="/signup" method="POST">` — NOT fetch. Four fields: `username`, `email`, `password`, `confirm_password`.
- **JS validation (client-side):** on form submit, check `password === confirm_password`. If not equal, show an inline red `<span id="confirm-error">` directly beneath the confirm field and `return false` to prevent submission.
- **No CSRF token.** (Vulnerability #8.)

### 7.3 `dashboard.html`

- **Hero banner** (full width, beneath header) with `#1a237e → #3949ab` gradient.
  - Left: title "Security Vulnerability Lab" + subtitle.
  - Right: literal text `Logged in as {{username}}` plus a logout button styled with semi-transparent white background.
  - Logout button is an `<a href="/logout">` styled to look like a button.
- **Content area:** `max-width: 1100px`, centered, `padding-top` to clear the fixed header.
- **Mission card:** a single white card with a section title and description paragraph.
- **"Vulnerabilities to Discover" section:** uppercase, small, bold heading.
  - Two-column grid (`grid-template-columns: 1fr 1fr;` at desktop).
  - Eight cards, each with a colored pill tag and a description.
  - Tag colors per category (from `app-foundation.md §5.5`): SQLi=yellow, XSS=red, Session=purple, Brute=orange, Crypto=green, Exposed=blue, CSRF=pink. The eighth card uses the CSRF tag.
- **Process steps section:** three step cards in a row at desktop (single column on mobile).
  - Each: `#1a237e` background, white text, circular numbered badge (white circle, primary-color numeral), step label.
  - Labels in order: **Find**, **Exploit**, **Mitigate**.
- The literal token `{{username}}` MUST appear in the HTML and MUST be replaced at request time by the route handler via `str.replace`. Do not bind it via Jinja or any templating engine.

---

## Phase 8 — Styling (`frontend/static/css/styles.css`)

**Files to create:**
- `frontend/static/css/styles.css`

Apply the full visual spec from `app-foundation.md §5`:

- **Typography:** `"Segoe UI", system-ui, -apple-system, sans-serif`. Apply the scale (main titles `2rem/800`, section titles `1.4rem/700`, form titles `1.7rem/700`, card titles `0.95rem/700`, body `0.9rem/400`, labels `0.82rem/600`, buttons `1rem/600`).
- **Primary palette:** `#1a237e`, `#3949ab`, `#283593`, `#0f172a`, `#eef1f8`, `#ffffff`.
- **Text palette:** `#1e293b`, `#475569`, `#64748b`, `#c5cae9`, `#1a237e`.
- **Radii:** inputs/buttons `8px`, cards `10–12px`, tags `6px`.
- **Shadows:** header `0 2px 10px rgba(26, 35, 126, 0.08)`, card hover `0 4px 16px rgba(26, 35, 126, 0.10)`, input focus glow `0 0 0 3px rgba(57, 73, 171, 0.12)`.
- **Header:** `position: fixed; top: 0; left: 0; right: 0; height: 70px; background: #ffffff; box-shadow: ...;` with a flex row: title on the left, three logos (`54×54px`) on the right.
- **Login/signup layout:** `display: grid; grid-template-columns: 1fr 1fr;` for desktop. Inputs: `background: #f8f9ff; border: 1.5px solid #c5cae9;`. Input focus: border `#3949ab` + focus glow. Buttons: primary `#1a237e`, white text.
- **Dashboard:** body background `#eef1f8`; hero with `linear-gradient(135deg, #1a237e, #3949ab)`; vulnerability card grid (2 columns desktop, 1 column mobile); process step cards with `#1a237e` background.
- **Responsive:** at `@media (max-width: 900px)`, collapse the split-screen to one column, vulnerability grid to one column, process steps to a vertical stack, and shrink the header logos.
- **Error styling:** light red background, red border, dark red text for `.error-message` and the inline confirm-password mismatch span.

Link from templates via `<link rel="stylesheet" href="/static/css/styles.css">`.

---

## Phase 9 — `CLAUDE.md`

**Files to create:**
- `CLAUDE.md` (at the repo root, not under `backend/`)

Sections to include (keep it concise; this is for an LLM agent, not for end users):

1. **Project context** — what the app is (intentionally vulnerable security lab), where the spec lives (`.claude/specs/app-foundation.md`), where the design lives (`docs/PRD.md`, `docs/TDD.md`).
2. **Development commands** — `uv sync` inside `backend/`, `uv run backend/app/main.py` to start the server, default port 3001.
3. **Architecture overview** — three-layer: presentation (HTML/CSS/JS) → application (FastAPI routes + services) → data (SQLite). Mention that string concatenation is intentional.
4. **Vulnerability map** — short table: 1=SQLi (`auth_service.py`), 2=Stored XSS (`auth.py:welcome_page`), 3=Reflected XSS (`auth.py:search_user`), 4=Session secret (`main.py`), 5=MD5 (`core/security.py`), 6=DB download (`auth.py:download_db`), 7=No rate limiting (global), 8=CSRF (no tokens anywhere).
5. **Frontend ↔ backend integration** — login is async fetch returning JSON; signup is standard form POST + redirect; dashboard is HTML with server-side `str.replace`; static files served via `/static/*`.
6. **Security education context** — explicitly state that the vulnerabilities are intentional and must be reproduced as specified; "fixing" them defeats the educational purpose.
7. **Specification hierarchy** — if `app-foundation.md` and the PRD/TDD conflict, the implementation spec is authoritative for runtime reproduction; PRD/TDD are authoritative for intent.

---

## Phase 10 — Testing and Validation

**No automated test suite is required for this scaffold.** Manual verification steps:

1. **Start the server:**
   ```
   cd backend
   uv sync
   uv run python app/main.py
   ```
   Confirm log output shows uvicorn listening on `0.0.0.0:3001`. The first run should also create `vulnerable_app.db` at the repo root.
2. **Static assets:** visit `http://localhost:3001/static/css/styles.css` and `/static/images/PUCIT_Logo.png` — both should load with `200`.
3. **Signup flow:** visit `/signup`, submit valid data → should redirect to `/login`.
4. **Login flow:** visit `/login`, submit valid credentials → should redirect to `/welcome` (via JS `window.location.href`), and `/welcome` should display the username.
5. **Welcome protection:** open a private/incognito window, visit `/welcome` directly → should redirect to `/login`.
6. **Logout:** from `/welcome`, click logout → should land on `/login`. Then `/welcome` redirects back to `/login`.
7. **Search reflection:** visit `/search?q=hello` → the response HTML should contain `hello` literally. Visit `/search?q=<img src=x onerror=alert(1)>` and confirm in the rendered DOM that the `<img>` tag is present unescaped (Reflected XSS sanity check).
8. **DB download:** in an unauthenticated browser, visit `/download/db` → the SQLite file should download.
9. **Persistence:** create a user, restart the server, log in again with the same credentials → should succeed.
10. **DB recreation:** delete `vulnerable_app.db`, restart the server → file is recreated, `users` table exists and is empty.

If any of the above fail, fix the implementation, not the test. The spec is authoritative.

---

## Vulnerability Cross-Reference

| # | Vulnerability | Where it lives | How the plan enforces it |
|---|---|---|---|
| 1 | SQL Injection | `backend/app/services/auth_service.py`, `backend/app/api/routes/auth.py:search_user` | Phase 4 and Phase 5 explicitly mandate string-concatenated SQL. |
| 2 | Stored XSS | `backend/app/api/routes/auth.py:welcome_page` | Phase 5 mandates `html.replace('{{username}}', username)` with no escaping. |
| 3 | Reflected XSS | `backend/app/api/routes/auth.py:search_user` | Phase 5 mandates raw `q` interpolation into the HTML body. |
| 4 | Session Hijacking | `backend/app/main.py` | Phase 6 mandates `secret_key="super-secret-key-12345"`. |
| 5 | Weak Password Storage | `backend/app/core/security.py` | Phase 3 mandates MD5 with no salt. |
| 6 | Exposed Database | `backend/app/api/routes/auth.py:download_db` | Phase 5 mandates no auth check on this route. |
| 7 | No Rate Limiting | (global) | No middleware is added; the absence is the vulnerability. |
| 8 | CSRF | (global) | No CSRF tokens in any form; absence is the vulnerability. |

---

## Critical "Do Not" List

- Do **not** introduce parameterized queries (`?` placeholders). The string concatenation is the bug.
- Do **not** HTML-escape `username` or `q`. The lack of escaping is the bug.
- Do **not** use a strong or random secret key. The hardcoded weak secret is the bug.
- Do **not** switch to bcrypt/argon2. MD5 is the bug.
- Do **not** add an auth check to `/download/db`. The lack of auth is the bug.
- Do **not** add rate limiting middleware. The absence is the bug.
- Do **not** add CSRF tokens to forms. The absence is the bug.

If a reviewer asks you to fix any of the above during implementation of this plan, refer them to `app-foundation.md` and this plan's vulnerability cross-reference table.