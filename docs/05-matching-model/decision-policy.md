# Decision Policy

Owner: matching model workstream · Status: proposed baseline (initial thresholds, to be tuned)
Cross-refs: matching-model-overview.md · score-breakdown-schema.md · user-flows.md (F4) · system-architecture.md (invariant 5) · permissions.md

## 1. Thresholds
Returned by `PolicyService.getMatchThresholds()`:
```json
{
  "version": "thresholds-1.0.0",
  "suggest_min": 0.40,
  "claim_min": 0.65,
  "auto_ready_min": 0.85,
  "ambiguity_margin": 0.10,
  "jev": {
    "mode": "OFF",
    "weight": 0.30,
    "veto_below": 0.50,
    "min_baseline": 0.30,
    "candidate_limit": 10,
    "timeout_ms": 2000
  }
}
```
- Values are proposed initial settings, not validated probabilities or empirically established safety cutoffs. Confirm the repository supports this configuration and audit path. Any change must be versioned and auditable (probability-research.md §6).
- `jev.mode`: `OFF`, `SHADOW` (answers recorded, not used), `ACTIVE`. Effective mode is `OFF` when feature flag `JEV_MATCHING` is disabled.
- Per-category floor: `categories.minimum_auto_ready_score`.
  `auto_ready_effective = max(auto_ready_min, category.minimum_auto_ready_score)`. The stricter value wins.

## 2. At match time
- `final_score < suggest_min` → no `potential_matches` row.
- `final_score >= suggest_min` → row created with status `NEW`; student sees a `MatchCard`.

## 3. At claim time (F4)
Evaluated when a claim is requested (idempotent, rate limited, ownership checked first). First rule that applies wins:

| # | Condition | Outcome | Reason code |
|---|---|---|---|
| 1 | `final_score < claim_min` | NEEDS_PROOF (no active claim; extra verification questions) | `BELOW_CLAIM_MIN` |
| 2 | Ambiguous (§4) | NEEDS_PROOF | `AMBIGUOUS` |
| 3 | Item `is_valuable` | PENDING_REVIEW | `VALUABLE_ITEM` |
| 4 | Category requires admin approval | PENDING_REVIEW | `CATEGORY_REQUIRES_APPROVAL` |
| 5 | Category does not allow auto-ready | PENDING_REVIEW | `CATEGORY_NO_AUTO_READY` |
| 6 | Any conflict flag | PENDING_REVIEW | `CONFLICT_FLAG` |
| 7 | `baseline_score < auto_ready_effective` or `final_score < auto_ready_effective` | PENDING_REVIEW | `BELOW_AUTO_READY` |
| 8 | Jev used and `same_object < jev.veto_below` | PENDING_REVIEW | `JEV_VETO` |
| 9 | otherwise | READY_FOR_PICKUP (auto-ready: pickup code + deadline) | `AUTO_READY` |

The decision and reasons are stored in the claim's `score_breakdown_json` (`decision` block).

## 4. Ambiguity
A pair is ambiguous if another candidate on the same side scores within `ambiguity_margin` of it:
- for a lost report: another found item with `final_score >= best − 0.10`
- for a found item: another open lost report with `final_score >= best − 0.10`

Only one active claim per found item is allowed (DB partial unique index), so an ambiguous pair must not auto-ready.

## 5. Invariant 5, extended
Original: auto-ready only if category allows + not valuable + not approval-required + score ≥ threshold.
Adds: no conflict flag, not ambiguous, baseline score also ≥ threshold, and no Jev veto.

Consequences:
- Jev can move a pair from NEEDS_PROOF into PENDING_REVIEW (a human still decides) or veto auto-ready.
- Jev can never produce READY_FOR_PICKUP: rule 7 requires the baseline alone to pass.
- With Jev OFF or failing, every rule above still runs unchanged.

## 6. What users see
Students see a label only. Numbers and the breakdown are admin-only (permissions.md).

| `final_score` | Student label |
|---|---|
| `suggest_min` to `< claim_min` | Possible match |
| `claim_min` to `< auto_ready_min` | Likely match |
| `>= auto_ready_min` | Strong match |

The label describes similarity only; it does not promise auto-ready (valuable items always go to review).

## 7. Re-evaluation
- Manual tag or report edits trigger rematch (`rerunMatchesForFoundItem`, `rerunMatchesForLostReport`); scores and bands are recomputed.
- An existing active claim is not silently changed by a rematch. A materially lower score on a rematch raises an admin notice (MonitoringService), it does not cancel the claim.
- Expired reports/items move their `potential_matches` to `EXPIRED` (FR-10).

## 8. Why this shape
- Pickup verification checks that the physical ID belongs to the claimant, not that the item belongs to the claimant. A wrongly auto-readied claim is therefore the costly error, and the policy is deliberately stricter on auto-ready than on suggestion.
- Sparse reports ("black wallet") cannot reach auto-ready because unknown features score 0.

