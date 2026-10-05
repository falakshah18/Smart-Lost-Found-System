# Matching Test Cases

Owner: matching model workstream · Status: proposed baseline
Cross-refs: testing-strategy.md · feature-spec.md · decision-policy.md · score-breakdown-schema.md · jev-integration.md

Extends the Unit and Integration sections of testing-strategy.md ("Matching scoring, policy rules").

## 1. Unit — gates
- Category mismatch → no candidate.
- `found_at < lost_at_from` → no candidate; `found_at = lost_at_from` → passes; `lost_at_from` null → gate skipped.
- Item not `AVAILABLE` → no candidate. Report not `OPEN`/`MATCH_SUGGESTED` → no candidate.

## 2. Unit — features
- Each feature: equal, partial, conflict, and missing input (table-driven).
- Time: inside window = 1.0; 7+ days after window = 0.0; `lost_at_to` null uses `lost_at_from`.
- Color: primary/secondary cross-match = 0.5; both primaries present with no overlap sets `COLOR_CONFLICT`.
- Model: token subset = 0.6; disjoint sets set `MODEL_CONFLICT`.
- Normalization: case, punctuation, alias table, controlled color list.
- Missing feature contributes 0; `completeness` equals the weight share of known features.
- Weights sum to 1.0 for every `model_version`.

## 3. Unit — composition
- OFF and SHADOW: `final_score = baseline_score`.
- ACTIVE with answers: `(1 − w) × baseline + w × signal`.
- ACTIVE with Jev failed / timeout / skipped: `final_score = baseline_score`.
- Rounding: sums are rounded half-up to 3 decimals before comparison; a value like 0.8499999 must compare as 0.850.

## 4. Unit — policy
- Rule order in decision-policy.md §3 (first match wins), one test per reason code.
- Category floor: `auto_ready_effective = max(global, category.minimum_auto_ready_score)`.
- Ambiguity at the margin: difference exactly 0.10 is ambiguous; 0.101 is not.
- Student label boundaries at `suggest_min`, `claim_min`, `auto_ready_min`.

## 5. Golden cases (Jev OFF unless stated)
| ID | Scenario | Score | Expected |
|---|---|---|---|
| G1 | Five features match fully; text is partial (location 1, time 1, color 1, brand 1, model 1, text 0.5) | 0.925 | READY_FOR_PICKUP (category allows, not valuable) |
| G2 | location 1, time 0.86, color 0.5, brand 1, model unknown, text 0.42 | 0.710 | PENDING_REVIEW (`BELOW_AUTO_READY`) |
| G3 | Generic "black wallet": location 1, time 1, color 1, rest unknown | 0.600 | NEEDS_PROOF (`BELOW_CLAIM_MIN`); match card shown |
| G4 | location 1, time 1, color 1, brand conflict, model 1, text 1 | 0.850 | PENDING_REVIEW (`CONFLICT_FLAG`), boundary at auto-ready score |
| G5 | Found before loss window start | — | No candidate |
| G6 | Different category | — | No candidate |
| G7 | Score 0.95 but `is_valuable` | 0.950 | PENDING_REVIEW (`VALUABLE_ITEM`) |
| G8 | Two found items for one report, scores 0.88 and 0.82 | — | NEEDS_PROOF for both (`AMBIGUOUS`) |
| G9 | Category `minimum_auto_ready_score` 0.90, score 0.88 | 0.880 | PENDING_REVIEW (`BELOW_AUTO_READY`) |
| G10 | ACTIVE: baseline 0.90, Jev answers 0.30 / 0.40 / 0.50 | final 0.743 | PENDING_REVIEW (`JEV_VETO`) |
| G11 | ACTIVE: baseline 0.60, Jev all 1.0 | final 0.720 | PENDING_REVIEW, never READY_FOR_PICKUP |
| G12 | ACTIVE: Jev times out on a G1 pair | 0.925 | Same as OFF: READY_FOR_PICKUP |

G10: signal = 0.5×0.30 + 0.25×0.40 + 0.25×0.50 = 0.375; final = 0.7×0.90 + 0.3×0.375 = 0.743.
G11: final = 0.7×0.60 + 0.3×1.0 = 0.720.

## 6. Property tests
- Determinism: same inputs and `model_version` → identical breakdown (byte-equal JSON).
- Replay: recomputing from a stored breakdown (no network) reproduces `final_score` and the decision.
- Jev cannot create auto-ready: for any Jev answers, if `baseline_score < auto_ready_effective` the outcome is never READY_FOR_PICKUP.
- Jev veto monotonic: lowering `same_object` never moves an outcome from review to auto-ready.
- Unknown never helps: replacing a feature value with `null` never raises `baseline_score`.
- Idempotent rerun: same `input_hash` reuses stored answers and makes no Jev call.

## 7. Integration (API + DB)
- Breakdown validates against score-breakdown-schema.md on write; invalid JSON rejected.
- `potential_matches.score` equals `final_score`; the claim snapshot copies score, breakdown and adds `decision`.
- Idempotent claim creation returns the previous decision (existing test) and does not re-score.
- Second active claim for an item is blocked by `uniq_active_claim_per_found_item`.
- Rematch after a guard tag edit updates scores and does not cancel an existing active claim.
- Score range check (if the migration is adopted) rejects values outside [0, 1].

## 8. Privacy and security
- Jev state builder emits only the allow-listed fields (snapshot test); a lost report containing a phone number or student ID in `description` arrives redacted.
- No `private_identifiers_*`, `guard_notes`, location, times or ids in any Jev request.
- No consent → `jev.status = SKIPPED`, no request made.
- Logs contain no request state or free text.
- Students cannot read `score_breakdown_json` (IDOR test); admins can.

## 9. Operational
- Jev down, slow (> `timeout_ms`), returning malformed data: matching completes with baseline; alert fires after the failure-rate threshold.
- Flag `JEV_MATCHING` off in a running system → next job runs OFF with no restart.
- Load: a matching job with the maximum 10 candidates stays inside the job time budget with Jev timing out.

