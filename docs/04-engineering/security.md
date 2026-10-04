# Security

## AuthN/AuthZ
SSO-first; role checks + object ownership on every endpoint (permissions.md).

## Abuse protection
Rate limits (upload, report, claim, code verify, signed URLs); pickup code hashed, expiry, attempt counter, lockout.

## Uploads
Presigned URLs; server-side magic-byte/size/dimension checks; EXIF strip; face-blur check; quarantine + delete on failure.

## Storage & URLs
Private bucket; short-lived signed URLs; grants recorded; no public access.

## Data protection
Encryption at rest/in transit; field-level encryption for private identifiers/claim answers/chat; keys in KMS; minimal student ID storage (last4/hash).

## Web
HTTPS only; secure cookies; CSRF; CORS/CSP restricted; no secrets in frontend.

## Supply chain
Dependency scanning; container/image scanning; secrets scanning in CI.

## Incident basics
Runbook: detect → contain → notify AU IT/security → audit review → post-mortem.