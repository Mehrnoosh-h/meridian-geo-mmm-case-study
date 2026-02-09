## Multicollinearity check (Meridian geo MMM, 1 brand + 1 SKU)

### What is multicollinearity (and why it matters in MMM)?
**Multicollinearity** happens when two or more predictors (here: media channels) move together so strongly that the model cannot reliably separate their individual effects. In MMM, this usually means:
- the model can still fit the KPI well (good overall R² / wMAPE),
- but **channel-level attribution and ROI can become unstable** (the model may “swap credit” between correlated channels).

### How we checked multicollinearity
I quantified multicollinearity using **Variance Inflation Factor (VIF)**. VIF measures how much a coefficient’s variance is inflated due to correlation with other predictors:
- **VIF ≈ 1**: no issue
- **VIF 5–10**: concerning
- **VIF > 10**: strong multicollinearity (channel-level interpretation becomes unreliable)

For my selected brand/SKU, VIF was extremely large:

| Channel   | VIF |
|----------|-----:|
| YouTube  | 1.233863e+14 |
| Facebook | 8.677456e+12 |
| Instagram| 1.324588e+14 |

These values indicate **near-perfect collinearity** between the channels (they carry almost the same signal), so estimating separate effects for each one is not identifiable from the data alone.

### Options to handle multicollinearity
Common strategies in MMM are:
1) **Combine correlated channels** into one “bundle” feature (e.g., `paid_social_video_bundle`).
   - Pros: keeps signal, improves stability, attribution is defensible at the bundle level.
   - Cons: you lose separate ROI for each channel inside the bundle.
2) **Remove one or more channels** and keep a representative channel.
   - Pros: simple; improves identifiability.
   - Cons: you may drop true signal and shift credit to other variables (potentially biased attribution).
3) **Stronger priors / regularization**
   - Can help, but with near-perfect collinearity it often cannot fully resolve channel-level identifiability.
4) **Calibration with experiments**
   - Best for separating channels, but requires extra data (lift tests / geo experiments).

### What I tested (two scenarios)
Because YouTube/Instagram/Facebook were extremely collinear, I evaluated two scenarios:

**Scenario A — Combine correlated channels**
- I combined the highly correlated channels into a single bundle feature and fit Meridian.
- `reviewer.ModelReviewer(mmm).run()` summary (combined):
  - Overall Status: **PASS**
  - Baseline Check: **REVIEW** (P(baseline < 0) = 0.54)
  - Goodness of Fit: **R² = 0.9496**, **MAPE = 0.1444**, **wMAPE = 0.1186**

**Scenario B — Remove Instagram and YouTube**
- I removed Instagram and YouTube and fit the model using the remaining channels.
- `reviewer.ModelReviewer(mmm).run()` summary (removed):
  - Overall Status: **PASS**
  - Baseline Check: **REVIEW** (P(baseline < 0) = 0.37)
  - Goodness of Fit: **R² = 0.9490**, **MAPE = 0.1448**, **wMAPE = 0.1192**

### Final choice and why
Both scenarios achieved **nearly identical fit**, which confirms multicollinearity is mainly an **interpretability** problem (not a pure predictive-performance problem).

**I chose Scenario B (remove Instagram and YouTube)** to reduce redundancy and improve identifiability in channel attribution. Given the extremely large VIF values (≈1e12–1e14), the three channels were carrying almost the same signal; keeping all of them can lead to unstable credit-splitting. Removing two highly collinear channels:
- simplifies the model and reduces attribution ambiguity,
- makes the remaining channel effects easier to interpret and compare,
- keeps overall model fit essentially unchanged while avoiding duplicated signals.

This choice is especially reasonable when the removed channels are **highly overlapping/duplicative** in timing and spend patterns, or when a simpler, more interpretable model is preferred for reporting and decision-making.
### Note on the “negative baseline” diagnostic
In both scenarios, the Model Reviewer flagged **Baseline Check: REVIEW**, meaning the model assigns some posterior probability that the **baseline contribution dips below zero** at certain time points. This does **not** mean the model “failed.” It typically happens when:
- the KPI is very low in some weeks (or noisy),
- strong seasonality/controls + media explain most variation,
- the model is fitting small residual fluctuations and a small portion of posterior draws cross below zero.

In our comparison, Scenario B (removing Instagram & YouTube) reduced this issue:
- Scenario A (combined): P(baseline < 0) = **0.54**
- Scenario B (removed): P(baseline < 0) = **0.37**

So, while both are still marked “REVIEW,” **Scenario B shows fewer negative-baseline draws** and is slightly more stable on this diagnostic. I treated this as a minor modeling artifact and verified it by visually inspecting the baseline curve in the model-fit charts; occasional small dips are expected in Bayesian MMM and do not invalidate the overall results.

## Out-of-sample (holdout) evaluation

I evaluated generalization using a **time-based holdout**:
- **Train window:** 2022-07-04 → 2025-03-31 (144 weeks)
- **Test (holdout) window:** 2025-04-07 → 2025-06-23 (12 weeks)

Because this Meridian version’s `Analyzer.predictive_accuracy()` reports metrics for a user-selected subset of dates (via `selected_times`) and does not automatically tag `Train/Test` inside the returned object, I computed **Train** and **Test** metrics by running `predictive_accuracy()` **twice**: once on the training weeks and once on the holdout weeks.

### Predictive accuracy (Meridian)
| Evaluation set | Metric     | Geo-level | National-level |
|---|---|---:|---:|
| Test  | R_Squared | 0.9483 | 0.9529 |
| Test  | MAPE      | 0.1343 | 0.0185 |
| Test  | wMAPE     | 0.1117 | 0.0182 |
| Train | R_Squared | 0.9480 | 0.9889 |
| Train | MAPE      | 0.1457 | 0.0277 |
| Train | wMAPE     | 0.1200 | 0.0244 |





## Data source
This project uses the **“MMM Weekly Data – Geo: India”** dataset from Kaggle:  
https://www.kaggle.com/datasets/subhagatoadak/mmm-weekly-data-geoindia

## Dataset description
The dataset is organized as a **weekly, geo-level panel** (multiple geographies observed over time) and includes multiple marketing/media variables along with outcome (KPI) variables suitable for marketing mix modeling.

It contains **multiple brands**, and for each brand, **multiple SKUs** (products). To keep the modeling scope focused and to produce interpretable results, I filtered the dataset to a single product line:

- **Brand:** Brand A  
- **SKU:** SKU 2

All Meridian model training and comparisons in this repository are based on this filtered **Brand A / SKU 2** subset.


