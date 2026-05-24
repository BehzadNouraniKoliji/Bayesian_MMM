Answering Questions About the Developed Model.

### Question 1. How to model spend carry-over?

In this project, carryover effects are modeled using a geometric adstock transformation applied to each channel's spend before it enters the saturation function.

The transformation works as follows: for each week, the effective spend is not just the current week's spend but a weighted sum of current and past spend, where the weights decay geometrically over time. Specifically, the weight for a spend that occurred *i* weeks ago is $\alpha^i$, where $\alpha$ is the decay parameter (between 0 and 1) estimated by the model for each channel.

A maximum lag of `l_max = 12` weeks is used, meaning spend can carry its effect forward up to 12 weeks. The transformation is implemented by creating 12 lagged copies of the spend series, stacking them, and computing their weighted sum using the geometric weights.

A higher $\alpha$ means slower decay — the channel's effect lingers longer. A lower $\alpha$ means the effect dissipates quickly after the spend occurs.

Since $\alpha$ is given a Beta(1,1) prior, the model learns the appropriate decay rate for each channel from the data, rather than it being fixed in advance.


### Question 2. Explain your choice of prior inputs to the model?

One of the unique features of Bayesian MMM is its ability to integrate prior knowledge into the modeling process. This could include insights from past marketing campaigns, industry benchmarks, or expert opinions. By combining this prior knowledge with data-driven insights, the Bayesian approach helps create a more robust model.

#### Intercept $a$ — $\mathcal{N}(0,\ 0.75)$

Captures baseline revenue with no marketing activity. A Normal prior is appropriate as the intercept can be positive or negative on the scaled revenue axis. $\sigma = 0.75$ is weakly informative, allowing reasonable flexibility around zero.

#### Trend $b_{\text{trend}}$ — $\mathcal{N}(0,\ 0.35)$

Models the linear growth or decline in revenue over time. A Normal prior allows both positive and negative trends. It is tighter than the intercept prior ($\sigma = 0.35$), reflecting the prior belief that trends are typically modest.

#### Seasonality $b_{\text{fourier}}$ — $\text{Laplace}(0,\ 0.4)$

Coefficients for the Fourier terms capturing recurring seasonal patterns. A Laplace prior is preferred over a Normal prior because its heavier tails and sharper peak at zero act as a sparsity-inducing prior — most Fourier modes should remain near zero, with only a few driving the seasonality signal.

#### Black Friday & January Sale $b_{\text{black\_friday}},\ b_{\text{january\_sale}}$ — $\mathcal{HN}(0.5)$

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
