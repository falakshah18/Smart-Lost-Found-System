# Matching Model — AU Smart Lost & Found

Owner: matching model workstream · Status: proposed baseline · Last updated: 2026
Cross-refs: feature-spec.md · score-breakdown-schema.md · decision-policy.md · jev-integration.md · probability-research.md · matching-test-cases.md · ADR-004

## 1. Purpose

This document describes a proposed matching design for FR-04 (deterministic lost/found matching with a score breakdown) and FR-06 (using match evidence in claim policy). Confirm each service, field, status and invariant against the current repository before treating it as implemented.

For each eligible lost-report and found-item pair, the proposed model produces:
- a heuristic similarity score in [0, 1]
- an explainable breakdown stored as `score_breakdown_json`

The score does not itself decide a claim outcome. `decision-policy.md` defines how policy code uses it.

## 2. Design

A deterministic baseline scorer is the model of record and must work with AI features disabled. Jev is an optional, feature-flagged signal for text and object-description ambiguity. It defaults to OFF. Final outcomes remain deterministic policy decisions. Jev may contribute to the score or veto auto-ready only in ACTIVE mode after the required approvals and evaluation; it can never make a pair auto-ready by itself.

In v1, scores are heuristic similarity values, not probabilities. See `probability-research.md` for the evidence required before any calibration claim.

## 3. Proposed service responsibilities

These names are proposed cross-references from the design notes. Verify that the services and methods exist before implementation.

| Service | Proposed role |
|---|---|
| MatchingService | Score eligible pairs and rerun matching after relevant edits |
| PolicyService | Return versioned thresholds and apply claim decision policy |
| FeatureFlagService | Provide the `JEV_MATCHING` kill switch |
| CVService | Handle image features and duplicate intake; not a v1 score input |
| LLMPrivacyService | Redact sensitive text before any approved external text processing |
| Job service | Run matching asynchronously and handle retries |
| Audit/Monitoring services | Record policy changes and monitor failures, if supported by the repository |

## 4. Pipeline

1. **Trigger** — proposed triggers include found-item intake, lost-report submission, relevant edits and expiry handling. Confirm exact triggers in the existing flow.
2. **Gates** — apply hard eligibility filters; a failed pair is not scored.
3. **Features** — compute six deterministic features in [0, 1] or `null` for unknown.
4. **Baseline** — weighted sum; unknown features contribute zero.
5. **Jev (optional)** — call only when enabled, approved, consent requirements are met and configured eligibility limits pass.
6. **Compose** — OFF and SHADOW use the baseline as `final_score`; ACTIVE may blend a successful Jev signal according to the documented formula.
7. **Persist** — store the score and its breakdown.
8. **Policy** — apply decision policy at the documented workflow point; do not assume claim-time evaluation until verified against the repository.

## 5. Guarantees required by this design

1. The same inputs, `model_version` and stored Jev answers produce the same score.
2. The baseline remains usable with AI disabled or Jev unavailable.
3. Jev receives only the approved allow-listed state. No private identifiers, student data, photos, pickup codes, claim answers, location or time fields are sent. Free text must pass the documented redaction and consent checks.
4. The breakdown is sufficient to explain and replay a score without a new Jev request.
5. Cached Jev answers are reused only when the input hash, question set and model version match.
6. Jev alone can never produce an auto-ready outcome.

These are design requirements, not claims that the current code already meets them.

## 6. Score semantics

The v1 score is a heuristic similarity value in [0, 1]. It must not be described to users or reported as `P(match)` or confidence unless a calibration method has been evaluated on appropriate held-out, labeled data. Thresholds in `decision-policy.md` are initial policy settings, not empirically validated probabilities.

## 7. Repository alignment before merge

Before merging these proposed docs, check and update relevant existing documentation and code references:

| Area | Alignment to verify |
|---|---|
| Requirements | FR-04 and FR-06 wording and links |
| Database | Actual score columns, JSON fields, constraints and status values |
| Integration | Third-party processor, feature flag and deployment configuration |
| Architecture/scope | AI defaults and any existing invariants |
| Privacy/security | Consent purpose, redaction behavior and data residency approval |
| Error handling/logging | Fallback behavior and fields safe to log |
| Testing | Existing test organization and implemented behavior |
| UI/components | Existing labels and who may see score breakdowns |

## 8. Open items

- Resolve any conflict among existing requirements about whether other AI features are enabled in the pilot.
- Obtain the required institutional IT/security and privacy approvals before sending report text to TypeSafe AI.
- Confirm the consent purpose, redaction guarantees, retention terms and service data residency.
- Confirm whether Jev can be configured only through state and questions; do not assume fine-tuning.
- Establish shared controlled vocabularies for colors and brands, if those fields are used by intake and matching.
- Decide whether CV-derived colors may populate the found-item color field; this is not a v1 score input unless explicitly adopted.
- Verify referenced ADR-001 to ADR-003 exist before relying on their decisions.
