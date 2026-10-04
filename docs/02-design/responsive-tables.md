# Responsive Tables

## Pattern
- ≥1024px: full table with sticky header + pagination.
- <1024px: cards (key fields only) or horizontal scroll with pinned first column.

## Tables & column priority
| Table | Always visible | Desktop-only |
|---|---|---|
| Pending claims | item, category, claimant, status, age | score, location, created_at |
| Found items | title, category, status, storage | label_code, custodian, found_at |
| Lost reports | category, status, created | location, user |
| Disputes | item, claimant, reason, status | guard, created_at |
| Audit logs | action, entity, actor, time | ip, before/after summary |

## Rules
- Dates localized; statuses use status colors; row click opens detail page.