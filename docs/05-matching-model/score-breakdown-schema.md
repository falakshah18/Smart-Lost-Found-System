# Score Breakdown Schema

Owner: matching model workstream · Status: proposed baseline
Cross-refs: feature-spec.md · decision-policy.md · jev-integration.md · database.md · components.md

## 1. Where it is stored
- `potential_matches.score` = `final_score`; `potential_matches.score_breakdown_json` = the object below.
- `claims.match_score` and `claims.score_breakdown_json` = a snapshot copied at claim creation, plus a `decision` block (§4).
- No new tables or columns. Jev answers live inside this JSON, so a score can be replayed without calling Jev.

## 2. Shape (Jev ACTIVE example)
```json
{
  "model_version": "matching-1.0.0",
  "gates": {
    "category": true,
    "found_after_loss_start": true,
    "item_available": true,
    "report_open": true
  },
  "features": {
    "location": { "value": 1.0,  "weight": 0.25 },
    "time":     { "value": 0.86, "weight": 0.20 },
    "color":    { "value": 0.5,  "weight": 0.15 },
    "brand":    { "value": 1.0,  "weight": 0.15 },
    "model":    { "value": null, "weight": 0.10 },
    "text":     { "value": 0.42, "weight": 0.15 }
  },
  "completeness": 0.90,
  "flags": [],
  "baseline_score": 0.710,
  "jev": {
    "mode": "ACTIVE",
    "status": "SUCCEEDED",
    "used": true,
    "model": "jev-1.13.0",
    "question_set": "match-q-1",
    "input_hash": "sha256:...",
    "answers": {
      "same_object": 0.83,
      "description_consistent": 0.74,
      "brand_model_consistent": 0.88
    },
    "signal": 0.820,
    "weight": 0.30,
    "latency_ms": 180
  },
  "final_score": 0.743
}
```
Jev OFF: `"jev": { "mode": "OFF" }` and `final_score = baseline_score`.
Jev SHADOW: answers recorded, `"used": false`, `final_score = baseline_score`.

## 3. Fields
| Field | Meaning |
|---|---|
| `model_version` | Version of weights, gates and parameters (feature-spec.md). Bumped on any change |
| `gates` | Result of each gate. A stored match always has all `true` |
| `features.*.value` | In [0, 1] or `null`; `weight` is the weight used |
| `completeness` | Share of total weight with a known value |
| `flags` | `BRAND_CONFLICT`, `MODEL_CONFLICT`, `COLOR_CONFLICT` |
| `baseline_score` | Σ weight × value, 3 decimals |
| `jev.status` | `SUCCEEDED`, `FAILED`, `TIMEOUT`, `SKIPPED` (flag off, below `min_baseline`, no consent, or over `candidate_limit`) |
| `jev.used` | `true` only in ACTIVE with `SUCCEEDED` |
| `jev.input_hash` | Hash of the redacted state + `question_set` + Jev model. Same hash ⇒ reuse stored answers |
| `jev.signal` | `0.5 × same_object + 0.25 × description_consistent + 0.25 × brand_model_consistent` |
| `final_score` | `baseline_score` unless `jev.used`; then `(1 − weight) × baseline + weight × signal` |

## 4. `decision` block (claims snapshot only)
Written when the claim is created (decision-policy.md):
```json
"decision": {
  "outcome": "PENDING_REVIEW",
  "reasons": ["BELOW_AUTO_READY"],
  "thresholds_version": "thresholds-1.0.0"
}
```
Reason codes: `BELOW_CLAIM_MIN`, `AMBIGUOUS`, `VALUABLE_ITEM`, `CATEGORY_REQUIRES_APPROVAL`, `CATEGORY_NO_AUTO_READY`, `BELOW_AUTO_READY`, `CONFLICT_FLAG`, `JEV_VETO`, `AUTO_READY`.

## 5. Rules
- Contains numbers, flags and ids only. No free text, no private identifiers, no student data (logging.md, security.md).
- Written once per score computation. A rematch writes a new breakdown with a new `input_hash` if inputs changed.
- Validated against this schema on write (Pydantic model in the backend).
- Readable by admins only (permissions.md). Students never receive it; they get a label (decision-policy.md §6).

## 6. Admin display (`ScoreBreakdown` component)
- One row per feature: label, value, weight, contribution. `null` shown as "not provided".
- Flags as amber chips; `jev` block shown only when `used = true`, with the three answers in plain language.
- Final score and the decision reasons from the claim snapshot.
- Statuses mapped to plain labels, never raw enums (components.md).

## 7. Retention
Follows `potential_matches` / `claims` retention (database.md §5). Because the JSON holds no free text, it needs no separate purge.
