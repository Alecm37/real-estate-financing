# Service 2: Escrow Forecasting, explained as a pipeline

*Market backdrop: mid-October 2026. All account-level numbers below are one worked example run end to end; they are illustrative, not quotes of any real county or insurer.*

---

## 0. What we are forecasting, and why it needs five models

### The object being forecast

Most mortgage borrowers do not pay their property tax and homeowners insurance directly. Instead, each month they pay the **servicer** (the company that collects payments on the loan, which may or may not be the lender) a sum on top of principal and interest, and the servicer holds it in an **escrow account** (a trust balance held on the borrower's behalf). When the tax bill or insurance renewal comes due, the servicer pays it out of that account. The borrower's total monthly payment is therefore **PITI**: Principal + Interest + Taxes + Insurance.

Once a year the servicer performs an **escrow analysis**: it projects the next twelve months of bills, sets a new monthly escrow payment, and checks whether the account has too little money (**shortage**) or too much (**surplus**). The rules for this analysis come from RESPA (the Real Estate Settlement Procedures Act) as implemented in Regulation X, section 1024.17. We will use them explicitly in step 2.5.

The quantity this service forecasts, for each account $i$, is the **change in the monthly escrow payment at the next escrow analysis**, together with its full distribution (P10/P50/P90, where P10 is the 10th percentile: a value the outcome falls below 10% of the time), and an explanation of *why* it moves: how much comes from tax and how much from insurance.

### Why it is worth forecasting now

The backdrop makes escrow the part of the housing payment that is actually moving:

- Homeowners insurance premiums are up roughly 64% since 2021, and about two-thirds of escrow accounts are **underfunded**, meaning the payment was set from bills that have since risen.
- With the 30-year fixed rate near 7.3% (Freddie Mac PMMS) and about half of outstanding mortgages below 4%, almost nobody can refinance their way out of a payment increase. Principal and interest are frozen; escrow is the only part that moves.
- Delinquencies are about 4.37% and rising, concentrated in FHA loans (loans insured by the Federal Housing Administration, typically lower down payments and lower credit scores). An escrow shock is a classic trigger for a borrower who was already stretched.
- Home prices are roughly flat in nominal terms, so property values are not rising fast. But tax bills can still jump for rule-driven reasons (a cap resetting at sale), and insurance can still jump for market reasons (insurer exits). Neither shows up in a smooth time series of past escrow payments.

### Why five models instead of one

Escrow is the sum of two very different things:

$$\text{annual escrow disbursements} = \underbrace{\text{property tax}}_{\text{legal rules} \times \text{market value} \times \text{tax rate}} + \underbrace{\text{homeowners insurance}}_{\text{actuarial pricing} + \text{occasional jumps}}$$

Property tax is mostly **deterministic law applied to an uncertain input**: given the market value and the tax rate, the bill follows mechanically from statute. Insurance is **statistical pricing plus discrete shocks**: most years it drifts with costs, but occasionally a policy is cancelled and replaced at a very different price. A single regression on past escrow payments would blur these together and would completely miss predictable shocks, such as a new owner's assessment resetting to market value. So we model each piece with the tool that fits it, then combine.

| Step | Model | Produces |
|---|---|---|
| 2.1 | Rules engine + valuation (from AVM 3.1) | Next year's taxable value |
| 2.2 | Hierarchical Bayesian time series | Distribution of next year's tax rate |
| 2.3 | Gamma GLM + distributed lag | Expected premium drift |
| 2.4 | Calibrated classifier + mixture | Probability and size of a premium jump |
| 2.5 | Monte Carlo + escrow rules | Distribution of the new escrow payment, with attribution |

### The actual flow of numbers

```
AVM (3.1) ──► market value V ──► 2.1 assessed & taxable value TV ─┐
                     │                                             ├─► tax bill T ─┐
                     └──► jurisdiction growth g ──► 2.2 tax rate τ ┘               │
                                                                                   ├─► 2.5 escrow
Climate EAL (3.2) ──► 2.3 premium drift ℓᵈ ─┐                                      │    analysis
Reinsurance index ──► 2.3                   ├─► premium I ─────────────────────────┘    (RESPA rules)
Policy features   ──► 2.4 jump prob p, size ℓʲ ┘                                        │
                                                                                         ▼
                                       new escrow payment E, change ΔE, attribution, DTI flags ──► report
```

Two links deserve emphasis because they are easy to miss. First, the **same market shock** that moves this home's value (2.1) also moves the whole jurisdiction's assessment growth, which feeds the tax-rate model (2.2): when assessments rise, local governments tend to cut the rate partially. So value and rate must be simulated *jointly* in 2.5, not independently. Second, the jump in 2.4 does not add to the drift in 2.3; it **replaces** it (a forced switch to a new insurer means new pricing), which is why the two are combined as a mixture.

### Timeline used in the worked example

Escrow forecasting is all about *which bill lands in which analysis*, so we fix the calendar:

- **Today:** mid-October 2026.
- **Bills:** property tax is paid in **November** of each year (month 11), based on the assessment as of **January 1** of that year. The insurance policy renews every **June** (month 6).
- **Analyses:** this servicer runs its escrow analysis for the account each **December**. The December 2026 analysis sets the 2027 payment $E_0$ from bills that are already known. The **December 2027 analysis** sets the 2028 payment $E_1$, after the November 2027 tax bill and the June 2027 renewal have been paid.
- **Forecast target:** $\Delta E = E_1 - E_0$, the payment change that hits the borrower in January 2028. We are forecasting it **14 months in advance**, before either bill exists.

### The running example: Account A

A house in a Florida county (Florida is the cleanest teaching case because its tax cap resets at sale and its insurance market is the most shock-prone in the country).

- Purchased **June 2026** for **\$420,000** with 20% down: a loan of **\$336,000**, 30-year fixed at **6.5%**. Monthly principal and interest $\text{PI} = \$2{,}123.75$.
- Borrower gross monthly income at origination **\$9,500**; other monthly debt payments **\$650**.
- The **seller** had owned the home for many years with a homestead exemption. The 2026 tax bill (paid November 2026) is based on the **seller's** January 1, 2026 assessment: assessed value \$205,000, homestead exemption \$50,000.
- The buyer has an insurance policy that started at closing with annual premium **\$5,400**, renewing each June.
- The buyer will file for their own homestead exemption, effective January 1, 2027.

The punchline we will derive: the escrow payment rises from about **\$679** to about **\$1,170** per month at the median, an increase of roughly **\$490**, and about 90% of the *uncertainty* around that number comes from insurance even though most of the *expected increase* comes from tax.

---

## Step 2.1: Tax projection (rules engine + valuation model)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Market value of property $i$ on January 1 of year $t$ | $V_{i,t}$ | AVM, step 3.1 (a distribution, not a point) |
| Last year's assessed value | $A_{i,t-1}$ | County assessor (property appraiser) roll |
| Whether the property sold since the last January 1 | $\text{sale}_{i,t} \in \{0,1\}$ | County recorder deeds, joined to the loan via record linkage (4.1) |
| Homestead status and other exemptions | $\text{EX}_{i,t}$ | Assessor roll; for a new buyer, the rules engine predicts the filing |
| Jurisdiction rules: cap rate, assessment ratio, reset rule, reassessment frequency | $\gamma^{\text{cap}}_{t}$, $\kappa_j$ | Statute, encoded by hand per state and county |
| Tax rate in mills | $\tau_{j,t}$ | Step 2.2 (forecast distribution) |

**Terms.** The **market value** (Florida calls it **just value**) is what the house would sell for. The **assessed value** is the figure the law says to tax; it can be below market value because of caps. An **exemption** is a fixed dollar amount subtracted before taxing; the **homestead exemption** applies to an owner's primary residence. The **taxable value** is assessed value minus exemptions. The **tax rate** is quoted in **mills**: one mill is \$1 of tax per \$1,000 of taxable value, so 18.5 mills means 1.85%. A **reassessment** is the assessor revaluing the property; some states do it yearly, others every few years.

### Process

The core of 2.1 is not statistical at all. It is a **rules engine**: a per-jurisdiction function that turns a market value into a tax bill exactly as the statute does. Statistics enters only through its input $V_{i,t}$, which comes from the valuation model.

**Rule 1: assessed value.** For a capped homestead property that did *not* sell,

$$A_{i,t} = \min\!\Big( A_{i,t-1}\,\big(1 + \gamma^{\text{cap}}_{t}\big),\; \kappa_j V_{i,t} \Big),$$

and for a property that **sold** (or lost its homestead) since last January 1, the cap resets and

$$A_{i,t} = \kappa_j V_{i,t}.$$

Here $\gamma^{\text{cap}}_{t}$ is the annual cap growth. Florida's **Save Our Homes** amendment caps the annual growth of a homestead's assessed value at the lesser of 3% and the change in CPI (the Consumer Price Index), so with CPI near 3.4%, $\gamma^{\text{cap}}_{2027} = \min(3\%, 3.4\%) = 3\%$. The **assessment ratio** $\kappa_j$ is the ratio of the assessor's value to market value; Florida appraisers deduct roughly 15% for costs of sale, so we use $\kappa_j = 0.85$. The $\min(\cdot)$ is there because the cap limits *growth*, but assessed value may never exceed $\kappa_j$ times market value (if the market falls far enough, the market value branch binds).

The reason a homesteaded home's assessed value is so often far below market is accumulation: a decade of 3% capped growth against 8% market growth opens a wide gap. When the home sells, that whole gap closes in one year. That is the **predictable shock** the service is designed to catch.

**Rule 2: taxable value.**

$$\text{TV}_{i,t} = \max\!\big(A_{i,t} - \text{EX}_{i,t},\; 0\big).$$

(Florida's homestead exemption is actually two \$25,000 tiers, one of which does not apply to school taxes; we simplify to a flat \$50,000 applied to all levies.)

**Rule 3: the tax bill and the disbursement.**

$$T_{i,t} = \text{TV}_{i,t} \cdot \frac{\tau_{j,t}}{1000}, \qquad T^{\text{disb}}_{i,t} = (1 - d)\, T_{i,t},$$

where $\tau_{j,t}$ is the total millage of every taxing authority covering the parcel (county, school board, city, special districts), and $d$ is the early-payment discount. Florida gives 4% off for paying in November, and servicers pay in November to capture it, so $d = 0.04$. The disbursement $T^{\text{disb}}$, not the bill, is what leaves the escrow account.

**Other rules the engine encodes, by jurisdiction.** Reassessment frequency (in a state that reassesses every 4 years, $A$ is frozen between cycles); non-homestead caps (Florida caps non-homestead growth at 10% for non-school levies); statutory rate ceilings (a $\min$ with the cap on $\tau$); and phase-ins. Each is a deterministic function, which is why this step is called a rules engine rather than a model.

### Worked example

**The 2026 bill (already known, sets $E_0$).** Seller's assessed value \$205,000, exemption \$50,000, 2026 millage 18.50 (adopted in September 2026, so known):

$$\text{TV}_{2026} = 205{,}000 - 50{,}000 = 155{,}000, \qquad T_{2026} = 155{,}000 \times \frac{18.50}{1000} = \$2{,}867.50,$$

$$T^{\text{disb}}_{2026} = 0.96 \times 2{,}867.50 = \$2{,}752.80.$$

**The 2027 bill (forecast).** The house sold in June 2026, so $\text{sale}_{i,2027} = 1$ and the cap resets. The AVM (3.1) gives a January 2027 market value distribution with median \$414,000 (slightly below the purchase price, consistent with flat nominal prices and a soft Florida market). At the median,

$$A_{2027} = 0.85 \times 414{,}000 = \$351{,}900, \qquad \text{TV}_{2027} = 351{,}900 - 50{,}000 = \$301{,}900.$$

If the millage stayed at 18.50, the bill would be $301{,}900 \times 0.0185 = \$5{,}585.15$, nearly double last year's. That is the reset, and it is coming no matter what happens to rates or the housing market.

**Contrast: the long-time neighbor.** An identical house next door, owned since 2010, with the same \$205,000 assessed value but no sale. The cap branch gives $205{,}000 \times 1.03 = \$211{,}150$ and the market branch gives \$351,900; the minimum is \$211,150, so the cap binds and the neighbor's bill rises only about 3%. Same house, same market, wildly different escrow outlook. A model trained on past escrow payments cannot tell these two apart; the rules engine can.

### Output

For each account and each simulation draw: the assessed value $A_{i,t}$ and taxable value $\text{TV}_{i,t}$ for next year. The bill itself needs the rate from 2.2, so strictly 2.1 hands off a **function**: "give me a rate $\tau$ and I return $T^{\text{disb}} = 0.96 \cdot \text{TV} \cdot \tau / 1000$." It also produces a flag, $\text{sale}_{i,t}$ or "cap gap" $= \kappa_j V_{i,t} - A_{i,t-1}$, which goes straight into the report as the reason a payment is moving.

Aggregated over every parcel in jurisdiction $j$, the same engine produces the **jurisdiction-wide growth in taxable value of existing property**, $g_{j,t}$, which is the main input to 2.2.

### Hand-off

$\text{TV}_{i,t}$ (per draw) goes to 2.5, where it is multiplied by the rate draw from 2.2. The aggregate growth $g_{j,t}$ goes to 2.2.

### Validation

Backtest the engine on past years: feed it the actual prior-year roll and actual market values and check that it reproduces the actual assessed values. Disagreement is a rule-encoding bug, not noise, and should be close to zero.

---

## Step 2.2: Tax rate forecasting (hierarchical Bayesian time series)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| History of millage rates by taxing jurisdiction | $\tau_{j,t}$ | County tax collector and state revenue department records |
| Growth in taxable value of existing property | $g_{j,t}$ | Aggregated from 2.1 (history: actual rolls; forecast: simulated) |
| Budget pressure | $b_{j,t}$ | Local budget documents: prior-year expenditure growth minus revenue growth, in percentage points |
| Nesting of jurisdictions | $j \subset c(j) \subset s(j)$ | Each jurisdiction's county $c(j)$ and state $s(j)$ |

### Why the rate responds to assessments

Many states have **truth-in-taxation** or **rollback** laws. In Florida the process is called **TRIM** (Truth in Millage). Each summer the jurisdiction must compute the **rolled-back rate**, the rate that would raise the same revenue from existing property as last year:

$$\tau^{\text{RB}}_{j,t} = \tau_{j,t-1} \cdot \frac{\text{taxable value of existing property}_{t-1}}{\text{taxable value of existing property}_{t}} \approx \tau_{j,t-1}\, e^{-g_{j,t}}.$$

A government that adopts a rate above rolled-back must advertise a tax increase and hold hearings. So in log terms, full rollback would mean $\Delta \log \tau_{j,t} = -g_{j,t}$. In practice governments roll back only partially, keeping part of the assessment windfall. The model estimates *how* partially.

### Process

Let $y_{j,t} = \log \tau_{j,t} - \log \tau_{j,t-1}$ be the log change in millage. The **observation equation** is

$$y_{j,t} = \alpha_j + \psi_g\, g_{j,t} + \psi_b\, b_{j,t} + \varepsilon_{j,t}, \qquad \varepsilon_{j,t} \sim \mathcal{N}(0, \sigma_\varepsilon^2).$$

- $\alpha_j$ is jurisdiction $j$'s baseline drift in its rate, its "habit."
- $\psi_g$ is the rollback response. $\psi_g = -1$ is full rollback; $\psi_g = 0$ is none.
- $\psi_b$ is the budget-pressure response: when spending outruns revenue, rates rise.
- $\varepsilon_{j,t}$ is the year-specific shock (an election, a referendum, a one-off project).

The **hierarchy** is on the baselines:

$$\alpha_j \sim \mathcal{N}\big(\alpha_{c(j)},\, \omega_J^2\big), \qquad \alpha_c \sim \mathcal{N}\big(\alpha_{s(c)},\, \omega_C^2\big), \qquad \alpha_s \sim \mathcal{N}\big(0,\, \omega_S^2\big),$$

with weakly informative priors (for example half-normal) on the standard deviations $\omega_J, \omega_C, \omega_S, \sigma_\varepsilon$, and normal priors on $\psi_g, \psi_b$. The slopes can also vary by state, because the legal regime differs (Texas, for example, requires voter approval above a fixed revenue growth limit, which should show up as a different $\psi$).

**Why hierarchical.** A small fire district may have only six years of usable history. Estimating its $\alpha_j$ alone would be noisy; assuming it equals the county average would ignore real differences. **Partial pooling** is the compromise: each jurisdiction's estimate is pulled toward its county's, and pulled harder the less data it has. The model learns from the data how much jurisdictions differ ($\omega_J$), and therefore how hard to pull.

**Estimation.** Fit the full joint posterior by MCMC (Hamiltonian Monte Carlo, for example the NUTS sampler). The output is a large set of posterior draws of all parameters. To see what pooling does, it helps to look at the formula that holds conditional on the variance parameters: the posterior mean of $\alpha_j$ is a precision-weighted average,

$$\hat\alpha_j = w_j\, \bar y^{\,*}_j + (1 - w_j)\, \alpha_{c(j)}, \qquad w_j = \frac{n_j / \sigma_\varepsilon^2}{n_j / \sigma_\varepsilon^2 + 1/\omega_J^2},$$

where $\bar y^{\,*}_j$ is the jurisdiction's average rate change after removing the $\psi_g g + \psi_b b$ part, and $n_j$ its number of years. The posterior variance is $1 / \big(n_j/\sigma_\varepsilon^2 + 1/\omega_J^2\big)$.

**Predictive distribution.** For next year, the forecast of $y_{j,t+1}$ combines three sources of uncertainty: the year shock $\varepsilon$, uncertainty in $\alpha_j$, and uncertainty in next year's assessment growth $g_{j,t+1}$ (which is itself a forecast from 2.1 and the AVM). With posterior draws you get this automatically by simulation. Under a normal approximation it is

$$y_{j,t+1} \sim \mathcal{N}\Big(\hat\alpha_j + \psi_g\, \mathbb{E}[g_{j,t+1}] + \psi_b\, b_{j,t+1},\;\; \sigma_\varepsilon^2 + \operatorname{Var}(\alpha_j) + \psi_g^2 \operatorname{Var}(g_{j,t+1})\Big),$$

and $\tau_{j,t+1} = \tau_{j,t}\, e^{y_{j,t+1}}$.

### Worked example

Account A's taxing district has six years of history. Estimated quantities:

- $\sigma_\varepsilon = 3\%$ (year shocks are sizable), $\omega_J = 1\%$ (jurisdictions in this county don't differ much in habit).
- Jurisdiction's own adjusted average change $\bar y^{\,*}_j = +1.8\%$/yr; county baseline $\alpha_c = +0.6\%$/yr.
- $\psi_g = -0.45$ (governments give back 45% of the assessment windfall and keep 55%), $\psi_b = +0.30$.

**Pooling.**

$$w_j = \frac{6 / 0.03^2}{6/0.03^2 + 1/0.01^2} = \frac{6{,}667}{6{,}667 + 10{,}000} = 0.40,$$

$$\hat\alpha_j = 0.40 \times 1.8\% + 0.60 \times 0.6\% = 1.08\%, \qquad \operatorname{sd}(\alpha_j) = \sqrt{1/16{,}667} = 0.77\%.$$

With only six noisy years, 60% of the weight goes to the county. The district's apparent 1.8% habit is mostly treated as luck.

**2027 inputs.** Taxable value of existing property in the district is forecast to grow $g = 3.0\%$ (sd 1.5%): market values are flat, but capped homesteads keep catching up by up to 3% a year and sales keep resetting. Budget pressure $b = +1.5$ points (cost inflation with CPI at 3.4%).

**Predictive mean and spread.**

$$\mathbb{E}[y] = 1.08\% + (-0.45)(3.0\%) + 0.30(1.5\%) = 1.08\% - 1.35\% + 0.45\% = +0.18\%,$$

$$\operatorname{Var}(y) = 0.03^2 + 0.0077^2 + 0.45^2 \times 0.015^2 = 0.000900 + 0.000060 + 0.000046 = 0.001006, \quad \operatorname{sd}(y) = 3.17\%.$$

So the 2027 millage is centered at $18.50 \times e^{0.0018} = 18.53$ mills, with

$$\text{P10} = 18.50\, e^{0.0018 - 1.2816 \times 0.0317} = 17.80, \qquad \text{P90} = 18.50\, e^{0.0018 + 1.2816 \times 0.0317} = 19.30 \text{ mills}$$

(1.2816 is the standard normal 90th percentile). The rate is genuinely uncertain, about $\pm 4\%$, but it is small next to the reset: a 4% rate move on a \$5,600 bill is about \$220 a year, versus a \$2,700 jump from the reset.

**Note on the conditional form.** The $\psi_g^2 \operatorname{Var}(g)$ term is in the marginal variance above, but in 2.5 we will simulate $g$ jointly with the house's value. So what 2.2 hands off is the **conditional** distribution

$$y \mid g \sim \mathcal{N}\big(1.08\% + 0.45\% - 0.45\,g,\;\; 0.03^2 + 0.0077^2\big) = \mathcal{N}\big(1.53\% - 0.45\,g,\;\; (3.10\%)^2\big).$$

Using the marginal distribution *and* drawing $g$ separately would double-count that uncertainty and destroy the negative correlation between market values and rates.

### Output

A predictive sampler: given a draw of jurisdiction growth $g_{j,t+1}$, return a draw of $\tau_{j,t+1}$. Equivalently, a set of posterior draws $(\alpha_j, \psi_g, \psi_b, \sigma_\varepsilon)$ from which 2.5 constructs rate draws.

### Hand-off

To 2.5, where each draw of $\tau$ is combined with the matching draw of taxable value from 2.1 to produce the tax disbursement.

### Validation

Out-of-time backtest: fit through year $t$, forecast $t+1$, compare to the adopted rate. Check interval coverage (about 80% of adopted rates should fall in the P10 to P90 band). A practical bonus: by late July each year, Florida jurisdictions publish a **proposed** (TRIM notice) rate, which caps what they can adopt in September. Once it exists, it replaces the forecast's upper tail.

---

## Step 2.3: Insurance premium model (log-link Gamma GLM + distributed lag)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Property rating characteristics (coverage amount, construction year, roof age and shape, wind-mitigation features, deductible, territory) | $x_{i1}, \dots, x_{iJ}$ | Servicer's insurance declarations page, assessor data, permits |
| Expected annual loss from climate hazards | $\text{EAL}_i$ | Climate model, step 3.2 |
| Rate filings: base rates and relativity tables by insurer and territory | | SERFF (the multistate System for Electronic Rate and Form Filing) and Florida's own OIR (Office of Insurance Regulation) filing portal |
| Aggregate premium studies | | State insurance departments, industry studies |
| Reinsurance price index (log change each renewal season) | $r_q$ | Published reinsurance rate-on-line indices |
| Replacement cost index | | Construction cost indices |
| Account's current premium | $I_{i,t}$ | Servicer escrow disbursement records |

**Terms.** A **premium** is what the policyholder pays per year. **Coverage A** is the insured dwelling amount, roughly the cost to rebuild. A **rate filing** is the insurer's submission to the state regulator to change prices, made up of a **base rate** (the price of a reference home in a territory) and **relativities** (multipliers for each characteristic). **Reinsurance** is insurance that insurers buy to cover their own catastrophe losses; in Florida most of it renews on June 1 each year, and its price is quoted as **rate on line** (premium as a share of coverage limit). A **wind-mitigation credit** is a discount for features like a hip roof or hurricane straps.

### Process: two layers

**Layer 1: the cross-section (what does this house cost to insure relative to others?).** Insurers price multiplicatively: premium = base rate × product of relativities. Taking logs turns that into a sum, which is exactly a GLM with a log link:

$$\log \mu_{i,t} = \beta_{0,t} + \sum_{j=1}^{J} \beta_j\, x_{ij,t}, \qquad \text{Premium}_{i,t} \sim \text{Gamma}\big(\text{mean } \mu_{i,t},\ \text{variance } \phi\, \mu_{i,t}^2\big).$$

- $\mu_{i,t}$ is the expected premium for property $i$ in year $t$.
- $\beta_{0,t}$ is the log base-rate level, which changes over time (layer 2 models how).
- $\beta_j$ is the log relativity for characteristic $j$; $e^{\beta_j}$ is the multiplier. For a continuous variable entered in logs, like $\log(\text{Coverage A})$, $\beta_j$ is an **elasticity**: a 1% larger coverage raises the premium by $\beta_j$%.
- **Gamma errors** mean the standard deviation is proportional to the mean ($\sqrt{\phi}$ is the coefficient of variation), so a \$10,000 policy has proportionally the same noise as a \$2,000 one. That matches premium data far better than constant-variance least squares. Estimate by maximum likelihood (iteratively reweighted least squares).

Training data are the filed rate manuals themselves (the GLM essentially reconstructs each insurer's pricing algorithm and averages across insurers in a territory) plus aggregate premium studies that pin down the overall level. $\log \text{EAL}_i$ from 3.2 enters as one of the $x_{ij}$, which is how physical climate risk gets into the premium.

**Layer 2: time dynamics (how does the base level move?).** Premiums react to reinsurance prices with a lag: reinsurance renews (June), insurers file new rates (months later), the regulator approves, and the new rates reach each policyholder only at their own renewal date over the following twelve months. A **distributed lag** model captures that delay. Let $\Delta\beta_{0,q}$ be the quarter-$q$ change in the log base-rate level for a state and territory:

$$\Delta \beta_{0,q} = c_0 + \sum_{k=0}^{K} \delta_k\, r_{q-k} + e_q,$$

where $r_{q}$ is the log change in the reinsurance index in quarter $q$ (non-zero only in the June quarter), $\delta_k$ is how much of a reinsurance change shows up $k$ quarters later, and $c_0$ is the quarterly trend from everything else (claim severity, litigation costs, expenses). The **long-run pass-through** is $\sum_k \delta_k$: the share of a reinsurance price change that ends up in premiums.

With $K = 8$ quarters and few data points, nine free $\delta_k$ would be erratic. **Almon smoothing** forces them to lie on a low-order polynomial in the lag:

$$\delta_k = a_0 + a_1 k + a_2 k^2,$$

so only three parameters $(a_0, a_1, a_2)$ are estimated (by substituting the polynomial into the regression and running least squares on the transformed regressors). Adding endpoint constraints ($\delta_{-1} = \delta_{K+1} = 0$) gives the natural hump shape: nothing immediately, a peak around a year, fading out by two years.

**Combining the layers for one account.** We usually observe the account's actual current premium $I_{i,t}$, and the account's deviation from the GLM's prediction (its residual, reflecting its particular insurer and history) tends to persist. So we project from the actual premium rather than from $\mu_{i,t}$, and apply only the *change* the model predicts:

$$\ell^{\text{d}}_{i} \equiv \log\frac{I_{i,t+1}}{I_{i,t}} = \underbrace{\sum_{q \in \text{renewal year}} \Delta\beta_{0,q}}_{\text{base-rate drift}} + \underbrace{\sum_j \beta_j\, \Delta x_{ij}}_{\text{change in this house's characteristics}} + \nu_i, \qquad \nu_i \sim \mathcal{N}(0, \sigma_\nu^2).$$

The superscript d stands for **drift**. The first term comes from layer 2; the second uses layer 1's coefficients applied to what changed about the house (coverage indexed up for inflation, a roof one year older, a revised EAL from 3.2); $\nu_i$ is the account-level noise (insurer-specific filings, re-underwriting), with $\sigma_\nu$ estimated from year-over-year changes in servicer premium records.

For accounts with no observed premium (or a new purchase with no policy yet), use the GLM prediction $\mu_{i,t}$ directly.

### Worked example

**Layer 1: Account A's premium level.** Suppose the fitted territory GLM gives a reference-home base of $e^{\beta_0} = \$3{,}076$ and these relativities:

| Characteristic | Account A | Relativity |
|---|---|---|
| Coverage A, elasticity 0.85 vs. \$250k reference | \$360,000 | $(360/250)^{0.85} = 1.363$ |
| Construction year (pre-2002 building code) | 1998 | 1.25 |
| Roof age | 12 years | 1.05 |
| Hip roof (wind-mitigation credit) | yes | 0.88 |
| EAL from 3.2, elasticity 0.4 vs. \$1,600 reference | \$2,100 | $(2100/1600)^{0.4} = 1.115$ |

$$\mu_{A} = 3{,}076 \times 1.363 \times 1.25 \times 1.05 \times 0.88 \times 1.115 \approx \$5{,}400,$$

which matches the actual premium, so the account's residual is about zero.

**Layer 2: Almon lag weights.** Suppose the fit gives $a_0 = 0$, $a_1 = 0.0286$, $a_2 = -0.00357$, so $\delta_k = 0.00357\, k (8 - k)$:

| Lag $k$ (quarters) | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|---|---|---|---|---|---|---|---|---|---|
| $\delta_k$ | 0 | .025 | .043 | .054 | .057 | .054 | .043 | .025 | 0 |

Long-run pass-through $\sum_k \delta_k = 0.30$: a 10% rise in reinsurance prices eventually raises premiums about 3%, which is plausible since reinsurance is a large but not dominant share of a Florida homeowners insurer's costs.

**The June 2027 renewal.** For the worked example, take the reinsurance index to have changed by $-8\%$ at June 2025 and $-5\%$ at June 2026 (log changes $-0.0834$ and $-0.0513$). The renewal in June 2027 sees the four quarterly base-rate changes from Q3 2026 through Q2 2027. Counting quarters from June 2025 = 0, those are quarters 5 to 8:

- The June 2026 shock (quarter 4) enters those quarters at lags 1 to 4: weight $\delta_1 + \delta_2 + \delta_3 + \delta_4 = 0.179$, contribution $0.179 \times (-0.0513) = -0.0092$.
- The June 2025 shock (quarter 0) enters at lags 5 to 8: weight $0.121$, contribution $0.121 \times (-0.0834) = -0.0101$.
- The June 2027 shock enters at lag 0 with weight $\delta_0 = 0$: **no effect on this renewal.** That is a useful property: the next renewal is mostly determined by reinsurance history that is already known.
- Trend: $4 \times c_0 = 4 \times 0.005 = +0.020$.

Base-rate drift: $0.020 - 0.0092 - 0.0101 = +0.0007$, essentially flat; softer reinsurance offsets cost trend.

**Change in characteristics.** Coverage A is indexed up 4% for construction cost inflation (materials costs are up with the oil shock). Through the GLM's coverage elasticity: $0.85 \times \log(1.04) = +0.0333$. Roof aging from 12 to 13 years doesn't cross a relativity band; EAL unchanged.

$$\mathbb{E}[\ell^{\text{d}}_A] = 0.0007 + 0.0333 = 0.0341 \quad (\text{about } +3.5\%), \qquad \sigma_\nu = 0.06.$$

At the central value, $I_{2027} = 5{,}400 \times e^{0.0341} = \$5{,}587$. The **mean** premium is slightly higher because of lognormal skew: $5{,}400 \times e^{0.0341 + 0.06^2/2} = \$5{,}597$.

### Output

For each account, the drift distribution $\ell^{\text{d}}_i \sim \mathcal{N}(m_i, \sigma_\nu^2)$: here $m_A = 0.0341$, $\sigma_\nu = 0.06$. Also the GLM's decomposition of *why* (base rate vs. coverage inflation vs. characteristics), which feeds the report.

### Hand-off

To 2.4, which decides whether the account gets the drift or a jump, and then to 2.5.

### Validation

Out-of-time: predict each account's renewal from last year's premium and compare to the actual disbursement on the escrow record. Check the Gamma assumption by plotting squared Pearson residuals against fitted means (should be flat).

---

## Step 2.4: Jump risk (calibrated classifier + mixture)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Account features: roof age, coastal distance, EAL (3.2), construction, current insurer, insurer financial strength, insurer's exits/withdrawals announced, territory share in the insurer of last resort, prior claims | $\mathbf{h}_i$ | Servicer records, assessor, 3.2, regulator notices, insurer financial statements |
| Historical labels: did a shock happen at renewal? | $\xi_i \in \{0,1\}$ | Servicer escrow records: a change of insurance **payee** together with an outsized premium change, or a force-placed policy |
| Historical shock sizes | $\ell^{\text{j}}$ | Same records: log premium change in shock cases |
| Drift distribution | $\ell^{\text{d}}_i$ | Step 2.3 |

**Terms.** A **nonrenewal** is the insurer declining to renew a policy. An **insurer exit** is a company leaving a state or territory (or failing). The **insurer of last resort** is the state-created insurer that covers homes no private insurer will: in Florida, **Citizens Property Insurance Corporation**; elsewhere the FAIR plans. **Surplus lines** are non-admitted insurers that cover hard risks at higher prices. **Force-placed insurance** is a policy the servicer buys itself when the borrower's lapses, usually much more expensive. All of these show up as one thing on the escrow account: a sudden, large change in the premium disbursed.

### Process

**The classifier.** A gradient-boosting classifier (an ensemble of shallow decision trees, each fit to the errors of the previous ones) produces a raw score

$$s_i = F(\mathbf{h}_i) = \sum_{m=1}^{M} \nu\, f_m(\mathbf{h}_i),$$

where each $f_m$ is a small tree and $\nu$ the learning rate, trained to minimize log loss on the label $\xi_i$ ("shock at the next renewal"). Boosting suits this problem because the risk is full of thresholds and interactions: a roof crossing 15 years matters a lot *in a coastal county with an insurer under financial stress* and much less otherwise.

**Calibration.** Raw boosting scores rank accounts well but are not reliable probabilities (boosting tends to push them toward the extremes). Since 2.5 will literally flip a coin with probability $p_i$, the probability must be right, so we **calibrate** on held-out data. Two standard methods:

- **Platt scaling:** fit a one-variable logistic regression on the score, $p_i = 1 / \big(1 + e^{a^{\text{P}} s_i + b^{\text{P}}}\big)$, with $a^{\text{P}}, b^{\text{P}}$ estimated on a held-out set. Smooth, needs little data.
- **Isotonic regression:** fit the best non-decreasing step function from score to observed frequency (pool-adjacent-violators algorithm). More flexible, needs more data.

**The mixture.** Let $\xi_i \sim \text{Bernoulli}(p_i)$ be the shock indicator. The log premium change is

$$\ell_i = (1 - \xi_i)\, \ell^{\text{d}}_i + \xi_i\, \ell^{\text{j}}_i, \qquad I_{i,t+1} = I_{i,t}\, e^{\ell_i},$$

where $\ell^{\text{d}}_i$ is the drift from 2.3 and $\ell^{\text{j}}_i$ is a **jump** drawn from the empirical distribution of historical shock sizes (optionally stratified by shock type or territory). The jump *replaces* the drift because a shocked policy is repriced from scratch by a new insurer; it does not get both.

The mixture's moments, which make the shape clear:

$$\mathbb{E}[\ell_i] = (1 - p_i)\, \mu_{\text{d}} + p_i\, \mu_{\text{j}}, \qquad \operatorname{Var}(\ell_i) = (1-p_i)\big(\sigma_{\text{d}}^2 + \mu_{\text{d}}^2\big) + p_i\big(\sigma_{\text{j}}^2 + \mu_{\text{j}}^2\big) - \big(\mathbb{E}[\ell_i]\big)^2.$$

The result is a distribution with a tall peak at a few percent and a long right tail: exactly the shape a single normal or lognormal model would get wrong.

### Worked example

**Probability.** Account A: roof 12 years, coastal distance 9 miles, current insurer adequately capitalized, territory with moderate last-resort share. Raw GBM score $s_A = 0.9$. Platt parameters fitted on held-out data: $a^{\text{P}} = -1.1$, $b^{\text{P}} = 2.98$.

$$p_A = \frac{1}{1 + e^{-1.1 \times 0.9 + 2.98}} = \frac{1}{1 + e^{1.99}} = \frac{1}{1 + 7.32} = 0.120.$$

**Calibration check.** On the held-out set, the accounts with predicted probability between 10% and 15% number 2,000, and 248 of them had a shock: 12.4%, inside that bin. Repeat for every bin; that is a reliability diagram.

**Jump sizes.** Historical shock premium changes in this territory (ten deciles of the empirical distribution): +12%, +20%, +28%, +35%, +42%, +48%, +60%, +75%, +95%, +130%. In logs: $\mu_{\text{j}} = 0.412$, $\sigma_{\text{j}} = 0.213$.

**Mixture moments.** With $\mu_{\text{d}} = 0.0341$, $\sigma_{\text{d}} = 0.06$, $p = 0.12$:

$$\mathbb{E}[\ell_A] = 0.88 \times 0.0341 + 0.12 \times 0.412 = 0.079, \qquad \operatorname{sd}(\ell_A) = 0.154.$$

In dollars: without a jump, the mean renewal is \$5,597; given a jump, the mean is \$8,343. Mixing,

$$\mathbb{E}[I_{2027}] = 0.88 \times 5{,}597 + 0.12 \times 8{,}343 = \$5{,}927 \quad (+9.8\%).$$

The **expected** increase (9.8%) is almost three times the **typical** increase (3.5%). The difference is entirely the 12% chance of a jump. Reporting only the typical case would understate risk; reporting only the mean would overstate what most borrowers see. That is why the report gives quantiles.

### Output

For each account: calibrated $p_i$, the jump-size distribution, and a sampler for $\ell_i$. Also the top features driving $p_i$ (from tree-based attributions such as SHAP values), which tell the servicer *what to do* (for example: a roof inspection or a wind-mitigation inspection before renewal).

### Hand-off

To 2.5, where each draw flips the $\xi$ coin and draws either drift or jump. The same shock model is reused in 3.x (the insurance channel of climate pricing) and 5.x (delinquency risk), per the shared-components table.

### Validation

Calibration plots and Brier score on out-of-time data (train through 2024, test on 2025 renewals). Check that the jump-size distribution is stable across years or re-estimate it with recent years weighted more heavily.

---

## Step 2.5: Combination and escrow mechanics (Monte Carlo + quantiles)

### Inputs

| Input | From |
|---|---|
| Market value distribution, with its common (market-wide) and idiosyncratic parts | 3.1 |
| Taxable value function | 2.1 |
| Conditional rate sampler $\tau \mid g$ | 2.2 |
| Premium drift sampler | 2.3 |
| Jump probability and size sampler | 2.4 |
| Current escrow state: last bills, current payment $E_0$, required starting balance | Servicer records |
| Loan P&I, borrower income and other debts | Origination file |
| Escrow rules (Regulation X §1024.17) | Statute |

### Why simulate rather than plug in expected values

Two reasons. First, the escrow rules are **nonlinear**: a shortage is collected but a surplus is refunded, and the low-point calculation switches depending on which bill is larger. Plugging in expected tax and expected premium gives the escrow payment at the expected bills, which is not the expected escrow payment. Second, the client needs a **distribution** (P10/P50/P90 and threshold probabilities), and the mixture in 2.4 makes that distribution skewed.

### Process, part 1: simulation

For each draw $n = 1, \dots, N$ (we use $N = 200{,}000$, plenty for stable P90s):

1. **Market factor.** Draw the common county-level price shock $u^{\text{mkt}}_n \sim \mathcal{N}(0, 0.035^2)$ and the property's idiosyncratic AVM error $u^{\text{idio}}_n \sim \mathcal{N}(0, 0.049^2)$. Value: $V_n = \tilde V \, e^{u^{\text{mkt}}_n + u^{\text{idio}}_n}$, with $\tilde V$ the AVM median. (Total log sd $\approx 0.06$, matching the AVM's conformal interval.)
2. **Jurisdiction growth**, correlated with the market factor: $g_n = \bar g + \lambda\, u^{\text{mkt}}_n + \zeta_n$, where $\lambda$ is estimated from history (how much district-wide taxable growth moves with county prices) and $\zeta_n$ is independent noise.
3. **Tax (2.1 + 2.2).** $\text{TV}_n$ from the rules engine at $V_n$; $y_n$ from $\mathcal{N}(1.53\% - 0.45\, g_n,\ 3.10\%^2)$; $\tau_n = \tau_{2026}\, e^{y_n}$; $T^{\text{disb}}_n = 0.96 \cdot \text{TV}_n \cdot \tau_n / 1000$.
4. **Insurance (2.3 + 2.4).** Draw $\xi_n \sim \text{Bernoulli}(p_i)$; if 0 draw $\ell_n$ from the drift, if 1 from the jump distribution; $I_n = I_{2026}\, e^{\ell_n}$.
5. **Escrow analysis** (part 2, below) on the bills $(T^{\text{disb}}_n, I_n)$ to get $E_{1,n}$ and $\Delta E_n = E_{1,n} - E_0$.

Then the forecast is the empirical distribution of $\{\Delta E_n\}$: its quantiles, its mean, and probabilities like $\Pr(\Delta E > \$300)$.

Tax and insurance draws are independent in this example. In a portfolio run, add the dependence that matters: a single hurricane season raises premiums *and* can lower values in the same county, so in the portfolio version, draw a county catastrophe shock that enters both the jump probability and the market factor.

### Process, part 2: escrow mechanics (RESPA / Regulation X)

Define for an analysis covering a 12-month computation year:

- $D$: projected **annual disbursements**, $D = T^{\text{disb}} + I$, using the most recent bills (that is what servicers do).
- Base monthly payment: $D / 12$.
- **Cushion** $C$: the maximum reserve the servicer may hold, one-sixth of annual disbursements, $C = D/6$ (two months' worth).
- Monthly balance $B_m$: deposits of the escrow payment at the start of each month $m = 1, \dots, 12$, bills paid at the end of their month (insurance at $m = 6$, tax at $m = 11$).
- **Required starting balance** $B^{*}$: the smallest starting balance such that the projected **low point** (lowest month-end balance) equals the cushion,

$$B^{*} = C + \max_{m} \Big(\text{cumulative disbursements through } m \; - \; m \cdot \tfrac{D}{12}\Big).$$

With only two bills, the maximum is at either month 6 or month 11. At month 11, cumulative disbursements $= D$, so the term is $D - 11 \cdot D/12 = D/12$. At month 6 it is $I - 6 \cdot D/12$. Hence

$$B^{*} = \frac{D}{6} + \max\Big(I - \frac{D}{2},\ \frac{D}{12}\Big).$$

At the analysis, the servicer compares $B^{*}$ with the **actual projected balance** $B^{\text{act}}$ carried in from the year just ended:

- **Shortage** $S = B^{*} - B^{\text{act}} > 0$. If $B^{\text{act}}$ is itself negative (the servicer had to advance money to pay a bill), the part below zero is technically a **deficiency** and the part from zero up to $B^{*}$ a shortage. Regulation X lets the servicer spread a shortage of at least one month's payment over 12 or more months, and a deficiency over 2 to 12 months. We assume, as is common, that the whole amount is spread over 12 months.
- **Surplus** $S < 0$: refunded to the borrower (if \$50 or more), so it does not reduce the new payment.

New monthly escrow payment:

$$E_1 = \frac{D_1}{12} + \frac{\max(S, 0)}{12}.$$

Note the structure of the increase. The new payment includes the **new level** $D_1/12$ *and* a **catch-up** $S/12$ for the money that should have been collected during the year but wasn't. In the following year the catch-up drops off. Borrowers experience this as a big spike followed by a partial retreat. The asymmetry between $\max(S, 0)$ and a refunded surplus is the nonlinearity that makes simulation necessary.

### Worked example: the December 2026 analysis (sets $E_0$, known today)

Bills known in December 2026: $T^{\text{disb}}_{2026} = \$2{,}752.80$ (from 2.1) and $I_{2026} = \$5{,}400$.

$$D_0 = 2{,}752.80 + 5{,}400 = \$8{,}152.80, \qquad E_0 = D_0 / 12 = \$679.40, \qquad C_0 = D_0/6 = \$1{,}358.80.$$

Low point: $I - D/2 = 5{,}400 - 4{,}076.40 = 1{,}323.60$ beats $D/12 = 679.40$, so the June insurance bill is the binding month:

$$B^{*}_0 = 1{,}358.80 + 1{,}323.60 = \$2{,}682.40.$$

We assume the account enters 2027 at exactly this balance (any 2026 shortage settled at that analysis).

### Worked example: living through 2027 on the central path

Take every input at its central value: $V = \$414{,}000$, $g = 3\%$, $\tau = 18.53$ mills, no jump, drift at its central value.

- Tax: $\text{TV} = \$301{,}900$; bill $= 301{,}900 \times 18.533/1000 = \$5{,}595.21$; disbursement $= 0.96 \times 5{,}595.21 = \$5{,}371.40$.
- Insurance: $I_{2027} = 5{,}400\, e^{0.0341} = \$5{,}587.04$.

The account still collects the old $E_0 = \$679.40$ every month. Month-end balances:

| Month | Jan | Feb | Mar | Apr | May | **Jun** | Jul | Aug | Sep | Oct | **Nov** | Dec |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Event | | | | | | pay insurance | | | | | pay tax | |
| Balance (\$) | 3,362 | 4,041 | 4,721 | 5,400 | 6,079 | 1,172 | 1,851 | 2,531 | 3,210 | 3,889 | **−803** | −123 |

In November the account goes **negative**: the servicer advances \$803 to pay the tax bill. Year-end balance:

$$B^{\text{act}} = B^{*}_0 + 12 E_0 - I_{2027} - T^{\text{disb}}_{2027} = 2{,}682.40 + 8{,}152.80 - 5{,}587.04 - 5{,}371.40 = -\$123.24.$$

### Worked example: the December 2027 analysis (sets $E_1$)

$$D_1 = 5{,}371.40 + 5{,}587.04 = \$10{,}958.44, \qquad \frac{D_1}{12} = \$913.20, \qquad C_1 = \frac{D_1}{6} = \$1{,}826.41.$$

Low point: now $I - D/2 = 5{,}587.04 - 5{,}479.22 = 107.82$, which is smaller than $D/12 = 913.20$. The tax bill has become the larger one, so November is now the binding month:

$$B^{*}_1 = 1{,}826.41 + 913.20 = \$2{,}739.61.$$

Shortage (including the \$123.24 deficiency):

$$S = 2{,}739.61 - (-123.24) = \$2{,}862.85, \qquad \frac{S}{12} = \$238.57.$$

$$E_1 = 913.20 + 238.57 = \$1{,}151.77, \qquad \Delta E = 1{,}151.77 - 679.40 = +\$472.37.$$

Read that decomposition: \$233.80 of the increase is the **new level** ($913.20 - 679.40$) and \$238.57 is **catch-up**. About half the shock is temporary. In January 2029, if bills stay flat, the payment falls back to about \$913.

### Worked example: Monte Carlo results

Running all 200,000 draws:

| Quantity | P10 | P50 | P90 | Mean |
|---|---|---|---|---|
| Market value $V$ (\$) | 383,300 | 414,000 | 447,200 | 414,800 |
| Tax disbursement 2027 (\$) | 4,881 | 5,371 | 5,905 | 5,384 |
| Insurance premium 2027 (\$) | 5,196 | 5,646 | 6,579 | 5,932 |
| New escrow payment $E_1$ (\$/mo) | 1,039 | 1,170 | 1,398 | 1,224 |
| **Change $\Delta E$ (\$/mo)** | **+360** | **+491** | **+718** | **+544** |

Things to notice:

- **The central path (+\$472) is not the median (+\$491), and neither is the mean (+\$544).** The 12% jump probability pulls the whole distribution right; the mean is pulled furthest. This is the Jensen's-inequality point from above, made concrete.
- **Conditional on a jump** ($\xi = 1$, 12% of draws), the median $\Delta E$ is **+\$920**; without one it is +\$474.
- $\Pr(\Delta E > \$300) = 97\%$ and $\Pr(\Delta E > \$400) = 81\%$: a large increase is close to certain. The balance goes negative in November in 95% of draws.

### Attribution: how much is tax, how much is insurance?

There are two different questions, and the report answers both.

**(a) Where does the expected increase come from?** Rerun the central path changing one bill at a time:

| Scenario | $\Delta E$ (\$/mo) |
|---|---|
| Tax changes, insurance held at 2026 premium | +437.30 |
| Insurance changes, tax held at 2026 bill | +41.56 |
| Both change (central path) | +472.37 |
| Interaction (both − sum of the two) | −6.49 |

Tax drives about 92% of the expected increase. The small interaction exists because the low-point month switched from June to November when tax overtook insurance.

**(b) Where does the uncertainty come from?** Hold one component at its *expected* value across all draws and measure the remaining variance of $\Delta E$:

$$\operatorname{Var}\big(\Delta E \mid \text{insurance fixed at } \mathbb{E}[I]\big) = 5{,}648 \quad (\text{sd } \$75), \qquad \operatorname{Var}\big(\Delta E \mid \text{tax fixed at } \mathbb{E}[T]\big) = 55{,}954 \quad (\text{sd } \$237),$$

against a total variance of 61,976 (sd \$249). Normalizing the two parts, **tax accounts for about 9% of the uncertainty and insurance about 91%** (the small remainder is interaction).

So for Account A, the honest summary is: *the increase is mostly tax, and it is nearly certain; the risk around it is mostly insurance.* Those call for different actions: the tax part calls for an early warning to the borrower, and the insurance part calls for a pre-renewal check on the roof and the policy.

### Affordability cross-check (DTI)

**DTI** (debt-to-income ratio) is total monthly debt payments divided by gross monthly income; 43% is a common affordability line (the old Qualified Mortgage limit).

| | PITI (\$/mo) | DTI |
|---|---|---|
| At origination / 2027 ($E_0$) | 2,803 | $(2{,}803 + 650)/9{,}500 = 36.3\%$ |
| 2028, central path ($E_1 = 1{,}152$) | 3,276 | 41.3% |
| 2028, P90 ($E_1 = 1{,}398$) | 3,522 | 43.9% |
| 2029, catch-up rolled off (central) | 3,037 | 38.8% |

The payment rises 17% at the median, and $\Pr(\text{DTI} > 43\%) = 16\%$. With rates at 7.3%, this borrower (at 6.5%) has no refinance escape, which is where the shared rate simulator (1.5) enters: for a portfolio, it gives the probability that rates fall far enough for refinancing to be an option. Under today's curve that probability is small. Income is from origination and may be stale, so DTI is a **signal for delinquency risk**, not a determination.

### Output

Per account: the distribution of $E_1$ and $\Delta E$ (quantiles, mean, threshold probabilities), the split into new-level vs. catch-up, both attributions, and DTI breach probability.

### Hand-off

To the report, and to the portfolio layer (sum across accounts, aggregate by geography and driver). The per-account $\Delta E$ distributions also feed 5.x as a delinquency-risk feature.

### Validation

Backtest on last year's analyses: forecast each account's 2026 payment change from data available 14 months earlier and check that about 80% of actual changes land inside P10 to P90, and that the threshold probabilities are calibrated (of accounts with $\Pr(\Delta E > \$300) \approx 0.5$, about half actually exceeded \$300). Validate each upstream step separately first, since errors compound.

---

## The report

### Per-account card (Account A)

- **Escrow payment:** \$679/mo now; forecast for January 2028 is \$1,170 (P10 \$1,039, P90 \$1,398). Increase +\$491 at the median.
- **Why:** the tax assessment resets to market value after the June 2026 sale (cap gap \$146,900 of assessed value), roughly doubling the tax bill. Insurance adds a little on average, but carries a 12% chance of a nonrenewal-type jump.
- **Structure:** about half the increase is a one-year catch-up for the 2027 shortfall; the payment should settle near \$913 in 2029.
- **Attribution:** expected increase about 92% tax; uncertainty about 91% insurance.
- **Affordability:** DTI 36% → 41% (median), 16% chance above 43%.
- **Actions:** notify the borrower now, 14 months ahead of the analysis; offer a voluntary payment increase from January 2027 to spread the shock (which would avoid the deficiency entirely); schedule a wind-mitigation inspection before the June 2027 renewal.

### Portfolio table (illustrative, four accounts run through the same engine)

| Account | Situation | $E_0$ | $\Delta E$ P10 / P50 / P90 | Mean | Tax share of uncertainty | Jump prob. | Flags |
|---|---|---|---|---|---|---|---|
| A | Recent buyer, cap reset | \$679 | +359 / +491 / +718 | +544 | 9% | 12% | Likely shock, tail risk |
| B | Long-time homestead, coastal, 17-yr roof | \$850 | −7 / +148 / +1,059 | +356 | ~0% | 30% | Tail risk |
| C | Recent buyer, inland, cheap policy | \$378 | +318 / +399 / +492 | +407 | 39% | 3% | Likely shock |
| D | Long-time homestead, inland | \$412 | −6 / +35 / +101 | +51 | 1% | 4% | None |

Flag rules used here: **likely shock** if P50 $\Delta E \ge \$150$; **tail risk** if P90 $\Delta E \ge \$500$. Account B shows why quantiles matter: its median looks mild, but its P90 is the worst in the book, and the driver is entirely insurance (cap keeps its tax nearly flat). Account C is the opposite: a near-certain tax-driven increase with little tail. They need different interventions, and a point forecast would have ranked them backwards.

### Portfolio heat map

Rows are counties (or zip codes), columns are drivers (**cap reset**, **millage change**, **premium drift**, **jump risk**), and each cell is that driver's share of the portfolio's total expected $\Delta E$ dollars in that area, with a second view showing the share of accounts flagged. This tells a servicer where to staff borrower outreach and tells a credit investor where escrow stress is concentrated.

### Affordability breaches as a delinquency signal

Cross-reference accounts whose $\Pr(\text{DTI} > 43\%)$ exceeds a chosen level (say 25%) with loan type. Given the backdrop (delinquencies near 4.37% with stress concentrated in FHA loans), an FHA loan with a high escrow-driven DTI breach probability is the highest-priority segment. For credit investors, the portfolio-level output is the expected number of such breaches by pool.

---

## Symbol reference

| Symbol | Meaning | Step |
|---|---|---|
| $i$, $j$, $c$, $s$ | Property/account, taxing jurisdiction, county, state | all |
| $t$, $q$, $m$ | Year, quarter, month within the escrow computation year | all |
| $V_{i,t}$, $\tilde V$ | Market (just) value on January 1; AVM median | 2.1, 3.1 |
| $A_{i,t}$ | Assessed value | 2.1 |
| $\kappa_j$ | Assessment ratio (assessed ÷ market) | 2.1 |
| $\gamma^{\text{cap}}_t$ | Annual cap on assessed value growth | 2.1 |
| $\text{EX}_{i,t}$, $\text{TV}_{i,t}$ | Exemptions; taxable value | 2.1 |
| $\tau_{j,t}$ | Total millage rate (mills, \$ per \$1,000) | 2.1, 2.2 |
| $T_{i,t}$, $T^{\text{disb}}_{i,t}$, $d$ | Tax bill; tax disbursed; early-payment discount | 2.1 |
| $g_{j,t}$ | Growth in taxable value of existing property in the jurisdiction | 2.1 → 2.2 |
| $b_{j,t}$ | Budget pressure | 2.2 |
| $y_{j,t}$ | Log change in millage | 2.2 |
| $\alpha_j, \alpha_c, \alpha_s$ | Baseline rate drift at jurisdiction, county, state level | 2.2 |
| $\omega_J, \omega_C, \omega_S$ | Spread of baselines at each level | 2.2 |
| $\psi_g, \psi_b$ | Rollback response; budget response | 2.2 |
| $\sigma_\varepsilon$, $w_j$ | Year-shock sd; pooling weight | 2.2 |
| $\mu_{i,t}$ | GLM expected premium | 2.3 |
| $\beta_{0,t}$, $\beta_j$, $x_{ij}$ | Log base rate; log relativities; rating characteristics | 2.3 |
| $\phi$ | Gamma dispersion | 2.3 |
| $r_q$ | Log change in reinsurance index | 2.3 |
| $\delta_k$, $a_0, a_1, a_2$ | Distributed-lag weights; Almon polynomial coefficients | 2.3 |
| $c_0$ | Quarterly non-reinsurance cost trend | 2.3 |
| $I_{i,t}$ | Insurance premium | 2.3–2.5 |
| $\ell^{\text{d}}, \ell^{\text{j}}, \ell$ | Log premium change: drift, jump, combined | 2.3–2.4 |
| $\sigma_\nu$ | Account-level drift noise | 2.3 |
| $\mathbf{h}_i$, $s_i$ | Classifier features; raw boosting score | 2.4 |
| $a^{\text{P}}, b^{\text{P}}$ | Platt scaling parameters | 2.4 |
| $p_i$, $\xi_i$ | Calibrated jump probability; jump indicator | 2.4 |
| $u^{\text{mkt}}, u^{\text{idio}}, \lambda, \zeta$ | Market and idiosyncratic value shocks; loading and noise of $g$ on the market | 2.5 |
| $D$, $C$, $B_m$, $B^{*}$, $B^{\text{act}}$ | Annual disbursements; cushion; monthly balance; required and actual starting balance | 2.5 |
| $S$ | Shortage (+ deficiency) or surplus (−) | 2.5 |
| $E_0, E_1, \Delta E$ | Current and new monthly escrow payment; change | 2.5 |
| $\text{PI}$, PITI, DTI | Principal & interest; total payment; debt-to-income | 2.5 |
