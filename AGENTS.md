# AGENTS.md

## Setup
- `composer setup` — full setup (composer install, .env copy, key generate, migrate, npm install & build)
- `.env` uses **MySQL** (db `vaccination_sys`); `.env.example` defaults to **SQLite** — tests always use SQLite in-memory regardless (see `phpunit.xml`)
- Admin seed: `php artisan db:seed` or `composer setup` (includes migrate+seed)
  - Admin login: `admin@covid.com` / `admin123`

## Development
- `composer dev` — runs 4 concurrent processes via `npx concurrently`: `php artisan serve`, `queue:listen`, `pail` (logs), `npm run dev` (Vite)
- Vite entrypoints: `resources/css/app.css`, `resources/js/app.js`
- Frontend stack: Tailwind CSS 4 (no PostCSS config), Vite 7, `laravel-vite-plugin`

## Testing
- `composer test` — runs `artisan config:clear --ansi` then `artisan test` (Laravel's test runner, not phpunit directly)
- Single test: `php artisan test tests/Feature/ExampleTest.php` (or `--filter`)
- Suites: `tests/Unit`, `tests/Feature`
- Feature tests that hit DB **must** use `DatabaseMigrations` trait (SQLite in-memory, no migration auto-run)
- `composer test` runs `config:clear` first (needed after changing `config/auth.php` guards)

## Code style
- Laravel Pint installed (`./vendor/bin/pint`) — no composer script, run explicitly

## Architecture

### Roles & Auth
- **3 roles**: `patient` + `admin` share `users` table (col `role`), `hospital` uses separate `hospitals` table
- Two auth guards: `web` (patients/admins via `User` model), `hospital` (via `Hospital` model)
- Hospitals must be `status = 'approved'` to log in (checked in `HospitalAuthController@login`)
- Admin seeded by `AdminSeeder` — email `admin@covid.com`, password `admin123`
- Guest redirect routing configured in `bootstrap/app.php` via `redirectGuestsTo`

### Database (MySQL in dev, SQLite :memory: in tests)
| Table | Key cols |
|---|---|
| `users` | `name`, `email`, `password`, `role` (patient/admin), `phone`, `address`, `location` |
| `hospitals` | `hospital_name`, `email`, `password`, `status` (pending/approved/rejected) |
| `vaccines` | `vaccine_name`, `status` (available/unavailable) |
| `appointments` | `patient_id`, `hospital_id`, `type` (test/vaccination), `appointment_date`, `status`, `test_result`, `vaccination_status` |

### Routes
All route groups use `name()` prefixes — see `routes/web.php`:
- `patient.*` — public: login, register; guarded: dashboard, search, book, appointments, profile
- `hospital.*` — public: login, register; guarded: dashboard, requests, approve/reject, test-result, vaccination
- `admin.*` — public: login; guarded: dashboard, hospitals, vaccines, reports, export

### Controllers (all in `app/Http/Controllers/`)
- `Auth/PatientAuthController`, `Auth/HospitalAuthController`, `Auth/AdminAuthController`
- `PatientController` — search, book, appointments, profile
- `HospitalController` — approvals, test results, vaccination status
- `AdminController` — hospital verification, vaccine toggles, reports, Excel export

### Views
- All use Bootstrap 5 CDN (no asset pipeline needed for CSS/JS)
- Layout: `resources/views/layouts/app.blade.php`
- Auth views: `resources/views/auth/`
- Role dashboards: `resources/views/{patient,hospital,admin}/`

### Known
- `requirement.txt` at root is a leftover empty file — not used
- No CI workflows, no opencode config, no pre-commit hooks
- `config/auth.php` has been extended with `hospital` guard and provider




