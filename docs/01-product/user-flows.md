# User Flows

## F1 — Finder handoff (physical)
1. Finder physically hands item to guard/reception.
2. No app usage by finder. Guard logs it with `intake_source = PUBLIC_HANDOFF/RECEPTION/GUARD_DIRECT`.

## F2 — Guard intake
Login (SSO) → capture photo → client compress/EXIF/blur attempt → select category/location/storage/label → presigned upload → server validation → item created → CV job + reverse matching.
Exceptions: validation failed → delete/quarantine; face blur failed → retake/manual review.

## F3 — Student lost report
Login → consent → Smart Assistant (LLM consent check → redaction → prefill) OR direct form → submit → private identifiers encrypted → matching runs → match result or "report open". (If LLM fails, falls back to direct form).

## F4 — Claim creation
Candidate found → claim request with idempotency key → policy evaluation:
- Weak/ambiguous → no active claim; NEEDS_PROOF; extra verification questions.
- Strong + low-risk + policy passes → auto-ready (code + deadline).
- Valuable / requires approval → PENDING_REVIEW.

## F5 — Admin review
Open queue → review details/score → approve (code generated) or reject (item available, match rejected).

## F6 — Pickup & handover
Student shows code + physical ID → guard verifies → checks (status, attempts, lockout, expiry, ID ownership) → confirm handover → claim/item RETURNED, report CLOSED, retention scheduled.
Failure → failed attempt recorded → escalate dispute.

## F7 — Dispute resolution
Admin approves (regenerate code if needed) or rejects (item available).

## F8 — System maintenance (background)
Expire overdue claims; expire stale reports; retry emails; retention (skip active/disputed/legal-hold); cleanup rate limits/uploads/quarantine/idempotency/chat; backups + restore tests; alerts.