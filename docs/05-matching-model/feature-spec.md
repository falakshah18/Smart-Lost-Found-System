# Feature Spec

Owner: matching model workstream · Status: proposed baseline
Cross-refs: matching-model-overview.md · score-breakdown-schema.md · database.md

## 1. Rules
- Features read only fields that already exist in `found_items` and `lost_reports`.
- Every feature returns a value in [0, 1] or `null` (unknown).
- `null` counts as 0 in the score. Missing evidence never raises a score.
- Weights sum to 1.0. Weights and parameters are config under `model_version`, not code.
- Scores are computed in decimal arithmetic and rounded half-up to 3 decimals before any threshold comparison.

## 2. Normalization
Applied to text fields before comparison:
- lowercase, trim, collapse whitespace, strip punctuation
- `color_primary`, `color_secondary`: map to one controlled color list
- `brand`, `model`: map through an alias table (config), e.g. abbreviations and common misspellings
- description/title tokens: split on whitespace, drop stopwords and tokens shorter than 2 characters

The same controlled lists must be used by the guard intake screen, the lost-report form and LLM `validateOutput`, otherwise exact-match features fail silently.

## 3. Gates
A pair that fails any gate is never scored and no `potential_matches` row is created.

| Gate | Rule |
|---|---|
| category | `found_items.category_id = lost_reports.category_id` |
| found after loss start | `found_at >= lost_at_from` (skipped if `lost_at_from` is null) |
| item available | `found_items.status = AVAILABLE` |
| report open | `lost_reports.status IN (OPEN, MATCH_SUGGESTED)` |

Gates map to the existing lookup indexes `idx_found_items_lookup (status, category_id, location_id)` and `idx_lost_reports_lookup`.

## 4. Features
| Feature | Weight | lost_reports | found_items | Rule |
|---|---|---|---|---|
| location | 0.25 | `location_id` | `location_id` | same id = 1.0, else 0.0 |
| time | 0.20 | `lost_at_from`, `lost_at_to` | `found_at` | `d = max(0, days(found_at − lost_at_to))`; value = `max(0, 1 − d / 7)`. Inside the window = 1.0. If `lost_at_to` is null use `lost_at_from`. Null if both missing |
| color | 0.15 | `color_primary`, `color_secondary` | same | primary = primary → 1.0; primary↔secondary or secondary = secondary → 0.5; else 0.0. Null if either primary is missing |
| brand | 0.15 | `brand` | `brand` | equal → 1.0; both present and different → 0.0; null if either missing |
| model | 0.10 | `model` | `model` | equal → 1.0; one token set contained in the other → 0.6; both present otherwise → 0.0; null if either missing |
| text | 0.15 | `description` | `title` + `guard_notes` | Jaccard overlap of normalized token sets. Null if either side is empty |

`time_decay_days = 7` is a parameter. Location is exact-match in v1 (pilot = one desk); a proximity map is a later-phase tuning item.

## 5. Baseline score
```
baseline_score = round3( Σ weight_i × value_i )     # null → 0
completeness   = Σ weight_i for features with value ≠ null
```
`completeness` is informational (admin view, analysis). It is not a decision input.

## 6. Conflict flags
Set when both sides have a value and they disagree. Flags do not change the score; they block auto-ready (decision-policy.md).
- `BRAND_CONFLICT` — brand both present, different
- `MODEL_CONFLICT` — model both present, token sets disjoint
- `COLOR_CONFLICT` — both primaries present and no primary/secondary overlap

## 7. Deliberately not used
- `private_identifiers_encrypted` / `private_identifiers_hash` — guards never see them; admins only via decrypt authorization. Not a score input in v1. If ever enabled: exact hash match, as a separate flag, never sent to Jev.
- Student ID, claim answers, user identity.
- pHash — lost reports carry no photo, so there is nothing to compare. pHash stays with CVService (duplicate-intake detection).
- CV dominant colors — not read directly. If the guard leaves `color_primary` blank, whether CV may fill it is an open item (overview §8).
- `is_valuable`, `legal_hold`, `storage_location`, `label_code` — policy or logistics, not similarity.
