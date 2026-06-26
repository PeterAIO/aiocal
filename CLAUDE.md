# aiocal — AIO Deployment & Onboarding Booking System

## Overview
A scheduling/booking web app for AIO (a restaurant POS company) used by deployment, onboarding, and demo teams to book installs and onboarding calls, assign staff via round-robin, and keep HubSpot in sync. It tracks installer/specialist availability and time off, books jobs against HubSpot deals/companies, sends confirmation emails with calendar invites, collects customer onboarding forms, and records deployment sign-offs. Deployed serverless on Vercel with a Postgres backend; the frontend is a set of static HTML pages with vanilla JS.

## Tech Stack
- **Language:** Python 3.9 (Vercel `@vercel/python` runtime; local `.venv` is Python 3.9.6).
- **Framework:** Flask 3.1.3.
- **Database:** PostgreSQL via `psycopg2-binary` 2.9.10 (production, `DATABASE_URL`). The legacy `app.py` uses SQLite instead.
- **HTTP:** `requests` 2.32.5 (HubSpot + SendGrid calls).
- **Frontend:** Plain HTML/CSS/vanilla JS (no build step, no npm, no external CDN libs). Calendar UI is hand-rolled in `public/index.html`.
- **Integrations:** HubSpot CRM v3 API, SendGrid email API.
- **Hosting:** Vercel (`vercel.json`).

## Architecture
Two backend implementations exist — **`api/index.py` is the live production app**; `app.py` at the repo root is an older, simpler standalone version.

- **`api/index.py`** (~1800 lines, the deployed serverless function): full app with Postgres, cookie/token session auth, role-based access, round-robin assignment, the multi-stage onboarding workflow, HubSpot note/property syncing, and SendGrid email. `VERSION` constant near the top (currently `2.14.0`).
- **`app.py`** (root): legacy single-file Flask app backed by SQLite (`bookings.db`), uses SMTP (not SendGrid), no auth, no onboarding/sign-off workflow. Useful as a reference but not what runs in production. Routes static files itself and runs on port 5001.
- **`public/`**: static frontend pages, each a self-contained HTML file with inline JS that calls the `/api/*` endpoints.

**Data model** (created/migrated in `init_db()` in `api/index.py`): `users` (with `role`, `color`, `password_hash`, `hubspot_owner_id`), `availability` (per user, per day-of-week, with start/end times), `time_off`, `bookings` (the central table, with HubSpot deal/company IDs, contact info, status, `created_by`, and onboarding form/sign-off columns), and `sessions` (auth tokens). New columns are added idempotently via `ALTER TABLE ... EXCEPTION WHEN duplicate_column` blocks.

**Roles:** `manager` (admin), `ob_admin` (owns onboarding/demo bookings), `deployment_specialist`, `installer`, `trainer`, `lead`. `require_auth` / `require_admin` decorators gate endpoints; `check_booking_owner()` restricts edit/delete to creators (managers and ob_admin have wider rights).

**Booking types & assignment:** `install`/deployment (blocks the whole day for that specialist), `onboarding`, and `demo_setup` (1-hour slots, funneled to `ob_admin`). `_roles_for_booking_type()` maps type → eligible roles. `/api/round-robin` picks the eligible available user with the fewest bookings (ties broken randomly), excluding those on time off or already booked.

**HubSpot sync:** On booking create/complete/sign-off, the app posts HTML notes to the associated deal (assoc type 214) and company (assoc type 190), @mentions the specialist/booker/admin, and PATCHes date properties: deployments → `install_date_new` (deal) + `migrated_00npw00000fv4pz2ad` (company); onboarding → `on_boarding_call`; demo → `demo_request_date` (company). Completion/sign-off sets company `migrated_00nfi000003lzy1uac = "Active"` (Deployed). Deal stage IDs of interest are in the `DEAL_STAGES` dict. Times are interpreted as `America/Los_Angeles` and converted to UTC ms via `local_dt_to_ms()`.

**Onboarding workflow (3 stages):** (1) booking created → `confirmed`; (2) customer opens a tokenized public link (`/onboarding/<token>`), submits the pre-call form → `prep_complete`; (3) AIO Buddy submits the sign-off form after the call → `completed`. Public (no-auth) endpoints `GET/POST /api/public/onboarding/<token>` serve the customer form.

## Key Files & Entry Points
- `api/index.py` — **production Flask app / Vercel serverless entry point.** All live `/api/*` routes, auth, DB schema, HubSpot + SendGrid logic.
- `app.py` — legacy standalone SQLite version (local dev / reference only). `python app.py` serves on port 5001.
- `vercel.json` — Vercel build & routing config (maps `/api/*` → `api/index.py`, pretty URLs → `public/*.html`, static assets).
- `requirements.txt` — Python dependencies.
- `public/index.html` — main calendar/booking UI (booking creation, round-robin, HubSpot deal lookup).
- `public/bookings.html` — list view of all bookings (largest frontend page).
- `public/admin.html` — admin: manage users, availability, time off, passwords.
- `public/login.html` — login page (served at `/login`).
- `public/onboarding.html` — public customer onboarding form (served at `/onboarding/<token>`).
- `public/whats-new.html` — changelog page (tracks released versions; latest ~2.14.x).
- `public/aio-logo.png`, `aio-logo-white.png` — brand assets.

## Build / Run / Test
There is **no build step, no package.json, no Makefile, and no test suite** in this repo. Frontend is static; backend is a single Python file.

**Run the legacy local version (SQLite, no auth):**
```bash
source .venv/bin/activate          # existing virtualenv (Python 3.9)
python app.py                      # serves http://localhost:5001, creates ./bookings.db
```

**Run the production app locally (Postgres):** there is no documented runner; `api/index.py` defines `app` but no `__main__` block. To run it locally you would point Flask at it with a `DATABASE_URL` set, e.g.:
```bash
source .venv/bin/activate
pip install -r requirements.txt    # if deps aren't already installed
export DATABASE_URL="postgres://...?sslmode=require"
flask --app api/index run          # not verified in-repo; api/index.py is built for Vercel
```

**Deploy:** push to the Vercel project; `vercel.json` drives the build (`@vercel/python` for `api/index.py`, `@vercel/static` for `public/**`).

**Required environment variables** (read in `api/index.py`):
- `DATABASE_URL` — Postgres connection string (connected with `sslmode="require"`).
- `HUBSPOT_ACCESS_TOKEN` — HubSpot private app token (HubSpot features no-op if unset).
- `SENDGRID_API_KEY`, `EMAIL_FROM`, `ADMIN_EMAIL` — email sending (no-op if unset).
- `VERCEL` — set automatically on Vercel; in `app.py` it switches the SQLite path to `/tmp`.
- (`app.py` only) `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM`.

## Conventions & Gotchas
- **`api/index.py` is the source of truth, not `app.py`.** They share names and schema but diverge heavily (DB engine, auth, email provider, workflow). Don't edit `app.py` expecting production to change.
- **First-time admin setup:** `POST /api/auth/setup` creates the first `manager` (or sets a password on an existing passwordless manager). Once a manager with a password exists, it refuses further self-setup. Passwords are stored as **plain SHA-256** (`hash_password`), no salt — adequate-for-internal-tool, not best practice; don't treat it as secure.
- **Sessions:** token in an `httponly` cookie `session_token` (30-day) or `X-Session-Token` header; rows live in the `sessions` table.
- **Schema migrations are inline** in `init_db()` using idempotent `ALTER TABLE` blocks — add new columns there, not in a separate migration system. `init_db()` runs once per process via the `_db_initialized` global.
- **Timezone:** all booking datetimes are treated as `America/Los_Angeles` (Pacific) when converting to HubSpot UTC-ms timestamps. Falls back gracefully if `zoneinfo` is unavailable.
- **HubSpot/SendGrid failures are intentionally swallowed** in most write paths (`except Exception: pass` or recorded into the JSON `result`) so a booking still succeeds if an integration is down. Check the returned `result` object for `hubspot_note` / `email_sent` status strings.
- **HubSpot association type IDs** (214 = note→deal, 190 = note→company) and the custom property internal names are hard-coded; they're specific to AIO's HubSpot portal.
- **`PRODID` in generated ICS still says `POSUP`** (a prior brand name) in both files — cosmetic.
- The repo's `.venv/` is committed/present locally; `.gitignore` excludes `*.db`, `.env`, `.venv/`. `README.md` is a one-line stub.
