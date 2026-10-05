# Probability Research

Owner: matching model workstream · Status: proposed plan
Cross-refs: matching-model-overview.md · decision-policy.md · jev-integration.md · testing-strategy.md · user-flows.md (F4, F6)

## 1. Questions this doc answers
1. What does a score mean, and when can it be read as a probability?
2. How are thresholds chosen, and what error matters most?
3. How do we measure and improve the model with almost no data at pilot start?
4. How do we decide whether Jev earns its place?

## 2. What a score means
- v1: heuristic similarity in [0, 1] (weights are expert-set, not fitted). Not a probability.
- Target: calibrated `p = P(claim is the true owner | score, flags)`, obtained by mapping score → p once labelled outcomes exist (§4).
- Until then thresholds are compared to the raw score and described as "initial".

## 3. Which error matters
Pickup verification checks that the physical ID belongs to the claimant. It does not check that the item belongs to the claimant (F6). So:

| Error | Effect | Cost |
|---|---|---|
| False auto-ready | Wrong person can collect the item | High (privacy, dispute, trust) |
| Unnecessary review | Admin spends a few minutes | Low |
| Missed match | Owner never sees their item | Medium (lower return rate) |

Auto-ready is chosen when expected loss of auto-ready is below expected loss of review:
```
(1 − p) × C_wrong  <  C_review      ⇒      p  >  1 − C_review / C_wrong
```
Illustration only: if a wrong handover costs 20 reviews' worth of effort, auto-ready needs `p > 0.95`. Real `C_wrong / C_review` is a policy choice for the admin, recorded in an ADR when set.
Suggestion and claim thresholds use the same logic with lower costs for a missed match.

## 4. Labels from the system itself
Potential outcome labels may be derived from workflow records, but they are not complete ground truth and require a documented labeling and review process. Do not assume that every schema outcome is a reliable ownership label.

| Label | Source |
|---|---|
| Candidate positive | claim reaches `RETURNED` with no dispute (review a sample; return alone does not prove ownership) |
| Candidate negative | student marks "not mine" (`potential_matches` → `REJECTED`), admin rejects the claim, or a dispute is upheld against the claimant (record the reason and review label quality) |
| Unlabelled | still open, expired without action, or insufficient evidence to assign a label |

Caveats: only pairs that were shown can be labelled (selection bias), and a dishonest claimant can tailor a report to the item. The score measures consistency, not ownership. Private-identifier proof stays a separate, later control.

## 5. Evaluation plan
### Before the pilot (synthetic)
- Generate lost reports from existing found items by perturbing fields: drop fields, swap color synonyms, alter brand/model spelling, shift the time window, add a distractor item in the same category.
- Positives = the source item; negatives = other items in the category.
- Use it to check logic and weights. Synthetic metrics are optimistic and are never used to claim real accuracy.

### During the pilot (shadow first)
- Run baseline and Jev in SHADOW. Record both; act on baseline only.
- Collect labels per §4. Review weekly.

### Metrics
| Metric | Why |
|---|---|
| False auto-ready rate (wrong returns / auto-ready claims) | primary safety metric |
| Recall of true matches at `suggest_min` | return rate |
| NEEDS_PROOF and PENDING_REVIEW rates | admin load |
| Brier score and reliability curve of score → outcome | calibration |
| Top-1 accuracy per report | ranking quality |
| For Jev: same metrics for `signal` alone, agreement with baseline, and gain over baseline | does Jev add anything |

### Sample size
With zero false auto-readies in `n` reviewed auto-ready claims, the 95% upper bound on the false rate is about `3 / n` (rule of three): `n = 60` → under 5%, `n = 300` → under 1%. A pilot at one desk may not reach 300; report the bound, not just the point estimate.

### Jev acceptance (to move SHADOW → ACTIVE)
- Brier score and recall at the same false-auto-ready rate are not worse than baseline, and better on a meaningful subset (synonym/brand cases).
- Confidence behaves: answers near 0/1 are right more often than answers near 0.5 (reliability curve).
- Failure rate and p95 latency inside the budget in jev-integration.md.
- AU approval and ADR-004 accepted.
If not met, Jev stays OFF; the baseline is a complete model.

## 6. Calibration and tuning
- With few labels: report counts and uncertainty by score band; avoid fitting or presenting a probability mapping when bins are sparse. If a Beta-Binomial estimate is used, state its prior and assumptions. Use confidence or credible bounds for policy decisions, and validate on data not used to choose the mapping.
- With adequate, representative labels: evaluate isotonic regression or logistic (Platt) calibration against a held-out time-based set and a simple baseline. A fitted score-to-outcome mapping is not proof of ownership and may drift; monitor and validate it before use.
- Weights: first adjust thresholds, then single-feature weights one at a time, each as a new `model_version` re-run in shadow. No tuning on the same data used to report accuracy (hold out the latest weeks).
- Every threshold or weight change: new `version`, `AuditService.record`, and a re-run of matching-test-cases.md.

## 7. Monitoring (MonitoringService)
- Score distribution per week (shift = drift or new item mix).
- NEEDS_PROOF / PENDING_REVIEW / auto-ready shares.
- Approve vs reject ratio in admin review; any false auto-ready is an incident.
- Jev failure rate, latency, and agreement with baseline.
Alerts create SYSTEM_ALERTS, as for other jobs.

## 8. Deliverables
1. Synthetic data generator and evaluation script (repo `tools/`, not docs).
2. Weekly evaluation report during the pilot (metrics in §5).
3. Calibrated mapping and proposed thresholds with intervals.
4. Jev go/no-go note against §5 acceptance.

