# Components

- **PhotoCapture**: camera/file input; client compress + EXIF strip + blur attempt.
- **UploadProgress**: retry + progress; error state with reason.
- **CategorySelect / LocationSelect**: searchable dropdowns from active lists.
- **MatchCard**: blurred teaser, category, location, score label, actions (claim / not mine).
- **ClaimStatusTimeline**: visual status history with deadline.
- **PickupCodeCard**: code + QR + expiry countdown.
- **ScoreBreakdown**: admin-only expandable match explanation.
- **DisputeBanner**: red banner with escalate/resolve actions.
- **EmptyState / ErrorState / Toast / ConfirmDialog**: standard patterns.
- **AdminTable**: sortable, paginated table (see responsive-tables.md).

Rules: components never show raw status enums to users; always mapped labels.