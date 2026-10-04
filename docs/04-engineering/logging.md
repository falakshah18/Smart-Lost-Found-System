# Logging & Monitoring

## Operational logs
- Structured JSON logs with request/correlation IDs.
- Log: endpoint, status, latency, actor id, entity ids, error type.
- NEVER log: secrets, pickup codes, full student IDs, private identifiers, photo bytes/URLs, LLM raw text.

## Audit logs (separate)
- actor, action, entity, minimized before/after summaries, ip/ua.
- Anonymize per retention rules.

## Monitoring & alerts
- Queue backlog, dead jobs, failed emails, failed uploads, pickup-code failure spikes, API 5xx rate, DB errors, backup/restore failures.
- Alerts create SYSTEM_ALERTS + notify on-call.

## Retention of logs
Operational logs short-lived; audit per policy; DataMinimization job purges old chat/notification/idempotency data.