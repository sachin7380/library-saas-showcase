# Library SaaS — Backend

FastAPI backend for a multi-tenant study-library management SaaS. Organizations manage library branches, staff, students, batches, attendance, and fee collection. A separate platform-admin surface manages customer organizations and subscription plans.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | FastAPI |
| ORM | SQLAlchemy 2.x |
| Database | PostgreSQL |
| Migrations | Alembic |
| Schemas | Pydantic v2 |
| Auth | JWT (`python-jose`) + Passlib bcrypt |
| Logging | structlog |

---

## Local Setup

All commands run from the `backend/` directory.

**1. Activate the virtual environment**

```powershell
venv\Scripts\activate
```

**2. Install dependencies**

```powershell
pip install -r requirements.txt
```

**3. Create `.env`**

```env
DATABASE_URL=postgresql://postgres:root@localhost/library_saas
JWT_SECRET=your-random-secret-here
```

**4. Run migrations**

```powershell
alembic upgrade head
```

**5. Start the dev server**

```powershell
uvicorn app.main:app --reload
```

| URL | Purpose |
|---|---|
| `http://127.0.0.1:8000/docs` | Swagger UI |
| `http://127.0.0.1:8000/api/v1/` | API root |
| `http://127.0.0.1:8000/api/v1/health/db` | DB health check |

---

## Environment Variables

| Variable | Purpose | Default |
|---|---|---|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://postgres:root@localhost/library_saas` |
| `JWT_SECRET` | Required. Signs both tenant and platform JWTs | none |
| `JWT_ALGORITHM` | JWT algorithm | `HS256` |
| `JWT_ACCESS_TOKEN_EXPIRE_MINUTES` | Access token lifetime (minutes) | `60` |
| `JWT_REFRESH_TOKEN_EXPIRE_DAYS` | Refresh session lifetime (days) | `30` |
| `CORS_ALLOWED_ORIGINS` | Comma-separated allowed frontend origins | `http://localhost:3000,http://localhost:5173` |
| `CORS_ALLOW_CREDENTIALS` | Whether CORS allows credentials | `true` |
| `TRUSTED_HOSTS` | Comma-separated allowed `Host` header values | `localhost,127.0.0.1,testserver` |
| `LOGIN_RATE_LIMIT_ATTEMPTS` | Login attempts allowed per window | `5` |
| `LOGIN_RATE_LIMIT_WINDOW_SECONDS` | Rate-limit window in seconds | `300` |

> **Note:** Use `JWT_ACCESS_TOKEN_EXPIRE_MINUTES`, not `JWT_EXPIRE_MINUTES` — the old name has no effect.

---

## Database Migrations

```powershell
# Apply all migrations
alembic upgrade head

# Generate a new migration from model changes
alembic revision --autogenerate -m "description"

# Roll back one step
alembic downgrade -1
```

When adding a new model, import it in `app/core/database/models.py` so Alembic autogenerate can detect it.

### Migration chain

| Migration | Purpose |
|---|---|
| `6295225ef71d` | Initial multi-tenant architecture |
| `1e9d20da5586` | Library tenancy (no-op link) |
| `3f4a2b7c9d10` | Platform subscriptions — seeds TRIAL/BASIC/PRO/ENTERPRISE plans |
| `7a8b9c0d1e2f` | Student batches, memberships, payments |
| `8b9c0d1e2f30` | Attendance, dashboard, seat map (`capacity` on libraries) |
| `9c1d2e3f4a50` | Composite indexes for hot API queries |
| `a1b2c3d4e5f6` | Partial unique indexes for soft-delete uniqueness |
| `b2c3d4e5f6a7` | Auth sessions and access-token revocation |
| `c3d4e5f6a7b8` | Database index optimization |

---

## Testing

```powershell
# All tests (DB-backed tests skip without TEST_DATABASE_URL)
venv\Scripts\python.exe -m pytest

# By marker
venv\Scripts\python.exe -m pytest -m unit
venv\Scripts\python.exe -m pytest -m api
venv\Scripts\python.exe -m pytest -m auth
venv\Scripts\python.exe -m pytest -m tenant
venv\Scripts\python.exe -m pytest -m concurrency
venv\Scripts\python.exe -m pytest -m subscriptions

# Single file or test
venv\Scripts\python.exe -m pytest tests/test_auth_api.py
venv\Scripts\python.exe -m pytest tests/test_auth_api.py::test_login_success
```

**Database-backed tests** require `TEST_DATABASE_URL`:

```powershell
$env:TEST_DATABASE_URL="postgresql://postgres:root@localhost/library_saas_test"
venv\Scripts\python.exe -m pytest
```

The URL must contain `test` — the fixture refuses to reset any database whose URL does not look like a test database.

| Marker | What it covers |
|---|---|
| `unit` | Pure service logic, no database |
| `api` | Response envelope, versioning, OpenAPI shape |
| `auth` | Tenant registration, login, refresh rotation, logout, revocation |
| `tenant` | Cross-org data isolation |
| `concurrency` | PostgreSQL row-lock behavior for payment collection |
| `subscriptions` | Plan limit enforcement for students, staff, libraries |

See `TESTING_STRATEGY.md` for fixture details and the expansion checklist for new modules.

---

## Architecture

### Folder layout

```
app/
  core/
    config/settings.py        # Frozen dataclass, loaded from .env
    database/
      base.py                 # SQLAlchemy Base
      config.py               # Engine setup
      models.py               # Central import of all ORM models (required for Alembic)
      session.py              # get_db() dependency
    middleware.py             # RequestContextMiddleware, SecurityHeadersMiddleware
    security/
      passwords.py            # bcrypt via passlib
      tokens.py               # JWT creation and verification
  modules/
    auth/                     # Tenant registration, login, refresh, logout
    platform/                 # Platform admin auth and org management
    organizations/            # Tenant org profile and subscription usage
    libraries/                # Library branches
    batches/                  # Time-slot/fee batches
    users/                    # Tenant staff accounts
    students/                 # Student records, memberships, payments, attendance
    payments/                 # Org-level payment and outstanding lists
    subscriptions/            # Subscription plans, org subscriptions, invoices
    attendance/               # Today's attendance list
    dashboard/                # Aggregated dashboard summary
    audit_logs/               # Audit event log
  shared/
    models/base_entity.py     # BaseEntity: UUID PK, timestamps, soft delete
    exceptions.py             # AppException + exception handlers
    responses.py              # ResponseEnvelopeMiddleware
    pagination.py             # PaginationParams + PaginatedResponse
    rate_limit.py             # In-memory login rate limiter
    request_context.py        # Request ID context var
```

Each domain module follows: `models.py → repository.py → service.py → schemas.py → router.py`.

### Two identity types

| Type | Module | JWT `token_type` | Scope |
|---|---|---|---|
| Tenant user | `modules/auth` | `tenant` | Scoped to one organization |
| Platform admin | `modules/platform` | `platform` | Cross-organization |

Tenant and platform tokens cannot call each other's APIs.

### Response envelope

Every JSON response is wrapped by `ResponseEnvelopeMiddleware`:

```json
{
  "success": true,
  "data": {},
  "error": null,
  "meta": { "request_id": "..." }
}
```

Raise `AppException` for domain errors, never `HTTPException` directly.

### Tenant data scoping rules

- Every tenant table includes `organization_id`. Repositories always filter by it.
- `STAFF` and `RECEPTION` roles are additionally scoped to their assigned `library_id`.
- Commits happen in the service layer, not the repository layer.
- Always filter `is_deleted.is_(False)` — soft deletes are used everywhere.

### Tenant roles

| Role | Access |
|---|---|
| `OWNER` | Full tenant administration |
| `ADMIN` | Manage libraries, staff, batches, students, payments, attendance |
| `STAFF` | Assigned-library operational data only |
| `RECEPTION` | Same operational scope as STAFF |

### Common error codes

| Code | HTTP | Meaning |
|---|---|---|
| `UNAUTHENTICATED` | 401 | Missing, invalid, expired, or wrong-type token |
| `PAYMENT_REQUIRED` | 402 | No active subscription |
| `FORBIDDEN` | 403 | Wrong role or library scope |
| `NOT_FOUND` | 404 | Resource not found |
| `CONFLICT` | 409 | Uniqueness violation or plan limit exceeded |
| `VALIDATION_ERROR` | 422 | Request body failed schema validation |
| `RATE_LIMITED` | 429 | Too many login attempts |

> All monetary amounts are in **paise** (1 INR = 100 paise). Convert to rupees only in the UI.

---

## API Reference

All endpoints live under `/api/v1`. Legacy unversioned routes remain mounted but hidden from OpenAPI.

---

### Health

#### `GET /api/v1/`
Returns a JSON message confirming the backend is running. No auth required.

#### `GET /api/v1/health/db`
Executes `SELECT 1` against PostgreSQL and returns `{"database": "connected"}`. No auth required. Useful for container health checks.

---

### Authentication — `/api/v1/auth`

#### `POST /auth/register`
Self-signup for a new organization owner. Creates an `Organization`, an `OWNER` `User`, attaches the owner, and creates a default `TRIAL` subscription in one transaction. Returns a tenant JWT pair plus the created user and organization. Requires a seeded `TRIAL` plan to exist.

**Request:** `organization_name`, `full_name`, `email`, `password` (required) + `billing_email`, `phone` (optional).

**Errors:** `409` if email already registered; `500` if no active `TRIAL` plan is seeded.

---

#### `POST /auth/login`
Authenticates a tenant user by email and password. Returns an access token (short-lived JWT), a refresh token (opaque, stored hashed), and the user object. Rate-limited to `LOGIN_RATE_LIMIT_ATTEMPTS` per IP/email window.

**Request:** `email`, `password`.

**Errors:** `401` invalid credentials; `429` rate limited.

---

#### `POST /auth/token`
Identical to `/auth/login` but accepts `application/x-www-form-urlencoded` (`username`, `password`). Used exclusively by the Swagger UI OAuth form. Response is **not** wrapped in the envelope so Swagger can parse the raw OAuth shape.

---

#### `POST /auth/refresh`
Rotates the refresh token. Locks the active auth session row, validates that the tenant user and their organization are still active, replaces the stored refresh token hash, and returns a new access+refresh token pair. Always persist the new `refresh_token` — reusing the old one after rotation returns `401`.

**Request:** `refresh_token`.

**Errors:** `401` if the token is invalid, expired, already rotated, or the organization is suspended.

---

#### `POST /auth/logout`
Revokes the current access token (stores its `jti` in `revoked_access_tokens` until expiry) and either the provided refresh session or all active refresh sessions for the user when `revoke_all: true`.

**Auth:** Tenant bearer token.

**Request:** `refresh_token`, `revoke_all` (boolean, default `false`).

---

#### `GET /auth/me`
Returns the currently authenticated tenant user's profile: `id`, `full_name`, `email`, `role`, `organization_id`, `library_id`, `is_active`.

**Auth:** Tenant bearer token. All roles.

---

### Platform Auth — `/api/v1/platform/auth`

#### `POST /platform/auth/bootstrap`
Creates the first platform super admin. Fails once any active platform admin exists — this is a one-time setup endpoint. Returns a platform JWT pair and the created admin.

**Request:** `full_name`, `email`, `password`.

**Errors:** `409` if an active platform admin already exists.

> **Security note:** This endpoint is only gated by "no active admin exists". Add environment-level protection before production deployment.

---

#### `POST /platform/auth/login`
Authenticates a platform admin by email and password. Returns a platform access token (JWT with `token_type: "platform"`), a refresh token, and the admin object. Rate-limited same as tenant login.

**Request:** `email`, `password`.

**Errors:** `401` invalid credentials; `429` rate limited.

---

#### `POST /platform/auth/refresh`
Same rotation logic as tenant refresh but for platform admin sessions. Validates the platform admin is still active before issuing a new token pair.

**Request:** `refresh_token`.

---

#### `POST /platform/auth/logout`
Revokes the current platform access token JTI and one or all platform refresh sessions.

**Auth:** Platform bearer token.

**Request:** `refresh_token`, `revoke_all` (boolean).

---

### Platform Organizations — `/api/v1/platform/organizations`

#### `GET /platform/organizations`
Lists all customer organizations with pagination. Returns `name`, `slug`, `status`, `subscription_plan`, `subscription_status`, and timestamps.

**Auth:** Platform bearer token.

---

#### `POST /platform/organizations`
Manually onboards a new customer organization on behalf of an owner. Creates the `Organization`, the owner `User` (role `OWNER`), attaches the owner, creates a default `TRIAL` subscription, and writes an `organization.created` audit log entry. Returns both the organization and the owner user.

**Auth:** Platform bearer token.

**Request:** `organization_name`, `owner_full_name`, `owner_email`, `owner_password` (required) + `billing_email`, `phone` (optional).

**Errors:** `409` if owner email already registered.

---

#### `GET /platform/organizations/{organization_id}`
Fetches a single customer organization by UUID.

**Auth:** Platform bearer token.

**Errors:** `404` if not found.

---

#### `PATCH /platform/organizations/{organization_id}/suspend`
Suspends a customer organization. Sets status to `SUSPENDED`, records `suspended_at` and `suspended_reason`, and writes a suspension audit log. Tenant users from this organization will be rejected by `get_current_user` and `refresh` until the org is reactivated.

**Auth:** Platform bearer token.

**Request:** `reason` (3–255 chars, required).

**Errors:** `404` if not found.

---

#### `PATCH /platform/organizations/{organization_id}/activate`
Reactivates a suspended organization. Sets status back to `ACTIVE`, clears `suspended_at` and `suspended_reason`, and writes an activation audit log.

**Auth:** Platform bearer token.

**Errors:** `404` if not found.

---

### Platform Subscriptions — `/api/v1/platform`

#### `GET /platform/subscription-plans`
Lists all subscription plans with their limits (`max_libraries`, `max_staff`, `max_students`) and pricing in paise. `null` limits mean unlimited.

**Auth:** Platform bearer token.

---

#### `POST /platform/subscription-plans`
Creates a new subscription plan. `code` and `currency` are uppercased automatically. Writes an audit log.

**Auth:** Platform bearer token.

**Request:** `code` (unique), `name` (required) + `description`, `monthly_price_paise`, `annual_price_paise`, `currency`, `max_libraries`, `max_staff`, `max_students`, `features`, `is_active` (all optional).

**Errors:** `409` if plan code already exists.

---

#### `PATCH /platform/subscription-plans/{plan_id}`
Updates editable fields on an existing plan. Cannot change `code`. All fields are optional.

**Auth:** Platform bearer token.

**Errors:** `404` if not found.

---

#### `GET /platform/organizations/{organization_id}/subscription`
Returns the current active subscription for an organization, including plan details, status, period dates, and trial end date.

**Auth:** Platform bearer token.

**Errors:** `404` if the organization has no subscription.

---

#### `PATCH /platform/organizations/{organization_id}/subscription`
Replaces the organization's current subscription with a new plan, status, and billing period. Updates the denormalized `subscription_plan` and `subscription_status` fields on the `Organization` row and writes an audit log.

**Auth:** Platform bearer token.

**Request:** `plan_id` (required active plan UUID) + `status`, `starts_at`, `ends_at`, `trial_ends_at`, `current_period_start`, `current_period_end` (all optional).

**Errors:** `404` if organization or plan not found.

---

#### `GET /platform/payments`
Lists all subscription payment transactions across all organizations. Payment gateway creation and capture workflows are not yet implemented — records are currently created manually.

**Auth:** Platform bearer token.

---

#### `GET /platform/invoices`
Lists all subscription invoices across all organizations, with `invoice_number`, `amount_paise`, `tax_paise`, `total_paise`, `status`, `issued_at`, `due_at`, and `paid_at`.

**Auth:** Platform bearer token.

---

### Tenant Organization — `/api/v1/organizations`

#### `GET /organizations/current`
Returns the authenticated user's organization: `name`, `slug`, `status`, `subscription_plan`, `subscription_status`, and related fields.

**Auth:** Tenant bearer token. Roles: `OWNER`, `ADMIN`.

---

#### `GET /organizations/current/subscription`
Returns the organization's current subscription details including plan limits and billing period.

**Auth:** Tenant bearer token. Roles: `OWNER`, `ADMIN`.

**Errors:** `404` if no subscription exists.

---

#### `GET /organizations/current/usage`
Returns current usage counts vs. plan limits: `current_library_count`, `current_staff_count`, `current_student_count` alongside the plan's `max_*` values. Requires an active subscription.

**Auth:** Tenant bearer token. Roles: `OWNER`, `ADMIN`.

**Errors:** `402` if no active subscription.

---

### Libraries — `/api/v1/libraries`

#### `POST /libraries/`
Creates a library branch inside the organization. Enforces the subscription `max_libraries` limit. Library `name` and `code` must be unique within the organization.

**Auth:** Tenant bearer token. Roles: `OWNER`, `ADMIN`.

**Request:** `name` (required) + `code`, `address_line1`, `city`, `state`, `contact_phone`, `capacity` (all optional).

**Errors:** `402` inactive subscription; `409` library limit exceeded; `409` name or code conflict.

---

#### `GET /libraries/`
Lists all library branches the user can access. Owners/admins receive all organization libraries. `STAFF`/`RECEPTION` users receive only their assigned library.

**Auth:** Tenant bearer token. All roles.

---

#### `GET /libraries/{library_id}`
Fetches a single library branch by UUID. Staff/reception users are rejected if the ID does not match their assigned library.

**Auth:** Tenant bearer token. All roles.

**Errors:** `403` if staff tries to access another library; `404` if not found.

---

#### `GET /libraries/{library_id}/seat-map`
Returns the seat occupancy grid for a library. Uses `library.capacity` as the total seat count; falls back to the highest assigned `seat_number` if capacity is not set. Each occupied seat includes the student's name and a computed fee summary (fee, paid, remaining, status). Optimized to avoid N+1 queries — performs one aggregate join across students, memberships, and grouped payments after the library lookup.

**Auth:** Tenant bearer token. All roles.

**Response:** `{ library_id, capacity, seats: [{ seat_number, is_occupied, student_id, student_name, fee_summary }] }`

---

#### `PATCH /libraries/{library_id}`
Updates editable fields on a library branch. Can also set `is_active` to deactivate a branch. Name/code uniqueness is still enforced.

**Auth:** Tenant bearer token. Roles: `OWNER`, `ADMIN`.

**Errors:** `404` not found; `409` name or code conflict.

---

#### `DELETE /libraries/{library_id}`
Soft-deletes a library branch. Returns `204 No Content`.

**Auth:** Tenant bearer token. Roles: `OWNER`, `ADMIN`.

---

### Batches — `/api/v1/batches`

#### `POST /batches/`
Creates a time-slot and fee batch inside a library. Owners/admins must provide `library_id`. `(library_id, name)` must be unique.

**Auth:** Tenant bearer token. Roles: `OWNER`, `ADMIN`.

**Request:** `library_id` (required for owner/admin), `name`, `start_time` (`HH:MM:SS`), `end_time`, `fee_amount_paise` (all required) + `billing_period` (default `MONTHLY`), `capacity` (optional).

**Errors:** `400` missing `library_id`; `404` library not found; `409` batch name conflict.

---

#### `GET /batches/`
Lists batches for the organization or a specific library (`?library_id=`). Staff/reception users are automatically scoped to their assigned library.

**Auth:** Tenant bearer token. All roles.

---

#### `GET /batches/{batch_id}`
Fetches a single batch by UUID.

**Auth:** Tenant bearer token. All roles.

**Errors:** `404` not found.

---

#### `PATCH /batches/{batch_id}`
Updates editable batch fields. `library_id` cannot be changed. Changing `fee_amount_paise` does **not** affect existing student memberships — memberships store a fee snapshot taken at the time of creation.

**Auth:** Tenant bearer token. Roles: `OWNER`, `ADMIN`.

**Errors:** `404` not found; `409` name conflict.

---

#### `DELETE /batches/{batch_id}`
Soft-deletes a batch. Returns `204 No Content`.

**Auth:** Tenant bearer token. Roles: `OWNER`, `ADMIN`.

---

### Users (Staff) — `/api/v1/users`

#### `GET /users/`
Lists all staff/admin users in the organization with their `role`, `library_id`, and `is_active` status.

**Auth:** Tenant bearer token. Roles: `OWNER`, `ADMIN`.

---

#### `POST /users/`
Creates a new tenant staff user. Enforces the subscription `max_staff` limit for billable roles (`ADMIN`, `STAFF`, `RECEPTION`). Cannot create an `OWNER` through this endpoint. `STAFF`/`RECEPTION` require a `library_id`; `ADMIN` must not have one.

**Auth:** Tenant bearer token. Roles: `OWNER`, `ADMIN`.

**Request:** `full_name`, `email`, `password`, `role` (required) + `library_id` (required for STAFF/RECEPTION).

**Errors:** `400` role/library rule violations; `403` only owners can create admins; `404` library not found; `409` email conflict or staff limit exceeded.

---

#### `GET /users/{user_id}`
Fetches a single staff user by UUID.

**Auth:** Tenant bearer token. Roles: `OWNER`, `ADMIN`.

**Errors:** `404` not found.

---

#### `PATCH /users/{user_id}`
Edits a staff user's `full_name`, `role`, `library_id`, or `is_active`. Owner users cannot be managed here. Only owners can promote users to `ADMIN`. Staff/reception must keep a `library_id`; admins cannot have one.

**Auth:** Tenant bearer token. Roles: `OWNER`, `ADMIN`.

**Errors:** `400`/`403` role rule violations; `404` not found; `409` uniqueness conflict.

---

#### `DELETE /users/{user_id}`
Soft-deactivates a staff user. Cannot deactivate yourself or an owner. Returns `204 No Content`.

**Auth:** Tenant bearer token. Roles: `OWNER`, `ADMIN`.

**Errors:** `400` if trying to deactivate self or an owner; `404` not found.

---

### Students — `/api/v1/students`

#### `POST /students/`
Creates a new student record and optionally assigns them to a batch. If `batch_id` is provided, immediately creates an active `StudentMembership` using the batch's current fee and billing period (fee is snapshotted). Enforces the subscription `max_students` limit. `(library_id, phone)` must be unique.

**Auth:** Tenant bearer token. All roles.

**Request:** `full_name`, `phone` (required) + `library_id` (required for owner/admin, inferred for staff/reception), `batch_id`, `email`, `guardian_phone`, `address_line1`, `seat_number`, `date_of_joining` (all optional).

**Errors:** `402` inactive subscription; `403` library scope violation; `404` library or batch not found; `409` student limit exceeded or phone conflict.

---

#### `GET /students/`
Lists students in the organization or a specific library (`?library_id=`). Staff/reception users are automatically scoped to their assigned library. Supports pagination, search, and sorting.

**Auth:** Tenant bearer token. All roles.

---

#### `GET /students/{student_id}`
Fetches a single student by UUID, scoped to the user's organization and library.

**Auth:** Tenant bearer token. All roles.

**Errors:** `404` not found.

---

#### `PATCH /students/{student_id}`
Updates student profile fields: `full_name`, `phone`, `email`, `guardian_phone`, `address_line1`, `seat_number`, `status`. Phone uniqueness is still enforced within the library.

**Auth:** Tenant bearer token. All roles.

**Errors:** `404` not found; `409` phone conflict.

---

#### `DELETE /students/{student_id}`
Soft-deletes a student. Returns `204 No Content`.

**Auth:** Tenant bearer token. All roles.

---

#### `GET /students/{student_id}/fee-summary`
Returns the computed fee status for the student's current active membership. Calculates `fee_amount_paise`, `paid_amount_paise`, `remaining_amount_paise`, `cycle_start`, `cycle_end`, and `payment_status`.

Payment status logic:
- `PAID` — paid ≥ fee
- `OVERDUE` — remaining > 0 and today is after `cycle_end`
- `PARTIAL` — some paid but less than fee
- `DUE` — no payment yet and cycle has not expired

If the student has no active membership, returns a zero-fee `PAID` summary.

**Auth:** Tenant bearer token. All roles.

---

#### `GET /students/{student_id}/payments`
Lists all recorded fee payments for the student, ordered by `payment_date`. Each record includes `amount_paise`, `mode`, `receipt_number`, `note`, and `created_by_user_id`.

**Auth:** Tenant bearer token. All roles.

---

#### `POST /students/{student_id}/payments`
Records a student fee payment against the current active membership. Generates a `receipt_number`. Does not allow overpayment (amount must not exceed `remaining_amount_paise`). Returns both the created payment record and the updated fee summary.

**Auth:** Tenant bearer token. All roles.

**Request:** `amount_paise` (required, > 0) + `payment_date` (default today), `mode` (default `CASH`), `note` (optional).

**Errors:** `400` amount exceeds remaining fee; `404` no active membership.

---

#### `GET /students/{student_id}/memberships`
Lists all memberships (current and historical) for the student, including batch reference, fee snapshot, billing period, cycle dates, and status (`ACTIVE`, `CANCELLED`, `COMPLETED`).

**Auth:** Tenant bearer token. All roles.

---

#### `GET /students/{student_id}/memberships/current`
Returns only the student's current active membership. Used by the fee collection modal and billing card.

**Auth:** Tenant bearer token. All roles.

**Errors:** `404` if no active membership exists.

---

#### `POST /students/{student_id}/memberships`
Assigns the student to a new batch (or renews their current batch). Cancels and soft-deletes the existing active membership, creates a new one using the selected batch's current fee and billing period, and updates `student.batch_id`.

**Auth:** Tenant bearer token. All roles.

**Request:** `batch_id` (required, must belong to student's library) + `cycle_start` (default today).

**Errors:** `404` batch not found or not in student's library.

---

#### `PATCH /students/{student_id}/memberships/{membership_id}`
Manually edits membership fields: `batch_id`, `billing_period`, `fee_amount_paise`, `cycle_start`, `cycle_end`, `status`. All fields are optional. Can override fee snapshots and cycle dates for admin corrections. If `batch_id` changes, updates `student.batch_id`.

**Auth:** Tenant bearer token. All roles.

**Errors:** `404` membership or batch not found.

---

#### `POST /students/{student_id}/attendance/check-in`
Marks the student as present for today. If a `PRESENT` row already exists for today, returns it unchanged. If a `CHECKED_OUT` row exists, resets it to `PRESENT` and clears `check_out_at`.

**Auth:** Tenant bearer token. All roles.

**Response:** The attendance row with `attendance_date`, `status: PRESENT`, `check_in_at`.

---

#### `POST /students/{student_id}/attendance/check-out`
Marks today's attendance row as `CHECKED_OUT` and sets `check_out_at` to now. Fails if the student has not checked in today.

**Auth:** Tenant bearer token. All roles.

**Errors:** `404` student not checked in today.

---

#### `GET /students/{student_id}/attendance`
Lists the full attendance history for a student: date, status (`PRESENT` / `CHECKED_OUT`), `check_in_at`, `check_out_at`, and who marked it.

**Auth:** Tenant bearer token. All roles.

---

#### `PATCH /students/{student_id}/seat`
Assigns or changes a student's seat number. `seat_number` must be ≥ 1 and unique within the library.

**Auth:** Tenant bearer token. All roles.

**Request:** `seat_number`.

**Errors:** `409` seat already assigned to another student in this library.

---

### Payments — `/api/v1/payments`

#### `GET /payments/`
Lists all student fee payment records across the organization or a specific library (`?library_id=`). Staff/reception users are automatically scoped to their assigned library. Supports pagination, search, and sorting.

**Auth:** Tenant bearer token. All roles.

---

#### `GET /payments/outstanding`
Lists all students who have a remaining unpaid balance on their current active membership (`remaining_amount_paise > 0`). Each result includes the student name, seat number, and a full fee summary. Uses one SQL aggregate query (joins students to current membership and grouped payment totals) — does not iterate per-student in Python.

**Auth:** Tenant bearer token. All roles.

**Query params:** `library_id` (optional).

---

### Attendance — `/api/v1/attendance`

#### `GET /attendance/today`
Returns all attendance rows for the current date. Staff/reception users are automatically scoped to their assigned library. Each row includes `student_id`, `attendance_date`, `status`, `check_in_at`, `check_out_at`, and who marked it.

**Auth:** Tenant bearer token. All roles.

**Query params:** `library_id` (optional).

---

### Dashboard — `/api/v1/dashboard`

#### `GET /dashboard/summary`
Returns aggregated operational metrics for the main desk view. Uses fixed-round-trip SQL aggregation — approximately 5 SQL statements regardless of student count.

**Auth:** Tenant bearer token. All roles.

**Query params:** `library_id` (optional).

**Response fields:**

| Field | Description |
|---|---|
| `total_students` | Active student count |
| `present_today` | Students checked in today |
| `pending_fees_paise` | Total remaining fee across all active memberships |
| `overdue_count` | Students with overdue payment status |
| `collection_rate` | Percentage of fee collected this cycle |
| `seats_filled` | Students with an assigned seat number |
| `today_collection_paise` | Total payments recorded today |
| `batch_attendance` | Per-batch present vs. total student counts |
| `priority_collections` | Top overdue/partial students with their fee summaries |

---

### Audit Logs — `/api/v1/audit-logs`

#### `GET /audit-logs/`
Returns the 100 most recent audit events for the organization, ordered by `created_at desc`. Each record includes `actor_type` (`PLATFORM_ADMIN` or `TENANT_USER`), `actor_id`, `action` (e.g. `organization.created`, `organization.suspended`), `entity_type`, `entity_id`, `metadata_json`, and `ip_address`.

**Auth:** Tenant bearer token. All roles.

---

## Pagination, Filtering, and Sorting

All list endpoints support SQL-level pagination.

| Parameter | Default | Validation |
|---|---|---|
| `page` | `1` | ≥ 1 |
| `page_size` | `20` | 1–100 |
| `search` | none | 1–120 chars |
| `sort_by` | endpoint default | must be one of the allowed fields for that endpoint |
| `sort_order` | `asc` | `asc` or `desc` |

Paginated responses include a `pagination` object:

```json
{
  "items": [],
  "pagination": {
    "page": 1,
    "page_size": 20,
    "total_items": 0,
    "total_pages": 0
  }
}
```

---

## Known Gotchas

- `JWT_EXPIRE_MINUTES` in `.env` has **no effect**. The code reads `JWT_ACCESS_TOKEN_EXPIRE_MINUTES`.
- A `TRIAL` subscription plan must exist and be active before the first `POST /auth/register` call succeeds. The migration `3f4a2b7c9d10` seeds this.
- The platform bootstrap endpoint is only gated by "no active admin exists". Add environment-level protection before production deployment.
- Changing a batch fee does **not** update existing memberships — memberships store a fee snapshot.
- Refresh-token reuse after rotation returns `401`. Always persist the new `refresh_token` from refresh responses.
- `GET /libraries/{library_id}/seat-map` can return `capacity: 0` and an empty `seats` array if capacity is unset and no seats are assigned.
- Student payment cannot exceed `remaining_amount_paise`. Record a payment only when an active membership exists.
- Payment gateway creation/capture for platform subscriptions is not yet implemented. Transactions are manual.
