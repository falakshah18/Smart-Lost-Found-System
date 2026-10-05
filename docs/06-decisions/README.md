# Architecture Decision Records (ADRs)

Cross-refs: system-architecture.md (version policy) · integration.md

## Rule
Any new technology, third-party service, or change to a core design rule needs an ADR here before use (system-architecture.md, Version policy). One decision per file. Accepted ADRs are not edited; a new ADR supersedes them.

## Index
| ADR | Title | Status | Referenced from |
|---|---|---|---|
| ADR-001 | PostgreSQL as database and job-queue backing | Referenced, file not yet in repo | database.md |
| ADR-002 | AU SSO as primary authentication | Referenced, file not yet in repo | system-architecture.md, auth.md |
| ADR-003 | Hosting and deployment approach | Referenced, file not yet in repo | system-architecture.md, integration.md |
| ADR-004 | Matching model architecture (deterministic baseline + optional Jev) | Proposed | 05-matching-model/ |

Owners of ADR-001 to ADR-003 should add their files; numbering is kept so existing references stay valid.

## Status values
`Proposed` → `Accepted` → (`Superseded by ADR-NNN` | `Rejected`)

## Template
```
# ADR-NNN — Title

Status: Proposed · Date: YYYY-MM-DD · Owner: <workstream>
Cross-refs: <docs>

## Context
What problem, which constraints (PRD, privacy, AU approvals).

## Decision
What we will do, in one short paragraph.

## Alternatives considered
- Option — why not.

## Consequences
- Positive / negative / follow-ups (approvals, migrations, tests, flags).

## Revisit when
Concrete triggers.
```

## File naming
`ADR-NNN-short-title.md`, numbers never reused.
