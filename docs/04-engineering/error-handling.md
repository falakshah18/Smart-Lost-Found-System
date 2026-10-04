# Error Handling

## API
- 4xx for client errors with plain-language message; 5xx logged + generic message.
- Never leak stack traces or internals.

## Retries
- Uploads: client retry with backoff (Uppy).
- Emails: outbox + worker retry with attempts/next_attempt_at.
- Jobs: Procrastinate retryWithBackoff; DEAD jobs alert.

## Must never fail silently
- Backup failure, restore-test failure, retention failure, queue backlog → SYSTEM_ALERTS + notification.

## Graceful degradation
- LLM down/disabled → structured form.
- CV failed → item still usable; processing_status FAILED.
- Email down → claim flows still commit; email retries later.

## User-facing errors
Plain language + next step ("Photo too large. Please retake with phone camera settings.").