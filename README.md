# RestingPlace

**A Place of Remembrance. A Legacy of Care.**

RestingPlace is a memorial lot booking and management system for memorial parks: a public
website, a client portal, a caretaker workspace and an administration console, built on
Flask and Supabase (PostgreSQL, Auth, Storage).

| Area | What it does |
|---|---|
| Public website | Brand introduction, landing page, lot catalog with filters and an interactive map, FAQ, contact form, privacy and terms pages, public QR memorial pages |
| Client portal | Reservations, bookings, payment schedule and proof upload, documents, family sharing, burial records, memorial profiles, notifications, support |
| Caretaker workspace | Approve/reject reservations, verify documents and payments, confirm/cancel bookings, burial calendar, maintenance, waiting list, inquiries, reports |
| Administration | Users, caretaker accounts and permissions, lot types and pricing, sections, payment methods, settings, monitoring, reports, audit logs, documented corrections |

## How it is secured

Business rules live **in the database**, so they hold even if someone calls the Supabase API directly:

- **Double booking is impossible**: a partial unique index allows only one pending/approved reservation per lot, and one live booking per lot.
- **Workflows are SQL functions** (`request_reservation`, `review_reservation`, `review_payment`, `confirm_booking`, `schedule_burial`, …). Each checks the caller's role from `profiles`, locks rows, writes the audit log and sends notifications in one transaction.
- **Admins cannot approve operations**: approval functions require an *active caretaker with the specific permission*. Admins monitor and make documented corrections only.
- **Protective triggers** stop raw `UPDATE`s of lot status, payment status, roles, verification fields and QR settings outside those workflows.
- **RLS on every table**; anonymous visitors have no table access at all and read lots through the `public_lot_catalog` view (no owner data; prices only when enabled).
- **Financial and history records are append-only**: payments, receipts, installments, reservations, bookings, burials, status history and audit logs cannot be deleted; payment amounts cannot be edited.
- **Uploads** are validated by magic bytes (PDF, JPG, PNG, WEBP; 5 MB), stored in private buckets under the owner's folder, and served through 2-minute signed URLs created with the requester's own token.
- **Flask** sends every request with the signed-in user's JWT (RLS applies), uses the service-role key only server-side for staff provisioning and scheduled jobs, enforces CSRF, rate limits, a strict CSP (`script-src 'self'`, all assets self-hosted), and `Cache-Control: no-store` on private pages.

## Project layout

```
app/
  __init__.py          app factory, error handlers, CLI (create-admin, run-jobs)
  config.py            environment configuration
  supa.py              thread-safe Supabase REST/Auth/Storage client
  security.py          sessions, role guards, upload validation, security headers
  utils.py             data helpers, formatting, status labels
  nav.py               sidebar menus per role
  blueprints/          public, auth, client, caretaker, admin, staff (shared), documents, api
  services/            pdf (ReportLab), excel (OpenPyXL), qr, maplayout, mailer, jobs (APScheduler)
  templates/           Jinja templates (public, auth, client, caretaker, admin, staff, shared)
  static/              css, js, icons, self-hosted fonts, Leaflet, Chart.js
supabase/
  migrations/0001-0007 schema, triggers, workflows, reporting, RLS, storage, settings
  seed/                optional DEMO data (flagged is_demo) and its cleanup script
  tests/               Supabase stub + SQL workflow tests for a local PostgreSQL
tests/                 unit tests + end-to-end tests (Flask -> PostgREST -> PostgreSQL)
docs/                  DATABASE, DEPLOYMENT, TESTING, QUALITY_ISO25010
```

## Quick start

1. **Create a Supabase project.** In the SQL editor run, in order:
   `supabase/migrations/0001_schema.sql` … `0007_settings.sql`. Every file is safe to re-run.
2. *(Optional, for trying the system)* run `supabase/seed/demo_data.sql`. Demo rows are labelled **DEMO** everywhere and PDFs carry a "DEMO DATA" watermark. Remove them later with `demo_cleanup.sql`.
3. **Auth settings** (Supabase → Authentication):
   - URL configuration → Site URL = your `PUBLIC_BASE_URL`; add `PUBLIC_BASE_URL/auth/callback` to Redirect URLs.
   - Keep "Confirm email" enabled. Configure SMTP for production email delivery.
4. **Install and configure**
   ```bash
   python -m venv .venv && source .venv/bin/activate
   pip install -r requirements.txt
   cp .env.example .env    # fill in the values
   ```
5. **Create the first administrator** (server-side only; there is no public way to become admin):
   ```bash
   flask --app wsgi create-admin
   ```
6. **Run**: `python run.py` (development) or `gunicorn -w 3 -b 0.0.0.0:8000 wsgi:app`.
7. Sign in as admin → **System Settings**: enter the park's verified name, address, phone and email; activate payment methods with real account details; add sections and lot types. Then **Caretaker Management** → add caretakers, and **Lot Inventory** → add lots.

Scheduled jobs (reservation expiry, payment reminders, overdue penalties, email delivery) run with
`flask --app wsgi run-jobs` every 15 minutes, or in-process with `ENABLE_SCHEDULER=true` on a single worker.
See `docs/DEPLOYMENT.md`.

## Environment variables

| Variable | Required | Purpose |
|---|---|---|
| `FLASK_SECRET_KEY` | yes | Signs the session cookie. Long random value. |
| `SUPABASE_URL`, `SUPABASE_ANON_KEY` | yes | Project API URL and public key |
| `SUPABASE_SERVICE_ROLE_KEY` | yes | Server-only key for staff provisioning and jobs |
| `PUBLIC_BASE_URL` | yes | Absolute site URL for email links and QR codes |
| `APP_ENV` | no | `development` enables debug |
| `SESSION_COOKIE_SECURE` | no | `true` in production (HTTPS) |
| `RATELIMIT_STORAGE_URI` | no | `redis://…` when running several workers |
| `ENABLE_SCHEDULER`, `JOB_INTERVAL_MINUTES` | no | In-process background jobs |
| `SMTP_*`, `MAIL_FROM` | no | Email notifications; skipped when empty |

## Business flow

```
visitor ── sign up (always CLIENT) ── verify email ── sign in
client  ── request reservation ─────────────▶ pending   (one active request per lot)
caretaker ── approve (sets expiry) ─────────▶ lot RESERVED, booking + payment schedule created
client  ── upload ID & signed form, pay + upload proof ─▶ booking "under verification"
caretaker ── verify documents & payment (receipt issued) ─▶ confirm ─▶ lot SOLD (purchased)
caretaker ── schedule burial (hours, daily limit, no overlaps) ─▶ complete ─▶ lot OCCUPIED + QR code
job     ── unpaid approved reservation past expiry ─▶ expired, lot released (never if payments exist)
```

## Testing

```bash
pip install -r requirements-dev.txt
pytest tests/test_unit.py                  # no database needed
```
The SQL workflow tests and the end-to-end suite need a local PostgreSQL + PostgREST; see `docs/TESTING.md`.
Current results: 62 database assertions and 38 Python tests passing.

## Notes and limits

- The cemetery map is a **schematic layout** in abstract units; it never claims to show real geography. Enter surveyed positions per lot (and set `map_is_surveyed`) if you have them.
- Contact details, payment account numbers, penalties and policies are **left empty or at conservative defaults** and must be entered by the administrator; nothing is invented.
- The privacy policy and terms describe how the system handles data; have them reviewed by the park's counsel before launch.
