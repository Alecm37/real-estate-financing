# Service 3: Climate Pricing and Price Discovery, explained as a pipeline

*Market backdrop: mid-October 2026. All property-level and portfolio numbers below are one worked example run end to end on simulated data; they are illustrative, not quotes of any real parcel, flood map, or insurer.*

---

## 0. What we are estimating, and why it needs five models

### The object being estimated

A house's price already reflects some of its exposure to floods, hurricanes, and wildfire, but usually not all of it. This service splits a home's value into three pieces:

- **(a) Climate-blind value** $V^0_i$: what home $i$ would sell for if it carried no climate risk at all, holding everything else (size, location amenities, condition) fixed.
- **(b) Full cost of the climate risk** $\text{PV}_i$: the **present value** (today's dollar equivalent of a stream of future costs, discounted for time) of the damage the home can expect to suffer, plus the cost of bearing or insuring that damage, over the home's life.
- **(c) The part of that cost the market already prices**, $\theta\,\text{PV}_i$, where $\theta$ (the **pass-through** or **capitalization rate**) is the share of the risk cost that actually shows up as a lower sale price.

From those three pieces come the two numbers clients act on:

$$V^{\text{mkt}}_i = V^0_i - \theta\,\text{PV}_i \quad (\text{what it sells for today}), \qquad V^{*}_i = V^0_i - \text{PV}_i \quad (\text{climate-adjusted fair value}),$$

$$U_i \equiv V^{\text{mkt}}_i - V^{*}_i = (1-\theta)\,\text{PV}_i \quad (\text{unpriced risk, the mispricing}).$$

$U_i$ is the amount by which the market overvalues the home relative to its true risk. The service also forecasts **how fast** that gap closes once something forces the risk into view (a new flood map, an insurance repricing, a disaster), and what a whole portfolio of such homes stands to lose.

### Why it is worth estimating now

- Homeowners insurance premiums are up roughly 64% since 2021, and FEMA's NFIP (National Flood Insurance Program) moved to **Risk Rating 2.0** pricing in 2021–2022, which prices each property's own flood risk and phases premiums up toward that level with annual increases capped at 18% for primary residences. Climate risk is turning from an invisible cost into a visible line item, and visible costs get capitalized into prices.
- Home prices are roughly flat in nominal terms and falling in real terms. In a flat market, a climate repricing is not masked by general appreciation: it shows up as an outright loss.
- With the 30-year rate near 7.3%, buyers are payment-constrained, so a rising insurance bill reduces what they can pay for the house almost dollar for dollar in payment terms.
- Lenders and mortgage investors care because a climate repricing lowers collateral value, raising **LTV** (loan-to-value ratio, the loan balance divided by the home's value) on loans that looked safe at origination.

### Why five models instead of one

The obvious shortcut, regress price on a flood-risk score and read off the coefficient, fails for a specific reason: risky locations are usually also desirable locations. Homes near the water flood more *and* have better views. A naive regression mixes the amenity premium with the risk discount and can even get the sign wrong (we will show this happen in 3.3). So the problem is split into pieces that each have a clean method:

| Step | Model | Produces |
|---|---|---|
| 3.1 | Automated valuation: gradient boosting + repeat-sales index + spatial kriging + conformal intervals | Market value $V^{\text{mkt}}_i$ with a calibrated range |
| 3.2 | Hazard × depth-damage × probability integration | Expected annual loss $\text{EAL}_{i,h}$, tail loss, and $\text{PV}_i$ |
| 3.3 | Causal designs: within-block elevation regression, spatial regression discontinuity, event-study difference-in-differences | Pass-through $\theta$ by information regime |
| 3.4 | Decomposition + partial-adjustment panel + Monte Carlo | $V^0_i$, $V^{*}_i$, unpriced risk $U_i$ with a range, adjustment speed $\lambda$ and timeline |
| 3.5 | Correlated loss simulation (Gaussian copula) + scenario reruns | Portfolio expected loss, 1-in-100 loss, expected shortfall, repricing losses, concentration |

### The actual flow of numbers

```
Sales + assessor data ──► 3.1 AVM ──► market value V^mkt ───────────────────────┐
                              │                                                  │
                              └──► (shared: 2.1 escrow tax, 4.5 equity gap,      │
                                    5.2 current LTV)                              │
Hazard maps, elevation ──► 3.2 EAL by hazard ──► PV of risk cost ──┬────────────►├─► 3.4 split:
                              │                                     │            │    V⁰, V*, U,
                              └──► (shared: 2.3 insurance premium)  │            │    adjustment λ
Sales near map changes ──► 3.3 pass-through θ ◄── PV/P from 3.2 ────┘            │       │
                              (by regime: Zone X, AE, wind) ─────────────────────┘       ▼
                                                                       3.5 portfolio: correlated
                                                                       losses + repricing scenarios
                                                                                │
                                                                                ▼
                                                                             report
```

Two links deserve emphasis. First, **3.1 measures the market price, not the climate-blind value.** The AVM learns from actual sales, which already include whatever discount the market applies, so $V^{\text{mkt}}$ comes out of 3.1 and $V^0$ is reconstructed in 3.4 by adding the priced discount back. Second, **3.3 needs 3.2.** Pass-through is a ratio (observed price effect divided by the present value of the risk cost), so the causal estimate in 3.3 is meaningless without the engineering estimate of risk cost from 3.2 in its denominator.

### Timeline used in the worked example

- **Today:** mid-October 2026.
- **Valuation date:** the AVM values the home as of today.
- **Flood map:** the home currently sits in FEMA flood **Zone X** (outside the mapped 1%-annual-chance floodplain). FEMA released a **preliminary** updated map for the county in spring 2026 that places the home in **Zone AE** (inside it). We assume the new map becomes **effective in the third quarter of 2028**, after the appeal period; that is the trigger date for the adjustment forecast in 3.4. (The map history here is illustrative.)
- **Risk horizon:** costs are treated as a perpetual stream, because a home's price capitalizes all future costs (each buyer resells to the next, who bears the later years).

### The running example: Property F

A single-family home in a coastal Florida county, about 1.0 km from the bay.

- 1,850 square feet, 3 bedrooms, 2 baths, built 1994, lot 0.22 acres, pool. Renovated kitchen in 2023.
- Last sold **March 2019** for **\$300,000**.
- Ground elevation is low; the **first-floor elevation** (height of the lowest finished floor) is **1.5 feet above the surrounding ground**, taken from an **elevation certificate** (a surveyor's form recording a building's elevations, used for flood insurance).
- Structure **replacement cost** (cost to rebuild, not market value) **\$280,000**; contents **\$84,000** (30% of structure, a standard assumption).
- Currently in Zone X, so the owner's mortgage does not require flood insurance, and the owner does not carry it.

The punchline we will derive: the market values the home at about **\$468,000**, but its full climate risk cost is worth about **\$90,000** in present value, of which the market prices only about **\$40,000**. The home is overvalued by about **\$49,000 (10.5%)**, with a plausible range of 7% to 17%. About \$19,000 of that gap is likely to close within a few years of the new flood map taking effect (half of it within about 1.7 years); the rest persists until insurance or disclosure forces it into view.

---

## Step 3.1: Automated valuation (gradient boosting + repeat-sales + spatial smoothing)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Arm's-length sale prices and dates | $P_j$, $t(j)$ | County recorder deeds (sale price from transfer tax or deed), MLS where licensed; joined to parcels by record linkage (4.1) |
| Property characteristics (living area, lot size, beds/baths, year built, pool, condition, distance to water, school zone, etc.) | $\mathbf{z}_i$ | County assessor (property appraiser) roll, permits, GIS layers |
| Coordinates | $(\text{lat}_i, \text{lon}_i)$ | Parcel centroids |
| Pairs of sales of the same home | $(j_1, j_2)$ | Recorder deeds, matched by parcel ID |

**Terms.** An **AVM** (automated valuation model) is a statistical model that estimates a property's market value without a human appraiser. An **arm's-length** sale is one between unrelated parties at market terms (excluding, for example, transfers between family members or foreclosure auctions, which are filtered out). A **hedonic** model explains price as the sum of the values of a home's characteristics. A **repeat-sales index** measures market-wide price change using only homes that sold more than once, so the home's own characteristics cancel out.

### Process: four pieces in sequence

**Piece 1: repeat-sales index (put all sales on today's price level).** Sales from 2019 and 2026 cannot be compared directly, because the whole market moved in between. For each home sold twice, at times $t_1 < t_2$,

$$\log\frac{P_{j_2}}{P_{j_1}} = \pi_{t_2} - \pi_{t_1} + e_j,$$

where $\pi_t$ is the log price index in period $t$ (normalized so the first period is $\pi_0 = 0$) and $e_j$ is the home-specific change (renovation, neglect, luck). Stack all pairs into a regression whose regressors are $-1$ in the column of the first sale's period and $+1$ in the column of the second sale's period, and solve by least squares. This is the **Case-Shiller method**. The actual Case-Shiller procedure adds a refinement: pairs with a long gap between sales are noisier (more time for home-specific change), so it regresses the squared residuals on the gap length and re-estimates with weights inversely proportional to that fitted variance (a three-stage weighted least squares).

Every sale is then **time-adjusted** to the valuation date $T$:

$$\log P^{\text{adj}}_j = \log P_j + \pi_T - \pi_{t(j)}.$$

**Piece 2: hedonic gradient boosting (cross-sectional value from characteristics).** On the time-adjusted sales, fit

$$\log P^{\text{adj}}_j = F(\mathbf{z}_j) + \epsilon_j, \qquad F(\mathbf{z}) = f_0 + \sum_{m=1}^{M} \nu\, f_m(\mathbf{z}),$$

where each $f_m$ is a small regression tree fit to the residuals left by the previous trees, $\nu$ is the learning rate (a small number like 0.05 that makes each tree take a cautious step), and $f_0$ is the average log price. Boosting is the right tool because value is full of interactions and thresholds (a pool adds more in some neighborhoods than others; square footage matters less above a certain size), and the model learns these without being told their form. Predictions are explained with **SHAP values**: an additive split of each prediction into a base value plus one contribution per feature, which sum exactly to $F(\mathbf{z}_i)$.

**Piece 3: spatial smoothing of residuals (kriging).** After boosting, residuals $\epsilon_j$ are still spatially correlated: homes on the same block share things the features miss (a view, a noisy road, a well-kept street). **Kriging** (Gaussian-process regression) predicts home $i$'s residual from its neighbors' residuals. Model the residual field as having covariance

$$\operatorname{Cov}(\epsilon_j, \epsilon_{j'}) = \sigma_s^2\, e^{-d_{jj'}/\rho} + \sigma_n^2\, \mathbf{1}[j = j'],$$

where $d_{jj'}$ is the distance between homes, $\sigma_s^2$ is the variance of the smooth spatial part, $\rho$ the **range** (how fast correlation fades with distance), and $\sigma_n^2$ the **nugget** (house-specific noise that does not spread to neighbors). These three are estimated by maximum likelihood on the residuals. With $\mathbf{K}$ the covariance matrix among the $J$ nearby sales and $\mathbf{k}$ the vector of covariances between home $i$ and each of them (spatial part only, since $i$'s own nugget is not shared), the kriged residual and its variance are

$$\hat\epsilon_i = \mathbf{k}^{\top}\mathbf{K}^{-1}\boldsymbol{\epsilon}, \qquad \operatorname{Var}(\epsilon_i - \hat\epsilon_i) = \sigma_s^2 - \mathbf{k}^{\top}\mathbf{K}^{-1}\mathbf{k}.$$

The weights $\mathbf{w} = \mathbf{K}^{-1}\mathbf{k}$ give closer neighbors more say, give less to neighbors that are redundant with each other, and sum to less than one, so the estimate shrinks toward zero when neighbors are far or noisy.

The final point estimate is

$$\log V^{\text{mkt}}_i = F(\mathbf{z}_i) + \hat\epsilon_i.$$

**Piece 4: conformal prediction interval.** An AVM must say how wrong it might be, and the range must be honest. **Split conformal prediction** does this without distributional assumptions. Hold out a **calibration set** of $n$ recent sales not used in fitting. For each, compute the absolute error $|\log P_j - \log \hat V_j|$. Let $\hat q$ be the $\lceil (n+1)(1-\alpha) \rceil$-th smallest of those errors. Then

$$\Big[\, \hat V_i\, e^{-\hat q},\;\; \hat V_i\, e^{+\hat q} \,\Big]$$

contains the true sale price with probability at least $1-\alpha$, provided future sales resemble the calibration sales (statisticians call this **exchangeability**). We use $1-\alpha = 0.80$, giving a P10 to P90 range. In practice the interval is made **locally adaptive** by dividing errors by a fitted measure of difficulty (unusual homes get wider intervals), but the logic is the same.

### Worked example

**Repeat-sales index (toy version, three periods).** Periods 0 = 2019, 1 = 2022, 2 = 2026. Six repeat-sale pairs with log price ratios: two pairs 2019→2022 (0.40, 0.46), two pairs 2022→2026 (−0.05, 0.01), two pairs 2019→2026 (0.38, 0.43). The unknowns are $\pi_1, \pi_2$. The design rows are $(1, 0)$ for 0→1, $(-1, 1)$ for 1→2, and $(0, 1)$ for 0→2, so the normal equations are

$$\begin{pmatrix} 4 & -2 \\ -2 & 4 \end{pmatrix}\begin{pmatrix} \pi_1 \\ \pi_2 \end{pmatrix} = \begin{pmatrix} 0.90 \\ 0.77 \end{pmatrix} \quad \Longrightarrow \quad \pi_1 = 0.4283, \quad \pi_2 = 0.4067.$$

Prices rose 53% from 2019 to 2022 ($e^{0.4283} = 1.535$) and then slipped about 2% by 2026, consistent with the flat-to-soft backdrop. Property F's March 2019 sale time-adjusts to

$$300{,}000 \times e^{0.4067} = \$450{,}500.$$

**Hedonic prediction.** The boosted model's prediction for F, split by SHAP contributions (in log points):

| Component | Contribution |
|---|---|
| Base value $f_0$ (county average log price, \$362,200) | 12.800 |
| Living area 1,850 sq ft | +0.105 |
| Two baths | +0.010 |
| Lot 0.22 acres | +0.015 |
| Built 1994 | −0.030 |
| 1.0 km to the bay | +0.085 |
| Pool | +0.040 |
| Renovated (2023 kitchen) | +0.010 |
| **Total $F(\mathbf{z}_F)$** | **13.035** ($e^{13.035} = \$458{,}200$) |

Notice the +0.085 for distance to the bay. That number is the market's *net* valuation of being near water: the amenity premium minus whatever flood discount the market applies. The AVM cannot separate the two, and it should not try. Its job is the market price. Separating them is 3.3's job.

**Kriging.** Four recent nearby sales have boosting residuals and distances from F:

| Neighbor | Distance (km) | Residual $\epsilon_j$ |
|---|---|---|
| 1 | 0.15 | +0.034 |
| 2 | 0.30 | +0.048 |
| 3 | 0.45 | −0.012 |
| 4 | 0.70 | +0.022 |

The fitted covariance parameters are $\sigma_s = 0.05$, $\rho = 0.5$ km, $\sigma_n = 0.04$. For neighbor 1, the covariance with F is $0.05^2 e^{-0.15/0.5} = 0.0025 \times 0.741 = 0.00185$. Doing the same for all four and building $\mathbf{K}$ from the pairwise distances among the neighbors (plus the nugget $0.04^2 = 0.0016$ on the diagonal), solving $\mathbf{K}\mathbf{w} = \mathbf{k}$ gives weights

$$\mathbf{w} = (0.352,\ 0.200,\ 0.140,\ 0.050), \qquad \textstyle\sum w = 0.74,$$

$$\hat\epsilon_F = 0.352(0.034) + 0.200(0.048) + 0.140(-0.012) + 0.050(0.022) = +0.021.$$

The block is selling about 2% above what characteristics alone predict, and F inherits most of that. Kriging standard deviation: $\sqrt{0.0025 - \mathbf{k}^{\top}\mathbf{w}} = 0.037$.

$$\log V^{\text{mkt}}_F = 13.035 + 0.021 = 13.056, \qquad V^{\text{mkt}}_F = \$467{,}900.$$

That is about 4% above the time-adjusted 2019 sale, which is sensible: the 2023 renovation and the block's recent strength are both post-2019.

**Conformal interval.** On a calibration set of 5,000 sales from the last two quarters, the 80% conformal quantile of absolute log errors is $\hat q = 0.072$:

$$\big[\,467{,}900\, e^{-0.072},\;\; 467{,}900\, e^{0.072}\,\big] = [\,\$435{,}400,\; \$502{,}800\,].$$

### Output

For each property: $V^{\text{mkt}}_i$ (point estimate), its 80% conformal interval, the SHAP breakdown, and a draw-able distribution (we use a lognormal with log standard deviation $\hat q / 1.2816 = 0.056$, where 1.2816 is the standard normal 90th percentile, so it reproduces the interval).

### Hand-off

To 3.3 as the price level $P$ that turns dollar risk costs into percentages ($\text{PV}/P$), to 3.4 as $V^{\text{mkt}}$, and to 3.5 as the portfolio's exposure values. As a **shared module**: to escrow 2.1 (the market value the tax rules engine needs), to loan transfers 4.5 (the equity gap a buyer must cover), and to MBS turnover 5.2 (current LTV, a key driver of whether owners can sell or refinance).

### Validation

Out-of-time: fit through a date, value the homes that sell in the following quarter, and report the median absolute percentage error and the share within ±10%. Check conformal **coverage**: about 80% of those sales should land inside their intervals, overall and within subgroups (price tier, flood zone, distance to water); undercoverage in one subgroup is a sign the interval needs to be locally adaptive. Check that the repeat-sales index matches published indices (FHFA HPI) at the county level.

---

## Step 3.2: Risk quantification (expected annual loss)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Flood water depth at the property for a set of return periods | $\text{WSE}_i(p)$ | FEMA NFHL (National Flood Hazard Layer) base flood elevations, storm-surge models (NOAA SLOSH), pluvial (rainfall) flood models, sea-level rise adjustments |
| Ground and first-floor elevation | $G_i$, $\text{FFE}_i$ | USGS 3DEP lidar elevation; elevation certificates; else imputed from foundation type and year built |
| Wind speed at return periods | $v_i(p)$ | Hurricane wind hazard models (ASCE 7 wind maps as a baseline; catastrophe models) |
| Depth-damage and wind-damage functions | $D^{\text{str}}_h(\cdot)$, $D^{\text{con}}_h(\cdot)$ | FEMA Hazus; USACE (Army Corps of Engineers) generic curves |
| Replacement cost of structure, value of contents | $S_i$, $C_i$ | Cost estimators (square footage × local cost per foot × quality), or Coverage A on the insurance declaration page |
| Discount rate, growth of risk cost, insurance loading | $r$, $g$, $\Lambda$ | Chosen assumptions, shown with sensitivity |

**Terms.** A **return period** $\text{RP}$ is the average time between events at least that severe; its reciprocal is the **annual exceedance probability** $p = 1/\text{RP}$, the chance in any year of an event at least that severe. The "100-year flood" has $p = 0.01$; it is not an event that happens once a century but one with a 1% chance every year (about a 26% chance over a 30-year mortgage). A **depth-damage curve** gives the fraction of a building's replacement cost destroyed by water of a given depth above its first floor. **Expected annual loss (EAL)** is the long-run average loss per year. **Tail loss** is the loss at a specified rare probability, such as the 1-in-100 loss.

### Process

**Step A: hazard to damage at each return period.** For flood, the depth inside the structure is the water depth at the house minus the first-floor height:

$$d_i(p) = \underbrace{\text{WSE}_i(p) - G_i}_{\text{depth at the house}} - \underbrace{(\text{FFE}_i - G_i)}_{\text{first floor above ground}}.$$

A negative $d$ still causes damage if water reaches the house (crawlspace, ductwork, HVAC, garage), which is why depth-damage curves start below zero. The dollar loss at probability $p$ is

$$L_{i,h}(p) = D^{\text{str}}_h\big(d_i(p)\big)\, S_i + D^{\text{con}}_h\big(d_i(p)\big)\, C_i,$$

with $h$ indexing hazard (flood $f$ or wind $w$). For wind the input is the gust speed $v_i(p)$ instead of depth.

**Step B: integrate over probability.** If $L(p)$ is the loss with exceedance probability $p$, the expected annual loss is the area under the **loss-exceedance curve**:

$$\text{EAL}_{i,h} = \int_0^1 L_{i,h}(p)\, dp.$$

Why this formula? The annual loss is a random variable $L$, and for any nonnegative random variable, $\mathbb{E}[L] = \int_0^\infty \Pr(L > \ell)\, d\ell$. Flipping the axes of that same area, it equals $\int_0^1 L(p)\,dp$, where $L(p)$ is the loss exceeded with probability $p$. With losses known at a handful of return periods $p_1 > p_2 > \dots > p_K$, approximate the integral by trapezoids, take the loss to be zero for $p$ above the most frequent damaging event, and hold it flat beyond the rarest:

$$\text{EAL}_{i,h} \approx \sum_{k=1}^{K-1} (p_k - p_{k+1})\,\frac{L(p_k) + L(p_{k+1})}{2} \;+\; p_K\, L(p_K).$$

The flat tail is conservative in one direction (it ignores even larger, rarer events) and generous in another (rare events can exceed the 500-year loss). It is a choice to state, not hide. Total EAL is the sum across hazards; tail losses are reported per hazard, and the combined tail is computed by simulation in 3.5, because quantiles do not add.

**Step C: from annual loss to present value of risk cost.** An owner bears climate risk either by insuring it (paying a premium above EAL to cover the insurer's expenses and cost of capital) or by self-insuring (bearing the losses plus the cost of the uncertainty). Either way the annual cost is the EAL scaled up by a **loading** $\Lambda > 1$:

$$k_{i,h} = \Lambda\, \text{EAL}_{i,h}.$$

That cost grows over time at rate $g$ (rebuilding costs rise with inflation; hazard frequency rises with sea level and warming). Discounting at rate $r$ over a perpetual horizon, using the **growing perpetuity** formula $\sum_{t=1}^{\infty} k(1+g)^{t-1}/(1+r)^t = k/(r-g)$,

$$\text{PV}_{i,h} = \frac{\Lambda\, \text{EAL}_{i,h}}{r - g} \equiv M\, \text{EAL}_{i,h}, \qquad M = \frac{\Lambda}{r - g}.$$

$M$ is the capitalization **multiplier**: one dollar of expected annual loss is worth $M$ dollars of house value.

### Worked example: flood

Property F's flood hazard model gives these depths at the house (above ground), with first floor 1.5 ft above ground. Damage fractions are approximately the USACE generic one-story, no-basement curves (structure: 2.5% at −1 ft, 13.4% at 0 ft, 23.3% at +1 ft, 32.1% at +2 ft; contents: 2.4%, 8.1%, 13.3%, 17.9%), interpolated linearly. $S = \$280{,}000$, $C = \$84{,}000$.

| RP (years) | $p$ | Depth at house (ft) | Depth $d$ vs. first floor (ft) | Structure damage | Contents damage | Loss $L(p)$ |
|---|---|---|---|---|---|---|
| 10 | 0.100 | 0.0 | −1.5 | 0 | 0 | \$0 |
| 25 | 0.040 | 0.4 | −1.1 | 2.25% | 2.16% | \$8,114 |
| 50 | 0.020 | 1.0 | −0.5 | 7.95% | 5.25% | \$26,670 |
| 100 | 0.010 | 1.7 | +0.2 | 15.38% | 9.14% | \$50,742 |
| 250 | 0.004 | 2.7 | +1.2 | 25.06% | 14.22% | \$82,113 |
| 500 | 0.002 | 3.5 | +2.0 | 32.10% | 17.90% | \$104,916 |

For example at the 100-year event: $0.1538 \times 280{,}000 + 0.0914 \times 84{,}000 = 43{,}064 + 7{,}678 = \$50{,}742$.

Trapezoids:

| Interval of $p$ | Width | Average loss | Area |
|---|---|---|---|
| 0.100 → 0.040 | 0.060 | \$4,057 | \$243.4 |
| 0.040 → 0.020 | 0.020 | \$17,392 | \$347.8 |
| 0.020 → 0.010 | 0.010 | \$38,706 | \$387.1 |
| 0.010 → 0.004 | 0.006 | \$66,428 | \$398.6 |
| 0.004 → 0.002 | 0.002 | \$93,515 | \$187.0 |
| below 0.002 (flat tail) | 0.002 | \$104,916 | \$209.8 |
| **EAL, flood** | | | **\$1,773.8** |

Notice where the EAL comes from: the four intervals between the 50-year and 500-year events contribute about equally. Rare, severe floods matter as much as moderately frequent ones. A model that only looked at "is the home in the 100-year floodplain" would miss most of this structure. Also notice that the 100-year water reaches 1.7 ft at the house, yet the current map puts the home outside the 100-year floodplain. That is the gap the preliminary map fixes, and the reason the market has underpriced it.

**Tail loss.** The 1-in-100 flood loss is $L(0.01) = \$50{,}742$.

### Worked example: wind

Gust speeds from the hurricane wind model, with an illustrative Hazus-style damage ratio for a 1994 hip-roof house (contents damage taken as half the structure ratio, since contents are damaged only once the envelope is breached):

| RP | $p$ | Gust (mph) | Structure damage | Loss |
|---|---|---|---|---|
| 5 | 0.200 | 75 | 0.20% | \$644 |
| 10 | 0.100 | 88 | 0.72% | \$2,318 |
| 25 | 0.040 | 102 | 1.76% | \$5,667 |
| 50 | 0.020 | 114 | 3.60% | \$11,592 |
| 100 | 0.010 | 126 | 6.48% | \$20,866 |
| 250 | 0.004 | 140 | 11.50% | \$37,030 |
| 500 | 0.002 | 150 | 16.50% | \$53,130 |

The same trapezoid sum gives $\text{EAL}_w = \$1{,}092.7$. Wind has a lower tail than flood for this house (1-in-100 wind loss \$20,866 vs. \$50,742) but damages it more often.

**Total:** $\text{EAL}_F = 1{,}773.8 + 1{,}092.7 = \$2{,}866.5$ per year.

### Worked example: present value

Assumptions: $\Lambda = 1.25$ (premiums run about 25% above expected loss), $r = 7\%$ (a buyer's opportunity cost of money, near today's mortgage rate), $g = 3\%$ (2% repair-cost inflation plus 1% hazard intensification). Then

$$M = \frac{1.25}{0.07 - 0.03} = 31.25,$$

$$\text{PV}_{F,f} = 31.25 \times 1{,}773.8 = \$55{,}430, \qquad \text{PV}_{F,w} = 31.25 \times 1{,}092.7 = \$34{,}146, \qquad \text{PV}_F = \$89{,}576.$$

The full climate risk cost is about 19% of the home's market value. $M$ is the most assumption-sensitive number in the pipeline: at $r = 6\%$ it is 41.7, at $r = 8\%$ it is 25. The report always shows this sensitivity.

### Output

Per property and hazard: the loss-exceedance curve $L_{i,h}(p)$, $\text{EAL}_{i,h}$, the 1-in-100 tail loss, and $\text{PV}_{i,h}$. Also the ratio $\text{PV}_{i,h}/V^{\text{mkt}}_i$ (here 11.8% for flood, 7.3% for wind), which is the scale on which pass-through is measured in 3.3.

### Hand-off

$\text{PV}/P$ ratios to 3.3 (the denominator of pass-through) and $\text{PV}_{i,h}$ to 3.4. The full loss-exceedance curves to 3.5, which turns them into correlated loss simulations. As a **shared module**, $\text{EAL}_{i,w}$ goes to escrow 2.3 as a rating variable in the insurance premium GLM.

### Validation

Hazard: compare modeled flood depths with high-water marks from past storms (USGS) and with NFIP claim locations (OpenFEMA). Damage: compare modeled losses with paid NFIP claims for the same events, by depth bin; a systematic gap means the damage curves need regional recalibration. EAL as a whole cannot be "backtested" on one house (the 100-year event rarely happens), but over thousands of homes and many years, total modeled EAL should match total paid losses after adjusting for insurance take-up.

---

## Step 3.3: Capitalization (causal inference on pass-through)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Arm's-length sales with prices, dates, characteristics | $P_j$, $t(j)$, $\mathbf{z}_j$ | Recorder and assessor (as in 3.1) |
| Flood zone at each date, and historical map versions | $\text{zone}_{j,t}$ | FEMA NFHL; historical FIRMs (Flood Insurance Rate Maps) and LOMRs (Letters of Map Revision, which change a map between full updates) with effective dates |
| Signed distance to the floodplain boundary | $s_j$ | GIS, from parcel centroid to the SFHA boundary |
| Risk cost relative to price | $x_j = \text{PV}_{j,f}/\bar P_{b(j)}$ | 3.2, divided by the block's median price from 3.1 |
| Block (small neighborhood) of each sale | $b(j)$ | Census blocks or a grid |

**Terms.** The **SFHA** (Special Flood Hazard Area) is the area a FEMA map designates as having at least a 1% annual chance of flooding; its zones are labeled A, AE, V, VE. Inside it, any mortgage backed or regulated by the federal government must carry flood insurance (the **mandatory purchase requirement**). **Zone X** is outside it: no requirement, and most owners there go uninsured. A **causal** effect is the change in price that the risk itself causes, as opposed to a correlation that might reflect something else (like a water view).

### What we are trying to measure

If buyers fully understood and cared about the risk, a home's price would fall by exactly $\text{PV}$. In log terms, for a risk cost that is a modest share of price,

$$\log P_j \approx \log V^0_j - \theta\, \frac{\text{PV}_j}{V^0_j},$$

so a regression of log price on $x_j = \text{PV}_j/\bar P$ has slope $-\theta$. The trouble is that $x_j$ is correlated with things buyers love (proximity to water). Each design below finds variation in risk, or in how visible risk is, that is *not* tangled with amenities.

And $\theta$ is not one number. It depends on the **information regime**: how visible the risk is to buyers. We estimate three:

- $\theta_X$: pass-through of flood risk for homes outside the mapped floodplain, where it is invisible on paper.
- $\theta_{AE} - \theta_X$: the extra pass-through when the map labels the home as in the floodplain (mandatory insurance, disclosure, lender scrutiny).
- $\theta_w$: pass-through of wind risk, which is visible through the homeowners premium every buyer must pay.

### Design 1: within-block elevation comparison (estimates $\theta_X$)

**Idea.** On a single block, every home has the same view, the same school, the same walk to the water. But first-floor heights differ (one house was built on a raised slab, the next on grade), so physical flood risk differs a lot between next-door neighbors. Comparing prices *within* a block isolates risk from location.

**Model.** For sales in Zone X,

$$\log P_j = \alpha_{b(j)} + \mathbf{z}_j^{\top}\boldsymbol\beta + \delta_{t(j)} - \theta_X\, x_j + \varepsilon_j,$$

where $\alpha_b$ is a **block fixed effect** (a separate intercept per block, absorbing everything shared on the block), $\delta_t$ is a period fixed effect (market-wide price level), and $\mathbf{z}_j$ are home characteristics. The block fixed effect is what removes the amenity confound. Estimate by least squares with standard errors **clustered** by block (allowing errors within a block to be correlated, which they are).

**Worked example.** 3,600 Zone X sales in 600 blocks, 2023–2026. Within-block standard deviation of $x$ is 0.035 (neighbors' risk costs differ by about 3.5% of price, from elevation alone).

| Specification | Coefficient on $x$ | Std. error |
|---|---|---|
| Pooled, no block effects | **+0.280** | 0.028 |
| Within block | **−0.233** | 0.029 |

The pooled regression says riskier homes sell for *more*: the amenity has swamped the risk. Within blocks the sign flips and the estimate is $\hat\theta_X = 0.233$. Buyers of Zone X homes price about 23% of the flood risk cost. This contrast is the single best illustration of why a naive "price on risk score" regression should never be used.

**What to check.** That first-floor elevation is not proxying for something else buyers value (newer construction tends to be elevated, so year built and condition must be in $\mathbf{z}$), and that buyers can observe it (it is visible from the street, which is the point: some buyers notice, most do not).

### Design 2: spatial regression discontinuity at the floodplain boundary (estimates $\theta_{AE} - \theta_X$)

**Idea.** Walk across a floodplain boundary that has been on the map for decades. Physical flood risk changes smoothly across that line (water does not stop at a drawn boundary), and so does everything else about location. But the *label* jumps: on one side, mandatory insurance and a disclosure; on the other, neither. Any jump in price exactly at the line is the effect of the label.

**Model.** Let $s_j$ be signed distance to the boundary (positive inside the SFHA). Fit separate straight lines on each side, using only sales within a **bandwidth** $H$ of the boundary and weighting closer sales more heavily (a **triangular kernel**, weight $1 - |s_j|/H$):

$$\log P_j = a_0 + a_1 s_j + \mathbf{1}[s_j > 0]\,\big(\Delta^{\text{RD}} + a_2 s_j\big) + \varepsilon_j.$$

$\Delta^{\text{RD}}$ is the jump at $s = 0$. Fitting lines rather than comparing means stops a slope (prices falling as you approach water, say) from being mistaken for a jump. Bandwidth is chosen by a data-driven rule that trades off bias (wide windows include homes too far from the line) against variance (narrow windows have few sales).

Converting to pass-through, using the risk cost at the boundary from 3.2:

$$\theta_{AE} - \theta_X = \frac{-\Delta^{\text{RD}}}{(\text{PV}/P)_{s=0}}.$$

**Worked example.** 2,997 sales within $H = 400$ m of a boundary in place since the 1990s. Average log price by 100 m bin:

| Distance (m) | −400 to −300 | −300 to −200 | −200 to −100 | −100 to 0 | **0 to 100** | 100 to 200 | 200 to 300 | 300 to 400 |
|---|---|---|---|---|---|---|---|---|
| Side | outside | outside | outside | outside | **inside** | inside | inside | inside |
| Mean log price | 12.937 | 12.942 | 12.943 | 12.944 | **12.897** | 12.908 | 12.919 | 12.916 |

The local-linear intercepts at the boundary are 12.947 (outside) and 12.895 (inside):

$$\hat\Delta^{\text{RD}} = -0.0524 \quad (\text{s.e. } 0.0056).$$

Homes just inside the line sell about 5.1% below homes just outside. At the boundary, 3.2 puts flood $\text{PV}/P$ at 0.15, so

$$\hat\theta_{AE} - \hat\theta_X = \frac{0.0524}{0.15} = 0.349, \qquad \hat\theta_{AE} = 0.233 + 0.349 = 0.582.$$

Being on the map more than doubles how much of the risk is priced, but even then buyers price only about 58% of it.

**What to check.** That physical risk really is continuous at the line (plot 3.2's EAL against distance: no jump allowed); that sale counts are not bunched on one side (a **density test**, since sellers might avoid listing inside); and that characteristics like size and age are balanced across the line.

### Design 3: event study around a map revision (traces how pass-through builds over time)

**Idea.** In 2021 a neighboring county's map update moved 800 homes from Zone X into Zone AE. Compare their prices before and after with 2,200 nearby homes that stayed in Zone X. This is **difference-in-differences** (DiD): the change over time for treated homes minus the change for controls, which removes both permanent differences between the groups and market-wide trends. An **event study** is DiD estimated separately for each period relative to the event, which shows the dynamics and lets you check the key assumption.

**Model.**

$$\log P_{j} = \alpha_{i(j)} + \delta_{t(j)} + \sum_{k \neq -1} \beta_k\, \mathbf{1}\big[i(j) \text{ treated},\ \text{event year}(j) = k\big] + \varepsilon_j,$$

with $\alpha_i$ a home fixed effect (each home compared with itself), $\delta_t$ a quarter fixed effect, and $k$ the year relative to the map's effective date ($k = 0$ is the first year after). The year before ($k = -1$) is the reference, so each $\beta_k$ is the treated-minus-control price change since then. The key assumption is **parallel trends**: without the remap, treated and control prices would have moved together. It cannot be proven, but if the pre-period $\beta_k$ ($k < -1$) are near zero, the groups were moving together before the event, which makes it credible.

**Worked example.** 12,185 sales of 3,000 homes, 2017–2024, standard errors clustered by neighborhood:

| Event year $k$ | −4 | −3 | −2 | −1 | 0 | 1 | 2 | 3 |
|---|---|---|---|---|---|---|---|---|
| $\hat\beta_k$ | +0.001 | +0.008 | +0.002 | 0 (ref.) | −0.008 | −0.021 | −0.026 | −0.036 |
| Std. error | 0.004 | 0.004 | 0.005 | | 0.005 | 0.004 | 0.005 | 0.005 |

Pre-trends: a joint test that $\beta_{-4} = \beta_{-3} = \beta_{-2} = 0$ gives $\chi^2_3 = 3.89$, $p = 0.27$. No evidence the groups were diverging before the remap.

After the remap, prices of treated homes fall gradually, reaching −3.6% in year 3 and still falling. With treated homes' average flood $\text{PV}/P = 0.155$, that is an increase in pass-through of $0.036 / 0.155 = 0.23$ after three years, on its way to the 0.349 long-run value from Design 2. **The adjustment is not instant**, which is precisely what 3.4 models.

### Wind pass-through

Wind risk is priced through the homeowners premium that every mortgaged buyer pays and sees at closing. The same event-study machinery, applied around the 2022–2023 Florida premium spikes (homes whose premium jumped vs. similar homes whose did not), gives $\hat\theta_w = 0.80$ (s.e. 0.07) in this worked example. Visible costs are mostly capitalized; invisible ones mostly are not.

### Output

| Regime | Pass-through | Std. error | Design |
|---|---|---|---|
| Flood, Zone X | $\theta_X = 0.233$ | 0.029 | Within-block elevation |
| Flood, extra from SFHA label | $\theta_{AE} - \theta_X = 0.349$ | 0.037 | Boundary RD |
| Flood, Zone AE | $\theta_{AE} = 0.582$ | | Sum |
| Wind | $\theta_w = 0.80$ | 0.07 | Event study, premium shocks |

Plus the event-study path $\hat\beta_k$, which 3.4 uses to check its adjustment-speed estimate.

### Hand-off

All $\theta$ estimates, with standard errors, to 3.4 (to split value into priced and unpriced) and to 3.5 (repricing scenarios).

### Validation

Each design has its own falsification test, described above (pre-trends, continuity of physical risk at the boundary, density, covariate balance). Across designs, the estimates must be mutually consistent: the event-study path should converge toward the RD value. Placebo tests: run the RD at a fake boundary 500 m away (should find nothing), and run the event study with a fake event date (should find nothing). Re-estimate by region and period; pass-through has risen after recent salient disasters, so a stale $\theta$ understates current pricing.

---

## Step 3.4: Mispricing and adjustment speed (decomposition + partial adjustment)

### Inputs

| Input | From |
|---|---|
| Market value distribution $V^{\text{mkt}}_i$ | 3.1 |
| $\text{PV}_{i,h}$ by hazard, with uncertainty in EAL | 3.2 |
| Pass-through by regime, with standard errors | 3.3 |
| Current and scheduled flood zone (preliminary map, expected effective date) | FEMA NFHL and preliminary FIRM data |
| Panel of neighborhood log price indices around past triggers | 3.1's repeat-sales machinery applied by neighborhood |

### Process, part 1: the decomposition

For home $i$ in regime $z$ (its current flood zone),

$$\underbrace{\theta_z\,\text{PV}_{i,f} + \theta_w\,\text{PV}_{i,w}}_{\text{priced}}, \qquad \underbrace{U_i = (1 - \theta_z)\,\text{PV}_{i,f} + (1 - \theta_w)\,\text{PV}_{i,w}}_{\text{unpriced}},$$

$$V^0_i = V^{\text{mkt}}_i + \text{priced}, \qquad V^{*}_i = V^0_i - \text{PV}_i = V^{\text{mkt}}_i - U_i.$$

### Process, part 2: partial adjustment

When a trigger changes the regime (Zone X to AE), the market's equilibrium price falls by $(\theta_{AE} - \theta_X)\,\text{PV}_{i,f}$. But prices do not jump there; they move part of the way each quarter. The **partial-adjustment model** for neighborhood $n$ (in logs, after removing market-wide moves):

$$\Delta p_{n,t} = \lambda\,\big(p^{*}_{n,t} - p_{n,t-1}\big) + \delta_t + \varepsilon_{n,t},$$

where $p_{n,t}$ is the neighborhood's log price index, $p^{*}_{n,t}$ its target given the current regime ($p^{*}$ drops by $(\theta_{AE}-\theta_X)\,\text{PV}/P$ at the trigger), $\delta_t$ is a quarter effect, and $\lambda \in (0,1)$ is the share of the remaining gap closed each quarter. Why partial? Sellers anchor on past prices and would rather wait than cut; appraisals lean on stale comparable sales; and information spreads gradually as listings, inspections, and insurance quotes accumulate.

After $k$ quarters, the fraction of the gap closed is $1 - (1-\lambda)^k$. The **half-life** is the $k$ at which this hits one half:

$$(1-\lambda)^{t_{1/2}} = \tfrac12 \quad \Longrightarrow \quad t_{1/2} = \frac{\ln 2}{-\ln(1-\lambda)}.$$

**Estimation.** Regress $\Delta p_{n,t}$ on the gap $p^{*}_{n,t} - p_{n,t-1}$ with quarter effects, across neighborhoods that experienced triggers at different times, clustering by neighborhood. Two cautions. First, the target $p^{*}$ is built from 3.3's estimate, so its uncertainty should be carried through (bootstrap the two steps together). Second, a regression on a lagged price with neighborhood fixed effects is biased in short panels (**Nickell bias**); with 10 years of quarters it is small, and building the target from pre-trigger levels avoids estimating the fixed effects from the same data.

### Process, part 3: Monte Carlo for the range

Every input is uncertain: the AVM, the hazard and damage estimates, and the pass-through estimates. Draw each from its distribution and recompute the decomposition. For each of $N = 200{,}000$ draws:

1. $V^{\text{mkt}}$ from the AVM lognormal (log sd 0.056).
2. $\text{EAL}_f$ lognormal around 3.2's value with log sd 0.40 (flood hazard and damage curves are uncertain); $\text{EAL}_w$ with log sd 0.30.
3. $\theta_X \sim \mathcal N(0.233, 0.029^2)$, $\theta_w \sim \mathcal N(0.80, 0.07^2)$.
4. Compute $\text{PV}$, $U$, $V^0$, $V^{*}$.

### Worked example: the decomposition at central values

$$\text{priced} = 0.233 \times 55{,}430 + 0.80 \times 34{,}146 = 12{,}915 + 27{,}317 = \$40{,}232,$$

$$U_F = 0.767 \times 55{,}430 + 0.20 \times 34{,}146 = 42{,}515 + 6{,}829 = \$49{,}344,$$

$$V^0_F = 467{,}888 + 40{,}232 = \$508{,}120, \qquad V^{*}_F = 467{,}888 - 49{,}344 = \$418{,}544.$$

| | Amount | Share of market value |
|---|---|---|
| Climate-blind value $V^0$ | \$508,120 | 108.6% |
| − Priced risk (flood \$12,915, wind \$27,317) | −\$40,232 | −8.6% |
| = **Market value $V^{\text{mkt}}$** | **\$467,888** | 100% |
| − Unpriced risk (flood \$42,515, wind \$6,829) | −\$49,344 | −10.5% |
| = **Climate-adjusted value $V^{*}$** | **\$418,544** | 89.5% |

86% of the unpriced risk is flood. Wind is mostly priced already because buyers see it in the premium.

### Worked example: adjustment speed

Panel of 150 neighborhoods over 39 quarters (5,850 observations), 84 of which experienced a remap at some point:

$$\hat\lambda = 0.097 \ \ (\text{s.e. } 0.005), \qquad t_{1/2} = \frac{\ln 2}{-\ln(0.903)} = 6.8 \text{ quarters} \approx 1.7 \text{ years}.$$

Consistency check against the event study in 3.3: with $\lambda = 0.097$, the average gap closed over quarters 12–15 after a remap is about 75%, which predicts $0.75 \times 0.349 \times 0.155 = 0.041$ in year 3. The event study estimated 0.036 (s.e. 0.005). The two methods agree within about one standard error.

**Property F's timeline.** When the new map takes effect (assumed Q3 2028), F's target drops by the extra priced share of its flood cost:

$$(\theta_{AE} - \theta_X)\,\text{PV}_{F,f} = 0.349 \times 55{,}430 = \$19{,}345.$$

Holding the rest of the market flat:

| Quarters after the map takes effect | 0 | 2 | 4 | 8 | 12 | 20 |
|---|---|---|---|---|---|---|
| Share of gap closed $1-(1-\lambda)^k$ | 0% | 18% | 33% | 56% | 71% | 87% |
| Expected price (\$) | 467,900 | 464,300 | 461,400 | 457,100 | 454,200 | 451,100 |

After the remap fully works through, F still has \$49,344 − \$19,345 = \$29,999 of unpriced risk (the 42% of flood cost that buyers in Zone AE still ignore, plus the unpriced 20% of wind). That remainder closes only under a bigger trigger: insurance priced at full risk *and* enforced, or a damaging flood in the neighborhood. 3.5 stresses that case.

An important caveat: the preliminary map is already public, so some adjustment may begin before 2028. The model above is conservative on timing. A refinement estimates a separate, smaller $\lambda$ for the announcement-to-effective window.

### Worked example: Monte Carlo range

| Quantity | P10 | P50 | P90 | Mean |
|---|---|---|---|---|
| Market value $V^{\text{mkt}}$ (\$) | 435,400 | 467,900 | 503,000 | 468,600 |
| Full risk cost $\text{PV}$ (\$) | 65,300 | 91,700 | 131,200 | 95,700 |
| Unpriced risk $U$ (\$) | 32,000 | 49,800 | 78,500 | 53,200 |
| Unpriced share of market value | 6.8% | 10.6% | 16.9% | |
| Climate-adjusted value $V^{*}$ (\$) | 373,900 | 416,100 | 456,600 | 415,500 |

$\Pr(U > 5\% \text{ of value}) = 99\%$; $\Pr(U > 10\%) = 57\%$. The home is almost certainly overvalued; the question is by how much, and the answer is "probably 7% to 17%." The width comes mostly from flood EAL uncertainty, which is why better elevation data (an updated elevation certificate) is the cheapest way to narrow it.

**Sensitivity to the discount assumptions** (central $U$): \$65,800 at $r = 6\%$; \$49,300 at $r = 7\%$; \$39,500 at $r = 8\%$ or at $g = 2\%$.

### Output

Per property: $V^0$, $V^{*}$ and $U$ with P10/P50/P90, split by hazard and by "reachable on the next trigger" vs. "persistent"; the expected price path after the scheduled trigger; and $\lambda$ with its half-life (per region, if estimated regionally).

### Hand-off

To the report; to 3.5 for the repricing scenarios (portfolio sum of $U$ and of the trigger-reachable parts); and to lenders' LTV monitoring (a repriced value raises current LTV, which feeds 5.2's default and turnover hazards).

### Validation

Backtest the decomposition: for homes remapped in past revisions, did the realized price change match $(\theta_{AE} - \theta_X)\,\text{PV}/P$ at the predicted speed? Check coverage of the Monte Carlo range against those realized changes. Re-estimate $\lambda$ on recent triggers only; adjustment may be speeding up as climate risk becomes more salient.

---

## Step 3.5: Stress testing and portfolio aggregation

### Inputs

| Input | From |
|---|---|
| Each home's loss-exceedance curves $L_{i,f}(p)$, $L_{i,w}(p)$ | 3.2 |
| Coordinates, values, flood zones | 3.1, FEMA |
| $\text{PV}$, $\theta$, $U$ for each home | 3.2–3.4 |
| Spatial correlation parameters | Estimated from historical event footprints and claims |
| Scenario definitions | Client and climate science (sea-level and hurricane projections) |

### Why correlation is the whole story

A portfolio's expected loss is just the sum of each home's EAL; correlation does not change it. But the **tail** depends entirely on correlation. One hurricane damages every home in its path in the same year. Treating losses as independent makes a portfolio look far safer than it is, because bad years for different homes would rarely coincide. The portfolio's risk lives in the bad years.

**Terms.** The **1-in-100 portfolio loss** (also **VaR**$_{0.99}$, value at risk) is the annual loss exceeded with probability 1%. **Expected shortfall** ES$_{0.99}$ is the average loss in the worst 1% of years: it answers "when it's bad, how bad?" and, unlike VaR, it adds up sensibly across sub-portfolios.

### Process

Two standard ways to get correlated losses:

- **Event-based catastrophe simulation:** generate thousands of synthetic hurricanes and floods with physical footprints, compute every home's loss in each event, and sum. Most realistic; requires a catastrophe model.
- **Gaussian copula:** keep each home's own loss distribution from 3.2 exactly, and impose correlation through a hidden normal variable. Much simpler, and what we show here.

**The copula.** A **copula** is a way to join separate marginal distributions with a chosen dependence structure. For each hazard $h$, give each home a latent standard normal $Z_{i,h}$ with correlation decaying with distance:

$$\operatorname{Corr}(Z_{i,h}, Z_{j,h}) = e^{-d_{ij}/\ell_h},$$

with **correlation length** $\ell_f = 3$ km for flood (floods are local) and $\ell_w = 40$ km for wind (a hurricane's damaging winds span tens of kilometers). Flood and wind come from the same storms, so link them: $Z_{i,f} = c\, Z_{i,w} + \sqrt{1-c^2}\, Z'_{i,f}$ with $c = 0.5$, where $Z'_{i,f}$ has the flood correlation structure. Draw all latents jointly (through the Cholesky factor of the correlation matrix, the matrix "square root" that turns independent normals into correlated ones).

Then map each latent to a loss through that home's own curve. Since $Z$ is standard normal, $1 - \Phi(Z)$ is uniform on $(0,1)$, where $\Phi$ is the standard normal cumulative distribution function. Treat it as the exceedance probability of this year's event at that home:

$$L_{i,h} = L_{i,h}\big(1 - \Phi(Z_{i,h})\big).$$

So a home whose $Z$ lands in its top 1% gets its 1-in-100 loss, and neighbors with correlated $Z$ tend to land in their tails in the same year. Each home's EAL is preserved exactly; only the co-movement is added.

Portfolio loss in each simulated year: $L^{\text{port}} = \sum_i (L_{i,f} + L_{i,w})$. Simulate 200,000 years and read off the mean, VaR$_{0.99}$, and ES$_{0.99}$.

**Concentration.** Attribute the tail to homes by their average loss in the worst-1% years, $\mathbb E[L_i \mid L^{\text{port}} \geq \text{VaR}_{0.99}]$. These contributions add up exactly to ES$_{0.99}$ (the **Euler allocation**) and identify what drives the bad years, which can be very different from what drives the average.

**Scenarios.** Rerun with changed inputs:

1. **Hazard frequency up:** multiply flood exceedance probabilities by 1.5 (roughly sea-level rise by mid-century: today's 100-year flood becomes a 67-year flood) and wind probabilities by 1.25.
2. **Map revision:** homes the preliminary maps move into Zone AE take the $(\theta_{AE} - \theta_X)\,\text{PV}_f$ value hit from 3.4.
3. **Full insurance repricing:** risk becomes fully visible ($\theta \to 1$ for all homes); the portfolio loses its whole unpriced risk $\sum_i U_i$ in value.
4. **Combined:** hazard frequency up and full repricing.

Scenarios 1 to 4 measure two different things: annual damage losses (a cash-flow risk) and value losses (a mark-to-market risk). The report keeps them separate.

### Worked example: a 250-home single-family rental portfolio

A fund holds 250 rental homes in the same coastal county as Property F (F included), total value **\$99.3 million**. 212 have some flood exposure; 48 are already in Zone AE. Each home's curves are F's curves scaled by its value and shifted by its flood and wind exposure (closer to the coast, more frequent events). Total EAL: flood \$327,000, wind \$237,000, together **\$564,000** a year (0.57% of value).

| Annual damage loss | Expected | 1-in-100 (VaR$_{0.99}$) | ES$_{0.99}$ |
|---|---|---|---|
| Baseline, correlated | \$568,000 | **\$5.85M** | **\$8.39M** |
| Baseline, if losses were independent | \$564,000 | \$0.95M | \$1.01M |
| Hazard frequency up | \$787,000 | \$6.95M | \$9.68M |

(The simulated mean matches the \$564,000 sum of EALs to within simulation error, as it must.)

Things to notice:

- **Independence understates the 1-in-100 loss by a factor of six.** \$0.95M vs. \$5.85M, with the same expected loss. That is the error a model makes when it sums home-level risk scores without correlation.
- **Diversification is real but limited.** The sum of the 250 homes' individual 1-in-100 losses is \$10.6M; the portfolio's is \$5.85M. Not every home has its worst year at once, but many do.
- **Flood drives the average; wind drives the tail.** Flood is 58% of expected loss but only 36% of ES$_{0.99}$. Floods are local; hurricane wind hits the whole portfolio at once.
- **Hazard frequency up raises the expected loss 39% but the 1-in-100 only 19%**, because the tail is already dominated by the region-wide wind events that change less in this scenario.

**Concentration by distance to the coast:**

| Band | Homes | Share of value | Share of expected loss | Share of ES$_{0.99}$ |
|---|---|---|---|---|
| Within 1 km | 65 | 26% | 46% | 37% |
| 1 to 3 km | 98 | 40% | 41% | 43% |
| Beyond 3 km | 87 | 34% | 13% | 20% |

The coastal band is the obvious concentration in expected loss. The tail is more spread out, because wind reaches inland homes too. Property F alone contributes \$51,000 to ES$_{0.99}$.

**Value (mark-to-market) scenarios:**

| Scenario | Value loss | Share of portfolio |
|---|---|---|
| Map revision (44 Zone X homes move to AE), once fully adjusted | \$1.02M | 1.0% |
| Full insurance repricing ($\theta \to 1$) | \$7.50M | 7.6% |
| Hazard frequency up + full repricing | \$14.47M | 14.6% |

The last line combines the unpriced risk at today's hazard (\$7.50M) with the new risk cost created by more frequent hazards, none of which is priced. A fund marking its homes at AVM values is carrying a latent 7.6% value loss before any storm happens. With a half-life of 1.7 years once triggered (3.4), the map-revision part would mostly arrive within the fund's holding period.

### Output

Portfolio expected annual loss, VaR$_{0.99}$, ES$_{0.99}$, and 1-in-250 by scenario; the independence comparison; the Euler allocation of the tail by home, band, zip code, and hazard; and the value-loss scenarios with timing from 3.4.

### Hand-off

To the report. Portfolio losses also feed credit models (a lender's or MBS investor's expected default in a cat year) and the escrow insurance channel (2.3–2.4), since a cat year is also a year of insurer exits and premium jumps.

### Validation

Check the copula against history: in past hurricanes, did the share of a region's homes with claims match the model's joint-exceedance rates? Calibrate $\ell_f$ and $\ell_w$ to the spatial pattern of NFIP and homeowners claims in past events. Compare the copula's portfolio tail with an event-based catastrophe model on a sample portfolio; the copula is known to understate tail dependence (a Gaussian copula's extremes become independent in the far tail), so a Student-t copula is the standard upgrade if the comparison shows a gap.

---

## The report

### Per-property card (Property F)

- **Market value:** \$467,900 (80% range \$435,400 to \$502,800).
- **Climate-adjusted value:** \$418,500 (80% range \$373,900 to \$456,600).
- **Full climate risk cost:** \$89,600 present value: flood \$55,400 (EAL \$1,774/yr; 1-in-100 loss \$50,700), wind \$34,100 (EAL \$1,093/yr; 1-in-100 loss \$20,900).
- **Priced vs. unpriced:** the market prices \$40,200 (23% of the flood cost, 80% of the wind cost). Unpriced: **\$49,300 (10.5% of value)**, likely 7% to 17%.
- **Why:** the current map leaves the home outside the floodplain although the 100-year flood reaches 1.7 ft at the house. Buyers of unmapped homes price less than a quarter of flood risk.
- **Adjustment timeline:** the preliminary map moves the home into Zone AE, expected effective Q3 2028. About \$19,300 of the gap should close after that, half within about 1.7 years. About \$30,000 remains unpriced until insurance is priced and enforced at full risk.
- **Actions:** for a buyer, negotiate toward the climate-adjusted value and price in a flood policy now; for a lender, use the adjusted value for LTV on a new loan; for an owner, elevation or flood-proofing that lowers $d$ by a foot cuts flood EAL substantially (rerun 3.2 with the new first-floor height to price the mitigation).

### Portfolio summary (250-home SFR fund)

| Metric | Baseline | Hazard up | Note |
|---|---|---|---|
| Expected annual damage | \$568,000 | \$787,000 | 0.57% → 0.79% of value |
| 1-in-100 annual damage | \$5.85M | \$6.95M | 6× what independence would suggest |
| ES$_{0.99}$ | \$8.39M | \$9.68M | |
| Unpriced risk (value) | \$7.50M | \$14.47M with full repricing | 7.6% → 14.6% of value |
| Near-term repricing from map revisions | \$1.02M | | 44 homes, half-life 1.7 years |

### Concentration map

Rows are zip codes or coastal-distance bands; columns are **expected loss**, **tail contribution** (ES$_{0.99}$), **unpriced value**, and **trigger exposure** (homes on preliminary maps). Each cell is that area's share of the portfolio total. Reading it across tells a fund where losses happen on average, where they happen in the bad year, and where value is most exposed to repricing, which are three different places.

---

## Symbol reference

| Symbol | Meaning | Step |
|---|---|---|
| $i$, $j$, $b$, $n$ | Property; sale; block; neighborhood | all |
| $t$, $k$ | Time period (quarter or year); event time or lag | all |
| $V^{\text{mkt}}_i$, $V^0_i$, $V^{*}_i$ | Market value; climate-blind value; climate-adjusted value | 3.1, 3.4 |
| $P_j$, $P^{\text{adj}}_j$, $\bar P_b$ | Sale price; time-adjusted price; block median price | 3.1, 3.3 |
| $\pi_t$ | Log repeat-sales price index | 3.1 |
| $\mathbf{z}_i$, $F(\cdot)$, $f_m$, $\nu$ | Characteristics; boosted model; one tree; learning rate | 3.1 |
| $\epsilon_j$, $\hat\epsilon_i$ | Boosting residual; kriged residual | 3.1 |
| $\sigma_s$, $\rho$, $\sigma_n$ | Spatial sd; range; nugget | 3.1 |
| $\mathbf{K}$, $\mathbf{k}$, $\mathbf{w}$ | Neighbor covariance matrix; covariance with target; kriging weights | 3.1 |
| $\hat q$, $\alpha$ | Conformal quantile; miscoverage level | 3.1 |
| $h \in \{f, w\}$ | Hazard: flood, wind | 3.2–3.5 |
| $p$, RP | Annual exceedance probability; return period $=1/p$ | 3.2 |
| WSE, $G$, FFE, $d$ | Water surface elevation; ground elevation; first-floor elevation; depth above first floor | 3.2 |
| $v$ | Wind gust speed | 3.2 |
| $D^{\text{str}}_h$, $D^{\text{con}}_h$ | Damage fraction functions, structure and contents | 3.2 |
| $S_i$, $C_i$ | Structure replacement cost; contents value | 3.2 |
| $L_{i,h}(p)$ | Loss exceeded with probability $p$ | 3.2, 3.5 |
| $\text{EAL}_{i,h}$ | Expected annual loss | 3.2 |
| $\Lambda$, $r$, $g$, $M$ | Loading; discount rate; growth of risk cost; multiplier $\Lambda/(r-g)$ | 3.2 |
| $k_{i,h}$, $\text{PV}_{i,h}$ | Annual risk cost; its present value | 3.2 |
| $\theta$, $\theta_X$, $\theta_{AE}$, $\theta_w$ | Pass-through overall; flood in Zone X; flood in Zone AE; wind | 3.3–3.5 |
| $x_j$ | Flood PV divided by block median price | 3.3 |
| $\alpha_b$, $\alpha_i$, $\delta_t$ | Block, home, and period fixed effects | 3.3 |
| $s_j$, $H$, $\Delta^{\text{RD}}$ | Signed distance to boundary; bandwidth; jump at boundary | 3.3 |
| $\beta_k$ | Event-study coefficient for event year $k$ | 3.3 |
| $U_i$ | Unpriced risk $= (1-\theta)\,\text{PV}$ | 3.4, 3.5 |
| $p_{n,t}$, $p^{*}_{n,t}$ | Neighborhood log price index; its target | 3.4 |
| $\lambda$, $t_{1/2}$ | Adjustment speed per quarter; half-life | 3.4 |
| $Z_{i,h}$, $\Phi$ | Latent normal for the copula; standard normal CDF | 3.5 |
| $\ell_f$, $\ell_w$, $c$ | Correlation lengths; flood-wind link | 3.5 |
| $L^{\text{port}}$, VaR$_{0.99}$, ES$_{0.99}$ | Portfolio annual loss; 1-in-100 loss; expected shortfall | 3.5 |
| LTV | Loan-to-value ratio | 3.1, 3.4 |
