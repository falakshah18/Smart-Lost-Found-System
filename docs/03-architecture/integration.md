# Integration

## AU official site
- Deploy as secure subdomain/portal link (e.g., lostfound.au.edu), not embedded in public CMS pages.
- CORS/CSP restricted to AU domains; no public API docs in prod.

## Required approvals
- AU IT: hosting, storage, email, SSO role mapping.
- Security/privacy: photo handling, retention, LLM usage.

## Third parties (each must be approved)
- Storage: R2 or AU-approved S3 (private bucket, lifecycle rules).
- Email: AU SMTP or approved transactional provider (SPF/DKIM).
- LLM: ENABLED in pilot with explicit LLM_ASSISTANCE consent, redaction, no private identifiers, paid/approved tier, rate limits + cost cap; form fallback mandatory.

## Feature flags
`LLM_ASSISTANT`, `CV_OBJECT_DETECTION`, etc. via PolicyService/FeatureFlagService.