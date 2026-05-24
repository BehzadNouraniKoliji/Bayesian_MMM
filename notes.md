Answering Questions About the Developed Model.

### Question 1. How do you model spend carry over?

In this project, carryover effects are modeled using a geometric adstock transformation applied to each channel's spend before it enters the saturation function.

The transformation works as follows: for each week, the effective spend is not just the current week's spend but a weighted sum of current and past spend, where the weights decay geometrically over time. Specifically, the weight for a spend that occurred *i* weeks ago is $\alpha^i$, where $\alpha$ is the decay parameter (between 0 and 1) estimated by the model for each channel.

A maximum lag of `l_max = 12` weeks is used, meaning spend can carry its effect forward up to 12 weeks. The transformation is implemented by creating 12 lagged copies of the spend series, stacking them, and computing their weighted sum using the geometric weights.

A higher $\alpha$ means slower decay — the channel's effect lingers longer. A lower $\alpha$ means the effect dissipates quickly after the spend occurs.

Since $\alpha$ is given a Beta(1,1) prior, the model learns the appropriate decay rate for each channel from the data, rather than it being fixed in advance.
