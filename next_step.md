# Resort Churn Project: Next Steps

## Where We Are Now

Current best model is CatBoost tuned with Optuna (interrupted at trial 19), holdout F1 = 0.824, AUC = 0.913. XGBoost 5-fold CV baseline sits at 0.828 F1. KNN baseline is at 0.713. We have 10 engineered features across 27 total, and our first submission predicts 48% churn on the test set.

The plan from here is organized into six areas: additional EDA, the K-Means innovative approach, new feature engineering, more tree-based models with proper tuning, ensembling, and items needed to complete the presentation rubric.

---

## 1. Additional EDA

**Booking year signal**

We extracted Booking_Month from BookingDate but never looked at the year. The data spans 2023 and 2024. If churn rates differ between years it could reflect a pricing change, a resort policy shift, or simply different booking cohorts behaving differently. We run a quick churn rate comparison between 2023 and 2024 bookings first. If the gap is meaningful, we add Booking_Year as a feature alongside Booking_Month.

**High-spend cohort deep dive**

The original high-spend audit found 635 rows (9.1% of train) with at least one spend category above $100k, and their churn rate was 52% vs a 50% baseline. The gap was small but the EDA code comment flagged this as a candidate feature. Before building it, we need to understand whether these rows represent a real luxury segment or data entry errors. If the spend distribution within this group looks plausible (a smooth tail rather than isolated spikes), we treat it as real and test a binary flag.

**VIP x AllInclusive interaction**

Univariate analysis showed VIP=1 guests churn at ~37% and AllInclusive=1 guests churn at ~82%. These are the strongest opposing signals in the data. Their combination was not included in the systematic pairwise interaction search. The question is whether VIP status overrides the all-inclusive churn effect: VIP guests may book all-inclusive as a loyalty perk and stay retained regardless, while non-VIP all-inclusive guests churn at the full 82%. We compute the four-cell churn table (VIP x AllInclusive) and decide whether the combination warrants a new feature.

---

## 2. Innovative Approach: K-Means Cluster as a Model Feature

In EDA, K-Means with k=4 produced four guest personas that aligned almost perfectly with the manual EDA segments. Cluster 1 (young, mostly all-inclusive, near-zero extra spend) had 76% churn. Cluster 2 (older, not all-inclusive, moderate spend) had 21% churn. These are real, distinct groups.

The hypothesis is that the cluster label encodes a combination of features that the model would otherwise need several splits to approximate. We validate with 5-fold CV: run one version with `Guest_Cluster` included and one without, and keep it only if CV F1 improves. We also check whether it adds value on top of the already-engineered AllInclusive_Region, HasAnySpend, and other FE features.

---

## 3. New Feature Engineering

Each feature below is motivated by a specific EDA finding. We validate every candidate with a 5-fold CV delta before including it in the final feature set. Only features that improve CV F1 make it in.

**AllInclusive x AgeGroup**

The systematic categorical interaction search flagged AllInclusive x AgeGroup as a high-lift pair. Within the current pipeline we only encoded the AllInclusive x Region combination. AgeGroup=Minor within all-inclusive bookings (likely family or school-holiday packages) represents a distinct behavioral segment that the model currently has to reconstruct through multiple splits across two separate columns. We encode it as a combined string category using the same approach as AllInclusive_Region.

**High_Spend_Flag**

If the high-spend cohort deep dive (section 1 above) confirms the rows are genuine, we create a binary flag for any guest with any single spend category above $100k. The standalone churn lift is small (+2 pp), but this group may interact with AllInclusive or Region in ways the flag alone does not show. We test it on top of Total_Spend and HasAnySpend to see if it adds CV value.

**Booking_Year**

If the booking year EDA check (section 1 above) finds a meaningful churn difference between 2023 and 2024, we extract the year as a numeric feature alongside Booking_Month. It costs one column and can be dropped if the CV delta is flat.

**Spend ratios**

The AllInclusive x HasAnySpend interaction showed that for non-all-inclusive guests, extra spending is a strong retention signal (30% churn vs 62%). Within the spending group, individual column MI and LGBM gain scores showed Spa and Entertainment carrying more signal than RoomService or Retail. This suggests that what guests spend on, not just how much, may matter. We create per-category ratios (Spa / Total_Spend, Dining / Total_Spend, etc.) for rows where Total_Spend > 0, and set them to zero otherwise. These capture the spending profile independently of magnitude and may help separate guest types within the non-zero spend population.

**VIP_AllInclusive**

If the VIP x AllInclusive interaction EDA (section 1 above) finds a meaningful divergence from the marginals, we encode the combination as a string concat (e.g., "True_True", "False_True"). The feature costs one column and is validated by CV delta.

**PromoCode_Region**

The missingness delta analysis ranked PromoCode=None at +31 pp churn lift and Region=Europe at +23 pp, two of the strongest signals in the data. The original pairwise interaction search excluded PromoCode because it was ~47% missing at the time. Now that missing PromoCode is recoded as "None", we can test this pair. The combination Europe + no promo code may produce a churn rate comparable to what AllInclusive x Region showed. We check the four-cell table first, then encode as a string concat if the lift is substantial.

**DaysSinceEmail buckets**

Decile binning showed DaysSinceEmail flat at 47-53% churn across all bins. But decile binning distributes observations evenly and can mask cliff effects at specific thresholds. The meaningful boundaries are behavioral: 0-30 days means the guest is actively receiving marketing, 31-180 days is a normal gap, and 180+ days means the guest has effectively been dropped from the mailing list or stopped engaging. We create an explicit 3-level categorical bucket and check whether these groups show differentiated churn rates where the decile bins did not.

---

## 4. More Tree-Based Models

We are focusing on tree-based models going forward. KNN already satisfies the non-tree rubric requirement.

**LightGBM as a competition model**

LightGBM was used during EDA for feature importance ranking but was never run as a proper competition model. We run it with default hyperparameters first (5-fold stratified CV, report F1), then submit to Kaggle to get a leaderboard score. The comparison between CV F1 and LB F1 tells us whether our cross-validation setup is well-calibrated. LGBM also trains faster per iteration than CatBoost, which makes it better suited to the 200-trial Optuna runs.

**Random Forest**

Quick 5-fold CV baseline. Random Forest typically underperforms boosted trees on tabular data but occasionally finds value in datasets where boosting overfits high-signal features. More importantly, it adds to the list of algorithms tried for the presentation rubric. We run it with default parameters and report CV F1, nothing more.

**CatBoost with 200-trial Optuna**

We re-run the Optuna session cleanly from scratch, this time targeting 200 trials and explicitly saving the best parameters at the end. We also expand the search space from the original three parameters (depth, learning_rate, iterations) to five: add `l2_leaf_reg` (controls L2 regularization, helps prevent overfit on high-signal features like AllInclusive) and `min_child_samples` (controls minimum leaf size, prevents splits on tiny groups). 200 trials gives the Bayesian search enough budget to explore this 5-dimensional space and find stable optima. We print the best params and best CV F1 explicitly before moving on.

**LightGBM with 200-trial Optuna**

Parallel 200-trial Optuna study for LightGBM tuning five parameters: `num_leaves` (tree complexity, analogous to CatBoost depth), `learning_rate`, `n_estimators`, `min_child_samples`, and `colsample_bytree` (feature subsampling per tree, helps reduce correlation between trees). LGBM trains faster per trial than CatBoost so the 200-trial budget runs in less wall time.

For every model in this section: record default CV F1, record tuned CV F1, submit to Kaggle, record LB F1. This directly satisfies the rubric requirement to distinguish CV scores from leaderboard scores.

---

## 5. Ensemble

After tuning both CatBoost and LightGBM, we try blending their predictions. The approach is to average the out-of-fold predicted probabilities from both models before applying the classification threshold. If the two models make different errors (one catches churners the other misses), the average probability will rank those borderline cases more confidently than either model alone.

We validate the blend with out-of-fold predictions to avoid leakage. If the blended OOF F1 is better than either model individually, we submit the blended probabilities to Kaggle.

---

## 6. Threshold Tuning

All current submissions use the default 0.5 threshold. The F1-optimal threshold is not necessarily 0.5; it depends on the model's calibration and the specific distribution of predicted probabilities on this dataset.

After generating final out-of-fold probability predictions (from either CatBoost, LightGBM, or the ensemble), we sweep the threshold from 0.3 to 0.7 in small steps and compute OOF F1 at each value. We pick the threshold that maximizes OOF F1 and apply it to the test set predictions. This does not touch the test labels and does not cause leakage.

---

## 7. Rubric Completeness

**3 correct and 3 incorrect prediction examples**

From the holdout set, we pull three true positives (correctly predicted churners) and three false negatives (churners the model missed) and display their feature values side by side. The goal is to find a pattern in the misses: were they AllInclusive=False guests with moderate spend that the model misjudged as low-risk? Were they European guests with a promo code (which typically reduces churn risk but they churned anyway)? These examples make for a strong presentation slide and satisfy the rubric requirement for detailed model description.

**Lessons Learned**

Write concise bullets for the presentation covering what surprised us during the project. Key candidates: missingness being predictive (PromoCode absent was a stronger signal than PromoCode present), Floor needing normalization (high LGBM split count but low gain revealed the structural problem), and the AllInclusive x Region interaction being nearly deterministic for European guests. Also cover what did not work as expected.

**Presentation next steps slide**

Separate from this planning doc, the rubric asks for a "Next Steps" slide in the presentation itself. That slide should be specific: what data would we want (longer booking history, email open rates, clickstream behavior, external seasonality data), what compute would we need (longer Optuna runs, multi-GPU for neural approaches if the dataset grew 10x), and what techniques we would try next (SHAP values for individual guest explainability, testing k=5 and k=6 for the Guest_Cluster feature, threshold tuning per region segment).