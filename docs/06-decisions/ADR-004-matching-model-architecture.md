# ADR-004 — Matching Model Architecture

Status: Proposed · Date: 2026-10-05 · Owner: matching model workstream
Cross-refs: FR-04 · FR-06 · system-architecture.md (invariants 5, 7) · integration.md · 05-matching-model/

## Context
- FR-04 requires deterministic two-way matching with a score breakdown; FR-06 needs a score to choose auto-ready, admin review or needs-proof.
- The core flow must work with every AI feature off (vision principle 5, invariant 7).
- No labelled data exists at pilot start (one desk, no history), so a trained model cannot be fitted yet.
- A wrongly auto-readied claim is the costly error: pickup checks the claimant's ID, not item ownership.
- Fuzzy parts (brand/model synonyms, free-text wording) are brittle in hand-written rules.
- TypeSafe's Jev returns typed probabilities (Noul / Choice / Score) instead of text, has no hallucinated strings, and is fast. It is early access, third-party, US-hosted, and its calibration claims are the vendor's own.

## Decision
Use a hybrid:
1. A deterministic baseline scorer (gates, six weighted features, conflict flags) is the model of record and works alone.
2. Jev is an optional, feature-flagged signal (`JEV_MATCHING`) with three yes/no object-similarity questions. Modes: OFF (default), SHADOW, ACTIVE.
3. Final decisions are made by deterministic policy code. Jev can move a pair into admin review or veto auto-ready; it can never produce auto-ready.
4. Score semantics: a score in [0, 1], rounded to 3 decimals, with `model_version`, gates, features, flags and Jev answers stored in `score_breakdown_json`. Heuristic in v1, calibrated to a probability once pilot labels exist.
5. Thresholds live in `PolicyService.getMatchThresholds()`. The effective auto-ready threshold is the stricter of the global value and the category's `minimum_auto_ready_score`.
6. Jev moves to ACTIVE only after AU approval and the shadow-mode acceptance criteria in probability-research.md §5.

## Alternatives considered
- **Rules only, no Jev.** Simplest and fully deterministic, but weak on synonyms and wording. Kept as the baseline and as the fallback.
- **Jev only.** Not deterministic or replayable alone, no AI-off path, and breaks invariant 7.
- **General LLM (Gemini) for judgments.** Returns strings that need parsing and validation, slower and costlier per call, and already scoped to prefill with mandatory fallback.
- **Trained ML model (logistic regression / gradient boosting).** No labels at pilot start. Revisit once outcomes accumulate; the stored breakdown is already a usable feature set.
- **Embedding similarity.** Needs model hosting or another third party and is harder to explain to admins.

## Consequences
Positive
- Core flow and audit trail unchanged if Jev is unavailable, unapproved, or removed.
- Every decision is explainable and replayable from stored data.
- Evidence about Jev is gathered safely in SHADOW before it affects any claim.

Negative / follow-ups
- Second component to maintain (config, question wording, caches).
- New third party: needs AU IT/security approval, a data-processing review, and a consent purpose for report text (integration.md).
- Vendor claims unverified; Jev stays OFF if acceptance criteria are not met.
- Docs to update: prd.md, integration.md, system-architecture.md, scope.md, error-handling.md, logging.md, testing-strategy.md, components.md (matching-model-overview.md §7). Optional migration: score range CHECK constraint.
- Fine-tuning of Jev is not assumed; if TypeSafe offers it later, a new ADR covers it.

## Revisit when
- Jev leaves early access, changes pricing/terms, or AU rejects the vendor (remove ACTIVE path).
- Pilot yields enough labelled outcomes to fit weights or a trained model.
- False auto-ready rate exceeds the agreed bound (tighten thresholds or disable auto-ready).
- Multi-location rollout (location feature and thresholds need review).
