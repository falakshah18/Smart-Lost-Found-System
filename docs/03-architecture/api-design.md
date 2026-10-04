# API Design

REST over HTTPS, JSON, session-cookie auth, CSRF protection.

## Conventions
- Idempotency: `Idempotency-Key` header on claim creation; server stores minimized response.
- Rate limits per user+action; 429 with retry-after.
- Errors: `{ "error": { "code", "message", "details?" } }`; no stack traces.
- Pagination: `?page=&size=` with totals; all lists paginated.
- Versioning: `/api/v1`.

## Endpoint groups (main)
- `POST /upload-sessions`, `POST /upload-sessions/{id}/finalize`
- `POST /found-items`, `PATCH /found-items/{id}`
- `POST /lost-reports`, `POST /consent`
- `POST /claims`, `GET /claims/{id}`, `GET /claims/{id}/original-photo-url`
- `POST /handover/verify-code`, `POST /handover/confirm`, `POST /claims/{id}/dispute`
- Admin: `GET /admin/claims?status=`, `POST /admin/claims/{id}/approve|reject|resolve-dispute`
- Optional: `GET /assistant/config`, `POST /assistant/messages` (flag-gated)

## Authorization
Every endpoint enforces role + object ownership (see permissions.md).