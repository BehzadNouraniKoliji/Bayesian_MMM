## Answering Questions About the Developed Model.

### Question 1. How to model spend carry-over?

In this project, carryover effects are modeled using a geometric adstock transformation applied to each channel's spend before it enters the saturation function. The transformation works as follows: for each week, the effective spend is not just the current week's spend but a weighted sum of current and past spend, where the weights decay geometrically over time. Specifically, the weight for a spend that occurred *i* weeks ago is $\alpha^i$, where $\alpha$ is the decay parameter (between 0 and 1) estimated by the model for each channel. A maximum lag of `l_max = 12` weeks is used, meaning spend can carry its effect forward up to 12 weeks. The transformation is implemented by creating 12 lagged copies of the spend series, stacking them, and computing their weighted sum using the geometric weights. A higher $\alpha$ means slower decay — the channel's effect lingers longer. A lower $\alpha$ means the effect dissipates quickly after the spend occurs. Since $\alpha$ is given a Beta(1,1) prior, the model learns the appropriate decay rate for each channel from the data, rather than it being fixed in advance.

### Question 2. Explaining the choice of prior inputs to the model.

One of the unique features of Bayesian MMM is its ability to integrate prior knowledge into the modeling process. This could include insights from past marketing campaigns, industry benchmarks, or expert opinions. By combining this prior knowledge with data-driven insights, the Bayesian approach helps create a more robust model.

#### Intercept $a$ — $\mathcal{N}(0,\ 0.75)$

Captures baseline revenue with no marketing activity. A Normal prior is appropriate as the intercept can be positive or negative on the scaled revenue axis. $\sigma = 0.75$ is weakly informative, allowing reasonable flexibility around zero.

#### Trend $b_{\text{trend}}$ — $\mathcal{N}(0,\ 0.35)$

Models the linear growth or decline in revenue over time. A Normal prior allows both positive and negative trends. It is tighter than the intercept prior ($\sigma = 0.35$), reflecting the prior belief that trends are typically modest.

#### Seasonality $b_{\text{fourier}}$ — $\text{Laplace}(0,\ 0.4)$

Coefficients for the Fourier terms capturing recurring seasonal patterns. A Laplace prior is preferred over a Normal prior because its heavier tails and sharper peak at zero act as a sparsity-inducing prior — most Fourier modes should remain near zero, with only a few driving the seasonality signal.

#### Black Friday & January Sale $b_{\mathrm{black\ friday}},\ b_{\mathrm{january\ sale}}$ — $\mathcal{HN}(0.5)$

These events only produce positive revenue lifts, never negative, so a HalfNormal prior is appropriate. $\sigma = 0.5$ is weakly informative, allowing a substantial lift while not forcing one.

#### Adstock Decay $\alpha_{1\ldots7}$ — $\text{Beta}(1,\ 1)$

Controls the geometric decay rate of carryover effects per channel. A Beta distribution is the natural choice as $\alpha$ must lie between 0 and 1. $\text{Beta}(1,1)$ is a uniform prior over that interval, letting the data determine the decay rate for each channel without strong assumptions.

#### Saturation Rate $\lambda_{1\ldots7}$ — $\text{Gamma}(1,\ 1)$

Controls the curvature of the logistic saturation function — how quickly a channel reaches diminishing returns. A Gamma prior is appropriate because $\lambda$ must be strictly positive. $\text{Gamma}(1,1)$ is weakly informative, allowing a wide range of saturation shapes.

#### Channel Coefficients $b_{z_1\ldots z_7}$ — $\mathcal{HN}(\sigma)$

Represent each channel's maximum contribution to revenue after adstock and saturation transformations. A HalfNormal prior is used because channel effects can only be positive — marketing spend cannot decrease revenue.

The $\sigma$ values are set individually per channel, reflecting their multicollinearity structure and spend magnitude:

- **$z_6$**: loosest ($\sigma = 0.25$) — most identifiable, highest credibility.
- **$z_1,\ z_5$**: tightened ($\sigma = 0.05,\ 0.08$) — correlated with others, prone to credit stealing.
- **$z_3,\ z_4,\ z_7$**: moderate ($\sigma = 0.05$) — collinear culprits, balanced between shrinkage and flexibility.
- **$z_2$**: tightest ($\sigma = 0.007$) — tiny spend, highly correlated with $z_7$, strong skepticism applied.

In cases where a channel exhibits high posterior uncertainty, unstable ROAS, low spend, and strong correlation with a high-spend channel, tightening the prior reflects skepticism unless the data provides strong evidence otherwise.

#### Observation Noise $\sigma$ — $\mathcal{HN}(0.05)$

Controls the spread of the likelihood around the mean prediction. A HalfNormal prior is used because noise must be positive. The tight prior $\sigma = 0.05$ reflects the expectation that the model should fit the scaled revenue closely.

#### Degrees of Freedom $\nu$ — $\text{Gamma}(25,\ 2)$

Parameter of the Student-T likelihood controlling tail heaviness. $\text{Gamma}(25,2)$ has a mean of $12.5$, pushing $\nu$ toward higher values where the Student-T distribution approaches a Normal distribution, while still allowing heavier tails to robustly handle the revenue outliers identified in the data.

#### Point: Posterior Predictive Check (as one way of evaluating the goodness of the developed model)

After sampling, `pm.sample_posterior_predictive()` generates revenue predictions using the posterior parameter distributions. This is used to assess model fit — specifically, whether the posterior predictive distribution adequately covers the observed revenue signal. It is also used to run counterfactual simulations for ROAS and ROI estimation by zeroing out individual channels and comparing predicted revenue against the baseline. The key distinction is that prior sampling reflects only what the model assumed before seeing data, while posterior sampling reflects what the model learned after observing the data. The gap between the two indicates how much information the data contributed beyond the priors.

### Questions 3. What are the main insights in terms of channel performance/effects? 

Looking at both the ROI distributions and the response curves together in the Jupyter Notebook:

#### $z_6$ — Best Performing Channel

This is the only channel with a clearly positive mean ROI (`1.305`). The response curve also shows a strong, near-linear response to spend with a wide spend range, including near-zero periods. This makes $z_6$ the most reliable and identifiable channel in the model. The evidence strongly suggests increasing budget allocation to this channel.

#### $z_1$ and $z_2$ — Unreliable Attribution

Both channels have HDIs spanning massively from negative to positive values (for example, $z_2$: `-23` to `+23`). Although the response curves appear reasonable, the extreme uncertainty in ROI makes budget decisions based on these channels unreliable. This is consistent with the multicollinearity and low-spend identification problem diagnosed earlier in the Jupyter Notebook. To improve identification, deliberately pausing the $z_2$ channel for several weeks would introduce the independent variation required for the model to better separate the contributions of $z_1$ and $z_2$.

#### $z_3$ — Likely Negative ROI

This channel has a mean ROI of `-0.685`, and the upper bound of the HDI barely reaches zero (`0.076`), suggesting it is highly likely to destroy value. The response curve shows a narrow spend range combined with a high baseline contribution, indicating that the model struggles to separate its impact from the baseline trend — likely because the channel rarely or never drops to zero spend. A blackout test would be especially valuable for validating this channel's true effect.

#### $z_4$ and $z_5$ — Likely Negative ROI

Both channels have negative mean ROI estimates and HDIs that mostly remain below zero. Although the response curves show reasonable spend variation, the model cannot confidently attribute positive incremental revenue to either channel.

#### $z_7$ — Negative ROI with Strong Carryover Effects

This channel has a mean ROI of `-0.511` with a relatively tight HDI (`-1.211` to `0.280`). The response curve indicates a high baseline contribution even at low spend levels, suggesting strong carryover/adstock effects. However, the marginal return from additional spend appears poor, implying potential overspending on this channel.

#### Overall Strategic Takeaway

- Concentrate incremental budget on $z_6$.
- Treat $z_1$ and $z_2$ cautiously until blackout tests provide cleaner identification.
- Seriously reconsider investment in $z_3$, $z_4$, $z_5$, and $z_7$ due to their likely negative marginal returns.
- Use the saturation curves to define upper spend limits for each channel in future media planning.
