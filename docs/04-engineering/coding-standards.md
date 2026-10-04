# Coding Standards

- Frontend: React + TypeScript; strict mode; no `any`.
- Backend: FastAPI + Pydantic models; type hints everywhere.
- DB changes only via migrations; never manual edits in prod.
- Formatting/linting: Prettier + ESLint (TS); Ruff/Black (Py); CI must pass.
- Git: branch per feature; PR requires review; small commits; meaningful messages.
- No secrets in code; env/config via secret manager.
- Validate at boundaries (API); services assume validated input.
- Naming: domain-first (ClaimService.expireOverdueClaims), no abbreviations.
- Review checklist: authz checked? idempotency? audit? rate limit? error path? tests?