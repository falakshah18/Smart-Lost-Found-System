# Testing Strategy

## Unit
Matching scoring, policy rules (auto-ready/admin review), authorization checks, pickup-code hashing/lockout, retention eligibility.

## Integration (API + DB)
- Claim idempotency returns previous response.
- Partial unique index blocks second active claim.
- Original photo URL denied for PENDING/DISPUTED/EXPIRED/legal-hold.
- Rate limit triggers 429 and lockout after N failed codes.
- Status CHECK constraints reject invalid values.

## E2E (core journeys)
Intake → match → claim → approve → pickup → return; dispute path; expiry path.

## Security tests
IDOR (other user's claim/report), role escalation (guard→admin), brute force code, upload of bad/malicious files, signed URL expiry.

## Operational
Backup restore test scheduled; staging environment for UAT; pilot = real-world UAT.