# Permissions

## Roles
STUDENT · GUARD · ADMIN (users may hold multiple).

## Guard rule (v1)
`canPerformGuardAction(user) = user.status == ACTIVE AND user.hasRole(GUARD)`
All active guards permitted; no location scoping in v1.

## Matrix
| Action | Student | Guard | Admin |
|---|---|---|---|
| Report loss / claim / view own claims | ✔ | – | – |
| Intake item / update tags | – | ✔ | ✔ |
| Verify pickup / confirm handover / escalate | – | ✔ | – |
| Approve / reject / resolve dispute | – | – | ✔ |
| View score breakdown / original during review | – | – | ✔ |
| Manage policies | – | – | ✔ |

## Object-level rules
- Students see only own reports/claims/chats.
- Original photo: owner + READY + not expired + not DISPUTED + not legal_hold.
- Guards never see private_identifiers; admins only via decrypt authorization.