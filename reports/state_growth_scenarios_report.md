# State growth scenarios and repricing impact

#### Maria Silva — November 2025

The goal of this analysis is to understand how the Ethereum state (database size per full node) can evolve under increased block gas limits and how repricing (changing effective gas costs for state bytes and burst resources) affects both state growth and burst throughput.

To achieve this, we:

- Project state size under three gas-limit trajectories (base, aggressive, conservative).
- Build and calibrate a simple isoelastic demand model for state creation and burst resource consumption.
- Use that model to compute the new EIP-1559 equilibrium base fee and resulting per-block resource shares when the gas limit and/or the gas price for adding state changes.
- Report scenario comparisons across ranges of price elasticities for state and burst demand.

### TLDR

- The state can grow substantially if throughput increases and state creation costs remain relatively cheap. By the middle of 2027, the conservative schedule reaches a total state size of 686 GiB, the base schedule 859 GiB, and the aggressive schedule 1.08 TiB. In all cases, this is above the critical 650 GiB threshold identified by the [boatnet initiative](https://cperezz.github.io/bloatnet-website/index.html).
- We need to align on what the acceptable level of state growth is for Ethereum in the medium and long term.
- Raising the gas cost for state creation reduces the expected state growth across the explored elasticity regimes, but it slightly reduces burst-resource throughput.
- The throughput loss is more impacted by the burst-demand elasticity than state creation costs. When burst demand is price‑elastic, doubling capacity still yields near‑linear throughput gains and repricing has only a small effect.

Numeric summary (median / 90th percentile across explored elasticity grid):

## 1. Why do we care about state growth?

As of May 2025, the current uncompressed database size in a Geth node dedicated to state is ~340 GiB. After increasing the gas limit from 30M to 36M gas units, the median size of new states created each day doubled, from ~102 MiB to ~205 MiB.

<p align="center">
  <img src="figures/state_growth_scenarios_report/state_growth_since_2024.png" alt="state_growth_since_2024" style="width:700px;"/>
</p>

The relationship we are seeing in this example is not linear as expected. This is likely due to other factors impacting user behavior. However, all else being equal, we expect a proportional increase in the number of new states created as gas limits increase.

High state growth rates have two downstream impacts:

1. If state grows faster than disk improvements and the decrease in storage per byte, we will eventually reach a point where validators get priced out and cannot keep up with the disk requirements needed to run an Ethereum validator node.
2. The performance of state access operations such as `SSTORE` significantly depends on the size of the state, with larger states leading to worse execution times for these operations. The [boatnet initiative](https://cperezz.github.io/bloatnet-website/index.html) identified critical bottlenecks in state access patterns with a state size of 650 GiB, resulting in 40% increases in state access times, exponentially higher memory consumption, and longer sync times.

It is still unclear how hardware improvements and client optimizations will affect these bottlenecks; however, given the current focus on scalability and the expected increase in the gas limit, this is an issue we need to look into more carefully. Above all, we need to align on what the acceptable level of state growth for Ethereum is in the medium and long term.

However, in this report, we focus on a slightly different question: how the gas limit is expected to affect state growth, and how gas repricings can alter its trajectory. This is especially relevant for [EIP-8037](https://eips.ethereum.org/EIPS/eip-8037), which is proposed for the upcoming Glamsterdam fork.

## 2. State growth scenarios under increasing block limits: how big can state get?

To answer this question, we designed three different schedules for gas limit increases from 2025-05-01 through mid-2027:

| Date       | Conservative | Base | Aggressive |
|-----------:|-------------:|-----:|-----------:|
| 2025-05-01 |           36 |   36 |         36 |
| 2025-07-01 |           45 |   45 |         45 |
| 2025-12-01 |           60 |   60 |         60 |
| 2026-06-01 |           80 |  100 |        150 |
| 2026-12-01 |          150 |  300 |        500 |
| 2027-06-01 |          200 |  400 |        700 |

Then, for each schedule, we compute the corresponding state growth rate, assuming a proportional relationship between state growth per day and gas limit and a baseline measured state creation rate of 205 MiB/day at a 36M gas limit:

`new_state_per_day = 205 MiB * (G / 36M)`

With this, we get the following state growth rates per day at each gas limit change day:

| Date       | Conservative (MiB/day) | Base (MiB/day) | Aggressive (MiB/day) |
|-----------:|-----------------------:|---------------:|---------------------:|
| 2025-05-01 |                    205 |            205 |                  205 |
| 2025-07-01 |                    256 |            256 |                  256 |
| 2025-12-01 |                    342 |            342 |                  342 |
| 2026-06-01 |                    456 |            569 |                  854 |
| 2026-12-01 |                    854 |           1708 |                 2847 |
| 2027-06-01 |                   1139 |           2278 |                 3986 |

The next plot shows total state size in GiB over the next 2 years for the three gas schedules.

<p align="center">
  <img src="figures/state_growth_scenarios_report/state_size_by_schedule.png" alt="state_size_by_schedule" style="width:700px;"/>
</p>

Under proportional scaling (keeping the same gas composition), increasing the block limit linearly increases the daily state creation (because we scale the measured 205 MiB/day by the gas limit ratio). Because daily additions compound, the total state size diverges over months and years between gas schedules. By the middle of 2027, the conservative schedule reaches a total state size of 686 GiB, the base schedule 859 GiB, and the aggressive schedule 1.08 TiB. In all cases, this is above the critical 650 GiB threshold identified by the [boatnet initiative](https://cperezz.github.io/bloatnet-website/index.html).

## 3. How does repricing impact state growth and throughput?

Now that we have identified potential scenarios of state growth under no pricing changes, we can analyze how increasing the gas price for state creation operations affects state growth and scalability.

### 3.1 Model and derivation

We start by defining a simple model that allows us to estimate the average daily state growth and the amount of gas available in a block for burst operations (compute, state access, and data).

#### Model inputs

- $G$: The block gas limit.
- $b$: The equilibrium base fee $b$ in gwei.
- $g_\text{state}$: The gas cost per new byte added to the state byte.
- $g_\text{burst}$: The gas costs for each second of burst resource usage.
- $S(p)$: The demand curve for state creation, measured as the total number of net state bytes willing to be added in a block at the price $p$. Note that $p$ is cost in gwei of adding one byte to state and is computed as $p=b\cdot g_\text{state}$.
- $B(p)$: The demand curve burst resource usage, measured as the number of slot time in seconds willing to be added in a block at the price $p$. In this case, $p=b\cdot g_\text{burst}$.

#### Model assumptions

- The base fee follows the EIP-1559 design, where block usage above 50% of $G$ leads to an increase in the base fee for the next block, wile a block usage below 50% of $G$ leads to a decrease in the base fee for the next block.
- With current pricing, blocks occupy on average 50% of $G$, with 30% of that gas allocated to state creation and the remaining to burst resources.
- The average block is achieved at equilibrium, when 50% of $G$ is sufficient to cover the demand for state creation and burst resource usage at the price $p$.
- Both demand curves are modelled with two independent isoelastic models:

$$
S(p)=A_s\,p^{-\varepsilon_s},\qquad
B(p)=A_b\,p^{-\varepsilon_b},\qquad
\varepsilon_s,\varepsilon_b>0
$$

Here, we assume independent isoelastic demands for tractability. In practice, however, demand for state creation and burst resources is likely correlated: a change in the effective price of state bytes will affect demand for burst seconds (and vice‑versa). A joint multivariate demand specification with cross‑price elasticities would better capture substitution or complementarity between resources, but it introduces additional parameters and identification challenges.

#### Scenario

- Gas limit increases by a factor of $n$, i.e.,  $G = n \, G^0$
- State gas costs increases by a factor of $m$, i.e., $g_\text{state} = m \, g_\text{state}^0$

We expect the base fee to decrease until reaching a new equilibrium where both more state and more burst resources are used.

#### Base derivation

Let the per-block equilibrium usage at price $p$ be

- State bytes: $S(p)=S(b g_\text{state})$
- Burst seconds: $B(p)=B(b g_\text{burst})$

The gas used in a block at base fee b is defined as:

$$
U(b)=g_\text{state}\,S(b\,g_\text{state})\;+\;g_\text{burst}\,B(b\,g_\text{burst})
$$

For the scenario, EIP-1559 drives $b$ toward the value $b^*$ such that:

$$
U(b^*)=0.5\, n\, G_\text{limit}^0
$$

At this new equilibrium, we get the following per-block usage:

- $S^* = S(b^* \, m\, g_\text{state}^0)$, which leads to an average state growth of $2,628,000\cdot S^*$ bytes per year
- $B^* = B(b^* g_\text{burst}^0)$

and the following shares per resource type:

- $\text{Share}_\text{state}=\frac{m\,g_\text{state}^0S^*}{0.5\, n\, G_\text{limit}^0}$
- $\text{Share}_\text{burst}=1-\text{Share}_\text{state}$

#### Calibration

With current $G=G^0$, equilibrium $b=b^0$ and initial gas prices $g_\text{state}^0$ and $g_\text{burst}^0$, we get:

- $g_\text{state}^0 S(b^0 g_\text{state}^0)= 0.3 \cdot 0.5 \cdot G^0 = 0.15\, G^0$
- $g_\text{burst}^0 B(b^0 g_\text{burst}^0)= 0.7 \cdot 0.5 \cdot G^0 = 0.35\, G^0$

With this, we can calibrate $A_s$ and $A_b$ as:

- $A_s = S(p)p^{\varepsilon_s} = \frac{0.15\,G^0}{g_\text{state}^0}\,(b^0g_\text{state}^0)^{\varepsilon_s}$
  
- $A_b = B(p)p^{\varepsilon_b} = \frac{0.35\,G^0}{g_\text{burst}^0}\,(b^0g_\text{burst}^0)^{\varepsilon_b}$

Under this choice, for any $b$ we get the following formula for gas used in the block under equilibrium:

$$
U(b)=
0.15\,G^0 \Big(\frac{g_\text{state}}{g_\text{state}^0}\Big)^{1-\varepsilon_s} \Big(\frac{b}{b^0}\Big)^{-\varepsilon_s}
\;+\;
0.35\,G^0 \Big(\frac{g_\text{burst}}{g_\text{burst}^0}\Big)^{1-\varepsilon_b} \Big(\frac{b}{b^0}\Big)^{-\varepsilon_b}
$$

We can also estimate $g_\text{burst}^0$ from gas limit and the current state growth rate. Here we are taking an average cost assuming a 50% gas limit utilization:

$$
g_\text{state}^0 \, S^0 = 0.15\, G_0 \implies g_\text{state}^0 = \frac{0.15\, G^0}{S^0}
$$

For the burst resources, we will price based on the worse case block, i.e., assume that if the block was full of burst resource operations until the block limit, it would take 2 seconds to execute:

$$
g_\text{burst}^0 \, 2  = G^0 \implies g_\text{burst}^0 = \frac{G^0}{2}
$$

From this, we can directly derive the base throughput for burst resources:

$$
g_\text{burst}^0 \, B^0 = 0.35\,G^0 \implies \frac{G^0}{2} \, B^0 = 0.35\,G^0 \implies B^0 = 0.7 \, \, \text{seconds}
$$

#### Scenario derivation

In the scenario, $G=n\,G^0$, $g_\text{state}=m\,g_\text{state}^0$ and $g_\text{burst}=g_\text{burst}^0$, which makes the total gas used in the block under equilibrium be:

$$
U(b)=
0.15\,G^0\, m^{1-\varepsilon_s} \Big(\frac{b}{b^0}\Big)^{-\varepsilon_s}
\;+\;
0.35\,G^0 \Big(\frac{b}{b^0}\Big)^{-\varepsilon_b}
$$

Also, at equilibrium, we are using 50% of the new gas limit so:

$$
U(b)= 0.5\cdot n\cdot G^0 \iff
$$

$$
0.15\,G^0\, m^{1-\varepsilon_s} \Big(\frac{b}{b^0}\Big)^{-\varepsilon_s}\;+\; 0.35\,G^0 \Big(\frac{b}{b^0}\Big)^{-\varepsilon_b} = G^0 \iff
$$

$$
0.15\, m^{1-\varepsilon_s} \Big(\frac{b}{b^0}\Big)^{-\varepsilon_s}\;+\; 0.35 \Big(\frac{b}{b^0}\Big)^{-\varepsilon_b} = 1
$$

We can compute the roots of this polynomial to find the equilibrium base fee $b^*$. If the polynomial has a root $r \in \mathopen] 0,1\mathclose[$, then $b^* = b^0 \, r$. From this, we can compute the per-block usages and gas shares as defined in the base derivation.

In the following sections, we consider the impact of three scenarios:

- Base (no repricing): $n=2$ (double gas limit), $m=1$ (state gas cost unchanged)
- Double state gas prices: $n=2, m=2$
- Triple state gas prices: $n=2, m=3$

In all cases, we assume that the gas limit doubles ($n=2$).

For each scenario, we sweep $\varepsilon_s$ and $\varepsilon_b$ across a grid (0.1 to 1.5) and solve for $b^*$. With the equilibrium base fee $b^*$, we then compute $S^*$, $B^*$, the annual state growth in GiB, and the resource gas shares.

### 3.2 Impact of repricing on the share of gas used by state creation

The following plots show the share of gas used in the equilibrium block by state creation operations for all price elasticity combinations $(​\varepsilon_s, \varepsilon_b)$.

<div style="display:flex;gap:12px;flex-wrap:wrap;">
  <div style="flex:1 1 30%;min-width:300px;">
    <img src="figures/state_growth_scenarios_report/share_state_m1.png" alt="State share m1" style="width:100%;height:auto;" />
    <p style="font-size:0.9em;margin:6px 0;">State gas share (n=2, m=1).</p>
  </div>
  <div style="flex:1 1 30%;min-width:300px;">
    <img src="figures/state_growth_scenarios_report/share_state_m2.png" alt="Annual growth m1" style="width:100%;height:auto;" />
    <p style="font-size:0.9em;margin:6px 0;">State gas share (n=2, m=2).</p>
  </div>
  <div style="flex:1 1 30%;min-width:300px;">
    <img src="figures/state_growth_scenarios_report/share_state_m3.png" alt="Annual growth m1" style="width:100%;height:auto;" />
    <p style="font-size:0.9em;margin:6px 0;">State gas share (n=2, m=3).</p>
  </div>
</div>

With $m = 1$ (no repricing) and $n = 2$ (double gas limit), the share of block gas consumed by state creation increases relative to the current 30% when demand for state creation is more elastic than demand for burst resources. Thus, if state creation adapts more to changing prices than to burst resource usage, there is a risk that scaling up gas limits shifts more block capacity to state creation, as more state-creation demand will flood the block to take advantage of cheaper base fees.

Increasing $m$ (making state creation more expensive) impacts the block share dedicated to state creation in the low-elasticity regimes of state and the high-elasticity regimes of burst resources:

- When demand for state creation has low-elasticity (the left side of the plots), we see a higher share of gas dedicated to state creation operations, with increasing magnitude the larger $m$ is.
- Conversely, when both demands have high elasticity (top-right corner), the effect is reversed, with a decreasing portion of the block dedicated to state creation when compared with the $m=1$ scenario. The magnitude of the change depends again on $m$.
- The cases with the highest share of state creation occur when demand for burst resources has low elasticity (bottom side of the plot). In this case, increasing the cost of state creation has no effect.

In conclusion, increasing the gas costs of state creation may have a positive or negative effect on the share of the equilibrium block dedicated to state creation. This depends on the elasticity regime. Yet, for the regimes with the highest share of state (when burst resources have low demand elasticity), increasing the gas costs of state creation does not impact this share.

### 3.3 Impact of repricing on the annual state growth

The following plots show the annual state growth for all price elasticity combinations $(​\varepsilon_s, \varepsilon_b)$. Here we assume that the average block is the equilibrium block. This metric is key to understand how effective increasing the cost of state creation operations is at mitigating state growth.

<div style="display:flex;gap:12px;flex-wrap:wrap;">
  <div style="flex:1 1 30%;min-width:300px;">
    <img src="figures/state_growth_scenarios_report/state_growth_m1.png" alt="State share m1" style="width:100%;height:auto;" />
    <p style="font-size:0.9em;margin:6px 0;">Annual state growth (n=2, m=1).</p>
  </div>
  <div style="flex:1 1 30%;min-width:300px;">
    <img src="figures/state_growth_scenarios_report/state_growth_m2.png" alt="Annual growth m1" style="width:100%;height:auto;" />
    <p style="font-size:0.9em;margin:6px 0;">Annual state growth (n=2, m=2).</p>
  </div>
  <div style="flex:1 1 30%;min-width:300px;">
    <img src="figures/state_growth_scenarios_report/state_growth_m3.png" alt="Annual growth m1" style="width:100%;height:auto;" />
    <p style="font-size:0.9em;margin:6px 0;">Annual state growth (n=2, m=3).</p>
  </div>
</div>

Unsurprisingly, increasing the costs of state-creation operations decreases the state growth rate across all elasticity regimes compared with the scenario without repricing. This indicates that increasing the gas cost of state creation is effective at mitigating state growth across a wide range of price elasticities.

Increasing the gas cost of state creation has another interesting effect. We can observe it by plotting the annual state growth against the state demand elasticity $\varepsilon_s$ for each scenario.

<p align="center">
  <img src="figures/state_growth_scenarios_report/state_growth_all.png" alt="state_growth_all" style="width:700px;"/>
</p>

- No reprice: higher $\varepsilon_s \implies$ more state growth
- Reprice equal to the increase in block limit: higher $\varepsilon_s \implies$ no change on the median state growth
- Reprice higher than the increase in block limit: higher $\varepsilon_s \implies$ less state growth

We should note that the demand elasticity of burst resources also affects state growth: the lower the elasticity, the higher the state growth rate across all scenarios.

### 3.4 Impact of repricing on the throughput of burst resources

The following plots show the throughput gains on burst resources in the equilibrium block for all price elasticity combinations $(​\varepsilon_s, \varepsilon_b)$. The gain is computed over the 0.7-second baseline throughput.

This metric is essential for measuring the impact on scalability of increasing state-creation costs. If we double the available gas, do we still observe at least double the throughput for burst resources? Recall that the baseline throughput is 0.7 seconds.

<p align="center">
  <img src="figures/state_growth_scenarios_report/burst_throughput_m1.png" alt="burst_throughput_m1" style="width:300px;"/>
  <img src="figures/state_growth_scenarios_report/burst_throughput_m2.png" alt="burst_throughput_m2" style="width:300px;"/>
  <img src="figures/state_growth_scenarios_report/burst_throughput_m3.png" alt="burst_throughput_m3" style="width:300px;"/>
</p>

In all three scenarios, the dominant driver of burst-throughput gains is the burst-side price elasticity, $\varepsilon_b$. As $\varepsilon_b$ rises, we see an increasing gain over the 0.7-s baseline:

- When burst demand is sufficiently price-elastic (top side of the plots), doubling capacity translates to ~double (or slightly more) burst throughput.
- When burst demand is inelastic (bottom side of the plots), doubling capacity translates to less than double the burst throughput.

By contrast, the horizontal variation with state elasticity $\varepsilon_s$ is more complex. Gains decrease slightly as $\varepsilon_s$ increases when $m = 1$. As for $m=2$ or $m=3$, the highest throughput gains on burst resources are achieved when both price elasticities are high.

In the following plot, we can see more clearly the impact of raising state creation costs on throughput. In general, higher costs slightly decrease the throughput gains. However, the effect is small when compared with the impact of the demand elasticity of burst resources. If we are in a regime with high $\varepsilon_b$, then increasing state creation affects throughput by less than 25%.

<p align="center">
  <img src="figures/state_growth_scenarios_report/burst_throughput_all.png" alt="burst_throughput_all" style="width:700px;"/>
</p>

## 4. Discussion and next steps

The first takeaway is that, under our simplified model, increasing the cost of state-creation operations effectively mitigates state growth across all price elasticity regimes. However, this still comes at a cost of slightly lower throughput gains for burst resources. This effect is compounded by the elasticity of demand for burst resources — the more elastic they are, the higher the through gains achieved across all pricing scenarios.

There are a couple of follow-ups we should do after this:

1. Consider a pricing scenario where state creation costs are metered independently of burst resources. This would be akin to doing [multidimensional metering](https://eips.ethereum.org/EIPS/eip-8011) with only two resources. [EIP-8075](https://eips.ethereum.org/EIPS/eip-8075) also does something similar.
2. Do more empirical measurements of the price elasticity of the various resources. We already have [an initial analysis](https://github.com/misilva73/evm-gas-repricings/blob/main/notebooks/0.4-state_price_elasticity.ipynb) for state creation, but burst resources are still missing.
3. Align on what is the "safe" level of state growth in the medium term, so we can correctly set a target for it.
4. Repricing is only one lever for solving state growth. Combining it with better state management (compaction, pruning strategies, caching, ephemeral storage layers, etc.) will further mitigate state growth-related bottlenecks.
