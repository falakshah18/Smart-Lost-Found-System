# Authentication

## Primary: AU SSO (OIDC/SAML)
- Login via university IdP; map profile to local user; sync status/roles.
- Disabled users blocked at login and on every request.

## Fallback
Magic link exists in code but disabled by default; enable only if SSO unavailable and approved.

## Sessions
- Secure, HttpOnly, SameSite cookies; CSRF tokens; session timeout; logout.

## Service auth
Workers use scoped DB credentials; R2 keys scoped per operation; no shared secrets in code.