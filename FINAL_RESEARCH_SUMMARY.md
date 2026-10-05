# BDG2 Final Research Summary

## 1. Research Problem
Evaluating the stability and faithfulness of AI electricity-demand forecasts under distribution shifts (temporal, building, modality). The core hypothesis is that high predictive accuracy under familiar conditions does not guarantee robust generalization or reliable explanations under distribution shift.

## 2. Experimental Design
- **Train**: 384k rows, 32 buildings
- **Validation**: 82k rows
- **Test**: Temporal (80k), Unseen Building (152k), Joint Shift (22k).
- **Models**: Linear Regression, XGBoost (Default & Optimized), LSTM, Transformer.

## 3. Findings

### RQ1 (Weather Modality)
Weather provides marginal improvement over pure demand+calendar on in-distribution data, but does not guarantee OOD robustness.

### RQ2 (Optimization)
Optuna tuning improves validation and temporal performance, but does *not* solve unseen-building generalization degradation.

### XGBoost Unseen-Building Failure (OOD Diagnosis)
XGBoost exhibits extreme degradation on unseen buildings (e.g., RMSE > 99). Building-level diagnostics confirm this is a genuine algorithmic failure, not a data bug. The failure is primarily concentrated in specific outlier buildings (e.g., `Bear_education_Chad`) whose typical electricity demand falls into the extreme right tail (< 1%) of the training distribution. Tree-based models fail to extrapolate or capture variance in these sparse terminal leaves, whereas Linear Regression and LSTM generalize effectively by applying learned continuous weights.

### RQ4-RQ7 (Explanation Stability)
SHAP global feature rankings exhibit strong stability across random seeds (Spearman ρ ≥ 0.90), establishing that model attributions are reliable with respect to initialization.

### RQ8 (Faithfulness - Repaired)
Explanation faithfulness was rigorously tested using top-k feature perturbation with proper alignment. Perturbing Top-k SHAP features causes drastically higher RMSE degradation than perturbing Random-k or Bottom-k features.
- K=10 Top Degradation: ~255 RMSE
- K=10 Random Degradation: ~113 RMSE
This provides statistical confirmation that SHAP rankings are faithful to the model's actual predictive reliance.

### Attention Analysis
Given the simplified tabular-lag structure of the Transformer, explicit multimodal attention analysis has been removed to avoid unsupported interpretability claims.

## 4. Final Recommendation
Predictive accuracy alone is insufficient to characterize trustworthy building-energy forecasting. Models that perform well under familiar conditions (XGBoost) can exhibit catastrophic vulnerability to target-scale shifts on unseen buildings. Linear models and sequence models (LSTM) demonstrated superior structural robustness. 
