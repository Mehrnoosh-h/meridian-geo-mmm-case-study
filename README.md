# meridian-geo-mmm-case-study
Multicollinearity check (what it is and why it matters)

Multicollinearity happens when two or more predictor variables (here: media channels) move together so strongly that the model can’t reliably separate their individual effects. In MMM, this usually shows up as unstable channel coefficients/contributions: the total marketing effect can still be estimated well, but the model may “swap credit” between correlated channels (e.g., YouTube vs Instagram), making channel-level ROI and budget recommendations less trustworthy.

How we detected multicollinearity

We used Variance Inflation Factor (VIF) to quantify it. VIF measures how much the variance of a coefficient is inflated due to correlation with other predictors:

VIF ≈ 1: little to no correlation with other predictors

VIF 5–10: concerning multicollinearity

VIF > 10: strong multicollinearity (interpretation of individual effects becomes unreliable)

In our data (one selected brand + one SKU), VIF values were extremely large:

YouTube: 1.23e+14

Instagram: 1.32e+14

Facebook: 8.68e+12

These values indicate near-perfect collinearity between the channels (practically the same signal), so keeping them as separate regressors would make channel attribution and ROI estimates unstable.

Options to address multicollinearity in MMM

Common approaches include:

Combine correlated channels

Create a composite feature (e.g., “Video+Social”) by summing spend/impressions across the correlated channels.

Pros: keeps the overall marketing signal; reduces instability; attribution becomes stable at the group level.

Cons: you lose separate ROI for each channel within the combined group.

Remove one or more correlated channels

Drop channels that duplicate the same signal, especially if they’re minor or redundant for the business question.

Pros: simplest; improves identifiability.

Cons: you may drop real signal and shift credit to remaining channels, potentially biasing channel-level conclusions.

Regularization / stronger priors (Bayesian shrinkage)

Encourage the model to avoid extreme attributions under collinearity.

Pros: can help stability without dropping features.

Cons: with near-perfect collinearity, regularization often can’t fully fix identifiability—credit splitting may still be arbitrary.

Use experimental calibration / external priors

If you have lift tests or geo experiments for a channel, you can anchor its effect.

Pros: best for true channel-level attribution.

Cons: requires extra data.

What we did in this project (two scenarios)

Because YouTube/Instagram/Facebook were extremely collinear, we tested two practical scenarios:

Scenario A — Combine correlated channels
We combined the highly correlated media channels into a single grouped channel and fit Meridian.
Model reviewer summary (combined):

Overall status: PASS

R² ≈ 0.9496, wMAPE ≈ 0.1186

Baseline check: REVIEW (posterior P(baseline < 0) = 0.54)

Scenario B — Remove Instagram and YouTube
We removed two of the three highly collinear channels and refit.
Model reviewer summary (removed):

Overall status: PASS

R² ≈ 0.9490, wMAPE ≈ 0.1192

Baseline check: REVIEW (posterior P(baseline < 0) = 0.37)
