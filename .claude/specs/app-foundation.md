# Software Specification Document (Implementation Addendum)

**Document Type:** Implementation Addendum
**Scope:** App Foundation (authentication + protected dashboard)

---

## 1. Scope

This document captures **implementation-level behavior only** — the specific runtime, UI, and data-handling details needed to reproduce the application exactly. It deliberately omits the following, which are already documented elsewhere:

- Product goals, user stories, success metrics, and roadmap → `docs/PRD.md`
- Architecture, technology stack, vulnerability descriptions, database schema, and endpoint inventories → `docs/TDD.md`
- Vulnerability exploitation walkthroughs → `docs/EXPLOITS.md` (out of scope for this addendum)

If anything in this addendum conflicts with the PRD or TDD, this document is authoritative for **implementation reproduction**; the PRD/TDD are authoritative for **intent and design rationale**.

---

## 2. Runtime Behavior

The application must exhibit the following runtime characteristics:

- **Automatic database initialization on startup.** On every application start, the system checks whether the SQLite database file exists. If it does not exist, the database file is created automatically and the schema is applied. No manual migration step is required.
- **Missing DB files recreated automatically.** If the SQLite file is deleted while the application is stopped, the next startup recreates it from scratch with the empty schema. No data recovery is attempted.
- **Data preserved across restarts.** User records inserted into the database persist across application restarts; only deliberate file deletion clears them.
- **Static assets available after boot.** Logo images and stylesheets under `frontend/static/` are served by the framework from the moment the server accepts connections. No manual wiring step is required beyond mount configuration.
- **Templates loaded from disk at request time with no caching.** HTML templates are read from disk on each request; modifications to template files are visible on the next page load without restarting the server.
- **Dashboard content modified via runtime string substitution before response.** The dashboard template contains a `{{username}}` placeholder token. The username value from the active session is substituted into this token via string replacement immediately before the response is sent. No template engine variable binding is used for this value.
- **Authentication state based solely on session presence.** Whether a user is considered "logged in" is determined exclusively by the presence of a `user_id` key in the session object. No token verification, expiry check, or database lookup is performed on each authenticated request.

---

## 3. User Flows

### 3.1 Registration Flow

1. User navigates to `/signup`.
2. The signup page renders a form with four inputs: username, email, password, confirm-password.
3. The browser performs client-side validation: all four fields must be non-empty, and the password value must equal the confirm-password value.
4. If client-side validation fails, a red error message is rendered directly beneath the confirm-password field. No network request is sent.
5. If client-side validation passes, the browser submits the form as a standard HTTP POST to `/signup`.
6. The server validates that all four fields are present. If any field is missing, an error is rendered and the user remains on the signup page.
7. The server computes an MD5 hash of the submitted password.
8. The server inserts a new user row via SQL string concatenation (no parameterized query).
9. If the username violates the unique constraint, an error is rendered on the signup page.
10. On successful insert, the server issues an HTTP redirect to `/login`.

### 3.2 Login Flow

1. User navigates to `/login`.
2. The login page renders a form with two inputs: username and password.
3. The browser submits the form via an asynchronous `fetch()` request (AJAX) — not a standard form POST.
4. The server validates that both fields are present.
5. The server computes an MD5 hash of the submitted password.
6. The server looks up the user via SQL string concatenation matching username and hashed password.
7. If no row is found, the server returns an error response; the browser displays the error message inline without a page reload.
8. If a row is found, the server writes `user_id`, `username`, and `email` into the session and returns a success response.
9. The browser, upon receiving the success response, navigates the user to `/welcome`.

### 3.3 Dashboard Flow

1. User navigates to `/welcome`.
2. The server checks the session for a `user_id` value.
3. If `user_id` is absent, the server issues a redirect to `/login`.
4. If `user_id` is present, the server loads `dashboard.html` from disk.
5. The server performs a literal string replacement of the `{{username}}` token in the loaded template with the `username` value from the session.
6. The server returns the modified HTML as the response body.

### 3.4 Logout Flow

1. User initiates logout by visiting `/logout` (typically via a link/button on the dashboard).
2. The server clears all keys from the session.
3. The server issues a redirect to `/login`.
4. Subsequent attempts to access `/welcome` are denied (redirected back to `/login`) because the session no longer contains `user_id`.

---

## 4. Functional Requirements

| ID | Requirement |
|----|-------------|
| **FR-01: Session Management** | The server MUST maintain a signed session containing at least `user_id`, `username`, and `email`. The session MUST be established after successful login and CLEARED entirely on logout. The signing secret is hardcoded in the application entry point. |
| **FR-02: Dynamic User Context** | The dashboard MUST display the logged-in user's username. The display MUST be produced by literal string substitution of the `{{username}}` token in the loaded template — not by passing a context object to a templating engine. |
| **FR-03: Route Protection** | Access to `/welcome` MUST be conditional on the presence of `user_id` in the session. If absent, the user is redirected to `/login`. No other state (cookie expiry, token verification, DB lookup) is consulted. |
| **FR-04: Error Handling** | Validation failures (missing fields, password mismatch, duplicate username, invalid credentials) MUST be presented to the user as inline messages without crashing the application or losing in-progress form data. |
| **FR-05: Search Processing** | The `/search` endpoint MUST accept a `q` query parameter. The response MUST be an HTML response that includes the raw query value rendered into the body. Matching rows from the `users` table (username or email) MUST be listed in the response. |
| **FR-06: Persistence** | User records MUST persist across application restarts. On first run after the database file is absent, the schema MUST be created automatically. |

---

## 5. Complete Visual Design Specification

### 5.1 Global Design System

#### Typography

Font stack:

```
"Segoe UI", system-ui, -apple-system, sans-serif
```

Typography scale:

| Role | Size | Weight |
|------|------|--------|
| Main page titles | `2rem` | `800` |
| Section titles | `1.4rem` | `700` |
| Form titles | `1.7rem` | `700` |
| Card titles | `0.95rem` | `700` |
| Body text | `0.9rem` | `400` |
| Labels | `0.82rem` | `600` |
| Buttons | `1rem` | `600` |

#### Colors

Primary palette:

| Token | Value |
|-------|-------|
| Primary Deep | `#1a237e` |
| Primary Mid | `#3949ab` |
| Primary Dark | `#283593` |
| Ink | `#0f172a` |
| Surface Tint | `#eef1f8` |
| Surface | `#ffffff` |

Text colors:

| Token | Value |
|-------|-------|
| Text Primary | `#1e293b` |
| Text Secondary | `#475569` |
| Text Muted | `#64748b` |
| Text On-Primary (light) | `#c5cae9` |
| Text On-Primary (strong) | `#1a237e` |

#### Border Radius

| Element | Radius |
|---------|--------|
| Inputs | `8px` |
| Buttons | `8px` |
| Cards | `10–12px` |
| Status tags | `6px` |

#### Shadows

| Element | Shadow |
|---------|--------|
| Header | `0 2px 10px rgba(26, 35, 126, 0.08)` |
| Card (hover) | `0 4px 16px rgba(26, 35, 126, 0.10)` |
| Input focus glow | `0 0 0 3px rgba(57, 73, 171, 0.12)` |

### 5.2 Shared Header

- **Position:** fixed at the top of the viewport.
- **Height:** `70px`.
- **Background:** `#ffffff`.
- **Bottom border:** `1px solid #e5e7eb` (or equivalent light divider).
- **Shadow:** header shadow as specified above.
- **Layout:** flex with the application title on the left and a cluster of three organizational logos on the right.
- **Logos:** three images at `54×54px`, square aspect, preserved aspect ratio, rendered with a small gap between them.
- **Logo files present on disk:** `PUCIT_Logo.png` (already present in `frontend/static/images/`). The other two logo files (`blue-logo-scl2.png`, `excaliat-logo.png`) MUST also be present under `frontend/static/images/`.

### 5.3 Login Page

- **Layout:** two-column split-screen at `50% / 50%` on desktop.
- **Left panel:**
  - Background: deep blue gradient running top-to-bottom: `#0d1b5e → #1a237e → #283593`.
  - Contains a small uppercase badge label, a large welcome heading, a description paragraph, and a bullet list of value statements.
  - Decorative overlays: two or more semi-transparent white circles at approximately `7%` opacity, positioned off the natural text flow.
- **Right panel:**
  - Background: `#ffffff`.
  - Form container: `max-width: 400px`, centered vertically.
  - Form contents, in order: form title, subtitle, username field, password field, error-message area (initially hidden), full-width login button, "create account" link.
  - **Login button:** full width, background `#1a237e`, text `#ffffff`, button radius per global spec.
  - **Input styling:** background `#f8f9ff`, border `1.5px solid #c5cae9`, radius per global spec. On focus, border becomes `#3949ab` and the input glows with the focus shadow.
  - **Error message area:** when shown, light red background, red border, dark red text.

### 5.4 Signup Page

- **Layout:** identical to the login page — same gradient left panel, same decorative circles, same right-panel form structure.
- **Form fields, in order:** username, email, password, confirm-password.
- **Password mismatch behavior:** if password does not equal confirm-password after the user finishes editing either field, red text is rendered directly beneath the confirm-password field. The page does NOT reload and no network request is sent.
- **Submit button:** full width, primary background, identical to login button styling.
- **Link to login page** below the form.

### 5.5 Dashboard Page

- **Body background:** `#eef1f8`.
- **Hero banner:** rendered immediately beneath the fixed header.
  - Background: gradient `#1a237e → #3949ab`.
  - **Left section:** title and subtitle in white/light text.
  - **Right section:** the logged-in username (text), and a logout button styled with a semi-transparent white background (white text on translucent surface) so it reads against the gradient.
- **Content area:** `max-width: 1100px`, horizontally centered, with vertical padding below the hero.
- **Mission card:** a single white card with the section title and description text.
- **"Vulnerabilities to Discover" section:**
  - Section header: uppercase, small, bold.
  - Two-column responsive grid of eight vulnerability cards.
  - Each card: white background, rounded corners (per global spec), light border, hover shadow per global spec.
  - Each card contains: a colored pill tag at the top and a description paragraph below.
  - **Pill tag colors per category:**

    | Tag | Color |
    |-----|-------|
    | SQLi | yellow |
    | XSS | red |
    | Session | purple |
    | Brute | orange |
    | Crypto | green |
    | Exposed | blue |
    | CSRF | pink |

  - The full eight-card set MUST cover all categories from the PRD vulnerability list (the eighth category is filled in to match the PRD's eight vulnerabilities).
- **Process steps section:** three step cards arranged in a row (stack vertically on small screens).
  - Each card: background `#1a237e`, white text.
  - Each card contains a circular numbered badge (white circle, primary-color number) and a short step label.
  - Step labels, in order: **Find**, **Exploit**, **Mitigate**.

### 5.6 Responsive Behavior

| Breakpoint | Auth pages | Dashboard | Header |
|------------|------------|-----------|--------|
| Desktop (≥ ~900px) | Two-column 50/50 split | Two-column vulnerability grid, row of process steps | Full-size logos at 54×54px |
| Mobile (< ~900px) | Stacked vertically (gradient panel collapses above form; form panel full width) | Single-column vulnerability grid, process steps stack vertically | Logos shrink; layout reflows but header remains fixed |

---

## 6. Form Specifications

### 6.1 Registration Form

- **Method:** standard HTTP POST (form submission, not fetch).
- **Action:** `/signup`.
- **Fields:** `username`, `email`, `password`, `confirm_password`.
- **Client-side validation BEFORE submit:**
  1. All four fields must be non-empty.
  2. `password` MUST equal `confirm_password`.
  3. On mismatch, render red text below the confirm-password field; do not submit.
- **Server-side behavior on failure:** redisplay the form with an error message; do not redirect.

### 6.2 Login Form

- **Submission method:** asynchronous `fetch()` request (AJAX). NOT a standard form POST.
- **Action target:** `/login`.
- **Fields:** `username`, `password`.
- **On server success:** the browser navigates to `/welcome`.
- **On server failure:** the browser displays the error message inline; the page does NOT reload.

---

## 7. Validation Rules

| Context | Rule |
|---------|------|
| Registration | `username`, `email`, `password`, `confirm_password` MUST all be present (non-null/non-empty after trimming). |
| Registration | `password` MUST equal `confirm_password` (validated client-side before submit). |
| Registration | `username` MUST be unique; uniqueness is enforced at the database level (the column has a `UNIQUE` constraint and the application performs no additional dedup logic). |
| Login | `username` and `password` MUST both be present. |
| Search | The `q` query parameter MUST be present; the endpoint must accept an empty string and render it, but missing parameter behavior is application-defined. |

---

## 8. Session State Model

**Keys written into the session on successful login:**

| Key | Source | Purpose |
|-----|--------|---------|
| `user_id` | `users.id` (the auto-incremented primary key of the matched row) | Authorization gate on `/welcome` |
| `username` | `users.username` (the matched row's username column) | Display on dashboard |
| `email` | `users.email` (the matched row's email column) | Available for downstream features |

**Lifecycle:**

- **Creation:** All three keys are written together on successful login.
- **Usage during route access:** `/welcome` checks `user_id` for protection and reads `username` for the dashboard template substitution.
- **Destruction:** All keys are cleared on `/logout`.

---

## 9. Data Lifecycle Rules

- A user record is created **only** via the signup flow.
- There is **no** in-application workflow for modifying a user record after creation.
- There is **no** in-application workflow for deleting a user record.
- There is **no** recovery workflow; deletion of the database file is the only way to clear all records.

---

## 10. Success Paths

| ID | Path | Steps |
|----|------|-------|
| **SP-01** | Successful registration | User submits signup form with valid unique data → server inserts row → server redirects to `/login`. |
| **SP-02** | Successful login | User submits login form with valid credentials → server writes session keys → browser navigates to `/welcome`. |
| **SP-03** | Dashboard access | Authenticated user visits `/welcome` → server loads template → server substitutes `{{username}}` → server returns HTML. |
| **SP-04** | Successful logout | Authenticated user triggers `/logout` → server clears session → server redirects to `/login` → subsequent `/welcome` visits redirect to login. |

---

## 11. Alternate Paths

| ID | Path | Behavior |
|----|------|----------|
| **AP-01** | Duplicate username at signup | Signup form redisplayed with an error message indicating the username is taken. No row inserted. |
| **AP-02** | Invalid credentials at login | Login form redisplayed with an inline error message; no session written. |
| **AP-03** | Unauthorized dashboard access | Visiting `/welcome` without an active session redirects to `/login`. |
| **AP-04** | Empty / missing search query | `/search` endpoint renders an HTML response without matching rows; the empty query value (if provided) is rendered into the response body. |

---

## 12. Edge Cases

| ID | Scenario | Expected Behavior |
|----|----------|-------------------|
| **EC-01** | Registration with an existing username | Signup fails; error shown to user; no second row inserted (database uniqueness constraint prevents it). |
| **EC-02** | Registration with any empty field | Client-side and/or server-side validation rejects the submission; no row inserted. |
| **EC-03** | Login with any empty field | Server rejects the submission; no session written. |
| **EC-04** | Dashboard access with no session | Redirect to `/login`. |
| **EC-05** | Dashboard access with a corrupted / partial session | Treated as no session for the purpose of the `user_id` gate; redirects to `/login`. |
| **EC-06** | Missing template file | Server fails to load the template; an HTTP error response is produced. |
| **EC-07** | Missing database file at startup | Database file is recreated; schema is applied; the application starts cleanly with zero users. |
| **EC-08** | Application restart with existing database | Existing user records are preserved; no schema migration is performed (schema is created idempotently with `IF NOT EXISTS`). |

---

## 13. Business Rules

The following rules are derived from the implementation:

1. **Authentication depends on session, not on credentials.** Once a session is established, all authorization decisions are based purely on session contents — credentials are never re-checked on protected routes.
2. **Dashboard requires runtime substitution.** The username shown on the dashboard MUST be injected into the template via literal string replacement at response time, not via templating engine variables.
3. **User records are immutable after creation.** There is no UI or API surface for editing or deleting a user record once it exists.
4. **Login and registration use different response formats.** Login uses an asynchronous fetch flow with JSON-style success/failure handling on the client. Registration uses a standard full-page form POST and server-side redirect.
5. **Template updates are visible without restart.** Because templates are read from disk on each request, editing a template file is reflected on the next page load without restarting the server.
6. **DB constraint enforcement is the primary uniqueness mechanism.** Application code does not perform a pre-insert username check; uniqueness is enforced exclusively by the database `UNIQUE` constraint on the username column.

---

## 14. Rebuild Requirements

A compatible implementation MUST reproduce all of the following:

1. Automatic database file creation and schema initialization on every startup.
2. SQLite database with a single `users` table containing `id`, `username`, `email`, `password` columns.
3. Username column has a `UNIQUE` constraint; `id` is the auto-incrementing primary key.
4. Session middleware configured with a fixed secret string.
5. Static files served from `frontend/static/`.
6. Templates read from disk on each request (no in-memory template cache).
7. Dashboard username rendered via literal string substitution of a `{{username}}` token.
8. Signup via standard form POST; login via async fetch; both write into the same session model on success.
9. `/welcome` guarded by `user_id` session presence; redirects to `/login` otherwise.
10. `/logout` clears the session and redirects to `/login`.
11. `/search?q=…` renders an HTML response including the raw query value and any matching rows.
12. `/download/db` serves the SQLite database file with no authentication.
13. MD5 (no salt) used for password hashing.
14. SQL queries constructed via string concatenation (no parameterized queries).
15. Header (70px, fixed, white, three logos on the right) rendered on every page.
16. Login and signup pages use a 50/50 split-screen with the deep-blue gradient left panel and white right panel.
17. Dashboard body background `#eef1f8`, hero banner with `#1a237e → #3949ab` gradient, white cards, and eight vulnerability cards with color-coded tags.
18. Three process step cards (Find, Exploit, Mitigate) with `#1a237e` background and circular numbered badges.
19. Responsive layout: vertical stacking on mobile, two-column grid on desktop.
20. All inputs styled with `#f8f9ff` background and `1.5px solid #c5cae9` border; focus changes border to `#3949ab` with a blue glow.

---

## 15. Acceptance Criteria

| ID | Criterion |
|----|-----------|
| **AC-01** | Registration: submitting a signup form with a fresh username, valid email, and matching passwords creates a new user row and redirects to `/login`. |
| **AC-02** | Login: submitting a login form with valid credentials writes `user_id`, `username`, `email` into the session and the browser navigates to `/welcome`. |
| **AC-03** | Dashboard: visiting `/welcome` while authenticated returns a page that displays the logged-in user's username in place of the `{{username}}` token. |
| **AC-04** | Logout: triggering logout clears the session; a subsequent visit to `/welcome` redirects to `/login`. |
| **AC-05** | Search: visiting `/search?q=<value>` returns an HTML response containing the raw query value and any matching username/email rows. |
| **AC-06** | Persistence: a user created in one run is still present after restarting the application (provided the database file has not been deleted). |

---

## 16. Test Cases

| ID | Scenario | Input / Action | Expected Result |
|----|----------|----------------|-----------------|
| **TC-01** | Signup with valid new data | `username=test1`, valid email, matching passwords | Account created; redirect to `/login` |
| **TC-02** | Signup with mismatched passwords | password ≠ confirm_password | Red text under confirm field; no submit |
| **TC-03** | Signup with existing username | Re-use a username already in DB | Error rendered on signup; no duplicate row |
| **TC-04** | Signup with empty field | Leave any field blank | Submission rejected; no row inserted |
| **TC-05** | Login with valid credentials | Existing username + correct password | Session populated; navigate to `/welcome` |
| **TC-06** | Login with wrong password | Existing username + incorrect password | Inline error; no session |
| **TC-07** | Login with empty fields | Username and/or password blank | Submission rejected; no session |
| **TC-08** | Dashboard authenticated | `/welcome` while logged in | Page renders with substituted username |
| **TC-09** | Dashboard unauthenticated | `/welcome` with no session | Redirect to `/login` |
| **TC-10** | Logout effect | `/logout` then `/welcome` | Redirect to `/login` |
| **TC-11** | Search existing user | `/search?q=<known_username>` | HTML response lists the user row |
| **TC-12** | Search empty query | `/search?q=` | HTML response with no result rows |
| **TC-13** | DB recreation on missing file | Delete DB file, restart app | App starts; `users` table exists and is empty |
| **TC-14** | Data persistence | Create user, restart app, login | User can still log in |
| **TC-15** | Login response format | Inspect login response | Login response is consumed client-side via fetch, not a full page form post |

---

## 17. Documentation Gaps

The following discrepancies were observed between `PRD.md` / `TDD.md` and the implementation reality implied by this addendum. They are documented here so reviewers can resolve them deliberately rather than by accident:

1. **Login HTTP response format.** The TDD's data-flow diagram implies a standard form POST/redirect for login (similar to signup), but the actual implementation uses an asynchronous `fetch()` request with client-side handling of the success/failure response. Treat login as a JSON-style endpoint, not a redirect endpoint.

2. **Dashboard username delivery mechanism.** The TDD describes template rendering with `{{username}}` as a "placeholder substitution," which is accurate, but the implication is ambiguous — the implementation uses literal Python string `replace()`, not a Jinja2-style variable binding. Any rebuild that uses a templating engine must still produce the same final output (raw, unescaped username in the HTML).

3. **Template freshness.** Neither the PRD nor the TDD explicitly states that templates are re-read from disk on every request. The implementation must behave this way to support live template editing during development, even though the docs do not call it out.

4. **Session shape vs. session usage.** The TDD states that `user_id`, `username`, and `email` are written into the session, but does not clarify that the only key actually consulted on protected routes is `user_id`. The other two keys (`username`, `email`) are written for downstream use but the gate is `user_id` alone.