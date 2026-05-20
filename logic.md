# Resort Guest Churn Prediction: Pipeline Logic

This document walks through every section of the pipeline, explaining the thinking behind each step before describing what we did and what we found. The goal is to make the reasoning traceable from raw data all the way to submission.

---

## 1. Problem Framing

The first decision is what kind of problem this is. The business question is: which guests are likely to cancel? That is a binary question (yes or no), not a ranking or a score, so we frame it as binary classification with a target of 1 (churned) and 0 (stayed).

The metric choice matters because it shapes everything downstream. We use F1 score instead of accuracy. Even with roughly balanced classes, accuracy rewards a model that simply predicts the majority class well. F1 balances precision (when we flag a churner, are we right?) and recall (are we catching most of the actual churners?). For a churn intervention problem, both matter: missing a churner means a lost guest, and flagging too many non-churners wastes retention budget.

---

## 2. Setup

Before any analysis, we define three named column groups: `CATEGORICALS`, `SPEND_COLS`, and `NUMERICS`. These lists get referenced in EDA, cleaning, feature engineering, and model building. Defining them once at the top means there is a single place to update if a column is renamed or added, and it makes every downstream cell self-documenting.

`RANDOM_STATE = 42` is set once and passed to every stratified split, cross-validation, and model call. This ensures the notebook produces the same results on every run, which matters when comparing model variants or sharing results.

---

## 3. Load Data

Before writing any analysis, we load both files and immediately print their shapes and preview the first rows. This is a sanity check, not analysis. We want to confirm the CSV loaded without silent errors (wrong delimiter, wrong encoding, truncated rows), verify the column count matches expectations, and get a first look at what the raw values actually look like. Seeing NaN values in the head preview also primes us for the systematic quality checks that follow in EDA.

Train: 6,954 rows, 21 columns. Test: 1,739 rows, 20 columns (no target column, as expected).

---

## 4. EDA

EDA is structured as a deliberate sequence of questions, each building on the last. We start with data quality before touching any signal analysis, because a badly misunderstood column can poison every conclusion that follows.

### 4.1 Missingness by Column

We need to know which columns have gaps and how large those gaps are before deciding on any imputation strategy. A column that is 47% empty is a fundamentally different problem from one that is 2% empty. You cannot impute 47% of a column with the mode and expect the result to be meaningful; the missing values are likely not random.

What we found: PromoCode is missing ~47% of the time. Room and AgeGroup are missing 5-6%. Most spend columns and VIP are missing 2-4%. Over 4,500 of the 6,954 rows have at least one missing value, so this is not an edge case we can ignore.

The PromoCode finding immediately raises a question: are guests without a promo code just "unknown", or does not having a code mean something? That question gets answered next.

### 4.2 Missingness Correlation

If two columns tend to be missing together, there may be a single underlying cause: a guest segment that skips those fields, a data collection failure at a specific booking channel, etc. In that case, we might want to handle them jointly. If columns are independently missing, we can treat each one separately.

We also specifically expect Age and AgeGroup to correlate in their missingness since AgeGroup is derived from Age. Seeing that confirmed is a good data integrity check.

What we found: most columns are independent in their missingness. Age and AgeGroup correlate at ~0.36, which is expected. No surprising co-missingness patterns emerged.

This gives us confidence to handle each column's missing values on its own terms rather than needing a joint strategy.

### 4.3 Target Balance and First AllInclusive Look

Class imbalance changes everything: the right metric, whether we need class weights, how we sample for cross-validation. We check this before any feature analysis because the answer affects all of it. If churn were 5% of the data, we would need to adjust our approach significantly.

We also look at AllInclusive vs churn immediately because it is the most structurally distinctive feature in the column list, a binary flag that signals a fundamentally different booking type.

What we found: overall churn rate is ~50%, roughly balanced. No class weighting adjustments needed. AllInclusive=1 guests churn at ~82%, which is a very strong signal at first glance and puts it on the watchlist for the interaction analysis later.

### 4.4 Is Missingness Itself Predictive?

When a field is blank, the instinct is to fill it and move on. But the absence of a value can carry real information. A guest who never provided a promo code might behave differently from one who provided a code (or actively confirmed they had none). We compute the churn rate separately for rows where each column is missing vs rows where it is present, then look at the gap.

What we found: PromoCode missing drives a +31 percentage point lift in churn compared to guests where it is filled in. Room missing is +18 pp. AgeGroup missing is +16 pp. These are large, real differences, not noise.

This changes the cleaning strategy entirely. We do not impute these columns with a mode or median. For PromoCode, the "missing" state is a meaningful signal that we preserve as a literal category called "None". For Age and Room, we create binary flags (`Age_was_missing`, `Room_was_missing`) so the model can use the absence signal directly, even after we impute the underlying numeric value.

### 4.5 Univariate Churn by Categorical

Before building interactions or engineered features, we want to know which raw categorical features already separate churners from stayers on their own. This gives us a ranked list of where FE effort is worth spending. It also surfaces unexpected patterns that might signal data quality issues.

We compute churn rate per category level across all eight categoricals, alongside the count per group and the lift versus the overall base rate.

Key findings:
- AllInclusive=1: 82% churn (lift +0.315). The strongest single categorical signal.
- PromoCode="None" (no promo): 67% churn vs 36% for guests with a code. Confirms that PromoCode missingness matters.
- Region=Europe: 66% churn. Region=Americas: 43%. A 23 percentage point spread, which is large.
- ReferralSource shows the widest spread of any feature: TikTok and Billboard guests churn at 82-86%, while Email and Friend referrals are at 22-32%.
- AgeGroup=Minor: 58% churn. Adult groups cluster around 47-48%.

The Region and AllInclusive findings together raise a specific question: do they compound each other? Europe might be driving high churn partly because European guests are more likely to book all-inclusive. Or the combination might be even more extreme than either alone. This sets up the interaction analysis in sections 4.9 and 4.10.

### 4.6 Univariate Churn by Binned Numeric

Numeric features need binning before we can read churn rates across their range. Raw correlation coefficients miss non-linear patterns, particularly the kind we already suspect: a spike at zero, a cliff at a specific threshold. We use decile bins for Age, LoyaltyPoints, and DaysSinceEmail, and a direct groupby for SurveyScore since it only has a handful of distinct integer values.

What we found:
- LoyaltyPoints, DaysSinceEmail, SurveyScore: all flat, ~48-53% churn per bin regardless of the value. Weak standalone signals.
- Age: rows with Age=0 churn at 67%. All other age groups sit at 45-51%. This is not a demographic segment; it is a data entry error where missing age was recorded as zero.

The Age=0 finding means these 138 rows need to be recoded as NaN in cleaning. The flatness of LoyaltyPoints, DaysSinceEmail, and SurveyScore on their own does not mean they are useless. It means they may need to be combined into something that measures engagement more holistically, which leads to the Engagement_Score feature later.

### 4.7 Spend Univariate with Manual Bins

The five spend columns (RoomService, Dining, Retail, Spa, Entertainment) have a structural issue: a massive spike at exactly $0 followed by a long tail into six figures. Decile binning would lump thousands of zero-spend rows together with small-spenders in the bottom deciles, obscuring the most important boundary.

We use manual bins: exactly $0, $0-$100, $100-$1k, $1k-$10k, $10k-$100k, and above $100k. This preserves the zero group as its own segment and lets us read the churn rate at each tier cleanly.

What we found: zero spend leads to 59-64% churn across every single spend column. Any spend at all drops churn to ~30%. The gradient above zero (from $1 to $100k) is much less dramatic than the zero vs non-zero gap. Whether someone spent anything is more informative than how much they spent.

This directly motivates two features. `HasAnySpend` is a binary flag capturing the zero vs non-zero boundary where the churn signal is sharpest. `Total_Spend` captures the magnitude for the cases where it does matter.

### 4.8 Mutual Information and LightGBM Feature Ranking

Univariate churn tables show one feature at a time and treat each independently. Mutual information and a quick LightGBM model give us a ranking that accounts for the fact that features share information with each other. They also catch non-linear signals that correlation-based methods would miss.

The specific diagnostic we care about from LGBM is the split count vs gain comparison. Split count tells us how often a feature is used. Gain tells us how much error reduction each use delivers. If a feature has high split count and low gain, the model is trying hard to use it but not getting much return. That usually means the raw form of the feature is not the right representation.

What we found:
- Mutual information: AllInclusive is the single strongest feature (MI = 0.115), consistent with the univariate finding. Spa and ReferralSource follow.
- LGBM gain: AllInclusive first, then PromoCode, ReferralSource, and Spa.
- Floor: very high split count, low gain. The model keeps reaching for it but raw floor numbers are not helping.

The Floor split/gain mismatch points to a structural problem. Floor numbers are not comparable across wings: floor 5 in Wing A might be near the top of that wing, while floor 5 in Wing G might be near the bottom. The absolute number is meaningless without knowing how tall the wing is. This motivates normalizing Floor within its wing, which becomes `Floor_relative`.

### 4.9 Interaction: AllInclusive x HasAnySpend

AllInclusive and extra spending both showed strong standalone churn signals. The question is whether they operate independently or whether one overrides the other. This matters for feature engineering: if the effect of spending on churn completely disappears for all-inclusive guests, we may need a conditional representation rather than just two separate columns.

We cross the two variables and compute churn rates for each of the four combinations.

What we found:
- AllInclusive=True, no extra spend: ~82% churn
- AllInclusive=True, has extra spend: ~82% churn
- AllInclusive=False, no extra spend: ~62% churn
- AllInclusive=False, has extra spend: ~30% churn

For all-inclusive guests, extra spending makes no difference. They churn at ~82% regardless. For non-all-inclusive guests, spending is a very strong retention signal. These are two fundamentally different behavioral regimes. The effect of spending on churn is entirely conditional on AllInclusive status.

This means a model using two separate columns for AllInclusive and HasAnySpend will partially capture this, but it needs to learn the interaction through deep splits. The systematic search below makes the case for encoding the interaction explicitly.

### 4.10 Systematic Categorical Interaction Search

After finding one strong interaction manually, we run a systematic search over all pairs of categorical features to avoid missing others. For each pair, we compute the joint churn rate per combination and filter for groups with at least 30 observations (to avoid noise from tiny segments). We rank by how far each combination deviates from what we would expect if both features acted independently.

What we found:
- Region=Europe x AllInclusive=1: 98.6% churn, lift +0.483 above the overall base rate
- Region=AsiaPacific x AllInclusive=1: 91.5%
- Region=Americas x AllInclusive=1: substantially lower

This is not just "AllInclusive=1 is always high". The geography dramatically modifies the effect. European all-inclusive guests are near-certain churners. This interaction is far stronger than any single column alone.

A tree model can learn this by first splitting on AllInclusive, then on Region at the next level. But that requires two levels of depth and the signal can diffuse across branches. Encoding it as a single string feature (e.g., "True_Europe") gives the model direct access to the combined category in a single split. This is the primary motivation for `AllInclusive_Region`.

### 4.11 K-Means Clustering

After all the pairwise analysis, we want to step back and ask whether the data naturally groups into coherent guest personas without us guiding it. Clustering is unsupervised and will surface structure we did not specifically look for. It also serves as a sanity check: if the natural clusters in the data align with the segments our EDA identified, we have more confidence that our feature choices are capturing real patterns.

We test k from 2 to 10 using inertia and silhouette scores, and k=4 emerges as the best by silhouette.

What we found:
- Cluster 1 (76% churn): young guests (avg age 26), 81% on all-inclusive, near-zero extra spend (~$150 total)
- Cluster 0 (34% churn): older guests (avg 33), mostly not all-inclusive, high extra spend (~$63k)
- Cluster 2 (21% churn): lowest churn, older, not all-inclusive, moderate spend
- Cluster 3 (47% churn): mixed

These clusters map almost exactly onto the segments the EDA identified: high-spend non-AI guests churn less, all-inclusive guests churn more, and within all-inclusive, young low-spenders are the most at-risk group. The alignment gives us confidence in the feature set.

The cluster labels themselves are not added to the model. Assigning test rows to clusters would require either fitting the clustering on training data (which risks leakage) or re-fitting on test (which gives different cluster definitions). More fundamentally, the features that define the clusters (AllInclusive, spend, age) are already in the model, so adding the cluster label would be redundant at best and a source of noise at worst.

---

## 5. Data Cleaning

Cleaning happens after EDA because EDA told us exactly what needs fixing and why. We do not apply generic rules like "fill all NaN with the median". Every decision here is traced to a specific finding.

We combine train and test into a single dataframe before cleaning. The reason is consistency: any categorical recoding must produce the same labels in both sets. If we cleaned them separately, a category that only appears in test would be invisible to the model at inference time.

**PromoCode NaN becomes "None"**: The missingness delta analysis showed +31 pp churn lift for missing PromoCode. The missing state is a real signal, not random absence. Recoding it as "None" turns it into a category that CatBoost can encode with its own target statistics, preserving the signal cleanly.

**Age=0 becomes NaN**: These 138 rows have an entry error, not a real zero age. Keeping 0 would make CatBoost treat it as a valid demographic. Converting to NaN lets CatBoost handle it as missing internally, and `Age_was_missing` captures the associated churn signal.

**AllInclusive 0/1/NaN becomes "False"/"True"/"unknown"**: AllInclusive has three meaningfully different states. Zero means the guest did not take all-inclusive. One means they did. NaN means we do not know, and from EDA the unknown group behaves differently from a confirmed 0. Encoding all three as strings gives CatBoost three distinct categories to learn from, rather than trying to handle a mix of numeric 0/1 and NaN.

**Other categoricals NaN becomes "Unknown"**: Region, PackageType, BookingChannel, ReferralSource, and AgeGroup all get a literal "Unknown" string. This keeps them as proper categoricals and lets the model learn whatever churn rate is associated with the unknown group.

**BookingDate is parsed to datetime**: The raw string is not usable for feature extraction. Parsing it to datetime is a prerequisite for pulling out the booking month in FE.

---

## 6. Feature Engineering

Every feature here was motivated by a specific EDA finding. The sequence of EDA was not exploratory decoration; it was the process of identifying exactly which transformations were justified by the data.

**Total_Spend**

We have five separate spend columns. Before asking whether any individual category matters, we want a single number capturing overall spend. This serves as a baseline aggregate and also ensures the model has access to the combined signal even if individual columns interact with each other in complex ways. Computed as the sum of all five spend columns, with NaN treated as zero.

**HasAnySpend**

The spend binning EDA showed that the churn difference between $0 and any positive spend (~59-64% vs ~30%) is far larger than the difference between $100 and $10,000. The binary question "did this guest spend anything at all?" is more predictive than the dollar amount. HasAnySpend is a 1/0 flag that captures this boundary directly. It also makes the interaction finding from section 4.9 more model-accessible: AllInclusive guests have HasAnySpend=1 at a different rate than non-AI guests.

**AllInclusive_Region**

The systematic interaction search found that Region=Europe combined with AllInclusive=1 produces 98.6% churn, lift +0.483 above the base rate. The region effect on churn is not independent of AllInclusive status. A tree model can in theory discover this by splitting on AllInclusive first and then Region, but that requires at least two levels of depth and the signal spreads across branches. Encoding the combination as a single string (e.g., "True_Europe", "False_Americas") gives CatBoost direct access to all region/AllInclusive combinations as first-class categories in a single split.

**Wing, View, Floor**

The Room column stores a raw string like "G/630/S". The LGBM importance analysis flagged Floor with high split frequency, meaning structured information is buried in that string. We parse it into three columns: Wing (building section, the letter prefix), Floor (the numeric level), and View (the suffix letter indicating room type). These represent structurally different signals and should not stay bundled as one opaque string.

**Floor_relative**

After parsing Floor, LGBM showed high split count but low gain on the raw number. The reason is that floors are not comparable across wings. Floor 10 in Wing G (a tall building) is a very different position than floor 10 in Wing A (a shorter one). The absolute number misleads the model. We normalize Floor as Floor divided by the maximum floor in that wing, producing a 0-to-1 value that represents relative height within the wing and is comparable across the building.

**Age_was_missing**

The missingness delta analysis found that guests with missing Age churn at +16 pp above guests with a recorded age. This is not random. It likely reflects a guest segment that engaged less with the booking system. Simply imputing Age with the median and moving on would discard this signal. The binary flag `Age_was_missing` lets the model use the absence of age information as a predictor independently of whatever Age value we impute.

**Room_was_missing**

Same logic as Age_was_missing. Guests with no Room record show +18 pp higher churn than guests with a room assigned. A missing room suggests an incomplete or problematic booking, which is a meaningful state. The flag preserves that signal after we otherwise fill in NaN.

**Booking_Month**

BookingDate as a raw timestamp is not usable by the model. The most plausible seasonal pattern is the month a booking was made, since cancellation behavior at resorts often varies by season (holiday bookings vs off-season, summer vs winter). We extract the month as a numeric 1-12 and let the model determine which months, if any, are associated with higher churn.

**Engagement_Score**

LoyaltyPoints, SurveyScore, and DaysSinceEmail all showed near-zero standalone predictive power in the univariate numeric checks. Each of them is flat across its range. But they all measure facets of the same thing: how engaged a guest is with the resort. A guest who rated their stay highly and was emailed last week is in a very different state from one who rated highly but has not been contacted in 11 months.

We compute `SurveyScore * 100 - DaysSinceEmail`. High values mean a satisfied guest who has been in contact recently. Low values mean either low satisfaction or a long gap since contact. The multiplier of 100 on SurveyScore (which ranges 1-5) brings it to a comparable magnitude to DaysSinceEmail (which can range from single digits to 365+). The result is a composite engagement signal that none of the three source columns could provide on their own.

---

## 7. Baseline Models

We run two baseline models before touching Optuna or the final CatBoost configuration. The goal is not to compete with the final model; it is to answer two specific questions.

First: is the problem even learnable from these features? If KNN gets near-random F1, we need to go back to EDA. KNN is a simple distance-based model that requires explicit imputation and one-hot encoding, so a good score here is a lower bound on what a stronger model can achieve. KNN scores 0.713 F1 (5-fold CV), which confirms the features carry real signal.

Second: do the engineered features actually help a tree model? XGBoost is architecturally close to CatBoost and uses the same feature set. A strong XGBoost baseline means the FE work is paying off for tree methods, not just for the final model. XGBoost scores 0.828 F1, a meaningful jump from KNN and a strong validation of the feature set.

We choose CatBoost for the final model for specific reasons. It handles categorical features natively via ordered target encoding during training, without us needing to one-hot encode or label encode. It handles missing values internally, removing the need for imputation that could leak information. For a dataset with 10 categorical features and considerable missingness, this matters both for code simplicity and for reducing leakage risk.

---

## 8. Optuna Hyperparameter Tuning

The three CatBoost hyperparameters that most affect behavior are tree depth, learning rate, and number of iterations. They cannot be tuned independently because they interact: a deep model with many iterations and a high learning rate will overfit; a shallow model with few iterations and a low rate will underfit. We need to find a configuration where all three are balanced together.

Grid search over this three-dimensional space would require hundreds of model fits. We use Optuna, which uses a Bayesian-style search to learn which regions of the parameter space are promising and allocate more trials there.

Each Optuna trial runs 3-fold stratified CV (not 5-fold) and returns mean F1. Using 3 folds instead of 5 keeps each trial fast enough that 25 trials are feasible. We search depth 4-8, learning rate 0.01-0.3 on a log scale (log scale because learning rate sensitivity is multiplicative, not additive), and iterations 100-600.

The session was interrupted at trial 19. Optuna stores completed trial results continuously, so the best parameters from the 18 completed trials are used for the final model. The best hyperparameters are not printed in the saved notebook output, but the model trained with them achieves 0.824 F1 on a holdout split.

---

## 9. Evaluation

A single F1 number tells us overall performance but not where the model fails or whether it is failing in a costly direction. We evaluate using three views together.

The confusion matrix breaks down predictions into true positives, false positives, true negatives, and false negatives in absolute counts. For a churn model, the cost of each error type is different: missing a churner (false negative) means a lost guest we could not intervene on; incorrectly flagging a stayer (false positive) means wasted intervention effort. The matrix shows 561 churners correctly identified, 140 missed, 590 stayers correctly identified, and 100 incorrectly flagged as churners.

The ROC curve and AUC evaluate the model across all possible thresholds, not just 0.5. AUC = 0.913 means strong discriminative ability. The steep initial rise in the ROC curve means the model is very confident about the clearest churners (the all-inclusive European guests identified in EDA) before the harder cases start appearing.

Feature importance closes the loop on the FE work. The top 10 features include AllInclusive, Total_Spend, Floor_relative, ReferralSource, and Engagement_Score. Three of those five (Total_Spend, Floor_relative, Engagement_Score) are engineered, not raw columns. They appear in the top 10 alongside the strongest raw features, which validates that the transformations added real predictive value and were not just noise.

---

## 10. Submission

During cross-validation and holdout evaluation, we withheld portions of the training data to get unbiased performance estimates. For the final submission, there is no reason to keep any training data out. We retrain on the full 6,954-row training set so the model has seen as many real examples as possible before predicting on the 1,739 test rows.

The classification threshold stays at 0.5. Shifting the threshold requires validation on a held-out set to avoid overfitting the threshold to the evaluation data. Since we used our holdout set for the performance report in section 9, we do not have a clean holdout left to tune the threshold against, and the default of 0.5 is consistent with how F1 is computed in the competition.

The final model predicts 48% churn on the test set. The training base rate was ~50%, so the prediction is consistent with what we learned from the data. The submission file contains one row per test guest with GuestID and the predicted Churned label.
