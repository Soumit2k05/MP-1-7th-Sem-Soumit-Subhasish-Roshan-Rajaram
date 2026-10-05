# BDG2 Final Research Summary

## 1. Research Problem
Evaluating the stability and faithfulness of AI electricity-demand forecasts under distribution shifts (temporal, building, modality).

## 2. Experimental Design
- **Train**: 384k rows, 32 buildings
- **Validation**: 82k rows
- **Test**: Temporal (80k), Unseen Building (152k), Joint Shift (22k).

## 3. Findings
- **RQ1 (Weather Modality):** Weather provides marginal/moderate improvement over pure demand+calendar, as shown in ablation.
- **RQ2 (Optimization):** Optuna tuning improves generalization bounds.
- **RQ4-RQ7 (Explanation Stability):** Explanations show high stability (Spearman Rho > 0.90) across random seeds, but can shift under joint temporal+building variations.
- **RQ8 (Faithfulness):** Perturbing top-SHAP features leads to drastically higher RMSE degradation than bottom-SHAP features, proving high attribution correlates to actual model reliance.

## 4. Final Recommendation
Optimized XGBoost provides the most trustworthy balance of accuracy, explanation faithfulness, and generalization robustness across building populations.
