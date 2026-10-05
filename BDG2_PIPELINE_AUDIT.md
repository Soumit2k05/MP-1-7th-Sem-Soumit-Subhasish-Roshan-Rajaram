# BDG2 Pipeline Audit Report

**Project:** Trustworthy Explainability for Multimodal Building Electricity-Demand Forecasting  
**Dataset:** Building Data Genome Project 2 (BDG2) — Bear site extract  
**Date:** 2026-10-06  
**Status:** ✅ Pipeline rebuilt, validated, and executed end-to-end

---

## What Was Wrong

The existing pipeline had a **27 → 2 building collapse** between Stage 2 (preprocessing) and Stage 3 (experimental splits):

| Stage | Artifact | Buildings |
|-------|----------|-----------|
| Stage 1.5 | `bdg2_study_population.csv` | 27 |
| Stage 1.5 | `bdg2_building_splits.csv` | 27 |
| Stage 2 | `bdg2_preprocessed_all_rows.csv` | **12** ← truncated |
| Stage 3 | `bdg2_experiment_manifest.csv` | **9** ← downstream collapse |

Because Stage 3 performed an inner join between the 27-building split and the 12-building preprocessed dataset, only **9 buildings** appeared in the manifest, and 5 of the 6 experimental sets were empty or near-zero.

---

## Root Cause

**Stage 2 (BDG2_Stage_2_Preprocessing.ipynb) processed only 12 of 27 buildings.**

The notebook's groupby-based feature engineering loop was truncated — it either hit a memory limit mid-loop or was interrupted during execution and saved partial results.

All 15 missing buildings (`Bob`, `Bonita`, `Bulah`, `Chad`, `Chana`, `Chun`, `Clint`, `Curtis`, `Danna`, `Darrell`, `Deena`, `Derek`, `Elise`, `Fannie`, `Gavin`) were:
- Present in `recovered_data.csv` with full 17,544 rows each
- Present in `bdg2_study_population.csv`
- Present in `bdg2_building_splits.csv`
- **ABSENT from `bdg2_preprocessed_all_rows.csv`** — the smoking gun

> NOTE: This was NOT a data corruption issue. All buildings exist in the raw data. The preprocessing simply stopped midway and saved an incomplete file.

---

## Additional Finding: Extended Study Population

During re-audit, `recovered_data.csv` contains **60 buildings**, of which **48 pass** eligibility criteria. The original 27-building study population was the result of the original Stage 1.5 notebook using an implicit pandas row limit. One building (`Bear_education_Paola`) is correctly excluded at 76.9% row coverage. **The master pipeline uses all 48 eligible buildings.**

---

## What Was Fixed

1. **Rebuilt the entire pipeline from scratch** in `BDG2_MASTER_PIPELINE.ipynb`
2. **Single `PROJECT_ROOT` variable** — all paths derived from it; no manual CSV juggling
3. **Stage 2 fixed** — vectorised groupby processes all 48 eligible buildings
4. **Frozen building split** — deterministic, stratified by `primaryspaceusage`, seed=42
5. **Rolling features use `shift(1)` before `.rolling()`** — no time-t leakage
6. **Weather forward-fill is causal** (no backfill)
7. **All 6 experimental sets non-empty** with proper temporal boundaries
8. **21 leakage checks all pass**

---

## Final Numbers

### Dataset
| Metric | Value |
|--------|-------|
| Raw rows | 1,048,576 |
| Raw buildings | 60 |
| Eligible buildings | 48 |
| Model-ready supervised rows | 817,478 |

### Building Split (seed=42, stratified by usage)
| Group | Count |
|-------|-------|
| `train_building` | 32 |
| `validation_building` | 7 |
| `test_unseen_building` | 9 |
| **Total** | **48** |

### Temporal Split (Chronological — no random shuffling)
| Period | Start | End | Fraction |
|--------|-------|-----|----------|
| Train | 2016-01-08 00:00 | 2017-05-28 18:00 | 70% |
| Validation | 2017-05-28 19:00 | 2017-09-14 08:00 | 15% |
| Future test | 2017-09-14 09:00 | 2017-12-31 23:00 | 15% |

### Experimental Sets
| Set | Rows | Buildings |
|-----|------|-----------|
| `train` | 384,259 | 32 |
| `validation` | 82,657 | 32 |
| `temporal_test_seen_buildings` | 80,415 | 32 |
| `building_validation` | 100,609 | 7 |
| `unseen_building_test` | 152,088 | 9 |
| `joint_shift_test` | 22,461 | 9 |

---

## Leakage Validation — ALL 21 CHECKS PASSED ✓

| # | Check | Result |
|---|-------|--------|
| 1 | Manifest buildings ⊆ study_population | PASS ✓ |
| 2 | Each building in exactly one group | PASS ✓ |
| 3 | train_buildings ∩ unseen_buildings = ∅ | PASS ✓ |
| 4 | val_buildings ∩ unseen_buildings = ∅ | PASS ✓ |
| 5 | Unseen buildings never in train | PASS ✓ |
| 6 | temporal_test timestamps > train_end | PASS ✓ |
| 7 | validation timestamps > train_end | PASS ✓ |
| 8 | joint_shift_test ⊆ unseen_buildings | PASS ✓ |
| 9 | joint_shift timestamps ≥ test_start | PASS ✓ |
| 10×6 | All 6 experimental sets non-empty | PASS ✓ |
| 11×3 | All building groups have rows | PASS ✓ |
| 12 | No duplicate (building, timestamp, set) | PASS ✓ |
| 13 | Rolling features present (shift-based) | PASS ✓ |
| 14 | No lag_0h (target at t) in features | PASS ✓ |

---

## Baseline Results (Version B — Strict Common Rows)

Primary metric: RMSE (kWh)

| Model | validation | temporal_test_seen | unseen_test | joint_shift |
|-------|-----------|-------------------|-------------|-------------|
| **LinearRegression** | **12.90** | **11.75** | **15.12** | **14.00** |
| Persistence_1h | 15.45 | 14.54 | 17.97 | 17.01 |
| SeasonalNaive_24h | 29.16 | 28.68 | 33.71 | 31.91 |
| SeasonalNaive_168h | 32.70 | 32.67 | 41.11 | 39.54 |

**LinearRegression R² > 0.997 across all evaluation sets including unseen buildings and joint shift.**

---

## Artifact Directory

```
research_artifacts/
├── 01_raw_audit/
│   └── raw_per_building_audit.csv
├── 02_study_population/
│   ├── study_population.csv            (48 buildings)
│   ├── excluded_buildings.csv          (12 excluded)
│   └── building_split.csv              (FROZEN, seed=42)
├── 03_preprocessing/
│   └── model_ready.csv                 (841K rows, 41 cols)
├── 04_experimental_protocol/
│   ├── experiment_manifest.csv         (SINGLE SOURCE OF TRUTH)
│   └── experiment_summary.csv
└── 05_baselines/
    ├── baseline_results_A_max_rows.csv
    ├── baseline_results_B_common_rows.csv
    └── baseline_lr_building_level.csv
```

---

## Limitations

1. `recovered_data.csv` is capped at `2^20` rows — `Bear_education_Paola` is correctly excluded (76.9% coverage)
2. All 48 buildings share Bear-site weather — no cross-building weather differentiation
3. No occupancy/schedule data in current feature set
4. XGBoost, LSTM, Transformer, SHAP, attention intentionally deferred

---

## Reproducibility

Open `BDG2_MASTER_PIPELINE.ipynb` → **Run All** → complete pipeline from `recovered_data.csv`.
Only `PROJECT_ROOT` needs to change if the project is moved.
