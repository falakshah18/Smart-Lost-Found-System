# Jev Integration

Owner: matching model workstream · Status: proposed, pending AU approval (ADR-004)
Cross-refs: matching-model-overview.md · score-breakdown-schema.md · integration.md · security.md · logging.md · error-handling.md

## 1. What Jev is
TypeSafe AI's first "System One" model. The caller sends a **state** and typed **questions**; Jev returns typed answers with probabilities. It does not generate text.

| Primitive | Asks | Returns |
|---|---|---|
| Noul | Is this true? | `noul` (0–1). No confidence field |
| Choice | Which option? | `choice`, `probabilities`, `confidence` |
| Score | Which level? (max 10 levels) | `score`, `probabilities`, `confidence` |

- Questions are evaluated in parallel and independently against one state.
- Public docs recommend atomic questions combined in code, which is the shape of our score breakdown.
- Model referenced in public docs at time of writing: `jev-1.13.0`. Early access.
- Speed, cost and calibration figures are TypeSafe's own claims. We test them ourselves (probability-research.md §5).
- Nothing found in public docs about fine-tuning. What we control: the questions, the state, the blend weight, the thresholds.

## 2. Role in matching
Jev answers three yes/no questions about whether two descriptions refer to the same object. It does not see places or times, and it does not decide anything. Its output enters `final_score` through a small weight and a veto (decision-policy.md §5).

## 3. Modes
| Mode | Behavior |
|---|---|
| OFF (default) | No Jev call. `final_score = baseline_score` |
| SHADOW | Jev called, answers stored, `used = false`. Used to collect evidence |
| ACTIVE | Answers blended into `final_score`; veto applies |

Controls: feature flag `JEV_MATCHING` (kill switch via FeatureFlagService) and `getMatchThresholds().jev.mode`. Flag off ⇒ OFF regardless of mode. ACTIVE requires ADR-004 accepted, AU approval, and shadow-mode results meeting probability-research.md §5.

## 4. State sent to Jev (allow-list)
Per pair, only these fields:

| From | Fields |
|---|---|
| lost report | category name, `color_primary`, `color_secondary`, `brand`, `model`, `description` (redacted) |
| found item | category name, `color_primary`, `color_secondary`, `brand`, `model`, `title` |

Rules:
- `description` passes through LLMPrivacyService (`redactSensitiveText`, `removePrivateIdentifiers`) first.
- Items and reports are referenced by request-scoped labels ("report", "item"), never database ids.
- Never sent: `private_identifiers_*`, student ID, name/email/user id, `guard_notes`, location, times, photos or URLs, pickup codes, claim answers, `label_code`.
- A lost report's text is sent only if `ConsentService.hasValidConsent(user, purpose)` passes. Purpose name (reuse `LLM_ASSISTANCE` or add one) is an open item. Without consent the pair is scored baseline-only (`jev.status = SKIPPED`).
- The state is built by one function with a fixed allow-list; adding a field requires a doc change and review.

## 5. Question set `match-q-1`
| Key | Type | Question | Used for |
|---|---|---|---|
| `same_object` | Noul | Do the lost report and the found item most likely describe the same physical object? | signal (0.5), veto |
| `description_consistent` | Noul | Is the lost report description consistent with the found item's title, with no contradicting detail? | signal (0.25) |
| `brand_model_consistent` | Noul | Are the brand and model on both sides consistent, allowing for abbreviations, synonyms and typos? | signal (0.25) |

Later, not in v1: a Choice question that picks among the top candidates for one report (uses `confidence`). Choice/Score `confidence` can be gated; Noul has none, which is why Jev stays small-weight and veto-only.

Question wording is versioned (`question_set`). Changing wording = new set name = new `model_version` and a fresh shadow run.

## 6. Flow
1. Baseline score computed and gates applied (always).
2. Jev is called only if: effective mode ≠ OFF, consent OK, `baseline_score >= jev.min_baseline` (0.30), and the pair is within the top `jev.candidate_limit` (10) for its report/item.
3. `input_hash` = hash of (redacted state, `question_set`, Jev model). If a stored breakdown has the same hash, reuse its answers; no call.
4. One request per pair with all three questions. Timeout `jev.timeout_ms` (2000). Retry with Procrastinate `retryWithBackoff`.
5. All three answers present ⇒ `signal` computed. Any missing ⇒ treated as failed.
6. Result written into `score_breakdown_json` (`jev` block).

Matching runs in the background job queue, so Jev latency never blocks intake or report submit.

## 7. Failure handling
| Case | Behavior |
|---|---|
| Timeout / 5xx / network | `jev.status = TIMEOUT/FAILED`; score = baseline; flow continues |
| Invalid or incomplete response | treated as `FAILED` |
| Rate limit / cost cap hit | `SKIPPED`; baseline only |
| Consent missing | `SKIPPED`; baseline only |
| Repeated failures | `MonitoringService` raises a SYSTEM_ALERT (failure-rate and latency spikes) |

A stored score is not revised later when Jev recovers; it changes only on a rematch.

## 8. Cost and rate limits
- Volume is small: one request per candidate pair, at most 10 per job.
- TypeSafe advertises very low input pricing; verify against the real bill during shadow mode.
- Per-user and global rate limits and a monthly cost cap are enforced by RateLimitService and config, as for the LLM assistant (integration.md).

## 9. Logging and privacy
- Log: pair job id, Jev status, latency, question set, mode.
- Never log: request state, free text, student identifiers (logging.md).
- Answers (numbers only) are stored in the breakdown JSON and follow the retention of `potential_matches`.
- API key lives in the secret manager, never in code or the frontend.

## 10. Approvals needed (integration.md)
- AU IT/security: TypeSafe as a third-party processor, data residency (service currently US West Coast), retention terms, DPA.
- Privacy: consent purpose for report text.
- Any alternate access route (for example a model router) is a separate third party needing its own approval.
- Exit plan: flip the flag to OFF. Nothing else depends on Jev.
