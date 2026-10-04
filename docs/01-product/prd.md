# Product Requirements Document (PRD)

## Objective
Digitize campus lost & found with secure intake, matching, claim review, pickup verification, and retention.

## Roles
Student, Guard, Admin, Finder (non-app actor).

## Core features (MVP / pilot)
| ID | Feature | Summary |
|---|---|---|
| FR-01 | Guard intake | Photo upload, category/location/storage/label, server validation |
| FR-02 | Private storage | Originals private; blurred thumbnails; signed URLs |
| FR-03 | Lost report | Structured form + Smart Assistant (LLM prefill enabled, consent-gated, redacted, form fallback) |
| FR-04 | Matching | Deterministic two-way matching; score breakdown |
| FR-05 | Claims | Idempotent claim creation; one active claim per item |
| FR-06 | Policy engine | Auto-ready vs admin review vs needs-proof |
| FR-07 | Admin review | Approve / reject with reason; dispute resolution |
| FR-08 | Pickup | Pickup code + physical student ID verification |
| FR-09 | Disputes | Guard escalation; admin resolution |
| FR-10 | Expiry | Overdue claims and stale reports expire; item returns to available |
| FR-11 | Notifications | Queued emails with retry |
| FR-12 | Retention | Cleanup with legal-hold/active-record protection |
| FR-13 | Audit | Action logs with minimized payloads |

## Non-functional requirements
- Security: SSO, role checks, object-level authorization, rate limits.
- Privacy: encryption for private identifiers/claim answers; minimized student ID data.
- Reliability: background jobs, retries, backups + restore tests.
- Accessibility: WCAG 2.1 AA; mobile-first.

## Constraints / dependencies
- AU SSO for login; roles mapped from IdP.
- Storage/email/LLM must be AU-approved services.
- Pilot at one desk before full rollout.

## Acceptance criteria (examples)
- A guard can intake an item in < 30s on a phone.
- A duplicate claim request returns the previous result (idempotency).
- A READY claim past deadline becomes EXPIRED and item becomes AVAILABLE.
- Original photo URL is denied unless claim is READY, not expired, not disputed, no legal hold.