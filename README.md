## Multicollinearity check (Meridian geo MMM, 1 brand + 1 SKU)

## Executive summary 

**Goal.** Build a geo-level Marketing Mix Model (MMM) using Google Meridian to estimate media contributions and derive actionable budget insights.

**Data & scope.**
- Source: Kaggle — “MMM Weekly Data (Geo India)”  
- Subsetting: **Brand A**, **SKU 2** (weekly, multiple geos)
- Media channels used in this run: TV, Facebook, Print, Radio
- Evaluation: rolling backtest with 12-week holdouts (multiple time splits)

**Key modeling decisions.**
- **Multicollinearity check:** I computed VIF and found extreme multicollinearity among some media signals (VIF on the order of **1e12–1e14**), meaning separate channel effects are not reliably identifiable without additional information.
- **Resolution chosen:** I used the scenario where highly collinear channels are removed / simplified (rather than forcing the model to split nearly identical signals), prioritizing **stable, defensible attribution**.

**Model validation (quality + out-of-sample).**
- Meridian’s built-in reviewer checks: PASS overall, with a minor baseline review flag (occasional small negative baseline values).
- Rolling 12-week holdouts show performance is stable across time (not driven by a single split).  
  See: *Rolling backtest plots* (wMAPE and R² across holdout windows).

**Attribution stability.**
- I extracted **posterior incremental outcome** by channel for each rolling holdout window and tracked contribution share.
- Result: channel shares are generally stable across windows, supporting robust attribution rather than window-specific artifacts.

**Decision insights (holdout window).**
- **ROI (posterior mean):** TV **0.230**, Radio **0.223**, Print **0.147**, Facebook **0.080**
- **Marginal ROI (posterior mean):** TV **0.108**, Radio **0.059**, Print **0.033**, Facebook **0.032**
- **Implication:** TV and Radio are consistently the strongest candidates for incremental budget (highest “next-dollar” efficiency), while Facebook is the weakest in this window.

**Repo outputs.**
- Figures: rolling backtest plots, attribution stability plots, ROI & marginal ROI charts
- Tables: holdout metrics by split, contribution shares by split, ROI summaries (posterior mean + uncertainty)


### What is multicollinearity (and why it matters in MMM)?
**Multicollinearity** happens when two or more predictors (here: media channels) move together so strongly that the model cannot reliably separate their individual effects. In MMM, this usually means:
- the model can still fit the KPI well (good overall R² / wMAPE),
- but channel-level attribution and ROI can become unstable (the model may “swap credit” between correlated channels).

### How we checked multicollinearity
I quantified multicollinearity using Variance Inflation Factor (VIF). VIF measures how much a coefficient’s variance is inflated due to correlation with other predictors:
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
Both scenarios achieved **nearly identical fit**, which confirms multicollinearity is mainly an interpretability problem (not a pure predictive-performance problem).

**I chose Scenario B (remove Instagram and YouTube)** to reduce redundancy and improve identifiability in channel attribution. Given the extremely large VIF values (≈1e12–1e14), the three channels were carrying almost the same signal; keeping all of them can lead to unstable credit-splitting. Removing two highly collinear channels:
- simplifies the model and reduces attribution ambiguity,
- makes the remaining channel effects easier to interpret and compare,
- keeps overall model fit essentially unchanged while avoiding duplicated signals.

This choice is especially reasonable when the removed channels are highly overlapping/duplicative in timing and spend patterns, or when a simpler, more interpretable model is preferred for reporting and decision-making.
### Note on the “negative baseline” diagnostic
In both scenarios, the Model Reviewer flagged Baseline Check: REVIEW, meaning the model assigns some posterior probability that the baseline contribution dips below zero at certain time points. This does not mean the model “failed.” It typically happens when:
- the KPI is very low in some weeks (or noisy),
- strong seasonality/controls + media explain most variation,
- the model is fitting small residual fluctuations and a small portion of posterior draws cross below zero.

In our comparison, Scenario B (removing Instagram & YouTube) reduced this issue:
- Scenario A (combined): P(baseline < 0) = **0.54**
- Scenario B (removed): P(baseline < 0) = **0.37**

So, while both are still marked “REVIEW,” Scenario B shows fewer negative-baseline draws and is slightly more stable on this diagnostic. I treated this as a minor modeling artifact and verified it by visually inspecting the baseline curve in the model-fit charts; occasional small dips are expected in Bayesian MMM and do not invalidate the overall results.

## Out-of-sample (holdout) evaluation

I evaluated generalization using a **time-based holdout**:
- **Train window:** 2022-07-04 → 2025-03-31 (144 weeks)
- **Test (holdout) window:** 2025-04-07 → 2025-06-23 (12 weeks)

Because this Meridian version’s `Analyzer.predictive_accuracy()` reports metrics for a user-selected subset of dates (via `selected_times`) and does not automatically tag `Train/Test` inside the returned object, I computed Train and Test metrics by running `predictive_accuracy()` twice: once on the training weeks and once on the holdout weeks.

### Predictive accuracy (Meridian)
| Evaluation set | Metric     | Geo-level | National-level |
|---|---|---:|---:|
| Test  | R_Squared | 0.9483 | 0.9529 |
| Test  | MAPE      | 0.1343 | 0.0185 |
| Test  | wMAPE     | 0.1117 | 0.0182 |
| Train | R_Squared | 0.9480 | 0.9889 |
| Train | MAPE      | 0.1457 | 0.0277 |
| Train | wMAPE     | 0.1200 | 0.0244 |

**Interpretation:** Geo-level performance is consistent between Train and Test (R² ≈ 0.95 and wMAPE ≈ 0.11–0.12), suggesting the model generalizes reasonably well to unseen weeks. National-level errors are much smaller because aggregating across geographies reduces noise and cancels some geo-specific fluctuations, so MAPE/wMAPE on the national series can look substantially lower than geo-level metrics.

## Rolling backtest (stability across time)

To ensure the results are not dependent on a single train/test split, I ran a rolling time-series backtest using 6 non-overlapping holdout windows (each 12 weeks). For each split, I computed Meridian’s predictive accuracy metrics on the Test window only.

### Test-window stability (across 6 splits)

**Geo-level (harder; one observation per geo×week):**
- **wMAPE:** mean **0.1173**, std **0.0137** (range **0.1065–0.1442**)
- **MAPE:** mean **0.1471**, std **0.0153** (range **0.1329–0.1719**)
- **R²:** mean **0.9436**, std **0.0205** (range **0.9040–0.9607**)

**National-level (easier; aggregated across geos, lower noise):**
- **wMAPE:** mean **0.0183**, std **0.0098** (range **0.0071–0.0342**)
- **MAPE:** mean **0.0188**, std **0.0091** (range **0.0077–0.0347**)
- **R²:** mean **0.9541**, std **0.0512** (range **0.8580–0.9959**)

### Interpretation
Overall, geo-level metrics are **stable across time** (Test wMAPE ≈ **0.11–0.14**, R² ≈ **0.90–0.96**), suggesting the model generalizes reasonably well to unseen weeks. National-level errors are much smaller because aggregation across geographies reduces noise and cancels some geo-specific fluctuations, so national metrics can look substantially better than geo-level metrics.

Notably, the **worst geo-level Test window** (wMAPE ≈ **0.1442**, R² ≈ **0.9040**) indicates a period where the model had more difficulty generalizing—potentially due to a regime shift (promotions, distribution changes, or seasonality effects) during that specific time window.


### Rolling backtest (12-week holdouts)

<table border="0" cellspacing="0" cellpadding="0">
  <tr>
    <td align="center" valign="top">
      <img src="https://github.com/user-attachments/assets/7b8f1872-3aa6-4fea-b7e2-7eee3d18e11e" width="480" />
    </td>
    <td align="center" valign="top">
      <img src="https://github.com/user-attachments/assets/957fd566-f58b-488b-8a3c-e05a0897a3f7" width="480" />
    </td>
  </tr>
</table>

<p align="center">
  <sub><em>Rolling backtest: geo-level Test wMAPE (left) and Test R² (right) across 12-week holdout windows.</em></sub>
</p>

Each point represents one 12-week holdout window. Tracking wMAPE and R² across rolling splits helps confirm performance is stable over time and not driven by a single train/test split.




**Attribution stability:** Using the same rolling 12-week holdouts, I extracted posterior incremental outcome by channel and tracked each channel’s contribution share. The box/line plots show how stable (or variable) channel attribution is across time windows—useful for assessing whether ROI/contributions are robust or sensitive to the evaluation period.
<img width="1416" height="473" alt="image" src="https://github.com/user-attachments/assets/49a6be7a-a284-4465-98c7-bcb181c57a53" />

## ROI vs Marginal ROI (decision-oriented)

After fitting the model, I computed both ROI and Marginal ROI on the most recent 12-week holdout window.

### What they mean
- **ROI** answers: *“On average, how much incremental outcome do I get per unit spend in this channel over the evaluation period?”*
- **Marginal ROI** answers: *“If I add one more unit of spend right now, how much incremental outcome do I get?”*  
  This is the key metric for budget reallocation, because MMM channels often show diminishing returns.

**ROI (posterior mean):**
- TV: **0.230**
- Radio: **0.223**
- Print: **0.147**
- Facebook: **0.080**
<p align="center">
  <img src="https://github.com/user-attachments/assets/c76b2e12-96c7-4c53-8e27-43e86f13bcb9" width="520" />
  <br/>
  <sub><em>ROI by channel on the most recent 12-week holdout window (posterior mean). </em></sub>
</p>


**Marginal ROI (posterior mean):**
- TV: **0.108**
- Radio: **0.059**
- Print: **0.033**
- Facebook: **0.032**

<p align="center">
  <img src="https://github.com/user-attachments/assets/ccc59c50-d016-4d82-9850-17cb47f081cb" width="520" />
  <br/>
  <sub><em>Marginal ROI by channel on the most recent 12-week holdout window (posterior mean).</em></sub>
</p>


### Interpretation
- TV and Radio are the most efficient channels in this window (highest ROI), and they also have the strongest marginal ROI, meaning additional spend is expected to generate the largest incremental gains at the margin.
- Facebook has the lowest ROI and a low marginal ROI, suggesting it is the weakest candidate for incremental budget in this period.

### Actionable reallocation rule
A simple decision rule is to move budget from the lowest marginal-ROI channels toward the highest marginal-ROI channels until marginal ROIs begin to converge (i.e., the “next dollar” is equally valuable across channels).  
In this holdout window, that implies shifting budget away from Facebook (and possibly Print) and toward TV (first) and Radio (second). The what-if simulation below shows the approximate expected lift from shifting 10% of Facebook spend to TV/Radio in the holdout window.


### Simple what-if budget reallocation (back-of-the-envelope)

To make the MMM outputs decision-oriented, I ran a simple reallocation simulation on the most recent 12-week holdout window:

- Shift **10% of Facebook spend** to another channel  
- Approximate impact using: **ΔOutcome ≈ (ROI_to − ROI_from) × shifted_spend**
<img width="656" height="464" alt="image" src="https://github.com/user-attachments/assets/27af49b1-a299-4805-9ef9-884fafbc0bf6" />

Result: FB→TV yields the largest expected lift (≈96.9k), followed by FB→Radio (≈92.4k) in this holdout window.
For the Facebook → TV shift:
- Facebook holdout spend ≈ **6.44M**
- Shifted spend (10%) ≈ **0.644M**
- ROI(TV) − ROI(Facebook) ≈ **0.150**
- Expected ΔOutcome ≈ **96,904** (in outcome units)

This is an intentionally simple approximation (not a full optimizer), but it translates ROI results into a concrete, testable budget action.



## Data source
This project uses the **“MMM Weekly Data – Geo: India”** dataset from Kaggle:  
https://www.kaggle.com/datasets/subhagatoadak/mmm-weekly-data-geoindia

## Dataset description
The dataset is organized as a weekly, geo-level panel (multiple geographies observed over time) and includes multiple marketing/media variables along with outcome (KPI) variables suitable for marketing mix modeling.

It contains multiple brands, and for each brand, multiple SKUs (products). To keep the modeling scope focused and to produce interpretable results, I filtered the dataset to a single product line:

- **Brand:** Brand A  
- **SKU:** SKU 2

All Meridian model training and comparisons in this repository are based on this filtered Brand A / SKU 2 subset.


